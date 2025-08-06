# 【多线程篇07】：一文看懂synchronized之外的的线程同步方案

> 原创 于 2025-07-13 09:00:23 发布 · 公开 · 1k 阅读 · 23 · 9 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149305476

**文章目录**

[TOC]


`synchronized` 强大、易用，能够解决绝大多数的线程同步问题。然而，随着我们对系统性能和并发控制的要求越来越高，仅仅掌握 `synchronized` 是不够的

Java 的 `java.util.concurrent` (JUC) 包为我们提供了更丰富、更强大的并发工具。本文将深入探讨三种 `synchronized` 之外的关键并发机制： `ReentrantLock` 、 `volatile` 和原子类（Atomic Classes）

## 1. `ReentrantLock` ：更灵活的显式锁

`ReentrantLock` 是 `java.util.concurrent.locks` 包下的一个类，它实现了 `Lock` 接口，提供了一种比 `synchronized` 更灵活、功能更强大的锁机制

**核心区别：** 

-  `synchronized` 是 Java 关键字，是 **隐式锁** 。锁的获取和释放由 JVM 自动完成，代码更简洁

-  `ReentrantLock` 是一个 **显式锁** 。你需要手动调用 `lock()` 方法获取锁，并且 **必须** 在 `finally` 块中调用 `unlock()` 方法释放锁，以确保锁总能被释放

**标准用法：** 

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class Counter {
    private final Lock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock(); // 1. 手动获取锁
        try {
            count++;
            // ... 其他业务逻辑
        } finally {
            lock.unlock(); // 2. 必须在 finally 块中释放锁
        }
    }
}
```

**`ReentrantLock` 的高级特性：** 

1.  **可中断的锁获取 ( `lockInterruptibly` )** 

   - 当一个线程使用 `lock()` 等待锁时，它是不可被中断的

   - 而使用 `lock.lockInterruptibly()` 时，如果线程在等待锁的过程中被中断（ `thread.interrupt()` ），它会抛出 `InterruptedException` 并停止等待，给予了我们处理阻塞的能力

2.  **可限时的锁获取 ( `tryLock` )** 

   -  `lock.tryLock()` ：尝试立即获取锁，如果锁可用则返回 `true` ，否则立即返回 `false` ，线程 **不会阻塞** 

   -  `lock.tryLock(long time, TimeUnit unit)` ：在指定的时间内尝试获取锁，如果在时间内成功获取，返回 `true` ；超时则返回 `false` 。这对于避免死锁和实现优雅降级非常有用

3.  **公平锁与非公平锁** 

   -  `synchronized` 只有非公平锁，即允许新来的线程“插队”获取锁，这可能导致某些线程长时间等待（饥饿）

   -  `ReentrantLock` 允许我们通过构造函数选择锁的类型：

     -  `new ReentrantLock()` ：默认创建非公平锁（性能更高）

     -  `new ReentrantLock(true)` ：创建公平锁，等待时间最长的线程将优先获取锁

**何时使用 `ReentrantLock` ？** 

> 当 `synchronized` 的功能无法满足需求时，比如需要可中断的锁、限时等待、公平锁等高级功能时，使用 `ReentrantLock` 更好

---

## 2. `volatile` ：保证可见性

`volatile` 是一个变量修饰符，它不是锁，但它在并发编程中扮演着至关重要的角色。它被称为最轻量级的同步机制。

**核心保证：可见性** 

-  **可见性问题** ：在多核 CPU 架构下，每个线程可能拥有自己的高速缓存。当一个线程修改了主内存中的共享变量后，它可能只是更新了自己的缓存，而没有立即写回主内存。其他线程此时读取的可能是旧的、未更新的值

-  **`volatile` 的作用** ：当一个变量被声明为 `volatile` 后，JVM 会确保：

  1. 对该变量的 **写操作** 会立即刷新到主内存

  2. 对该变量的 **读操作** 会直接从主内存读取，而不是使用线程缓存

**重要限制：不保证原子性** 

> 这是 `volatile` 最容易被误解的地方。它只保证了可见性， **不保证操作的原子性** 

以 `count++` 为例，这个操作实际上包含三个步骤：

1. 读取 `count` 的值

2. 将值加 1

3. 将新值写回

即使 `count` 被 `volatile` 修饰，多个线程也可能同时执行第一步（读取到相同的值），然后各自加一并写回，最终导致结果小于预期

**何时使用 `volatile` ？** 

>  `volatile` 的最佳使用场景是“一写多读”或者作为状态标记，并且写入操作不依赖于变量当前的值

**示例：使用 `volatile` 作为停止标记** 

```java
class StoppableTask implements Runnable {
    // 使用 volatile 保证 stopRequested 的可见性
    private volatile boolean stopRequested = false;

