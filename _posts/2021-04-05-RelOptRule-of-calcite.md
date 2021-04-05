---
layout: post
title: Apache Calcite 优化规则介绍
category: Calcite
---

在 Calcite 中，优化规则主要用于对关系代数表达式做转换，优化规则的转换可以是算子下推，列裁剪等等类型的转换。比如`LogicalJoin`上层的 Filter 条件下推到`LogicalJoin` 下面，也就是先做选择，再做连接，这种转换可以减少 Join 处理的数据量，从而提升处理效率。优化规则的转换也可以是是逻辑算子到物理算子的转换，比如将 LogicalJoin 转换为物理的算子`NestedLoopJoin` ，其中逻辑关系代数表达式 LogicalJoin 表明这是一个 join 操作，并且指定了这个 join 的类型（inner，left outer，right outer 等等），join 的条件（比如 a.id=b.id），而 `NestedLoopJoin` 则表明了这个 join 在执行时的算法为 nested loop join。总的来说，规则就是用来优化关系代数表达式的具体执行算法。

## Calcite 中优化规则的实现

在 Calcite 中，所有的优化规则都继承自 `org.apache.calcite.plan.RelOptRule` （在 2.0 版本之前会被 RelRule给替换掉，目前构造 operand 可以由 RelRule 中提供的类来构建，参考 RelRule 的注释）。
`RelOptRule` 主要的方法有

```java
  public boolean matches(RelOptRuleCall call) {
    return true;
  }

  public abstract void onMatch(RelOptRuleCall call);
```

### 一个简单的优化规则

先来看 Calcite 内置的一个简单的优化规则，`org.apache.calcite.rel.rules.FilterToCalcRule`，它将 LogicalFilter 转换成 LogicalCalc，其类定义如下。

```java
public class FilterToCalcRule
    extends RelRule<FilterToCalcRule.Config>
    implements TransformationRule {
  protected FilterToCalcRule(Config config) {
    super(config);
  }
  
  @Override public void onMatch(RelOptRuleCall call) {
    final LogicalFilter filter = call.rel(0);
    final RelNode rel = filter.getInput();
    // Create a program containing a filter.
    final RexBuilder rexBuilder = filter.getCluster().getRexBuilder();
    final RelDataType inputRowType = rel.getRowType();
    final RexProgramBuilder programBuilder =
        new RexProgramBuilder(inputRowType, rexBuilder);
    programBuilder.addIdentity();
    programBuilder.addCondition(filter.getCondition());
    final RexProgram program = programBuilder.getProgram();
    final LogicalCalc calc = LogicalCalc.create(rel, program);
    call.transformTo(calc);
  }
  /** Rule configuration. */
  public interface Config extends RelRule.Config {
    Config DEFAULT = EMPTY
        .withOperandSupplier(b ->
            b.operand(LogicalFilter.class).anyInputs())
        .as(Config.class);
    @Override default FilterToCalcRule toRule() {
      return new FilterToCalcRule(this);
    }
  }
}
```

除了构造参数，还有 FilterToCalcRule.Config 接口定义，以及 onMatch 方法定义。这个规则将 LogicalFilter 算子转换成了 LogicalCalc。比如对于 SQL 语句 `select order_id, user_id,status from t_order where t_order.status='FINISHED'`，其通过 AST 转换成的逻辑执行计划为。

```sql
LogicalProject(order_id=[$0], user_id=[$1], status=[$2])
  LogicalFilter(condition=[=($2, 'FINISHED')])
    LogicalTableScan(table=[[logical_db, t_order]])

```

