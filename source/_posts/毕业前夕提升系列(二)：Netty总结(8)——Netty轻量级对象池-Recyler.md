---
title: 毕业前夕提升系列(二)：Netty总结(8)——Netty轻量级对象池-Recyler
tag: 
- summary
categories: Netty
---

## Recyler的使用

## 为什么要使用Recyler

在对象池里直接复用之前已经创建好的对象

1. 减少内存分配的频率
2. 减少对象的创建与销毁，减少young gc的次数

## Recyler细节

### Recyler结构

- ``threadLocal``

```java
private final FastThreadLocal<Stack<T>> threadLocal = new FastThreadLocal<Stack<T>>() {
        @Override
        protected Stack<T> initialValue() {
            return new Stack<T>(Recycler.this, Thread.currentThread(), maxCapacityPerThread, maxSharedCapacityFactor,
                    ratioMask, maxDelayedQueuesPerThread);
        }

        @Override
        protected void onRemoval(Stack<T> value) {
            // Let us remove the WeakOrderQueue from the WeakHashMap directly if its safe to remove some overhead
            if (value.threadRef.get() == Thread.currentThread()) {
               if (DELAYED_RECYCLED.isSet()) {
                   DELAYED_RECYCLED.get().remove(value);
               }
            }
        }
    };
```

每一个线程都有一个FastThreadLocal的线程变量，存储的数据是Stack<T>

- ``Stack``

```java
Stack(Recycler<T> parent, Thread thread, int maxCapacity, int maxSharedCapacityFactor,
              int ratioMask, int maxDelayedQueues) {
            this.parent = parent;
            threadRef = new WeakReference<Thread>(thread);
    //element的最大容量
            this.maxCapacity = maxCapacity;
    //该字段表示该线程创建的对象能在其他线程最大缓存对象的数量
            availableSharedCapacity = new AtomicInteger(max(maxCapacity / maxSharedCapacityFactor, LINK_CAPACITY));
    //存储handle
            elements = new DefaultHandle[min(INITIAL_CAPACITY, maxCapacity)];
    //并不是每次都能把对象回收，回收的比率
            this.ratioMask = ratioMask;
    //通过get获取对象，这个对象是由当前thread管理，但是最终在其他线程进行释放，会将该对象放在其他线程里数据结构里(WeakOrderQueue),该字段表示该线程最大缓存对象的数量
            this.maxDelayedQueues = maxDelayedQueues;
        }
```

存储DefaultHandle对象，即可复用对象。内部elment数组，是一个个handle，handle会被外部变量引用

各参数默认值：

maxCapacity：4 * 1024;

maxSharedCapacityFactor：2

ratioMask：7

maxDelayedQueues：默认是2倍cpu核数

availableSharedCapacity：16



- WeakOrderQueue：存储其它线程回收到本线程stack的对象，当某个线程从stack中获取不到对象时，就会从WeakOrderQueue中获取对象

  //todo



## 从Recyler获取对象

### 步骤

1. 获取当前线程的stack
2. 从stack里面弹出对象
3. 创建对象并绑定Stack



### 源码

``Recyler#get()``:从获取handle对象

```java
public final T get() {
    //maxCapacityPerThread为0，指对象池不再容纳对象
    if (maxCapacityPerThread == 0) {
        //直接创建新的对象
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    //1. 获取当前线程的stack
    Stack<T> stack = threadLocal.get();
    //2. 从stack里pop出对象
    DefaultHandle<T> handle = stack.pop();
    if (handle == null) {
        //2.1 如果stack里为空，创建handle
        handle = stack.newHandle();
        //3. 创建新对象并绑定Stack
        handle.value = newObject(handle);
    }
    return (T) handle.value;
}
```

``NOOP_HANDLE``不作任何回收的Handle对象

```java
@SuppressWarnings("rawtypes")
private static final Handle NOOP_HANDLE = new Handle() {
    @Override
    public void recycle(Object object) {
        // NOOP
    }
};
```



2.  从stack弹出对象``DefaultHandle<T> handle = stack.pop();``

```java
DefaultHandle<T> pop() {
    int size = this.size;
    //当前线程的stack已经没有对象
    if (size == 0) {
        //2.1 可能有其他线程在共用该线程的对象
        if (!scavenge()) {
            return null;
        }
        size = this.size;
    }
    size --;
    DefaultHandle ret = elements[size];
    elements[size] = null;
    if (ret.lastRecycledId != ret.recycleId) {
        throw new IllegalStateException("recycled multiple times");
    }
    //初始化
    ret.recycleId = 0;
    ret.lastRecycledId = 0;
    this.size = size;
    return ret;
}
```

