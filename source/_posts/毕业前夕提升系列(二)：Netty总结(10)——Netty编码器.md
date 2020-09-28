---
title: 毕业前夕提升系列(二)：Netty总结(10)——Netty编码器
tag: 
- summary
categories: Netty
---

### Netty编码器

将我们业务层的对象进行encode操作，转换成byte最终写入到channel中，我们一般需要两步：

1. 实现``MessageToByteEncoder`` 类``encode()``方法
2. 作为``ChannelOutboundHandlerAdapter``加入到``ChannelPipeline``



### 编码解密

步骤：

1. 类型检查
2. 分配内存
3. 子类实现的具体编码方法
4. 有可能是ByteBuf对象，会释放掉
5. 传播数据



- 入口是重写``ChannelHandlerAdapter``的``write()``方法，当写入pipeline的数据传到该``ChannelOutboundHandler``触发调用。

```java
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    ByteBuf buf = null;
    try {
        //1. 类型检查,是否是可以接受编码的对象
        if (acceptOutboundMessage(msg)) {
            @SuppressWarnings("unchecked")
            I cast = (I) msg;
            //2. 分配内存
            buf = allocateBuffer(ctx, cast, preferDirect);
            try {
                //3. 子类实现的具体编码方法
                encode(ctx, cast, buf);
            } finally {
                //4. 释放掉
                ReferenceCountUtil.release(cast);
            }

            //5. 传播数据
            if (buf.isReadable()) {
                ctx.write(buf, promise);
            } else {
                buf.release();
                ctx.write(Unpooled.EMPTY_BUFFER, promise);
            }
            buf = null;
        } else {
            ctx.write(msg, promise);
        }
    } catch (EncoderException e) {
        throw e;
    } catch (Throwable e) {
        throw new EncoderException(e);
    } finally {
        if (buf != null) {
            buf.release();
        }
    }
}
```



1. 类型检查,是否是可以接受编码的对象``acceptOutboundMessage(msg)``

```java
public boolean acceptOutboundMessage(Object msg) throws Exception {
    return matcher.match(msg);
}
```

这里的``matcher``为``TypeParameterMatcher``接口实现对象，该接口有个match方法，来检测类型参数是否匹配。

- ``TypeParameterMatcher#get()``

```java
public static TypeParameterMatcher (final Class<?> parameterType) {
    final Map<Class<?>, TypeParameterMatcher> getCache =
        InternalThreadLocalMap.get().typeParameterMatcherGetCache();

    TypeParameterMatcher matcher = getCache.get(parameterType);
    if (matcher == null) {
        if (parameterType == Object.class) {
            matcher = NOOP;
        } else {
            //class自带的isInstance判断类型
            matcher = new ReflectiveMatcher(parameterType);
        }
        getCache.put(parameterType, matcher);
    }

    return matcher;
}
```

2. 分配内存

```java
protected ByteBuf allocateBuffer(ChannelHandlerContext ctx, @SuppressWarnings("unused") I msg,
                               boolean preferDirect) throws Exception {
    	//默认是true，分配堆外内存
        if (preferDirect) {
            return ctx.alloc().ioBuffer();
        } else {
            return ctx.alloc().heapBuffer();
        }
}
```



5. 传播数据

`` AbstractChannelHandlerContext#write()``

```java
@Override
public ChannelFuture write(final Object msg, final ChannelPromise promise) {
    if (msg == null) {
        throw new NullPointerException("msg");
    }

    try {
        if (isNotValidPromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            // cancelled
            return promise;
        }
    } catch (RuntimeException e) {
        ReferenceCountUtil.release(msg);
        throw e;
    }
    write(msg, false, promise);

    return promise;
}
```



```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    //5.1 找到下一个outbound的ctx
    AbstractChannelHandlerContext next = findContextOutbound();
    //5.2 touch：记录一下内存的写入位置
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    //5.3 最终调用下一个ctx重写的write
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        final AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        if (!safeExecute(executor, task, promise, m)) {
            // We failed to submit the AbstractWriteTask. We need to cancel it so we decrement the pending bytes
            // and put it back in the Recycler for re-use later.
            //
            // See https://github.com/netty/netty/issues/8343.
            task.cancel();
        }
    }
}
```



5.3 最终调用下一个ctx重写的write()

`` AbstractChannelHandlerContext#invokeWrite0()``

```java
   private void invokeWrite0(Object msg, ChannelPromise promise) {
        try {
            //最终会到head节点
            ((ChannelOutboundHandler) handler()).write(this, msg, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    }
```



