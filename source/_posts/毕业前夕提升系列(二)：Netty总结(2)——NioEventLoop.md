---
title: 毕业前夕提升系列(二)：Netty总结(1)——服务端的启动
tag: 
- summary
categories: Netty
---

## 问题

之前猫眼电影面试官，曾经问过我服务端起多少个NIO线程(boss)？

*   默认情况下，Netty服务端起多少线程？何时启动
*   Netty是如何解决jdk空轮询bug的
*   Netty如何保证异步串行无锁化



<!--more-->

## 线程池的创建

用户调用构造函数创建NIO线程组：

```java
private EventLoopGroup bossGroup = new NioEventLoopGroup();
private EventLoopGroup workerGroup = new NioEventLoopGroup();
```



1. 构造函数

``NioEventLoopGroup``

```java
public NioEventLoopGroup() {
    this(0);//实际传的nThreads为0
}

public NioEventLoopGroup(int nThreads) {
    this(nThreads, (Executor) null);
}

public NioEventLoopGroup(int nThreads, ThreadFactory threadFactory) {
    this(nThreads, threadFactory, SelectorProvider.provider());
}

public NioEventLoopGroup(int nThreads, Executor executor) {
    this(nThreads, executor, SelectorProvider.provider());
}


public NioEventLoopGroup(
        int nThreads, ThreadFactory threadFactory, final SelectorProvider selectorProvider) {
    this(nThreads, threadFactory, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}

public NioEventLoopGroup(int nThreads, ThreadFactory threadFactory,
    final SelectorProvider selectorProvider, final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, threadFactory, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}
```




调用父类构造函数：

```java
protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    	//之前传入的线程数默认为0，所以这里默认设置为cpu*2
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
}
```



