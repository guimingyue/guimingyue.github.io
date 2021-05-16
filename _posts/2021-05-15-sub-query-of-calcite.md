---
layout: post
title: Calcite 中的子查询优化
category: Calcite
---

## 子查询的概念
子查询在 project，filter 等地方，分为相关和非相关子查询

## 非相关子查询的转换
* in 子查询，转换成 inner join。

SQL 语句和逻辑执行计划

```sql
select empno, gender, name from EMPS
 where gender = 'F' and empno > 110 and DEPTNO in (select DEPTNO from DEPTS where NAME='Marketing')

 LogicalProject(EMPNO=[$0], GENDER=[$3], NAME=[$1])
  LogicalFilter(condition=[AND(=($3, 'F'), >($0, 110), IN($2, {
LogicalProject(DEPTNO=[$0])
  LogicalFilter(condition=[=($1, 'Marketing')])
    LogicalTableScan(table=[[SALES, DEPTS]])
}))])
    LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
      LogicalTableScan(table=[[SALES, EMPS]])

```

转换后的逻辑执行计划以及对应的 SQL 语句。

```sql

SELECT `t`.`EMPNO`, `t`.`GENDER`, `t`.`NAME`
FROM (SELECT `EMPNO`, `NAME`, `DEPTNO`, `GENDER`
FROM `SALES`.`EMPS`) AS `t`
INNER JOIN (SELECT `DEPTNO`
FROM `SALES`.`DEPTS`
WHERE `NAME` = 'Marketing'
GROUP BY `DEPTNO`) AS `t2` ON `t`.`DEPTNO` = `t2`.`DEPTNO`
WHERE `t`.`GENDER` = 'F' AND `t`.`EMPNO` > 110

LogicalProject(EMPNO=[$0], GENDER=[$3], NAME=[$1])
  LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
    LogicalFilter(condition=[AND(=($3, 'F'), >($0, 110))])
      LogicalJoin(condition=[=($2, $4)], joinType=[inner])
        LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
          LogicalTableScan(table=[[SALES, EMPS]])
        LogicalAggregate(group=[{0}])
          LogicalProject(DEPTNO=[$0])
            LogicalFilter(condition=[=($1, 'Marketing')])
              LogicalTableScan(table=[[SALES, DEPTS]])


```

* agg 子查询，转换为 inner join。

SQL 语句和逻辑执行计划
```sql
select empno, gender, name from EMPS
 where gender = 'F' and empno > 110 and DEPTNO = (select max(DEPTNO) from DEPTS where NAME='Marketing')

LogicalProject(EMPNO=[$0], GENDER=[$3], NAME=[$1])
  LogicalFilter(condition=[AND(=($3, 'F'), >($0, 110), =($2, $SCALAR_QUERY({
LogicalAggregate(group=[{}], EXPR$0=[MAX($0)])
  LogicalProject(DEPTNO=[$0])
    LogicalFilter(condition=[=($1, 'Marketing')])
      LogicalTableScan(table=[[SALES, DEPTS]])
})))])
    LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
      LogicalTableScan(table=[[SALES, EMPS]])

```
转换后的逻辑执行计划以及对应的 SQL 语句。

```sql
SELECT `t`.`EMPNO`, `t`.`GENDER`, `t`.`NAME`
FROM (SELECT `EMPNO`, `NAME`, `DEPTNO`, `GENDER`
FROM `SALES`.`EMPS`) AS `t`
LEFT JOIN (SELECT MAX(`DEPTNO`) AS `EXPR$0`
FROM `SALES`.`DEPTS`
WHERE `NAME` = 'Marketing') AS `t2` ON TRUE
WHERE `t`.`GENDER` = 'F' AND `t`.`EMPNO` > 110 AND `t`.`DEPTNO` = `t2`.`EXPR$0`

LogicalProject(EMPNO=[$0], GENDER=[$3], NAME=[$1])
  LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
    LogicalFilter(condition=[AND(=($3, 'F'), >($0, 110), =($2, $4))])
      LogicalJoin(condition=[true], joinType=[left])
        LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
          LogicalTableScan(table=[[SALES, EMPS]])
        LogicalAggregate(group=[{}], EXPR$0=[MAX($0)])
          LogicalProject(DEPTNO=[$0])
            LogicalFilter(condition=[=($1, 'Marketing')])
              LogicalTableScan(table=[[SALES, DEPTS]])


```
* not in 非关联子查询 anti semi join

