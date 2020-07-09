---
title: 分布式锁实现 - zookeeper
tag: 
- lock
- zookeeper
categories: distributed
---

1. 分布式锁应用场景
2. Curator实现的demo
3. 原理解析
4. 存在的问题分析
5. 简单实现
<!-- more -->

## 分布式锁应用场景
1. 在单机情况下，我们代码管理一台机器上的多进程或者多线程进入临界区，进程通常用互斥锁，信号量，而多线程使用lock接口或者JVM层面的synchronized。
2. 而集群里多个节点的情况下，需要通过socket通信知道其他节点的状态，保证一个时刻下只有一个节点能进入临界区。

## 基于Curator实现的demo
zookeeper分布锁InterProcessMutex
```java
public static void main(String[] args) throws Exception {
    //创建zookeeper的客户端
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.newClient("10.21.41.181:2181,10.21.42.47:2181,10.21.49.252:2181", retryPolicy);
    client.start();

    //创建分布式锁, 锁空间的根节点路径为/curator/lock
    InterProcessMutex mutex = new InterProcessMutex(client, "/curator/lock");
    mutex.acquire();
    //获得了锁, 进行业务流程
    System.out.println("Enter mutex");
    //完成业务流程, 释放锁
    mutex.release();
    
    //关闭客户端
    client.close();
}
```

## 原理解析
1. 创建一个持久存在的父路径,假设/lock，CreateMode.PERSISTENT
2. 每个节点在/lock下注册/lock/pid-，CreateMode.EPHEMERAL_SEQUENTIAL（短暂有序的节点)，例如/lock/pid-0000000001、pid-0000000002...会在尾部加上10位数
3. 核心：如果当前节点是集群中最小的id，则获取锁。如果不是则等待。
4. 实现：在获取锁的时候，检查自己是否为集群中最小的id，如果不是，则监听自己的前一位机器的情况(exits())，如果是则获取；
5. 实现：如果监听到了删除事件，则重复4

## 存在的问题分析
1. "惊群效应"：每个子节点注册watcher监听/lock下的子节点，zk服务器会对所有服务器发送通知，网络io过大。
2. 网络问题：如果当前最小id的节点获取了锁，但因为网络原因导致session超时，临时路径被删除，这时候下一个节点会监听到而获取到锁，此时会有多个节点进入临界区。
3. 在注册watcher的时候，上一个路径已经删除，那么这个事件不是永远都不触发？


解决：
1. 每个节点只用监听自己的前一个节点的删除事件
2. 需要设置一个恰当的session超时时间
3. ZooKeeper保证查询和监听是一个原子操作，客户端查询数据之后的所有数据变化都能收到监听

## 网上参考的实现
```java
  public boolean checkMinPath() throws KeeperException, InterruptedException {
        List<String> subNodes = zk.getChildren(GROUP_PATH, false);
        Collections.sort(subNodes);
        //判断此节点
        int index = subNodes.indexOf(selfPath.substring(GROUP_PATH.length() + 1));
        switch (index) {
            case -1: {
                System.out.println(PREFIX_OF_THREAD + "本节点已不在了..." + selfPath);
                return false;
            }
            case 0: {
                System.out.println(PREFIX_OF_THREAD + "子节点中，我最小，可以获得锁了！哈哈" + selfPath);
                return true;
            }
            default: {
                this.waitPath = GROUP_PATH + "/" + subNodes.get(index - 1);
                System.out.println(PREFIX_OF_THREAD + "排在我前面的节点是 " + waitPath);
                try {
                    zk.getData(waitPath, true, new Stat());
                    return false;
                } catch (KeeperException e) {
                    if (zk.exists(waitPath, false) == null) {
                        System.out.println(PREFIX_OF_THREAD + "排在我前面的" + waitPath + "已消失 ");
                        return checkMinPath();
                    } else {
                        throw e;
                    }
                }
            }
        }
    }
```
## 参考
[如何能通俗的讲解Zookeeper分布式锁的应用场景？](https://www.zhihu.com/question/65946103)
[curator教程二——分布式锁](https://www.cnblogs.com/520playboy/p/6441651.html)
[基于Zookeeper的分布式锁](http://www.dengshenyu.com/java/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F/2017/10/23/zookeeper-distributed-lock.html)

