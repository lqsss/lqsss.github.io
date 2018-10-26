---
title: Lamport-全序组播（totally-ordered-mulitcast）
tag: 
- logic clock
categories: distributed
---

之前跟老师没有汇报清楚，于是写了这篇文章。
1. 意义：保证分布式系统的各个子系统都按相同的顺序执行一组操作
2. 消息传递时，逻辑时钟的变化
<!-- more -->
## 背景
如何保证分布式系统中的各个子系统都按相同的顺序执行一组操作？一个等价的问题就是如何在分布式系统下实现全序组播(totally-ordered mulitcast)。这个可以利用Lamport提出的逻辑时钟实现。
但是规则的逻辑时钟是一个偏序，而非全序。

## 解决
使用Lamport的算法，但使用进程ID打破关系
L(e) = M * Li(e) + i
M = maximum number of processes
i = process ID

## 假设
>Assume all messages sent by one sender are received in the order they were sent and that no messages are lost.

Lamport全序组播是在如下的假设下进行的：
1. 发送者发送的顺序和接受者接收的顺序一致
2. 消息不会丢失

## 实现
关键：每一个进程都维护一个本地请求队列(Request Queue)，此队列里的消息按时间戳排序。

1. 进程Pi给消息m加上时间戳Ti(m)，并广播给所有进程(包括Pi);
2. 进程Pj接收到消息m后，首先更新本地逻辑时钟，并把消息m加入请求队列，然后向所有进程（包括Pj）广播接收到消息m的确认信息ACK。
3. 当进程的请求队列中的某个消息位于队列顶部，且已经收到所有进程关于此消息的确认信息ACK时，可以把此消息从队列顶部取走，并传给应用程序。

## 实例
![
更新操作顺序不一致，导致数据副本状态不一致](https://upload-images.jianshu.io/upload_images/5753761-ed0370f8d7446632.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**全局的逻辑时间**
假设图左边为A，右边为B

### B收到A之后，B才发送
![图1](https://upload-images.jianshu.io/upload_images/5753761-5f38169f1e5ff676.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

L(e) = M * Li(e) + i
计算例如：A->B(request)，此时规则的逻辑时间为1，M（最大进程数）为2，i为pid 1，所以此时时间戳应该为1*2+1 = 3
- queue：[{1,request,timestamps:3},{2,request,timestamps :10}]

### AB同时发（1）

![图2](https://upload-images.jianshu.io/upload_images/5753761-1c1088f905de6cb5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里ACK讨论的是在双方都未接收对方ACK前发送的。
队列里有两个请求，按时间戳从小到大排序
- queue：[{1,request,timestamps:3},{2,request,timestamps :4}]

### AB同时发（2）
![图3](https://upload-images.jianshu.io/upload_images/5753761-703211dfb682b5c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- queue：[{1,request,timestamps:3},{2,request,timestamps :4}]

### AB同时发（3）
![图4](https://upload-images.jianshu.io/upload_images/5753761-8fa6a14bd21fb52f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- queue：[{1,request,timestamps:3},{2,request,timestamps :4}]

### 情况论述
**没讨论的情况：**B收到A的ACK穿插发生在B收到A和马上发回A一个ACK的间隙中。

**不可能出现的情况：**根据设定``发送者发送的顺序和接受者接收的顺序一致``，不可能出现**B都已经收到了A发送的ACK，还没有收到A的请求消息的情况。**

### ACK分析
后面ACK接收情况的不同影响的是，执行消息的时机（只有当接收到所有其他进程对此消息的ACK，并且处在队列头部），执行顺序依旧是一样
（例如AB同时发的3种情况里的队列{1,request,timestamps:3}{2,request,timestamps :4}）


## 关键
1. 收到的ACK的时间戳是高于接收到的消息
例如AB同时发的(图2)。B收到A的请求时间戳（3）小于收到A对于B请求回复的ACK（7）。
Lamport全序组播是在``发送者发送的顺序和接受者接收的顺序一致``的假设下进行的，假设B的请求m，当B收到A对于消息m的ACK时，说明了以下两点：
- 进程B知道了A进程成功收到了m消息
- A发送ACK消息之前发送的请求消息N1、N2......，B进程也收到了，确保消息m不会提前执行，需要按照执行消息规则（时间戳顺序）来执行。

2. 所有的进程的本地都有一份一致的请求队列副本




