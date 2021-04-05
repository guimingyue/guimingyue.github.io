---
layout: post
title: Apache Calcite HepPlanner 原理
category: Calcite
---

## 什么是启发式优化

启发式优化（heuristic）是数据库查询优化中常用的而一种技术手段，也称为基于规则的优化（RBO）。其使用场景常常是作为基于代价的优化（CBO）的一种补充，因为基于代价的优化的一个缺点是优化本身是有代价的，在等价集合中找到最优的计划是仍然需要很多计算代价，所以基于代价的优化器可以使用启发式的方法来减少优化代价。比如 CBO 优化器在执行代价优化之前，会有一个查询改写（rewrite）的阶段，这个阶段就应用了启发式的优化手段。
启发式优化本质上是给定一组优化规则，通过将这组规则运用到关系代数表达式的每个节点上，从而优化关系代数表达式。而运用规则时，则会先判断规则是否适用于当前的关系代数表达式的节点，如果适用，则运用规则（执行转换），否则就继续试探下一个节点。这些规则是前人总结出来的宝贵经验，运用该规则优化通会带来优化的收益，但也不总是会带来优化收益。


## Calcite 的启发式优化实现

在 Calcite 中，`HepPlanner` 是查询优化器（`RelOptPlanner`）的一个启发式优化实现。

### 使用 HepPlanner

