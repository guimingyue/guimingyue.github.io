---
layout: post
title: Raft 论文阅读笔记
category: Raft
---

## 1. 介绍

由于一致性算法可以让一个集群在有部分机器宕机的情况下继续工作，所以它在构建可靠的大型软件系统方面扮演着重要的角色。Paxos 算法在过去十年一直是一致性算法中的焦点，大部分的一致性算法的实现都是基于 Paxos 的，并且 Paxos 算法已经成为了教授一致性算法的主要的工具。但是 Paxos 同样以难以理解著称。
通过学习和参考 Paxos 算法，我们设计了 Raft 算法，它比 Paxos 更容易理解和实现。Raft 算法主要有一下特性：

* Strong leader，Raft 集群中只有一个主节点，所有的日志都是从主节点流向其他节点。
* Leader election，Raft 使用随机的计时器来选主，这种机制在心跳的基础上增加了少量的复杂度，但是它能更简单和迅速的解决冲突。
* Membership changes，Raft 使用了一种称为 joint consensus 的方法来改变集群节点，`where the majorities of
two different configurations overlap during transitions`。这使得整个集群在配置改变的过程中能继续工作。

## 2. 复制状态机

一致性算法伴随着复制状态机出现，这个算法中，状态机在一组服务器上计算相同的副本的同一个状态，并且能够在部分机器宕机的情况下继续工作。复制状态机用于解决分布式系统中的各种容错问题。比如，像 GFS，HDFS，RAMCloud 这样的大型系统中，都有一个主节点的集群，它们使用一个单独的复制状态机来处理选主和存储配置信息，以防止主节点宕机。这样的复制状态机包括 Chubby 和 ZooKeeper。

典型的复制状态机使用复制日志来实现，如图1 所示。每个节点存储的日志包含一些列的命令，每个节点上的日志包含相同的命令，并且命令的顺序也相同，所以每个状态机处理相同序列的命令。

![figure 1 replicated state machine](/images/raft_paper_notes/figure1-replicated-state-machine.png)

一致性算法的作用是使得复制日志保持一致性。服务节点上的一致性模块从客户端接收命令，然后将命令添加到它的日志中。服务节点上的一致性模块它与其他服务节点上的的一致性模块通信，并且保证日志最终包含相同的顺序的请求，即使部分节点宕机。一旦命令顺利复制了，每个服务节点上的状态机都能按照日志的顺序处理他们，最终将结果返回给客户端。所以，这些服务节点表现的像一个单一的，高可用的状态机。


实际的应用中的一致性算法主要有以下特性：

* 安全性，不会返回错误的结果。
* 可用性，只要保持大多数服务节点正常。
* 不依赖时间来保持一致性。
* 一旦多数的节点完成命令执行，少数的节点失败不影响整体性能。


## 3. Paxos 存在什么问题？

在过去的十年中，Paxos 协议成为了一致性的代名词，它首先定义了一个能够在单个提议上达成一致决定的协议，比如单条日志复制，这也称为`single-decree Paxos`。然后，Paxos 又定义了能够对连续的多个提议达成一致的版本，称为 `multi-Paxos`。Paxos 协议的正确性在理论上是成立的，它保证了集群节点的安全性和存活性，并且支持集群节点的变更。

但是 Paxos 有两个比较明显的缺点，首先是它以难以理解著称，其次是它并没有考虑如何来实现这个协议，比如大家对`multi-Paxos`算法，并没有一致的结论，Paxos 协议的作者主要在介绍`single-decree Paxos`，对`multi-Paxos`只是做了大致的描述。

基于 Paxos 的问题，我们觉得 Paxos 并没有为系统实现和教授提供一个好的基础，考虑到一致性在大型分布式软件系统中的重要性，我们设计了 Raft 协议。

## 4. 考虑可理解性的协议设计

新协议最重要的目标和最难的挑战是可理解性，它必须对大多数人来说是易懂的。

## 5. Raft 一致性算法

Raft 算法是用来管理日志复制的（第 2 部分描述）。该算法首先会选举一个主节点，然后由主节点来管理复制日志，基于这样的策略，它实现了一致性。

主节点从客户端接收日志（命令），再复制到其他节点上，其他节点能放心的将日志应用于状态机中。设计一个主节点能简化日志的复制。比如，主节点能决定如何应用日志，而不用依赖其他节点，并且数据流向只能是从主节点到其他节点。主节点宕机了，会再发起一个选择新主节点的流程。基于主节点的设计，Raft 将一致性问题划分成了三个相对独立的子问题：

* Leader election，主节点选举。
* Log replication，日志复制，由主节点从客户端接收日志，然后复制到其他节点。
* Safety，简单点理解是，如果一个节点对决策问题的一个提议达成了一致，并且应用到了状态机，那么不会接收这次决策问题的其他提议。

### 5.1 Raft 基础