通过 `FilterToCalcRule`规则转换后的逻辑执行计划如下所示。可以看到`LogicalFilter`已经被转换成了`LogicalCalc`。详细的代码参考 [HepPlannerTest](https://github.com/guimingyue/shardingsphere/blob/optimizer-demo/shardingsphere-infra/shardingsphere-infra-optimizer/src/test/java/org/apache/shardingsphere/infra/optimizer/planner/HepPlannerTest.java#LC77)。

```sql
LogicalProject(order_id=[$0], user_id=[$1], status=[$2])
  LogicalCalc(expr#0..2=[{inputs}], expr#3=['FINISHED':VARCHAR], expr#4=[=($t2, $t3)], proj#0..2=[{exprs}], $condition=[$t4])
    LogicalTableScan(table=[[logical_db, t_order]])
```

在 Calcite 中，定义了 `Calc` 用于聚合相邻的`Project`和`Filter`算子，这样就可以将`Project`和`Filter`这两个算子聚合在一起进行计算，在`Volcano`执行模型中就不必执行两次 `next`计算。如果再将上述的转换规则列表加上 `ProjectToCalcRule`和`CalcMergeRule`最后生成的逻辑执行计划如下。也就是`LogicalProject`和`LogicalFilter`合并成了一个`LogicalCalc`算子。详细的代码参考[HepPlannerTest](https://github.com/guimingyue/shardingsphere/blob/optimizer-demo/shardingsphere-infra/shardingsphere-infra-optimizer/src/test/java/org/apache/shardingsphere/infra/optimizer/planner/HepPlannerTest.java#LC102)

```sql
LogicalCalc(expr#0..2=[{inputs}], expr#3=['FINISHED':VARCHAR], expr#4=[=($t2, $t3)], proj#0..2=[{exprs}], $condition=[$t4])
  LogicalTableScan(table=[[logical_db, t_order]])
```

### 优化规则匹配与执行过程

在上述的优化规则示例中可以看到，其内部的接口 Config 定义了其操作数（operand，称算子即 operator 更合适一些）为`LogicalFilter`，这个算子可以有任意输入（`anyInputs`）。在 RelOptPlanner 的优化过程中，会先确定规则中的`RelOptRuleOperand`与当前的算子（`RelNode`）是否匹配以及`RelOptRuleOperand`的子 operand 与算子的子算子（`RelNode.etInputs()`）是否匹配。如果匹配，则创建`RelOptRuleCall`的对象，再调用规则的 `matches` 方法，确定是否执行 `onMatch` 来执行真正的转换规则。所以`matches`方法的作用是判断规则是否与给定的`RelOptRuleCall` 参数匹配（whether this RelOptRule matches a given RelOptRuleCall）。默认情况下`matches`方法会返回 `true`即匹配成功，如果规则定义者需要确定规则是否与传入的`RelOptRuleCall` 参数是否匹配，可以重写这个方法。

HepPlanner 中该匹配与执行规则过程代码如下。

```java
final List<RelNode> bindings = new ArrayList<>();
final Map<RelNode, List<RelNode>> nodeChildren = new HashMap<>();
boolean match =
    matchOperands(
        rule.getOperand(),
        vertex.getCurrentRel(),
        bindings,
        nodeChildren);

if (!match) {
  return null;
}

HepRuleCall call = new HepRuleCall(this,
        rule.getOperand(),
        bindings.toArray(new RelNode[0]),
        nodeChildren,
        parents);

// Allow the rule to apply its own side-conditions.
if (!rule.matches(call)) {
  return null;
}

fireRule(call);

```
`fireRule`就是执行优化规则的方法，在该方法中会调用`onMatch`方法。

### RelOptRuleOperand

`RelOptRuleOperand`用来确定优化规则是否与算子匹配，比如对于 `LogicalProject -> LogicalFilter -> LogicalJoin` 这样一个逻辑执行计划的一部分，LogicalJoin 的输入一定是两个逻辑算子。Calcite 内置的规则 `FilterIntoJoinRule`的作用是将 `Join`上面的`Filter`下推到`Join`的下面，其 Config 接口定义为。

```java
/** Rule configuration. */
public interface Config extends FilterJoinRule.Config {
  Config DEFAULT = EMPTY
      .withOperandSupplier(b0 ->
          b0.operand(Filter.class).oneInput(b1 ->
              b1.operand(Join.class).anyInputs()))
      .as(FilterIntoJoinRule.Config.class)
      .withSmart(true)
      .withPredicate((join, joinType, exp) -> true)
      .as(FilterIntoJoinRule.Config.class);

  @Override default FilterIntoJoinRule toRule() {
    return new FilterIntoJoinRule(this);
  }
}
```
operand 为输入为`Join`的`Filter`，那么与上述的逻辑执行计划匹配。

### RelOptRuleCall
`RelOptRuleCall`是对一次优化规则执行参数的封装，它会作为优化规则方法`matches`和`onMatch`方法的参数。封装了当前调用需要的算子（`rels`），`RelOptRuleOperand`等执行规则的必要参数。对于`FilterIntoJoinRule`规则，其需要的算子包括`Filter`和`Join`，通过`RelOptRuleCall.rel(0)`和`RelOptRuleCall.rel(1)`就可以获取这两个在规则转换中需要的算子。
此外`RelOptRuleCall.transformTo`方法就是将最终转换而成的算子传递给优化器（`RelOptPlanner`），对于 HepPlanner 来说，调用该方法会替换掉原来的`RelNode`，而对于 Volcano ，则可以将其加入`Relset`中，等待后续的代价计算。

## 总结

本文首先介绍了 Calcite 中的优化规则的作用以及实现方式，并通过逻辑执行计划的转换例子，详细介绍了优化规则所操作的对象以及执行完的结果。再通过优化规则的匹配过程以及执行过程，深入介绍了优化规则在优化器中所扮演的角色。优化规则在 Calcite 中属于核心概念，对于深入理解 Calcite 的原理有巨大的帮助。
