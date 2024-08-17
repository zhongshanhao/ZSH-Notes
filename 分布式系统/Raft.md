---
title: Raft分布式一致性协议简述
date: 2021-07-22
categories:
 -  分布式系统
---

### Raft简介

Raft 就是用来管理复制日志（replicated log）的一致性协议。一致性算法允许多台机器作为一个集群协同工作，并且在其中的某几台机器出故障时集群仍然能正常工作。它是如何做到的？

### 复制状态机

复制状态机简单来说就是说多个不同的机器，运行着相同的代码，如果每个机器处理的输入序列一样，那么它们的输出序列也一样，那么，在其中一个机器宕机时，那么其他机器就可以代替该机器处理请求，一致性算法就是在复制状态机的背景下产生的。

复制状态机通常使用复制日志实现，如下图所示。每个服务器存储一个包含一系列命令的日志，其状态机按顺序执行日志中的命令。 每个日志中命令都相同并且顺序也一样，因此每个状态机处理相同的命令序列。 这样就能得到相同的状态和相同的输出序列。

<img src="https://gitee.com/Krains/FigureBed/raw/master/img/%E5%A4%8D%E5%88%B6%E7%8A%B6%E6%80%81%E6%9C%BA.png" alt="复制状态机" style="zoom: 50%;" align="middle"/>

### 简述Raft的工作机制

在Raft所处的集群中，每个节点可以有三个状态

- follower：被动接受 leader 的请求，在timeout时间内没有收到心跳则会进入到 candidate 状态
- candidate：该状态下会尝试竞选 leader ，发出投票请求给其他节点，达到半数同意就晋升为 leader 
- leader ： leader 处理所有来自客户端的请求，同步日志到各个 follower，周期性地发送心跳避免 follower 进入 candidate

Raft状态转移图

<img src="https://gitee.com/Krains/FigureBed/raw/master/img/%E7%8A%B6%E6%80%81%E5%8F%98%E5%8C%96.png" alt="状态变化" style="zoom: 67%;" align="middle"/>

在Raft中，有个两个比较重要的概念

- timeout：如果 follower 没有在timeout时间内从 leader 处得到消息，那么它将会成为 candidate。timeout被随机为150-300ms。
- term：当前是第几任 leader，如果集群内出现了多个 leader，那么 term 最大的 leader 是真正的 leader

leader 选举

一开始所有节点都处于 follower 状态，如果 follower 没有在 timeout 时间内从 leader 处得到消息，那么它将会成为 candidate， candidate 给其他节点发出投票请求，节点会选择最先到达的投票请求，把它的票投给该请求的发起者，一个节点只能投一次票，如果

- 其中一个 candidate 得到了超过半数的投票，那么该 candidate 将会成为 leader，并持续向 follower 发送心跳
- 其中两个 candidate 得到了相同的票数，那么两个 candidate 将重新计一个 timeout

如果 leader 挂了，那么节点在 timeout 时间之后，会成为 candidate，竞选 leader，收到半数的投票的节点将会成为下一任 leader

日志复制

当 leader 产生，所有的对系统改变的请求将会通过 leader 处理。

- 客户端的请求到达 leader ， leader 将会记录日志，但是未提交给状态机
- leader 将信息发给 follower，follower 收到请求之后返回确认信息给 leader
- leader 接收到半数以上的请求之后提交更新（应用到状态机中，得到输出），给客户端做出响应，然后发送请求给 follower 让它们也提交更新

网络分区故障

<img src="https://gitee.com/Krains/FigureBed/raw/master/img/image-20210325191614711.png" alt="image-20210325191614711" style="zoom: 50%;" align="middle"/>

如果集群遭遇网络分区故障，划分为两个分区，节点B是原来的 leader ，节点C会成为新的 leader ，如果此时客户端的请求

- 打在了B上，那么B记录日志之后将消息发送给 follower，但是发现没有收到半数的响应，所以这两个节点将不会提交更新
- 打在了C上，上半分区还是能够正常进行日志复制操作

当网络分区故障解除， leader B发现Term更高的 leader ，于是自己变成 follower，回滚之前未提交的操作，并同步日志，保持与 leader 的数据一致性。

