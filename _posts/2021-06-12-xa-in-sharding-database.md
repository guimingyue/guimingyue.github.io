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

以下是一个使用 JTA 的 XA 事务接口的例子。其中 Xid 接口是 JTA 对 XA 中 xid 的抽象，下面代码中的 MyXid 为实现类。

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

TODO

## XA 事务恢复



## Reference

* [MySQL XA Transactions](https://dev.mysql.com/doc/refman/8.0/en/xa.html)
* [《MySQL技术内幕：InnoDB存储引擎(第2版)](https://book.douban.com/subject/24708143/)
* [Designing Data-Intensive Applications](https://book.douban.com/subject/26197294/)
* [ACID vs. BASE: Comparison of Database Transaction Models](https://phoenixnap.com/kb/acid-vs-base)