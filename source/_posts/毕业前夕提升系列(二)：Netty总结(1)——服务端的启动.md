---
title: 毕业前夕提升系列(二)：Netty总结(1)——服务端的启动
tag: 
- summary
categories: Netty
---


本文主要描述Netty的启动过程。

<!-- more -->

## 服务端启动样例
```java
try {
  
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(channelInitializer)
            .option(ChannelOption.SO_BACKLOG, 128)
            .childOption(ChannelOption.SO_KEEPALIVE, true);

    // Bind and start to accept incoming connections.
    ChannelFuture f = b.bind(port).sync();
    // Wait until the server socket is closed.
    // In this example, this does not happen, but you can do that to gracefully
    // shut down your server.
    f.channel().closeFuture().sync();

} finally {
    workerGroup.shutdownGracefully();
    bossGroup.shutdownGracefully();
}
```



**Q:**

**1. 服务端的socket在哪里初始化？**

**2. 在哪里accept连接？**



服务端启动伴随着4个过程：

1. 创建服务端Channel
2. 初始化服务端Channel
3. 注册selector
4. 端口绑定


## 服务端启动过程
### 创建服务端Channel

从`bind()`的调用一步一步地深入，其中方法里

1. 创建服务端Channel对象以及初始化

``AbstractBootstrap#initAndRegister()``

```java
   final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
          //1 创建channel
            channel = channelFactory.newChannel();
         	//2 初始化
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                channel.unsafe().closeForcibly();
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }
		//3 将channel注册到selector
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        return regFuture;
    }
```



1 创建channel

``ReflectiveChannelFactory#newChannel()``

```java
public T newChannel() {
    try {
        return (Channel)this.clazz.getConstructor().newInstance();
    } catch (Throwable var2) {
        throw new ChannelException("Unable to create Channel from class " + this.clazz, var2);
    }
}
```




上述代码在`ReflectiveChannelFactory`类里，它是``ChannelFactory``的一种实现方式，**是将该对象的clazz字段反射实例化生成channel，那么这个clazz又是从何而来呢？**这就要追溯到`.channel(NioServerSocketChannel.class)`这行代码里:

 

``AbstractBootstrap#channel()``

```java
public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        } else {
            return this.channelFactory((io.netty.channel.ChannelFactory)(new ReflectiveChannelFactory(channelClass)));
        }
}
```



设置`channelFactory`字段。将class传入到`ReflectiveChannelFactory`构造函数。之前newChannel反射实例化的channel就是`NioServerSocketChannel`



### 反射创建服务端Channel

1.1 调用jdk底层方法，创建ServerSocketChannel

``NioServerSocketChannel#NioServerSocketChannel()``

```java
//反射实例化调用的构造函数 
public NioServerSocketChannel() {
        this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```


```java
private static java.nio.channels.ServerSocketChannel newSocket(SelectorProvider provider) {
    try {
      //2.1.1 调用jdk底层方法，创建ServerSocketChannel
        return provider.openServerSocketChannel();
    } catch (IOException var2) {
        throw new ChannelException("Failed to open a server socket.", var2);
    }
}
```



1.2 NioServerSocketChannelConfig：tcp参数配置类

```java
public NioServerSocketChannel(java.nio.channels.ServerSocketChannel channel) {
    super((Channel)null, channel, 16);
    this.config = new NioServerSocketChannel.NioServerSocketChannelConfig(this, this.javaChannel().socket());
}
```

1.3 设置非阻塞模式

在父类构造函数中`ch.configureBlocking(false);`设置为非阻塞模式。

1.4 AbstractChannel

```java
protected AbstractChannel(Channel parent) {
  this.parent = parent;
  this.id = this.newId(); //唯一标志
  this.unsafe = this.newUnsafe();
//unsafe主要用于实现底层的 rergister,read,write等操作
  this.pipeline = this.newChannelPipeline(); 
}
```



2 初始化channel

- 设置：tcp参数(options)、一些用户自定义的属性(attributeKey)
- 配置服务端handler加入到pipeline
- 创建一个Acceptor连接器  下文会提到



### 注册selector

> 为了实现NIO中把ServerSocketChannel注册到 Selector中去，这样就是可以实现client请求的监听



3. 将channel注册到selector

`` ChannelFuture regFuture = config().group().register(channel);``

``AbstractChannel$AbstractUnsafe#register()``

```java
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    if (eventLoop == null) {
        throw new NullPointerException("eventLoop");
    }
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
            new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }
    //3.1 绑定bossNio线程
    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {
        //3.2 实际注册方法
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```



config：ServerBootstrapConfig

group：EventLoopGroup

**这里调用的register()，会调用父类的next()， chooser策略从 EventExecutor[]数组中选择一个  ，使用最终会调用unsafe接口的register方法，AbstractChannel里的内部类AbstractUnsafe实现了该接口。**

3.2 实际注册方法

