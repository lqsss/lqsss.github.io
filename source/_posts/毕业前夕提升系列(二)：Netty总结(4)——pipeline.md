---
title: 毕业前夕提升系列(二)：Netty总结(4)——pipeline
tag: 
- summary
categories: Netty
---




pipeline负责读写事件的传播

<!--more-->

Q1：Netty是如何判断ChannelHandler类型的

Q2：对于ChannelHandler的添加应该遵循什么样的顺序？

Q3：**用户手动触发事件传播**，不同的触发方式有什么样的区别？

## 1. pipeline初始化

### 1.1 发生时机

发生在NioSocketChannel创建时，`newChannelPipeline()`为创建一个ChannelPipeline的过程。每一个新创建的Channel都将会被分配一个新的ChannelPipeline。

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```



### 1.2 pipeline节点结构

pipeline的节点结构：`ChannelHandlerContext`

我们从HeadContext进入发现

```java
public interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker{
      Channel channel();

      EventExecutor executor();//哪一个NIO线程执行

      String name();

      ChannelHandler handler();//节点处理器

      boolean isRemoved();//表示这节点是否被移除
  
  		ChannelPipeline pipeline();//属于哪个pipeline
}
```




ChannelHandlerContext的继承父类：

*   `AttributeMap`设置属性

*   `ChannelInboundInvoker`Inbound事件：读事件、注册、active

*   `ChannelOutboundInvoker`Outbound事件：传播写事件

### 1.3 两大哨兵

`TailContext、HeadContext`

#### 1.3.1 TailContext
```java
final class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler {

        TailContext(DefaultChannelPipeline pipeline) {
            super(pipeline, null, TAIL_NAME, true, false);
            setAddComplete();
        }

    	//本身自己也是handler
        @Override
        public ChannelHandler handler() {
            return this;
        }
  
  //未捕获的异常，会在这里捕获
          @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            onUnhandledInboundException(cause);
        }
  //未处理的inbound消息，会在这里(tail)处理
         @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            onUnhandledInboundMessage(msg);
        }
  
  //...
}
```


#### 1.3.2 HeadContext
```java
HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true);//1
    unsafe = pipeline.channel().unsafe();
    setAddComplete();
}
```




与TailContext区别在于，HeadContext是设置的为outbound。

## 2. 往pipeline里添加childHandler

Q:什么时候添加？ 

> A：《毕业前夕提升系列(二)：Netty总结(3)——新连接接入》：accept创建新连接后，传播read事件到ServerBootstrapAcceptor里的channelRead()方法里添加。

流程：

1.  判断是否重复添加
2.  创建节点并添加至链表
3.  回调添加完成事件



```java
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    //每一个channel都有pipeline
    synchronized (this) {
        checkMultiplicity(handler); //2.1 

        newCtx = newContext(group, filterName(name, handler), handler);//2.2

        addLast0(newCtx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);//2.3
    return this;
}
```

### 2.1  判断是否重复添加

``DefaultChannelPipeline#checkMultiplicity``

```java
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        //如果hanlder非共享的，但已经被添加到pipeline
        if (!h.isSharable() && h.added) {
            throw new ChannelPipelineException(
                    h.getClass().getName() +
                    " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        h.added = true;
    }
}
```



### 2.2 创建节点并添加至链表

```java
newCtx = newContext(group, filterName(name, handler), handler);
‘//在tail前插入newCtx
addLast0(newCtx);
```


### 2.3 回调添加完成事件

`DefaultChannelPipeline#callHandlerAdded0(newCtx);`

```java
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
    try {
      // We must call setAddComplete before calling handlerAdded. Otherwise if the handlerAdded method generates
      // any pipeline events ctx.handler() will miss them because the state will not allow it.
      ctx.setAddComplete(); //添加完毕
      ctx.handler().handlerAdded(ctx); //handler添加完成的回调
    } catch (Throwable t) {
      boolean removed = false;
      try {
        remove0(ctx);
        try {
          ctx.handler().handlerRemoved(ctx);
        } finally {
          ctx.setRemoved();
        }
        removed = true;
      } catch (Throwable t2) {
        if (logger.isWarnEnabled()) {
          logger.warn("Failed to remove a handler: " + ctx.name(), t2);
        }
      }

      if (removed) {
        fireExceptionCaught(new ChannelPipelineException(
          ctx.handler().getClass().getName() +
          ".handlerAdded() has thrown an exception; removed.", t));
      } else {
        fireExceptionCaught(new ChannelPipelineException(
          ctx.handler().getClass().getName() +
          ".handlerAdded() has thrown an exception; also failed to remove.", t));
      }
    }
}
```


## 3. 删除childHandler

场景：权限校验，校验成功后就不需要校验handler

``DefaultChannelPipeline#remove``

```java
@Override
public final ChannelPipeline remove(ChannelHandler handler) {
    remove(getContextOrDie(handler));
    return this;
}
```



### 3.1 找到节点

``DefaultChannelPipeline#getContextOrDie``

```java
private AbstractChannelHandlerContext getContextOrDie(ChannelHandler handler) {
  //1. 获取节点
    AbstractChannelHandlerContext ctx = (AbstractChannelHandlerContext) context(handler);
    if (ctx == null) {
        throw new NoSuchElementException(handler.getClass().getName());
    } else {
        return ctx;
    }
}
```



