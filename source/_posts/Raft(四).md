---
title: Raft(四) safety
tag: consenus
categories: distributed
---


从安全性角度来看，前面的机制不够完善。
1. 选举限制
2. 推迟提交
<!-- more -->

## 安全性
>安全性的定义：如果有任何的服务器节点已经应用了一个确定的日志条目到它的状态机中，那么其他服务器节点不能在同一个日志索引位置应用一个不同的指令。

## 矛盾抛出
**我们按照之前的领导选举和日志复制的规则看下面一个图**

![](http://op7scj9he.bkt.clouddn.com/raft4.png)

1. 在term1过后，进入term2，S1经过投票选举为了leader，复制client传来的任期2的entries。
2. 此时{index:2,term:2}只复制到了S2，S1就崩掉了，此时S5上位（S3、S4、S5三票），此时term为3，收到了来自client的请求。
3. 过了一会处于term3的leader(S5)崩掉了，S1在恢复后，重新成为leader(term4)，收到client的新请求(index：3，term：4)，复制过去{term：2，index：2}的entry到S3，此时按照过去的committed标准（只要leader知道该条日志存在大多数副本的日志里），index为2的这条entry被认为可提交，之后S1崩溃了。
4. S5当选leader，进行{term:3,index:2}的复制，此时进行日志校验，发现只有index：1一致，于是从此之后的{term:2,index:2}被覆盖为{term:3,index:2}

>出现的问题：已经被认为committed的entry被后面新的leader覆盖掉，导致**不安全！**


## 完善协议（一）选举限制

### 前景再现
1. 问题描述：新的领导人不知道过去老的日志条目是否是committed的状态，而被覆盖后，导致状态机不一致的情况。

 - committed entries：最终被应用到状态机

2. 那么我们希望**新的领导人也都知道之前已提交过的日志！**这需要在领导选举上重新下功夫。

### 分析改善

#### 分析
1. 之前的选举方案：candidate通过在一个随机的election_timeout中，获取大多数的票数成为leader。
2. **加上限制：成为领导人的候选者存在的log是最"新"的，那么它最有可能包含所有提交的日志条目。**

#### 具体细节
![](http://op7scj9he.bkt.clouddn.com/raft5.png)
1. 重点关注lastLogIndex 、lastLogTerm参数。

- lastLogIndex 候选人的最后日志条目的索引值
- lastLogTerm 候选人最后日志条目的任期号

情景假设：投票选举阶段，candidateA向节点B发送了投票请求，如果B觉得A不够"新"，则拒绝！(这里讨论的情景是 term>= currentTerm)
``if(A.lastLogTerm<B.lastLogTerm || A.lastLogTerm == B.lastLogTerm && A.lastLogIndex< B.lastLogIndex)``

在原来得到众多派的基础上加上两条规则限制：
1. 首先比较日志最后的条目任期，任期大的 新！
2. 在任期相同的情况下，比谁的任期收到的日志条目多，条目多的新！

### 用完善后的选举算法再次看待
![](http://op7scj9he.bkt.clouddn.com/raft6.png)

- 假设此时S1在term为4的时期，并没有崩溃，还是leader（忽略第4块）
(c). S1将{term:2,index:2}复制给S3
(e). S1复制{term:4,index:3}给S2、S3，崩掉了，选取term5的leader

- 按照之前的选举：S5运气好，在term5成为了leader，那么接下来做的事情，就是覆盖掉index为2的entry，和删除掉index为3的entry。
- 按照加了限制后的选举：S5的最后一项任期值是3，比其他节点的最后一项小，无法成为leader，term5的leader只有可能从S2和S3中选出。

## 完善协议（二）推迟提交
### 情景再现
(c->d)S1在term4提交了一个term2的日志，被term5的leader（S5）覆盖
结果：一条已经被存储到大多数节点上的老日志条目，也依然有可能会被未来的领导人覆盖掉

### 之前是如何定义可提交
- committed entries：存储在大部分服务中的日志中，最终被应用到状态机

- 只有leader可以决定committed

### 修改
思考：之前是通过**多数派协议,由leader决定**是否committed

>推迟提交：Raft 永远不会通过计算副本数目的方式去提交一个之前任期内的日志条目。**只有领导人当前任期里的日志条目通过计算副本数目可以被提交**

分析：

- 存储在大部分server里
- 至少存在一个最新term的log entry也同时被存储在大部分server里
- 一旦当前任期的日志条目以这种方式被提交，那么由于日志匹配特性，之前的日志条目也都会被间接的提交

### 情景最终分析
结合选举限制和延迟提交来重新看一下这个情景
![](http://op7scj9he.bkt.clouddn.com/raft6.png)

- 如图(c):**在term4时期的leader（S1）无法判断{index：2,term：2}committed**
- 那么崩溃后term5，由S5成为leader覆盖掉非committed状态的日志，属于安全操作。