# MySQL 8.4 InnoDB Cluster on AWS EC2 (2 Nodes) with MySQL Router and a PHP Client

> **Purpose of this document:** a complete, self-contained record of how a two-node MySQL 8.4 InnoDB Cluster was built on AWS EC2 (converted from an existing source/replica setup), fronted by MySQL Router on a separate application server, and tested with PHP. It also records every problem hit along the way and its fix, so it can be used to answer questions about the setup.
>
> **Conventions:** passwords are shown as placeholders such as `<CLUSTERADMIN_PASSWORD>`. Hostnames `node1` and `node2` are aliases defined in `/etc/hosts` (see section 4.2). Items marked **(not yet tested in this lab)** were described but not executed.

---

## 1. Summary (TL;DR)

- Two EC2 instances running MySQL Server 8.4.11 were previously configured as classic master/slave (source/replica) replication. The data was non-critical test data.
- They were converted into a **single-primary InnoDB Cluster** (Group Replication) using **MySQL Shell's AdminAPI in Python mode**.
- The second node was added using the **clone** recovery method, which overwrote its data with a copy of the primary.
- A **third EC2 instance** (the app server) runs **MySQL Router**, bootstrapped in a standalone directory (no systemd), and a **PHP CLI** test script.
- Verified results:
  - Router port `6446` routes writes to the primary (`read_only = 0`).
  - Router port `6447` routes reads to the secondary (`read_only = 1`).
  - Data written on the primary appears on the secondary.
  - A controlled **switchover** (`set_primary_instance`) moves the primary without changing application code.
- **Key limitation:** with only 2 nodes the cluster status is `OK_NO_TOLERANCE`. If one node fails unexpectedly, the survivor has no majority and cannot continue automatically. Production needs at least 3 nodes.

---

## 2. Environment and topology

### 2.1 Hosts

| Role | OS hostname | Alias in `/etc/hosts` | Private IP |
|---|---|---|---|
| DB node 1 (initial primary, formerly master) | `master-ec2` | `node1` | `10.0.41.224` |
| DB node 2 (secondary, formerly replica) | `ip-10-0-8-85` | `node2` | `10.0.8.85` |
| App server (Router + PHP) | `ip-10-0-5-95` | n/a | `10.0.5.95` |

All three instances are in the **same VPC**. Each DB node has its **own, separate security group**.

### 2.2 Software versions

| Component | Version / note |
|---|---|
| OS | Ubuntu 26.04 |
| MySQL Server | 8.4.11 (Ubuntu package `8.4.11-0ubuntu0.26.04.1`) |
| MySQL Shell | 8.4.8, Ubuntu "source distribution" build, **no JavaScript support** (Python mode only) |
| MySQL Router | Ubuntu `mysql-router` package. It ships the binary but **no systemd unit and no `mysqlrouter` system user** |
| PHP | 8.5 CLI with `php-mysql` (PDO MySQL) |
| Instance RAM | about 1 GB (908 MB visible), no swap initially |

### 2.3 Architecture

```
                    +--------------------------------------+
                    |   App server  (10.0.5.95)            |
                    |   PHP app  --->  MySQL Router        |
                    |                  6446 (R/W)          |
                    |                  6447 (R/O)          |
                    +-----------------+--------------------+
                                      | TCP 3306
                +---------------------+---------------------+
                |                                           |
   +------------v-------------+              +--------------v-----------+
   | node1  10.0.41.224       |  Group       | node2  10.0.8.85         |
   | master-ec2               | Replication  | ip-10-0-8-85             |
   | PRIMARY (R/W)            |<------------>| SECONDARY (R/O)          |
   +--------------------------+  (port 3306) +--------------------------+
```

---

## 3. Concepts

### 3.1 What an InnoDB Cluster is

An InnoDB Cluster is made of three components:

1. **MySQL Group Replication**: the engine that keeps the data on all members in sync.
2. **MySQL Shell (AdminAPI)**: the tool used to create, inspect and administer the cluster (`dba.*` and `cluster.*` calls).
3. **MySQL Router**: a lightweight proxy that applications connect to; it reads the cluster metadata and forwards each connection to the right node.

### 3.2 Single-primary mode

