---
layout: post
title: Apache Calcite HepPlanner 原理
category: Calcite
---

## 什么是启发式优化

启发式优化本质上是给定一组优化规则，通过将这组规则运用到关系代数表达式的每个节点上，从而优化关系代数表达式。而运用规则时，则会先判断规则是否适用于当前的关系代数表达式的节点，如果适用，则运用规则（执行转换），否则就继续试探下一个节点。启发式规则是前人总结出来的宝贵经验，运用该规则优化一定会带来优化的收益。


## Calcite 的启发式优化实现 HepPlanner

### 如何使用 HepPlanner

如下代码片段是 HepPlanner 的使用方式，详细代码请参考 [AbstractPlanner#rewrite](https://github.com/guimingyue/shardingsphere/blob/master/shardingsphere-infra/shardingsphere-infra-optimize/src/main/java/org/apache/shardingsphere/infra/optimize/planner/AbstractPlanner.java)。[`PlannerRules.HEL_RULES`](https://github.com/guimingyue/shardingsphere/blob/master/shardingsphere-infra/shardingsphere-infra-optimize/src/main/java/org/apache/shardingsphere/infra/optimize/planner/PlannerRules.java) 是一个启发式优化规则的集合。

```java
HepProgramBuilder hepProgramBuilder = HepProgram.builder();
// 添加优化规则，PlannerRules.HEL_RULES 是预先定义好的优化规则集合
for(Collection<? extends RelOptRule> rules : PlannerRules.HEL_RULES) {
    hepProgramBuilder.addGroupBegin();
    rules.forEach(hepProgramBuilder::addRuleInstance);
    hepProgramBuilder.addGroupEnd();
}
// 创建 HepPlanner 对象
HepPlanner hepPlanner = new HepPlanner(hepProgramBuilder.build(), null, true, null, RelOptCostImpl.FACTORY);
// 初始化待优化的逻辑关系代数表达式
hepPlanner.setRoot(logicalRelNode);
// 执行优化，rewritedRelNode 就是优化完的关系代数表达式
RelNode rewritedRelNode = hepPlanner.findBestExp();

```

最后，通过`findBestExp`方法计算得到的就是优化完的执行计划。整个优化流程就是应用规则的过程。关于规则，可以参考文章 [Apache Calcite 核心概念梳理](http://guimy.me/other/2021/01/02/introduction-to-apache-calcite.html#reloptrule-和-reloptplanner)。

### HepPlanner 核心概念

Calcite HepPlanner 整个优化流程，首先是将关系代数表达式的树形结构转换成一个图形结构，再通过遍历加入到 HepPlanner 中的规则集合，对于每个规则集合应用到初始化时生成的图形结构中。

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
