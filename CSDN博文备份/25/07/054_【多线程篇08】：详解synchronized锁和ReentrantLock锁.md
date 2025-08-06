# 【多线程篇08】：详解synchronized锁和ReentrantLock锁

> 原创 于 2025-07-13 09:27:33 发布 · 公开 · 675 阅读 · 16 · 22 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149305734

**文章目录**

[TOC]


`synchronized` 和 `ReentrantLock` 都实现了“可重入锁”的功能，但其设计哲学、使用方式和底层实现却大相径庭。本文将从五个核心维度，深入剖析二者的区别，帮助你理解在何种场景下该如何选择

## 1. API 与用法层面：隐式 vs. 显式

这是两者最直观的区别。 `synchronized` 是隐式的，而 `ReentrantLock` 是显式的

-  **`synchronized` (隐式锁)** 
  它是一个Java 关键字，由JVM自动管理锁的获取和释放，语法简洁。它可以修饰实例方法、静态方法和代码块

  ```java
  // 修饰实例方法，锁是 this 对象
  public synchronized void syncMethod() {
      // ...
  }

  // 修饰代码块，锁是括号内的对象
  public void syncBlock() {
      synchronized (this) {
          // ...
      }
  }
  ```

-  **`ReentrantLock` (显式锁)** 
  它是一个实现了 `Lock` 接口的类，需要开发者手动获取 ( `lock()` ) 和释放 ( `unlock()` ) 锁。为了保证锁一定会被释放， **释放锁的操作必须放在 `finally` 块中** 

  ```java
  import java.util.concurrent.locks.Lock;
  import java.util.concurrent.locks.ReentrantLock;

  class Counter {
      private final Lock lock = new ReentrantLock();

      public void performAction() {
          lock.lock(); // 手动获取锁
          try {
              // 临界区代码
          } finally {
              lock.unlock(); // 必须在 finally 中手动释放锁
          }
      }
  }
  ```

---

## 2. 锁的公平性：非公平 vs. 可选择

当多个线程等待同一个锁时，锁的分配策略决定了其公平性

-  **`synchronized` ：非公平锁** 
   `synchronized` 只支持 **非公平锁** 。当锁被释放时，任何一个正在等待的线程都有机会获取锁，甚至一个刚刚到达的线程也可能“插队”成功，这可能导致某些线程长时间等待，即“线程饥饿”

-  **`ReentrantLock` ：可公平可非公平** 
   `ReentrantLock` 提供了选择的灵活性。通过构造函数可以指定锁的类型：

  -  `new ReentrantLock()` 或 `new ReentrantLock(false)` ：创建 **非公平锁** （默认）。非公平锁的吞吐量通常更高，因为减少了线程调度的开销

  -  `new ReentrantLock(true)` ：创建 **公平锁** 。锁会按照线程请求的先后顺序进行分配，遵循先来后到（FIFO）的原则，避免了线程饥饿

---

## 3. 等待可中断：不可中断 vs. 可中断

这是 `ReentrantLock` 相较于 `synchronized` 的一个核心优势

-  **`synchronized` ：不可中断** 
  如果一个线程在等待获取 `synchronized` 锁时被阻塞，它会一直等待下去，无法响应中断 ( `Thread.interrupt()` )。这种特性在某些情况下可能导致死锁无法被解除

-  **`ReentrantLock` ：可中断** 
   `ReentrantLock` 提供了 `lockInterruptibly()` 方法。如果一个线程通过此方法等待锁，它可以在等待过程中响应中断信号，即立即停止等待并抛出 `InterruptedException` 。这使得开发者可以构建更灵活的、能够处理线程阻塞的系统

  ```java
  try {
      // 尝试获取一个可以被中断的锁
      lock.lockInterruptibly();
      // ...
  } catch (InterruptedException e) {
      // 线程在等待锁的过程中被中断，可以在这里处理中断逻辑
      Thread.currentThread().interrupt(); // 重新设置中断状态
  } finally {
      if (lock.isHeldByCurrentThread()) {
          lock.unlock();
      }
  }
  ```

---

## 4. 功能丰富性：基础 vs. 高级

`ReentrantLock` 作为 `Lock` 接口的实现，提供了更多高级功能

-  **`synchronized` ：** 功能相对单一，主要就是 **互斥和可重入** 

-  **`ReentrantLock` ：** 

  -  **尝试获取锁 ( `tryLock` )** ： `tryLock()` 方法可以尝试 **非阻塞地** 获取锁，如果锁可用则立即返回 `true` ，否则返回 `false` 。还有一个带超时时间的版本 `tryLock(long time, TimeUnit unit)` ，这对于避免死锁非常有用

  -  **条件变量 ( `Condition` )** ： `ReentrantLock` 可以绑定多个 `Condition` 对象 ( `lock.newCondition()` )。 `Condition` 提供了比 `Object` 的 `wait()` 、 `notify()` 、 `notifyAll()` 更强大的线程协作能力，可以实现更精细的线程等待与唤醒控制（例如，生产者-消费者模型中，可以有“仓库未满”和“仓库不空”两个独立的等待队列）

---

## 5. 底层实现：JVM vs. AQS

-  **`synchronized` ：JVM 层面** 
  它是由 JVM 直接实现的，涉及到 `monitorenter` 和 `monitorexit` 这两个字节码指令。现代 JVM 对 `synchronized` 进行了大量优化，包括偏向锁、轻量级锁和重量级锁的锁升级机制，使其在无竞争或低竞争场景下的性能表现非常好。

-  **`ReentrantLock` ：API 层面 (AQS)** 
  它是基于 Java API 实现的，其核心是 **AQS (AbstractQueuedSynchronizer)** 。AQS 是 JUC 包的基石，它通过一个 `volatile` 的 `int` 类型变量表示同步状态，并使用一个 FIFO 的双向队列来管理等待的线程。像 `Semaphore` 、 `CountDownLatch` 等并发工具都是基于 AQS 构建的

---

## 总结与如何选择

| 特性 |  `synchronized`  |  `ReentrantLock`  |
|:---|:---|:---|
|  **本质**  | Java 关键字 (隐式) | JUC 中的类 (显式) |
|  **用法**  | 修饰方法、代码块 | 只能用于代码块，需手动加/解锁 |
|  **公平性**  | 仅非公平锁 | 可选公平/非公平 |
|  **中断响应**  | 不可中断 | 可中断 ( `lockInterruptibly` ) |
|  **高级功能**  | 基础 (wait/notify) | 丰富 ( `tryLock` , `Condition` ) |
|  **底层实现**  | JVM (Monitor) | JDK API (AQS) |


**选择建议：** 

1.  **首选 `synchronized`** ：在绝大多数情况下，如果同步逻辑不复杂， `synchronized` 是首选。它的语法更简单，代码可读性更高，不易出错（不会忘记释放锁），并且 JVM 对其持续优化，性能已经非常出色

2.  **选择 `ReentrantLock` 的场景** ：当且仅当你需要 `synchronized` 无法提供的以下高级功能时，才应该考虑使用 `ReentrantLock` ：

   - 需要使用 **公平锁** 来避免线程饥饿

   - 需要 **可中断的锁获取** 操作，以构建更具响应性的系统

   - 需要 **限时获取锁** ( `tryLock` )，以避免死锁或进行优雅降级

   - 需要使用 **多个条件变量 ( `Condition` )** 来实现复杂的线程间通信