One member is the **PRIMARY** (read/write); the others are **SECONDARY** (read-only, `super_read_only=ON`). If the primary changes, the roles swap and Router follows automatically.

### 3.3 Quorum and why 2 nodes is not fault tolerant

Group Replication needs a **majority** of members to be reachable to make decisions.

| Members | Majority | Failures tolerated |
|---|---|---|
| 1 | 1 | 0 |
| 2 | 2 | **0** |
| 3 | 2 | 1 |
| 5 | 3 | 2 |

With 2 nodes, losing one leaves 1 of 2, which is not a majority, so the survivor blocks writes. `cluster.status()` therefore reports `OK_NO_TOLERANCE` even when both nodes are healthy. This is expected and not an error.

### 3.4 Clone recovery

When adding an instance with `recoveryMethod: 'clone'`, MySQL's clone plugin **wipes the new instance's data** and copies a full snapshot from a donor. The recipient restarts itself at the end. This requires MySQL to be managed by a supervisor such as systemd (it was).

### 3.5 Communication stack

In this setup the AdminAPI printed: *"Group Replication will communicate with other members using 'node1:3306'"*. That means the cluster uses the **MySQL communication stack on port 3306**, not the older XCom stack on `33061`. Consequently, **only port 3306 needs to be open between nodes.** (An early instruction to open `33061` was unnecessary for this setup.)

### 3.6 MySQL Router ports (as seen listening in this lab)

| Port | Purpose |
|---|---|
| `6446` | Classic protocol, **read/write**, routed to the primary |
| `6447` | Classic protocol, **read-only**, routed to secondaries |
| `6448` | X Protocol, read/write |
| `6449` | X Protocol, read-only |
| `6450` | Classic protocol, **read/write splitting** (Router separates reads and writes on one connection) |
| `8443` | Router REST / monitoring API |

---

## 4. Step-by-step build

### 4.1 AWS networking

All instances must be in the **same VPC**. Security group inbound rules needed:

| Security group | Port | Source | Why |
|---|---|---|---|
| node1's SG | `3306` | node2's private IP `/32` | node-to-node traffic |
| node2's SG | `3306` | node1's private IP `/32` | node-to-node traffic |
| node1's SG | `3306` | app server private IP `/32` | Router connects to the nodes |
| node2's SG | `3306` | app server private IP `/32` | Router connects to the nodes |
| app server's SG | `22` | your admin IP | SSH (and `8000` only if you later serve a web app) |

Notes:
- Because the two nodes are in **different** security groups, each group must name the other node's IP (or the other node's SG id) as a source.
- `33061` and `33060` are **not required** for this setup. `33060` (X Protocol) is only needed if clients use X Protocol through remote hosts.
- The nodes bind X Protocol to `127.0.0.1:33060` by default on Ubuntu. This did not block anything here.

### 4.2 Name resolution (`/etc/hosts`)

On **all three servers** (nodes and app server) add the same two lines:

```
10.0.41.224  node1
10.0.8.85    node2
```

Verify with `getent hosts node1` and `getent hosts node2`. Prefer `getent`/`nc` over `ping`, because ICMP is often blocked by security groups.

Why it matters: each MySQL instance advertises itself to the cluster by its `report_host` name, and Router reads those names from the metadata. Every machine that talks to the cluster must therefore resolve `node1` and `node2`.

### 4.3 MySQL server settings (each DB node)

Required per node:

| Setting | node1 | node2 |
|---|---|---|
| `server_id` | `1` | `2` |
| `report_host` | `node1` | `node2` |
| `bind-address` | `0.0.0.0` | `0.0.0.0` |

Verify:

```sql
SELECT @@server_id, @@report_host;
```

**Important Ubuntu gotchas:**

- The **server** config file is `/etc/mysql/mysql.conf.d/mysqld.cnf` (section `[mysqld]`).
- `/etc/mysql/mysql.conf.d/mysql.cnf` is the **client** config (section `[mysql]`). Putting `report_host` there has no effect on the server and the variable stays `NULL`.
- A reliable alternative that avoids file editing is `PERSIST_ONLY`, followed by a restart:

```sql
SET PERSIST_ONLY report_host = 'node1';   -- 'node2' on the second node
```
```bash
sudo systemctl restart mysql
```

