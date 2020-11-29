
layout: post
- title:  Calcite Schema
category: calcite


数据库的 schema 是什么  数据库系统概念和实现

SchemaPlus

CalciteSchema 是 calcite 面向 JDBC 层提供的 schema 实现

CalciteSchema.createRootSchema 方法来创建 root schema


SchameFactory

变量 parentSchema 表明 calcite 的 schema 继承体系

SchemaPlus 是什么？

A schema’s job is to produce a list of tables. (It can also list sub-schemas and table-functions, but these are advanced features and calcite-example-csv does not support them.)  https://calcite.apache.org/docs/tutorial.html


calcite 有两层 schema，第一层是以 jdb 的形式向外提供接口时定义的 schema，第二层是以 adaptor 形式对接存储层时，定义的 schema

看 csv 的 root schema 的实现

看 sqlnode 的校验是怎么使用 schema 的

A schema can also contain sub-schemas, to any level of nesting. Most
 * providers have a limited number of levels; for example, most JDBC databases
 * have either one level ("schemas") or two levels ("database" and
 * "catalog").  https://github.com/apache/calcite/blob/master/core/src/main/java/org/apache/calcite/schema/Schema.java


 calcite 在创建 JDBC Connection 时 CalciteJdbc41Factory.CalciteJdbc41Connection 会调用 CalciteSchema.createRootSchema(true)
而这里的 rootSchema 是 org.apache.calcite.jdbc.CalciteConnectionImpl.RootSchema，该类继承自 org.apache.calcite.schema.impl.AbstractSchema，它又继承自 org.apache.calcite.schema.Schema，这是 calcite 统一的 Schema 接口。