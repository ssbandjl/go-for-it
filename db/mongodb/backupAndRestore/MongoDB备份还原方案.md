# MongoDB备份还原方案

## Overview

When deploying MongoDB in production, you should have a strategy for capturing and restoring backups in the case of data loss events.

## MongoDB Backup and Restore Methods

`MongoDB`官方给出了四种`bur`方案：

- Back Up with Atlas

MongoDB Atlas, the official MongoDB cloud service

- Back Up with MongoDB Cloud Manager or Ops Manager

MongoDB Cloud Manager is a hosted back up, monitoring, and automation service for MongoDB. MongoDB Cloud Manager supports backing up and restoring MongoDB replica sets and sharded clusters from a graphical user interface.

- [通过拷贝底层数据文件来备份(快照)](https://docs.mongodb.com/manual/tutorial/backup-with-filesystem-snapshots/)

You can create a backup of a MongoDB deployment by making a copy of MongoDB’s underlying data files.

These filesystem snapshots, or “block-level” backup methods, use system level tools to create copies of the device that holds MongoDB’s data files. These methods complete quickly and work reliably, but require additional system configuration outside of MongoDB.

- [使用MongoDB工具mongodump来备份](https://docs.mongodb.com/manual/tutorial/backup-and-restore-tools/)

mongodump reads data from a MongoDB database and creates high fidelity 高保真的 BSON files which the mongorestore tool can use to populate(填充/恢复) a MongoDB database. `mongodump` and `mongorestore` are simple and efficient tools for backing up and restoring small(小型) MongoDB deployments, but are not ideal for capturing backups of larger systems.(For resilient and non-disruptive backups, use a file system or block-level disk snapshot function 建议使用文件系统或块级磁盘快照方法, 来实现弹性和非破坏性备份)

When connected to a MongoDB instance, `mongodump` can adversely affect mongod performance 对性能产生负面影响. If your data is larger than system memory, the queries will push the working set out of memory, causing page faults. 如果数据比系统内存大, 查询操作会把工作集从内存中挤出来, 导致页面错误.

Use these tools for backups if other backup methods, such as MongoDB Cloud Manager or file system snapshots are unavailable.

> 注意
>
> `mongodump` and `mongorestore` cannot be part of a backup strategy for 4.2+ sharded clusters that have sharded transactions in progress as these tools cannot guarantee a atomicity guarantees of data across the shards. `mongodump` 和 `mongorestore`工具不能用来作为MongoDB4.2+的分片集群备份策略, 因为工具无法保证跨分片数据的原子性.
>
> For 4.2+ sharded clusters with in-progress sharded transactions 正在进行的分片事务, for coordinated backup and restore processes that maintain the atomicity guarantees of transactions across shards, see: 有关保持跨分片事务原子性的协调备份和恢复进程，请参考:
>
> - [MongoDB Atlas](https://www.mongodb.com/cloud/atlas?jmp=docs),
> - [MongoDB Cloud Manager](https://www.mongodb.com/cloud/cloud-manager?jmp=docs), or
> - [MongoDB Ops Manager](https://www.mongodb.com/products/ops-manager?jmp=docs).

> For MongoDB 4.0 and earlier deployments, refer to the corresponding versions of the manual. For example:
>
> - https://docs.mongodb.com/v4.0
> - https://docs.mongodb.com/v3.6
> - https://docs.mongodb.com/v3.4

## 结论

- 1、`MongoDB Atlas`与`MongoDB Cloud Manager or Ops Manager`需要特定使用场景，不通用，这里舍弃
- 2、`Underlying Data Files`是官方推荐的`bur`方案，适用于大数据量的`MongoDB`备份与还原，但是需要额外的系统配置，比如：[LVM](https://docs.mongodb.com/manual/reference/glossary/#term-lvm)
- 3、`mongodump`简单有效，适用于`MongoDB`数据量较小的情况；另外这种方式会影响`mongod`性能，而且数据量越大影响越大；`mongodump`不能用于`Sharded Cluster`，可用于`Replica-Set`([Difference between Sharding And Replication on MongoDB](https://dba.stackexchange.com/questions/52632/difference-between-sharding-and-replication-on-mongodb))

这里由于我们的`MongoDB`数据量较小，最多100M，所以采用`mongodump`方案进行备份与还原

## 使用mongodump和mongorestore

### mongodump

> > The mongodump utility backs up data by connecting to a running mongod.通过连接到实例来备份

> > The utility can create a backup for an entire server, database or collection, or can use a query to backup just part of a collection.可以整体备份, 也可以指定一部分集合进行备份

```
全量备份:
#mongodump --host=mongodb1.example.net --port=3017 --username=user --password="pass" --out=/opt/backup/mongodump-2013-10-24
```

### mongorestore

> > The mongorestore utility restores a binary backup created by mongodump. By default, mongorestore looks for a database backup in the dump/ directory. 从mongodump工具创建的二进制备份文件中恢复, 默认查找目录为:dump/

```
#mongorestore --host=mongodb1.example.net --port=3017 --username=user  --authenticationDatabase=admin /opt/backup/mongodump-2013-10-24
```

## 参考

- [Back Up and Restore with MongoDB Tools](https://docs.mongodb.com/manual/tutorial/backup-and-restore-tools/)
- [MongoDB Backup Methods](https://docs.mongodb.com/manual/core/backups/)
- [Back Up and Restore with Filesystem Snapshots](https://docs.mongodb.com/manual/tutorial/backup-with-filesystem-snapshots/)
- [difference-between-sharding-and-replication-on-mongodb](https://dba.stackexchange.com/questions/52632/difference-between-sharding-and-replication-on-mongodb)