- Do **not** run a bare `RESET PERSIST;`. It removes *all* persisted variables, including a persisted `server_id`. Always name the variable.
- `server_id` must be **unique** per node.

### 4.4 Memory: swap and tuning (important on 1 GB instances)

The instances had about 908 MB RAM and no swap. During the first `create_cluster`, the Linux OOM killer terminated `mysqld` (visible in `dmesg` as `Out of memory: Killed process ... (mysqld)` and in `systemctl status` as `Failed with result 'oom-kill'`). Fix, applied on **every DB node**:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -m          # Swap total should show about 2047
```

Recommended MySQL limits to reduce Group Replication memory use:

```sql
SET PERSIST group_replication_message_cache_size = 134217728;  -- 128 MB (default is 1 GB cap)
SET PERSIST max_connections = 50;
```

If a node is read-only and `SET PERSIST` is refused, first run `SET GLOBAL super_read_only = OFF;`. A larger instance (for example 2 GB RAM) is the more robust fix.

### 4.5 Retire the old source/replica replication

On the old replica (node2):

```sql
STOP REPLICA;
RESET REPLICA ALL;
SHOW REPLICA STATUS\G      -- must return: Empty set
```

Group Replication and classic async replication on the same node must not be left configured together. Because the replica is overwritten by clone later, its previous data does not matter.

### 4.6 Cluster admin account (both nodes)

```sql
CREATE USER 'clusteradmin'@'%' IDENTIFIED BY '<CLUSTERADMIN_PASSWORD>';
GRANT ALL PRIVILEGES ON *.* TO 'clusteradmin'@'%' WITH GRANT OPTION;
```

Notes:
- The replica runs with `super_read_only=ON`, so `CREATE USER` there fails with `ERROR 1290 ... --super-read-only`. In this lab the user already existed on node2 because it had been **replicated from the master** earlier. Check first with `SELECT user, host FROM mysql.user WHERE user='clusteradmin';`.
- `ALL PRIVILEGES` is acceptable for a lab. Production should grant only the privileges the AdminAPI documents.
- The password must be the same on both nodes.
- A remote login test that worked: `mysql -h node2 -u clusteradmin -p -e "SELECT @@hostname, @@server_id;"`.

### 4.7 MySQL Shell (Python mode)

Install on the node used for administration (node1):

```bash
sudo apt install mysql-shell
```

The Ubuntu build has **no JavaScript**: `\js` gives `Unknown command` and `mysqlsh --js` gives `JavaScript is not supported.` Use Python:

```bash
mysqlsh --py clusteradmin@node1:3306
```

Python-mode rules:
- Prompt shows `Py >`.
- AdminAPI names are **snake_case** (`dba.create_cluster`, not `dba.createCluster`).
- Options are passed as a dict: `{'recoveryMethod': 'clone'}`.
- `\sql` / `\py` switch modes; `\connect user@host:port` changes the target; `\quit` exits.
- In **SQL mode** a statement without `;` is treated as incomplete and the prompt becomes `->`.
- Variables such as `cluster` exist only inside one shell session.

### 4.8 Validate both instances

From the shell connected to node1:

```python
dba.check_instance_configuration('clusteradmin@node1:3306')
dba.check_instance_configuration('clusteradmin@node2:3306')
```

Both returned `"status": "ok"`. (GTID was already enabled from the earlier replication, so `dba.configure_instance()` was not needed. If the status is `error`, run `dba.configure_instance()` on that instance; it fixes GTID/binlog settings and tells you if a restart is required.)

### 4.9 Create the cluster and add the second node

Connected to **node1**:

```python
cluster = dba.create_cluster('myCluster')
cluster.status()
```

Expected: node1 `PRIMARY`, `R/W`, `ONLINE`, cluster status `OK_NO_TOLERANCE`.

If a previous attempt crashed midway (see section 7), clean up node1 first (section 6.4).

Then add node2:

```python
cluster.add_instance('clusteradmin@node2:3306', {'recoveryMethod': 'clone'})
```

What happens: node2's data is dropped, a full copy is cloned from node1, node2 restarts itself, and it rejoins. Do not press Ctrl+C during this. The output ended with:
`The instance 'node2:3306' was successfully added to the cluster.`

Final status (verified): node1 `PRIMARY`/`R/W`/`ONLINE`, node2 `SECONDARY`/`R/O`/`ONLINE`, cluster `OK_NO_TOLERANCE`, topology `Single-Primary`.

### 4.10 Quick replication test

On node1:

```sql
CREATE DATABASE cluster_test;
CREATE TABLE cluster_test.t1 (id INT PRIMARY KEY, name VARCHAR(50));
INSERT INTO cluster_test.t1 VALUES (1, 'hello cluster');
```

On node2: `SELECT * FROM cluster_test.t1;` returns the row. An `INSERT` on node2 fails with a `super-read-only` error, which is correct behavior.

Group Replication requires all tables to be InnoDB with a **primary key**.

### 4.11 Application server preparation

1. Launch in the same VPC. Add the `/etc/hosts` lines from 4.2.
2. Add the security-group rules from 4.1 (app server IP to both nodes on `3306`).
3. Confirm connectivity:

```bash
nc -zv -w 3 node1 3306
nc -zv -w 3 node2 3306
```

4. Install packages:

```bash
sudo apt update
sudo apt install mysql-router mysql-client php-cli php-mysql
```

5. On the **primary** (node1) create the application user (it replicates to node2):

```sql
CREATE USER 'appuser'@'%' IDENTIFIED BY '<APPUSER_PASSWORD>';
GRANT ALL PRIVILEGES ON cluster_test.* TO 'appuser'@'%';
```

### 4.12 MySQL Router bootstrap (standalone, no systemd)

The Ubuntu package does not create a `mysqlrouter` user or a systemd unit. This command therefore fails:

```bash
sudo mysqlrouter --bootstrap clusteradmin@node1:3306 --user=mysqlrouter
# Error: Can't use user 'mysqlrouter'. Please check that the user exists!
```

The working approach is a self-contained directory, run as the normal user without `sudo`:

```bash
mysqlrouter --bootstrap clusteradmin@node1:3306 --directory $HOME/myrouter
$HOME/myrouter/start.sh
sleep 3
ss -ltnp | grep -E "6446|6447"
```

Bootstrap output ends with lines like `Read/Write Connections: localhost:6446` and `Read/Only Connections: localhost:6447`. Bootstrap creates a dedicated Router account in the cluster and writes the configuration, `start.sh`, `stop.sh` and logs into `$HOME/myrouter`.

Rules:
- Bootstrap must target the **current primary**.
- `--force` can be added to redo a previous bootstrap in an existing directory.
- Router listens on `0.0.0.0` by default. For tighter exposure add `--conf-bind-address 127.0.0.1` at bootstrap when Router and the app share a host.
- Because there is no systemd unit, Router does **not** start automatically after an instance reboot. Run `$HOME/myrouter/start.sh` again.
- Logs: `$HOME/myrouter/log/mysqlrouter.log`. Stop: `$HOME/myrouter/stop.sh`.

### 4.13 Test through Router

With the MySQL client:

```bash
mysql -h 127.0.0.1 -P 6446 -u appuser -p -e "SELECT @@hostname, @@read_only;"
mysql -h 127.0.0.1 -P 6447 -u appuser -p -e "SELECT @@hostname, @@read_only;"
```

With PHP (`~/router_test.php`):

```php
<?php
$opt = [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION];