``HeadContext#write()``

```java
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    //调用unsafe的write方法
    unsafe.write(msg, promise);
}
```



`` AbstractUnsafe$AbstractUnsafe#write()``

```java
@Override
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        // If the outboundBuffer is null we know the channel was closed and so
        // need to fail the future right away. If it is not null the handling of the rest
        // will be done in flush0()
        // See https://github.com/netty/netty/issues/2362
        safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
        // release message now to prevent resource-leak
        ReferenceCountUtil.release(msg);
        return;
    }

    int size;
    try {
        //5.3.1 过滤掉一部分不需要的对象
        msg = filterOutboundMessage(msg);
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }

    //5.3.2 加入到缓冲队列中
    outboundBuffer.addMessage(msg, size, promise);
}
```



5.3.1 过滤掉一部分不需要的对象 ``filterOutboundMessage``

```java
@Override
protected final Object filterOutboundMessage(Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (buf.isDirect()) {
            return msg;
        }
		//5.3.1.1 进行堆内内存到堆外内存的转换
        return newDirectBuffer(buf);
    }

    if (msg instanceof FileRegion) {
        return msg;
    }

    throw new UnsupportedOperationException(
        "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
}
```



5.3.2 ``ChannelOutboundBuffer#addMessag()``加入到缓冲队列中

``ChannelOutboundBuffer``里有三个指针：

```java
// Entry(flushedEntry) --> ... Entry(unflushedEntry) --> ... Entry(tailEntry)
//
// The Entry that is the first in the linked-list structure that was flushed
private Entry flushedEntry;
// The Entry which is the first unflushed in the linked-list structure
private Entry unflushedEntry;
// The Entry which represents the tail of the buffer
private Entry tailEntry;
```

- flushedEntry-> unflushedEntry：表示已经flush过的Entry

- unflushedEntry->tailEntry：表示没有被flush过的Entry

```java
public void addMessage(Object msg, int size, ChannelPromise promise) {
    //5.3.2.1 创建一个Entry对象
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
   	//5.3.2.2 设置指针
    //如果tail指针是空的，表示第一次
    if (tailEntry == null) {
        //已flush指针置为空
        flushedEntry = null;
    } else {
        //将当前尾指针指向该entry
        Entry tail = tailEntry;
        tail.next = entry;
    }
    //重置为指针为当前entry
    tailEntry = entry;
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }

    // increment pending bytes after adding message to the unflushed arrays.
    // See https://github.com/netty/netty/issues/1619
    //5.3.2.3 增加待刷新的数据大小
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
```



``incrementPendingOutboundBytes()``

```java
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
    if (size == 0) {
        return;
    }

    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
    //如果新要写入的buffer数据数量大于配置的高水平线
    if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
        //5.3.2.3.1设置为不可写
        setUnwritable(invokeLater);
    }
}
```

``ChannelOutboundBuffer#setUnwritable()``

```java
private void setUnwritable(boolean invokeLater) {
    //自旋更新成员变量unwritable
    for (;;) {
        final int oldValue = unwritable;
        final int newValue = oldValue | 1;
        if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
            //如果之前是可写，变成了不可写，往pipeline里
            if (oldValue == 0 && newValue != 0) {
                fireChannelWritabilityChanged(invokeLater);
            }
            break;
        }
    }
}
```



### 刷新队列

写入队列在上面的编码过程中描述了，最终调用了unsafe的write方法，写入到``ChannelOutboundBuffer``中。我们这里要讲一下flush的过程。

- ``HeadContext#flush()``

```java
@Override
public void flush(ChannelHandlerContext ctx) throws Exception {
    unsafe.flush();
}
```

``unsafe#flush()``

步骤：

1. 添加刷新标志并设置写状态
2. 遍历buffer队列，过滤ByteBuf
3.  调用jdk底层api进行自旋写

```java
@Override
public final void flush() {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }
    
    outboundBuffer.addFlush();
    flush0();
}
```

 

``ChannelOutboundBuffer#addFlush()``

```java
public void addFlush() {
    // There is no need to process all entries if there was already a flush before and no new messages
    // where added in the meantime.
    //
    // See https://github.com/netty/netty/issues/2577
    
    //entry获取还未进行flush操作的第一个元素
    Entry entry = unflushedEntry;
    if (entry != null) {
       
        if (flushedEntry == null) {
            // there is no flushedEntry yet, so start with the entry
            //指向第一个
            flushedEntry = entry;
        }
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) {
                // Was cancelled so make sure we free up memory and notify about the freed bytes
                int pending = entry.cancel();
                //每flush一个对象，需要把对象的大小从总的pendsize
                decrementPendingOutboundBytes(pending, false, true);
            }
            //指向下一个entry
            entry = entry.next;
        } while (entry != null);

        // All flushed so reset unflushedEntry
        unflushedEntry = null;
    }
}
```

