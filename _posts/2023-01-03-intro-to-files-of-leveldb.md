---
layout: post
title: LevelDB 文件简介
category: Database
---

LevelDB 是 Google 开源的一个用 C++ 语言开发的键值存储库，它提供了简单的 KV 的接口，通过这些接口能够实现高效的数据的存取。RocksDB 就是基于 LevelDB 开发的键值数据库，目前已经被广泛应用于各种新一代的分布式数据库中。

__系列文章__

[LevelDB 架构简介](/database/2023-01-03-intro-to-files-of-leveldb.md)

## LevelDB 文件分类

LevelDB 在运行过程中涉及到的文件比较多，主要可以分为几类：
* WAL 文件
* 元数据文件
* SSTable 文件
* 程序运行文件

## WAL 文件
WAL 文件记录着一系列 LevelDB 最近的更新（`Put`，`Delete`）,其数据与内存中的 Memtable 一致。当其记录的数据大小达到一个阈值（默认 4MB）时，Memtable 中的内容就会被转换为 Level 0 的 SSTable 文件。

## 元数据文件
元数据文件用于描述 LevelDB 数据分布的文件


## SSTable 文件

## 程序运行文件


## 总结



## Reference
* [LevelDB Implementation notes](http://github.com/google/leveldb/blob/main/docs/impl.md)

