---
layout: post
title: MySQL 的 Prepared Statement
category: MySQL
---

## Prepared Statement

Prepared Statement 也称为预编译，表示预先编译用户的 SQL 语句，SQL 语句中的变量使用占位符表示，在执行时，只需要传递占位符所需的变量，就能再数据库中执行。对于单词 SQL 语句的执行，其与普通的 Statement 没有区别，但是如果要执行大量的 SQL 语句，而这些 SQL 语句只是在部分参数上不一样，那就可以使用 Prepared Statement 来提升执行性能，因为在  Prepared Statement 执行过程中，对 SQL 语句的解析只执行了一次。

在 MySQL 中，Prepared Statement 分为客户端执行方式和服务端执行方式。客户端的 Prepared Statement 是对服务端的 Prepared Statement 的一种模拟，在发送到服务端执行时，会将占位符替换为真实的变量参数，其执行方式与普通的 Statement 没有任何区别。而服务端的 Prepared Statement 则由 PREPARE 和 EXECUTE 两部分组成，如下所示。

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

## Proxy 对 Prepared Statement 的处理

## 总结


未完待续...

## Reference 
[Prepared Statements](https://dev.mysql.com/doc/refman/5.7/en/sql-prepared-statements.html)
[What's the difference between cachePrepStmts and useServerPrepStmts in MySQL JDBC Driver](https://stackoverflow.com/questions/32286518/whats-the-difference-between-cacheprepstmts-and-useserverprepstmts-in-mysql-jdb/32645365#32645365)

