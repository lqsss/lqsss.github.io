---
title: 再探ThreadLocal
tag: 
- Concurrency
- ThreadLocal
categories: Concurrency programming
---

主要阐述threadLocal的应用场景
<!-- more -->

## 前景回顾
[之前写的一篇：ThreadLocal和弱引用](https://lqsss.github.io/2018/04/30/ThreadLocal/)
ThreadLocal相当于一个线程的局部变量，底层原理:每一个thread都有一个default访问权限的``ThreadLocalMap<ThreadLocal,Object value>`` threadlocals对象，key都是对ThreadLocal的弱引用，value才是真正存储变量值。

主要讲了部分ThreadLocal源码，但尚未对应用场景做一个具体的分析描述。

## 应用场景构造
>ThreadLocal不是用来解决对象共享访问问题的，而主要提供了**线程保持对象的方法和避免参数传递的方便的对象访问方式**。

>ThreadLocal使用场合主要解决多线程中数据因并发产生不一致的问题。ThreadLocal为每个线程的中并发访问的数据提供一个副本，通过访问副本来运行业务，这样的结果是耗费了内存，但大大减少了线程同步所带来的线程消耗，也减少了线程并发控制的复杂度。

### 避免参数传递，方便访问
假设在一个线程里的多个方法都需要访问某个对象
例如：
```
//[Thread-main]
funA(Object o){
    funB(Object o);
    funC(Object o);
}
```
假如我们在主线程，调用funA方法，而在方法内又通过传递o对象，调用了funB、funC方法。
```java
//[Thread-main]
ThreadLocal<Object> o = new ThreadLocal();
Object value = new Object();
o.set(value);
funA(){
    funB();
    funC();
}
```
改成上述后，funB、funC方法内调用``o.get()``即可。

### 线程封闭，解决线程安全的问题
- SimpleDateFormat对象在多线程的环境下是不安全的。
- 创建一个SimpleDateFormat对象实例耗费很大。
1. SimpleDateFormat为一个static对象。
```java
private static final  SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    
public static  String formatDate(Date date)throws ParseException{
        return sdf.format(date);
}
    
public static Date parse(String strDate) throws ParseException{
        return sdf.parse(strDate);
}
```
**线程不安全**,至于SimpleDateFormat为什么不是线程安全的(留坑)

2. 解决：使用加锁
```java
    private static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
      
    public static String formatDate(Date date)throws ParseException{
        synchronized(sdf){
            return sdf.format(date);
        }  
    }
    
    public static Date parse(String strDate) throws ParseException{
        synchronized(sdf){
            return sdf.parse(strDate);
        }
    } 
```
性能差。

3. 使用ThreadLocal
一个线程内通过get()得到本线程拥有的SimpleDateFormat对象实例，通过它来调用``parse``和``format``操作

## 参考
[聊一聊ThreadLoca](https://blog.csdn.net/u013256816/article/details/51776846)
[深入理解Java：SimpleDateFormat安全的时间格式化](https://www.cnblogs.com/peida/archive/2013/05/31/3070790.html)