如下代码片段是 `HepPlanner` 的使用方式，详细代码请参考 [HepPlannerTest](https://github.com/guimingyue/shardingsphere/blob/optimizer-demo/shardingsphere-infra/shardingsphere-infra-optimizer/src/test/java/org/apache/shardingsphere/infra/optimizer/planner/HepPlannerTest.java)。

```java
HepProgramBuilder hepProgramBuilder = HepProgram.builder();
hepProgramBuilder.addGroupBegin();
hepProgramBuilder.addRuleCollection(ImmutableList.of(CoreRules.FILTER_INTO_JOIN));
hepProgramBuilder.addGroupEnd();

HepPlanner hepPlanner = new HepPlanner(hepProgramBuilder.build(), null, true, null, RelOptCostImpl.FACTORY);
hepPlanner.setRoot(logicalRelNode);
RelNode best = hepPlanner.findBestExp();

```
使用 HepPlanner 分四步：
0. 通过 `HepProgramBuilder`构建一个`HepProgram`，它制定了在优化过程中，使用规则的顺序。
1. 创建 `HepPlanner` 对象。
2. 设置待优化的关系代数表达式（`setRoot`）。
3. 执行优化（`findBestExp`），该方法计算得到的就是优化完的执行计划。

整个优化流程就是应用规则的过程，在上面的代码片段中仅仅使用了一个优化规则 `CoreRules.FILTER_INTO_JOIN`，该优化规则的作用是将 Join 上面的 Filter 中可以下推到 Join 下面的条件，下推下去。关于规则，可以参考文章 [Apache Calcite 核心概念梳理](http://guimy.me/other/2021/01/02/introduction-to-apache-calcite.html#reloptrule-和-reloptplanner)。

比如，对于如下 SQL 语句，

```sql
select o1.order_id, o1.order_id, o1.user_id, o2.status from t_order o1 join t_order_item o2 
on o1.order_id = o2.order_id where o1.status='FINISHED' and o2.order_item_id > 1024 and 1=1
```

由 AST 转换而成的关系代数表达式为。

```sql
LogicalProject(order_id=[$0], order_id0=[$0], user_id=[$1], status=[$6])
  LogicalFilter(condition=[AND(=($2, 'FINISHED'), >($3, 1024), =(1, 1))])
    LogicalJoin(condition=[=($0, $4)], joinType=[inner])
      LogicalTableScan(table=[[logical_db, t_order]])
      LogicalTableScan(table=[[logical_db, t_order_item]])
```

经过以上测试代码优化后的关系代数表达式为。

```sql
LogicalProject(order_id=[$0], order_id0=[$0], user_id=[$1], status=[$6])
  LogicalJoin(condition=[=($0, $4)], joinType=[inner])
    LogicalFilter(condition=[=($2, 'FINISHED')])
      LogicalTableScan(table=[[logical_db, t_order]])
    LogicalFilter(condition=[>($0, 1024)])
      LogicalTableScan(table=[[logical_db, t_order_item]])

```

可以看到，Filter 条件已经被下推到 JOIN 下面了

### HepPlanner 核心概念

Calcite HepPlanner 优化流程中，首先是将关系代数表达式的树形结构转换成一个图形结构，再通过遍历加入到 HepPlanner 中的规则集合，对于每个规则集合应用到初始化时生成的图形结构中。

#### DirectedGraph

Calcite HepPlanner 在初始化时（`org.apache.calcite.plan.hep.HepPlanner#setRoot`），会将逻辑关系代数表达式（`RelNode`）转换成一个图形结构（`DirectedGraph`的对象）。为什么需要将 RelNode 因为 HepPlanner 在应用规则优化的过程中会使用新创建的关系代数表达式（RelNode）替换掉旧的，而 Calcite 定义的关系代数表达式本质上是一颗从上到下的树形结构，在 RelNode 中无法知道父节点是什么，所以通过将 RelNode 转换成图，可以有效的解决这种问题，查找父节点的代码可以参考 `org.apache.calcite.plan.hep.HepPlanner#applyTransformationResults` 该方法是在应用完规则以后，替换旧的关系代数表达式中节点的方法。

`DirectedGraph`的默认实现是`DefaultDirectedGraph`，在`DefaultDirectedGraph`中有代表所有节点的属性`vertexMap`和代表所有边的属性`edges`，它们的元素分别为`HepRelVertex` 和 `DefaultEdge`。其中`vertexMap`属性是 Map 结构，其键类型是`HepRelVertex`，值的类型是`VertexInfo`，它表示每个节点的出线和入线。


### HepPlanner 优化流程

HepPlanner 分为初始化和优化两个流程，初始化会将关系代数表达式（`RelNode`）转换为图（`DirectedGraph`），优化流程才是真正的遍历和运用规则来优化关系代数表达式。

#### 初始化`setRoot`

`setRoot`核心逻辑是调用方法`org.apache.calcite.plan.hep.HepPlanner#addRelToGraph`将当前的 `RelNode`添加到图（`graph`）中，对于每个`RelNode`都会先将其子节点加入到图中，再将当前节点生成节点（`HepRelVertex`）和边（`DefaultEdge`）加入到图中。从`HepRelVertex`的代码可以看到，其属性`currentRel`就是当前节点。而`currentRel`的子节点（通过`getInputs`方法获取）是`HepRelVertex`类型。图结构生成完成以后，会将该图的根节点返回，作为优化流程（`findBestPlan`）的输入节点。


#### findBestPlan

HepPlanner 的优化流程就是遍历所有的规则，然后对图中的每个节点应用规则，当然应用规则前会判断当前节点是否适用与当前规则。应用规则的核心代码入口在`org.apache.calcite.plan.hep.HepPlanner#applyRules`。可以看到，在开始应用规则前，会将图形结构转换成一个迭代器，其实就是节点的一个列表，从图中获取节点列表也就是一个遍历图的过程，而遍历图的算法，有深度优先，广度优先和拓扑排序，HepPlanner 主要提供了深度优先和拓扑排序，默认情况下是深度优先。

将图转换成所有节点的迭代器之后，每遍历到一个节点，都会遍历规则，判断该规则是否适用于该节点，应用规则会生成一个新的节点，因为旧的节点可能被丢弃了，所以此时要么重新生成迭代器要么从新的节点开始生成一个局部的迭代器。代码如下

```java
while (iter.hasNext()) {
    HepRelVertex vertex = iter.next();
    for (RelOptRule rule : rules) {
      HepRelVertex newVertex =
          applyRule(rule, vertex, forceConversions);
      if (newVertex == null || newVertex == vertex) {
        continue;
      }
      ++nMatches;
      if (nMatches >= currentProgram.matchLimit) {
        return;
      }
      if (fullRestartAfterTransformation) {
        iter = getGraphIterator(root);
      } else {
        // To the extent possible, pick up where we left
        // off; have to create a new iterator because old
        // one was invalidated by transformation.
        iter = getGraphIterator(newVertex);
        if (currentProgram.matchOrder == HepMatchOrder.DEPTH_FIRST) {
          nMatches =
              depthFirstApply(iter, rules, forceConversions, nMatches);
          if (nMatches >= currentProgram.matchLimit) {
            return;
          }
        }
        // Remember to go around again since we're
        // skipping some stuff.
        fixedPoint = false;
      }
      break;
    }
  }
```

`org.apache.calcite.plan.hep.HepPlanner#applyRule`方法封装了应用单个规则的流程。大致逻辑是先判断规则是否适用于当前节点（`matchOperands`）,如果适用，就生成`HepRuleCall`对象，它封装了应用规则所需的节点信息，最后应用规则（`fireRule`）。如果规则执行成功并且生成了新的节点，那么会调用`org.apache.calcite.plan.hep.HepRuleCall#transformTo`方法将新的节点加入到`HepRuleCall`定义的结果集中，最后调用方法`org.apache.calcite.plan.hep.HepPlanner#applyTransformationResults`将新的节点加入图中。


## 总结

本文总结了 Calcite HepPlanner 的原理以及核心的调用流程，相对于 Calcite 的 VolcanoPlanner，HepPlanner 还是比较简单的，理解概念了，浏览一遍代码基本能弄懂其原理。弄懂 HepPlanner 的原理对设计启发式的优化器和优化规则的帮助是巨大的。

## Reference 

* [Calcite](http://calcite.apache.org/)
