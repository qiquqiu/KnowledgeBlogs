# 【多线程篇04】：synchronized应用与面向对象视角下的锁机制

> 原创 已于 2025-07-12 11:12:47 修改 · 公开 · 1k 阅读 · 40 · 22 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149242172

**文章目录**

[TOC]


为了保证共享数据的安全和程序的正确性，Java 提供了多种并发控制机制，其中 `synchronized` 关键字是最基础和常用的一个

在面向对象的思想下， `synchronized` 关键字的应用与对象和类紧密相连理解 `synchronized` 如何与对象和类结合，是掌握其用法和避免并发陷阱的关键

本文将从面向对象的视角，深入探讨 `synchronized` 在实例方法、静态方法以及代码块中的应用

##  `synchronized` 的本质：锁住对象

首先，我们需要明确 `synchronized` 关键字的本质：它用于获取一个 **对象** 的监视器锁（Monitor Lock），任何时刻，一个对象的监视器锁只能被一个线程持有，当一个线程获取了对象的锁后，其他试图获取该对象锁的线程将被阻塞，直到锁被释放

`synchronized` 可以应用于以下两种场景：

1. 同步代码块： `synchronized (object) { 需要同步的代码 }` 

2. 同步方法：直接修饰方法，包括实例方法和静态方法

无论哪种场景， `synchronized` 背后都需要一个锁对象

## 1. 实例方法的同步：锁住当前对象 ( `this` )

当我们使用 `synchronized` 修饰一个非静态方法时，实际上是为该方法添加了一个锁，这个锁就是该方法所属的当前对象实例 ( `this` )

示例：

```java
public class Number {
    private int count = 0;

    // 实例方法，使用 synchronized 修饰
    public synchronized void increase() {
        // 这段代码是临界区，需要同步
        count++;
        System.out.println(Thread.currentThread().getName() +  "increased count to"  + count);
    }

    // 等价于在方法内部使用 synchronized(this)
    public void decrease() {
        synchronized (this) {
            // 这段代码是临界区，需要同步
            count--;
            System.out.println(Thread.currentThread().getName() +  "decreased count to"  + count);
        }
    }

    public int getCount() {
        return count;
    }
}
```

理解：

>  `increase()` 方法和 `decrease()` 方法都使用了实例锁，如果存在两个不同的 `Number` 对象 `n1` 和 `n2` ：
> 
> 

- 线程 A 调用 `n1.increase()` ，它获取的是 `n1` 的锁

- 线程 B 调用 `n1.decrease()` ，它也尝试获取 `n1` 的锁，由于 `n1` 的锁被线程 A 持有，线程 B 会被阻塞，直到线程 A 释放 `n1` 的锁

- 线程 C 调用 `n2.increase()` ，它获取的是 `n2` 的锁

- 线程 A 和线程 C 可以并发执行，因为它们获取的是不同的锁（ `n1` 的锁和 `n2` 的锁）

结论： 实例方法的 `synchronized` 锁是对象级别的，它确保了同一时刻，同一个对象的同步实例方法不会被多个线程同时访问

---

## 2. 静态方法的同步：锁住类对象 ( `ClassName.class` )

当我们使用 `synchronized` 修饰一个静态方法时，由于静态方法不属于任何一个特定的对象实例，它没有 `this` 引用，此时， `synchronized` 锁住的是该方法所属的类的 `Class` 对象

> 每个类在JVM中都只有一个唯一的 `Class` 对象

示例：

```java
public class Counter {
    private static int totalCount = 0;

    // 静态方法，使用 synchronized 修饰
    public static synchronized void incrementTotal() {
        // 这段代码是临界区，需要同步
        totalCount++;
        System.out.println(Thread.currentThread().getName() +  "incremented totalCount to"  + totalCount);
    }

    // 等价于在方法内部使用 synchronized(Counter.class)
    public static void decrementTotal() {
        synchronized (Counter.class) {
            // 这段代码是临界区，需要同步
            totalCount--;
            System.out.println(Thread.currentThread().getName() +  "decremented totalCount to"  + totalCount);
        }
    }

    public static int getTotalCount() {
        return totalCount;
    }
}
```