// Write: port 6446 (primary)
$w = new PDO('mysql:host=127.0.0.1;port=6446;dbname=cluster_test', 'appuser', '<APPUSER_PASSWORD>', $opt);
$w->prepare('INSERT INTO t1 (id, name) VALUES (?, ?)')->execute([rand(10, 999999), 'from php']);
print_r($w->query('SELECT @@hostname AS host, @@read_only AS ro')->fetch(PDO::FETCH_ASSOC));

// Read: port 6447 (secondary)
$r = new PDO('mysql:host=127.0.0.1;port=6447;dbname=cluster_test', 'appuser', '<APPUSER_PASSWORD>', $opt);
print_r($r->query('SELECT @@hostname AS host, @@read_only AS ro')->fetch(PDO::FETCH_ASSOC));
echo $r->query('SELECT COUNT(*) FROM t1')->fetchColumn() . " rows\n";
```

**Verified output:** write side `master-ec2`, `ro = 0`; read side `ip-10-0-8-85`, `ro = 1`; `3 rows` visible on the secondary (proving replication).

### 4.14 Switchover test

On node1, in `mysqlsh --py`:

```python
cluster = dba.get_cluster()
cluster.set_primary_instance('node2:3306')
cluster.status()
```

Then rerun `php ~/router_test.php`. The roles invert: the write side now reports `ip-10-0-8-85` with `ro = 0` and the read side reports `master-ec2` with `ro = 1`, with no application change. A one-to-few-second pause or one failed connection can occur while Router notices the change and old connections are dropped. Restore with `cluster.set_primary_instance('node1:3306')`.

This is a **controlled switchover** with both nodes online, not an automatic failover test (see section 7).

### 4.15 Laravel **(not yet tested in this lab)**

Laravel only needs to point at Router instead of a MySQL host. In `.env`:

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=6446
DB_DATABASE=cluster_test
DB_USERNAME=appuser
DB_PASSWORD=<APPUSER_PASSWORD>
```

