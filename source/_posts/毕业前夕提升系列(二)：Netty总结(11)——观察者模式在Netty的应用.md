---
IINAIINAIINAtitle: 毕业前夕提升系列(二)：Netty总结(11)——观察者模式在Netty的应用
tag: 
- summary
categories: Netty
---

netty中在使用``writeAndFlush``方法最终在``doWrite()``方法，是通过NIO的socketChannel的``write``方法，该方法在设置none-blocking模式下是异步写入的，如果client需要在写入成功后加上一些后续逻辑，一般是下面做法:

```java
ChannelFuture channelFuture = chan.writeAndFlush(object);
channelFuture.addListener((future) -> {
    if (future.isSuccess()) {

    } eles {
        
    }
});
```

## Netty的Promise

netty中的Promise类就是观察者的体现，我们用``writeAndFlush``来跟踪一下源码

``writeAndFlush()``

```java
@Override
public ChannelFuture writeAndFlush(Object msg) {
    return writeAndFlush(msg, newPromise());
}
```



### 创建Promise对象

``AbstractChannelHandlerContext#newPromise()``

```java
@Override
public ChannelPromise newPromise() {
    return new DefaultChannelPromise(channel(), executor());
}
```

``DefaultPromise``类实现了``Promise``：``public class DefaultPromise<V> extends AbstractFuture<V> implements Promise<V>``

``Promise``最终实现的jdk的``Future``



### Promise的传递

```java
@Override
public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    if (msg == null) {
        throw new NullPointerException("msg");
    }

    if (isNotValidPromise(promise, true)) {
        ReferenceCountUtil.release(msg);
        // cancelled
        return promise;
    }

    //2. ctx传播，执行ctx的写入方法
    write(msg, true, promise);

    return promise;
}
```



``AbstractChannelHandlerContext#write()``

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            //2.1 调用invokeWriteAndFlush()方法，promise代入
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
        //2.2 将包装好的task交付给线程池
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



``DefaultChannelPipeline$HeadContext#write()``

```java
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
```



