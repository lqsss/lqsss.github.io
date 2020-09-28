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
          //1.1 创建channel
            channel = channelFactory.newChannel();
         	//2.
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                channel.unsafe().closeForcibly();
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }

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



1.1 创建channel

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

2.1 调用jdk底层方法，创建ServerSocketChannel

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
      //2.1.1 
        return provider.openServerSocketChannel();
      	//调用jdk底层方法，创建ServerSocketChannel
    } catch (IOException var2) {
        throw new ChannelException("Failed to open a server socket.", var2);
    }
}
```



2. NioServerSocketChannelConfig：tcp参数配置类

```java
public NioServerSocketChannel(java.nio.channels.ServerSocketChannel channel) {
    super((Channel)null, channel, 16);
    this.config = new NioServerSocketChannel.NioServerSocketChannelConfig(this, this.javaChannel().socket());
}
```



3. 设置非阻塞模式

在父类构造函数中`ch.configureBlocking(false);`设置为非阻塞模式。

4. AbstractChannel

```java
protected AbstractChannel(Channel parent) {
  this.parent = parent;
  this.id = this.newId(); //唯一标志
  this.unsafe = this.newUnsafe();
//unsafe主要用于实现底层的 rergister,read,write等操作
  this.pipeline = this.newChannelPipeline(); 
}
```



### 注册selector

    > 为了实现NIO中把ServerSocketChannel注册到 Selector中去，这样就是可以实现client请求的监听

继续探讨`bind()`中的`initAndRegister()`方法里`ChannelFuture regFuture = this.config().group().register(channel);`



config：ServerBootstrapConfig

group：EventLoopGroup

**这里调用的register()，会调用父类的next()， chooser策略从 EventExecutor[]数组中选择一个 SingleThreadEventLoop，使用最终会调用unsafe接口的register方法，AbstractChannel里的内部类AbstractUnsafe实现了该接口。**

```java
protected abstract class AbstractUnsafe implements Unsafe {
        private volatile ChannelOutboundBuffer outboundBuffer = new ChannelOutboundBuffer(AbstractChannel.this);
        private Handle recvHandle;
        private boolean inFlush0;
        private boolean neverRegistered = true;

        protected AbstractUnsafe() {
        }


        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            } else if (AbstractChannel.this.isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
            } else if (!AbstractChannel.this.isCompatible(eventLoop)) {
                promise.setFailure(new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
            } else {
                AbstractChannel.this.eventLoop = eventLoop;//绑定bossNio线程
                if (eventLoop.inEventLoop()) {
                    this.register0(promise);
                } else {
                    try {
                        eventLoop.execute(new Runnable() {
                            public void run() {
                                AbstractUnsafe.this.register0(promise);
                            }
                        });
                    } catch (Throwable var4) {
                        AbstractChannel.logger.warn("Force-closing a channel whose registration task was not accepted by an event loop: {}", AbstractChannel.this, var4);
                        this.closeForcibly();
                        AbstractChannel.this.closeFuture.setClosed();
                        this.safeSetFailure(promise, var4);
                    }
                }
    
            }
        }
    
        private void register0(ChannelPromise promise) {
            try {
                if (!promise.setUncancellable() || !this.ensureOpen(promise)) {
                    return;
                }
    
                boolean firstRegistration = this.neverRegistered;
                AbstractChannel.this.doRegister();//jdk底层
                this.neverRegistered = false;
                AbstractChannel.this.registered = true;
                AbstractChannel.this.pipeline.invokeHandlerAddedIfNeeded();
              //触发事件:对应用户的handlerAdded
                this.safeSetSuccess(promise);
                AbstractChannel.this.pipeline.fireChannelRegistered();
              //传播事件：channelRegistered
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



`doRegister()`由子类`AbstractNioChannel`实现

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

*   端口绑定

`doBind()`-&gt;`doBind0()`-&gt;AbstractChannel内部类`unsafe的bind`

isActive():`ch.isOpen() &amp;&amp; ch.isConnected()`

1.  最终调用jdk底层的channel

2.  底层channel在绑定成功后，触发active事件

    `pipeline.fireChannelActive();`

```java
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    ctx.fireChannelActive();
    readIfIsAutoRead();
  //tail.read();传播一个read事件
  //为selectionKey新增一个之前传入readInterestOp参数的感兴趣事件(SelectionKey.OP_ACCEPT)
}
```



3. 触发read事件，经过层层调用，最终会向selector注册一个accept的感兴趣事件

## 总结

1.  反射创建channel，包装成Netty自己的channel
2.  初始化的过程，添加一些组件，例如pipeline，为此添加一个handler
3.  将jdk的channel绑定到事件轮询器selector，并且将netty抽象的一个channel作为一个attchment绑定jdk底层的channel
4.  调用底层的bind，监听端口，向select注册一个OP_ACCEPT事件，这样Netty就可以接收新的连接了