Optional read/write splitting in `config/database.php`: in the `mysql` connection add a `read` array with `'host' => ['127.0.0.1'], 'port' => 6447`, a `write` array with `'host' => ['127.0.0.1'], 'port' => 6446`, and `'sticky' => true`. `composer` can use significant RAM on a 1 GB host; swap is helpful. Laravel migrations create tables with primary keys, which Group Replication requires. To reach a dev server from a browser, open the chosen port (for example `8000`) in the app server's security group.

---

## 5. Operations

### 5.1 Everyday commands (MySQL Shell, Python mode)

```python
cluster = dba.get_cluster()
cluster.status()                                   # health overview
cluster.describe()                                 # topology
cluster.set_primary_instance('node2:3306')         # controlled switchover
cluster.remove_instance('clusteradmin@node2:3306') # remove a member
```

SQL view of Group Replication membership:

```sql
SELECT * FROM performance_schema.replication_group_members;
```

### 5.2 Recovering after **both** nodes were stopped

If both DB nodes are shut down at the same time (a complete outage), the cluster does not re-form by itself, because nobody bootstraps the group. After both are running again, from a shell connected to either node:

```python
dba.reboot_cluster_from_complete_outage('myCluster')
```

Answer `y` when asked to rejoin the other instance, then verify with `dba.get_cluster().status()`. Start **Router again** afterwards if needed.

### 5.3 Starting and stopping Router

```bash
$HOME/myrouter/start.sh
$HOME/myrouter/stop.sh
ss -ltnp | grep mysqlrouter        # shows 6446/6447/... when running
```

### 5.4 Cost note

Three always-on EC2 instances cost money. Stop them when not needed. A stopped instance causes `No route to host` from other servers, and stopping both DB nodes requires the recovery in 5.2.

---

