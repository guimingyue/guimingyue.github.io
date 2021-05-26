---
layout: post
title: Calcite 中的子查询优化
category: Calcite
---

## 子查询的概念
子查询出现在 SQL 语句的 project，filter 等地方，分为相关和非相关子查询，本文通过列举部分场景，测试了 Calcite 的子查询转换规则，以探究 Calcite 在子查询处理的处理逻辑以及转换规则。

## 非相关子查询
针对非相关子查询，子查询不依赖外层的变量，所以可以子查询转换成与外层查询的 join 操作，但是在转换过程中，会涉及到 null 值的处理，以下是子查询的部分场景，使用 Calcite 的`SubQueryRemoveRule`这个子查询转换规则，转换处理后的 SQL 语句和逻辑执行计划会一起列出。

比如，对于以下这个 in 子查询，Calcite 会将 in 子查询转换成 inner join。

```sql
select empno, gender, name from EMPS
 where gender = 'F' and empno > 110 and DEPTNO in (select DEPTNO from DEPTS where NAME='Marketing')
```

其逻辑执行计划为

```sql
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

又例如以下的 agg 子查询，则会转换为 left join。

```sql
select empno, gender, name from EMPS
 where gender = 'F' and empno > 110 and DEPTNO = (select max(DEPTNO) from DEPTS where NAME='Marketing')
```

其逻辑执行计划为

```sql
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
由上面的例子可见，对于非相关子查询，由于外层查询不依赖子查询，将子查询的 `RexSubQuery` expression 按照一定的规则约束转换成对应的 join 操作即可，而对于相关子查询的处理则更复杂一些。
## 相关子查询

相关子查询的查询条件依赖于外层查询，在 SQL 引擎处理中，也是尽量去除关联。比如如下关联子查询的 SQL 语句，子查询出现在 SELECT 列表中，而子查询的查询条件依赖外层条件的一个字段。

```sql
select 
empno, name, (select name from DEPTS where DEPTS.DEPTNO=EMPS.DEPTNO) as dpt_name 
from EMPS where gender = 'F' and empno > 110
```
Calcite 转换成的逻辑执行计划是，可以看到，其最外层的`LogicalProject`算子有一个属性是`$SCALAR_QUERY`，这是`RexSubQuery` 类型（一种`RexNode`），对应的又是一个算子树，包含`LogicalProject`，`LogicalFilter`和`LogicalTableScan`。

```sql
LogicalProject(EMPNO=[$0], NAME=[$1], DPT_NAME=[$SCALAR_QUERY({
LogicalProject(NAME=[$1])
  LogicalFilter(condition=[=($0, $cor0.DEPTNO)])
    LogicalTableScan(table=[[SALES, DEPTS]])
})])
  LogicalFilter(condition=[AND(=($2, 'F'), >($0, 110))])
    LogicalProject(EMPNO=[$0], NAME=[$1], GENDER=[$3])
      LogicalTableScan(table=[[SALES, EMPS]])

```

经过 Calcite 的`SubQueryRemoveRule`处理后，转换而成的逻辑执行计划如下所示，可以看到`RexSubQuery`类型的`$SCALAR_QUERY`被替换成了`LogicalCorrelate`算子，这就表示相关子查询。接下来就需要借助`RelDecorrelator`将相关子查询转换成 join 了。

```sql
LogicalProject(EMPNO=[$0], NAME=[$1], DPT_NAME=[$3])
  LogicalCorrelate(correlation=[$cor0], joinType=[left], requiredColumns=[{2}])
    LogicalFilter(condition=[AND(=($2, 'F'), >($0, 110))])
      LogicalProject(EMPNO=[$0], NAME=[$1], GENDER=[$3])
        LogicalTableScan(table=[[SALES, EMPS]])
    LogicalAggregate(group=[{}], agg#0=[SINGLE_VALUE($0)])
      LogicalProject(NAME=[$1])
        LogicalFilter(condition=[=($0, $cor0.DEPTNO)])
          LogicalTableScan(table=[[SALES, DEPTS]])
```

通过`RelDecorrelator#decorrelateQuery`去关联化之后的逻辑执行计划如下所示。Calcite 的关联子查询转 join 的逻辑在`org.apache.calcite.sql2rel.RelDecorrelator#decorrelateRel(Correlate rel, boolean isCorVarDefined)`这个重载方法中实现。

```sql
LogicalProject(EMPNO=[$0], NAME=[$1], DPT_NAME=[$4])
  LogicalJoin(condition=[=($2, $3)], joinType=[left])
    LogicalProject(EMPNO=[$0], NAME=[$1], GENDER=[$3])
      LogicalFilter(condition=[AND(=($3, 'F'), >($0, 110))])
        LogicalTableScan(table=[[SALES, EMPS]])
    LogicalAggregate(group=[{0}], agg#0=[SINGLE_VALUE($1)])
      LogicalProject(DEPTNO=[$0], NAME=[$1])
        LogicalFilter(condition=[IS NOT NULL($0)])
          LogicalTableScan(table=[[SALES, DEPTS]])
```

## Reference
* https://github.com/apache/calcite
* https://docs.pingcap.com/zh/tidb/stable/subquery-optimization
* https://www.bookstack.cn/read/TiDB-4.0/explain-subqueries.md
