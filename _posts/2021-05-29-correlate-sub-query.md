---
layout: post
title: 数据库子查询优化
category: Database
---

数据库子查询以其简洁，易用并且强大的特性，被广泛应用于各种使用 SQL 语句的查询场景中。在数据库中，子查询在数据库中是如何执行的？是否有优化手段对子查询进行优化？在阅读了与子查询相关的几篇论文以及互联网上的一些资料后，通过这篇文章做一些总结。

## 子查询的概念
子查询是嵌套在另一个查询中的查询表达式，它可以出现在 where 子句中，可以出现在 from 子句中，也可以出现在 select 子句中。从子查询的条件是否依赖外层查询的角度，子查询可以分为非相关子查询和相关子查询。对于非相关子查询，由于子查询的执行不依赖外层查询条件，所以处理比较容易，而相关子查询则比较复杂，因为子查询中依赖外层查询的数据。比如如下两个 SQL 语句分别为非相关子查询和相关子查询。

```sql
-- 非相关子查询
select empno, gender, name from EMPS where gender = 'F' and empno > 110 and DEPTNO in 
    (
        select DEPTNO from DEPTS where NAME='Marketing'
    )

-- 相关子查询
select c_custkey from customer where 1000000 < 
    (
        select sum(o_totalprice) from orders where o_custkey = c_custkey
    )
```

在子查询的语法层面，可以将子查询分为如下三种。

* 标量子查询（Scalar-valued）
* 存在性测试（Existential test）
* 量化比较（Quantified comparison）

## 子查询的优化与执行

对于非相关子查询，优化和执行比较简单，很容易想到的一种执行方式是先执行子查询（或者在执行过程中第一次执行到子查询），然后将子查询的结果物化，在外层查询的过程中直接使用对子查询的物化结果即可。另一种方式就是直接将子查询转换成 join，然后再利用优化器对 join 执行进行优化，这样就是将对子查询的处理转换为了对 join 的处理，在 Calcite 的`SubQueryRemoveRule`实现中，就是将子查询转换为 join 的。

对于相关子查询，内层的子查询依赖外层查询的参数，最简单的执行方式就是基于 nested loop join 的方式执行，也就是对于外层查询的每一条记录，将其作为子查询的参数来执行子查询，如果外层查询的记录数比较少，并且子查询中的表有合适的索引，这样执行子查询会有不错的执行效率，否则这样执行效率会非常低。所以对于相关子查询，需要找到一种优化手段对其进行优化。子查询的消除与去关联就是前人总结的优化子查询的方法。

本文以下部分，非特殊说明，子查询都是指相关子查询。

### 子查询移除

当带有子查询的 SQL 语句被转换成关系代数表达式以后，子查询是由一个标量表达式（Scalar expression）表示的，比如，在 Calcite 中，这个表达式的类型就是`RexSubQuery`。子查询移除是指将表示子查询的标量表达式转换成关系代数表达式，也就是算子，在微软的论文章，引入的算子是`Apply`，在 Calcite 的实现中该算子是`Correlate`，这两者是等价的，下文中对于该算子的描述使用 Apply，而在贴出的关系代数表达式中则使用`Correlate`，比如`LogicalCorrelate`。

比如对于下面相关子查询的 SQL 语句，使用 Calcite 测试，由 SQL 转换而成的关系代数表达式，以及经过子查询移除后的关系代数表达式分别如下所示。SQL 语句中，子查询位于 select 子句中，`$SCALAR_QUERY`是类型为`RexSubQuery`的标量表达式，该表达式代表一个子查询，经过子查询消除后，将标量表达式转换成`LogicalCorrelate`（与 Apply 等价）。 

```sql
-- SQL 语句
select empno, name, (select name from DEPTS where DEPTS.DEPTNO=EMPS.DEPTNO) as dpt_name 
from EMPS where gender = 'F' and empno > 110

-- 子查询移除前
LogicalProject(EMPNO=[$0], NAME=[$1], DPT_NAME=[$SCALAR_QUERY({
LogicalProject(NAME=[$1])
  LogicalFilter(condition=[=($0, $cor0.DEPTNO)])
    LogicalTableScan(table=[[SALES, DEPTS]])
})])
  LogicalFilter(condition=[AND(=($2, 'F'), >($0, 110))])
    LogicalProject(EMPNO=[$0], NAME=[$1], GENDER=[$3])
      LogicalTableScan(table=[[SALES, EMPS]])

-- 子查询移除后
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

经过子查询移除，`LogicalCorrelate`所代表的表达式就是一个参数化的查询（parameterized query），因为如果对其进行求值，它依赖外层查询的输入，执行模式也就是上面所指的 nested loop join。显然经过子查询移除，仍然不足以提升整个查询的执行效率，那么接下来就需要去关联，也就是将 Apply（Correlate）算子转换为其他算子。

### 去关联

Apply 算子本质上就是使用外层查询的每一行去执行内层查询的参数化查询，其实是可以通过 nested loop join 去执行，那么是否可以将 Apply 直接转换为 join 呢？理论上是可行的，这个过程就是去关联，也就是消除参数化表达式，下图是去关联的一些规则。

![rules to remove correlation](/images/sub_query/rules_to_remove_correlation.jpg)

在详细了解去关联规则之前，先看看四种 join。
* cross product
* left outer join
* left semi-join
* left anti-join


## Reference

* [Execution Strategies for SQL Subqueries](http://www.cse.iitb.ac.in/infolab/Data/Courses/CS632/Papers/subquery-proc-elhemali-sigmod07.pdf)
* [Orthogonal Optimization of Subqueries and Aggregation](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.563.8492&rep=rep1&type=pdf)
* [Parameterized Queries and Nesting Equivalencies](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-2000-31.pdf)
* [What is a Database NULL Value?](https://www.essentialsql.com/get-ready-to-learn-sql-server-what-is-a-null-value/?spm=ata.21736010.0.0.59044ec8euPmE1)
* [数据库系统概念](https://book.douban.com/subject/10548379/)
* [Apache Calcite](https://calcite.apache.org/)