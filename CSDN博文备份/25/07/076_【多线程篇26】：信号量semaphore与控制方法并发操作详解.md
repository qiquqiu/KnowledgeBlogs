# 【多线程篇26】：信号量semaphore与控制方法并发操作详解

> 原创 于 2025-07-24 08:30:00 发布 · 公开 · 669 阅读 · 18 · 17 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149587087

**文章目录**

[TOC]



> 对于Java后端开发者来说，并发编程是必须掌握的核心技能。在我们的工具箱中，除了大家熟知的 `synchronized` 和 `ReentrantLock` 用于保证线程互斥外，还有一个强大的工具—— `Semaphore` （信号量），它专注于另一个维度： **控制对特定资源的并发访问线程数量** 
> 
> 本文将从 `Semaphore` 是什么、核心方法、实战应用到其底层原理（AQS），进行一次全面且精准的解析

## 一、 什么是Semaphore？

`Semaphore` ，通常被称为“信号量”，是计算机科学中一个经典的概念，由Edsger Dijkstra在20世纪60年代提出。在Java的JUC包中， `Semaphore` 是一个 **计数信号量** 

我们可以用一个非常形象的例子来理解它：

**停车场模型** ：
想象一个有N个车位的停车场

-  **`Semaphore`** 就是这个停车场的管理员

-  **`permits` （许可）** 就是停车场的总车位数N

-  **线程** 就是想要进入停车场的汽车

**工作流程** ：

1.  **`acquire()` （获取许可）** ：一辆车（线程）到达停车场入口，向管理员（ `Semaphore` ）申请一个车位（ `permit` ）

   - 如果停车场有空位（ `permits > 0` ），管理员放行，车辆进入，可用车位数减一

   - 如果停车场已满（ `permits == 0` ），车辆就必须在入口外排队等待（线程阻塞），直到有车离开

2.  **`release()` （释放许可）** ：一辆车（线程）离开停车场，它会通知管理员（ `Semaphore` ），管理员将可用车位数加一。这时，如果入口外有正在排队的车辆，管理员会唤醒其中一辆，让它进入

**核心区别** ：与 `synchronized` 或 `ReentrantLock` 一次只允许一个线程访问的“互斥锁”不同， `Semaphore` 可以 **允许多个线程同时访问一个资源** ，但它会 **限制这个“同时”的数量** 

---

## 二、 核心机制与关键方法

`Semaphore` 的API非常直观，主要围绕“获取”和“释放”许可

### 1. 构造函数

```java
// 创建一个具有给定许可数量的Semaphore，默认采用非公平策略
public Semaphore(int permits)

// 创建一个具有给定许可数量和指定公平策略的Semaphore
public Semaphore(int permits, boolean fair)
```

-  `permits` : 初始许可数量，也就是资源池的大小

-  `fair` (公平策略):

  -  `false` (默认，非公平): 当一个许可被释放时，任何正在等待的线程（以及刚到达的线程）都可以尝试获取它，可能会出现“插队”现象，整体吞吐量较高

  -  `true` (公平): 严格按照线程请求的先后顺序（FIFO）来分配许可。等待时间最长的线程将优先获得许可

### 2. 主要方法

-  **`void acquire()`** : 获取一个许可。如果当前没有可用许可，线程将进入休眠状态，直到有许可被释放。此方法会响应中断

-  **`void acquire(int permits)`** : 一次性获取指定数量（ `permits` ）的许可。如果可用许可不足，线程将阻塞

-  **`void release()`** : 释放一个许可，使其返回信号量。可用许可数量加一

-  **`void release(int permits)`** : 一次性释放指定数量的许可

-  **`boolean tryAcquire()`** : 尝试获取一个许可。无论成功与否，都会立即返回。成功返回 `true` ，失败返回 `false` 。这是非阻塞的

-  **`int availablePermits()`** : 返回当前可用的许可数量

**一个非常重要的细节** ： `release()` 方法并不要求必须由获取了许可的那个线程来调用。任何线程都可以调用 `release()` 来增加许可数量。这与 `ReentrantLock` 的 `unlock()` 必须由加锁线程执行形成了鲜明对比

---

## 三、 实战：如何用Semaphore控制方法并发访问数

这是 `Semaphore` 最经典的应用场景。假设我们有一个服务，需要调用一个第三方API，但该API限制了我们的应用最多只能有3个并发请求

**场景** ：模拟10个用户同时请求一个最多只允许3个并发的服务

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * 使用Semaphore控制对某个方法的并发访问线程数
 */
public class SemaphoreDemo {

    // 1. 创建一个Semaphore实例，设置许可数量为3
    private static final Semaphore semaphore = new Semaphore(3);

    public static void accessLimitedResource() {
        try {
            // 2. 在方法开始时，获取一个许可
            semaphore.acquire();

            System.out.println(Thread.currentThread().getName() + " 获取了许可，正在访问资源...");
            // 模拟业务处理耗时
            TimeUnit.SECONDS.sleep(2);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // 恢复中断状态
            e.printStackTrace();
        } finally {
            // 3. 关键：无论业务是否成功，都必须在finally块中释放许可
            System.out.println(Thread.currentThread().getName() + " 访问完毕，释放许可");
            semaphore.release();
        }
    }