``decrementPendingOutboundBytes()``

```java
private void decrementPendingOutboundBytes(long size, boolean invokeLater, boolean notifyWritability) {
    if (size == 0) {
        return;
    }

    //减去pendingSize，更新TOTAL_PENDING_SIZE_UPDATER
    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, -size);
    //当减去pendingSize后的大小小于32时(低水位),整个channel设置可写状态
    if (notifyWritability && newWriteBufferSize < channel.config().getWri teBufferLowWaterMark()) {
        setWritable(invokeLater);
    }
}
```

``AbstractUnsafe#flush0()``

```java
 protected void flush0() {
    if (inFlush0) {
        //已经flush
        // Avoid re-entrance
        return;
    }

    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) {
        return;
    }

    inFlush0 = true;

    // Mark all pending write requests as failure if the channel is inactive.
    if (!isActive()) {
        try {
            if (isOpen()) {
                outboundBuffer.failFlushed(FLUSH0_NOT_YET_CONNECTED_EXCEPTION, true);
            } else {
                // Do not trigger channelWritabilityChanged because the channel is closed already.
                outboundBuffer.failFlushed(FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
            }
        } finally {
            inFlush0 = false;
        }
        return;
    }

    try {
        doWrite(outboundBuffer);
    } catch (Throwable t) {
        if (t instanceof IOException && config().isAutoClose()) {
            /**
                     * Just call {@link #close(ChannelPromise, Throwable, boolean)} here which will take care of
                     * failing all flushed messages and also ensure the actual close of the underlying transport
                     * will happen before the promises are notified.
                     *
                     * This is needed as otherwise {@link #isActive()} , {@link #isOpen()} and {@link #isWritable()}
                     * may still return {@code true} even if the channel should be closed as result of the exception.
                     */
            close(voidPromise(), t, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
        } else {
            try {
                shutdownOutput(voidPromise(), t);
            } catch (Throwable t2) {
                close(voidPromise(), t2, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
            }
        }
    } finally {
        inFlush0 = false;
    }
}
```

   

``AbstractNioByteChannel#doWrite()``

``AbstractNioByteChannel``实现了``AbstractChannel#doWrite()``

```java
@Override
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    //3. 自旋写入底层
    //自选写次数，默认是16次
    int writeSpinCount = config().getWriteSpinCount();
    do {
   		//3.1. 获取flushedEntry的entry 
        Object msg = in.current();
        if (msg == null) {
            // Wrote all messages.
            //如果没有要写入的数据了，取消注册到selector的OP_WRITE事件
            clearOpWrite();
            // Directly return here so incompleteWrite(...) is not called.
            return;
        }
        //3.2 写入底层socket数据
        writeSpinCount -= doWriteInternal(in, msg);
    } while (writeSpinCount > 0);

    //3.3 完成写过程后，对写标识的处理
    incompleteWrite(writeSpinCount < 0);
}
```



3.2 写入底层socket数据``AbstractNioByteChannel#doWriteInternal()``

```java
private int doWriteInternal(ChannelOutboundBuffer in, Object msg) throws Exception {
        if (msg instanceof ByteBuf) {
            ByteBuf buf = (ByteBuf) msg;
            //writerIndex - readerIndex >0 ? true: flase
            if (!buf.isReadable()) { 
                in.remove();
                return 0;
            }

            //3.2.1 写入底层socket传输数据，并记录下flush的数目
            final int localFlushedAmount = doWriteBytes(buf);
            //如果写入了数据
            if (localFlushedAmount > 0) {
                //3.2.2 记录一下进度
                in.progress(localFlushedAmount);
                if (!buf.isReadable()) {
                    //如果发送完了，就从ChannelOutboundBuffer中
                    in.remove();
                }
                //出现写半包的情况，直接返回1，外层自旋继续写入
                return 1;
            }
        } else if (msg instanceof FileRegion) {
            FileRegion region = (FileRegion) msg;
            if (region.transferred() >= region.count()) {
                in.remove();
                return 0;
            }

            long localFlushedAmount = doWriteFileRegion(region);
            if (localFlushedAmount > 0) {
                in.progress(localFlushedAmount);
                if (region.transferred() >= region.count()) {
                    in.remove();
                }
                return 1;
            }
        } else {
            // Should not reach here.
            throw new Error();
        }
        return WRITE_STATUS_SNDBUF_FULL;
}
```