``AbstractChannel$AbstractUnsafe#register0()``

```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !this.ensureOpen(promise)) {
            return;
        }

        boolean firstRegistration = this.neverRegistered;
        //3.2.1 jdk底层注册
        AbstractChannel.this.doRegister();
        this.neverRegistered = false;
        AbstractChannel.this.registered = true;
        
        //3.2.2 触发事件：对应用户的handlerAdded
        AbstractChannel.this.pipeline.invokeHandlerAddedIfNeeded();
        
        this.safeSetSuccess(promise);
        //3.2.3 传播事件：channelRegistered
        AbstractChannel.this.pipeline.fireChannelRegistered();
        
        //这个时机还没有进行绑定端口，所以这里还是false
        if (AbstractChannel.this.isActive()) {
            if (firstRegistration) {
                AbstractChannel.this.pipeline.fireChannelActive();
            } else if (AbstractChannel.this.config().isAutoRead()) {
                this.beginRead();
            }
        }
    } catch (Throwable var3) {
        this.closeForcibly();
        AbstractChannel.this.closeFuture.setClosed();
        this.safeSetFailure(promise, var3);
    }

}
```



3.2.1 jdk底层注册

`AbstractNioChannel#doRegister()`

```java
protected void doRegister() throws Exception {
    boolean selected = false;

    while(true) {
        try {
            this.selectionKey = this.javaChannel().register(this.eventLoop().unwrappedSelector(), 0, this);
          //javaChannel()就是ch字段，即之前创建的底层jdk的channel
          //然后调用jdk底层的注册，把channel绑定到selector
            return;
        } catch (CancelledKeyException var3) {
            if (selected) {
                throw var3;
            }
    
            this.eventLoop().selectNow();
            selected = true;
        }
    }
}
```




总结：

1.  对应的NIO线程与当前的channel进行绑定
2.  然后实际注册

    1. 调用jdk底层的注册，把实际的channel绑定到selector
    
    2.  触发对应用户的handlerAdded事件
    3.  传播对应用户的channelRegistered事件

`initAndRegister()`到此已经完毕了，包括了channel的创建、初始化、注册过程



### 端口绑定

`doBind()`-&gt;`doBind0()`-&gt;AbstractChannel内部类`unsafe的bind`



4. ``AbstractChannel$AbstractUnsafe#bind()``

```java
@Override
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();

    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    // See: https://github.com/netty/netty/issues/576
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
        localAddress instanceof InetSocketAddress &&
        !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
        !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
        // Warn a user about the fact that a non-root user can't receive a
        // broadcast packet on *nix if the socket is bound on non-wildcard address.
        logger.warn(
            "A non-root user can't receive a broadcast packet if the socket " +
            "is not bound to a wildcard address; binding to a non-wildcard " +
            "address (" + localAddress + ") anyway as requested.");
    }

    boolean wasActive = isActive();
    try {
        //4.1 调用jdk底层的bind
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }

    //4.2 绑定之后，active变更，触发事件
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }

    safeSetSuccess(promise);
}
```



4.1 调用jdk底层的bind 

``NioServerSocketChannel#doBind()``

```java
@Override
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```



4.2 绑定之后，active变更，触发事件

```java
 if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                //4.2.1 事件传播
                pipeline.fireChannelActive();
            }
        });
 }
```



`DefaultChannelPipeline$HeadContext#channelActive();`

```java
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    //4.2.1.2 传播active事件
    ctx.fireChannelActive();
    //4.2.1.2 传播read事件
    readIfIsAutoRead();
  //tail.read();传播一个read事件
  //为selectionKey新增一个之前传入readInterestOp参数的感兴趣事件(SelectionKey.OP_ACCEPT)
}
```



4.2.1.2  传播read事件，最终事件执行逻辑

``AbstractChannel$AbstractUnsafe#beginRead()``

```java
@Override
public final void beginRead() {
    assertEventLoop();

    if (!isActive()) {
        return;
    }

    try {
        //4.2.1.2.1 selector注册一个accept的感兴趣事件
        doBeginRead();
    } catch (final Exception e) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireExceptionCaught(e);
            }
        });
        close(voidPromise());
    }
}
```



4.2.1.2.1 selector注册一个accept的感兴趣事件

```java
@Override
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    //服务端channel的selectionKey
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    //感兴趣的事件,注册时interestOps为0
    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        //增加一个accept事件，readInterestOp为之前注册时传的SelectionKey.OP_ACCEPT
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```



## 总结

1.  反射创建channel，包装成Netty自己的channel
2.  初始化的过程，添加一些组件，例如pipeline，为此添加一个handler
3.  将jdk的channel绑定到事件轮询器selector，并且将netty抽象的一个channel作为一个attchment绑定jdk底层的channel
4.  调用底层的bind，监听端口，向select注册一个OP_ACCEPT事件，这样Netty就可以接收新的连接了
