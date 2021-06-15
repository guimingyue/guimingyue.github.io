---
layout: post
title: 基于 XA 接口的分布式事务在分库分表场景中的应用
category: Database
---

## 分布式事务

分布式事务指的是允许多个独立的事务资源（Transactional Resources）参与到一个全局的事务中，所有在事务资源中的变动比如与全局事务一起成功或者失败。分布式事务分为 BASE 事务模型和强一致的分布式事务模型（ACID）。

BASE 常常被称为柔性事务，具体是指：
* 基本可用，Basically Available，通过复制技术保证整个集群是可用的。
* 柔性状态，Soft state，事务中的数据可能在事务执行过程中不断改变，并且对事务外可见。
* 最终一致，Eventually consistent，数据会达到最终一致性。

相对于强一致事务，柔性事务的局限在于仅仅能实现读未提交的事务隔离级别，而强一致的分布式事务则可以实现强于读未提交的事务隔离级别，比如基于 XA 接口，可以实现读已提交的事务隔离级别。

### 两阶段提交

![2PC](/images/xa-in-sharding-database/two-phase-commit.png)

### XA 接口

X/Open 组织在 1991 年定义了 The X/Open Distributed Transaction Processing 模型，简称为 DTP 模型，它是一种分布式事务的规范。在 DTP 模型中，存在三种组件。

* Application Program（AP），也就是指应用程序。
* Resource Managers（RM），分布式事务的参与者，它是事务资源持有者。
* Transaction Manager（TM），分布式事务的协调器，主要负责指定事务标志，监控事务进程，并且保证事务成功完成或者失败恢复。

DTP 的示意图，如下图所示。

![DTP Model](/images/xa-in-sharding-database/DTP-model.png)

XA 接口则是 DTP 模型中，TM 和 RM 的交互接口，它定义了一个分布式事务流程中，所需要的接口，接口描述如下图所示。

![XA Interface](/images/xa-in-sharding-database/xa-interface.png)


### MySQL 对 XA 接口的支持

MySQL 数据库的 InnoDB 存储引擎支持 XA 接口，并且 MySQL 数据库的 Server 层面也通过 XA 开头的语句支持了 XA 事务。

```sql
XA {START|BEGIN} xid [JOIN|RESUME]
XA END xid [SUSPEND [FOR MIGRATE]]
XA PREPARE xid
XA COMMIT xid [ONE PHASE]
XA ROLLBACK xid
XA RECOVER [CONVERT XID]
```

除了 XA RECOVER，其他语句中都会带有一个 `xid` 的参数，这个参数是一个 XA 分支事务的标识，比如一个全局的分布式事务中，通常会包含多个 XA 事务，每个 XA 事务都会再某一个分库中执行，每一个分库上执行的事务都被称为一个分支事务，所以每一个分支事务都需要一个全局唯一的标识，它由全局事务的事务管理器（TM）生成。

在一个由多个 MySQL 分库参与的，基于 XA 接口的分布式事务中，MySQL 分库就代表资源管理者（RM），那应用程序（AP）就是分布式事务的使用者。所以基于 XA 接口的分库分表场景下的分布式事务的示意图，如下图所示。

![XA Interface](/images/xa-in-sharding-database/xa-database.png)

在一个基于 XA 接口的全局事务中，所有的分支事务要么全部成功，要么全部失败，而事务管理器（TM）无法看到各个分支事务中的数据的版本信息，所以，也就没法在分布式事务层面实现 MVCC 的能力，那么基于 XA 接口的分布式事务也就无法实现可重复读的（REPEATABLE READ）隔离级别了。

## XA 的使用

以下是一个使用 JTA（Java Transaction API） 的 XA 事务接口的例子。其中 Xid 接口是 JTA 对 XA 中 xid 的抽象，MyXid 为其实现类。在 main 方法中，会创建 db1 和 db2 的 XA DataSource，再分别获取到对应的数据库连接。创建完连接以后，就开始各个分支事务，即每个分库的 XA 事务，调用`javax.transaction.xa.XAResource#start`接口，然后就是执行分支事务内的 SQL 语句，结束后就执行`javax.transaction.xa.XAResource#end`标识分支事务的 SQL 语句结束，接下来就是两阶段提交的 prepare 流程，即执行`javax.transaction.xa.XAResource#prepare`，告知 RM 为事务提交做准备，如果每个分库的 prepare 操作都是成功的，就执行最后的提交。这就是一个基于 XA 接口的分布式事务的提交流程。

