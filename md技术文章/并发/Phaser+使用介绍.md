---
name: phaser-tutorial
title: Phaser 使用介绍
date: 2021-02-04 14:09:47
tags: 
categories: concurrent
---
本文将介绍 `java.util.concurrent.Phaser`，一个常常被大家忽略的并发工具。它和 `CyclicBarrier` 以及 `CountDownLatch` 很像，但是使用上更加的灵活，本文会进行一些对比介绍。

和之前的文章不同，本文不写源码分析了，就只是从各个角度介绍下它是怎么用的。本文比较简单，我觉得对于初学者大概需要 20 分钟左右吧。

> 其实我对这个需要多少时间很没概念，有没有读者愿意记录下所花费的时间，在评论区反馈一下。

## 使用示例

我们来实现一个小需求，启动 10 个线程执行任务，由于启动时间有先后，我们希望等到所有的线程都启动成功以后再开始执行，让每个线程在同一个起跑线上开始执行业务操作。

下面，分别介绍 CountDownLatch、CyclicBarrier 和 Phaser 怎么实现该需求。

1、这种 case 最容易使用的就是 CountDownLatch，代码很简单：

```java
// 1. 设置 count 为 1
CountDownLatch latch = new CountDownLatch(1);

for (int i = 0; i < 10; i++) {
    new Thread(() -> {
        try {
            // 2. 每个线程都等在栅栏这里，等待放开栅栏，不会因为有些线程先启动就先跑路了
            latch.await();
            
            // doWork();
            
        } catch (InterruptedException ignore) {
        }
    }).start();
}

doSomethingELse(); // 确保在下面的代码执行之前，上面每个线程都到了 await() 上。

// 3. 放开栅栏
latch.countDown();

```

> 简单回顾一下 CountDownLatch 的原理：AQS 共享模式的典型使用，构造函数中的 **1** 是设置给 AQS 的 state 的。latch.await() 方法会阻塞，而 latch.countDown() 方法就是用来将 state-- 的，减到 0 以后，唤醒所有的阻塞在 await() 方法上的线程。

2、这种 case 用 CyclicBarrier 来实现更简单：

```java
// 1. 构造函数中指定了 10 个 parties
CyclicBarrier barrier = new CyclicBarrier(10);

for (int i = 0; i < 10; i++) {
    executorService.submit(() -> {
        try {
            // 2. 每个线程"报告"自己到了，
            //    当第10个线程到的时候，也就是所有的线程都到齐了，一起通过
            barrier.await();
            
            // doWork()
            
        } catch (InterruptedException | BrokenBarrierException ex) {
            ex.printStackTrace();
        }
    });
}
```

> CyclicBarrier 的原理不是 AQS 的共享模式，是 AQS Condition 和 ReentrantLock 的结合使用

CyclicBarrier 可以被重复使用，我们这里只使用了一个周期，当第十个线程到了以后，所有的线程一起通过，此时开启了新的一个周期，在 CyclicBarrier 中，周期用 **generation** 表示。

3、我们来介绍今天的主角 `Phaser`，用 Phaser 实现这个需求也很简单：

```java
Phaser phaser = new Phaser();
// 1. 注册一个 party
phaser.register();

for (int i = 0; i < 10; i++) {
    
    phaser.register();
    
    executorService.submit(() -> {
        // 2. 每个线程到这里进行阻塞，等待所有线程到达栅栏
        phaser.arriveAndAwaitAdvance();
        
        // doWork()
    });
}
phaser.arriveAndAwaitAdvance();
```

Phaser 比较灵活，它不需要在构造的时候指定固定数目的 **parties**，而 CountDownLatch 和 CyclicBarrier 需要在构造函数中明确指定一个数字。

我们可以看到，上面的代码总共执行了 **11** 次 `phaser.register()` ，可以把 11 理解为 CountDownLatch 中的 count 和 CyclicBarrier 中的 parties。

这样读者应该很容易理解 `phaser.arriveAndAwaitAdvance()` 了，这是一个阻塞方法，直到该方法被调用 11 次，所有的线程才能同时通过。

> 这里和 CyclicBarrier 是一个意思，凑齐了所有的线程，一起通过栅栏。
>
> Phaser 也有周期的概念，一个周期定义为一个 **phase**，从 0 开始。

## Phaser 介绍

