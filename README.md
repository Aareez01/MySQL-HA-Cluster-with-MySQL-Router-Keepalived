# How to Deploy MySQL InnoDB Cluster (3-Node HA Setup with MySQL Router & Keepalived)

---

# Introduction

If one MySQL server fails…
does your application survive?

In this guide, I’ll show you how to build a **production-grade MySQL InnoDB Cluster** that can tolerate one server failure with automatic failover.

Minimum requirement:

* 3 MySQL servers
  Ideal setup:
* 5 servers including MySQL Router and Keepalived

By the end of this, you’ll have:

* High Availability
* Automatic Primary Election
* Cluster-based replication
* Optional Virtual IP failover

Let’s get started.

---

# ARCHITECTURE OVERVIEW

Minimum Production Setup (Fault Tolerant):

* MySQL Server 1
* MySQL Server 2
* MySQL Server 3

Optional:
2 additional servers for:
  * MySQL Router
  * Keepalived (VIP layer)

Why 3 servers?

Because MySQL InnoDB Cluster uses Group Replication.
To tolerate 1 failure you need majority voting.

3 nodes → Can survive 1 failure
5 nodes → Can survive 2 failures

---

# REQUIRED TOOLS

We will install:

1. MySQL Server
2. MySQL Shell
3. MySQL Router
4. Keepalived (optional, for VIP)

---

# STEP 1 – Add MySQL Repository & Install Required Software

**Why not use default repository?**

Because system default repositories may not contain:
* Latest MySQL Router
* Compatible version with your MySQL cluster

So you add official MySQL repo first. This ensures version compatibility with your cluster.
```bash https://dev.mysql.com/downloads/repo/apt/
```

On all database servers:

Ubuntu:

```bash
apt update
apt install mysql-server mysql-shell mysql-router -y
```

RHEL/CentOS:

```bash
yum install mysql-server mysql-shell mysql-router -y
```

Set clear hostnames:

```bash
hostnamectl set-hostname dbserver-1
hostnamectl set-hostname dbserver-2
hostnamectl set-hostname dbserver-3
```

Restart server if required.

---

# STEP 2 – Create Cluster Admin User

On all MySQL servers:

```bash
mysql -u root -e "CREATE USER 'clusteradmin'@'%' IDENTIFIED BY '123123';"
```

Grant privileges:

```bash
mysql -u root -e "GRANT ALL PRIVILEGES ON *.* TO 'clusteradmin'@'%' WITH GRANT OPTION;"
```

Reload privileges:

```bash
mysql -u root -e "FLUSH PRIVILEGES;"
```

---

# STEP 3 – Reset Binary Logs (Fresh Setup Only)

**Only Reset Master on fresh installations.**

```bash
mysql -u root -e "RESET MASTER;"
```

Verify hostname:

```bash
mysql -u root -e "SELECT @@hostname;"
```

Verify user exists:

```bash
mysql -u root -e "SELECT user FROM mysql.user WHERE user='clusteradmin';"
```

---

# STEP 4 – Configure MySQL Network Binding

Edit configuration:

```bash
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Under `[mysqld]` add:

```
bind-address = 0.0.0.0
mysqlx-bind-address = 0.0.0.0
```

Restart MySQL:

```bash
systemctl restart mysql
```

---

# STEP 5 – Use MySQL Shell

Now we switch to MySQL Shell:

```bash
mysqlsh
```

---

# STEP 6 – Check Instance Configuration

Run for each server:

```javascript
dba.checkInstanceConfiguration("clusteradmin@dbserver1:3306")
```

Fix issues if reported.

---

# STEP 7 – Configure Instance for Cluster

Run:

```javascript
dba.configureInstance("clusteradmin@dbserver1:3306")
```

Repeat for dbserver2 and dbserver3.

Restart MySQL if prompted.

---

# STEP 8 – Connect to Primary Node

```javascript
\connect clusteradmin@dbserver1:3306
```

---

# STEP 9 – Create the Cluster

```javascript
var myCluster = dba.createCluster("dbcluster")
```

This initializes Group Replication.

---

# STEP 10 – Add Remaining Nodes

```javascript
myCluster.addInstance('clusteradmin@dbserver2:3306')
myCluster.addInstance('clusteradmin@dbserver3:3306')
```

Cluster will synchronize automatically.

---

# STEP 11 – Verify Cluster Status

```javascript
myCluster.status()
```

You should see:

* 1 PRIMARY
* 2 SECONDARY
* Status: OK

Congratulations 🎉
You now have a fault-tolerant MySQL cluster.

---

# MySQL Router — Why We Need MySQL Router

---

In InnoDB Cluster:

* Group Replication handles database failover
* But your application still needs a stable endpoint

That’s where MySQL Router comes in.

MySQL Router:

* Knows who is PRIMARY
* Automatically reroutes traffic
* Removes the need for manual DB host switching

Without Router:
Your app must manually change DB host on failover 

With Router:
Your app connects once — and Router handles the rest 

---

# What You’re Doing in These Steps

You are:
1. Installing MySQL Router (Ignore if already installed)
2. Creating dedicated router accounts
3. Bootstrapping router instances
4. Connecting routers to the cluster metadata

Let’s break each part down.

---

# STEP 2 – Install MySQL Router

```bash
apt install mysql-router
```

This installs:

* Router binaries
* Routing engine
* Metadata integration components

Router is lightweight.
It does NOT store data.
It just forwards traffic intelligently.

---

# STEP 3 – Connect to Any Cluster Node

You connect using:

```bash
mysqlsh
```

---

# STEP 4 – Create Router Accounts

```javascript
dba.getCluster("dbcluster").setupRouterAccount("router-1")
dba.getCluster("dbcluster").setupRouterAccount("router-2")
```

What does this do?

It creates internal accounts inside the cluster with:

* Limited privileges
* Metadata read access
* Authentication for router

Why create separate accounts?

Because in production:

* Each router instance should have its own identity
* Better auditing
* Better security
* Easy revocation

**This is enterprise best practice.**

---

# STEP 5 – Bootstrapping Router

Now the important part.

Example:

```bash
mysqlrouter --bootstrap clusteradmin@dbserver1 --user=root -d myrouter_idc --account=router_1
```

Let’s break this down:

### --bootstrap

This tells Router:
"Connect to the cluster and auto-configure yourself."

During bootstrap, Router:

* Reads cluster metadata
* Creates routing configuration
* Generates config files
* Sets routing ports (6446, 6447 etc.)

---

### clusteradmin@dbserver1

Router connects to one cluster node.
It does NOT matter which node — cluster metadata is shared.

---

### --account=router_1

Tells Router which router account to use.
Must match account created earlier.

---

### -d myrouter_idc

Defines configuration directory.
Useful when running multiple router instances.

---

# 🔹 Second & Third Router Instances

```bash
mysqlrouter --bootstrap clusteradmin@dbserver2 --user=root
```

What this means:

You are creating multiple router instances, each bootstrapped from different nodes.

Important:

You only need bootstrap access to ONE healthy node.
After bootstrap, Router learns about entire cluster.

In production, you usually:

* Install Router on 2 separate servers
* Use Keepalived to create VIP
* Application connects to VIP
* VIP points to active router

Router handles DB failover
Keepalived handles router failover

---

# Production-Grade Architecture (Tolerates 1 Failure)

Minimum HA Design:

### Layer 1 – Database Cluster

3 MySQL nodes

* 1 PRIMARY
* 2 SECONDARY
* Majority voting

Can survive 1 node failure.

---

### Layer 2 – Router Layer

2 Router servers

Can survive 1 router failure.

---

### Layer 3 – Virtual IP (Recommended)

Keepalived provides:

* Single VIP
* Automatic router failover

Application connects only to:

```
DB_HOST=10.0.0.100
```

No changes required on DB failover.

---

# What Happens During Failure?

### Scenario 1 – Primary DB Fails

* Group Replication elects new PRIMARY
* Router detects change
* Traffic automatically shifts
* Application continues working

Zero downtime (except election time ~ few seconds)

---

### Scenario 2 – Router Server Fails

* Keepalived moves VIP
* Second Router becomes active
* Application unaffected

---

---

# Keepalived Setup Guide for MySQL Router High Availability

**Goal:** Provide a **floating Virtual IP (VIP)** in front of MySQL Routers so that applications always connect to the active router, even if one router server fails.

---

## Prerequisites

* Two or more servers for MySQL Router (Router1 & Router2)
* MySQL Routers installed and running
* Root or sudo access
* Network interface name (e.g., `enp1s0`)
  Check using:

```bash
ip addr
```

* Install Keepalived:

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install keepalived -y

# RHEL/CentOS
sudo yum install keepalived -y
```

---

# Diving into Keepalived

---

### Before diving into the keepalived configuration we need to understand two main concepts:

1. **VIP (Virtual IP)** - A Virtual IP (VIP) is a single IP address that doesn't belong to one specific physical server. Instead, it "floats" between servers in a cluster.

2. **VRRP (Virtual Router Redundancy Protocol)** - It’s a network protocol used to provide **high availability at the network level** by creating a **Virtual IP (VIP)** that can float between multiple servers. 

---

## Key Idea

* You have multiple routers (or servers) in a group
* One router is **MASTER** — it owns the VIP
* Others are **BACKUP** — they monitor MASTER
* If MASTER fails → a BACKUP automatically takes over the VIP
* Applications keep connecting to the same VIP → no downtime

Think of VRRP as a **VIP election system**.

---

## How It Works

1. Each node sends **VRRP advertisements** (heartbeat) every second
2. The node with **highest priority** becomes MASTER
3. MASTER owns the VIP
4. BACKUP nodes listen for heartbeats
5. If heartbeats stop (MASTER down), the highest-priority BACKUP becomes MASTER
6. VIP moves automatically to the new MASTER

---

## Important Terms