    public void requestStop() {
        this.stopRequested = true;
    }

    @Override
    public void run() {
        while (!stopRequested) {
            // ... 执行任务
        }
        System.out.println("Task stopped.");
    }
}
```

---

## 3. 原子类：无锁编程

对于 `count++` 这种非原子性操作的线程安全问题，除了使用 `synchronized` 或 `ReentrantLock` 加锁，我们还有一种性能更好的选择——原子类

Java 在 `java.util.concurrent.atomic` 包下提供了一系列原子类，如 `AtomicInteger` , `AtomicLong` , `AtomicBoolean` , `AtomicReference` 等

**核心机制：CAS (Compare-And-Swap)** 

原子类底层利用了现代 CPU 提供的 **CAS（比较并交换） **指令，这是一种** 乐观锁** 思想

-  **工作原理** ：CAS 操作包含三个操作数——内存位置 V、预期原值 A 和新值 B。当且仅当内存位置 V 的值与预期原值 A 相同时，处理器才会原子地将该位置的值更新为新值 B。否则，它什么也不做

-  **Java 中的实现** ：原子类中的 `incrementAndGet()` 等方法内部是一个循环：

  1. 读取当前值（预期原值 A）

  2. 计算新值 B

  3. 使用 CAS 尝试将 A 更新为 B

  4. 如果 CAS 失败（说明在步骤 1 到 3 之间，有其他线程修改了值），则重复步骤 1-3，直到成功为止

**示例：使用 `AtomicInteger` 实现线程安全的计数器** 

```java
import java.util.concurrent.atomic.AtomicInteger;

class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet(); // 原子地加 1
    }

    public int getCount() {
        return count.get();
    }
}
```

**何时使用原子类？** 

> 当需要对 **单个共享变量** 进行原子更新时（如计数器、状态切换），原子类是最佳选择。它避免了线程阻塞和上下文切换的开销，通常比重量级锁有更好的性能

---

## 总结与选择

| 特性 |  `synchronized`  |  `ReentrantLock`  |  `volatile`  | 原子类 (Atomic) |
|:---|:---|:---|:---|:---|
|  **本质**  | 关键字 (隐式锁) | 类 (显式锁) | 关键字 (变量修饰符) | 类 (CAS无锁) |
|  **原子性**  | 保证 | 保证 |  **不保证**  | 保证 (对单个变量) |
|  **可见性**  | 保证 | 保证 | 保证 | 保证 |
|  **灵活性**  | 低 | 高 (可中断、可超时、可公平) | 不适用 | 中 (针对单个变量) |
|  **使用场景**  | 通用的同步代码块和方法 | 复杂的同步逻辑 | 状态标记、一写多读 | 原子计数、原子更新 |
|  **性能**  | 重量级，有锁竞争时开销大 | 重量级，但更灵活 | 轻量级 | 轻量级，性能通常最优 |


**选择建议：** 

-  **优先考虑简单性** ：如果 `synchronized` 能够满足需求，它通常是首选，因为代码更简洁，不易出错

-  **需要高级功能** ：如果需要中断、超时、公平锁等高级功能，选择 `ReentrantLock` 

-  **仅需保证可见性** ：对于简单的状态标记，使用 `volatile` 是最高效的选择

-  **原子更新单个变量** ：对于计数器等场景， `Atomic` 原子类是性能最佳的解决方案

