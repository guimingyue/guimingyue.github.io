---
layout: post
title: JRaft 实现分析
category: raft
---

断断续续看完了 Raft 的论文，基本了解了 Raft 的几个部分，现在再结合着 JRaft 这个 Raft 的实现看看具体的细节。JRaft 是 Java 版的 Raft 实现，由蚂蚁金服开发，定位于高性能（high-performance），生产级（production-level） 的 Raft 一致性算法的 Java 语言实现库，参考 BRaft 实现的。

## JRaft 概览

JRaft 作为 Raft 的 Java 实现库，从使用的角度，可以分为两部分，客户端和服务端，客户端主要涉及到如何将请求发送给 JRaft 的主节点，比较简单。服务端包含了选主（leader election），日志复制（log replication），快照（snapshot）等 Raft 能力。接下来主要介绍服务端的部分。

将 Raft 作为一个库使用时，需要启动一个 RPC 服务，接受客户端的 RPC 请求，如下面代码就是 JRaft 的例子 `CounterServer`的注册客户端 RPC 的初始化代码。当收到客户端的 RPC 请求时，在`CounterServiceImpl`会调用节点的服务，处理 RPC 请求。

```java
// 这里让 raft RPC 和业务 RPC 使用同一个 RPC server, 通常也可以分开
final RpcServer rpcServer = RaftRpcServerFactory.createRaftRpcServer(serverId.getEndpoint());
// 注册业务处理器
CounterService counterService = new CounterServiceImpl(this);
rpcServer.registerProcessor(new GetValueRequestProcessor(counterService));
rpcServer.registerProcessor(new IncrementAndGetRequestProcessor(counterService));
```
Raft 概念中的节点的实现类是`com.alipay.sofa.jraft.core.NodeImpl`，会在`com.alipay.sofa.jraft.RaftServiceFactory#createAndInitRaftNode` 创建并初始化节点。完成之后，就可以开始接收业务 Client 的 RPC 请求了

## JRaft 实现

### Leader 选举

选主有一个单独的定时器 `electionTimer`，在定时器开始后会开始选主

`voteTimer` 是作为选主的调度器，在选自己为主时开启（`electSelf`方法），在选主完成之后结束。

Candidate 节点发起选举方法`handleElectionTimeout` -> `preVote`，其中`preVote`是针对网络分区场景下的一种优化，比如网络分区使得某个节点 A 发现当前没有 leader 了，所以会一直尝试选主从而 term 变大，但是会一直选主不成功（比如，5 个节点，网络节点分区为两个部分，分别有 3 个节点和 2 个节点），当网络恢复后，A 节点会发现 term 比较大，继续选主，但是这种选主是没有必要的，所以通过`preVote`来避免这种情况。

在`preVote`的 RPC 回调中（`OnPreVoteRpcDone`），会检查是否可以发起选主，如果可以，则调用`electSelf`发起选主（`com.alipay.sofa.jraft.core.NodeImpl#handlePreVoteResponse`）。

### LogStorage 的实现

节点的 LogStorage 在`initLogStorage`方法中初始化，其中 LogStorage 作为 LogManager 持久化日志的存储实现。LogManager 做用户维护节点上的的日志信息，通过 LogStorage 来持久化到磁盘，而 LogStorage 的实现则是通过 RocksDB 实现的，参见`com.alipay.sofa.jraft.storage.impl.RocksDBLogStorage`。

### MetaStorage 的实现

MetaStorage 主要用来存储元数据，实现类为`LocalRaftMetaStorage`，这个实现使用了本地文件存储元数据信息，其中本地文件为 protobuf 格式，其定义为
```protobuf
message StablePBMeta {
    required int64 term = 1;
    required string votedfor = 2;
};
```
保存元数据的方法为`com.alipay.sofa.jraft.storage.impl.LocalRaftMetaStorage#save`。在`com.alipay.sofa.jraft.core.NodeImpl#initMetaStorage`方法初始化 MetaStorage 后，会从中读取`currTerm`和`votedId`两个属性。

### FSMCaller 的实现

FSMCaller 是调用状态机的实现，由于状态机是有使用 JRaft 这个库的使用方来实现的，FSMCaller 封装了对状态机接口（`com.alipay.sofa.jraft.StateMachine`）的调用

### 日志复制
在 JRaft 的实现中，多个地方使用了 Disruptor 的 RingBuffer 来处理 LogEntry，这使得在阅读代码的过程中异常困难。

