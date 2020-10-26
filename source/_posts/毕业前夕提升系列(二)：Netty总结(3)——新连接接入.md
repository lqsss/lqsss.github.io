---
title: 毕业前夕提升系列(二)：毕业前夕提升系列(二)：Netty总结(3)——新连接接入
tag: 
- summary
categories: Netty
---



## 问题

1.  在哪里检测新连接

2.  新连接是怎么注册到NioEventLoop的
   <!--more-->

## 1. 检测新连接

**检测新连接**发生在Boss线程在事件循环里的第二步骤(处理到来的I/O事件`processSelectedKeys`)。

``NioEventLoop#processSelectedKey(SelectionKey k, AbstractNioChannel ch)``

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    //1.1 获取NioSocketChannel的unsafe(封装的底层读写连接操作)
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe(); 

    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop != this || eventLoop == null) {
            return;
        }
        // close the channel if the key is not valid anymore
        unsafe.close(unsafe.voidPromise());
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
          //1.2 新连接到来事件，调用read
            unsafe.read();
          
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```



1.2 ``NioMessageUnsafe#read()``

```java
private final class NioMessageUnsafe extends AbstractNioUnsafe {
	//存储读到的连接(NioSocketChannel对象)
    private final List<Object> readBuf = new ArrayList<Object>();


    @Override
    public void read() {
        //1.2.1 断言是否是NioEventLoop
        assert eventLoop().inEventLoop();

        final ChannelConfig config = config();
        final ChannelPipeline pipeline = pipeline();
        //1.2.2 处理服务端接收的速率
        final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
        allocHandle.reset(config);

        boolean closed = false;
        Throwable exception = null;
      
        try {
            try {
                do {
                    
                    //1.2.3 accept接收新来的连接,jdk底层的accept，创建channel加到readBuf
                    int localRead = doReadMessages(readBuf);
                 
                    if (localRead == 0) {
                        break;
                    }
                    if (localRead < 0) {
                        closed = true;
                        break;
                    }
					//1.2.4 增加接收连接数
                    allocHandle.incMessagesRead(localRead);
                  //1.2.5 是否自动读配置 && 读取的连接数比一次read最大的连接数(16)小
                } while (allocHandle.continueReading());
              
            } catch (Throwable t) {
                exception = t;
            }

            int size = readBuf.size();
            for (int i = 0; i < size; i ++) {
                readPending = false;
                pipeline.fireChannelRead(readBuf.get(i));
            }
            readBuf.clear();
            allocHandle.readComplete();
            pipeline.fireChannelReadComplete();

            if (exception != null) {
                closed = closeOnReadError(exception);

                pipeline.fireExceptionCaught(exception);
            }

            if (closed) {
                inputShutdown = true;
                if (isOpen()) {
                    close(voidPromise());
                }
            }
        } finally {
            // Check if there is a readPending which was not processed yet.
            // This could be for two reasons:
            // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
            // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
            //
            // See https://github.com/netty/netty/issues/2254
            if (!readPending && !config.isAutoRead()) {
                removeReadOp();
            }
        }
    }
}
```



**总结：**

1.  **`processSelectedKeys`处理到来的I/O事件(accept)**
2.  while循环accept接收，创建新的NioSocketChannel存入readBuf

## 2. NioSocketChannel的创建

当accept一个jdk底层的NioChannel时，需要新包装一个Netty需要的NioSocketChannel。``buf.add(new NioSocketChannel(this, ch));``

构造函数：

```java
public NioSocketChannel(Channel parent, SocketChannel socket) {
    //2.1 调用父类构造函数
    super(parent, socket);
    //2.2 创建config
    config = new NioSocketChannelConfig(this, socket.socket());
}
```



#### 2.1 调用父类构造函数 

``AbstractNioChannel``

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent); //2.1.1 传入服务端channel
    this.ch = ch;  //2.1.2 保存jdk的客户端channel
    this.readInterestOp = readInterestOp;//2.1.3 保存感兴趣的事件，这里传的是读事件
    try {
      ch.configureBlocking(false);//2.1.4 设置阻塞模型为非阻塞模式
    } catch (IOException e) {
      try {
        ch.close();
      } catch (IOException e2) {
        if (logger.isWarnEnabled()) {
          logger.warn(
            "Failed to close a partially initialized socket.", e2);
        }
      }

      throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
```

``AbstractChannel``

```java
protected AbstractChannel(Channel parent) {
       this.parent = parent; //服务端channel
       id = newId();
       unsafe = newUnsafe();
       pipeline = newChannelPipeline();
}
```

**创建流程如下：**

2.1.1 创建`id`、`unsafe`、`pipeline`，其中客户端channel持有的unsafe是`NioSocketChannelUnsafe`

2.1.2 保存jdk的客户端channel

2.1.3 保存感兴趣的事件——`SelectionKey.OP_READ`

2.1.4 设置阻塞模型为非阻塞模式



#### 2.2 创建config

新建客户端channel的配置——`NioSocketChannelConfig`。设置`TcpNoDelay`为`true`。

``DefaultSocketChannelConfig#new()``

