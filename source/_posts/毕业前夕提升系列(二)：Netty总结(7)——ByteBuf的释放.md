---
title: 毕业前夕提升系列(二)：Netty总结(7)——ByteBuf的释放
tag: 
- summary
categories: Netty
---



<!--more-->

## ByteBuf的释放                                                                                                                                                                                                                                                                                                                                                                                                                 

``byteBuf.release();`` ->``release0(int decrement)``，最终会走到``PoolByteBuf#deallocate（）``

```java
private boolean release0(int decrement) {
    
    int oldRef = refCntUpdater.getAndAdd(this, -decrement);
    
    //过去的引用值和减去的值相等，引用为0，就可以被释放了
    if (oldRef == decrement) {
        //具体释放逻辑
        deallocate();
        return true;
    }
    if (oldRef < decrement || oldRef - decrement > oldRef) {
        // Ensure we don't over-release, and avoid underflow.
        refCntUpdater.getAndAdd(this, decrement);
        throw new IllegalReferenceCountException(oldRef, -decrement);
    }
    return false;
}
```



```java
@Override
protected final void deallocate() {
    if (handle >= 0) {
        final long handle = this.handle;
        //该pooledByteBuf引用地址不再指向任何一块内存
        this.handle = -1;
        
        memory = null;
        tmpNioBuf = null;
        //1.2.
        chunk.arena.free(chunk, handle, maxLength, cache);
        chunk = null;
        //3. 
        recycle();
    }
}
```

释放流程：

``chunk.arena.free(chunk, handle, maxLength, cache);``对应下列1、2过程

`` recycle();``对应下列3过程

1. 释放的话，会把连续的内存区段加到缓存
2. 如果加缓存失败，会标记连续的内存区段为未使用（page按层标记，subPage会标记位视图为0）
3. ByteBuf会加到对象池里（ByteBuf被释放后，本身这个对象也不会被销毁，加入到对象池，等到下次复用）



## 加缓存或者标记未使用

### 释放内存入口

1. ``PoolAreana#free()``

通过``PoolAreana``创建多个``PoolChunk``，``PoolChunk``里有个属性是``PoolAreana``

```java
void free(PoolChunk<T> chunk, long handle, int normCapacity, PoolThreadCache cache) {
    if (chunk.unpooled) {
        int size = chunk.chunkSize();
        destroyChunk(chunk);
        activeBytesHuge.add(-size);
        deallocationsHuge.increment();
    } else {
        //1.1 池化的内存才会有加入缓存复用的机制
        SizeClass sizeClass = sizeClass(normCapacity);
        //1.2 添加到cache
        if (cache != null && cache.add(this, chunk, handle, normCapacity, sizeClass)) {
            // cached so not free it.
            //添加到cache则不释放这块内存
            return;
        }

        //1.3 未成功添加缓存则释放
        freeChunk(chunk, handle, sizeClass);
    }
}
```



### 添加缓存

2. ``cache.add(this, chunk, handle, normCapacity, sizeClass)``

```java
boolean add(PoolArena<?> area, PoolChunk chunk, long handle, int normCapacity, SizeClass sizeClass) {
    // 2.1 得到tiny/normal规格的MemoryRegionCache
    MemoryRegionCache<?> cache = cache(area, normCapacity, sizeClass);
    // 没有cache则直接返回false
    if (cache == null) {
        return false;
    }
    // 2.2 将相应的PoolChunk和handle信息添加到缓存，方便下次取出
    return cache.add(chunk, handle);
}
```



2.1 ``MemoryRegionCache#cache(area, normCapacity, sizeClass);``得到tiny规格的``MemoryRegionCache``

```java
private MemoryRegionCache<?> cacheForTiny(PoolArena<?> area, int normCapacity) {
    //2.1.1 获取subPage的idex，16则对应的idx为1
    int idx = PoolArena.tinyIdx(normCapacity);
    if (area.isDirect()) {
        //2.1.2 通过idx从tinySubPageDirectCaches中找出对应的MemoryRegionCache
        return cache(tinySubPageDirectCaches, idx);
    }
    return cache(tinySubPageHeapCaches, idx);
}
```



2.1.2 ``cache(tinySubPageDirectCaches, idx);``

这里``tinySubPageDirectCaches``是``PooledByteBufAllocator``创建时，创建线程本地变量``PoolThreadCache``时初始化的。

