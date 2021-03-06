﻿---
title: JVM总结
tag: JVM
categories: JVM
---

JVM知识小结
<!--more-->
## 什么是JVM
JVM是一种软件模拟出来的计算机，它用于执行Java程序，有一套非常严格的技术规范。Java语言编译程序只需要生成在JVM上运行的目标代码（字节码），JVM在执行字节码时，再把字节码解释成具体平台上的机器指令，这是Java跨平台特性的依赖基础。

## JVM结构

![](http://op7scj9he.bkt.clouddn.com/1526909141%281%29.jpg)

1. 程序计数器是一块较小的内存空间，是线程的私有内存。它可以看做是当前线程所执行的	字节码的行号指示器。在虚拟机的概念模型里面（仅是在概念模型，各种虚拟机可能会通过一些更高效的方法实现），**字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。**
2. Java虚拟机栈也是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是Java执行的内存模型：**每个方法在执行的同时都会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机中入栈到出栈的过程。**
3. 本地方法栈和虚拟机栈所发挥的作用都非常相似，他们之间的区别是虚拟机栈为虚拟机执	行Java方法（也是字节码）服务，而本地方法栈则为虚拟机使用到的Native方法服务。
4. Java堆是Java虚拟机所管理的内存中最大的一块。Java堆是被所有的线程共享的一块内存区域，在虚拟机启动时创建，此内存区域的**唯一目的就是存放对象实例**。Java堆是垃圾收集齐管理的主要区域，也称为"GC堆"。从内存回收的角度来看，由于现在的垃圾收集器基本都采用分代收集的方法，所以Java堆还可以细分为：**新生代和老年代：再细致一点的有Eden空间，From Survivor空间、To Survivor空间**。从内存的分配角度来看，线程共享的	Java堆中**可能划分出多个线程私有的分配缓冲区（TLAB）**，不过无论怎么划分，存储的都是对象实例，进一步划分目的只是为了更好的回收内存。
5. 方法区与Java堆一样，是各个线程共享的内存区域，**它用于存储已经被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据**。虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫Non-Heap（非堆），目的是与Java堆区分开来。对于习惯在HotSpot虚拟机上开发、部署程序的开发者来说，很多人更愿意把方法区称为“永久代”。相对而言，垃圾收集器在这个区域出现的比较少，但并非数据进入方法区就	永久存在了。
6. 常量池：jdk1.7之前常量池在方法区里，又分class文件常量池和运行时常量池，class文件常量池包含类信息、常量、静态变量。运行时常量池 中保存着一些class文件中描述的符号引用，同时在类加载的“解析阶段”还会将这些符号引用所翻译出来的直接引用(直接指向实例对象的指针)存储在 运行时常量池中。

>字符串常量池和类引用被移动到了Java堆中,字符串字面量在class文件常量池中，全局字符串池会有它的一个引用。

JDK1.7以后：堆里有字符串常量池，方法区里有一个class文件常量池和运行时常量池。
```java
String a = "123";
String b = "123";
//a==b true
String c = new String("123");
//a==c false
```
a这种等号赋值的方式，首先会在字符串常量池寻找"123"的引用，如果没有则创建这个"123"的对象，首先"123"字面量会在**方法区里的class文件常量池里**，堆里字符串常量池会保留一个对这个字面量的引用，然后栈上a引用变量指向堆上的"123"。b也是一样，不过此时堆里的字符串常量池已经存在"123"这个引用了，则直接指向它即可。

而c是通过new这种指令方式创建对象，那必然是直接在堆中新启一片内存创建新对象。
## 符号引用和直接引用
1. **符号引用就是字符串，文字形式来描述引用关系。**包括类和接口的全限定名、字段的名称和描述符以及方法的名称和描述符;
2. 直接引用就是偏移量，通过偏移量虚拟机可以直接在该类的内存区域中找到方法字节码的起始位置。


## 堆和栈的区别
1. 堆内存用来存放由 new 创建的对象和数组，在堆中分配的内存，由 Java 虚拟机的自动垃圾回收器来管理。
2. 栈中的引用变量指向堆内存中的变量,**存取速度比堆要快**

## Java内存溢出和内存泄漏
1. 内存溢出（out of memory），是指程序在申请内存时，没有足够的内存空间供其使用，出现out of memory；比如申请了一个integer，但给它存了long才能存下的数，那就是内存溢出。
2. 内存泄露（memory leak），是指程序在申请内存后，无法释放已申请的内存空间，一次内存泄露危害可以忽略，但内存泄露堆积后果很严重，无论多少内存,迟早会被占光。memory leak会最终会导致out of memory。

## java内存泄漏的原因：	
>对象可达，但程序不会再用这个对象，而又无法gc。

1. 静态集合类引起内存泄漏：像HashMap、Vector等的使用最容易出现内存泄露，这些静态	变量的生命周期和应用程序一致，他们所引用的所有的对象Object也不能被释放，因为他们也将一直被Vector等引用着。
2. 监听器：在java编程中，我们都需要和监听器打交道，通常一个应用当中会用到很多监听器，我们会调用一个控件的诸如addXXXListener()等方法来增加监听器，但往往在释放对象的时候却没有记住去删除这些监听器，从而增加了内存泄漏的机会。
3. 单例模式：不正确使用单例模式是引起内存泄漏的一个常见问题，单例对象在初始化后将在VM的整个生命周期中存在（以静态变量的方式），如果单例对象持有外部的引用，那么这对象将不能被JVM正常回收，导致内存泄漏。
3. 连接没有close():比如数据库连接（dataSourse.getConnection()），网络连接(socket)和io连接，**除非其显式的调用了其close()方法将其连接关闭，否则是不会自动被GC回收的。**

解决：
1. 将一些连接资源如数据库连接，网络连接等在finally中及时关闭
2. 多用基本类型

## 如何判断一个对象是否已经死亡
1. 引用计数法：给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1，	当引用失效时，计数器值就减1，任何时刻计数器为0的对象就是不可能再被使用的。
2. 可达分析法：这个算法的基本思想就是通过一系列的称为”GC Roots”的对象作为起始点，从这个节点开始向下搜索，搜索所有的路径称为引用链，当一个对象到GC Root没有任何的引用链相连时，则证明这个对象不可达。
>不一定不可达就被马上清理，至少经历两次标记过程，第一次标记，如果没有发现引用链，如果该对象没有重写finalize方法或者调用过，那么会被虚拟机视为"没有必要执行"；如果被视为有必要执行的话，那么会放在F-Queue里，稍后会创建一个低优先级的线程对此队列进行第二次标记，如果该对象在fianlize方法里重新与gc roots连接，则逃脱"死亡",如果未逃脱，则基本上被gc了。

## GC算法
1. 标记-清除算法：该算法分为两个阶段：“标记”和“清除”。首先标记处所有需要回收的对象，然后统一清除被标记的对象。该算法，标记和清除两个阶段的效率不高。此外，回收后会产生大量的不连续的内存碎片，进一步会导致垃圾回收次数的增加。
2. 复制算法：该算法的思想是将可用内存按容量划分为大小相等的两块，每次只使用其中的	一块。当这块的内存用完了，则将还存活的对象复制到另外一块内存上去，然后再把刚使用过的内存空间一次清理掉。从而达到了每次垃圾回收时只是针对其中一块，避免了产生内存碎片等情况。该算法的代价是只是使用了其中一半的内存，代价有点高。
3. 标记-整理算法：标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后清理掉端边界以外的内存。
4. 分代收集：一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最	适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记-清理”或“标记-整理”算法	来进行回收。

## 垃圾收集器
1. Serial(串行GC)收集器：Serial收集器是一个新生代收集器，单线程执行，使用**复制算法**。它在进行垃圾收集时，必须暂停其他所有的工作线程(用户线程)。是Jvm client模式下默认的新生代收集器。对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程收集效率。
2. ParNew(并行GC)收集器：ParNew收集器其实就是serial收集器的多线程版本，除了使用多条线程进行垃圾收集之外，其余行为与Serial收集器一样。
3. Parallel Scavenge(并行回收GC)收集器：Parallel Scavenge收集器也是一个新生代收集器，它也是使用**复制算法**的收集器，又是并行多线程收集器。parallel Scavenge收集器的特点是它的关注点与其他收集器不同，CMS等收集器的关注点是尽可能地缩短垃圾收集时用户线程的停顿时间，而parallel Scavenge收集器的目标则是**达到一个可控制的吞吐量。**吞吐量=程序运行时间/(程序运行时间 +垃圾收集时间)，虚拟机总共运行了	100分钟。其中垃	圾收集花掉1分钟，那吞吐量就是99%。
4. Serial Old(串行GC)收集器：Serial Old是Serial收集器的老年代版本，它同样使用一个单线	程执行收集，使用“标记-整理”算法。主要使用在Client模式下的虚拟机。
5. Parallel Old(并行GC)收集器：Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。
6. CMS(并发GC)收集器：CMS(Concurrent Mark Sweep)收集器是一种**以获取最短回收停顿时间**	为目标的收集器。CMS收集器是基于**“标记-清除”**算法实现的，整个收集过程大致分为4个	步骤：
	- 初始标记(CMS initial mark)
	- 并发标记(CMS concurrenr mark)
	- 重新标记(CMS remark)：修正并发标记期间，用户程序允许而导致标记产生变动的那一部分对象的标记记录
	- 并发清除(CMS concurrent sweep)
7. G1收集器：G1(Garbage First)收集器是JDK1.7提供的一个新收集器，G1收集器基于**“标记-整理”**算法实现，也就是说不会产生内存碎片。还有一个特点之前的收集器进行收集的范围都是整个新生代或老年代，而G1将整个Java堆(包括新生代，老年代)划分为多个大小相等的独立区域Region，虽然还保留有老年代和新生代的概念，但老年代和新生代不再是物理隔离的了，他们都是一部分Region（不需要连续）的集合。

## 内存分配
1. 对象优先分配在新生代的Eden区：大多数情况下对象在新生代Eden区中分配。当Eden区没有足够的空间进行分配时，虚拟机将发起一次MinorGC。
2. 大对象直接进入老年代：所谓大对象是指，需要大量连续内存的Java对象，最典型的大对象就是那种很长的字符串以及数组。
3. 长期存活的对象将进入老年代：虚拟机给每个对象定义一个对象年龄（Age）计数器，	如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将会被移动到Survivor空间中，并且对象的年龄设为1。对象在Survivor区中每“熬过一次Minor	GC”，年龄就增加一岁。它的年龄增加到一定程度（默认为15岁），将会被晋升为老年代中，这个年龄阈值是可以通过参数设置的。
4. 动态对象年龄判断：虚拟机并不是永远要求对象年龄必须达到年龄阀值才能晋升老年代，	如果在Survivor空间中相同年龄对象大小的总和大于Survivor空间的一半，年龄大	或等于该年龄的对象直接就可以进入老年代。

Minor GC:从年轻代空间（包括 Eden 和 Survivor 区域）回收内存被称为 Minor GC。当 JVM 无法为一个新的对象分配空间时会触发 Minor GC，比如当 Eden 区满了。所以分配率越高，越频繁执行 Minor GC。

## 为什么要有Survivor
如果没有Survivor，Eden区每进行一次Minor GC，存活的对象就会被送到老年代。老年代很快被填满，触发Major GC。
>Survivor的存在意义，就是减少被送到老年代的对象，进而减少Full GC的发生，Survivor的预筛选保证，只有经历16次Minor GC还能在新生代中存活的对象，才会被送到老年代。

## 为什么要设置两个Survivor区
设置两个Survivor区最大的好处就是解决了碎片化，永远有一个survivor space是空的，另一个非空的survivor space无碎片。


## 类加载

q：
1. 初始化和实例化不是一个概念。
2. 类加载包括了加载和初始化，加载是第一步，初始化是最后一步。

![](http://op7scj9he.bkt.clouddn.com/F$WYFXQ%7DVN_%7BP8M%25_334BS2.png)

1. 加载：通过类的权限定名获取class二进制文件，转换为方法区运行时的数据结构（class文件常量池），在方法区里生成class对象
2. 链接
 - 验证:确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。
 - 准备:为类变量分配内存并设置类变量初始值
 - 解析:将常量池内的符号引用替换直接引用的过程
3. 初始化:才是真正的程序员编写代码进行初始化，执行<cinit>类构造器方法，这个<cinit>方法由编译器自动收集所有类变量的赋值动作和静态代码块
**静态代码块可以对后面的静态变量赋值，但不可以访问**

>在java里，类只是信息描述的，写明了有哪些内部属性及接口，你可以理解为是定义了一套规则；而Class对象在java里被用来对类的情况进行表述的一个实例，也就是是类的实际表征，可以理解为是对规则的图表化，这样JVM才能直观的看懂，可以看做是一个模版；而类的实例化对象，就是通过模版，开辟出的一块内存进行实际的使用。

## 类加载器
### 双亲委派
类加载器采用双亲委派模式，如果一个类加载器收到类加载请求，它首先不会先自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试去加载。

![](http://op7scj9he.bkt.clouddn.com/TIM%E5%9B%BE%E7%89%8720180522165242.png)

### 类加载器
1. Bootstrap ClassLoader负责加载``$JAVA_HOME中jre/lib/rt.jar``里所有的class，由C++实现，不是ClassLoader子类
2. Extension ClassLoader负责加载java平台中扩展功能的一些jar包，包括$JAVA_HOME中jre/lib/*.jar或-Djava.ext.dirs指定目录下的jar包
3. App ClassLoader负责记载classpath中指定的jar包及目录中class
4. Custom ClassLoader属于应用程序根据自身需要自定义的ClassLoader，如tomcat、jboss都会根据j2ee规范自行实现ClassLoader

q：什么时候需要用到自定义类加载器
1. 当一个类文件需要加载两次
2. 网络传过来java类的字节码，系统的加载器无法完成
3. 类文件需要卸载（空缺）

## 对象创建的过程
1. 虚拟机遇到一条new指令时，首先将先去检查这个指令的参数是否能在常量池中单位到一个类的符号引用代表的类是否被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程。
2. 在类加载检查通过后，接下来虚拟机将为新生对象分配内存。对象所需要的内存大小在类加载完成后便可以确定，加上Java堆中内存是绝对规整的，所有用过的内存都放在一边，空闲的内存放在另一边，之间放着一个指针作为分界点的指示器，那分配内存就仅仅是把那个指针向空闲空间那边挪动一段与对象大小相等的距离，这种方式称为“指针碰撞”。如果Java堆中的内存并不是规整的，已使用的内存和空闲的内存相互交错，那么虚拟机就必须维护一个列表，记录哪块内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式称为“空闲列表”。
3. 内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值，制一部保证了对象的实例字段在Java代码中可以不赋初始值就直接使用。
4. 接下来，虚拟机要对对象进行必要的设置，列如这个对象是哪个类的实例，如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。这些信息存放在对象的对象头之中。


## 指令与调优
### 配置
-Xms:初始堆大小
-Xmx:最大堆大小
-XX:NewSize=n:设置年轻代大小
### 指令
- JPS:显示指定系统内所有的HotSpot虚拟机进程。
- jstat:是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。

### 调优
对JVM内存的系统级的调优主要的目的是减少GC的频率和Full GC的次数，过多的GC和Full GC是会占用很多的系统资源（主要是CPU），影响系统的吞吐量。特别要关注Full GC，因为它会对整个堆进行整理