一个 Raft 集群包含多个机器，典型的是 5 个，这样就能够承受有两个节点宕机的风险。在任何时间点一个节点只能是`leader`，`follower`和 `candidate`这三种状态中的一个，并且只会有不超过一个 `leader`。`follower`只能从`leader`接收请求并响应。

Raft 将时间划分为 term。Term 由一组连续的整数进行编号，每个 Term 开始时会进行选主，在这个过程中一个或多个 candidate 会尝试变成主节点。在某些可能的场景下，一次选主可能不会达成一致，那么这样这个 Term 就不会有主节点产生。Raft 会确保在任一 Term，最多只会有一个主节点。在 Raft 中Term 是一个逻辑时钟，节点会根据 Term 检查收到的信息是否过期，比如主节点是否已经过期。每个节点都会维持一个单调递增的 `current term` 的值，节点在沟通过程中会传递这个值，如果一个节点的 `current term`  小于（与之交流的）另一个节点，那么它会其更新为更大的值，如果一个`candidate`或`leader`发现它的 Term 已经过期了，他会立马将其转变为`follower`状。如果一个节点接收到了带有过期的 term 的请求，它会拒绝这个请求。Raft 节点之间的交流是通过 RPC 进行的，主要两种类型的 RPC，Request Vote RPC 和 Append Entries RPC。

### 5.2 Leader election

Raft 使用心跳机制来触发 Leader election。每个节点启动时处于 follower 状态，并且一直处于 follower 状态，只要他能从 leader 收到心跳请求，否则 follower 会发起新的 leader election，从最后一次收到 leader 请求到发起新的选举的时间称为 election timeout。为了发起 leader election，follower 节点会先增加 Term 并且将节点状态从 follower 改成 candidate，然后并行的向集群中的其他节点发起投票请求（RequestVote RPC）。candidate 在以下两种情况下会改变其状态。

a. 赢得了选举，成为 leader。即获得了大多数的投票（大于等于 2/n + 1)。新的 leader 节点会给其他节点发送心跳来声明其 Leader 角色。
b. 另一个 candidate 节点称为了 leader 节点，该节点变为 follower。这种情况下 candidate 节点会接收到新的主节点的消息，该消息的 Term 不小于当前 candidate 的 Term。

另外一种情况是，在该一个 Term 周期内，没有节点成为 leader，那么 candidate 节点会增加 Term 并且开始新一次的 Leader election。Raft 使用了随机的 election timeout 来防止下一次仍然没有 leader 选举成功。

### 5.3 Log Replication

当 Leader 接收到客户端命令后，会将命令包装为 log entry 增加到日志条目中，再并行复制给其他（follower）节点，复制给大多数节点（n/2 + 1）成功以后，Leader 会 commit 该 log entry，也就是执行到状态机中。如果 follower 节点宕机或者运行比较慢，Leader 会一直重试复制请求直到所有的 follower 都存储了相同的 log entry。

如下图所示，每个 Log Entry 包含客户端请求的命令和 Leader 接收到客户端命令的 Term 数值，该值会被用于检查 Leader 和 Follower 直接的日志是否一致。每条日志还会包含一个索引（index）数值确定在日志列表中的位置。
![figure 6 commited entries](/images/raft_paper_notes/figure6-committed_entries.png)

Leader 节点负责确定日志应用到状态机的时机，该过程也称为 `committed`。Raft 会保证提交的日志条目已持久化，并且最终被状态机执行。当主节点（leader）将日志复制到多数节点之后，会将其提交，如上图所示的索引为 7 的 log entry，该提交过程也会将当前日志之前的日志提交，即使之前的日志是由其他 leader 创建的。一旦 follower 感知到一个 log entry 已经提交了，它也会将其应用到自己的状态机。
Raft 维护这下面两条日志复制的规则。

* 如果在不同节点上的两条 log entry 有相同的索引（index）和 Term，那么他们存储着相同的（客户端发起的）命令。
* 如果在不同节点上的两条 log entry 有相同的索引（index）和 Term，那么这两个节点上在该条 log entry 之前的日志是都相同的。

正常情况下，leader 和 follower 的日志是一致的，所以检查一致性的`AppendEntries`请求不会失败，但是 leader 的失败会导致日志的不一致，下图展示了各种不一致的场景。在 Raft 中，leader 会强制 follower 复制自己的日志，也就是说 follower 中冲突的日志日志会最终被 leader 中相同位置的日志覆盖。这个过程会再一致性检查中完成，RPC 请求为 AppendEntries 类型。leader 为每个 follower 维护着 `nextIndex` 的索引变量，它表示要复制到 follower 的下一个 log entry 的索引（index），初始情况下，leader 会将每个 follower 的`nextIndex`置位当前日志中的最后一个日志的下一个索引（下图中的索引 11），如果 follower 的的日志与 leader 不一致，下一次的一致性检查请求 AppendEntries 就会失败，这样 leader 就会将该 follower 的`nextIndex`减小，然后再执行一致性检查，直到leader 与 follower 的`nextIndex`的之前的日志一致，此时，AppendEntries 请求就会成功，也意味着 follower 中冲突日志会被删除。这样就可以继续执行日志复制的 AppendEntries 请求了。