理解：

> 

-  `incrementTotal()` 方法和 `decrementTotal()` 方法都使用了类锁
> 
> 

- 无论通过 `Counter.incrementTotal()` 直接调用，还是通过 `new Counter().incrementTotal()` （ *虽然不推荐这样调用静态方法* ），或者通过 `new Counter().decrementTotal()` ，它们竞争的都是同一个锁—— `Counter.class` 对象
> 
> 

- 因此，任何线程在访问 `Counter` 类的同步静态方法时，都会互斥
> 
> 

结论： 静态方法的 `synchronized` 锁是类级别的，它确保了同一时刻，无论有多少个该类的实例，甚至没有实例，只要是访问该类的同步静态方法，都会互斥

---

## 3. 同步代码块的灵活性：指定任意对象作为锁

除了修饰方法， `synchronized` 还可以用来修饰一个代码块，此时需要在括号中明确指定作为锁的对象

语法： `synchronized (object) { 需要同步的代码 }` 

这里的 `object` 可以是任何非 null 的对象引用

示例：

```java
public class MixedSync {
    private Object lock1 = new Object(); // 用于同步某个代码块的锁对象
    private Object lock2 = new Object(); // 用于同步另一个代码块的锁对象
    private int data1 = 0;
    private int data2 = 0;

    public void updateData1() {
        synchronized (lock1) {
            // 同步操作 data1
            data1++;
            System.out.println(Thread.currentThread().getName() +  "updated data1 to"  + data1);
        }
    }

    public void updateData2() {
        synchronized (lock2) {
            // 同步操作 data2
            data2++;
            System.out.println(Thread.currentThread().getName() +  "updated data2 to"  + data2);
        }
    }

    public void updateAllData() {
        // 如果需要同时同步 data1 和 data2，可以使用同一个锁
        synchronized (lock1) { // 或者 synchronized(lock2)
             data1++;
             data2++;
             System.out.println(Thread.currentThread().getName() +  "updated all data");
        }
    }

    // 注意：使用 this 作为锁等价于同步实例方法
    public void syncWithThis() {
        synchronized (this) {
            // ...
        }
    }

    // 注意：使用 ClassName.class 作为锁等价于同步静态方法
    public static void syncWithClass() {
         synchronized (MixedSync.class) {
            // ...
         }
    }
}
```

理解：

> 

- 同步代码块的优势在于可以更加精细地控制锁的粒度，我们可以只同步需要保护的临界区代码，而不是整个方法

- 通过使用不同的锁对象 ( `lock1` , `lock2` )，我们可以实现细粒度的同步，允许线程同时访问对象的不同部分，只要它们不竞争同一个锁，例如，一个线程在执行 `updateData1()` 时，另一个线程可以同时执行 `updateData2()` ，因为它们使用了不同的锁

- 使用同步代码块时，选择合适的锁对象至关重要，通常推荐使用专门的 `private final Object` 字段作为锁对象，以避免外部代码意外地获取到你的锁，或者因为自动装箱拆箱、对象缓存等问题导致锁失效

结论： 同步代码块提供了最大的灵活性，允许我们指定任意对象作为锁，从而实现更细粒度的同步控制

---

## 总结

| 场景 |  `synchronized` 用法 | 锁对象 | 锁的粒度 | 影响范围 |
|:---|:---|:---|:---|:---|
|  **实例方法**  |  `synchronized void method() { ... }`  | 当前实例 ( `this` ) | 对象级别 | 同一实例的同步方法互斥 |
|  **静态方法**  |  `static synchronized void method() { ... }`  | 类的 `Class` 对象 | 类级别 | 该类的所有静态同步方法互斥 |
|  **代码块**  |  `synchronized (object) { ... }`  | 指定的 `object`  | 任意 | 竞争同一个 `object` 锁的代码块会互斥 |