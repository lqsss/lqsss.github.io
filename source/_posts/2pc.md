---
title: 2PC
tag: 
- transaction
categories: distributed
---

对cmu440里2pc协议的一个总结
1. 2pc解决一个什么样的问题
2. 2pc的过程
3. 2pc存在的问题
<!-- more -->

## 分布式事务
### 本地事务
我们在单机节点的情况下，保持ACID的事务属性，是通过2PL完成的。
1. 准备阶段
- 获取所需要的所有锁
- 生成需要的更新列表
2. commit/abort阶段
- 一切正常，然后更新全局状态
- 事务不能完成，按原样的全局状态离开
- 在任何一种情况下，RELEASE ALL LOCKS

如果事务牵扯到不同的节点呢？就是我们接下来要谈的2pc协议

### 2pc阶段
1. 准备和投票
  - 参与者找出所有状态变化
  - 每个决定是否可以完成交易
  - 与协调员沟通
2. 提交
  - 协调员向参与者进行广播：COMMIT / ABORT
  - 如果COMMIT，参与者进行相应的状态更改


## 2pc具体细节
![The finite state machine](http://upload-images.jianshu.io/upload_images/5753761-b533dca4253e08d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 简要
1. 分布式中肯定是需要消息沟通（协调员和参与者之间）
2. 上图是2pc协议维护的一个状态机，(a)是协调者（相当于一个事务管理器），(b)是参与者

### 协作者
![iActions by Coordinator](http://upload-images.jianshu.io/upload_images/5753761-c40be07380893ab8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 广播投票请求给参与此次事务的所有参与者，进入WAIT状态阻塞，直到收到全部的参与者的投票响应，如果等待超时，则广播一个全局abort。
2. 如果收到了全部的响应投票，而且都是COMMIT的话，则发送一个全局commit，进入COMMIT状态。
3. 否则广播一个全局abort，进入ABORT状态。

### 参与者
![Actions by Participant](http://upload-images.jianshu.io/upload_images/5753761-aeee1157da0c9368.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 等待来自协作者的投票请求,如果在超时时间内未收到投票请求，则写Abort到log里。
2. 如果参与者对投票回应一个commit,等待协作者的下一个决定，如果在超时时间内未收到协作者的决定，则询问其他参与者（阻塞），猜测协作者的决定，将决定写入log里;否则，回应abort。
3. 收到的决定或者通过其他参与者的状态猜测出的决定：
    3.1 如果为global_commit，则将global_commit写入日志
    3.2 如果为global_abort，则将global_abort写入日志


### 参与者(等待其他参与者的询问)
![Handling Decision Request](http://upload-images.jianshu.io/upload_images/5753761-1cced223cdd1887b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
另起一个工作线程完成以下工作：
1. 阻塞等待接收其他参与者
2. 从本地log中读取最近的状态：
    2.1 如果状态为global_commit，则回应此次参与者的请求，发送一个global_commit。
    2.2 如果状态为INIT或者global_abort，则发送global_abort给参与者
    2.3 其他（Ready），判断不出状态，则skip

![image.png](http://upload-images.jianshu.io/upload_images/5753761-aaf04af3c36ea786.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 故障恢复
节点通过日志的状态判断

  - 参与者（INIT）：安全到当地中止，通知协调员
  - 参与者（READY）：联系其他人
  - 协调器（WAIT）：重新发送VOTE_REQ
  - 协调员（等待/决定）：重新传送VOTE_COMMIT
  
## 2pc存在的问题

问题抛出：如果每个参与者都收到了VOTE_REQ,但是协作者炸了！

结果：2pc是一个阻塞协议，参与者因为没有收到协作者的请求而超时，询问其他参与者，缺发现是ready状态，而无法做出决策，**阻塞**