``MultithreadEventExecutorGroup.java``

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            //1.1 线程创建器
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        children = new EventExecutor[nThreads];//1.2 构造NioEventLoop

        for (int i = 0; i < nThreads; i ++) {   
            boolean success = false;
            try {
                children[i] = newChild(executor, args);//1.2 构造NioEventLoop
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        chooser = chooserFactory.newChooser(children);
  //1.3 创建线程选择器，选择器的作用就是给每个新的连接绑定一个新的nio线程 

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```



### 1.1 Executor的创建

1.1.1 线程工厂，每次执行任务都会创建一个线程实体(FastLocalThread)

``ThreadPerTaskExecutor#newDefaultThreadFactory``

```java
  protected ThreadFactory newDefaultThreadFactory() {
        return new DefaultThreadFactory(getClass());
    }

public DefaultThreadFactory(Class<?> poolType, boolean daemon, int priority) {
        //poolType：NioEventLoopGroup.class
        this(toPoolName(poolType), daemon, priority); 
    }

	//构造线程池的名字：nioEventLoopGroup-1-
    public static String toPoolName(Class<?> poolType) {
        if (poolType == null) {
            throw new NullPointerException("poolType");
        }

        String poolName = StringUtil.simpleClassName(poolType);
        switch (poolName.length()) {
            case 0:
                return "unknown";
            case 1:
                return poolName.toLowerCase(Locale.US);
            default:
                if (Character.isUpperCase(poolName.charAt(0)) && Character.isLowerCase(poolName.charAt(1))) {
                    return Character.toLowerCase(poolName.charAt(0)) + poolName.substring(1);
                } else {
                    return poolName;
                }
        }
    }

    public DefaultThreadFactory(String poolName, boolean daemon, int priority, ThreadGroup threadGroup) {
        if (poolName == null) {
            throw new NullPointerException("poolName");
        }
        if (priority < Thread.MIN_PRIORITY || priority > Thread.MAX_PRIORITY) {
            throw new IllegalArgumentException(
                    "priority: " + priority + " (expected: Thread.MIN_PRIORITY <= priority <= Thread.MAX_PRIORITY)");
        }

        prefix = poolName + '-' + poolId.incrementAndGet() + '-';
        this.daemon = daemon;
        this.priority = priority;
        this.threadGroup = threadGroup;
    }

    public DefaultThreadFactory(String poolName, boolean daemon, int priority) {
        this(poolName, daemon, priority, System.getSecurityManager() == null ?
                Thread.currentThread().getThreadGroup() : System.getSecurityManager().getThreadGroup());
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = newThread(FastThreadLocalRunnable.wrap(r), prefix + nextId.incrementAndGet());
        try {
            if (t.isDaemon() != daemon) {
                t.setDaemon(daemon);
            }

            if (t.getPriority() != priority) {
                t.setPriority(priority);
            }
        } catch (Exception ignored) {
            // Doesn't matter even if failed to set.
        }
        return t;
    }

	//包装了一个FastThreadLocalThread extend Thread
    protected Thread newThread(Runnable r, String name) {
        return new FastThreadLocalThread(threadGroup, r, name);
    }
```




### 1.2 newChild()：创建NioEventLoop
> NioEventLoop -&gt; SingleThreadEventLoop -&gt; SingleThreadEventExecutor -&gt; AbstractScheduledEventExecutor

`SingleThreadEventExecutor`是netty中对本地线程的抽象，内部有一个Thread thread 属性，存储了一个java本地线程，所以NioEventLoop可以视为一个处理IO事件的线程模型。

``NioEventLoopGroup#newChild()``

```java
@Override
protected EventLoop newChild(Executor  executor, Object... args) throws Exception {
    //1.2.1 创建NioEventLoop对象
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
        ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
}
```



1.2.1 创建NioEventLoop对象

``NioEventLoop()``

```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
             SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler) {
    //1.2.1.1 父类构造函数，共享线程执行器Executor
    super(parent, executor, false, DEFAULT_MAX_PENDING_TASKS, rejectedExecutionHandler);
    if (selectorProvider == null) {
        throw new NullPointerException("selectorProvider");
    }
    if (strategy == null) {
        throw new NullPointerException("selectStrategy");
    }
    provider = selectorProvider;
    
    //1.2.1.2 创建一个selector
    final SelectorTuple selectorTuple = openSelector();
    selector = selectorTuple.selector;
    unwrappedSelector = selectorTuple.unwrappedSelector;
    selectStrategy = strategy;
}
```



1.2.1.1 保存线程执行器、创建一个任务队列：super里`taskQueue = newTaskQueue(this.maxPendingTasks);`

1.2.1.2 创建一个selector

> 一个NioEventLoop属于某一个NioEventLoopGroup， 且处于同一个NioEventLoopGroup下的所有NioEventLoop 公用Executor、SelectorProvider、SelectStrategyFactory和RejectedExecutionHandler。



### 1.3 线程选择器

作用：当新连接到来时，NioEventLoop[]里选择一个nioEventLoop进行绑定

初始化入口：``chooser = chooserFactory.newChooser(children);``

``DefaultEventExecutorChooserFactory#newChooser()``

```java
@Override
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {//根据NioEventLoop的数量是否2的幂
        return new PowerOfTwoEventExecutorChooser(executors);
      //index ++ &(length-1)
    } else {
        return new GenericEventExecutorChooser(executors);
      //abs(index ++ % length)
    }
}
```



Netty中有两种选择器的实现：

*   GenericEventExecutorChooser

```java
private static final class GenericEventExecutorChooser implements EventExecutorChooser {
    private final AtomicInteger idx = new AtomicInteger();
    private final EventExecutor[] executors;

    GenericEventExecutorChooser(EventExecutor[] executors) {
        this.executors = executors;
    }

