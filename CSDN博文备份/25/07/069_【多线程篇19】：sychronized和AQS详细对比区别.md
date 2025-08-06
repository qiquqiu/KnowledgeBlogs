# 【多线程篇19】：sychronized和AQS详细对比区别

> 原创 于 2025-07-21 08:15:00 发布 · 公开 · 685 阅读 · 29 · 30 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149480638

**文章目录**

[TOC]


谈到线程安全，就离不开“锁”， `synchronized` 关键字和 `AQS` (AbstractQueuedSynchronizer) 框架是Java中实现锁机制的两大基石，它们一个简单直接，一个灵活强大

> 关于二者具体的讲解，参考：
> 
> 

-  [【多线程篇04】：synchronized应用与面向对象视角下的锁机制](https://blog.csdn.net/lyh2004_08/article/details/149242172) 

-  [【多线程篇18】：一文搞懂AQS核心原理](https://blog.csdn.net/lyh2004_08/article/details/149480328) 

本文将深入对比这两者，帮助理解它们的本质区别、适用场景，从而在实际开发中做出最优选择

## 一、 `synchronized` 与 AQS

###  `synchronized` ：JVM内置的监视器

`synchronized` 是Java的一个关键字，也是最古老、最简单的线程同步方式。它可以修饰方法或代码块，确保同一时刻只有一个线程可以执行被保护的代码。

```java
public class SynchronizedExample {
    // 修饰方法
    public synchronized void doSomething() {
        // ... 临界区代码 ...
    }

    // 修饰代码块
    public void doAnotherThing() {
        synchronized (this) {
            // ... 临界区代码 ...
        }
    }
}
```

它的最大特点是 **简单、隐式** 。你只需要将它放在那里，JVM就会为你处理好一切，包括在代码块正常结束或抛出异常时自动释放锁。

### AQS：API层面的框架

`AQS` （ `java.util.concurrent.locks.AbstractQueuedSynchronizer` ）则完全不同。它不是一个可以直接使用的锁，而是一个用来构建锁和同步器的 **抽象框架** 。我们熟知的 `ReentrantLock` 、 `Semaphore` 、 `CountDownLatch` 等并发工具，其核心都是基于AQS实现的。

AQS内部维护了一个 `state` 变量（表示同步状态）和一个FIFO的线程等待队列。它通过CAS（Compare-And-Swap）原子操作来修改 `state` ，并管理那些获取锁失败的线程，将它们放入队列中挂起。

`ReentrantLock` 是AQS最典型的应用，我们通常用它来与 `synchronized` 进行对比。

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class LockExample {
    private final Lock lock = new ReentrantLock();

    public void performAction() {
        lock.lock(); // 手动加锁
        try {
            // ... 临界区代码 ...
        } finally {
            lock.unlock(); // 必须在finally块中手动解锁
        }
    }
}
```

---

## 二、全方位深度对比

从多个维度进行深入剖析：

| 特性 | synchronized | AQS (以ReentrantLock为例) |
|:---|:---|:---|
|  **本质与层面**  | Java关键字，JVM层面实现（底层C++） | Java类库，纯Java实现（API层面） |
|  **锁的释放**  |  **自动释放** （代码块结束或异常） |  **手动释放** （必须在 `finally` 中调用 `unlock` ） |
|  **功能灵活性**  | 功能单一，基本同步 | 功能丰富，高度可控 |
|  *公平性*  | 非公平锁 | 可选公平/非公平锁（默认非公平） |
|  *可中断*  | 不可中断 | 可中断（ `lockInterruptibly()` ） |
|  *尝试获取*  | 不支持 | 支持，可带超时（ `tryLock()` ） |
|  *条件变量*  | 单一条件 ( `wait` / `notify` ) | 支持多个Condition，可分组唤醒 |
|  **性能**  | JDK1.6后优化显著，低竞争下性能优异 | 性能与优化后 `synchronized` 相当，高竞争下功能优势明显 |


### 1. 实现层面：关键字 vs. API

-  `synchronized` 是由JVM直接支持的。当你使用它时，JVM会生成相应的 `monitorenter` 和 `monitorexit` 字节码指令，其具体实现依赖于底层操作系统。

-  `AQS` 及其子类是纯Java代码，位于JDK的 `java.util.concurrent` 包中。它为开发者提供了更透明、更可控的编程接口。

### 2. 锁操作：自动 vs. 手动

-  **`synchronized` 的省心** ：这是它最大的优点之一。你永远不用担心忘记释放锁，因为JVM会为你代劳，这极大地降低了死锁的风险。

-  **`ReentrantLock` 的严谨** ：必须手动调用 `lock()` 和 `unlock()` 。为了保证锁一定会被释放， `unlock()` 必须放在 `finally` 块中。这虽然增加了代码量，但也提供了在加锁和解锁之间执行特定逻辑的可能。

### 3. 功能灵活性：简单粗暴 vs. 精雕细琢

这是两者最核心的区别。 `synchronized` 像一把简单可靠的锤子，而 `ReentrantLock` 则像一套功能齐全的瑞士军刀。

-  **公平性** ： `synchronized` 是非公平的，新来的线程可能“插队”成功。而 `ReentrantLock` 允许你选择公平模式，严格按照线程请求的顺序来分配锁，虽然会带来一些性能开销。

-  **可中断等待** ：如果一个线程长时间等待 `synchronized` 锁，它是无法被中断的。而 `ReentrantLock` 的 `lockInterruptibly()` 方法允许正在等待的线程响应中断，这对于处理耗时任务和避免死锁非常有帮助。

-  **尝试与超时** ： `ReentrantLock` 的 `tryLock()` 方法可以立即返回是否成功获取锁，或者在指定时间内等待。这个功能在需要避免线程无限期阻塞的场景下极为有用。

-  **多条件等待** ： `synchronized` 的 `wait/notify` 机制只能唤醒一个或所有等待线程，无法精确控制。而 `ReentrantLock` 可以创建多个 `Condition` 对象，将线程分组放在不同的条件队列上，实现“精准唤醒”。

### 4. 性能

在JDK 1.6之前， `synchronized` 是一个纯粹的“重量级锁”，性能较差。但如今，JVM引入了 **偏向锁、轻量级锁、自旋** 等一系列优化，使得 `synchronized` 在锁竞争不激烈的情况下性能非常高。

-  **低竞争场景** ： `synchronized` 和 `ReentrantLock` 的性能几乎没有差别。

-  **高竞争场景** ：两者性能也相差不大。但 `ReentrantLock` 提供的公平性选择和更灵活的机制，能帮助开发者更好地调优和应对复杂的并发问题。

---

## 三、如何选择：场景决定一切

那么，在实际开发中我们应该如何选择呢？

-  **优先选择 `synchronized`** ：

  - 如果同步逻辑非常简单，锁的粒度明确。

  - 在低竞争环境下，或者可以确定锁的持有时间很短。

  - 当你不希望为锁的管理操心，追求代码的简洁性和安全性时。

-  **选择 `AQS` ( `ReentrantLock` 等)** ：

  - 当你需要 `synchronized` 不具备的高级功能时，例如：

    - 需要 **可中断** 的锁获取操作。

    - 需要 **超时** 获取锁，避免无限等待。

    - 需要实现 **公平锁** 。

    - 需要使用 **多个条件变量** 进行复杂的线程通信。

  - 在复杂的、高竞争的并发场景中，需要对锁进行更精细的控制和调优。

---

## 总结

`synchronized` 和 `AQS` (以 `ReentrantLock` 为代表) 并非相互替代的关系，而是互为补充的

-  `synchronized` 是Java并发的基石，简单、可靠，是处理简单同步问题的首选

-  `AQS` 则是Java并发工具的“大师”，它提供了构建高级同步工具的能力， `ReentrantLock` 等实现类赋予了我们应对复杂并发场景的强大武器

