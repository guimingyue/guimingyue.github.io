---
layout: post
title: MySQL 的 Prepared Statement
category: MySQL
---

## Prepared Statement

Prepared Statement 也称为预编译，表示预先编译用户的 SQL 语句，SQL 语句中的变量使用占位符表示，在执行时，只需要传递占位符所需的变量，就能再数据库中执行。对于单词 SQL 语句的执行，其与普通的 Statement 没有区别，但是如果要执行大量的 SQL 语句，而这些 SQL 语句只是在部分参数上不一样，那就可以使用 Prepared Statement 来提升执行性能，因为在  Prepared Statement 执行过程中，对 SQL 语句的解析与优化只执行了一次。下文除非特殊说明，都是只带 MySQL 支持的 Prepared Statement。

在 MySQL 中，Prepared Statement 分为客户端执行方式和服务端执行方式。客户端的 Prepared Statement 是对服务端的 Prepared Statement 的一种模拟，只会在将 SQL 发送到服务端执行时向服务端发送一次请求，也就是在这个时候会将占位符替换为真实的变量参数，其执行方式与普通的 Statement 没有任何区别。而服务端的 Prepared Statement 则由 PREPARE 和 EXECUTE 两部分组成。PREPARE 命令会先将 SQL 发送给服务端做 PREPARE 处理，返回给客户端此次 Prepared Statement 对应 stmt-id，然后执行 EXECUTE 命令，将占位符对应的参数发送给服务端执行，如下所示。

```sql
mysql> PREPARE stmt1 FROM 'SELECT SQRT(POW(?,2) + POW(?,2)) AS hypotenuse';
mysql> SET @a = 3;
mysql> SET @b = 4;
mysql> EXECUTE stmt1 USING @a, @b;
+------------+
| hypotenuse |
+------------+
|          5 |
+------------+
```

## proxy 对 Prepared Statement 的支持
在 proxy 中，由于对外提供的是实现了 MySQL 协议的代理，所以如果要提供 Prepared Statement，那么也需要支持 prepare 协议的 PREPARE 和 EXECUTE 两个命令。

### PREPARED

服务端接收到 PREPARED 命令后，解析 SQL，从解析的结果中获取参数的个数，然后解析出来的 SQL 进行优化得到物理执行计划。生成此次 PREPARED 请求生成的 stmt-id，并返回给客户端。

### EXECUTE
EXECUTE 命令会将待执行的参数以及 PREPARED 阶段返回的 stmt-id 发送给服务端，服务端通过 stmt-id 找到 PREPARED 节点生成的物理执行计划。拿到物理执行计划后，会根据此次客户端发送的参数执行该物理执行计划。

## 总结


未完待续...

## Reference 
[Prepared Statements](https://dev.mysql.com/doc/refman/5.7/en/sql-prepared-statements.html)
[What's the difference between cachePrepStmts and useServerPrepStmts in MySQL JDBC Driver](https://stackoverflow.com/questions/32286518/whats-the-difference-between-cacheprepstmts-and-useserverprepstmts-in-mysql-jdb/32645365#32645365)
[小米 Gaea prepare的设计与实现](https://github.com/XiaoMi/Gaea/blob/master/docs/prepare.md)