    @Override
    public EventExecutor next() {
        return executors[Math.abs(idx.getAndIncrement() % executors.length)];
    }
}
```




*   PowerOfTwoEventExecutorChooser
```java
private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
    private final AtomicInteger idx = new AtomicInteger();
    private final EventExecutor[] executors;

    PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
        this.executors = executors;
    }

    @Override
    public EventExecutor next() {
        return executors[idx.getAndIncrement() & executors.length - 1];
    }
}
```



**q：在nio线程数为2的幂时，``idx.getAndIncrement() & (executors.length - 1)``为什么可以达到递增循环的效果？**

 **a：因为2的幂-1转换为2进制的结果就是每个位置都是1，与下标做&amp;时，数值不会变(0&amp;1 = 0、1&amp;1 = 1 )；当index位置上的数都是1时，再递增就会进位，所谓位置变为0，首部新增一位1，再做&amp;，则轮询到了executors数组的首元素**

新连接绑定到NioEventLoop上：

```java
@Override
public EventExecutor next() {
    return chooser.next();
}
```

> *   NIOEventLoopGroup的线程池实现其实就是一个NIOEventLoop数组，一个NIOEventLoop可以理解成就是一个线程。
> *   所有的NIOEventLoop线程是使用相同的 executor、SelectorProvider、SelectStrategyFactory、RejectedExecutionHandler以及是属于某一个NIOEventLoopGroup的。 这一点从 newChild(executor, args); 方法就可以看出：newChild()的实现是在NIOEventLoopGroup中实现的。

## NioEventLoop启动

触发的时机：

- 服务端启动绑定端口
- 新连接接入通过chooser绑定一个NioEventLoop



### 服务端启动绑定端口

1. 触发时机
   1.1 `channel.eventLoop().execute`：创建的NioServerSockcetChannel与Boss线程组中的eventloop绑定，发生在unsafe的`register()`注册。

   1.2 register0是具体注册的逻辑，但是需要由`eventLoop.inEventLoop()`判断当前线程否为nioEventLoop里的线程

   ​		1.2.1 如果是直接执行register0逻辑；如果不是，则包装成一个注册的task交付给线程池新生成一个nio线程去执行`eventLoop.execute(new Runnable(){//...})`调用的是`SingleThreadEventExecutor`的`execute`方法。

   ​		1.2.2 unsafe的`register()`中才刚进行NioEventLoop与channel的绑定，里面的thread还是null，所以会进行接下来的启动线程

   

2. 创建并启动线程

``SingleThreadEventExecutor#execute()``

```java
@Override
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }
    
    //2.1 判断当前线程
    boolean inEventLoop = inEventLoop();
    addTask(task);					//加入taskQueue
    if (!inEventLoop) {
        //启动线程
        startThread();
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }
      
    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
```



startThread()和doStartThread()方法，创建NioEventLoop底层的thread属性。

```java
private void startThread() {
    if (state == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
          //CAS操作修改thread状态为ST_STARTED
            try {
                doStartThread();
              //真正启动线程的函数
            } catch (Throwable cause) {
                STATE_UPDATER.set(this, ST_NOT_STARTED);
                PlatformDependent.throwException(cause);
            }
        }
    }
}
      
private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() { //ThreadPerTaskExecutor
        @Override
        public void run() {
            thread = Thread.currentThread();
          	//将当前线程与thread属性绑定
            if (interrupted) {
                thread.interrupt();
            }
      
            boolean success = false;
            updateLastExecutionTime();
            try {
                SingleThreadEventExecutor.this.run();
              //core：NioEventLoop.run()
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                for (;;) {
                    int oldState = state;
                    if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                            SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                        break;
                    }
                }
      
                // Check if confirmShutdown() was called at the end of the loop.
                if (success && gracefulShutdownStartTime == 0) {
                    if (logger.isErrorEnabled()) {
                        logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                "be called before run() implementation terminates.");
                    }
                }
      
                try {
                    // Run all remaining tasks and shutdown hooks.
                    for (;;) {
                        if (confirmShutdown()) {
                            break;
                        }
                    }
                } finally {
                    try {
                        cleanup();
                    } finally {
                        STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                        threadLock.release();
                        if (!taskQueue.isEmpty()) {
                            if (logger.isWarnEnabled()) {
                                logger.warn("An event executor terminated with " +
                                        "non-empty task queue (" + taskQueue.size() + ')');
                            }
                        }
      
                        terminationFuture.setSuccess(null);
                    }
                }
            }
        }
    });
}
```



**总结：**

1.  **NioEventLoop的启动，发生在第一次调用NioEventLoop调用execute执行任务时，会通过`SingleThreadEventExecutor`里的`ThreadPerTaskExecutor`创建线程。**
2.  **将NioEventLoop里的底层thread属性与创建的本地线程绑定。**
3.  **NioEventLoop.run()启动。**

## NioEventLoop执行
```java
@Override
protected void run() {
  for (;;) {
    try {
      switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
        case SelectStrategy.CONTINUE:
          continue;

        case SelectStrategy.BUSY_WAIT:
          // fall-through to SELECT since the busy-wait is not supported with NIO

        case SelectStrategy.SELECT:
          select(wakenUp.getAndSet(false));

          if (wakenUp.get()) {
            selector.wakeup();
          }
          // fall through
        default:
      }

      cancelledKeys = 0;
      needsToSelectAgain = false;
      final int ioRatio = this.ioRatio;
      if (ioRatio == 100) {
        try {
          processSelectedKeys();
        } finally {
          // Ensure we always run tasks.
          runAllTasks();
        }
      } else {
        final long ioStartTime = System.nanoTime();
        try {
          processSelectedKeys();
          //查询就绪的 IO 事件, 然后处理它;
        } finally {
          // Ensure we always run tasks.
          final long ioTime = System.nanoTime() - ioStartTime;
          runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
          //运行 taskQueue 中的任务.
        }
      }
    } catch (Throwable t) {
      handleLoopException(t);
    }
    // Always handle shutdown even if the loop processing threw an exception.
    try {
      if (isShuttingDown()) {
        closeAll();
        if (confirmShutdown()) {
          return;
        }
      }
    } catch (Throwable t) {
      handleLoopException(t);
    }
  }
}
```


**总结：**

**NioEventLoop事件循环执行机制是一个死循环：**

1.  **select检查是否有IO事件的到来**
2.  **处理IO事件**
3.  **处理异步任务队列里的任务(taskQueue，其它线程加入该队列的任务)**

### 检测I/O事件



```java
private void select(boolean oldWakenUp) throws IOException {
  Selector selector = this.selector;
  try {
    int selectCnt = 0;
    long currentTimeNanos = System.nanoTime();
    long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

    for (;;) {
      long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
      //定时任务即将执行的剩余时间(如果没有定时任务，则默认为1s)，减去当前时间，加上5s的缓冲时间
      if (timeoutMillis <= 0) {
        if (selectCnt == 0) {
          selector.selectNow();
          //如果定时任务超时，调用非阻塞select
          selectCnt = 1;
        }
        break;
      }

      if (hasTasks() && wakenUp.compareAndSet(false, true)) {
        selector.selectNow();
        //如果任务队列里存在任务，则调用阻塞select
        selectCnt = 1;
        break;
      }

      int selectedKeys = selector.select(timeoutMillis);
      selectCnt ++; //轮询次数

      if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
        // select是否需要唤醒 /外部线程唤醒 /任务队列或者定时任务有任务 
        //满足这些条件就中断
        break;
      }
      if (Thread.interrupted()) {

        if (logger.isDebugEnabled()) {
          logger.debug("Selector.select() returned prematurely because " +
                       "Thread.currentThread().interrupt() was called. Use " +
                       "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
        }
        selectCnt = 1;
        break;
      }

      long time = System.nanoTime();
      if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
        //空轮询次数
        selectCnt = 1;
      } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                 selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
        logger.warn(
          "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
          selectCnt, selector);

        rebuildSelector();
        selector = this.selector;

        selector.selectNow(); 
        //更新完selector，需要立即调用一次非阻塞select。
        selectCnt = 1;
        break;
      }

      currentTimeNanos = time;
    }

    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
      if (logger.isDebugEnabled()) {
        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                     selectCnt - 1, selector);
      }
    }
  } catch (CancelledKeyException e) {
    if (logger.isDebugEnabled()) {
      logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                   selector, e);
    }
  }
}
```




总结：

1.  计算一个select的一个阻塞超时时间：定时任务即将执行的剩余时间(如果没有定时任务，则默认为1s)，减去当前时间，加上5s的缓冲时间
2.  **如果定时任务已经超时(阻塞超时时间小于0)**，而且一次都没有执行过select，则立即selecotNow返回；break
3.  **如果阻塞超时时间大于0**，而且任务列队里有任务，且CAS唤醒selector成功的话，立即执行selectNow，break
4.  **上述2、3都不满足的话**，则执行一次阻塞的select，select循环次数+1
5.  如果有就绪的io事件，或外部用户唤醒，或有待执行普通任务，或有待执行调度任务，break
6.  解决jdk空轮询的bug

        1.  记录一下当前的时间，**如果当前时间减去for循环开始的时间是大于selector阻塞超时时间**，表明正常完成一次select阻塞操作，将selectCnt赋值为1
    
    2.  **如果小于**，select轮询次数大于等于阈值(512)，则会重建一个新的selector对象，避免jdk的bug导致cpu打满。
```java
private void rebuildSelector0() {
        final Selector oldSelector = selector;
        final SelectorTuple newSelectorTuple;

        if (oldSelector == null) {
            return;
        }

        try {
            newSelectorTuple = openSelector();//1
        } catch (Exception e) {
            logger.warn("Failed to create a new Selector.", e);
            return;
        }

        // Register all channels to the new Selector.
        int nChannels = 0;
        for (SelectionKey key: oldSelector.keys()) {
            Object a = key.attachment(); //NioSocketChannel
            try {
              //jdk底层channel可能会多次向selector注册,保存到keys[]里
                if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
                    continue;
                }

                int interestOps = key.interestOps();
              	// 获取兴趣集
                key.cancel();
              //取消
                SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
               // 将老的兴趣集重新注册到前面新创建的selector上
                if (a instanceof AbstractNioChannel) {
                    // Update SelectionKey
                    ((AbstractNioChannel) a).selectionKey = newKey;
                }
                nChannels ++;
            } catch (Exception e) {
                logger.warn("Failed to re-register a Channel to the new Selector.", e);
                if (a instanceof AbstractNioChannel) {
                    AbstractNioChannel ch = (AbstractNioChannel) a;
                    ch.unsafe().close(ch.unsafe().voidPromise());
                } else {
                    @SuppressWarnings("unchecked")
                    NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                    invokeChannelUnregistered(task, key, e);
                }
            }
        }

        selector = newSelectorTuple.selector;
        unwrappedSelector = newSelectorTuple.unwrappedSelector;

        try {
            // time to close the old selector as everything else is registered to the new one
            oldSelector.close();
			      //  关闭老的seleclor
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close the old Selector.", t);
            }
        }

        if (logger.isInfoEnabled()) {
            logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
        }
    }
```


**rebuildSelector：新建selector，将注册在老selector的channel以及它们的感兴趣事件重新注册到新的selector，得到新的selectionKey，更新一下NioChannel对应的key。最终将老的selector关闭。**

### 处理selectionKey

#### select优化

发生在NioEventLoop被创建的时候。



```java
private SelectorTuple openSelector() {
  final Selector unwrappedSelector;
  try {
    unwrappedSelector = provider.openSelector();
  } catch (IOException e) {
    throw new ChannelException("failed to open a new selector", e);
  }

  if (DISABLE_KEYSET_OPTIMIZATION) {       
    //默认是false，需要优化
    return new SelectorTuple(unwrappedSelector);
  }

  //...

  final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;
  final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();
  // SelectedSelectionKeySet是我们优化的类型

  Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
    @Override
    public Object run() {
      try {
        Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
        Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");
        // 反射获取selectedKeys、publicSelectedKeys的field

        if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {

          long selectedKeysFieldOffset = PlatformDependent.objectFieldOffset(selectedKeysField);
          long publicSelectedKeysFieldOffset =
            PlatformDependent.objectFieldOffset(publicSelectedKeysField);

          if (selectedKeysFieldOffset != -1 && publicSelectedKeysFieldOffset != -1) {
            PlatformDependent.putObject(
              unwrappedSelector, selectedKeysFieldOffset, selectedKeySet);
            PlatformDependent.putObject(
              unwrappedSelector, publicSelectedKeysFieldOffset, selectedKeySet);
            return null;
          }
          // We could not retrieve the offset, lets try reflection as last-resort.
        }

        Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
        if (cause != null) {
          return cause;
        }
        cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
        if (cause != null) {
          return cause;
        }

        selectedKeysField.set(unwrappedSelector, selectedKeySet);
        publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
        //将这两个field设置为我们优化过的selectedKeySet结构，之后，有I/O事件到来的有关selectionKey就会存储在这个结构里 
        return null;
      } catch (NoSuchFieldException e) {
        return e;
      } catch (IllegalAccessException e) {
        return e;
      }
    }
  });

  if (maybeException instanceof Exception) {
    selectedKeys = null;
    Exception e = (Exception) maybeException;
    logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, e);
    return new SelectorTuple(unwrappedSelector);
  }
  selectedKeys = selectedKeySet
    //将NioEventLoop里的selectedKeys设置为新的优化结构
    logger.trace("instrumented a special java.util.Set into: {}", unwrappedSelector);
  return new SelectorTuple(unwrappedSelector,
                           new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet));
}
```




总结：

1.  通过provider生成一个原生的selecor，新创建一个优化的数据结构`SelectedSelectionKeySet`
2.  通过反射(selectorImplClass)获取`selectedKeys`、`publicSelectedKeys`的field
3.  将优化的数据结构塞进原生selector(`selectorImpl`)，替换掉步骤2得到的两个field
4.  将NioEventLoop里的selectedKeys设置为新的优化结构

**SelectorImpl是原生selector的实现**

```java
public abstract class SelectorImpl extends AbstractSelector {
    protected Set<SelectionKey> selectedKeys = new HashSet();
  // selectedKeys：可以准备被处理的key集合
    protected HashSet<SelectionKey> keys = new HashSet();
    private Set<SelectionKey> publicKeys;
    private Set<SelectionKey> publicSelectedKeys;
  // publicSelectedKeys：实际上就是selectedKeys，暴露出去而已，被设置为不能修改

    protected SelectorImpl(SelectorProvider var1) {
        super(var1);
        if (Util.atBugLevel("1.4")) {
            this.publicKeys = this.keys;
            this.publicSelectedKeys = this.selectedKeys;
        } else {
            this.publicKeys = Collections.unmodifiableSet(this.keys);
            this.publicSelectedKeys = Util.ungrowableSet(this.selectedKeys);
        }
    }
}
```


那么替代`selectedKeys`**hashSet**这种结构，优化的结构是什么样的呢？

**SelectedSelectionKeySet*



```java
final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {

    SelectionKey[] keys;
    int size;

    SelectedSelectionKeySet() {
        keys = new SelectionKey[1024];
    }

    @Override
    public boolean add(SelectionKey o) {
        if (o == null) {
            return false;
        }

        keys[size++] = o;
        if (size == keys.length) {
            increaseCapacity();
        }

        return true;
    }
}
```




底层其实就是维护了一个keys的数组，add操作是O(1)的复杂度，而HashSet这种需要更长的add时间。

#### 具体处理

`processSelectedKeysOptimized()`具体处理selectionKeys的函数。

```java
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
      final SelectionKey k = selectedKeys.keys[i];

      selectedKeys.keys[i] = null;
      //方便gc

      final Object a = k.attachment();

      if (a instanceof AbstractNioChannel) {
        processSelectedKey(k, (AbstractNioChannel) a);
        //具体处理逻辑
      } else {
        @SuppressWarnings("unchecked")
        NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
        processSelectedKey(k, task);
      }

      if (needsToSelectAgain) {
        // null out entries in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys.reset(i + 1);

        selectAgain();
        i = -1;
      }
    }
}

private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    // 获取Netty Channel中的 NioUnsafe 对象，用于后面的IO操作

    if (!k.isValid()) {
      final EventLoop eventLoop;
      try {
        eventLoop = ch.eventLoop();
      } catch (Throwable ignored) {

        return;
      }

      if (eventLoop != this || eventLoop == null) {
        return;
      }
      // close the channel if the key is not valid anymore
      unsafe.close(unsafe.voidPromise());
      return;
    }

    try {
      int readyOps = k.readyOps();

      if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
        // 先处理连接事件
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
        unsafe.read();
      }
    } catch (CancelledKeyException ignored) {
      unsafe.close(unsafe.voidPromise());
    }
}
```


### 任务处理

发生在NioEventLoop的事件循环里`run()`的`runAllTask()`

```java
if (ioRatio == 100) {
    try {
        processSelectedKeys();
    } finally {
        // Ensure we always run tasks.
        runAllTasks();
    }
} else {
    final long ioStartTime = System.nanoTime();
    try {
        processSelectedKeys();
    } finally {
        // Ensure we always run tasks.
        final long ioTime = System.nanoTime() - ioStartTime;
        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    }
}
```




*   `ioRatio`表示I/O任务和非I/O任务的时间比，默认是50
*   `ioRatio`为100时，就先完成select的I/O任务，然后完成非I/O任务
*   `ioRatio`为其它时，记录I/O任务完成的时间，完成非I/O任务的时间不得超过**ioTime * (100 - ioRatio) / ioRatio**



```java
protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue(); //1. 
    Runnable task = pollTask();
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }

    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        safeExecute(task);  //2. 

        runTasks ++;

        // Check timeout every 64 tasks because nanoTime() is relatively expensive.
        // XXX: Hard-coded value - will make it configurable if it is really a problem.
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```




1.  任务的聚合
2.  任务的执行

#### 任务的聚合

将定时任务队列里过期的任务聚合到taskQueue

```java
private boolean fetchFromScheduledTaskQueue() {
    long nanoTime = AbstractScheduledEventExecutor.nanoTime(); 
    // 算出距离定时任务队列开始时的时间

    Runnable scheduledTask  = pollScheduledTask(nanoTime);
    // 取出已经过期的延时任务

    while (scheduledTask != null) {
      if (!taskQueue.offer(scheduledTask)) {
        // 如果向taskQueue添加失败
        // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.             
        scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask);
        // 因为之前把定时队列取出的时候会remove，所以需要重新添加回去
        return false;
      }
      scheduledTask  = pollScheduledTask(nanoTime);
      // 继续取出定时队列的一个任务，scheduledTaskQueue是一个优先级队列
    }
    return true;
}
```



定时任务`ScheduledFutureTask`

```java
  static long deadlineNanos(long delay) {
      long deadlineNanos = nanoTime() + delay;
      // Guard against overflow
      return deadlineNanos < 0 ? Long.MAX_VALUE : deadlineNanos;
  }

//延时时间短的，排在前面
//如果时间相等，则比较Id大小，小的在前
@Override
  public int compareTo(Delayed o) {
      if (this == o) {
          return 0;
      }

      ScheduledFutureTask<?> that = (ScheduledFutureTask<?>) o;
      long d = deadlineNanos() - that.deadlineNanos();
      if (d < 0) {
          return -1;
      } else if (d > 0) {
          return 1;
      } else if (id < that.id) {
          return -1;
      } else if (id == that.id) {
          throw new Error();
      } else {
          return 1;
      }
  }
```




#### 任务的执行

safeExecute

```java
protected static void safeExecute(Runnable task) {
    try {
      task.run();
    } catch (Throwable t) {
      logger.warn("A task raised an exception. Task: {}", task, t);
    }
}
```


可以看到就是简单地执行run方法。

```java
for (;;) {
    safeExecute(task);  //2. 

    runTasks ++;

    // Check timeout every 64 tasks because nanoTime() is relatively expensive.
    // XXX: Hard-coded value - will make it configurable if it is really a problem.
    if ((runTasks & 0x3F) == 0) {
        lastExecutionTime = ScheduledFutureTask.nanoTime();
        if (lastExecutionTime >= deadline) {
            break;
        }
    }

    task = pollTask();
    if (task == null) {
        lastExecutionTime = ScheduledFutureTask.nanoTime();
        break;
    }
}
```


1.  执行任务次数+1
2.  判断执行任务次数是否达到64

    1. 如果到达了64个，就判断是否到达非I/O任务时间，是的话就退出，并记录下最后执行任务的时间，break
    
    2.  未达到64，且未超时，则取任务进行下一个循环，直到taskQueue为空

**检查（nanoTime的执行）也是相对耗时的操作，这里用了硬编码的方式**

## 参考

[netty源码分析–EventLoopGroup与EventLoop 分析netty的线程模型](https://blog.csdn.net/u010853261/article/details/620437)

[JDK Epoll空轮询bug](https://www.jianshu.com/p/3ec120ca46b2)