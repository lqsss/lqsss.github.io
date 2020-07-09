---
title: 毕业前夕提升系列(二)：Netty总结(5)——Netty中内存分配
tag: 
- summary
categories: Netty
---


本篇是讲Netty中byteBuf与底层I/O的交互

<!--more-->

## 问题

Q1： 内存类别有哪些？

堆内堆外？

Q2：如何减少多线程内存分配之间的竞争

Q3：不同大小的内存是如何进行分配的

## ByteBuf

### 结构

![](https://blog-1257900554.cos.ap-beijing.myqcloud.com/byteBuf%E7%BB%93%E6%9E%84.png)

3个重要的指针：

*   readerIndex
*   writerIndex
*   capacity

readerIndex ~ writerIndex （readable bytes）：可读的数据区域

writerIndex ~ capacity （writable bytes）：可以写的空闲空间

* * *

当要写的数据大于现有的writable bytes， 会进行扩容

当要写的数据导致整个数据大小可能会超过maxCapacity，则拒绝

`public abstract int maxCapacity();`

* * *

### 一些API

*   read
*   write
*   set

    - 不会移动任何指针

*   mark、reset

    - 标记读写index
    
    *   复原之前标记的index处

### 分类

![byteBuf分类](https://blog-1257900554.cos.ap-beijing.myqcloud.com/byteBuf%E5%88%86%E7%B1%BB.png)

#### Pooled和Unpooled

*   内存分配时，将预先分配好的内存分配

*   向系统申请分配内存

### unsafe和非unsafe

*   unsafe通过操作底层unsafe的offset+index的方式去操作数据
*   不会依赖jdk底层的unsafe

### Heap和Direct

*   堆上
*   jdk的api进行分配，不受jvm控制，也就不参与gc(堆外)，byteBuffer

## ByteBufAllocator

### AbstractByteBufAllocator
```java
public abstract class AbstractByteBufAllocator implements ByteBufAllocator {
	//...
}
```




`AbstractByteBufAllocator`抽象类，是ByteBufAllocator骨架类进行实现和扩展。提供byteBuf分配，内部实现交由底层实现(`PooledByteBufAllocator`和`ByteBufAllocator`继承`AbstractByteBufAllocator`)。

### UnpooledByteBufAllocator

**我们这里先不讲unsafe和非unsafe纳入讨论。**

#### heap
```java
@Override
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
    return PlatformDependent.hasUnsafe() ?
            new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
  //InstrumentedUnpooledUnsafeHeapByteBuf extends UnpooledUnsafeHeapByteBuf
            new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
}
```



追溯到`UnpooledHeapByteBuf`构造函数。

```java
public UnpooledHeapByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);

    checkNotNull(alloc, "alloc");

    if (initialCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
                "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
    }

    this.alloc = alloc;
    setArray(allocateArray(initialCapacity));
    setIndex(0, 0);
}

 protected byte[] allocateArray(int initialCapacity) {
        return new byte[initialCapacity];
}
```



`setArray(allocateArray(initialCapacity));`堆上创建字节数组，并保存。

#### direct
```java
public UnpooledDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);
    if (alloc == null) {
        throw new NullPointerException("alloc");
    }
    if (initialCapacity < 0) {
        throw new IllegalArgumentException("initialCapacity: " + initialCapacity);
    }
    if (maxCapacity < 0) {
        throw new IllegalArgumentException("maxCapacity: " + maxCapacity);
    }
    if (initialCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
                "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
    }

    this.alloc = alloc;
    setByteBuffer(allocateDirect(initialCapacity));
}
```

```java
protected ByteBuffer allocateDirect(int initialCapacity) {
    return ByteBuffer.allocateDirect(initialCapacity);  //1.
}
```



1.直接调用jdk底层的api分配内存

* * *

### unsafe和非unsafe

非unsafe堆上：直接通过jdk底层api获取内存 (`ByteBuffer.allocateDirect`)  

unsafe：通过直接通过jdk底层api获取内存，但是会获取该内存的地址，把内存地址保存一下。

调用_getByte(int index)-&gt;获取内存地址+index得到一个地址-&gt;unsafe.getByte(地址)

```java
BYTE_ARRAY_BASE_OFFSET = UNSAFE.arrayBaseOffset(byte[].class);
//获取byte[]数组base地址
```


### PooledByteBufAllocator

//未完待续。。。


