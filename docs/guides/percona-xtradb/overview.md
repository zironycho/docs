# About Percona XtraDB Cluster (PXC)

Percona XtraDB Cluster is a fully open-source high-availability solution for MySQL. It is a integration of Percona Server and Galera replication library that enables a synchronous multi-master replication.

It is offers,

- **Multi-Master Solution:** You can write to any nodes of the cluster and the data or the transactions that is written will be replicated to the other nodes too.
- **Synchronous Replication:** Whatever you write on any of nodes, it immediately get replicated to the nodes of the cluster and then the response is sent to the application.
- **Automatic Node Provision (SST/IST):** If any new node wants to join the cluster, you will have to boot the node pointed to the cluster and it will be added automatically. From a perspective of a cluster, it is a very important aspect that you can join a new node within a few minutes or a time whatever it takes for assisting to happen. Here,
  - IST (Incremental Snapshot Transfer) means if your nodes goes out of the cluster for a short period of time, it can rejoin the cluster by just getting the missing `writesets` (transactions) from any of nodes.
  - SST (State Snapshot Transfers) means when a new node (joiner) wants to join the cluster, it gets a full copy of data from a node (donor) of the cluster to be synchronized.
- **Consistent View of Data:** For a multi-master solution, the consistency of data is an important aspect. When you query on any of these nodes, you should able to see the same result for that given particular query.
- **Support Geo-Distributed Setup:** Percona XtraDB Cluster supports geo-distributed setup of nodes. That means your cluster can reside in different data centers and they can talk to other.
- **Comapatible with Master-Slave Setup:** Percona XtraDB Cluster nodes can be used as master and slave in traditional master-slave setup.
- **Read/Write Scalability:** Percona XtraDB Cluster routes queries automatically and any read query can be satisfied by a single node.

Here, we will talk about Percona XtraDB Cluster 5.7 (PXC-5.7).

It is released as GA in Sep 2016. After then the team added lot of things to it, fixed bugs and at a regular interval they have been bringing new releases.

## PXC Strict Mode (Cluster Safe Mode)

It is also called cluster safe mode. The idea behind this is as following.

There are a lot of experimental and unsupported features that are available and are not suitable for multi-master environment. PXC Strict Mode avoids the use of these features by performing some validation. For example,

- Percona XtraDB Cluster does not support following statements for a non-transactional storage engine like MyISAM, MEMORY, CSV, etc. to ensure data consistency:

  - Data manipulation statements that perform writing to table (for example, `INSERT`, `UPDATE`, `DELETE`, etc.)
  - The following administrative statements: `CHECK`, `OPTIMIZE`, `REPAIR`, and `ANALYZE`
  - `TRUNCATE TABLE` and `ALTER TABLE`

- Since MyISAM has non-transactional nature, Percona XtraDB Cluster restricts enabling its replication to ensure data consistency by setting the value of `wsrep_replicate_myisam` to `OFF` (which is default).
- The binary logging format supported by the Percona XtraDB Cluster has to be `ROW` which is default. It is controlled by setting variable `binlog_format` to `ROW`. Setting the value other than `ROW` at startup changes the global scope.
- DML on the table without primary key is not allowed. Because if you have a table without a primary key, it is difficult for it to recognize any particular entity across the cluster uniquely. So, data manipulation statements that perform writing to table (especially `DELETE`).
- Table locking operations are only experimental in Percona XtraDB Cluster. Anything which is local to a given node of a cluster is not good for any multi-master environment like Percona XtraDB Cluster. So Operations like `LOCK TABLES` , `GET_LOCK()`, `RELEASE_LOCK()`, `FLUSH TABLES <tables> WITH READ LOCK`, etc. that may lead to explicit table locking are blocked.
- Since `CREATE TABLE ... AS SELECT` is actually a mix of DDL and DML, it is not good from a multi-master perspective because you can not roll back DDL and DDLs are still non-transactional.
- Executing local operation like `ALTER` `IMPORT`/`DISCARD` to a node, is local to that node since that operation happens with the file systems . If we try to replicate it, other node will be inconsistent state. Scenario becomes like that we discard a table on this node but other node will still have the table and active.

Percona XtraDB Cluster introduced PXC Strict Mode which can take the following four values:

- **`ENFORCING`** is default one. If we try to use any of these features, corresponding operations will be blocked by raising an error.
- **`PERMISSIVE`** logs a warning instead of generating an error and continues as normal, if a validation fails.
- **`MASTER`** is used in the case when the writes are restricted to only one master. The effects are same as `ENFORCING` except the locks.
- **`DISABLED`** leaves the cluster as it is in 5.6. In `DISABLED` mode, there is no warning, no error messages and the cluster runs as normal.

## High Availability

High availability means continue to function even in unexpected situation like a node crashing or network failure. In a 3 nodes Percona XtraDB Cluster setup, the cluster will continue to work normally if any of the nodes down at any point in time. It will be able to run fine. In a situation like this there may happen any of the following two:

- SST
- IST

A cluster consists of nodes, where each node contains the same set of data synchronized across nodes. The recommended configuration is to have at least 3 nodes, but you can have 2 nodes as well. Each node is a regular MySQL Server instance (for example, Percona Server). You can convert an existing MySQL Server instance to a node and run the cluster using this node as a base. You can also detach any node from the cluster and use it as a regular MySQL Server instance.

![cluster-diagram1](/docs/images/percona-xtradb/cluster-diagram1.png)

## Benefits

- When you execute a query, it is executed locally on the node. All data is available locally, no need for remote access.
- No central management. You can loose any node at any point of time, and the cluster will continue to function without any data loss.
- Good solution for scaling a read workload. You can put read queries to any of the nodes.

## Drawbacks

- Overhead of provisioning new node. When you add a new node, it has to copy the full data set from one of existing nodes. If it is 100GB, it copies 100GB.
- This can’t be used as an effective write scaling solution. There might be some improvements in write throughput when you run write traffic to 2 nodes versus all traffic to 1 node, but you can’t expect a lot. All writes still have to go on all nodes.
- You have several duplicates of the data, for 3 nodes you have 3 duplicates.

## Components

Percona XtraDB Cluster is based on Percona Server running with the XtraDB storage engine. It uses the Galera library, which is an implementation of the write set replication (wsrep) API developed by Codership Oy. The default and recommended data transfer method is via Percona XtraBackup.