上面我们介绍了 Phaser 中的两个很重要的接口，register() 和 arriveAndAwaitAdvance()，这节我们来看它的其他的一些重要的接口使用。

画一张图压着：

![](https://javadoop.oss-cn-shanghai.aliyuncs.com/imgs/20510079/phaser/phaser-1.png)

### 重要接口介绍

Phaser 还是有 **parties** 概念的，但是它不需要在构造函数中指定，而是可以很灵活地动态增减。

我们来看 3 个代码片段，看看 parties 是怎么来的。

1、首先是 Phaser 有一个带 parties 参数的构造方法：

```java
public Phaser(int parties) {
    this(null, parties);
}
```

2、register() 方法：

```java
public int register() {
    return doRegister(1);
}
```

> 这个方法会使得 parties 加 1

3、bulkRegister(int parties) 方法：

```java
public int bulkRegister(int parties) {
    if (parties < 0)
        throw new IllegalArgumentException();
    if (parties == 0)
        return getPhase();
    return doRegister(parties);
}
```

> 一次注册多个，这个方法会使得 parties 增加相应数值

parties 也可以减少，因为有些线程可能在执行过程中，不和大家玩了，会进行退出，调用 `arriveAndDeregister()` 即可，这个方法的名字已经说明了它的用途了。

![](https://javadoop.oss-cn-shanghai.aliyuncs.com/imgs/20510079/phaser/phaser-1.png)

再看一下这个图，phase-1 结束的时候，黑色的线程离开了大家，此时就只有 3 个 parties 了。

这里说一下 Phaser 的另一个概念 **phase**，它代表 Phaser 中的周期或者叫阶段，phase 从 0 开始，一直往上递增。

通过调用 `arrive()` 或 `arriveAndDeregister()` 来标记有一个成员到达了一个 phase 的栅栏，当所有的成员都到达栅栏以后，开启一个新的 phase。

这里我们来看看和 phase 相关的几个方法：

**1、arrive()**

这个方法标记当前线程已经到达栅栏，但是该方法不会阻塞，注意，它不会阻塞。

> 大家要理解一点，party 本和线程是没有关系的，不能说一个线程代表一个 party，因为我们完全可以在一个线程中重复调用 arrive() 方法。这么表达纯粹是方便理解用。

**2、arriveAndDeregister()**

和上面的方法一样，当前线程通过栅栏，非阻塞，但是它执行了 deregister 操作，意味着总的 parties 减 1。

**3、arriveAndAwaitAdvance()**

这个方法应该一目了然，就是等其他线程都到了栅栏上再一起通过，进入下一个 phase。

**4、awaitAdvance(int phase)**

这个方法需要指定 phase 参数，也就是说，当前线程会进行阻塞，直到指定的 phase 打开。

**5、protected boolean onAdvance(int phase, registeredParties)**

这个方法是 protected 的，所以它不是 phaser 提供的 API，从方法名字上也可以看出，它会在一个 phase 结束的时候被调用。

它的返回值代表是否应该终结（terminate）一个 phaser，之所以拿出来说，是因为我们经常会见到有人通过覆写该方法来自定义 phaser 的终结逻辑，如：

```java
protected boolean onAdvance(int phase, int registeredParties) {
    return phase >= N || registeredParties == 0;
}
```

> 1、我们可以通过 `phaser.isTerminated()` 来检测一个 phaser 实例是否已经终结了
>
> 2、当一个 phaser 实例被终结以后，register()、arrive() 等这些方法都没有什么意义了，大家可以玩一玩，观察它们的返回值，原本应该返回 phase 值的，但是这个时候会返回一个负数。

### Phaser 的监控方法

介绍下几个用于返回当前 phaser 状态的方法：

getPhase()：返回当前的 phase，前面说了，phase 从 0 开始计算，最大值是 Integer.MAX_VALUE，超过又从 0 开始

getRegisteredParties()：当前有多少 parties，随着不断地有 register 和 deregister，这个值会发生变化

getArrivedParties()：有多少个 party 已经到达当前 phase 的栅栏

getUnarrivedParties()：还没有到达当前栅栏的 party 数

### Phaser 的分层结构

**Tiering** 这个词本身就不好翻译，大家将就一下，要表达的意思就是，将多个 Phaser 实例构造成一棵树。

1、第一个问题来了，为什么要把多个 Phaser 实例构造成一棵树，解决什么问题？有什么优点？

Phaser 内部用一个 `state` 来管理状态变化，随着 parties 的增加，并发问题带来的性能影响会越来越严重。

```java
/**
 * 0-15: unarrived
 * 16-31: parties，   所以一个 phaser 实例最大支持 2^16-1=65535 个 parties
 * 32-62: phase，     31 位，那么最大值是 Integer.MAX_VALUE，达到最大值后又从 0 开始
 * 63: terminated
 */
private volatile long state;
```

> 通常我们在说 0-15 位这种，说的都是从低位开始的

state 的各种操作依赖于 CAS，典型的无锁操作，但是，在大量竞争的情况下，可能会造成很多的自旋。

而构造一棵树就是为了降低每个节点（每个 Phaser 实例）的 parties 的数量，从而有效降低单个 state 值的竞争。

2、第二个问题，它的结构是怎样的？

这里我们不讲源码，用通俗一点的语言表述一下。我们先写段代码构造一棵树：

```java
Phaser root = new Phaser(5);

Phaser n1 = new Phaser(root, 5);
Phaser n2 = new Phaser(root, 5);

Phaser m1 = new Phaser(n1, 5);
Phaser m2 = new Phaser(n1, 5);
Phaser m3 = new Phaser(n1, 5);

Phaser m4 = new Phaser(n2, 5);
```

根据上面的代码，我们可以画出下面这个很简单的图：

![phaser](https://javadoop.oss-cn-shanghai.aliyuncs.com/imgs/20510079/phaser/phaser-2.png)

这棵树上有 7 个 phaser 实例，每个 phaser 实例在构造的时候，都指定了 parties 为 5，但是，对于每个拥有子节点的节点来说，每个子节点都是它的一个 party，我们可以通过 phaser.getRegisteredParties() 得到每个节点的 parties 数量：

- m1、m2、m3、m4 的 parties 为 5
- n1 的 parties 为 5 + 3，n2 的 parties 为 5 + 1
- root 的 parties 为 5 + 2

结论应该非常容易理解，我们来阐述一下过程。

在子节点注册第一个 party 的时候，这个时候会在父节点注册一个 party，注意这里说的是子节点添加第一个 party 的时候，而不是说实例构造的时候。

在上面代码的基础上，大家可以试一下下面的这个代码：

```java
Phaser m5 = new Phaser(n2);
System.out.println("n2 parties: " + n2.getRegisteredParties());
m5.register();
System.out.println("n2 parties: " + n2.getRegisteredParties());
```

第一行代码中构造了 m5 实例，但是此时它的 parties == 0，所以对于父节点 n2 来说，它的 parties 依然是 6，所以第二行代码输出 6。第三行代码注册了 m5 的第一个 party，显然，第四行代码会输出 7。

当子节点的 parties 降为 0 的时候，会从父节点中"剥离"，我们在上面的基础上，再加两行代码：

```java
m5.arriveAndDeregister();
System.out.println("n2 parties: " + n2.getRegisteredParties());
```

由于 m5 之前只有一个 parties，所以一次 arriveAndDeregister() 就会使得它的 parties 变为 0，此时第二行代码输出父节点 n2 的 parties 为 6。

> 还有一点有趣的是（其实也不一定有趣吧），在非树的结构中，此时 m5 应该处于 terminated 状态，因为它的 parties 降为 0 了，不过在树的结构中，这个状态由 root 控制，所以我们依然可以执行 m5.register()...

3、每个 phaser 实例的 phase 周期有快有慢，怎么协调的？

在组织成树的这种结构中，每个 phaser 实例的 phase 已经不受自己控制了，由 root 来统一协调，也就是说，root 当前的 phase 是多少，每个 phaser 的 phase 就是多少。

那又有个问题，如果子节点的一个周期很快就结束了，要进入下一个周期怎么办？需要等！这个时候其实要等所有的节点都结束当前 phase，因为只有这样，root 节点才有可能结束当前 phase。

我觉得 Phaser 中的树结构我们要这么理解，我们要把整棵树当做一个 phaser 实例，每个节点只是辅助用于降低并发而存在，整棵树还是需要满足 Phaser 语义的。

4、这种树结构在什么场景下会比较实用？设置每个节点的 parties 为多少比较合适？

这个问题留给读者思考吧。

（全文完）