```java
public DefaultSocketChannelConfig(SocketChannel channel, Socket javaSocket) {
        super(channel);
        if (javaSocket == null) {
            throw new NullPointerException("javaSocket");
        }
        this.javaSocket = javaSocket;

        // Enable TCP_NODELAY by default if possible.
        if (PlatformDependent.canEnableTcpNoDelayByDefault()) {
            try {
                setTcpNoDelay(true);
            } catch (Exception e) {
                // Ignore.
            }
        }
}
```



## 3. channel分类

![image-20190612200125222](https://blog-1257900554.cos.ap-beijing.myqcloud.com/image-20190612200125222.png)

### 3.1 AbstractChannel骨架类
```java
private final Channel parent;
private final ChannelId id;
private final Unsafe unsafe;
private final DefaultChannelPipeline pipeline;
private volatile SocketAddress localAddress;
private volatile SocketAddress remoteAddress;
private volatile EventLoop eventLoop;
```


### 3.2 AbstractNioChannel
```java
private final SelectableChannel ch;
protected final int readInterestOp;
volatile SelectionKey selectionKey;
```


比骨架类多了select部分的功能，select进行读写方式的监听、保存jdk底层channel、设置非阻塞

### 3.3 NioServerSocketChannel、NioSocketChannel

不同：

1.  向AbstractNioChannel父类构造函数里select注册的事件不同，服务端是accept事件，客户端是read事件
2.  底层unsafe不同，实现每一种读写逻辑不同，客户端是NioByteUnsafe 、服务端是NioMessageUnsafe
    - NioMessageUnsafe的读，调用底层socket的accept读取连接
    - NioByteUnsafe的读，读数据

## 4. 新连接接入

### ServerBootstrap里的ServerBootstrapAcceptor

在初始化channel(ServerBootstrap里的init()方法里)，该channel是服务端channel。

```java
@Override
void init(Channel channel) throws Exception {            
	ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
}
```


pipeline里是head-&gt;**ServerBootstrapAcceptor**-&gt;tail

### 新连接后续接入过程

4.1 NioMessageUnsafe的read()

```java
@Override
public void read() { 
  //...
	int size = readBuf.size();
      for (int i = 0; i < size; i ++) {
            readPending = false;
            pipeline.fireChannelRead(readBuf.get(i));//这里传播read事件
      }
}
```

4.2 传播到``ServerBootstrapAcceptor#channelRead()``

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;
    
	//4.2 1.添加用户自定义处理器childHandler，这里添加的是我们自定义实现的ChannelInitializer
    child.pipeline().addLast(childHandler); 


    setChannelOptions(child, childOptions, logger);
    //4.2 2.设置childOptions

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
      child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }
    //4.2 2 childAttrs
    try {
      //4.2 3 选择NioEventLoop并注册selector
      //之前是由BOSS线程组的NIO线程read()->发起的register行为(当前线程)，而这里的eventLoop从childGroup传来
      childGroup.register(child).addListener(new ChannelFutureListener() {                  
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
          if (!future.isSuccess()) {
            forceClose(child, future.cause());
          }
        }
      });
      //3. 
    } catch (Throwable t) {
      forceClose(child, t);
    }
}
```



4.2.1 添加用户自定义处理器childHandler的回调函数

``ChannelInitializer#handlerAdded``

```java
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isRegistered()) {
        // This should always be true with our current DefaultChannelPipeline implementation.
        // The good thing about calling initChannel(...) in handlerAdded(...) is that there will be no ordering
        // surprises if a ChannelInitializer will add another ChannelInitializer. This is as all handlers
        // will be added in the expected order.
        //4.2.1.1 初始化channel，添加用户自定handler
        initChannel(ctx);
    }
}
```



4.2.1.1 ``ChannelInitializer#initChannel``

```java
 private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
        if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
            try {
                //4.2.1.1 nioSocketChannel的实现方法，用户代码在pipeline添加自定义hanler
                initChannel((C) ctx.channel());
            } catch (Throwable cause) {
                // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
                // We do so to prevent multiple calls to initChannel(...).
                exceptionCaught(ctx, cause);
            } finally {
                ////4.2.1.2 删除当前ctx，也就是ChannelInitializer
                remove(ctx);
            }
            return true;
        }
        return false;
    }
```



触发read事件到ServerBootstrapAcceptor会做几件事情：

*   添加用户自定义处理器childHandler
*   设置设置childOptions、childAttrs
*   选择NioEventLoop跟channel绑定，并将该channel注册到selector

### NioSocketChannel读事件注册

该小节是讲述的register里的过程

1.  触发ChannelActive-&gt;head
2.  readIfIsAutoRead-&gt;pipeline.read()-&gt;unsafe.beginRead()-&gt;doBeginRead()
3.  selectionKey注册一个readInterestOp

## 总结

1.  检测新连接
2.  创建NioSocketChannel，主要是unsafe、pipeline
3.  通过ServerBootstrapAcceptor给当前的客户端channel分配eventloop，并且将该channel绑定到selector
4.  通过传播active方法，将读事件注册到selector

Q1：boss线程轮询出accept事件、通过jdk底层创建这条新连接

Q2：boss线程调用next从work线程组里拿到eventloop，将这条连接注册到eventloop