```java
private final MemoryRegionCache<ByteBuffer>[] tinySubPageDirectCaches;

/**
* 创建tiny规格的MemoryRegionCache
* cacheSize: 单位缓存的大小，默认512字节
* numCaches: 创建cache数组的数量，默认32
*/
private static <T> MemoryRegionCache<T>[] createSubPageCaches(
    int cacheSize, int numCaches, SizeClass sizeClass) {
    if (cacheSize > 0 && numCaches > 0) {
        @SuppressWarnings("unchecked")
        MemoryRegionCache<T>[] cache = new MemoryRegionCache[numCaches];
        for (int i = 0; i < cache.length; i++) {
            // TODO: maybe use cacheSize / cache.length
            cache[i] = new SubPageMemoryRegionCache<T>(cacheSize, sizeClass);
        }
        return cache;
    } else {
        return null;
    }
}
```



2.2 将相应的PoolChunk和handle信息添加到缓存，方便下次取出

``MemoryRegionCache#add(PoolChunk<T> chunk, long handle)``

```java
public final boolean add(PoolChunk<T> chunk, long handle) {
    //2.2.1 创建Entry对象并添加到queue里
    Entry<T> entry = newEntry(chunk, handle);
    boolean queued = queue.offer(entry);
    //2.2.2 如果加入queue失败
    if (!queued) {
        // If it was not possible to cache the chunk, immediately recycle the entry
        entry.recycle();
    }

    return queued;
}
```

首先，我们看看``PoolThreadCache$Entry``结构

```java
static final class Entry<T> {
    final Handle<Entry<?>> recyclerHandle;//回收handle
    PoolChunk<T> chunk;//具体某一块内存
    long handle = -1;//chunk+handle：指向某一块内存的位置，handle是一个相对地址

    Entry(Handle<Entry<?>> recyclerHandle) {
        this.recyclerHandle = recyclerHandle;
    }

    void recycle() {
        chunk = null;
        handle = -1;
        recyclerHandle.recycle(this);
    }
}
```



2.2.1 创建Entry对象

``PoolThreadCache#newEntry()``

```java
private static Entry newEntry(PoolChunk<?> chunk, long handle) {
    //从对象池里取Entry，减少gc
    Entry entry = RECYCLER.get();
    entry.chunk = chunk;
    entry.handle = handle;
    return entry;
}
```



### 添加缓存失败

1.3 未成功添加缓存则释放

``PoolArena#freeChunk(chunk, handle, sizeClass);``

```java
void freeChunk(PoolChunk<T> chunk, long handle, SizeClass sizeClass) {
    final boolean destroyChunk;
    synchronized (this) {
        switch (sizeClass) {
        case Normal:
            //统计normal规格的释放数量
            ++deallocationsNormal;
            break;
        case Small:
            ++deallocationsSmall;
            break;
        case Tiny:
            ++deallocationsTiny;
            break;
        default:
            throw new Error();
        }
        //1.3.1 根据不同的规格释放在chunk上指向的内存
        destroyChunk = !chunk.parent.free(chunk, handle);
    }
    //1.3.2 通过调用UNSAFE的本地方法，清理内存
    if (destroyChunk) {
        // destroyChunk not need to be called while holding the synchronized lock.
        destroyChunk(chunk);
    }
}
```



1.3.1  根据不同的规格释放在chunk上指向的内存

```java
boolean free(PoolChunk<T> chunk, long handle) {
    chunk.free(handle);
    if (chunk.usage() < minUsage) {
        remove(chunk);
        // Move the PoolChunk down the PoolChunkList linked-list.
        return move0(chunk);
    }
    return true;
}


void free(long handle) {
    int memoryMapIdx = memoryMapIdx(handle);
    int bitmapIdx = bitmapIdx(handle);
	
    //根据handle来释放page
    if (bitmapIdx != 0) { // free a subpage
        PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
        assert subpage != null && subpage.doNotDestroy;


        // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.
        // This is need as we may add it back and so alter the linked-list structure.
        //找到包含头节点的PoolSubpage
        PoolSubpage<T> head = arena.findSubpagePoolHead(subpage.elemSize);
        synchronized (head) {
            //同步将这个位视图的置为0
            if (subpage.free(head, bitmapIdx & 0x3FFFFFFF)) {
                return;
            }
        }
    }
    freeBytes += runLength(memoryMapIdx);
    setValue(memoryMapIdx, depth(memoryMapIdx));
    updateParentsFree(memoryMapIdx);
}
```