很好的一个使用动图描述 Raft 工作机制的网站：[Raft](http://thesecretlivesofdata.com/raft/)

### 一致性问题

简单介绍 Raft 工作机制后，我们来看看在遇到网络故障、节点崩溃等等之后，Raft是如何保证数据一致性的。

在 Raft 中，只要集群内任何过半的服务器正常运行，就能够保证复制日志的一致性，它能够确保在网络延迟、分区和数据包丢失、重复和乱序的安全性(不会向状态机提交不一样的结果)。

> 安全性：if a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index.

Raft 协议通过选举一个 leader 的方式，将一致性问题分为三个相对独立的子问题

-  leader 选举：当 leader 宕机时，一个新的 leader 会被选举出来，**过半投票机制**保证选举 leader 的唯一性
- 日志复制： leader 必须从客户端接受日志，然后复制到集群中的其他节点，通过**一致性检查机制**要求其他节点和自己保持一致
- 安全性：所有的状态机都会应用相同索引下的相同指令

在集群正常运行的过程中， leader 会和 follower 的日志保持一致。在下图中展示 leader 或 follower 崩溃之后日志发生不一致的现象：

<img src="https://gitee.com/Krains/FigureBed/raw/master/img/%E6%97%A5%E5%BF%97%E4%B8%8D%E4%B8%80%E8%87%B4.png" alt="日志不一致" style="zoom: 67%;" align="middle"/>

> 当一个 leader 成功当选时(最上面那条日志), follower 可能是(a-f)中的任何情况。每一个盒子就表示一个日志条目，里边的数字是任期号。follower 可能会缺少一些日志条目（a-b），可能会有一些未被提交的日志条目（c-d），或者两种情况都存在（e-f）。例如，场景 f 可能这样发生，f 对应的服务器在任期 2 的时候是 leader ，追加了一些日志条目到自己的日志中，一条都还没提交（commit）就崩溃了；该服务器很快重启，在任期 3 重新被选为 leader，又追加了一些日志条目到自己的日志中；在这些任期 2 和任期 3 中的日志都还没被提交之前，该服务器又宕机了，并且在接下来的几个任期里一直处于宕机状态。

在这些情况下， leader 将通过**一致性检查机制**保证日志一致性， leader 通过AppendEntries RPC将日志发送给follower。

> 一致性检查机制：在发送 AppendEntries RPC 的时候，leader 会将前一个日志条目的索引位置和任期号包含在里面。如果 follower 在它的日志中找不到包含相同索引位置和任期号的条目，那么他就会拒绝该新的日志条目。

要使得 follower 的日志跟自己一致，leader 必须找到两者达成一致的最大的日志条目（索引最大），删除 follower 日志中从那个点之后的所有日志条目，并且将自己从那个点之后的所有日志条目发送给 follower 

日志复制的过程如下

-  leader 针对每一个 follower 都维护了一个 nextIndex ，表示 leader 要发送给 follower 的下一个日志条目的索引。当选出一个新  leader 时，该 leader 将所有 nextIndex 的值都初始化为自己最后一个日志条目的 index 加1
- 如果 follower 的日志和 leader 的不一致，那么下一次 AppendEntries RPC 中的一致性检查就会失败
- 在被 follower 拒绝之后， leader 就会减小 nextIndex 值并重试 AppendEntries RPC 
- 最终 nextIndex 会在某个位置使得 leader 和 follower 的日志达成一致。此时，AppendEntries RPC 就会成功，将 follower 中跟 leader 冲突的日志条目全部删除然后追加 leader 中的日志条目（如果有需要追加的日志条目的话）
- 一旦 AppendEntries RPC 成功，follower 的日志就和 leader 一致，并且在该任期接下来的时间里保持一致

当然，在一致性检查机制中，需要保证 leader 必须存储所有已经提交的日志条目，已经提交就意味着有半数服务器已经将最新日志复制了，因此，在 candidate 投票过程中，需要增加额外的限制以保证新 leader 拥有所有已经提交的日志，Raft 通过比较两份日志中最后一条日志条目的索引值和任期号来定义谁的日志比较新。如果两份日志最后条目的任期号不同，那么任期号大的日志更新。如果两份日志最后条目的任期号相同，那么日志较长的那个更新。由于半数票选机制，最终新任的 leader 必然会包含所有已经提交的所有日志条目。

**参考资料**

- [In Search of an Understandable Consensus Algorithm (Extended Version)](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)
- [Raft动图描述](http://thesecretlivesofdata.com/raft/)