---
layout: post
title: ShardingSphere 查询优化的设计与实现
category: ShardingSphere
---

Apache ShardingSphere 的官方定义是一套开源的分布式数据库解决方案组成的生态圈，它由 JDBC、Proxy 和 Sidecar（规划中）这 3 款既能够独立部署，又支持混合部署配合使用的产品组成。它们均提供标准化的数据水平扩展、分布式事务和分布式治理等功能，可适用于如 Java 同构、异构语言、云原生等各种多样化的应用场景。
简单来说，它提供的核心功能就是基于关系型数据库的分库分表，读写分离以及分布式事物等能力，但是并没有支持跨库 Join 的能力。去年在学习 Apache Calcite 时与 ShardingSphere 社区沟通过基于 Calcite 实现跨库 Join 的能力，所以近半年时间实现了一个大致的框架，并且基于 Calcite 引入了 RBO 和 CBO 的架构。在这个框架下面，可以执行简单的 SQL 语句，可以完成简单的分布式 Join 的查询（不支持 Exchange 算子），但是还远远没有达到完整的优化的目的。


## 在 Sharding 模式下的跨库 join

跨库 Join 是指数据分布在不同分库中的两张表的 join 操作，比如 A 表是对主键使用 hash 函数进行拆分，而 B 表对主键使用 rang 函数进行拆分，如果它们做 join 操作，那么可能 join key 不是分布在同样的分库，需要另一层计算层来处理计算 join 条件，并合并数据，那么它们的 join 就属于跨库 join。

在 Sharding 模式中，对于每一张被拆分的逻辑表，每一条数据都会经过拆分函数的计算，被路由到指定的分库中的分表，所以对于跨库 join，就需要在计算层（ShardingJDBC 层）计算完成后才能将数据返回。由于跨库 join 是在计算层的数据操作，那么如果 SQL 语句中使用了聚集函数（group by，sum，min，max，avg 等），也需要在上层实现这些函数。另外
对于在分库所在的数据库上执行的函数，也需要在计算层实现，比如 SELECT 的列表中，对 join 列执行的函数。

基于这些，在 Sharding 的计算层的优化策略是，能够下推到分库中的操作尽量下推到分库中执行，无法下推的就在计算层实现。比如，如果两张表是 ShardingSphere 中的绑定表，那 INNER JOIN 是可以下推到分库中执行的，而如果两张表的拆分算法不一样，那么只能在上层处理 join。所以如果要在计算层要支持跨库 join，就需要在计算层 SQL 解析，SQL 优化和执行。
SQL 解析 ShardingSphere 已经有了基于 Antlr 的实现，SQL 优化可以基于 Apache Calcite 来做，SQL 的执行需要基于单独进行实现。整个 SQL 层基于 Calcite 来实现，有一些准备工作，比如将 ShardingSphere 的 AST 转换成 Calcite 能识别的 AST，这样才能交给 Calcite 去转换成逻辑执行计划，进而做 SQL 优化。基于 Calcite 的 SQL 优化主要的工作就是根据优化策略实现 Calcite 的优化规则。而 SQL 的执行则可以基于 Volcano 执行器模型来做。总结起来，整个执行过程可以用下图表示。

![SQL](/images/ss_optimizer/sql_execution.png.png)

## 查询优化器

在关系型数据库中，查询优化以关系代数为基础，目的是将关系代数表达式优化成一个最优的物理执行计划交给执行引擎去执行。优化过程会涉及到关系代数的转换和优化两个过程。转换也称为重写，它的输入是一个逻辑关系代数表达式，输出仍然是逻辑关系代数表达式。优化则不一样，该过程会根据优化规则，将逻辑关系代数表达式中的各个算子，优化成物理算子来给优化器执行，所以优化过程的输入是逻辑关系代数表达式，而输出则是物理关系代数表达式，该表达式中各个算子表明了执行器的执行方式。

Sharding 模式的查询优化属于一种分布式查询优化，而分布式查询优化以减少传输的次数和数据量作为查询优化的目标，分布式数据库系统中的代价估算模型，除了考虑CPU代价和IO代价外，还要考虑通过网络在结点间传输数据的代价。在目前的实现中，优化器基于 Calcite 的查询优化器 HepPlanner 和 VolcanoPlanner。HepPlanner 主要用于逻辑查询计划的重写，是基于规则的启发式的优化，而 VolcanoPlanner 则用于基于代价的优化，主要使用转换规则，针对转换的物理算子计算代价。由于还未定义代价模型以及统计数据，目前只是用于将逻辑算子转换为物理算子。关于 Calcite 的 HepPlanner 优化器和优化规则，可以参考如下两篇文章。

0. [Apache Calcite HepPlanner 原理](http://guimy.me/calcite/2021/01/16/apache-calcite-hepplanner.html)
1. [Apache Calcite 优化规则介绍](http://guimy.me/calcite/2021/04/05/RelOptRule-of-calcite.html)

### 逻辑查询计划重写


逻辑查询计划重写是一种启发式的手段，也称为基于规则的优化（RBO），会按照优化规则将关系代数表达式中的部分算子改写，输出一个改写后的关系代数表达式，这些规则必须是确保新生成的整个表达式是与重写前的关系代数表达式等价的。
Sharding 模式的逻辑查询计划重写的优化策略就是下推，一种是算子的下推，典型的是将 join 之上的 filter 下推到 join 下面。另一种是一部分算子整体下下推，这一点与存储计算分离的分布式数据库不同的一点是，分库分表模型的存储层是有计算能力的，所以可以将 tablescan + filter 一起下推到分库上执行，也可以将符合条件的join下推到分库上执行，所以定义了新的逻辑算子 LogicalScan，用于表示可以下推到分库上执行的算子集合。在执行时会将整个 LogicalScan （对应的物理算子 SSScan）算子转换成能在分库上执行的 SQL 语句。



### 基于代价的优化

由于还没有引入统计信息，所以只是利用 Calcite 的基于代价的优化器 VolcanoPlanner 实现了逻辑执行计划到物理执行计划的转换。


## 执行器

执行器是一个标准的 Volcano 执行器模型，每一种物理算子都会对应一个执行算法实现，执行器的接口是 `Executor`。比如 ，通过 `moveNext` 函数，定义访问下一行数据的操作，通过`current`函数定义访问当前的行。

## 总结

TODO

## Reference

* http://shardingsphere.apache.org/
* 《数据库查询优化的艺术》，作者李海翔 
* https://calcite.apache.org/