## 6. Troubleshooting (problems actually encountered)

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | `ping: node2: Temporary failure in name resolution` | No `/etc/hosts` entry | Add `node1`/`node2` lines on every server; verify with `getent hosts` |
| 2 | `report_host` stays `NULL` after restart | Line added to client file `mysql.cnf`, or no restart | Use `mysqld.cnf` under `[mysqld]`, or `SET PERSIST_ONLY report_host=...` then restart |
| 3 | `ERROR 1290 ... --super-read-only` when creating a user on the replica | Replica is read-only by design | Check if the user already replicated; otherwise stop replication and `SET GLOBAL super_read_only=OFF` temporarily |
| 4 | `Unknown command: '\js'` / `JavaScript is not supported.` | Ubuntu `mysqlsh` built without JS | Use `mysqlsh --py` and snake_case AdminAPI names |
| 5 | Prompt shows `->` and commands do nothing | Shell is in SQL mode and sees an unfinished statement | Enter `;` then `\py` (or `\quit` and restart the shell) |
| 6 | `Lost connection ... reading initial communication packet` during `create_cluster` | `mysqld` was **OOM-killed** (confirmed in `dmesg`) | Add swap, lower `group_replication_message_cache_size`, clean up (6.4), retry |
| 7 | `error.log`: `Unable to join the group: peers not configured`, `group_replication_group_name option is mandatory` | Half-finished cluster config left after the crash | Clean up as in 6.4 |
| 8 | `nc: ... No route to host` | Target EC2 instance was **stopped**, or VPC/OS firewall issue | Start the instance; confirm same VPC; check `iptables -S` for REJECT rules |
| 9 | `nc: ... timed out` | Missing security-group rule | Add inbound `3306` from the client's private IP `/32` |
| 10 | `NameError: name 'cluster' is not defined` | Variable lost with the old shell session | `\connect clusteradmin@node1:3306` then `cluster = dba.get_cluster()` |
| 11 | Cluster not found / members OFFLINE after both nodes restarted | Complete outage | `dba.reboot_cluster_from_complete_outage('myCluster')` |
| 12 | `Error: Can't use user 'mysqlrouter'` | Ubuntu package creates no such user | Bootstrap with `--directory $HOME/myrouter` |
| 13 | `Unit mysqlrouter.service could not be found` | Ubuntu package ships no unit | Use `start.sh` / `stop.sh` in the bootstrap directory |
| 14 | Router gone after reboot | No systemd unit | Run `$HOME/myrouter/start.sh` again |
| 15 | PHP: `could not find driver` | PDO MySQL missing | `sudo apt install php-mysql` |

### 6.4 Cleaning up a half-created cluster on node1

Use only when a failed `create_cluster` left leftovers (data in this lab was non-critical). In `sudo mysql` on node1:

```sql
STOP GROUP_REPLICATION;
RESET REPLICA ALL;
RESET PERSIST IF EXISTS group_replication_start_on_boot;
RESET PERSIST IF EXISTS group_replication_group_name;
RESET PERSIST IF EXISTS group_replication_local_address;
RESET PERSIST IF EXISTS group_replication_group_seeds;
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;
DROP DATABASE IF EXISTS mysql_innodb_cluster_metadata;
SELECT @@server_id, @@report_host;      -- confirm values are still intact
```

---

## 7. Limitations and production recommendations

- **2 nodes: no automatic failover.** If the primary crashes, the secondary cannot reach quorum. Manual recovery from the survivor is possible with `cluster.force_quorum_using_partition_of('node2:3306')` (use with care, only when the other node is truly down). The correct fix is a **third member**.
- **Resources:** about 1 GB RAM is too small for Group Replication without swap. Prefer at least 2 GB per DB node.
- **Router:** in production run Router on each app server (or behind a load balancer), under a service manager, and restrict its bind address.
- **Privileges:** use least-privilege accounts instead of `ALL PRIVILEGES`.
- **Network:** keep node-to-node and Router-to-node traffic on private IPs; restrict security groups to specific sources.
- **Backups:** a cluster is high availability, not a backup. Take regular backups.
- **Switchover is not failover:** the tested switchover needs both nodes healthy.

---

## 8. Frequently asked questions

**Q: Can this be built with only 2 nodes?**
Yes, it works, but it has no fault tolerance (status `OK_NO_TOLERANCE`). It is fine for learning; production needs 3 or more nodes.

**Q: Why did `create_cluster` fail with "Lost connection"?**
The instance had about 908 MB RAM and no swap, and the kernel OOM-killed `mysqld`. Adding swap and lowering the Group Replication message cache solved it.

**Q: What does `OK_NO_TOLERANCE` mean?**
The cluster is healthy now but cannot survive the loss of any member. With 2 members that is always the case.

**Q: Which ports must be open?**
`3306` between the two nodes and from the app server (Router) to both nodes. The default MySQL communication stack means `33061` is not needed.

**Q: Why is `report_host` required?**
It is the name each member advertises. Other members and Router connect to that name, so it must resolve everywhere through `/etc/hosts` (or DNS).

**Q: Why can't I run `\js` in MySQL Shell?**
Ubuntu's `mysqlsh` package is built without JavaScript. Use Python mode: `mysqlsh --py`, with snake_case function names.

**Q: Why did the config change not apply to `report_host`?**
On Ubuntu, `mysql.cnf` is the client file. Server options belong in `mysqld.cnf` under `[mysqld]`, or can be set with `SET PERSIST_ONLY`.