| Term                         | Meaning                                                               |
| ---------------------------- | --------------------------------------------------------------------- |
| **Virtual Router ID (VRID)** | Identifies the VRRP group. Must be the same on all nodes in the group |
| **Priority**                 | Determines which node becomes MASTER. Higher → MASTER                 |
| **Advertisement Interval**   | How often nodes send heartbeat packets                                |
| **State**                    | MASTER or BACKUP                                                      |
| **VIP (Virtual IP)**         | Floating IP that moves between nodes                                  |

---

## Why We Use VRRP with MySQL Router

* MySQL Router provides **database failover awareness**
* But the router itself could fail
* VRRP + Keepalived provides a **network-level VIP** in front of Router servers
* Applications connect to VIP → always reach the **active router**
* Combined with MySQL Router → **full HA stack**

---

## Train Station Announcer Analogy

* VIP = Announcement Microphone — the only way passengers hear important train updates.
* MASTER = Active Announcer — standing at the microphone and making announcements.
* BACKUP = Standby Announcers — ready to take over immediately if the MASTER leaves.
* Failover — if the MASTER steps away or is unavailable, a BACKUP instantly picks up the microphone.
* Passengers = Applications — they never miss announcements, just like apps never lose connection to the database.

---

### **Router 1 Configuration (Primary / MASTER)**

```conf
vrrp_instance VI_1 {
    state MASTER
    interface enp1s0
    virtual_router_id 50
    priority 110
    advert_int 1

    virtual_ipaddress {
        10.0.0.50
    }
}
```

### **Router 2 Configuration (Backup)**

```conf
vrrp_instance VI_1 {
    state BACKUP
    interface enp1s0
    virtual_router_id 50
    priority 100
    advert_int 1

    virtual_ipaddress {
        10.0.0.50
    }
}
```

---

## Explanation of Configuration Parameters

| Parameter              | Meaning                                                                 |
| ---------------------- | ----------------------------------------------------------------------- |
| `vrrp_instance VI_1`   | Defines a VRRP group. Must match on all participating nodes.            |
| `state MASTER/BACKUP`  | Initial role of the node. MASTER starts with VIP. BACKUP waits.         |
| `interface enp1s0`     | Network interface to bind the VIP. Must be correct.                     |
| `virtual_router_id 50` | Unique VRRP ID for this VIP. Must be the same on all nodes.             |
| `priority`             | Determines who becomes MASTER. Higher → MASTER. Backup must be lower.   |
| `advert_int`           | Heartbeat interval (seconds). Detects failures quickly.                 |
| `virtual_ipaddress`    | The floating IP shared between nodes. Application always connects here. |

---

## Start and Enable Keepalived

```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
sudo systemctl status keepalived
```

* Check VIP:

```bash
ip addr show enp1s0
```

You should see `10.0.0.50` on the MASTER node.

---

## How Failover Works

1. Both MASTER and BACKUP send VRRP heartbeats every 1 second.
2. MASTER owns the VIP (`10.0.0.50`).
3. If MASTER goes down:
   * BACKUP detects missing heartbeat
   * Backup becomes MASTER
   * VIP moves to BACKUP automatically
4. When the old MASTER comes back:
   * It may reclaim VIP if priority is higher (configurable with `nopreempt` option if you don’t want preemption)

---

## Optional Production Recommendations

* Add `nopreempt` if you don’t want MASTER reclaiming VIP automatically after recovery
* Use dedicated VLAN for VIP if possible
* Monitor Keepalived logs:

```bash
journalctl -u keepalived -f
```

* Combine with **MySQL Router**:

  * VIP points to active router
  * Router points to active MySQL PRIMARY
  * App connects to VIP → seamless failover

---

## Visual Architecture Summary

```
       +--------------------+
       |    Application     |
       +--------------------+
               |
               | 10.0.0.50 (VIP)
               |
   +--------------------+--------------------+
   |    Router 1 (MASTER)   Router 2 (BACKUP)|
   +--------------------+--------------------+
               |
       MySQL InnoDB Cluster (3 nodes)
```

* VIP floats between Router1 and Router2
* Router automatically forwards to correct MySQL PRIMARY
* Application never needs to know which router is active

---

## Troubleshooting Tips

* VIP not appearing:

  * Check network interface name
  * Check firewall blocking VRRP (Protocol 112)
* VRRP priority issues → MASTER not elected
* Logs:

```bash
tail -f /var/log/syslog | grep keepalived   # Ubuntu
tail -f /var/log/messages | grep keepalived # RHEL/CentOS
```

# Why This Is Production-Grade

**Because it eliminates:**
* Single point of failure at DB layer
* Single point of failure at routing layer
* Manual intervention during failover
* App reconfiguration during DB crash

**This is how enterprise HA is designed.**


**This setup provides:**
* Automatic Primary Election
* Synchronous Replication
* No Data Loss (when properly configured)
* Cluster Metadata Awareness
* Automatic Failover

**This is significantly more advanced than traditional Master–Slave replication.**

---

# Important Clarifications

You do NOT need:

* 3 routers
* Bootstrap from all nodes

You only need:

* 2 routers
* Bootstrap once per router
* Connect to any healthy node

---

# CONGRATS YOU'VE SUCCESSFULLY DEPLOYED A MYSQL HA CLUSTER

```