2.1 当前线程的stack没有可用的对象复用，其他线程可能有回收过本线程创建的对象

``scavenge()``

```java
boolean scavenge() {
    // continue an existing scavenge, if any
    //2.1.1 扫描一些可复用对象的逻辑
    if (scavengeSome()) {
        return true;
    }

    // reset our scavenge cursor
    //如果WeakOrderQueue也没有，cursor移到head
    prev = null;
    cursor = head;
    return false;
}
```

``scavengeSome()``

```java
boolean scavengeSome() {
    WeakOrderQueue prev;
    WeakOrderQueue cursor = this.cursor;
    if (cursor == null) {
        prev = null;
        cursor = head;
        if (cursor == null) {
            return false;
        }
    } else {
        prev = this.prev;
    }

    boolean success = false;
    do {
        //2.1.1.1 将该weakQueue的link对象转到该线程stack上
        if (cursor.transfer(this)) {
            success = true;
            break;
        }
        WeakOrderQueue next = cursor.next;
        if (cursor.owner.get() == null) {
            // If the thread associated with the queue is gone, unlink it, after
            // performing a volatile read to confirm there is no data left to collect.
            // We never unlink the first queue, as we don't want to synchronize on updating the head.
            if (cursor.hasFinalData()) {
                for (;;) {
                    if (cursor.transfer(this)) {
                        success = true;
                    } else {
                        break;
                    }
                }
            }

            if (prev != null) {
                prev.setNext(next);
            }
        } else {
            prev = cursor;
        }

        cursor = next;

    } while (cursor != null && !success);

    this.prev = prev;
    this.cursor = cursor;
    return success;
}
```



2.1.1.1 将该weakQueue的link对象转到该线程stack上

``transfer(Stack<?> dst)``

```java
     boolean transfer(Stack<?> dst) {
            Link head = this.head.link;
            if (head == null) {
                return false;
            }

         //如果当前WeakOrderQueue已经读到最后的link节点了（LINK_CAPACITY为handle数组的极限大小）
            if (head.readIndex == LINK_CAPACITY) {
                if (head.next == null) {
                    return false;
                }
                //将下一个weakOrderQueue作为当前的head
                this.head.link = head = head.next;
            }

         	//获取head的readIndex
            final int srcStart = head.readIndex;
            int srcEnd = head.get();
         
           //计算出需要转移的数据大小
            final int srcSize = srcEnd - srcStart;
            if (srcSize == 0) {
                return false;
            }

         	//获取待转移目标stack目前的可用偏移量
            final int dstSize = dst.size;
         	//预期需要的容量
            final int expectedCapacity = dstSize + srcSize;
			
         	//超出stack数组的最大容量
            if (expectedCapacity > dst.elements.length) {
                //最终实际大小：进行左移1位进行扩容
                final int actualCapacity = dst.increaseCapacity(expectedCapacity);
                //copy终点
                srcEnd = min(srcStart + actualCapacity - dstSize, srcEnd);
            }

            if (srcStart != srcEnd) {
                final DefaultHandle[] srcElems = head.elements;
                final DefaultHandle[] dstElems = dst.elements;
                int newDstSize = dstSize;
                for (int i = srcStart; i < srcEnd; i++) {
                    DefaultHandle element = srcElems[i];
                    if (element.recycleId == 0) {
                        element.recycleId = element.lastRecycledId;
                    } else if (element.recycleId != element.lastRecycledId) {
                        throw new IllegalStateException("recycled already");
                    }
                    srcElems[i] = null;

                    if (dst.dropHandle(element)) {
                        // Drop the object.
                        continue;
                    }
                    element.stack = dst;
                    dstElems[newDstSize ++] = element;
                }

                if (srcEnd == LINK_CAPACITY && head.next != null) {
                    // Add capacity back as the Link is GCed.
                    this.head.reclaimSpace(LINK_CAPACITY);
                    this.head.link = head.next;
                }

                head.readIndex = srcEnd;
                if (dst.size == newDstSize) {
                    return false;
                }
                dst.size = newDstSize;
                return true;
            } else {
                // The destination stack is full already.
                return false;
            }
        }
    }
```



3. 如果stack为空，创建一个默认handle：``newHandle()``

