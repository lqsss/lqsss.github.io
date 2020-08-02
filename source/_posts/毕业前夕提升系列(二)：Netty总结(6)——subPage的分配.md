---
title: 毕业前夕提升系列(二)：Netty总结(6)——subPage的分配
tag: 
- summary
categories: Netty
---

## 毕业前夕提升系列(二)：Netty总结(6)——subPage的分配



前言：之前是大于一个pageSize(8k)则在一个PoolChunk(6M)里进行分配，

前面逻辑还是跟page级别内存分配类似：

1. 内存规格化，将需要分配的内存进行两倍化(page、subPage级别规格)
2. 首先是从cache上分配
3. 如果cache上无法分配的话，从poolChunkList上分配
   1. 若chunk为null（首次）， chunk（初始化depth、memory数组、PoolSubpage数组）

 

### PoolSubpage数组

在``PoolArena``初始化PoolSubpage数组

```java
static final int numTinySubpagePools = 512 >>> 4;
tinySubpagePools = newSubpagePoolArray(numTinySubpagePools);
for (int i = 0; i < tinySubpagePools.length; i ++) {	
	tinySubpagePools[i] = newSubpagePoolHead(pageSize);
}
```

这里初始化的是头部节点，会创建出32个PoolSubpage，（0B、16B、32B...）都是16的倍数（最小是16字节），数组每一个head节点后面是链接的是具体分配相对应的subPage

![image-20200414213909671](C:\Users\51344\AppData\Roaming\Typora\typora-user-images\image-20200414213909671.png)

  ### subPage分配：allocateTiny

1. 定位一个subPage对象
2. 初始化subPage
3. PooledByteBuf初始化，拿到内存信息给pooledByteBuf进行初始化

核心分配逻辑``PoolChunk#allocateSubpage()`` 



```java
  private long allocateSubpage(int normCapacity) {

    // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.

    // This is need as we may add it back and so alter the linked-list structure.

	//1. 找到合适下标的PoolSubpage

    PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);

    synchronized (head) {

	//分配subPage是chunk一个page(8k)的子节点，所以最后一层11

    int d = maxOrder; // subpages are only be allocated from pages i.e., leaves

	//通过memoryMap数组找到

      int id = allocateNode(d);

      if (id < 0) {

        return id;

      }

 
	//获取PoolChunk初始化的subpage，一共2048个PoolSubPage
      final PoolSubpage<T>[] subpages = this.subpages;
      final int pageSize = this.pageSize;

      freeBytes -= pageSize;

      int subpageIdx = subpageIdx(id);

      PoolSubpage<T> subpage = subpages[subpageIdx];

        //1. 首次找到subPage需要将进行初始化
      if (subpage == null) {

        subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
		
        //将当前下标的
        subpages[subpageIdx] = subpage;

      } else {

        subpage.init(head, normCapacity);

      }

        //2. 从poolSubPage对象中分配
      return subpage.allocate();

    }

  }
```



#### 1. PoolSubpage的构造

```java
//PoolSubpage构造函数
	PoolSubpage(PoolSubpage<T> head, PoolChunk<T> chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize) {
        this.chunk = chunk;
        this.memoryMapIdx = memoryMapIdx;
        this.runOffset = runOffset;
        this.pageSize = pageSize;
        bitmap = new long[pageSize >>> 10]; // pageSize / 16 / 64
        init(head, elemSize);
    }

//初始化
    void init(PoolSubpage<T> head, int elemSize) {
        doNotDestroy = true;
        this.elemSize = elemSize;
        if (elemSize != 0) {
            maxNumElems = numAvail = pageSize / elemSize;
            nextAvail = 0;
            bitmapLength = maxNumElems >>> 6;
            if ((maxNumElems & 63) != 0) {
                bitmapLength ++;
            }

            for (int i = 0; i < bitmapLength; i ++) {
                bitmap[i] = 0;
            }
        }
        addToPool(head);
    }
```



1. 每个chunk都会维护一个subpage列表（数量2048），存放page被切分后的子page
2. 通过bimap来维护哪一个index的subpage被分配，0表示未分配，1表示已分配
   1. element代表subpage更小的单位，elementSize则表示一个element的大小，是外部传进的规格化内存分配大小（16byte*2的n次方）
   2. pageSize为8k，``bitmap = new long[pageSize >>> 10]; // pageSize / 16 / 64``，16是基础字节大小，page中element的数量是不会多于 pageSize/16个的，每个long有64位，每位便可标识一个element的使用情况。
   3. init代码：例如分配16byte的空间，将page/elemSize就是把当前page切分成多少个子page（elements），bitmap是long数组，所以再除以64，就是每一个bit位来表示64个子page的分配情况。（例如分配16byte，则当前page就有512个elements）
      1. ``addToPool（head）``将当前创建的节点连接到head节点后