#### 心跳发送
每个 follower 注册到 leader 中以后，在 leader 中会有新建一个 Replicator 对象来标识该 follower，代码参见`com.alipay.sofa.jraft.core.ReplicatorGroupImpl#addReplicator`。当 Replicator 创建完成以后，会启动用于发送心跳的定时器，`com.alipay.sofa.jraft.core.Replicator#startHeartbeatTimer`，当到了心跳发送时间，就会调用`com.alipay.sofa.jraft.core.Replicator#onTimeout`发送心跳，其发送心跳的入口在`com.alipay.sofa.jraft.core.Replicator#onError`。

#### 日志发送

当 leader 收到客户端发送的 command 后，会包装为 Task 对象进入到`com.alipay.sofa.jraft.core.NodeImpl#apply`进行处理，然后会转换为 `LogEntry`发布到 Disruptor 的 RingBuffer 队列中，再由`com.alipay.sofa.jraft.core.NodeImpl.LogEntryAndClosureHandler#onEvent`在队列的消费端进行处理，每个`LogEntryAndClosure`对象中都会持有，响应 RPC 的回调`Closure`对象。在消费端会将到达的任务放在集合中进行批量处理，具体的执行方法为`com.alipay.sofa.jraft.core.NodeImpl#executeApplyingTasks`，接下来这些 logentry 就会交给 LogManager 了。在`com.alipay.sofa.jraft.storage.impl.LogManagerImpl#appendEntries`中会继续将 LogEntry集合放入另一个 Disruptor 的 RingBuffer 中，由`com.alipay.sofa.jraft.storage.impl.LogManagerImpl.StableClosureEventHandler`在队列的消费端处理。这里则会专注于日志的持久化处理，这个过程会携带`LeaderStableClosure`进行异步的日志提交，同时在将日志异步发布到 Disruptor 队列后，`LogManagerImpl`还会唤醒在等待的 Replicator 进行日志的复制，代码在`com.alipay.sofa.jraft.storage.impl.LogManagerImpl#wakeupAllWaiter `。

为什么是唤醒等待的 Replicator 呢？因为在正常流程中，Replicator 初始化时，会发送一个探测请求（probe 包）给 follower，在`com.alipay.sofa.jraft.core.Replicator#sendEmptyEntries(boolean, com.alipay.sofa.jraft.rpc.RpcResponseClosure<com.alipay.sofa.jraft.rpc.RpcRequests.AppendEntriesResponse>)`这里会发送 probe 包，完成后，会调用`com.alipay.sofa.jraft.core.Replicator#sendEntries()`，而这里会通过`com.alipay.sofa.jraft.core.Replicator#waitMoreEntries`将每个 Replicator 放入等待队列中，进而进入到`com.alipay.sofa.jraft.storage.impl.LogManagerImpl#wait`等待并发送 LogEntry。

需要注意的是对客户端的响应事件，因为在 JRaft 处理客户端的请求会再多个异步队列中传递，所以响应客户端的时间会放在回调事件中处理，在处理过程中`com.alipay.sofa.jraft.core.NodeImpl#executeApplyingTasks`里调用`com.alipay.sofa.jraft.core.BallotBox#appendPendingTask`会将回调时间注册到回调队列，在 JRaft commit log entry 的时候会遍历回调队列，进行客户端的响应。

`com.alipay.sofa.jraft.core.Replicator#sendEntries(long)` 这是正式发送 LogEntry 的地方.

`com.alipay.sofa.jraft.core.Replicator#onAppendEntriesReturned` 这里会提交日志，会调用到`com.alipay.sofa.jraft.core.BallotBox#commitAt`提交，这个方法中会计算 quorum 是否小于等于 0

### Snapshot 的实现


### Reference

* [分布式一致性 Raft 与 JRaft](https://www.sofastack.tech/projects/sofa-jraft/consistency-raft-jraft/)
* [Raft算法原理](https://www.codedump.info/post/20180921-raft/)
* [etcd Raft库解析](https://www.codedump.info/post/20180922-etcd-raft/)
* [RAFT介绍](https://github.com/baidu/braft/blob/master/docs/cn/raft_protocol.md)
* [The Raft website](https://raft.github.io/)
* [Diego Ongaro's Ph.D. dissertation](https://github.com/ongardie/dissertation)
* [Raft paper](https://raft.github.io/raft.pdf)
