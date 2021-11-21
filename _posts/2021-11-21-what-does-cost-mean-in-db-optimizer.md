---
layout: post
title: 数据库优化器中的代价是什么？
category: Database
---

## 代价的定义

在数据库理论体系中，优化器是一个非常重要的组件，主要分为基于规则的优化器和基于代价的优化器。基于规则的优化器会根据预先提供的优化规则对关系代数表达式进行优化，除了数据库 schema，表信息等元信息，它仅仅依赖优化规则。而基于代价的优化器除了上述的数据库 schema，表信息等原信息，还会根据关系代数表达式计算出执行计划所对应的代价，最终由基于代价的优化器选择出代价相对最小的执行计划。

## 代价模型
// TODO 

## Reference
* Apache Calcite: A Foundational Framework for Optimized Query Processing Over Heterogeneous Data Sources
* Orca: A Modular Query Optimizer Architecture for Big Data
* [https://calcite.apache.org/](https://calcite.apache.org/)

