---
layout: post
title:  ShardingSphere sql parser介绍
category: ShardingSphere
---


ShardingSphere 的 SQL parser 是基于 ANTLR 实现的，整体实现还算简单，花了点时间看了这部分代码，记录下来。

## 基于 SPI 实现扩展
ShardingSphere 源码中，shardingsphere-sql-parser-spi 这个 module 利用 Java 的 SPI 机制定义了 parser 的扩展点，定义了 `org.apache.shardingsphere.sql.parser.spi.SQLParserFacade` 和 `org.apache.shardingsphere.sql.parser.spi.SQLVisitorFacade` 两个扩展点，其作用主要是定义不同数据库的 SQL 语言的解析接口，最终转换成 ShardingSphere 自定义的 AST：


- `SQLParserFacade`  定义 SQL 解析的扩展点，用于解析文本 SQL，最终返回 antlr 的 AST。
- `SQLVisitorFacade` 定义转换 ANTLR 的 AST 到 ShardingSphere 的 AST 的扩展点。比如 ANTLR 解析出来的是一个 `ParserTree` 的 AST 的数据结构，而 ShardingSphere 自己定义了一套 AST。



## SQL Parser Engine 的实现
ShardingSphere SQL 解析的入口在 shardingsphere-infra-parser 这个 module，具体的方法是 `org.apache.shardingsphere.infra.parser.ShardingSphereSQLParserEngine#parse` ，这个方法入参是 sql 语句字符串（还有一个是否使用缓存的参数，暂时不用关心），返回的是 ShardingSphere 的 AST。
对于 `ShardingSphereSQLParserEngine` ，在初始化的时候，会获取到真实的 parser engine 对象，代码如下
```java
public ShardingSphereSQLParserEngine(final String databaseTypeName) {
    // sql parser 的核心对象
	sqlStatementParserEngine = SQLStatementParserEngineFactory.getSQLStatementParserEngine(databaseTypeName);
    // 看文法文件，是处理 ShardingSphere 自定义的创建数据源和规则语句的
    distSQLStatementParserEngine = new DistSQLStatementParserEngine();
    // SQL 解析回调
    parsingHookRegistry = ParsingHookRegistry.getInstance();
}
```
`SQLStatementParserEngineFactory.getSQLStatementParserEngine(databaseTypeName)` 这行语句就是创建 SQL parser 的核心对象，这个方法中，会根据参数  `databaseTypeName`  创建出 `org.apache.shardingsphere.infra.parser.sql.SQLStatementParserEngine` 对象。在该类中有 `org.apache.shardingsphere.sql.parser.api.SQLParserEngine` 和 `org.apache.shardingsphere.sql.parser.api.SQLVisitorEngine`，这两个分别负责将 SQL 语句解析成 ANTLR 的 AST 和将 ANTLR 的 AST 转换为 ShardingSphere 定义的 AST。在处理的过程中会根据 `databaseTypeName` 参数来处理对应的数据库，比如针对 MySQL，在解析的过程中拿到 `MySQLParser` 的对象来解析 SQL 语句。
在 `SQLParserEngine` 中，执行解析 SQL 和转换 ANTLR 的 AST 的代码如下。首先由 SQLParserEngine.parse 方法将 SQL 解析成 `org.antlr.v4.runtime.tree.ParseTree`  对象，再由 `org.apache.shardingsphere.sql.parser.api.SQLVisitorEngine#visit`  将 ParseTree 的对象转换为 ShardingSphere 的 AST，即 org.apache.shardingsphere.sql.parser.sql.common.statement.SQLStatement 的子类对象。
```java
private SQLStatement parse(final String sql) {
	return visitorEngine.visit(parserEngine.parse(sql, false));
}
```
### SQLParserEngine
`org.apache.shardingsphere.sql.parser.api.SQLParserEngine` 中维护了一个 ConcurrentMap，key 是 
`databaseType`，标识数据库的类型，value  是 `org.apache.shardingsphere.sql.parser.core.parser.SQLParserExecutor`，SQL 解析的执行器。在 SQLParserExecutor 中 会创建 SQLParser 对象，比如 `MySQLParser`，然后执行真正的 SQL 文本解析。
```java
private ParseASTNode twoPhaseParse(final String sql) {
    SQLParser sqlParser = SQLParserFactory.newInstance(databaseType, sql);
    try {
        setPredictionMode((Parser) sqlParser, PredictionMode.SLL);
        return (ParseASTNode) sqlParser.parse();
    } catch (final ParseCancellationException ex) {
        ((Parser) sqlParser).reset();
        setPredictionMode((Parser) sqlParser, PredictionMode.LL);
        return (ParseASTNode) sqlParser.parse();
    }
}
```
在 ShardingSphere 中，SQLParser 的实现就比较简单了，因为核心逻辑都在 ANTLR 中，ShardingSphere 在 ANTLR 上做了一层封装。比如 `org.apache.shardingsphere.sql.parser.mysql.parser.MySQLParser` 的实现如下。
```java
/**
 * SQL parser for MySQL.
 */
public final class MySQLParser extends MySQLStatementParser implements SQLParser {
    
    public MySQLParser(final TokenStream input) {
        super(input);
    }
    
    @Override
    public ASTNode parse() {
        return new ParseASTNode(execute());
    }
}
```
### SQLVisitorEngine
SQLVisitorEngine 则将 ANTLR 的 AST 转换为 ShardingSphere 的 AST，整个转换过程基于 Visitor 模式，ANTLR 的 `ParserTree` 支持 Visit 模式（基于 antlr 的 org.antlr.v4.runtime.tree.ParseTreeVisitor类）。代码如下
```java
public <T> T visit(final ParseTree parseTree) {
    ParseTreeVisitor<T> visitor = SQLVisitorFactory.newInstance(databaseType, visitorType, SQLVisitorRule.valueOf(parseTree.getClass()));
    return parseTree.accept(visitor);
}
```
SQLVisitorFactory 是一个创建 `ParseTreeVisitor` 实现对象的工厂，在它的成员变量中持有了各种 `ParseTreeVisitor` 的实现对象，其 `newInstance` 方法会根据 databaseType，和 visitorType 返回对应的 `ParseTreeVisitor` 实现类的实例，需要注意的是第三个参数，其定义了 SQL 操作的类型，比如 SELECT，INSERT，UPDATE 等等。`newInstance` 的代码如下。
```java
public static <T> ParseTreeVisitor<T> newInstance(final String databaseType, final String visitorType, final SQLVisitorRule visitorRule) {
    SQLVisitorFacade facade = SQLVisitorFacadeRegistry.getInstance().getSQLVisitorFacade(databaseType, visitorType);
    return createParseTreeVisitor(facade, visitorRule.getType());
}
    
@SuppressWarnings("unchecked")
@SneakyThrows(ReflectiveOperationException.class)
private static <T> ParseTreeVisitor<T> createParseTreeVisitor(final SQLVisitorFacade visitorFacade, final SQLStatementType type) {
    switch (type) {
        case DML:
            return (ParseTreeVisitor) visitorFacade.getDMLVisitorClass().getConstructor().newInstance();
        case DDL:
            return (ParseTreeVisitor) visitorFacade.getDDLVisitorClass().getConstructor().newInstance();
        case TCL:
            return (ParseTreeVisitor) visitorFacade.getTCLVisitorClass().getConstructor().newInstance();
        case DCL:
            return (ParseTreeVisitor) visitorFacade.getDCLVisitorClass().getConstructor().newInstance();
        case DAL:
            return (ParseTreeVisitor) visitorFacade.getDALVisitorClass().getConstructor().newInstance();
        case RL:
            return (ParseTreeVisitor) visitorFacade.getRLVisitorClass().getConstructor().newInstance();
        default:
            throw new SQLParsingException("Can not support SQL statement type: `%s`", type);
    }
}
```
SQLVisitorFacadeRegistry 基于 Java 的 SPI 机制实现了 Visitor 的扩展，可以遍历整个 ANTLR 的 AST，对其做遍历的处理，比如在 ShardingSphere 中将各种不同的数据库的 SQL 语句的 ANTLR 的 AST 转换成 ShardingSphere 定义的 AST，如 `MySQLStatementSQLVisitorFacade` ，`OracleFormatSQLVisitorFacade` 等。


## Reference
 
* antlr：[https://www.antlr.org/](https://www.antlr.org/)
* ShardingSphere：[https://shardingsphere.apache.org/](https://shardingsphere.apache.org/)