``AbstractChannel$AbstractUnsafe#write()``

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
      
  			//2.1.2 channel已经close了，这里设置失败状态
        safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
        // release message now to prevent resource-leak
        ReferenceCountUtil.release(msg);
        return;
    }

    int size;
    try {
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

  	//2.1.3 将promise和消息一起包装成Entry对象塞到Entry链的末尾
    outboundBuffer.addMessage(msg, size, promise);
}
```



目前promise的流动：writeAndFlush()->pipeline上->最终到HeadContext，调用unsafe的write方法写入，将将promise和消息一起包装成Entry对象塞到Entry链的末尾，随着后续的flush操作的成功与否对promise设定成功或失败



### 设置失败结果

 2.1.2  写入判断，如果channel已经close了，这里设置失败状态

``AbstractChannel#safeSetFailure()``

```java
protected final void safeSetFailure(ChannelPromise promise, Throwable cause) {
    //2.1.2.1 尝试设置失败状态
    if (!(promise instanceof VoidChannelPromise) && !promise.tryFailure(cause)) {
        logger.warn("Failed to mark a promise as failure because it's done already: {}", promise, cause);
    }
}
```



2.1.2.1 尝试设置失败状态

``promise.tryFailure(cause)``

```java
@Override
public boolean tryFailure(Throwable cause) {
   //2.1.2.1.1 CAS设置状态
    if (setFailure0(cause)) {
        notifyListeners();
        return true;
    }
    return false;
}
```



2.1.2.1.1 CAS设置状态 ``DefaultPromise#setValue0``

```java
private boolean setValue0(Object objResult) {
  //如果这个没有结果或者还处于未取消的结果
    if (RESULT_UPDATER.compareAndSet(this, null, objResult) ||
        RESULT_UPDATER.compareAndSet(this, UNCANCELLABLE, objResult)) {
      //如果有监听者，notifyAll()
        checkNotifyWaiters();
        return true;
    }
    return false;
}
```



### flush过程中改变promise的结果

当成功写完，会调用``ChannelOutboundBuffer#remove()``，其中会设置promise为SUCCESS，通知监听者，执行回调函数。

``AbstractNioByteChannel$NioByteUnsafe#doWriteInternal()``

```java
private int doWriteInternal(ChannelOutboundBuffer in, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (!buf.isReadable()) {
          //3.1 当该buf已经写完了
            in.remove();
            return 0;
        }

        final int localFlushedAmount = doWriteBytes(buf);
        if (localFlushedAmount > 0) {
            in.progress(localFlushedAmount);
            if (!buf.isReadable()) {
                in.remove();
            }
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
        //3.1 设置promise为成功
        safeSuccess(promise);
        decrementPendingOutboundBytes(size, false, true);
    }

    // recycle the entry
    e.recycle();

    return true;
}
```



3.1 设置promise为成功

```java
private static void safeSuccess(ChannelPromise promise) {
    // Only log if the given promise is not of type VoidChannelPromise as trySuccess(...) is expected to return
    // false.
    PromiseNotificationUtil.trySuccess(promise, null, promise instanceof VoidChannelPromise ? null : logger);
}
```

```java
public static <V> void trySuccess(Promise<? super V> p, V result, InternalLogger logger) {
    if (!p.trySuccess(result) && logger != null) {
        Throwable err = p.cause();
        if (err == null) {
            logger.warn("Failed to mark a promise as success because it has succeeded already: {}", p);
        } else {
            logger.warn(
                    "Failed to mark a promise as success because it has failed already: {}, unnotified cause:",
                    p, err);
        }
    }
}
```



``DefaultPromise#trySuccess()``

```java
@Override
public boolean trySuccess(V result) {
  //cas更改result为SUCCESS
    if (setSuccess0(result)) {
      //通知监听者
        notifyListeners();
        return true;
    }
    return false;
}
```



### 通知监听者

``DefaultPromise#notifyListeners()``

```java
private void notifyListeners() {
    EventExecutor executor = executor();
    if (executor.inEventLoop()) {
        final InternalThreadLocalMap threadLocals = InternalThreadLocalMap.get();
        final int stackDepth = threadLocals.futureListenerStackDepth();
      //监听者可以嵌套，nio线程最大嵌套层数为8
        if (stackDepth < MAX_LISTENER_STACK_DEPTH) {
            threadLocals.setFutureListenerStackDepth(stackDepth + 1);
            try {
                notifyListenersNow();
            } finally {
                threadLocals.setFutureListenerStackDepth(stackDepth);
            }
            return;
        }
    }

    safeExecute(executor, new Runnable() {
        @Override
        public void run() {
            notifyListenersNow();
        }
    });
}
```



``DefaultPromise#notifyListeners0()``

```java
private void notifyListeners0(DefaultFutureListeners listeners) {
    GenericFutureListener<?>[] a = listeners.listeners();
    int size = listeners.size();
    for (int i = 0; i < size; i ++) {
        notifyListener0(this, a[i]);
    }
}
```

``DefaultPromise#notifyListener0()``

```java
private static void notifyListener0(Future future, GenericFutureListener l) {
    try {
      //调用监听器的回调函数
        l.operationComplete(future);
    } catch (Throwable t) {
        if (logger.isWarnEnabled()) {
            logger.warn("An exception was thrown by " + l.getClass().getName() + ".operationComplete()", t);
        }
    }
}
```



## 注册观察者

``DefaultPromise#addListener()``

```java
@Override
public Promise<V> addListener(GenericFutureListener<? extends Future<? super V>> listener) {
    checkNotNull(listener, "listener");

   //1. 同步
    synchronized (this) {
      //2. 具体加监听器逻辑
      addListener0(listener);
    }

  //3 添加监听器时，判断一次是否完成
    if (isDone()) {
      notifyListeners();
    }

    return this;
}
```



``DefaultPromise#addListener0()``

```java
private void addListener0(GenericFutureListener<? extends Future<? super V>> listener) {
    if (listeners == null) {
      //第一次添加
        listeners = listener;
      //第三次添加，直接往DefaultFutureListeners对象里的listeners数组里添加
    } else if (listeners instanceof DefaultFutureListeners) {
        ((DefaultFutureListeners) listeners).add(listener);
    } else {
      //第二次添加，将listeners转成DefaultFutureListeners对象
        listeners = new DefaultFutureListeners((GenericFutureListener<?>) listeners, listener);
    }
}
```



1. 这里为什么要同步呢？

监听器没有做channel和eventloop的绑定，其他的线程也可以向其注册监听器

3. 添加监听器时，判断一次是否完成

因为channel的操作都是NIO异步的过程，在添加监听器的时候可能已经完成了操作，所以检测一次是否完成，若执行完成，直接通知监听者



## 参考

[从源码上理解Netty并发工具-Promise](https://www.cnblogs.com/throwable/p/12231878.html)