3.2.1 写入底层socket传输数据，并记录下flush的数目 ``doWriteBytes``

```java
@Override
protected int doWriteBytes(ByteBuf buf) throws Exception {
    final int expectedWrittenBytes = buf.readableBytes();
    return buf.readBytes(javaChannel(), expectedWrittenBytes);
}
```

最终会调用到``PooledDirectByteBuf#getBytes()``

```java
private int getBytes(int index, GatheringByteChannel out, int length, boolean internal) throws IOException {
        checkIndex(index, length);
        if (length == 0) {
            return 0;
        }

        ByteBuffer tmpBuf;
        if (internal) {
            tmpBuf = internalNioBuffer();
        } else {
            tmpBuf = memory.duplicate();
        }
        index = idx(index);
        tmpBuf.clear().position(index).limit(index + length);
        return out.write(tmpBuf);
    }
}
```

一般是``PooledDirectBuf``，最终转成nio的``ByteBuffer``，写入到jdk原生的socket channel



3.2.2 记录一下进度，如果已经全部写完就删除

``ChannelOutboundBuffer#remove()``

```java
public boolean remove() {
    Entry e = flushedEntry;
    if (e == null) {
        clearNioBuffers();
        return false;
    }
    Object msg = e.msg;

    ChannelPromise promise = e.promise;
    int size = e.pendingSize;

    removeEntry(e);

    if (!e.cancelled) {
        // only release message, notify and decrement if it was not canceled before.
        ReferenceCountUtil.safeRelease(msg);
        safeSuccess(promise);
        decrementPendingOutboundBytes(size, false, true);
    }

    // recycle the entry
    e.recycle();

    return true;
}
```

``ChannelOutboundBuffer#removeEntry()``

```java
private void removeEntry(Entry e) {
    //flushed:The number of flushed entries that are not written yet
    if (-- flushed == 0) {
        //已经没有待flush操作的entry，清空指针
        // processed everything
        flushedEntry = null;
        if (e == tailEntry) {
            tailEntry = null;
            unflushedEntry = null;
        }
    } else {
        flushedEntry = e.next;
    }
}
```



3.3 完成写过程后，对写标识的处理incompleteWrite()``

- setOpWrite：写入16次半包，仍旧没有写完数据，一般是缓冲区满了返回0（``out.write(tmpBuf)``），此时返回给外层 ``WRITE_STATUS_SNDBUF_FULL``整数最大值，writeSpinCount小于0（即这里传入的setOpWrite参数有为true）

```java
protected final void incompleteWrite(boolean setOpWrite) {
    // Did not write completely.
    //3.3.1 说明还没有写完，在selector上注册写标识
    if (setOpWrite) {
        setOpWrite();
    } else {
        // It is possible that we have set the write OP, woken up by NIO because the socket is writable, and then
        // use our write quantum. In this case we no longer want to set the write OP because the socket is still
        // writable (as far as we know). We will find out next time we attempt to write if the socket is writable
        // and set the write OP if necessary.
       //3.3.2 写完了，清理Selector上注册的写标识。稍后再执行刷新计划
        clearOpWrite();

        // Schedule flush again later so other tasks can be picked up in the meantime
        eventLoop().execute(flushTask);
    }
}
```



3.3.1 说明还没有写完，在selector上注册写标识``AbstractNioByteChannel#setOpWrite()``

继希望于selector的下次轮询，待事件循环里检查到写标志，则执行强制flush操作``ch.unsafe().forceFlush();``

```java
protected final void setOpWrite() {
    final SelectionKey key = selectionKey();
    // Check first if the key is still valid as it may be canceled as part of the deregistration
    // from the EventLoop
    // See https://github.com/netty/netty/issues/2104
    //判断key是否还有效
    if (!key.isValid()) {
        return;
    }
    final int interestOps = key.interestOps();
    //判断Selector上是否注册了OP_WRITE标识，如果没有则注册上。
    if ((interestOps & SelectionKey.OP_WRITE) == 0) {
        key.interestOps(interestOps | SelectionKey.OP_WRITE);
    }
}
```

## 参考

[恶劣的网络环境下，Netty是如何处理写事件的?](https://www.cnblogs.com/kubixuesheng/p/12736350.html#_label0_0)

[Netty 之 AbstractNioByteChannel 源码分析](https://www.jianshu.com/p/d82fc1c52805)