```java
import com.mysql.jdbc.jdbc2.optional.MysqlXADataSource;

import javax.sql.XAConnection;
import javax.transaction.xa.XAException;
import javax.transaction.xa.XAResource;
import javax.transaction.xa.Xid;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;

public class XADemo {
    
    public static void main(String[]args) throws XAException, SQLException {
        MysqlXADataSource ds1 = createXADataSource("127.0.0.1", 3306, "db1", "peter", "123456");
        XAConnection xaConn1 = ds1.getXAConnection();
        XAResource xaRes1 = xaConn1.getXAResource();
        Connection conn1 = xaConn1.getConnection();
        Statement stmt1 = conn1.createStatement();
    
        MysqlXADataSource ds2 = createXADataSource("127.0.0.1", 3306, "db2", "david", "123456");
        XAConnection xaConn2 = ds2.getXAConnection();
        XAResource xaRes2 = xaConn2.getXAResource();
        Connection conn2 = xaConn2.getConnection();
        Statement stmt2 = conn2.createStatement();
        
        Xid xid1 = new MyXid(100, new byte[]{0x01}, new byte[]{0x02});
        Xid xid2 = new MyXid(100, new byte[]{0x11}, new byte[]{0x12});
        
        // start branch transaction
        xaRes1.start(xid1, XAResource.TMNOFLAGS);
        stmt1.execute("UPDATE account SET money=money-10000 WHERE user='david'");
        xaRes1.end(xid1, XAResource.TMSUCCESS);
        // start branch transaction
        xaRes2.start(xid2, XAResource.TMNOFLAGS);
        stmt2.execute("UPDATE account SET money=money+10000 WHERE user='mariah'");
        xaRes2.end(xid2, XAResource.TMSUCCESS);
        
        // prepare
        int ret1 = xaRes1.prepare(xid1);
        int ret2 = xaRes2.prepare(xid2);
        
        if (ret1 == XAResource.XA_OK && ret2 == XAResource.XA_OK) {
            // commit
            xaRes1.commit(xid1, false);
            xaRes2.commit(xid2, false);
        }
    }

    public static MysqlXADataSource createXADataSource(String ip, int port, String dbName, String user, String passwd){
        MysqlXADataSource ds =new MysqlXADataSource();
        ds.setUrl("jdbc:mysql://" + ip + ":" + port + "/" + dbName);
        ds.setUser(user);
        ds.setPassword(passwd);
        return ds;
    }
}

class MyXid implements Xid {
    private final int formatId;
    private final byte[] gtrid;
    private final byte[] bqual;
    
    public MyXid(int formatId, byte[] gtrid, byte[] bqual) {
        this.formatId = formatId;
        this.gtrid = gtrid;
        this.bqual = bqual;
    }
    
    @Override
    public int getFormatId() {
        return formatId;
    }
    
    @Override
    public byte[] getGlobalTransactionId() {
        return gtrid;
    }
    
    @Override
    public byte[] getBranchQualifier() {
        return bqual;
    }
}
```

## Sharding 模式下如何使用 XA 事务

在 Sharding （分库分表）模式下，如果一个事务要跨多个分库时，就需要分布式事务的支持，如果要支持强一致的分布式事务，那基于底层关系型数据库的 XA 事务来实现强一致的分布式事务，就是一个比较合适的选择。