``DefaultChannelPipeline#context``

```java
@Override
public final ChannelHandlerContext context(ChannelHandler handler) {
    if (handler == null) {
        throw new NullPointerException("handler");
    }

    AbstractChannelHandlerContext ctx = head.next;
    for (;;) {

        if (ctx == null) {
            return null;
        }

        if (ctx.handler() == handler) {
            return ctx;
        }

        ctx = ctx.next;
    }
}
```



### 3.2 删除

``DefaultChannelPipeline#remove``

```java
private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
    assert ctx != head && ctx != tail;

    synchronized (this) {
        remove0(ctx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we remove the context from the pipeline and add a task that will call
        // ChannelHandler.handlerRemoved(...) once the channel is registered.
        if (!registered) {
            callHandlerCallbackLater(ctx, false);
            return ctx;
        }

        EventExecutor executor = ctx.executor();
        if (!executor.inEventLoop()) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerRemoved0(ctx);
                }
            });
            return ctx;
        }
    }
    callHandlerRemoved0(ctx);//3
    return ctx;
}
```



### 3.回调删除事件

```java
private void callHandlerRemoved0(final AbstractChannelHandlerContext ctx) {
    // Notify the complete removal.
    try {
        try {
            ctx.handler().handlerRemoved(ctx);//回调处
        } finally {
            ctx.setRemoved();//设置为REMOVE_COMPLETE状态
        }
    } catch (Throwable t) {
        fireExceptionCaught(new ChannelPipelineException(
                ctx.handler().getClass().getName() + ".handlerRemoved() has thrown an exception.", t));
    }
}
```



## inBound事件的传播

![ChannelHandler](https://blog-1257900554.cos.ap-beijing.myqcloud.com/ChannelHandler.png)

channelRead():

*   服务端channel：连接
*   客户端channel：bytebuf数据

### channelRead事件传播

有A、B、C三个InboundHandler，在pipeline里顺序：A-&gt;B-&gt;C，调用`pipeline.fireChannelRead()`传递消息。

```java
@Override
public final ChannelPipeline fireChannelRead(Object msg) {
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}
```



从Head开始传播，找到下一个inbound类型的handle

```java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRead(m);
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRead(m);
            }
        });
    }
}
```

```java
private void invokeChannelRead(Object msg) {
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
						// head的channelRead回调处
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRead(msg);
    }
}
```

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ctx.fireChannelRead(msg); //head传播至后面
}
```

```java
@Override
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(), msg); //循环找到一个inbound类型的handler
    return this;
}
```

```java
private AbstractChannelHandlerContext findContextInbound() {
    AbstractChannelHandlerContext ctx = this;
    do {
        ctx = ctx.next;
    } while (!ctx.inbound);
    return ctx;
}
```




  `ctx.fireChannelRead(msg);`是从当前channelhandlerContext开始往后面传播；而`pipeline.fireChannelRead()`是从head节点开始往后面传播。如果传播到tail节点，还有没处理channelRead事件的话，会debug模式下提醒。

### SimpleInboundHandler处理器
```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    boolean release = true;
    try {
        if (acceptInboundMessage(msg)) {
            @SuppressWarnings("unchecked")
            I imsg = (I) msg;      //1. 转型
            channelRead0(ctx, imsg); //2. 用户代码回调处
        } else {
            release = false;
            ctx.fireChannelRead(msg);
        }
    } finally {
      //3. 自动释放
        if (autoRelease && release) {
            ReferenceCountUtil.release(msg);
        }
    }
}
```




1.  转型
2.  用户代码回调处
3.  帮用户自动释放bytebuf

## outBound事件的传播

outBound和inBound事件的不同：

*   outBound:bind、connect、deregister、write、flush等，给用户主动调用的方法

*   inBound:channelRegister、ChannelActive、ChannelRead、ChannelReadComplete。更多的是事件触发，更多的是一个被动

*   **传播顺序与pipeline里添加的顺序相反。**

### write事件传播

ctx.channel().write()-&gt;pipeline.write()：从tail开始传播写入事件，如果一直传播到head，会调用`unsafe.write()`。

```java
@Override
public final ChannelFuture write(Object msg, ChannelPromise promise) {
    return tail.write(msg, promise);
}
```

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
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
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

    private void invokeWrite(Object msg, ChannelPromise promise) {
        if (invokeHandler()) {
            invokeWrite0(msg, promise);
        } else {
            write(msg, promise);
        }
    }

    private void invokeWrite0(Object msg, ChannelPromise promise) {
        try {
            ((ChannelOutboundHandler) handler()).write(this, msg, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    }
```




*   ctx.write()从当前节点向下传播。

*   ctx.channel().write()从tail开始

## 异常事件传播

`exceptionCaught()`是ctx的捕获异常的回调函数，如果用户没有重写，默认是调用`fireExcetionCaught()`

`fireExcetionCaught()`传递异常方法。

传递顺序：pipeline双向链表的顺序。

如果传播到tail节点，没有捕获的话，会在最后打印出来提醒。

### 应用程序中通常做法

添加一个异常处理器。