```sql

select empno, gender, name from EMPS
 where gender = 'F' and empno > 110 and DEPTNO NOT IN (select DEPTNO from DEPTS where NAME='Marketing')

LogicalProject(EMPNO=[$0], GENDER=[$3], NAME=[$1])
  LogicalFilter(condition=[AND(=($3, 'F'), >($0, 110), NOT(IN($2, {
LogicalProject(DEPTNO=[$0])
  LogicalFilter(condition=[=($1, 'Marketing')])
    LogicalTableScan(table=[[SALES, DEPTS]])
})))])
    LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
      LogicalTableScan(table=[[SALES, EMPS]])

```
转换后

```sql

SELECT `t`.`EMPNO`, `t`.`GENDER`, `t`.`NAME`
FROM (SELECT `EMPNO`, `NAME`, `DEPTNO`, `GENDER`
FROM `SALES`.`EMPS`) AS `t`,
(SELECT COUNT(*) AS `c`, COUNT(`DEPTNO`) AS `ck`
FROM `SALES`.`DEPTS`
WHERE `NAME` = 'Marketing') AS `t2`
LEFT JOIN (SELECT `DEPTNO`, TRUE AS `i`
FROM `SALES`.`DEPTS`
WHERE `NAME` = 'Marketing'
GROUP BY `DEPTNO`, TRUE) AS `t5` ON `t`.`DEPTNO` = `t5`.`DEPTNO`
WHERE `t`.`GENDER` = 'F' AND `t`.`EMPNO` > 110 AND (`t2`.`c` = 0 OR `t5`.`i` IS NULL AND `t2`.`ck` >= `t2`.`c` AND `t`.`DEPTNO` IS NOT NULL)

LogicalProject(EMPNO=[$0], GENDER=[$3], NAME=[$1])
  LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
    LogicalFilter(condition=[AND(=($3, 'F'), >($0, 110), OR(=($4, 0), AND(IS NULL($7), >=($5, $4), IS NOT NULL($2))))])
      LogicalJoin(condition=[=($2, $6)], joinType=[left])
        LogicalJoin(condition=[true], joinType=[inner])
          LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
            LogicalTableScan(table=[[SALES, EMPS]])
          LogicalAggregate(group=[{}], c=[COUNT()], ck=[COUNT($0)])
            LogicalProject(DEPTNO=[$0])
              LogicalFilter(condition=[=($1, 'Marketing')])
                LogicalTableScan(table=[[SALES, DEPTS]])
        LogicalAggregate(group=[{0, 1}])
          LogicalProject(DEPTNO=[$0], i=[true])
            LogicalFilter(condition=[=($1, 'Marketing')])
              LogicalTableScan(table=[[SALES, DEPTS]])



```

* `< All` 子查询

```sql
select empno, gender, name from EMPS
 where gender = 'F' and empno > 110 and DEPTNO < all (select max(DEPTNO) from DEPTS where NAME='Marketing')

LogicalProject(EMPNO=[$0], GENDER=[$3], NAME=[$1])
  LogicalFilter(condition=[AND(=($3, 'F'), >($0, 110), NOT(>= SOME($2, {
LogicalAggregate(group=[{}], EXPR$0=[MAX($0)])
  LogicalProject(DEPTNO=[$0])
    LogicalFilter(condition=[=($1, 'Marketing')])
      LogicalTableScan(table=[[SALES, DEPTS]])
})))])
    LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
      LogicalTableScan(table=[[SALES, EMPS]])


```