```java
DefaultHandle<T> newHandle() {
    return new DefaultHandle<T>(this);
}
```



4. 创建新对象并绑定Stack ``handle.value = newObject(handle);``

``Recyler#newObject(Handle<T> handle)``，实现该接口，将handle与对象绑定，随后被对象的handle进行回收

```java
protected abstract T newObject(Handle<T> handle);
```



## 回收对象

### 同线程回收对象

```java
@Override
public void recycle(Object object) {
    if (object != value) {
        throw new IllegalArgumentException("object does not belong to handle");
    }

    Stack<?> stack = this.stack;
    if (lastRecycledId != recycleId || stack == null) {
        throw new IllegalStateException("recycled already");
    }

    stack.push(this);
}
```



回收对象入口：``Recycler$Stack#push()``

```java
void push(DefaultHandle<?> item) {
    Thread currentThread = Thread.currentThread();
    //判断即将回收的对象是否是当前进行回收的线程创建的
    if (threadRef.get() == currentThread) {
        // The current Thread is the thread that belongs to the Stack, we can try to push the object now.
        //1. 同线程回收入口
        pushNow(item);
    } else {
        // The current Thread is not the one that belongs to the Stack
        // (or the Thread that belonged to the Stack was collected already), we need to signal that the push
        // happens later.
        //2. 回收非本线程的对象
        pushLater(item, currentThread);
    }
}
```



1. 同线程回收入口``pushNow()``

```java
private void pushNow(DefaultHandle<?> item) {
    //第一次的话肯定为0
    if ((item.recycleId | item.lastRecycledId) != 0) {
        throw new IllegalStateException("recycled already");
    }
    //OWN_THREAD_ID 默认值
    item.recycleId = item.lastRecycledId = OWN_THREAD_ID;

    int size = this.size;
    //1.1 大于最大容量或者达到需要drop的比例，并不是回收调所有对象
    if (size >= maxCapacity || dropHandle(item)) {
        // Hit the maximum capacity or should drop - drop the possibly youngest object.
        return;
    }
    if (size == elements.length) {
        elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
    }

    elements[size] = item;
    this.size = size + 1;
}
```

1.1 判断是否drop掉，不进行回收：``Stack#dropHandle()``

```java
boolean dropHandle(DefaultHandle<?> handle) {
    if (!handle.hasBeenRecycled) {
        if ((++handleRecycleCount & ratioMask) != 0) {
            // Drop the object.
            return true;
        }
        handle.hasBeenRecycled = true;
    }
    return false;
}
```

``ratioMask``为8，每一次预回收时，handleRecycleCount++，每次进行&运算不等0就丢弃，不进行回收

### 异线程回收对象

1. 获取WeakOrderQueue
2. 创建WeakOrderQueue
3. 将对象追加到WeakOrderQueue

``pushLater()``异线程回收入口

```java
private void pushLater(DefaultHandle<?> item, Thread thread) {
    // we don't want to have a ref to the queue as the value in our weak map
    // so we null it out; to ensure there are no races with restoring it later
    // we impose a memory ordering here (no-op on x86)
    
    //1.1 获得本线程的Map<Stack<?>, WeakOrderQueue>
    Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
    //1.2 通过回收对象的stack来获取本线程的WeakOrderQueue
    WeakOrderQueue queue = delayedRecycled.get(this);
    //2.在找不到queue的情况下，创建weakOrderQueue
    if (queue == null) {
        //2.1 如果可以延迟回收的数量已经达到上限，则塞进去一个无用的WeakOrderQueue

        if (delayedRecycled.size() >= maxDelayedQueues) {
            // Add a dummy queue so we know we should drop the object
            delayedRecycled.put(this, WeakOrderQueue.DUMMY);
            return;
        }
        // Check if we already reached the maximum number of delayed queues and if we can allocate at all.
        
        //2.2 判断该stack是否满足条件，满足则创建新的queue对象
        if ((queue = WeakOrderQueue.allocate(this, thread)) == null) {
            // drop object
            return;
        }
        //2.3 将新创建的queue插入Map<Stack<?>, WeakOrderQueue>线程变量里
        //this
        delayedRecycled.put(this, queue);
        //发现是无用的WeakOrderQueue，则当前stack不进行回收，直接drop
    } else if (queue == WeakOrderQueue.DUMMY) {
        // drop object
        return;
    }

    //2.4 将该回收对象加入到weakQueue中
    queue.add(item);
}
```



``DELAYED_RECYCLED``

