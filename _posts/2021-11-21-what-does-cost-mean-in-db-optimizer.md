---
layout: post
title: 数据库优化器中的代价是什么？
category: Database
---

## 代价的定义

在数据库体系中，优化器是一个非常重要的模块。对一个查询进行了语法分析后，该查询会被转换成了一个逻辑查询计划，逻辑查询计划再由优化器转换为物理执行计划，最后交给执行器去执行。由于执行计划是一个关系代数表达式，所以可以根据关系代数的代数定律，对逻辑执行计划和物理执行计划进行等价的转换。比如根据交换律可以将 `A join B` 转换为 `B join A`，根据结合率可以将 `(A join B) join C` 转换为 `A join (B join C)`。数据库优化器接收的输入是一个逻辑执行计划，输出的是一个代价相对较小物理执行计划。而根据代数定律，逻辑执行计划可以转换为多个等价的逻辑执行计划，如果不加选择，逻辑执行计划最终被转换为多个逻辑上等价的物理执行计划。

优化器主要分为基于规则的优化器和基于代价的优化器。基于规则的优化器会根据预先提供的优化规则对关系代数表达式进行优化，除了数据库 schema，表信息等元信息，它仅仅依赖优化规则，最终会根据优化规则生成一个物理执行计划。而基于代价的优化器除了上述的数据库 schema，表信息等元信息外，还需要一些统计信息来估计每种物理执行计划的执行代价。比如对于以下的 SQL 语句，可能的物理执行计划可能是有如下两种，如图所示。

```sql

```



基于代价的优化器会根据预估的关系代数运算符的物理代价，计算出物理执行计划所对应的代价，最终由基于代价的优化器选择出物理代价相对最小的物理执行计划。

## 代价模型
为了比较各个不同的执行计划的代价，首先需要知道执行计划的代价是多少，而代价的量化则就需要先定义出代价大小的评判标准，这就是代价模型。比如对于一个物理执行计划，执行它的过程中，每一种操作符的执行消耗是多少。而执行消耗可以是 CPU 占用时长，可以使内存占用大小，也可以是执行过程中的 IO 开销。代价模型由优化器根据具体的实现进行定义，可以是一个标量值，也可以是多个标量值的集合，甚至还可以是一个向量，只要能在优化的过程中比较不同的操作符执行消耗的大小就行。

在 Apache Calcite 的优化器框架中，定义了[`org.apache.calcite.plan.RelOptCost`](https://github.com/apache/calcite/blob/master/core/src/main/java/org/apache/calcite/plan/RelOptCost.java)这个接口，用于在优化器中实现代价的抽象，在其定义中，与代价属性相关的方法定义如下。主要有数据行数，cpu 资源和 I/O 资源这三种代价相关属性。在该接口的默认实现中[org.apache.calcite.plan.RelOptCostImpl](https://github.com/apache/calcite/blob/master/core/src/main/java/org/apache/calcite/plan/RelOptCostImpl.java)，只使用了数据行数用于定义代价模型。

```java
public interface RelOptCost {

  /** Returns the number of rows processed; this should not be
   * confused with the row count produced by a relational expression
   * ({@link org.apache.calcite.rel.RelNode#estimateRowCount}). */
  double getRows();

  /** Returns usage of CPU resources. */
  double getCpu();

  /** Returns usage of I/O resources. */
  double getIo();
```
而在 Apache Calcite 的 Volcano/Cascades 的优化器的代价模型实现中[org.apache.calcite.plan.volcano.VolcanoCost](https://github.com/apache/calcite/blob/master/core/src/main/java/org/apache/calcite/plan/volcano/VolcanoCost.java)，使用的代价模型使用了数据行数，CPU 资源和 IO 资源来定义代价模型，代码如下。

```java
class VolcanoCost implements RelOptCost {

  final double cpu;
  final double io;
  final double rowCount;

  VolcanoCost(double rowCount, double cpu, double io) {
    this.rowCount = rowCount;
    this.cpu = cpu;
    this.io = io;
  }

  @Override 
  public double getCpu() {
    return cpu;
  }

  @Override 
  public double getIo() {
    return io;
  }

  @Override 
  public double getRows() {
    return rowCount;
  }
```


## 代价估算
由于物理执行计划的代价只有在执行的时候才能精确的测量具体的代价值是多少，比如每个操作符的执行占用了多少 CPU 时间，占用了多少内存空间等，但是将每个物理计划执行一遍来对比代价显然是不现实的，所以在优化阶段的代价是估算的，也就是说并不是精确的。代价的计算依赖于数据库中的元数据参数和统计信息，这些参数要么是精确元数据信息，要么是不精确的估算信息。

在优化阶段使用到的元数据信息主要有表结构信息，列索引信息和列值的分布（比如对于 sharding 场景下的拆分列和拆分算法）。统计信息主要有列值直方图，NVD 值，Null 值的数量。

在 Apache Calcite 中，定义了 `org.apache.calcite.rel.RelNode#computeSelfCost` 来计算每个操作符的代价。

```java
  /**
   * Returns the cost of this plan (not including children). The base
   * implementation throws an error; derived classes should override.
   *
   * @param planner Planner for cost calculation
   * @param mq Metadata query
   * @return Cost of this plan (not including children)
   */
  @Nullable RelOptCost computeSelfCost(RelOptPlanner planner, RelMetadataQuery mq);
```

例如 `org.apache.calcite.rel.core.Filter` 操作符计算代价就利用估算的行数和基于的子操作符的行数计算的 CPU 消耗，进行计算，由于该 Filter 操作符不涉及到 I/O，所以设置的 `dIo` 数值为 0.

```java
@Override 
public @Nullable RelOptCost computeSelfCost(RelOptPlanner planner, RelMetadataQuery mq) {
    double dRows = mq.getRowCount(this);
    double dCpu = mq.getRowCount(getInput());
    double dIo = 0;
    return planner.getCostFactory().makeCost(dRows, dCpu, dIo);
  }

```

## 总结


## Reference
* Apache Calcite: A Foundational Framework for Optimized Query Processing Over Heterogeneous Data Sources
* Orca: A Modular Query Optimizer Architecture for Big Data
* [https://calcite.apache.org/](https://calcite.apache.org/)
* [What is Cost-based Optimization?](https://www.querifylabs.com/blog/what-is-cost-based-optimization)
* 《数据库系统实现》第二版

