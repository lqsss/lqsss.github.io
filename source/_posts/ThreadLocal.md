---
title: ThreadLocal和弱引用
tag: 
- Concurrency
- ThreadLocal
- Source code
- WeakReference
categories: Concurrency programming
---

1. 针对ThreadLocal的部分源码，对其原理做一下笔记记录。
2. Java中各种引用的介绍
<!--more-->

## 背景
今天在看知乎关于弱引用知识时，看到ThreadLocal的内存泄漏，对ThreadLocal只是会用get() set(),其他的还未细细研究过。

## ThreadLocal
### 原理图

![原理图](http://op7scj9he.bkt.clouddn.com/threadlocal.jpg)

>ThreadLocal的实现是这样的：每个Thread 维护一个 ThreadLocalMap 映射表，这个映射表的 key 是 ThreadLocal 实例本身，value 是真正需要存储的 Object。

也就是说 ThreadLocal 本身并不存储值，它只是作为一个 key 来让线程从 ThreadLocalMap 获取 value。值得注意的是图中的虚线，表示 ThreadLocalMap 是使用 ThreadLocal 的弱引用作为 Key 的，弱引用的对象在 GC 时会被回收。


### 部分源码剖析
1. Thread类里的ThreadLocalMap变量
```java
//default访问权限
ThreadLocal.ThreadLocalMap threadLocals = null;
```

2. ThreadLocal里的get( )
```java
    public T get() {
    //获取当前线程变量
        Thread t = Thread.currentThread();
        //ThreadLocalMap<ThreadLoacl,Object>
        //得到当前线程的ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        //如果已经创建过
        if (map != null) {
            //通过TheadLocal 获得ThreadLocalMap的Entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                //拿到value
                T result = (T)e.value;
                return result;
            }
        }
        //如果ThreadLocalMap还没创建
        return setInitialValue();
    }
    
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

3. set()
```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

总结：Thread维护了一个``ThreadLocalMap<ThreadLocal,Object value>``
get、set()通过当前线程实例，获取ThreadLocalMap，如果没有则创建ThreadLocalMap实例

### 关于ThreadLocal的内存泄漏

#### 内存泄漏是什么？
>在Java中，我们不用（也没办法）自己释放内存，无用的对象由GC自动清理，这也极大的简化了我们的编程工作。但，实际有时候一些不再会被使用的对象，在GC看来不能被释放，就会造成内存泄露

#### ThreadLocal为什么会造成内存泄漏？

1. 源码
```java
static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        //other...
}
```

``static class Entry extends WeakReference<ThreadLocal<?>>``我们可以看到Entry继承了一个弱引用

2. 弱引用
>Java中的弱引用具体指的是java.lang.ref.WeakReference<T>类
>弱引用对象的存在**不会阻止它所指向的对象被垃圾回收器回收**。弱引用最常见的用途是实现规范映射(canonicalizing mappings，比如哈希表）。
假设垃圾收集器在某个时间点决定一个对象是弱可达的(weakly reachable)（**也就是说当前指向它的全都是弱引用**），这时垃圾收集器会**清除所有指向该对象的弱引用**，然后把这个弱可达对象标记为可终结(finalizable)的，这样它随后就会**被回收**。与此同时或稍后，垃圾收集器会把那些刚清除的弱引用放入创建弱引用对象时所指定的引用队列(Reference Queue)中。


3. ThreadLocal弱引用导致内存泄漏
>ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用来引用它，那么系统GC的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：**Thread Ref -> Thread -> ThreaLocalMap -> Entry ->value**永远无法回收，造成内存泄漏。

简单来说，就是因为ThreadLocalMap的key是弱引用，当TheadLocal外部没有强引用时，就被回收，此时会出现ThreadLocalMap<null,value>的情况，而线程没有结束的情况下，导致这个null对应的value一直无法回收，导致泄漏。

### ThreadLocal为什么还要弱引用呢？
1. ThreadLocalMap对于内存泄漏的防护措施(**ThreadLocalMap的set、get、remove**)
>在调用 ThreadLocal 的 get()，set() 和 remove() 的时候都会清除当前线程 ThreadLocalMap 中所有 key 为 null的value。这样可以降低内存泄漏发生的概率。所以我们在使用ThreadLocal 的时候，每次用完 ThreadLocal 都调用remove()方法，清除数据，防止内存泄漏。

2. 强引用vs.弱引用
- key 使用强引用：引用的ThreadLocal的对象被回收了，但是ThreadLocalMap还持有ThreadLocal的强引用，如果没有手动删除，ThreadLocal不会被回收，导致Entry内存泄漏。
- key 使用弱引用：引用的ThreadLocal的对象被回收了，由于ThreadLocalMap持有ThreadLocal的弱引用，即使没有手动删除，ThreadLocal也会被回收。value在下一次ThreadLocalMap调用set,get，remove的时候会被清除。

ThreadLocalMap的生命周期跟Thread一样长，如果都没有手动删除对应key，都会导致内存泄漏,但是弱引用的话进行set、get、remove方法时，会清除key为null的value，比较方便一点。

因此，ThreadLocal内存泄漏的根源是：由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应key就会导致内存泄漏，而**不是因为弱引用。**

### 安全操作
每次使用完ThreadLocal，都调用它的remove()方法，清除数据。
>ThreadLocal建议： 
1. ThreadLocal类变量因为本身定位为要被多个线程来访问，它通常被定义为static变量。 
2. 能够通过值传递的参数，不要通过ThreadLocal存储，以免造成ThreadLocal的滥用。 
3. 在线程池的情况下，在ThreadLocal业务周期处理完成时，最好显示的调用remove()方法，清空“线程局部变量”中的值。 
4. 在正常情况下使用ThreadLocal不会造成OOM, 弱引用的知识ThreadLocal,保存值依然是强引用，如果ThreadLocal依然被其他对象应用，线程局部变量将无法回收。

### 应用场景
//Todo


##参考
[十分钟理解Java中的弱引用](http://www.importnew.com/21206.html)
[Java中的强引用，软引用，弱引用，虚引用有什么用？](https://www.zhihu.com/question/37401125)