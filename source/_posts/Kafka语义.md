---
title: Kafka语义
tag: 
- mq
- Kafka
categories: distributed
---



kafka语义：从生产者和消费者角度

<!--more-->

## 消息队列的语义

- At most once—最多一次，消息可能会丢，但绝不会重复传递；

- At least once—至少一次，消息绝不会丢，但可能会重复传递；

- Exactly once—每条消息只会被精确地传递一次：既不会多，也不会少；

## 从生产者的角度来看
1. 至少一次
生产者生产消息至broker节点，等待broker节点的响应，如果因为网络通信或者机器故障的原因,没有收到ack，导致producer重新发送。
2. 恰好一次
每个Producer分配一个Id，通过每个消息的序列号进行去重

## 从消费者的角度来看
consumer自己保存offset，**消息处理和offset的更新导致不同的语义**

1. 至少一次
consumer消费消息之后，准备更新offset时，崩溃。之后，当前consumer恢复，或者一个consumer group的其他消费者又再次消费了这条消息。（至少一次）

2. 至多一次
更新保存offset在消费消息之前，在处理消息的时候，宕机了，这样会导致消息丢失的情况。

>默认Kafka提供at-least-once语义的消息分发，允许用户通过在处理消息之前保存位置信息的方式来提供at-most-once语义。如果我们可以实现消费是幂等的，这个时候就可以认为整个系统是Exactly once的了。

