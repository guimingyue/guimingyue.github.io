---
layout: post
title: Debug MySQL in CLion
category: MySQL
---

基于 MySQL 源码搭建本地 MySQL 环境遇到的一些问题记录

1. boost 文件找不到
```
/home/admin/mysql-5.7.23/boost/boost_1_59_0/boost/asio/impl/src.cpp:25:10: fatal error: 'boost/asio/impl/src.hpp' file not found
```
检查 CMakeList.txt 文件是不是被破坏了，第一次用 CLion 导入 MySQL 源码后，CLion 生成了一个 CMakeList.txt 文件，所以在 build 时一直报这个错，
后来重新解压了 MySQL 源码到另一个目录才发现问题的原因。使用 CLion 导入 MySQL 源码时直接 Open，别 Import Project。

2. 

```
[ERROR] Can't find error-message file '/home/admin/mysql_data/share/errmsg.sys'. Check error-message file location and 'lc-messages-dir' configuration directive.
```
mysqld 启动时 `--basedir` 参数错误，这个参数的值应该是 MySQL 安装的目录，通过源码启动时，应该是源码所在的目录。
