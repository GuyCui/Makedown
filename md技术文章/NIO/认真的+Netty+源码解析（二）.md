---
name: netty-2
title: 认真的 Netty 源码解析（二）
date: 2021-02-04 14:12:48
tags: netty-source
categories: 
---
## Channel 的 register 操作

经过前面的铺垫，我们已经具备一定的基础了，我们开始来把前面学到的内容揉在一起。这节，我们会介绍 register 操作，这一步其实是非常关键的，对于我们源码分析非常重要。

### register

我们从 EchoClient 中的 connect() 方法出发，或者 EchoServer 的 bind(port) 方法出发，都会走到 initAndRegister() 这个方法：

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // 1
        channel = channelFactory.newChannel();
        // 2 对于 Bootstrap 和 ServerBootstrap，这里面有些不一样
        init(channel);
    } catch (Throwable t) {
        ...
    }
	// 3 我们这里要说的是这行
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

initAndRegister() 这个方法我们已经接触过两次了，前面介绍了 1️⃣ Channel 的实例化，实例化过程中，会执行 Channel 内部 Unsafe 和 Pipeline 的实例化，以及在上面 2️⃣ init(channel) 方法中，会往 pipeline 中添加 handler（pipeline 此时是 head+channelnitializer+tail）。

> 我们这节终于要揭秘 ChannelInitializer 中的 initChannel 方法了~~~

现在，我们继续往下走，看看 3️⃣ **register** 这一步：

```java
ChannelFuture regFuture = config().group().register(channel);
```

> 我们说了，register 这一步是非常关键的，它发生在 channel 实例化以后，大家回忆一下当前 channel 中的一些情况：
>
> 实例化了 JDK 底层的 Channel，设置了非阻塞，实例化了 Unsafe，实例化了 Pipeline，同时往 pipeline 中添加了 head、tail 以及一个 ChannelInitializer 实例。

上面的 `config().group()` 方法会返回前面实例化的 NioEventLoopGroup 的实例，然后调用其 register(channel) 方法：

// MultithreadEventLoopGroup

```java
@Override
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```

next() 方法很简单，就是选择线程池中的一个线程（还记得 chooserFactory 吗），也就是选择一个 NioEventLoop 实例，这个时候我们就进入到 NioEventLoop 了。

NioEventLoop 的 register(channel) 方法实现在它的父类 **SingleThreadEventLoop** 中：

```java
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```

上面的代码实例化了一个 Promise，将当前 channel 带了进去：

