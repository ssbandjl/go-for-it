https://docs.mongodb.com/manual/core/backups/



- [Administration](https://docs.mongodb.com/manual/administration/) > 
- MongoDB Backup Methods

# MongoDB Backup Methods 备份方法

On this page

- [Back Up with Atlas](https://docs.mongodb.com/manual/core/backups/#back-up-with-atlas)
- [Back Up with MongoDB Cloud Manager or Ops Manager](https://docs.mongodb.com/manual/core/backups/#back-up-with-mms-or-ops-manager)
- [Back Up by Copying Underlying Data Files](https://docs.mongodb.com/manual/core/backups/#back-up-by-copying-underlying-data-files)
- [Back Up with `mongodump`](https://docs.mongodb.com/manual/core/backups/#back-up-with-mongodump)

When deploying MongoDB in production, you should have a strategy for capturing and restoring backups in the case of data loss events.

## Back Up with Atlas

MongoDB Atlas, the hosted MongoDB service option in the cloud, offers two fully-managed methods for backups:

1. Cloud Backups

   , which utilize the native snapshot functionality of the deployment’s cloud service provider to offer robust backup options. Cloud Backups provide:

   - [On-demand snapshots](https://docs.atlas.mongodb.com/backup/cloud-backup/overview/#on-demand-snapshots), which allow you to trigger an immediate snapshot of your deployment at a given point in time.
   - [Continuous Cloud Backups](https://docs.atlas.mongodb.com/backup/cloud-backup/overview/#continuous-cloud-backups), which allow you to schedule recurring backups for your deployment.

2. [Legacy Backups](https://docs.atlas.mongodb.com/backup/legacy-backup/overview/) *(Deprecated)*, which take incremental backups of data in your deployment. 已弃用, 做增量备份

## Back Up with MongoDB Cloud Manager or Ops Manager

MongoDB Cloud Manager is a hosted back up 托管备份, monitoring, and automation service for MongoDB. [MongoDB Cloud Manager](https://www.mongodb.com/cloud/cloud-manager/?tck=docs_server) supports backing up and restoring MongoDB [replica sets](https://docs.mongodb.com/manual/reference/glossary/#term-replica-set) and [sharded clusters](https://docs.mongodb.com/manual/reference/glossary/#term-sharded-cluster) from a graphical user interface.图形化界面, 支持副本集和分片集群的备份和恢复



### MongoDB Cloud Manager

The [MongoDB Cloud Manager](https://www.mongodb.com/cloud/cloud-manager/?tck=docs_server) supports the backing up and restoring of MongoDB deployments.

MongoDB Cloud Manager continually backs up MongoDB [replica sets](https://docs.mongodb.com/manual/reference/glossary/#term-replica-set) and [sharded clusters](https://docs.mongodb.com/manual/reference/glossary/#term-sharded-cluster) by reading the [oplog](https://docs.mongodb.com/manual/reference/glossary/#term-oplog) data from your MongoDB deployment 基于从oplog中读取数据做持续的备份. MongoDB Cloud Manager creates snapshots of your data at set intervals, and can also offer point-in-time recovery of MongoDB replica sets and sharded clusters. 根据设置的时间间隔创建数据快照, 还提供基于时间点的恢复功能

TIP

Sharded cluster snapshots are difficult to achieve with other MongoDB backup methods. 其他备份方式, 很难实现分片集群快照

To get started with MongoDB Cloud Manager Backup, sign up for [MongoDB Cloud Manager](https://www.mongodb.com/cloud/cloud-manager/?tck=docs_server). For documentation on MongoDB Cloud Manager, see the [MongoDB Cloud Manager documentation](https://docs.cloudmanager.mongodb.com/).



### Ops Manager 运维管家

With Ops Manager, MongoDB subscribers can install and run the same core software that powers [MongoDB Cloud Manager](https://docs.mongodb.com/manual/core/backups/#backup-with-mms) on their own infrastructure. Ops Manager is an on-premise solution that has similar functionality to MongoDB Cloud Manager and is available with Enterprise Advanced subscriptions. 内部解决方案, 支持企业订阅的高级功能

For more information about Ops Manager, see the [MongoDB Enterprise Advanced](https://www.mongodb.com/products/mongodb-enterprise-advanced?tck=docs_server) page and the [Ops Manager Manual](https://docs.opsmanager.mongodb.com/current/).



## Back Up by Copying Underlying Data Files 基于拷贝底层文件备份

Considerations for Encrypted Storage Engines using AES256-GCM

For [encrypted storage engines](https://docs.mongodb.com/manual/core/security-encryption-at-rest/#encrypted-storage-engine) that use `AES256-GCM` encryption mode, `AES256-GCM` requires that every process use a unique counter block value with the key.

For [encrypted storage engine](https://docs.mongodb.com/manual/core/security-encryption-at-rest/#encrypted-storage-engine) configured with `AES256-GCM` cipher:

- - Restoring from Hot Backup

    Starting in 4.2, if you restore from files taken via “hot” backup (i.e. the [`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) is running), MongoDB can detect “dirty” keys on startup and automatically rollover the database key to avoid IV (Initialization Vector) reuse.

- - Restoring from Cold Backup

    However, if you restore from files taken via “cold” backup (i.e. the [`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) is not running), MongoDB cannot detect “dirty” keys on startup, and reuse of IV voids confidentiality and integrity guarantees.Starting in 4.2, to avoid the reuse of the keys after restoring from a cold filesystem snapshot, MongoDB adds a new command-line option [`--eseDatabaseKeyRollover`](https://docs.mongodb.com/manual/reference/program/mongod/#cmdoption-mongod-esedatabasekeyrollover). When started with the [`--eseDatabaseKeyRollover`](https://docs.mongodb.com/manual/reference/program/mongod/#cmdoption-mongod-esedatabasekeyrollover) option, the [`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) instance rolls over the database keys configured with `AES256-GCM` cipher and exits.

TIP

- In general, if using filesystem based backups for MongoDB Enterprise 4.2+, use the “hot” backup feature, if possible.
- For MongoDB Enterprise versions 4.0 and earlier, if you use `AES256-GCM` encryption mode, do **not** make copies of your data files or restore from filesystem snapshots (“hot” or “cold”).

### Back Up with Filesystem Snapshots

You can create a backup of a MongoDB deployment by making a copy of MongoDB’s underlying data files.

If the volume where MongoDB stores its data files supports point-in-time snapshots, you can use these snapshots to create backups of a MongoDB system at an exact moment in time. File system snapshots are an operating system volume manager feature, and are not specific to MongoDB. With file system snapshots, the operating system takes a snapshot of the volume to use as a baseline for data backup. The mechanics of snapshots depend on the underlying storage system. For example, on Linux, the Logical Volume Manager (LVM) can create snapshots. Similarly, Amazon’s EBS storage system for EC2 supports snapshots.

To get a correct snapshot of a running [`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) process, you must have journaling enabled and the journal must reside on the same logical volume as the other MongoDB data files. Without journaling enabled, there is no guarantee that the snapshot will be consistent or valid.

To get a consistent snapshot of a [sharded cluster](https://docs.mongodb.com/manual/reference/glossary/#term-sharded-cluster), you must disable the balancer and capture a snapshot from every shard as well as a config server at approximately the same moment in time.

For more information, see the [Back Up and Restore with Filesystem Snapshots](https://docs.mongodb.com/manual/tutorial/backup-with-filesystem-snapshots/) and [Back Up a Sharded Cluster with File System Snapshots](https://docs.mongodb.com/manual/tutorial/backup-sharded-cluster-with-filesystem-snapshots/) for complete instructions on using LVM to create snapshots.

### Back Up with `cp` or `rsync`

If your storage system does not support snapshots, you can copy the files directly using `cp`, `rsync`, or a similar tool. Since copying multiple files is not an atomic operation, you must stop all writes to the [`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) before copying the files. Otherwise, you will copy the files in an invalid state.

Backups produced by copying the underlying data do not support point in time recovery for [replica sets](https://docs.mongodb.com/manual/reference/glossary/#term-replica-set) and are difficult to manage for larger sharded clusters. Additionally, these backups are larger because they include the indexes and duplicate underlying storage padding and fragmentation. [`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump), by contrast, creates smaller backups.



## Back Up with `mongodump` 使用mongodump备份

[`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) reads data from a MongoDB database and creates high fidelity BSON files which the [`mongorestore`](https://docs.mongodb.com/database-tools/mongorestore/#mongodb-binary-bin.mongorestore) tool can use to populate a MongoDB database. 创建高保真备份的BSON备份文件 [`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) and [`mongorestore`](https://docs.mongodb.com/database-tools/mongorestore/#mongodb-binary-bin.mongorestore) are simple and efficient tools for backing up and restoring small MongoDB deployments, but are not ideal for capturing backups of larger systems. 仅适合小型实例备份

[`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) and [`mongorestore`](https://docs.mongodb.com/database-tools/mongorestore/#mongodb-binary-bin.mongorestore) operate against a running [`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) process, and can manipulate the underlying data files directly. By default, [`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) does not capture the contents of the [local database](https://docs.mongodb.com/manual/reference/local-database/).

[`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) only captures the documents in the database. The resulting backup is space efficient, but [`mongorestore`](https://docs.mongodb.com/database-tools/mongorestore/#mongodb-binary-bin.mongorestore) or [`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) must rebuild the indexes after restoring data. 备份是空间高效的, 但是恢复数据后需要重建索引

When connected to a MongoDB instance, [`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) can adversely affect [`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) performance. If your data is larger than system memory, the queries will push the working set out of memory, causing page faults. 当连接到一个MongoDB实例时，mongodump会对mongod的性能产生负面影响。如果数据大于系统内存，查询将把工作集挤出内存，导致页面错误

Applications can continue to modify data while [`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) captures the output. For replica sets, [`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) provides the [`--oplog`](https://docs.mongodb.com/database-tools/mongodump/#std-option-mongodump.--oplog) option to include in its output [oplog](https://docs.mongodb.com/manual/reference/glossary/#term-oplog) entries that occur during the [`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) operation. This allows the corresponding [`mongorestore`](https://docs.mongodb.com/database-tools/mongorestore/#mongodb-binary-bin.mongorestore) operation to replay the captured oplog. To restore a backup created with [`--oplog`](https://docs.mongodb.com/database-tools/mongodump/#std-option-mongodump.--oplog), use [`mongorestore`](https://docs.mongodb.com/database-tools/mongorestore/#mongodb-binary-bin.mongorestore) with the [`--oplogReplay`](https://docs.mongodb.com/database-tools/mongorestore/#std-option-mongorestore.--oplogReplay) option. 执行备份时, 应用可以继续修改数据, 对于副本集, mongodump提供了--oplog选项来包含整个备份期间修改的oplog条目, 这就允许mongorestore在恢复的时候使用--oplogReplay选项恢复该数据

However, for replica sets, consider [MongoDB Cloud Manager](https://docs.mongodb.com/manual/core/backups/#backup-with-mms) or [Ops Manager](https://docs.mongodb.com/manual/core/backups/#backup-with-mms-onprem).

NOTE

[`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) and [`mongorestore`](https://docs.mongodb.com/database-tools/mongorestore/#mongodb-binary-bin.mongorestore) **cannot** be part of a backup strategy for 4.2+ sharded clusters that have sharded transactions in progress, as backups created with [`mongodump`](https://docs.mongodb.com/database-tools/mongodump/#mongodb-binary-bin.mongodump) *do not maintain* the atomicity guarantees of transactions across shards. 不支持MongoDB 4.2+, 分片集群备份, 因为mongodump无法维护跨分片事务的原子性保证

For 4.2+ sharded clusters with in-progress sharded transactions, use one of the following coordinated backup and restore processes which *do maintain* the atomicity guarantees of transactions across shards:

- [MongoDB Atlas](https://www.mongodb.com/cloud/atlas?tck=docs_server),
- [MongoDB Cloud Manager](https://www.mongodb.com/cloud/cloud-manager?tck=docs_server), or
- [MongoDB Ops Manager](https://www.mongodb.com/products/ops-manager?tck=docs_server).

See [Back Up and Restore with MongoDB Tools](https://docs.mongodb.com/manual/tutorial/backup-and-restore-tools/) and [Back Up a Sharded Cluster with Database Dumps](https://docs.mongodb.com/manual/tutorial/backup-sharded-cluster-with-database-dumps/) for more information.

←  [Workload Isolation in MongoDB Deployments](https://docs.mongodb.com/manual/core/workload-isolation/)[Back Up and Restore with Filesystem Snapshots](https://docs.mongodb.com/manual/tutorial/backup-with-filesystem-snapshots/) →

© MongoDB, Inc 2008-present. MongoDB, Mongo, and the leaf logo are registered trademarks of MongoDB, Inc.