---
layout: post
title: SimpleDB 架构简介
category: Database
---

## SimpleDB 简介
[SimpleDB](http://www.cs.bc.edu/~sciore/simpledb/)

## SimpleDB 的组成

### File Manager
File Manager（FileMgr）管理着 SimpleDB 的一个数据库运行过程中打开的文件。File Manager（FileMgr）在读和写时的参数是 BlockId，它持有该 Block 所在的文件和 Block 的 ID，根据 Block 的 ID 和 block size （启动时指定）确定读写的文件位置。

### Log Manager
Log Manager（LogMgr）对数据库运行过程中的日志进行读写，日志写入时只会在文件的最后进行追加写。当 SimpleDB 启动时，如果日志文件为空，则会向日志文件（simpledb.log）写入第一个 Block，这个 Block 仅有从文件开始处写入的 block size，如果启动 SimpleDB 时，日志文件非空，则会读取最后一个 block 的数据到 属性 `logpage` 中，并且属性 `currentblk` 最后一个 block。

### Buffer Manager
 Buffer Manager（BufferMgr）使用来管理 Buffer Pool 的。SimpleDB 的 Buffer Pool 由一个数组组成，每个数据元素是一个 `Buffer` 对象，它记录了其管理数据的数据和对应的磁盘上的 block，以及相关的事务号（如果有）和 lsn（如果有）。

### Metadata Manager
元数据管理（MetadataMgr）是用来对数据库运行过程中的元数据进行管理的，元数据包括表元数据，视图元数据，统计信息元数据，索引元数据。 

 ### 事务

 SimpleDB 的事务（Transaction）的并发控制实现的加锁粒度是是 Block 级别的。通过`simpledb.server.SimpleDB#newTx`可以创建一个事务。