注：这里每个chunk（page级别）有一个subPage[2048]、之前有Arean内也有subPage[32]，区别?

实际上两者之间的关系可以用下面的数据结构来展示：

![image-20200414213909671](C:\Users\51344\AppData\Roaming\Typora\typora-user-images\image-20200414213909671.png)

整个Chunk二叉树的最底层才可以分配，0~8k为第0个subPage、依次类推存入Chunk里的subpages数组里；而Arean的head是空的，来方便找寻后面链表中合适的subPage进行分配（找16B的则找idx为0的head节点进行分配）

#### 2. 从poolSubPage对象中分配

```java
    long allocate() {
        if (elemSize == 0) {
            return toHandle(0);
        }

        if (numAvail == 0 || !doNotDestroy) {
            return -1;
        }

        //2.1 拿到可用的bitMapIdx
        final int bitmapIdx = getNextAvail();
        //todo
        int q = bitmapIdx >>> 6;
        int r = bitmapIdx & 63;
        assert (bitmap[q] >>> r & 1) == 0;
        bitmap[q] |= 1L << r;

        //numAvail为0，表示当前subPage的elments被分配完了，把当前subPage从pool中移除
        if (-- numAvail == 0) {
            removeFromPool();
        }

        return toHandle(bitmapIdx);
    }
```



2.1 拿到合适的可用的bitMapIdx：``getNextAvail()``

```java
    private int getNextAvail() {
        //获得当前subPage可用bit位下标
        int nextAvail = this.nextAvail;
        if (nextAvail >= 0) {
        //当前可用bit下标是大于0的，则表明可用，赋值为不可用并返回
            this.nextAvail = -1;
            return nextAvail;
        }
        //如果没有找
        return findNextAvail();
    }

	private int findNextAvail() {
        final long[] bitmap = this.bitmap;
        final int bitmapLength = this.bitmapLength;
        for (int i = 0; i < bitmapLength; i ++) {
            long bits = bitmap[i];
            if (~bits != 0) {
                return findNextAvail0(i, bits);
            }
        }
        return -1;
    }

    private int findNextAvail0(int i, long bits) {
        final int maxNumElems = this.maxNumElems;
        final int baseVal = i << 6;

        for (int j = 0; j < 64; j ++) {
            if ((bits & 1) == 0) {
                int val = baseVal | j;
                if (val < maxNumElems) {
                    return val;
                } else {
                    break;
                }
            }
            bits >>>= 1;
        }
        return -1;
    }
```

2.2 toHandle(bitmapIdx) 

//todo

```java
   private long toHandle(int bitmapIdx) {
        return 0x4000000000000000L | (long) bitmapIdx << 32 | memoryMapIdx;
    }
```

handle指向的是当前chunk中的唯一的一块内存（哪一个chunk的哪一个page）,通过这个long，就可以找到对应的chunk，subpage以及element的位置信息

handle的组成信息是由高32位bitmapIdex和低32位memoryMapIdx组成



### buf的初始化

chunk上根据前面拿到的handle进行buf的初始化

```java
c.initBuf(buf, handle, reqCapacity);

 void initBuf(PooledByteBuf<T> buf, long handle, int reqCapacity) {
     //从handle拿到page偏移量(如果是初次分配16b：2048)
        int memoryMapIdx = memoryMapIdx(handle);
      //从handle拿到bitmapIdx偏移量(bit位置)
        int bitmapIdx = bitmapIdx(handle);
        if (bitmapIdx == 0) {
            byte val = value(memoryMapIdx);
            assert val == unusable : String.valueOf(val);
            buf.init(this, handle, runOffset(memoryMapIdx) + offset, reqCapacity, runLength(memoryMapIdx),
                     arena.parent.threadCache());
        } else {
            initBufWithSubpage(buf, handle, bitmapIdx, reqCapacity);
        }
    }

    private void initBufWithSubpage(PooledByteBuf<T> buf, long handle, int bitmapIdx, int reqCapacity) {
        assert bitmapIdx != 0;

        int memoryMapIdx = memoryMapIdx(handle);

        //拿到0到8k的subPage （memoryMapIdx = 2048）
        PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
        assert subpage.doNotDestroy;
        assert reqCapacity <= subpage.elemSize;

        buf.init(
            this, handle,
            runOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) * subpage.elemSize + offset,
                reqCapacity, subpage.elemSize, arena.parent.threadCache());
    }
```



注：``runOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) * subpage.elemSize + offset``page偏移量加上element的bit位置的偏移量*一个element的大小加上PoolChunk的地址就是buf的地址

## 参考

[Netty之SubPage级别的内存分配](https://www.cnblogs.com/wuzhenzhao/p/11304496.html)

[五、netty的内存管理](https://my.oschina.net/vqishiyu/blog/2998040)

