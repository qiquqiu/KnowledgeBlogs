# 【多线程篇16】：ThreadLocal 每个线程的专属“盒子”

> 原创 于 2025-07-19 08:39:53 发布 · 公开 · 416 阅读 · 4 · 4 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149459563

**文章目录**

[TOC]


`ThreadLocal` 类允许每个线程绑定自己的值，可以将其形象地比喻为一个“存放数据的盒子”

每个线程都有自己独立的盒子，用于存储私有数据，确保不同线程之间的数据互不干扰

当你创建一个 `ThreadLocal` 变量时，每个访问该变量的线程都会拥有一个独立的副本

这也是 `ThreadLocal` 名称的由来。线程可以通过 `get()` 方法获取自己线程的本地副本，或通过 `set()` 方法修改该副本的值，从而避免了线程安全问题

> 举个简单的例子：
> 
> 假设有两个人去宝屋收集宝物。如果他们共用一个袋子，必然会产生争执；
> 
> 但如果每个人都有一个独立的袋子，就不会有这个问题。
> 
> 如果将这两个人比作线程，那么 `ThreadLocal` 就是用来避免这两个线程竞争同一个资源的方法

## 一、ThreadLocal的简单使用演示

```java
public class ThreadLocalExample {
    private static ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) {
        Runnable task = () -> {
            int value = threadLocal.get();
            value += 1;
            threadLocal.set(value);
            System.out.println(Thread.currentThread().getName() + " Value: " + threadLocal.get());
        };

        Thread thread1 = new Thread(task, "Thread-1");
        Thread thread2 = new Thread(task, "Thread-2");

        thread1.start(); // 输出: Thread-1 Value: 1
        thread2.start(); // 输出: Thread-2 Value: 1
    }
}
```

在上面的例子中，尽管 `threadLocal` 是一个静态变量，但由于使用了 `ThreadLocal` ， `Thread-1` 和 `Thread-2` 各自对其副本进行操作， **互不影响** ，最终都输出了 `1` 

---

## 二、原理简析

要理解 `ThreadLocal` 的原理，我们需要从 `Thread` 类的源代码入手

```java
public class Thread implements Runnable {
    //......
    //与此线程有关的ThreadLocal值。由ThreadLocal类维护
    ThreadLocal.ThreadLocalMap threadLocals = null;

    //与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    //......
}
```

从 `Thread` 类源代码可以看出， `Thread` 类中包含两个 `ThreadLocalMap` 类型的变量： `threadLocals` 和 `inheritableThreadLocals` 。默认情况下，这两个变量都是 `null` 

**核心原理在于：每个 `Thread` 对象都维护着一个 `ThreadLocalMap` 实例** 

当我们调用 `ThreadLocal` 类的 `set()` 或 `get()` 方法时，实际上是操作当前线程的 `ThreadLocalMap` 

以 `ThreadLocal` 的 `set()` 方法为例：

```java
public void set(T value) {
    // 1. 获取当前请求的线程
    Thread t = Thread.currentThread();
    // 2. 取出当前线程内部的 threadLocals 变量(哈希表结构)
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 3. 将需要存储的值放入到这个哈希表中
        map.set(this, value); // 这里的 'this' 指的是当前的 ThreadLocal 对象
    else
        // 4. 如果 map 为 null，则创建并初始化
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

从上述代码流程可以清晰地看出：

1.  `ThreadLocal` 的 `set()` 方法首先通过 `Thread.currentThread()` 获取到当前执行的线程对象

2. 然后，它尝试从这个线程对象中获取其内部的 `threadLocals` 字段，这个字段是一个 `ThreadLocalMap` 实例

3. 如果 `threadLocals` 已经存在，则将当前 `ThreadLocal` 对象作为键（Key），传入的 `value` 作为值（Value），存入到这个 `ThreadLocalMap` 中

4. 如果 `threadLocals` 为 `null` （首次调用 `set` 或 `get` 时），则会创建一个新的 `ThreadLocalMap` 并赋值给 `t.threadLocals` ，然后将键值对存入

因此，最终的变量是存储在当前线程的 `ThreadLocalMap` 中，而不是直接存储在 `ThreadLocal` 对象本身。 `ThreadLocal` 对象可以理解为是 `ThreadLocalMap` 的一个“入口”或“封装”，它提供了 `get()` 和 `set()` 方法来间接操作对应线程的 `ThreadLocalMap` 

---

## 三、数据结构概览

每个 `Thread` 中都具备一个 `ThreadLocalMap` ，而 `ThreadLocalMap` 可以存储以 `ThreadLocal` 对象为键（Key），以任意 `Object` 对象为值（Value）的键值对

`ThreadLocalMap` 是 `ThreadLocal` 的一个静态内部类，它的构造方法如下：

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //......
}
```

这意味着，即使在同一个线程中声明了多个 `ThreadLocal` 对象，例如：

```java
private static ThreadLocal<Integer> threadLocal1 = new ThreadLocal<>();
private static ThreadLocal<String> threadLocal2 = new ThreadLocal<>();
```

这两个 `ThreadLocal` 对象在同一个线程中调用 `set` 方法时，它们都会将数据存入该线程唯一的一个 `ThreadLocalMap` 中。 `threadLocal1` 会作为键存储 `Integer` 类型的值， `threadLocal2` 会作为键存储 `String` 类型的值

**总结来说， `ThreadLocal` 的原理图可以这样表示：** 

```
+-----------------+      +---------------------+
|     Thread A    |      |     Thread B        |
| +-------------+ |      | +-------------+     |
| | threadLocals| |      | | threadLocals|     |
| | (ThreadLocalMap) |<---+---| (ThreadLocalMap) |
| |   +---------+ |      |   +-----------+      |
| |   | Key:TL1 | |      |   | Key:TL1   |      |
| |   | Val:ValA1 | |    |   | Val:ValB1 |      |
| |   +---------+ |      |   +-----------+      |
| |   | Key:TL2 | |      |   | Key:TL2   |      |
| |   | Val:ValA2 | |    |   | Val:ValB2 |      |
| |   +---------+ |      |   +-----------+      |
| +-------------+ |      | +-------------+      |
+-----------------+      +----------------------+
        ^                        ^
        |                        |
        +------------------------+
                ThreadLocal (TL1, TL2)
```

图中 `TL1` 和 `TL2` 代表不同的 `ThreadLocal` 实例。每个线程（ `Thread A` 和 `Thread B` ）都有自己独立的 `ThreadLocalMap` ，这个 `Map` 以 `ThreadLocal` 实例作为键，存储着该线程对应的私有值。通过这种机制， `ThreadLocal` 实现了线程之间数据的隔离