---
title: Raft(一)
tag: consenus
categories: distributed
---

Raft共识算法初体验
<!--more-->
## Raft干嘛的
跟Paxos差不多的，是一个**共识算法**。
### Raft的初衷
- 尽管有很多工作都在尝试降低它的复杂性，但是 Paxos  算法依然十分难以理解
- 提供了和 Paxos 算法相同的功能和性能，但是它的算法结构和 Paxos 不同
[Basic Paxos](https://lqsss.github.io/2018/05/28/paxos/)

### 共识算法(consenus)
1. 分布式系统的容错机制？
多个数据副本(replication) 起到一个数据冗余的作用。
2. 共识算法解决一个什么样的问题？
state machine replication的一致性。状态机上存储的可以是log，value，或者指令。
3. 最终一致性和强一致性？
分布式里的一致性(consistency)话题是一个老生常谈的问题，但是这个一致性并不能笼统地说明此处的一致性，对于数据副本，我们最好使用共识（consenus）来形容。
- 最终一致性指的是在不同的副本在最终达成相同的状态。
这里的强一致性对于client的角度来看，向系统写入什么，读出来的也会是什么。
- 当分布式系统中更新操作完成之后，任何多个进程或线程，访问系统都会获得最新的值。
4. 2pc和Raft解决的是一个问题吗？
不是。2pc解决的是一个分布式事务的一致性问题，而Raft解决的是分布式里副本的一致性，起到一个集群容灾的能力。比如存储系统经常做分片(partition)
[请问分布式事务一致性与raft或paxos协议解决的一致性问题是同一回事吗？](https://www.zhihu.com/question/275845393/answer/383419685)
[TiDB使用了raft之后为什么还需要2PC?](https://www.zhihu.com/question/266759495)

## Raft核心模块
1. 领导选举
2. 日志复制
3. 集群变化

## Raft一些相关资源
[Raft动画](http://thesecretlivesofdata.com/raft/)
[Raft :GitHub Pages](https://raft.github.io/)
[给导师汇报时做的ppt](http://op7scj9he.bkt.clouddn.com/Raft.ppt)