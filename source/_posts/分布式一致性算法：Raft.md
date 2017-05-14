---
title: 分布式一致性算法：Raft
date: 2017-05-06 23:53:11
categories: [分布式]
tags: [分布式一致性算法,Raft]
---
<Excerpt in index | 首页摘要>
分布式一致性算法：Raft<!-- more -->
<The rest of contents | 余下全文>
# Raft
## 节点状态
在集群中，每个节点可能具有一下3中状态
- Follower：从节点
- Candidate：候选人节点，Follower->Leader的中间态
- Leader：主节点，在集群中有且只有1个；如果存在多个，则term最大的Leader是合法Leader。
## 一致性协议
### 集群启动
假设存在1个只有3个节点的集群，分别为A、B、C
- <1>A、B、C的初始状态均为Follower
- <2>由于没有Leader，Follower不能收到Leader的心跳。然后Follower会成为Candidate。假设A成为了Candidate，则A会请求B、C投票让A成为Leader（A会给自己投一票，每个节点在一轮投票中只能投票1次）。一个Candidate如果获得了集群**大多数**节点的投票，则成为Leader。此过程成为选主（Leader Selection）。
- <3>所有交互现在通过Leader完成。
> 例如写入操作：

# 引用
[Raft:Understandable Distributed Consensus](http://thesecretlivesofdata.com/raft/)
