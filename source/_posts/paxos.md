---
title: Basic Paxos 
tag: distributed
categories: consenus
---


Paxos共识算法的简单认识（不带证明）
<!-- more -->

## 共识算法(consenus)
1. 分布式系统的容错机制？
多个数据副本(replication) 起到一个数据冗余的作用。
2. 共识算法解决一个什么样的问题？
state machine replication的一致性。状态机上存储的可以是log，value，或者指令。
3. 最终一致性和强一致性？
分布式里的一致性(consistency)话题是一个老生常谈的问题，但是这个一致性并不能笼统地说明此处的一致性，对于数据副本，我们最好使用共识（consenus）来形容。
- 最终一致性指的是在不同的副本在最终达成相同的状态。
- 这里的强一致性对于client的角度来看，向系统写入什么，读出来的也会是什么。
>当分布式系统中更新操作完成之后，任何多个进程或线程，访问系统都会获得最新的值。

[请问分布式事务一致性与raft或paxos协议解决的一致性问题是同一回事吗？](https://www.zhihu.com/question/275845393/answer/383419685)
    
## 角色
1. acceptor:议案接受者，value存储的节点
2. proposer:议案的发起者，接受client发出指令请求，执行一个请求转发
3. learner:议案记录者，通过的议案会被记录下来，并且同步到所有acceptor

## 两个阶段
- prepare阶段：判断proposer的议案候选资格
1. 多个proposer向acceptor发起议案投票请求``Prapare(n)``,这里n指的是议案编号(proposer number)，议案编号唯一。
2. acceptor保存的是之前最大的议案编号(accepted_proposer_num)，以及接受的议案(accepted_value)，某个acceptor收到请求，首先判断传来的议案编号是否大于存储的议案编号，首次acceptor里accepted_proposer_num是null
    2.1 如果大于，则获取acceptor的授权，修改accepted_proposer_num为n，返回accepted_proposer_num和上一个存储的状态值(accepted_value)
    2.2 如果小于，则拒绝proposer的请求。

proposer收到大部分的acceptor的返回，它就有进入第二阶段的权利，并计算其中最大的议案编号所对应的议案内容。

- accept阶段：讨论proposer提出的议案是否通过
1. proposer给第一阶段授权的acceptor发起accept(n,V)请求
    1.1 var为第一阶段计算的议案内容，如果var存在，则此次议案的内容则为var
    1.2 如果var不存在，则议案内容为V，提交上去。
2. acceptor接收proposer的accept请求，首先判断收到的议案号与存储当前最大的议案号的大小关系
    2.1 如果小于，则直接拒绝
    2.2 等于，设置accepted_value为V

**不会出现大于的情况。**

## 图解

### 结构
Acceptor：<accepted_proposer_num,accepted_value>,last_proposer_num(prepare_proposer_num)
prepare请求: {proposer_num}
prepare对应的ACK:{ok,< accepted_proposer_num,accepted_value>}
accept请求:{proposer_num,value}
accept对应的ACK:{ok,< accepted_proposer_num,accepted_value>}

![](http://op7scj9he.bkt.clouddn.com/1.JPG)
- prepare
1. ProposerA发出议案编号为#1的授权请求（希望Acceptor能够投票讨论这个议案）给集群中所有Acceptor
2. 由于是系统启动初期，此时Acceptor的accepted_proposer_num,accepted_value为{null,null}，last_proposer_num此时修改为#1
3. 返回ack{ok，< null,null>}，给予Proposer权利进入第二阶段（投票赋值阶段）
4. ProposerA收到超过N/2+1的ack，确认自己有赋值的权利

--- 

![](http://op7scj9he.bkt.clouddn.com/2.JPG)
- ProposerA的accept阶段
1. ProposerA向Acceptor1发起赋值请求< #1,v1>,Acceptor：< #1,v1>,#1
- ProposerB在ProposerA的accept期间（还未向Acceptor2、3发起赋值请求），进入prepare阶段
1. ProposerB向Acceptor1、2发起议案编号为#2的授权请求
2. Acceptor1发现#2>#1(下文的核心点1),修改last_proposer_num为#2，此时Acceptor1状态为<#1,v1>,#2
3. Acceptor1返回ack{ok，< #1，v1>}，给予ProposerB权利进入第二阶段（投票赋值阶段）
4. Acceptor2返回ProposerB一个ack{ok，< null,null>}，给予ProposerB权利进入第二阶段（投票赋值阶段）
5. ProposerB收到多数派的授权，更新收到的value（核心点3），收到的value：Acceptor1的v1，Acceptor2的null，于是第二阶段要赋值的value为v1；(如果收到的多数派的value为null的话，则第二阶段要赋值的value为自己本身要提交的v2)

---

![](http://op7scj9he.bkt.clouddn.com/3.JPG)

- ProposerB的accept阶段 （ProposerB只会向Acceptor1、2发出请求）
1. 这时Acceptor1、2收到了来自ProposerB的赋值请求< #2,v1>
2. Acceptor1修改状态< #2,v1>,#2,返回ACK{ok,#2,v1}
3. Acceptor2由< null,null>,#2更新为< #2,v1>,#2,返回ACK{ok,#2,v1}
4. ProposerB收到多数派的ACK，赋值成功
- ProposerA的accept阶段
1. 这时Acceptor2、3收到了来自ProposerA的赋值请求< #1,v1>
2. Acceptor2发现收到的请求议案编号为#1<#2(核心点3)，拒绝此次请求
3. Acceptor3由< null,null>,#1更新为< #1,v1>,#1,返回ACK{ok,#1,v1}
4. ProposerB收到多数派的ACK，赋值成功

这里我们看出ProposerA、B都赋值成功，然而并不冲突，整个集群保持了一致的状态（value:v1）。当集群中的大多数保持了一致，就确定了一致，由learn记录这个value，完成整个集群的同步，达到最终一致。
## 核心点
1. acceptor只能接受大于等于当前最大的议案编号，否则拒绝。
2. paxos是一个多数派协议，proposer在第一阶段收到N/2+1个授权，才能进入第二阶段，在第二阶段收到大部分赋值ack，表明大部分的acceptor存储了这个状态，才能算此次议案通过。
3. 已经确认最终保持一致的状态，不会再改变。后面就算存在议案编号更大的议案请求，当它发现大多数acceptor都已经存储了某个具体的value，自然也将此次请求改为{n,value};只有当proposer发现大多数acceptor里的value为null时，才将自己提案中的V对授权过的acceptor发出更新请求{n,V}。

## 参考
[架构设计：系统存储（24）——数据一致性与Paxos算法（中）](https://blog.csdn.net/yinwenjie/article/details/61933266)