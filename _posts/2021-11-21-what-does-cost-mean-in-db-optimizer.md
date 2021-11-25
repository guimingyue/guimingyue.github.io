---
layout: post
title: 数据库优化器中的代价是什么？
category: Database
---

## 代价的定义

在数据库理论体系中，优化器是一个非常重要的组件。对一个查询进行了语法分析后会将该查询转换成了一个逻辑查询计划，逻辑查询计划由优化器转换为物理执行计划，最后交给执行器去执行。由于逻辑执行计划是一个关系代数表达式，所以可以根据关系代数的代数定律，对逻辑执行计划进行等价的转换。比如根据交换律可以将 A join B 转换为 B join A，根据结合率可以将 (A join B) join C 转换为 A join (B join C)。数据库优化器接收的输入是一个逻辑执行计划，输出的是一个物理执行计划，而根据代数定律，逻辑执行计划可以转换为多个等价的逻辑执行计划，最终转换为多个物理执行计划。

优化器主要分为基于规则的优化器和基于代价的优化器。基于规则的优化器会根据预先提供的优化规则对关系代数表达式进行优化，除了数据库 schema，表信息等元信息，它仅仅依赖优化规则，最终会根据优化规则生成一个物理执行计划。而基于代价的优化器除了上述的数据库 schema，表信息等元信息外，还需要一些统计信息来估计每种物理执行计划的执行代价。比如对于以下的 SQL 语句，可能的物理执行计划可能是有如下两种，如图所示。

```sql

```




基于代价的优化器会根据预估的关系代数运算符的物理代价，计算出物理执行计划所对应的代价，最终由基于代价的优化器选择出物理代价相对最小的物理执行计划。

## 代价模型
// TODO 

## Reference
* Apache Calcite: A Foundational Framework for Optimized Query Processing Over Heterogeneous Data Sources
* Orca: A Modular Query Optimizer Architecture for Big Data
* [https://calcite.apache.org/](https://calcite.apache.org/)
* [What is Cost-based Optimization?](https://www.querifylabs.com/blog/what-is-cost-based-optimization)
* 《数据库系统实现》第二版