    public static void main(String[] args) {
        // 创建一个固定大小的线程池，模拟10个并发用户
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        for (int i = 0; i < 10; i++) {
            executorService.submit(() -> {
                accessLimitedResource();
            });
        }

        executorService.shutdown();
    }
}
```

**代码分析** ：

1. 我们创建了一个 `Semaphore` 实例，并将许可数初始化为 `3` 。这意味着我们的 `accessLimitedResource` 方法在任何时刻最多只能被3个线程同时执行

2. 在 `accessLimitedResource` 方法的入口处，线程调用 `semaphore.acquire()` 。如果当时已有3个线程在执行此方法，那么新的线程就会在此处阻塞等待

3.  **最重要的部分** ： `semaphore.release()` 被放置在 `finally` 块中。这是一个必须遵守的最佳实践。 **确保即使业务逻辑（ `try` 块内）发生异常，许可也一定会被释放** ，否则许可将会永久丢失，导致其他线程永远无法进入，造成“许可泄漏”

4.  `main` 方法中，我们用一个10个线程的线程池去并发调用这个方法

**预期输出（顺序可能不同，但规律一致）** ：

```
pool-1-thread-1 获取了许可，正在访问资源...
pool-1-thread-3 获取了许可，正在访问资源...
pool-1-thread-2 获取了许可，正在访问资源...
// (此时，另外7个线程正在acquire()处等待)

// --- 大约2秒后 ---
pool-1-thread-1 访问完毕，释放许可
pool-1-thread-4 获取了许可，正在访问资源...
pool-1-thread-2 访问完毕，释放许可
pool-1-thread-5 获取了许可，正在访问资源...
pool-1-thread-3 访问完毕，释放许可
pool-1-thread-6 获取了许可，正在访问资源...
// ... 后续依此类推
```

从输出可以看到，系统始终保持 **最多只有3个线程** 在“访问资源”，完美地实现了并发流量控制

---

## 四、 深入原理：AQS（AbstractQueuedSynchronizer）

下面来了解 `Semaphore` 的底层实现：和 `ReentrantLock` 、 `CountDownLatch` 一样， `Semaphore` 也是基于 **AQS（ `AbstractQueuedSynchronizer` ）** 构建的

-  **状态（ `state` ）** ：AQS内部维护一个 `int` 类型的 `state` 变量。在 `Semaphore` 中， **`state` 的值就代表当前可用的许可数量** 

-  **共享模式（Shared Mode）** ： `Semaphore` 使用的是AQS的 **共享模式** 。因为许可可以被多个线程同时持有。与之相对的是 `ReentrantLock` 使用的独占模式（ `state` 只能被一个线程持有）

-  **`acquire()` 的背后** ：调用 `semaphore.acquire()` 实际上是调用了AQS的 `acquireSharedInterruptibly(1)` 。这个方法会尝试以共享模式获取资源（在这里是许可）

  - 它会尝试通过CAS（Compare-And-Swap）操作去减少 `state` 的值

  - 如果成功，方法返回

  - 如果失败（比如 `state` 已经是0），AQS会将当前线程封装成一个节点（Node），并将其加入到一个同步队列（一个双向链表）中，然后将线程 **挂起** （ `park` ）

-  **`release()` 的背后** ：调用 `semaphore.release()` 实际上是调用了AQS的 `releaseShared(1)` 

  - 这个方法会通过CAS操作增加 `state` 的值

  - 操作成功后，它会去唤醒在同步队列中等待的后继节点（ `unpark` ），让被唤醒的线程再次尝试获取许可

**公平与非公平的实现** ：

-  **非公平** ： `acquire()` 时，新来的线程会直接尝试CAS修改 `state` ，如果成功就直接拿走许可，这就是“插队”

-  **公平** ： `acquire()` 时，线程会先检查同步队列中是否有比自己等待时间更长的线程。如果有，它就不会去尝试获取许可，而是乖乖排到队尾

---

## 五、 要点与总结

1.  **它的作用是什么？** 

   - 它是一个计数信号量，用于控制对共享资源并发访问的线程数量。它不是为了互斥，而是为了限流

2.  **它和 `synchronized` / `ReentrantLock` 有什么区别？** 

   -  **目的不同** ： `Lock` 是为保证临界区代码的互斥执行（一次一个）。 `Semaphore` 是为限制并发数量（一次N个）

   -  **许可管理** ： `Lock` 的加锁和解锁必须是同一个线程。 `Semaphore` 的 `acquire()` 和 `release()` 可以不是同一个线程，这为更复杂的协作场景提供了可能

3.  **使用 `Semaphore` 时最重要的注意事项是什么？** 

   -  **必须在 `finally` 块中调用 `release()`** ，以防止因异常导致的许可泄漏

4.  **它的底层实现是什么？** 

   - 基于JUC的核心框架 **AQS** 的 **共享模式** 实现

   - 内部的 `state` 变量表示可用许可数

   - 可以简单描述公平与非公平策略在AQS层面的实现差异（是否检查排队情况）