```sql
SELECT `t`.`EMPNO`, `t`.`GENDER`, `t`.`NAME`
FROM (SELECT `EMPNO`, `NAME`, `DEPTNO`, `GENDER`
FROM `SALES`.`EMPS`) AS `t`,
(SELECT MIN(`EXPR$0`) AS `m`, COUNT(*) AS `c`, COUNT(`EXPR$0`) AS `d`
FROM (SELECT MAX(`DEPTNO`) AS `EXPR$0`
FROM `SALES`.`DEPTS`
WHERE `NAME` = 'Marketing') AS `t2`) AS `t3`
WHERE `t`.`GENDER` = 'F' AND `t`.`EMPNO` > 110 AND (`t3`.`c` = 0 OR `t`.`DEPTNO` < `t3`.`m` AND `t3`.`c` <> 0 AND (`t`.`DEPTNO` >= `t3`.`m` OR `t3`.`c` > `t3`.`d`) IS NOT TRUE)

LogicalProject(EMPNO=[$0], GENDER=[$3], NAME=[$1])
  LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
    LogicalFilter(condition=[AND(=($3, 'F'), >($0, 110), OR(=($5, 0), AND(<($2, $4), <>($5, 0), IS NOT TRUE(OR(>=($2, $4), >($5, $6))))))])
      LogicalJoin(condition=[true], joinType=[inner])
        LogicalProject(EMPNO=[$0], NAME=[$1], DEPTNO=[$2], GENDER=[$3])
          LogicalTableScan(table=[[SALES, EMPS]])
        LogicalAggregate(group=[{}], m=[MIN($0)], c=[COUNT()], d=[COUNT($0)])
          LogicalAggregate(group=[{}], EXPR$0=[MAX($0)])
            LogicalProject(DEPTNO=[$0])
              LogicalFilter(condition=[=($1, 'Marketing')])
                LogicalTableScan(table=[[SALES, DEPTS]])

```


* `> any` 子查询
* `!= any` 子查询
* `= all` 子查询
* `exists` 子查询

```sql
select empno, gender, name from EMPS
 where gender = 'F' and empno > 110 and exists (select * from DEPTS where NAME='Marketing')

LogicalProject(EMPNO=[$0], GENDER=[$2], NAME=[$1])
  LogicalFilter(condition=[AND(=($2, 'F'), >($0, 110), EXISTS({
LogicalFilter(condition=[=($1, 'Marketing')])
  LogicalTableScan(table=[[SALES, DEPTS]])
}))])
    LogicalProject(EMPNO=[$0], NAME=[$1], GENDER=[$3])
      LogicalTableScan(table=[[SALES, EMPS]])

```
转换后

```sql
SELECT `t`.`EMPNO`, `t`.`GENDER`, `t`.`NAME`
FROM (SELECT `EMPNO`, `NAME`, `GENDER`
FROM `SALES`.`EMPS`) AS `t`,
(SELECT TRUE AS `i`
FROM `SALES`.`DEPTS`
WHERE `NAME` = 'Marketing'
GROUP BY TRUE) AS `t2`
WHERE `t`.`GENDER` = 'F' AND `t`.`EMPNO` > 110

LogicalProject(EMPNO=[$0], GENDER=[$2], NAME=[$1])
  LogicalProject(EMPNO=[$0], NAME=[$1], GENDER=[$2])
    LogicalFilter(condition=[AND(=($2, 'F'), >($0, 110))])
      LogicalJoin(condition=[true], joinType=[inner])
        LogicalProject(EMPNO=[$0], NAME=[$1], GENDER=[$3])
          LogicalTableScan(table=[[SALES, EMPS]])
        LogicalAggregate(group=[{0}])
          LogicalProject(i=[true])
            LogicalFilter(condition=[=($1, 'Marketing')])
              LogicalTableScan(table=[[SALES, DEPTS]])

```

* `>/>=/</<=/=/!=`子查询

非相关子查询默认会被转换成 semi join 来执行。有些特殊的比如则需要做一些逻辑上的转换以获得更好的查询性能。

## 相关子查询的转换


## Reference
* https://docs.pingcap.com/zh/tidb/stable/subquery-optimization
* https://www.bookstack.cn/read/TiDB-4.0/explain-subqueries.md