![figure 7 ](/images/raft_paper_notes/figure7-log-entries-of-leader.png)

### 5.4 Safty

上面介绍的 leader election 和日志复制这些机制不足以保证每个状态机都按照相同的顺序执行同样的命令，比如 leader 在提交日志时，有个 follower 不可用，然后这个 follower 又被选为了新的 leader 节点，那么已经提交的日志可能被覆盖掉。需要增加一些限制规则来保证命令应用到状态机的正确性。也就是 leader 在某个 Term 一定会包含已经提交的所有日志。

#### 5.4.1 Election 限制

部分一致性算法在主切换后，会通过额外的机制来识别新主缺失的日志条目，然后将这些缺失的日志条目传输到新主上。Raft 选择了更简单的方案，Raft 在选举过程中，只有包含最新的 log entry 能够选为 leader，这样 leader 就不需要覆盖任何日志。所以为了被选为新 leader，candidate 必须与集群中的多数节点沟通，这就意味着每条 committed 的日志都必须出现在至少一个 candidate 中。如果一个 candidate 的日志与大多数节点比较后是最新的，那么它就会包含素有已提交的 log entry。这个沟通通过 RequestVote Rpc 实现，如果被调用方的日志比调用方（candidate）更新，那么这个选主请求就会被否定掉。Raft 节点间比较日志是通过 index 和 Term 比较的，更大的 Term 更新，相同的 Term，更大的 index 就更新。

#### 5.4.2 最后提交的日志之前的日志提交

在 log entry 被复制到多数节点后，leader 会提交当前 Term，如果在 leader 提交之前，leader 节点宕机了，那新的主会继续未完成的复制和提交操作。但是 leader 无法推立即推断出之前的 log entry 是否有提交，即使它已经被复制到了大多数节点中。
Raft 仅仅会针对当前的 Term 的 log entry，依赖复制的副本数来提交，当前 Term 之前的 log entry 不会根据复制的副本数来确定是否提交，但是根据`Log Matching Property`，一旦某个 log entry 提交了，那么它之前的 log entry 也就提交了。

#### 5.4.3 安全性论证

先假设 Leader Completeness Property 不成立，然后通过提交日志和重新选主来推导出矛盾，从而证明 Raft 的安全性。

### 5.5 follower 和 cadidate 宕机

Raft 的 RequestVote 和 AppendEntries RPC 调用是幂等的，所以如果 follower 和 candidate 宕机，那么直接重试即可。

### 5.6 Timing and availability

Raft 的安全性不能依赖时序来保证，也就是不能因为部分事件发生的比预期快或者慢就导致结果不正确。但是可用性无法避免的会依赖时序，特别是 Leader election，所以 Raft 需要时序满足如下不等式。

```sql
broadcastTime << electionTimeout << MTBF

broadcastTime：节点并行调用其他节点并收到返回的时间；
electionTimeout：选举超时时间
MTBF：节点两次宕机时间间隔
```

由于Raft RPC 需要请求接收存储信息，所以 broadcastTime 在 0.5ms 到 20ms 之间，electionTimeout 在 10ms 到 500ms 之间。MTBF 比较大，几乎可以不考虑。

## 6. 集群节点变更

为了保证节点变更的安全性，Raft 的节点配置变更使用了两阶段的方法，首先切换到转换过度的`joint consensus`状态，然后再转换到最终的状态。在状态转换的整个过程中，Raft 集群可以支持客户端的请求。`joint consensus`包含新旧两种节点配置：
* log entry 会复制到两种配置中。
* 新旧两种配置都会存在与 leader 节点中。
* 一致性需要再新旧两种配置中达成多数规则。


集群配置的存储和传输是通过特定的 log entry 实现的，并且是以复制日志的形式传输。下图展示了配置变更的过程。当 leader 收到了将配置从 C<sub>old</sub>变更为C<sub>new</sub>的请求时，会在将配置变更存储为一个 log entry 并且复制到其他节点，这条 log entry 用 C<sub>old</sub>,<sub>new</sub>表示。一旦一个节点将收到的配置变更的 log entry 添加到日志队列中，它将在后续使用该配置，不管该 log entry 是否有提交。当 C<sub>old</sub>,<sub>new</sub> 被 leader 提交后，Leader 会再提交一个 C<sub>new</sub> 的 log entry，复制到其他节点，复制完成以后，旧的配置的节点就可以下线了。
![figure 7 ](/images/raft_paper_notes/figure11-timeline-joint-consensus.png)

## 7. Log compaction