在 Sharding 模式下，负责计算路由的节点作为分布式事务的事务管理器（TM），底层关系型数据库的分库作为资源管理器（RM）。具体的执行流程如下：
0. 首先 TM 所在的节点接收到事务请求，在节点上开启一个基于 XA 接口的分布式事务（属于当前连接），这个过程仅仅需要构造一个本地的代表 XA 事务的对象，接下来接收到的 SQL 都是在该事务下执行。
1. 接收事务内的 SQL 语句，并计算在哪些分库上执行，第一次获取到每个分库的连接时，执行`XA START ${xid}`语句，其中 xid 需要 TM 生成。
2. 当接收到事务提交的 commit 请求时，对每个底层分库连接执行`XA END ${xid}`语句。
3. 对每个底层分库连接执行 prepare 操作。
4. prepare 操作成功，再对每个分库连接执行分支事务提交 commit 操作，否则，
5. 执行回滚。

### ShardingSphere 中 XA 事务的实现

[ShardingSphere](http://shardingsphere.apache.org/) 分别基于 Atomikos，Narayana 和 Bitronix 这几个框架实现了 XA 事务，使用 SPI 的方式提供服务，使用者可以根据需求，选择具体的实现。XA 事务的类型的选择在构造 ShardingSphereDataSource 对象的时候通过配置项`xa-transaction-manager-type`指定.

在 ShardingSphere 的代码实现中，`ShardingSphereConnection`是 ShardingSphere 提供给客户端的连接，在其构造方法中会传入事务类型（LOCAL，XA，BASE），会根据这个参数构造出对应的事务管理器，如果选择的是 XA 事务，则会使用`XAShardingTransactionManager`类型的对象。

在 ShardingSphere 中，XA 事务运行流程如下：

0. 设置 autoCommit 为 false，代码位于`ShardingSphereConnection#setAutoCommit`，此时会通过 `shardingTransactionManager.begin()` 开启一个 XA 事务，因为`shardingTransactionManager`对象类型为`XAShardingTransactionManager`。这个过程会依赖具体的 XA 事务管理器框架，对于 Bitronix，会创建一个 BitronixTransaction 对象，用于代表开启的分布式事务。
1. 接下来就是正常执行事务内的 SQL 语句，执行 SQL 语句需要根据 SQL 语句计算待执行的分库，所以需要与分库建立连接，这个过程的代码实现在 `ShardingSphereConnection#getConnection`等获取连接的方法中，在执行到 `ShardingSphereConnection#createConnection`方法时，会判断当前事务是否在一个分布式事务中（`isInShardingTransaction`），如果在就通过事务管理器`shardingTransactionManager`获取连接。在事务管理器中会维持底层分库的`XATransactionDataSource`，所以这个获取连接的流程会执行到`XATransactionDataSource#getConnection`。
2. XA 事务中，获取分库连接时，会执行`XA START ${xid}`语句，对于 Bitronix ，相关代码在`bitronix.tm.BitronixTransaction#enlistResource`方法中。其生成 xid 的方法在`bitronix.tm.utils.UidGenerator#generateXid`中，Bitronix 生成的 xid 由 gtrid 和 bqual 两部分组成，它们均是由`bitronix.tm.utils.UidGenerator#generateUid`方法生成，具体生成规则是：timestamp + sequence + serverId，其中 serverId 可以通过配置指定，如果未配置，就使用当前节点的 IP。
3. 在获取到分库连接以后，就按照正常的流程执行 SQL 语句。执行完之后，通过`ShardingSphereConnection#commit`进行事务提交。事务提交会由具体的事务管理器实现。对于 Bitronix 则会执行到`bitronix.tm.BitronixTransactionManager#commit`，事务管理器的事务提交流程与上面的 demo 中的流程类似，首先是执行`XA END ${xid}`结束事务 SQL 语句执行，然后执行 prepare 操作，成功后，执行最终的 commit。
## XA 事务回滚与恢复

TODO


## Reference

* [MySQL XA Transactions](https://dev.mysql.com/doc/refman/8.0/en/xa.html)
* [《MySQL技术内幕：InnoDB存储引擎(第2版)](https://book.douban.com/subject/24708143/)
* [《Designing Data-Intensive Applications》](https://book.douban.com/subject/26197294/)
* [ACID vs. BASE: Comparison of Database Transaction Models](https://phoenixnap.com/kb/acid-vs-base)
* [ShardingSphere XA两阶段事务](https://shardingsphere.apache.org/document/current/cn/features/transaction/principle/2pc-xa-transaction/)