**Q: What happens to node2's old data when it is added with clone?**
It is deleted and replaced by a full copy of the donor's data. That is why only non-critical data should be on the joining node.

**Q: Why does node2 restart during `add_instance`?**
The clone operation finishes by restarting the recipient so it can start with the cloned data.

**Q: Why is the secondary read-only, and can I write to it?**
Secondaries run with `super_read_only=ON` in single-primary mode. Writes must go to the primary; Router port `6446` does that automatically.

**Q: What do Router ports 6446 and 6447 do?**
`6446` sends connections to the primary (read/write); `6447` sends them to secondaries (read-only). `6450` offers read/write splitting on a single connection.

**Q: Where is Router installed and why?**
On the app server, next to the application. Applications then connect to `127.0.0.1:6446/6447` and never need to know which node is primary.

**Q: Why not use systemd for Router?**
The Ubuntu package provides no unit and no `mysqlrouter` user. A standalone bootstrap directory with `start.sh`/`stop.sh` works without either. A custom systemd unit could be written later.

**Q: Router stopped after the server rebooted. Why?**
Without a systemd unit it is not started at boot. Run `$HOME/myrouter/start.sh`.

**Q: What if both DB nodes were stopped?**
The cluster must be rebooted with `dba.reboot_cluster_from_complete_outage('myCluster')`.

**Q: What does the switchover test show?**
After `cluster.set_primary_instance('node2:3306')`, the same PHP script writes to node2 and reads from node1, without any code or configuration change, because Router follows the cluster metadata.

**Q: What happens if the primary crashes suddenly in this 2-node setup?**
The remaining node loses quorum and blocks writes. An administrator must intervene (for example with `force_quorum_using_partition_of`). A third node avoids this.

**Q: Why do tables need a primary key?**
Group Replication requires InnoDB tables with a primary key; `check_instance_configuration` reports tables that do not comply.

**Q: Is Laravel working with this?**
Laravel was documented (section 4.15) but **not tested** in this lab. It only needs `DB_HOST=127.0.0.1` and `DB_PORT=6446`; plain PHP/PDO through Router was tested successfully.

---

## 9. Quick command reference

```bash
# Name resolution / connectivity
getent hosts node1
nc -zv -w 3 node2 3306

# Server settings
mysql -e "SELECT @@server_id, @@report_host, @@super_read_only;"

# Shell (Python mode)
mysqlsh --py clusteradmin@node1:3306
```

```python
dba.check_instance_configuration('clusteradmin@node1:3306')
cluster = dba.create_cluster('myCluster')
cluster.add_instance('clusteradmin@node2:3306', {'recoveryMethod': 'clone'})
cluster = dba.get_cluster()
cluster.status()
cluster.set_primary_instance('node2:3306')
dba.reboot_cluster_from_complete_outage('myCluster')
```

```bash
# Router (app server)
mysqlrouter --bootstrap clusteradmin@node1:3306 --directory $HOME/myrouter
$HOME/myrouter/start.sh
$HOME/myrouter/stop.sh
ss -ltnp | grep -E "6446|6447"
php ~/router_test.php
```

---

## 10. Glossary

| Term | Meaning |
|---|---|
| **Source/Replica (master/slave)** | Classic asynchronous replication; one writable source, read-only replicas |
| **Group Replication (GR)** | Plugin that replicates data across a group with majority-based consistency |
| **InnoDB Cluster** | Group Replication + MySQL Shell + MySQL Router managed as one system |
| **AdminAPI** | MySQL Shell's `dba` and `cluster` objects for administering clusters |
| **Single-primary** | One member accepts writes; others are read-only |
| **Quorum** | Majority of members needed to make progress |
| **Clone plugin** | Copies a full snapshot of a donor's data to a new member |
| **GTID** | Global transaction identifiers that track replicated transactions |
| **Router bootstrap** | Router reading cluster metadata and generating its own configuration |
| **Switchover** | Planned, controlled change of primary with all members healthy |
| **Failover** | Unplanned promotion after a member fails (needs a majority) |
| **Complete outage** | All cluster members down at the same time |
| **OOM kill** | Linux terminating a process that exhausts memory |
