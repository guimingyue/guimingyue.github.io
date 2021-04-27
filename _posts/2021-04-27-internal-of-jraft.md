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

## 实现

### Leader 选举

选主有一个单独的定时器 `electionTimer`，在定时器开始后会开始选主

voteTimer 是作为选主的调度器，在选自己为主时开启（`electSelf`方法），在选主完成之后结束。

### LogStorage 的实现

LogStorage 作为 LogManager 持久化日志的存储实现，核心逻辑在 LogManagerImpl 的代码中。

### MetaStorage 的实现

MetaStorage 主要是介绍元数据如何存储的，meta 中写入了哪些数据，在什么场景下会写入以及最终的写入的流程。

### FSMCaller 的实现

FSMCaller 是调用状态机的实现，由于状态机是有使用 JRaft 这个库的使用方来实现的，FSMCaller 封装了对状态机接口（`com.alipay.sofa.jraft.StateMachine`）的调用

### Snapshot 的实现


### 日志复制
使用了 RingBuffer 来处理 LogEntry

com.alipay.sofa.jraft.core.Replicator#onError  heartbeat

如何发送 heartbeat 消息：Replicator.heartbeatTimer 会再 com.alipay.sofa.jraft.core.Replicator#startHeartbeatTimer 设置超时的设置，定时器超时后，会调用 com.alipay.sofa.jraft.core.Replicator#onTimeout 最终会调用到 Replicator.onError


com.alipay.sofa.jraft.core.Replicator#sendEntries(long) 这是正式发送 log entry 的地方

com.alipay.sofa.jraft.core.Replicator#sendEmptyEntries(boolean, com.alipay.sofa.jraft.rpc.RpcResponseClosure<com.alipay.sofa.jraft.rpc.RpcRequests.AppendEntriesResponse>) 
这里会发送 probe 包，完成后，会调用 com.alipay.sofa.jraft.core.Replicator#sendEntries() 等待并发送 log entry

com.alipay.sofa.jraft.storage.impl.LogManagerImpl#wait 进入到这个方法后，在 LogManagerImpl 中会将 Wait 加入到 com.alipay.sofa.jraft.storage.impl.LogManagerImpl#waitMap 中，而这个 waitMap 会在 com.alipay.sofa.jraft.storage.impl.LogManagerImpl#wakeupAllWaiter 被唤醒，再次调用在 com.alipay.sofa.jraft.core.Replicator#waitMoreEntries 设置的回调 (arg, errorCode) -> continueSending((ThreadId) arg, errorCode)


com.alipay.sofa.jraft.core.NodeImpl#executeApplyingTaskswakeupAllWaiter 在 com.alipay.sofa.jraft.storage.impl.LogManagerImpl#appendEntries 中被调用到，这里则会被  com.alipay.sofa.jraft.core.NodeImpl#executeApplyingTasks 调用到，这里的调用路径就是 com.alipay.sofa.jraft.core.NodeImpl.LogEntryAndClosureHandler#onEvent <-  com.alipay.sofa.jraft.core.NodeImpl#apply

对客户端的响应事件会注册到回调队列里，com.alipay.sofa.jraft.core.NodeImpl#executeApplyingTasks 里调用 com.alipay.sofa.jraft.core.BallotBox#appendPendingTask 注册回调队列，在 commit 的时候会遍历回调队列。

com.alipay.sofa.jraft.core.Replicator#waitMoreEntries 会将每个 Replicator 放入等待队列中

com.alipay.sofa.jraft.core.Replicator#onAppendEntriesReturned 这里会提交日志，会调用到 com.alipay.sofa.jraft.core.BallotBox#commitAt 提交，这个方法中会计算 quorum 是否小于等于 0



#### Leader 日志复制

#### follower 日志复制


### Reference

* [分布式一致性 Raft 与 JRaft](https://www.sofastack.tech/projects/sofa-jraft/consistency-raft-jraft/)
* [Raft算法原理](https://www.codedump.info/post/20180921-raft/)
* [The Raft website](https://raft.github.io/)
* [Diego Ongaro's Ph.D. dissertation](https://github.com/ongardie/dissertation)
* [Raft paper](https://raft.github.io/raft.pdf)