```java
private static final FastThreadLocal<Map<Stack<?>, WeakOrderQueue>> DELAYED_RECYCLED =
    new FastThreadLocal<Map<Stack<?>, WeakOrderQueue>>() {
    @Override
    protected Map<Stack<?>, WeakOrderQueue> initialValue() {
        return new WeakHashMap<Stack<?>, WeakOrderQueue>();
    }
};
```

解释一下DELAYED_RECYCLED这个数据结构，它是一个本地线程变量，存储的key为Stack，value为WeakOrderQueue，当其他线程创建的对象被该线程回收，那么就存储创建对象的线程对应的Stack和WeakOrderQueue。

- 例如线程1创建的对象需要线程2回收，则DELAYED_RECYCLED会存储线程2的Stack和线程2对应的WeakOrderQueue，然后加入到该queue中



#### 1. 获取WeakOrderQueue 

假如线程1的对象由线程2来回收，这里DELAYED_RECYCLED.get()获取线程2的Map<Stack<?>, WeakOrderQueue>，然后获取相对应的WeakOrderQueue



#### 2. 创建WeakOrderQueue 

2.2 判断该stack是否满足条件，满足则创建新的queue对象

``WeakOrderQueue.allocate(this, thread))``

```java
static WeakOrderQueue allocate(Stack<?> stack, Thread thread) {
    // We allocated a Link so reserve the space
    return Head.reserveSpace(stack.availableSharedCapacity, LINK_CAPACITY)
        ? newQueue(stack, thread) : null;
}
```



``Recycler#reserveSpace(Stack<?> stack, Thread thread)``

参数：

- AtomicInteger availableSharedCapacity：stack目前能够允许外部线程能够缓存对象的空间大小
- int space：所需空间大小，这里传的是``INK_CAPACITY`` 默认16，需要16k大小

```java
INK_CAPACITY = safeFindNextPositivePowerOfTwo(
        max(SystemPropertyUtil.getInt("io.netty.recycler.linkCapacity", 16), 16));
```

```java
//2.2.1 判断该stack能否还能缓存space大小的对象
static boolean reserveSpace(AtomicInteger availableSharedCapacity, int space) {
    assert space >= 0;
   	//cas操作更新剩余空间
    for (;;) {
        int available = availableSharedCapacity.get();
        //不够满足的话则返回false
        if (available < space) {
            return false;
        }
        //最终更新量为available - space
        if (availableSharedCapacity.compareAndSet(available, available - space)) {
            return true;
        }
    }
}
```



创建新的WeakQueue对象``newQueue(Stack<?> stack, Thread thread)``

```java
static WeakOrderQueue newQueue(Stack<?> stack, Thread thread) {
    //2.2.2 创建WeakQueue对象
    final WeakOrderQueue queue = new WeakOrderQueue(stack, thread);
    // Done outside of the constructor to ensure WeakOrderQueue.this does not escape the constructor and so
    // may be accessed while its still constructed.
    stack.setHead(queue);

    return queue;
}
```

2.2.2 创建WeakQueue对象

```java
private WeakOrderQueue(Stack<?> stack, Thread thread) {
    //2.2.2.1 创建一个Link节点
    tail = new Link();

    // Its important that we not store the Stack itself in the WeakOrderQueue as the Stack also is used in
    // the WeakHashMap as key. So just store the enclosed AtomicInteger which should allow to have the
    // Stack itself GCed.
    //2.2.2.2 创建头结节点
    head = new Head(stack.availableSharedCapacity);
    //2.2.2.3 将link节点跟在head节点后
    head.link = tail;
    //owner拥有者字段是对回收对象创建者线程的弱引用
    owner = new WeakReference<Thread>(thread);
}
```



![image-20200516234034054](C:\Users\51344\AppData\Roaming\Typora\typora-user-images\image-20200516234034054.png)

> - 每一个thread对应stack，维护一个数组存储着可以复用的对象()handle)，该stack是用来存储该线程回收的对象。
> - 如果该对象是由本线程1创建，但是由其他线程2回收，可以通过类常量DELAYED_RECYCLED来获取，DELAYED_RECYCLED结构是(Map<Stack<?>,WeakQueue>) ，找到或创建合适的weakQueue，将该对象回收存储在一个link节点与该WeakQueue关联，然后放入线程2的DELAYED_RECYCLED里
>
> 

## 参考

[Netty之Recycler实现对象池](https://my.oschina.net/hutaishi/blog/1929605/print)