```java
@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    // promise 关联了 channel，channel 持有 Unsafe 实例，register 操作就封装在 Unsafe 中
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

拿到 channel 中关联的 Unsafe 实例，然后调用它的 register 方法：

> 我们说过，Unsafe 专门用来封装底层实现，当然这里也没那么“底层”

// AbstractChannel#**AbstractUnsafe**

```java
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    ...
    // 将这个 eventLoop 实例设置给这个 channel，从此这个 channel 就是有 eventLoop 的了
    // 我觉得这一步其实挺关键的，因为后续该 channel 中的所有异步操作，都要提交给这个 eventLoop 来执行
    AbstractChannel.this.eventLoop = eventLoop;

    // 如果发起 register 动作的线程就是 eventLoop 实例中的线程，那么直接调用 register0(promise)
    // 对于我们来说，它不会进入到这个分支，
    //     之所以有这个分支，是因为我们是可以 unregister，然后再 register 的，后面再仔细看
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            // 否则，提交任务给 eventLoop，eventLoop 中的线程会负责调用 register0(promise)
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            ...
        }
    }
}
```

> 到这里，我们要明白，NioEventLoop 中是还没有实例化 Thread 实例的。

这几步涉及到了好几个类：NioEventLoop、Promise、Channel、Unsafe 等，大家要仔细理清楚它们的关系。

对于我们前面过来的 register 操作，其实提交到 eventLoop 以后，就直接返回 promise 实例了，剩下的register0 是异步操作，它由 NioEventLoop 实例来完成。

我们这边先不继续往里分析 register0(promise) 方法，先把前面欠下的 NioEventLoop 中的线程介绍清楚，然后再回来介绍这个 register0 方法。

> Channel 实例一旦 register 到了 NioEventLoopGroup 实例中的某个 NioEventLoop 实例，那么后续该 Channel 的所有操作，都是由该 NioEventLoop 实例来完成的。
>
> 这个也非常简单，因为 Selector 实例是在 NioEventLoop 实例中的，Channel 实例一旦注册到某个 Selector 实例中，当然也只能在这个实例中处理 NIO 事件。

### NioEventLoop 工作流程

前面，我们在分析线程池的实例化的时候说过，NioEventLoop 中并没有启动 Java 线程。这里我们来仔细分析下在 register 过程中调用的 **eventLoop.execute(runnable)** 这个方法，这个代码在父类 SingleThreadEventExecutor 中：

```java
@Override
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }
	// 判断添加任务的线程是否就是当前 EventLoop 中的线程
    boolean inEventLoop = inEventLoop();
    
    // 添加任务到之前介绍的 taskQueue 中，
    // 	如果 taskQueue 满了(默认大小 16)，根据我们之前说的，默认的策略是抛出异常
    addTask(task);
    
    if (!inEventLoop) {
        // 如果不是 NioEventLoop 内部线程提交的 task，那么判断下线程是否已经启动，没有的话，就启动线程
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

> 原来启动 NioEventLoop 中的线程的方法在这里。
>
> 另外，上节我们说的 register 操作进到了 taskQueue 中，所以它其实是被归类到了非 IO 操作的范畴。

下面是 startThread 的源码，判断线程是否已经启动来决定是否要进行启动操作：

```java
private void startThread() {
    if (state == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            try {
                doStartThread();
            } catch (Throwable cause) {
                STATE_UPDATER.set(this, ST_NOT_STARTED);
                PlatformDependent.throwException(cause);
            }
        }
    }
}
```

我们按照前面的思路，根据线程没有启动的情况，来看看 doStartThread() 方法：

```java
private void doStartThread() {
    assert thread == null;
    // 这里的 executor 大家是不是有点熟悉的感觉，它就是一开始我们实例化 NioEventLoop 的时候传进来的 ThreadPerTaskExecutor 的实例。它是每次来一个任务，创建一个线程的那种 executor。
    // 一旦我们调用它的 execute 方法，它就会创建一个新的线程，所以这里终于会创建 Thread 实例
    executor.execute(new Runnable() {
        @Override
        public void run() {
            // 看这里，将 “executor” 中创建的这个线程设置为 NioEventLoop 的线程！！！
            thread = Thread.currentThread();
            
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                // 执行 SingleThreadEventExecutor 的 run() 方法，它在 NioEventLoop 中实现了
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                // ... 我们直接忽略掉这里的代码
            }
        }
    });
}
```

上面线程启动以后，会执行 NioEventLoop 中的 run() 方法，这是一个**非常重要**的方法，这个方法肯定是没那么容易结束的，必然是像 JDK 线程池的 Worker 那样，不断地循环获取新的任务的。它需要不断地做 select 操作和轮询 taskQueue 这个队列。

我们先来简单地看一下它的源码，这里先不做深入地介绍：

```java
@Override
protected void run() {
    // 代码嵌套在 for 循环中
    for (;;) {
        try {
            // selectStrategy 终于要派上用场了
            // 它有两个值，一个是 CONTINUE 一个是 SELECT
            // 针对这块代码，我们分析一下。
            // 1. 如果 taskQueue 不为空，也就是 hasTasks() 返回 true，
            // 		那么执行一次 selectNow()，该方法不会阻塞
            // 2. 如果 hasTasks() 返回 false，那么执行 SelectStrategy.SELECT 分支，
            //    进行 select(...)，这块是带阻塞的
            // 这个很好理解，就是按照是否有任务在排队来决定是否可以进行阻塞
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    // 如果 !hasTasks()，那么进到这个 select 分支，这里 select 带阻塞的
                    select(wakenUp.getAndSet(false));
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
            }
            
            
            cancelledKeys = 0;
            needsToSelectAgain = false;
            // 默认地，ioRatio 的值是 50
            final int ioRatio = this.ioRatio;
            
            if (ioRatio == 100) {
                // 如果 ioRatio 设置为 100，那么先执行 IO 操作，然后在 finally 块中执行 taskQueue 中的任务
                try {
                    // 1. 执行 IO 操作。因为前面 select 以后，可能有些 channel 是需要处理的。
                    processSelectedKeys();
                } finally {
                    // 2. 执行非 IO 任务，也就是 taskQueue 中的任务
                    runAllTasks();
                }
            } else {
                // 如果 ioRatio 不是 100，那么根据 IO 操作耗时，限制非 IO 操作耗时
                final long ioStartTime = System.nanoTime();
                try {
                    // 执行 IO 操作
                    processSelectedKeys();
                } finally {
                    // 根据 IO 操作消耗的时间，计算执行非 IO 操作（runAllTasks）可以用多少时间.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
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

上面这段代码是 NioEventLoop 的核心，这里介绍两点：

1. 首先，会根据 hasTasks() 的结果来决定是执行 selectNow() 还是 select(oldWakenUp)，这个应该好理解。如果有任务正在等待，那么应该使用无阻塞的 selectNow()，如果没有任务在等待，那么就可以使用带阻塞的 select 操作。
2. ioRatio 控制 IO 操作所占的时间比重：
   - 如果设置为 100%，那么先执行 IO 操作，然后再执行任务队列中的任务。
   - 如果不是 100%，那么先执行 IO 操作，然后执行 taskQueue 中的任务，但是需要控制执行任务的总时间。也就是说，非 IO 操作可以占用的时间，通过 ioRatio 以及这次 IO 操作耗时计算得出。

我们这里先不要去关心 select(oldWakenUp)、processSelectedKeys() 方法和 runAllTasks(…) 方法的细节，只要先理解它们分别做什么事情就可以了。

回过神来，我们前面在 register 的时候提交了 register 任务给 NioEventLoop，这是 NioEventLoop 接收到的第一个任务，所以这里会实例化 Thread 并且启动，然后进入到 NioEventLoop 中的 run 方法。

### 继续 register

我们回到前面的 register0(promise) 方法，我们知道，这个 register 任务进入到了 NioEventLoop 的 taskQueue 中，然后会启动 NioEventLoop 中的线程，该线程会轮询这个 taskQueue，然后执行这个 register 任务。

注意，此时执行该方法的是 eventLoop 中的线程：

// AbstractChannel

```java
private void register0(ChannelPromise promise) {
    try {
		...
        boolean firstRegistration = neverRegistered;
        // *** 进行 JDK 底层的操作：Channel 注册到 Selector 上 ***
        doRegister();
        
        neverRegistered = false;
        registered = true;
        // 到这里，就算是 registered 了
        
        // 这一步也很关键，因为这涉及到了 ChannelInitializer 的 init(channel)
        // 我们之前说过，init 方法会将 ChannelInitializer 内部添加的 handlers 添加到 pipeline 中
        pipeline.invokeHandlerAddedIfNeeded();

        // 设置当前 promise 的状态为 success
        //   因为当前 register 方法是在 eventLoop 中的线程中执行的，需要通知提交 register 操作的线程
        safeSetSuccess(promise);
        
        // 当前的 register 操作已经成功，该事件应该被 pipeline 上
        //   所有关心 register 事件的 handler 感知到，往 pipeline 中扔一个事件
        pipeline.fireChannelRegistered();

        // 这里 active 指的是 channel 已经打开
        if (isActive()) {
            // 如果该 channel 是第一次执行 register，那么 fire ChannelActive 事件
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // 该 channel 之前已经 register 过了，
                // 这里让该 channel 立马去监听通道中的 OP_READ 事件
                beginRead();
            }
        }
    } catch (Throwable t) {
        ...
    }
}
```

我们先说掉上面的 doRegister() 方法，然后再说 pipeline。

```java
@Override
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            // 附 JDK 中 Channel 的 register 方法：
            // public final SelectionKey register(Selector sel, int ops, Object att) {...}
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            ...
        }
    }
}
```

我们可以看到，这里做了 JDK 底层的 register 操作，将 SocketChannel(或 ServerSocketChannel) 注册到 Selector 中，并且可以看到，这里的监听集合设置为了 **0**，也就是什么都不监听。

> 当然，也就意味着，后续一定有某个地方会需要修改这个 selectionKey 的监听集合，不然啥都干不了

我们重点来说说 **pipeline** 操作，我们之前在介绍 NioSocketChannel 的 pipeline 的时候介绍到，我们的 pipeline 现在长这个样子：

![20](https://www.javadoop.com/blogimages/netty-source/20.png)

> 现在，我们将看到这里会把 LoggingHandler 和 EchoClientHandler 添加到 pipeline。

我们继续看代码，register 成功以后，执行了以下操作：

```java
pipeline.invokeHandlerAddedIfNeeded();
```

大家可以跟踪一下，这一步会执行到 pipeline 中 ChannelInitializer 实例的 handlerAdded 方法，在这里会执行它的 init(context) 方法：

```java
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isRegistered()) {
        initChannel(ctx);
    }
}
```

然后我们看下 initChannel(ctx)，这里终于来了我们之前介绍过的 init(channel) 方法：

```java
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
        try {
            // 1. 将把我们自定义的 handlers 添加到 pipeline 中
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            ...
        } finally {
            // 2. 将 ChannelInitializer 实例从 pipeline 中删除
            remove(ctx);
        }
        return true;
    }
    return false;
}
```

我们前面也说过，ChannelInitializer 的 init(channel) 被执行以后，那么其内部添加的 handlers 会进入到 pipeline 中，然后上面的 finally 块中将 ChannelInitializer 的实例从 pipeline 中删除，那么此时 pipeline 就算建立起来了，如下图：

![21](https://www.javadoop.com/blogimages/netty-source/21.png)

> 其实这里还有个问题，如果我们在 ChannelInitializer 中添加的是一个 ChannelInitializer 实例呢？大家可以考虑下这个情况。

pipeline 建立了以后，然后我们继续往下走，会执行到这一句：

```java
pipeline.fireChannelRegistered();
```

我们只要摸清楚了 fireChannelRegistered() 方法，以后碰到其他像 fireChannelActive()、fireXxx() 等就知道怎么回事了，它们都是类似的。我们来看看这句代码会发生什么：

// DefaultChannelPipeline

```java
@Override
public final ChannelPipeline fireChannelRegistered() {
    // 注意这里的传参是 head
    AbstractChannelHandlerContext.invokeChannelRegistered(head);
    return this;
}
```

也就是说，我们往 pipeline 中扔了一个 **channelRegistered** 事件，这里的 register 属于 Inbound 事件，pipeline 接下来要做的就是执行 pipeline 中的 Inbound 类型的 handlers 中的 channelRegistered() 方法。

从上面的代码，我们可以看出，往 pipeline 中扔出 channelRegistered 事件以后，第一个处理的 handler 是 **head**。

接下来，我们还是跟着代码走，此时我们来到了 pipeline 的第一个节点 **head** 的处理中：

// AbstractChannelHandlerContext

```java
// next 此时是 head
static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {

    EventExecutor executor = next.executor();
    // 执行 head 的 invokeChannelRegistered()
    if (executor.inEventLoop()) {
        next.invokeChannelRegistered();
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRegistered();
            }
        });
    }
}
```

也就是说，这里会先执行 head.invokeChannelRegistered() 方法，而且是放到 NioEventLoop 中的 taskQueue 中执行的：

// AbstractChannelHandlerContext

```java
private void invokeChannelRegistered() {
    if (invokeHandler()) {
        try {
            // handler() 方法此时会返回 head
            ((ChannelInboundHandler) handler()).channelRegistered(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRegistered();
    }
}
```

我们去看 head 的 channelRegistered 方法：

// HeadContext

```java
@Override
public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
    // 1. 这一步是 head 对于 channelRegistered 事件的处理。没有我们要关心的
    invokeHandlerAddedIfNeeded();
    // 2. 向后传播 Inbound 事件
    ctx.fireChannelRegistered();
}
```

然后 head 会执行 fireChannelRegister() 方法：

// AbstractChannelHandlerContext

```java
@Override
public ChannelHandlerContext fireChannelRegistered() {
    // 这里很关键
    // findContextInbound() 方法会沿着 pipeline 找到下一个 Inbound 类型的 handler
    invokeChannelRegistered(findContextInbound());
    return this;
}
```

> 注意：pipeline.fireChannelRegistered() 是将 channelRegistered 事件抛到 pipeline 中，pipeline 中的 handlers 准备处理该事件。而 context.fireChannelRegistered() 是一个 handler 处理完了以后，向后传播给下一个 handler。
>
> 它们两个的方法名字是一样的，但是来自于不同的类。

findContextInbound() 将找到下一个 Inbound 类型的 handler，然后又是重复上面的几个方法。

> 我觉得上面这块代码没必要太纠结，总之就是从 head 中开始，依次往下寻找所有 Inbound handler，执行其 channelRegistered(ctx) 操作。

说了这么多，我们的 register 操作算是真正完成了。

下面，我们回到 initAndRegister 这个方法：

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        ...
    }

    // 我们上面说完了这行
    ChannelFuture regFuture = config().group().register(channel);
    
    // 如果在 register 的过程中，发生了错误
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    // 源码中说得很清楚，如果到这里，说明后续可以进行 connect() 或 bind() 了，因为两种情况：
    // 1. 如果 register 动作是在 eventLoop 中发起的，那么到这里的时候，register 一定已经完成
    // 2. 如果 register 任务已经提交到 eventLoop 中，也就是进到了 eventLoop 中的 taskQueue 中，
    //    由于后续的 connect 或 bind 也会进入到同一个 eventLoop 的 queue 中，所以一定是会先 register 成功，才会执行 connect 或 bind
    return regFuture;
}
```

我们要知道，不管是服务端的 NioServerSocketChannel 还是客户端的 NioSocketChannel，在 bind 或 connect 时，都会先进入 initAndRegister 这个方法，所以我们上面说的那些，对于两者都是通用的。

大家要记住，register 操作是非常重要的，要知道这一步大概做了哪些事情，register 操作以后，将进入到 bind 或 connect 操作中。

## connect 过程和 bind 过程分析

上面我们介绍的 register 操作非常关键，它建立起来了很多的东西，它是 Netty 中 NioSocketChannel 和 NioServerSocketChannel 开始工作的起点。

这一节，我们来说说 register 之后的 connect 操作和 bind 操作。这节非常简单。

### connect 过程分析

对于客户端 NioSocketChannel 来说，前面 register 完成以后，就要开始 connect 了，这一步将连接到服务端。

```java
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    // 这里完成了 register 操作
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();

    // 这里我们不去纠结 register 操作是否 isDone()
    if (regFuture.isDone()) {
        if (!regFuture.isSuccess()) {
            return regFuture;
        }
        // 看这里
        return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
    } else {
		....
    }
}
```

这里大家自己一路点进去，我就不浪费篇幅了。最后，我们会来到 AbstractChannel 的 connect 方法：

```java
@Override
public ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
    return pipeline.connect(remoteAddress, promise);
}
```

我们看到，connect 操作是交给 pipeline 来执行的。进入 pipeline 中，我们会发现，connect 这种 Outbound 类型的操作，是从 pipeline 的 tail 开始的：

> 前面我们介绍的 register 操作是 Inbound 的，是从 head 开始的

```java
@Override
public final ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
    return tail.connect(remoteAddress, promise);
}
```

接下来就是 pipeline 的操作了，从 tail 开始，执行 pipeline 上的 Outbound 类型的 handlers 的 connect(...) 方法，那么真正的底层的 connect 的操作发生在哪里呢？还记得我们的 pipeline 的图吗？

![22](https://www.javadoop.com/blogimages/netty-source/22.png)

从 tail 开始往前找 out 类型的 handlers，每经过一个 handler，都执行里面的 connect() 方法，最后会到 head 中，因为 head 也是 Outbound 类型的，我们需要的 connect 操作就在 head 中，它会负责调用 unsafe 中提供的 connect 方法：

```java
// HeadContext
public void connect(
        ChannelHandlerContext ctx,
        SocketAddress remoteAddress, SocketAddress localAddress,
        ChannelPromise promise) throws Exception {
    unsafe.connect(remoteAddress, localAddress, promise);
}
```

接下来，我们来看一看 connect 在 unsafe 类中所谓的底层操作：

```java
// AbstractNioChannel.AbstractNioUnsafe
@Override
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
		......
            
        boolean wasActive = isActive();
        // 大家自己点进去看 doConnect 方法
        // 这一步会做 JDK 底层的 SocketChannel connect，然后设置 interestOps 为 SelectionKey.OP_CONNECT
        // 返回值代表是否已经连接成功
        if (doConnect(remoteAddress, localAddress)) {
            // 处理连接成功的情况
            fulfillConnectPromise(promise, wasActive);
        } else {
            connectPromise = promise;
            requestedRemoteAddress = remoteAddress;

            // 下面这块代码，在处理连接超时的情况，代码很简单
            // 这里用到了 NioEventLoop 的定时任务的功能，这个我们之前一直都没有介绍过，因为我觉得也不太重要
            int connectTimeoutMillis = config().getConnectTimeoutMillis();
            if (connectTimeoutMillis > 0) {
                connectTimeoutFuture = eventLoop().schedule(new Runnable() {
                    @Override
                    public void run() {
                        ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                        ConnectTimeoutException cause =
                                new ConnectTimeoutException("connection timed out: " + remoteAddress);
                        if (connectPromise != null && connectPromise.tryFailure(cause)) {
                            close(voidPromise());
                        }
                    }
                }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }

            promise.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isCancelled()) {
                        if (connectTimeoutFuture != null) {
                            connectTimeoutFuture.cancel(false);
                        }
                        connectPromise = null;
                        close(voidPromise());
                    }
                }
            });
        }
    } catch (Throwable t) {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}
```

如果上面的 doConnect 方法返回 false，那么后续是怎么处理的呢？

在上一节介绍的 register 操作中，channel 已经 register 到了 selector 上，只不过将 interestOps 设置为了 0，也就是什么都不监听。

而在上面的 doConnect 方法中，我们看到它在调用底层的 connect 方法后，会设置 interestOps 为 `SelectionKey.OP_CONNECT`。

剩下的就是 NioEventLoop 的事情了，还记得 NioEventLoop 的 run() 方法吗？也就是说这里的 connect 成功以后，这个 TCP 连接就建立起来了，后续的操作会在 `NioEventLoop.run()` 方法中被 `processSelectedKeys()` 方法处理掉。

### bind 过程分析

说完 connect 过程，我们再来简单看下 bind 过程：

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    // **前面说的 initAndRegister**
    final ChannelFuture regFuture = initAndRegister();
    
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        // register 动作已经完成，那么执行 bind 操作
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
		......
    }
}
```

然后一直往里看，会看到，bind 操作也是要由 pipeline 来完成的：

// AbstractChannel

```java
@Override
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return pipeline.bind(localAddress, promise);
}
```

bind 操作和 connect 一样，都是 Outbound 类型的，所以都是 tail 开始：

```java
@Override
public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return tail.bind(localAddress, promise);
}
```

最后的 bind 操作又到了 head 中，由 head 来调用 unsafe 提供的 bind 方法：

```java
@Override
public void bind(
        ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)
        throws Exception {
    unsafe.bind(localAddress, promise);
}
```

感兴趣的读者自己去看一下 unsafe 中的 bind 方法，非常简单，bind 操作也不是什么异步方法，我们就介绍到这里了。

本节非常简单，就是想和大家介绍下 Netty 中各种操作的套路。