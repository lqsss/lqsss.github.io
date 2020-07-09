---
title: ygc时间过长实例
tag: 
- gc
categories: jvm
---



session状态机，ygc时间过长，经常需要重启才能维持正常的服务。实习结束后，该问题仍存在，直到某天之前坐在我旁边的技术厉害、经验老道的大哥一眼看出端倪。

问题分析，顺便把部分jvm知识回顾

<!-- more -->

## 问题描述

session状态机中有需要将xml序列化为普通对象的过程吗，使用的XStream，但是每new一个XStream对象，就会新创建一个classloader，类加载器会和类的权限定名作为key，value为真正的Klass对象，会存储在SystemDirectionary里，最终越来越多的存活对象存储在内存里，ygc的时间包括标记活的对象，对象越多，导致需要占用长时间的cpu去标记。

## ygc的触发时机

*   当eden区的容量不在满足新建对象的需求，需要进行一次gc，将eden区和survivor from区清理复制到survivor to区，在survivor区里年龄超过15或者survivor区中存在超过半数年龄一样的对象，大于该年龄的对象会进入到老年代。

## ygc在一个生命周期做了哪些事情

*   查找GC Roots，拷贝所引用的对象到 to 区；
*   递归遍历步骤1中对象，并拷贝其所引用的对象到 to 区，当然可能会存在自然晋升，或者因为 to 区空间不足引起的提前晋升的情况；

## 对于垃圾收集器的选择

### 并发、并行、串行

*   并行：并行多个垃圾线程同时并行执行，用户线程处于等待状态（parller scavenge、parnew、Parallel Old、G1）
*   并发：用户线程和回收线程并发执行，交替工作，以尽可能减少应用程序的停顿时间(CMS、G1)
*   串行：单线程独占,需要暂停用户线程(Serial、Serial Old)

### 老年代和新生代

针对不同年代的收集器

*   新生代：Serial、Parallel Scavenge、G1、ParNew
*   老年代：Serial Old、Parallel Old、G1、CMS

### 清理算法

*   标记整理：用于老年代，Serial Old、Parallel Old、G1(除了CMS)
*   标记清理：CMS
*   复制：用于所有年轻代

### 其它

*   注重吞吐量：**吞吐量=(程序运行时间)/程序运行时间+gc时间**。Parallel Scavenge是针对年轻代的并行回收收集器，与ParNew这样的并行收集器不同的是，注重吞吐量，设置吞吐量和gc时间百分比参数，自适应的调整策略，会自动调整Eden与Survivor区的比例、晋升老年代对象年龄等细节参数
*   注重系统停顿时间：希望应用系统停顿的时间尽可能的少，服务器响应速度高，但是会分一部分cpu给回收线程，程序运行的时间可能要长，吞吐量不太高。CMS就是此类。
*   G1

## CMS、ParNew

存在问题的系统使用的是CMS+ParNew的组合，重点描述一下。

### CMS

#### 垃圾清理

垃圾清理分为了以下几个步骤：

1.  初始标记（STW）
2.  并发标记
3.  重新标记（STW）
4.  并发清理
5.  并发重置
> 并发重置：在垃圾回收完成后，重新初始化 CMS 数据结构和数据，为下一次垃圾回收做好准备。

整个过程中，初始标记、重新标记是需要停下所有的用户线程。

初始标记：

*   标记gcroot可达的老年代
*   标记年轻代可达的老年代对象

并发标记：遍历初始标记的可达对象，递归标记他们引用的对象

重新标记：需要STW，因为在并发标记的时候，会有新的对象指引需要被标记。

> 在CMS回收过程中，应用程序仍然在不停地工作。在应用程序工作过程中，又会不断地产生垃圾。这些新生成的垃圾在当前 CMS 回收过程中是无法清除的。同时，因为应用程序没有中断，所以在 CMS 回收过程中，**还应该确保应用程序有足够的内存可用**。因此，CMS 收集器不会等待堆内存饱和时才进行垃圾回收，而是当前堆内存使用率达到某一阈值时，便开始进行回收，以确保应用程序在 CMS 工作过程中依然有足够的空间支持应用程序运行。

#### 其它

1.  CMS失败时Serial Old替补
2.  full gc也可以设置参数，在full gc前进行一次ygc，提高效率
3.  full gc(major gc)专指清理老年代垃圾

### ParNew

新生代的并行收集器，相当于serial的多线程版，需要STW。

> 并行回收器也是独占式的回收器，在收集过程中，应用程序会全部暂停。但由于并行回收器使用多线程进行垃圾回收，因此，在并发能力比较强的 CPU 上，它产生的停顿时间要短于串行回收器，**而在单 CPU 或者并发能力较弱的系统中，并行回收器的效果不会比串行回收器好，由于多线程的压力，它的实际表现很可能比串行回收器差。**

## ygc时间过程具体分析

### 双亲委派

![](https://blog-1257900554.cos.ap-beijing.myqcloud.com/TIM%E5%9B%BE%E7%89%8720180522165242.png)

这里需要提及两个概念，**初始加载器和定义类加载器**

假如某个类交付A加载器加载(A-&gt;B-&gt;C),A收到了该类的加载请求，转交给父类加载器，最终C加载器加载了该类，那么A为**初始加载器**，C称为**定义类加载器**。

Q：在JVM如何确定一个类型实例？
A: 类的加载器+类的全限定名

###SystemDirectionary

![SystemDictionary](https://blog-1257900554.cos.ap-beijing.myqcloud.com/SystemDictionary.png)

类的加载器+类的全限定名作为key，value为真正的Klass对象，上述A-&gt;B-&gt;C的例子中，会在SystemDirectionary这样的结构中存储三份，它们的value都指向同一个Klass对象。

假设如果我们通过A的加载器+类的全限定名，发现有value存在(说明该类已经被A加载器加载过了)，如此就不会交付给后面的父加载器加载。

### xstream

每次来解析xml对象时，就会new一个自定义的classloader，使得SystemDirectionary越来越膨胀，ygc扫描时间过长。

#### 解决

1.  换一个构造函数，使用appclassloader
2.  静态单例

## 参考

[图解CMS垃圾回收机制，你值得拥有](https://www.jianshu.com/p/2a1b2f17d3e4)
[CMS垃圾回收器详解](https://blog.csdn.net/zqz_zqz/article/details/70568819)
[JVM 垃圾回收器工作原理及使用实例介绍](https://www.ibm.com/developerworks/cn/java/j-lo-JVMGarbageCollection/index.html)
[类加载机制之ClassLoader](https://blog.csdn.net/yangguosb/article/details/77988729?utm_source=blogxgwz8)
[JVM源码分析之自定义类加载器如何拉长YGC](http://lovestblog.cn/blog/2016/03/15/ygc-classloader/)

