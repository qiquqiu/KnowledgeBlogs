# 【多线程篇17】：源码剖析ThreadLocal内存泄露

> 原创 于 2025-07-19 08:44:06 发布 · 公开 · 759 阅读 · 15 · 19 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149459610

**文章目录**

[TOC]



## 引言

在Java后端开发面试中， `ThreadLocal` 几乎是一个绕不开的话题。它为解决多线程场景下的变量共享问题提供了一种优雅的思路。然而，伴随其强大功能而来的，是严重的“内存泄漏”问题

很多开发者知道“用完 `ThreadLocal` 要调用 `remove()` 方法”，但如果面试官追问“为什么？”，并要求结合源码解释其内部机制，很多人便会语塞。本文从源码层面出发，彻底剖析 `ThreadLocal` 内存泄漏的本质

---

## 一、 核心前置知识：强引用与弱引用

在解剖 `ThreadLocal` 之前，我们必须精准理解Java的两种引用类型，这是看懂其源码设计的钥匙

-  **强引用** 
  这是我们日常编码中最常见的引用类型

  ```java
  Object obj = new Object(); // obj 就是一个强引用
  ```

   **特点** ：只要一个对象存在任何强引用指向它，垃圾回收器（GC）就 **绝对不会** 回收它，哪怕系统内存耗尽抛出 `OutOfMemoryError` 

-  **弱引用** 
  它需要通过 `java.lang.ref.WeakReference` 类来创建

  ```java
  Object obj = new Object();
  WeakReference<Object> weakObj = new WeakReference<>(obj);
  ```

   **特点** ：弱引用关联的对象生命周期极短。 **无论内存是否充足，只要GC运行，一旦发现某个对象只被弱引用指向，就会立即回收它** 。回收后，通过弱引用对象的 `get()` 方法将返回 `null` 

---

## 二、 核心机制： `ThreadLocal` 的独特内部结构

要理解内存泄漏，首先要看数据到底存在了哪里

每个 `Thread` 对象内部都有一个成员变量 `threadLocals` ，它的类型是 `ThreadLocal.ThreadLocalMap` 

```java
// Thread.java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

这个 `ThreadLocalMap` 是一个由 `ThreadLocal` 内部维护的定制化 `Map` 。当调用 `threadLocal.set(value)` 时，数据实际上被存储在 **当前线程** 的 `threadLocals` 这个 `Map` 中

而问题的核心，就在于 `ThreadLocalMap` 内部的存储单元—— `Entry` 类的设计

```java
// ThreadLocal.java -> ThreadLocalMap.java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k); // 调用父类 WeakReference 的构造函数
        value = v;
    }
}
```

从这段源码中，我们可以得出两个决定性的结论：

1.  **`Entry` 的 Key 是弱引用** ： `Entry` 继承了 `WeakReference<ThreadLocal<?>>` ，并且在构造函数中通过 `super(k)` 将 `ThreadLocal` 实例本身（也就是Key）作为弱引用的目标

2.  **`Entry` 的 Value 是强引用** ： `Object value;` 这是一个普通的成员变量，它强引用着我们所存储的值（Value）

---

## 三、 内存泄漏全过程剖析

现在，我们来模拟一遍内存泄漏的完整发生过程

**场景** ：在一个线程池的工作线程中，执行了一个方法，该方法内使用了 `ThreadLocal` 但未调用 `remove()` 

```java
public void processTask() {
    ThreadLocal<MyBigObject> local = new ThreadLocal<>();
    local.set(new MyBigObject());
    // ... 业务逻辑执行 ...
} // 方法执行结束，任务完成，但线程回到线程池等待下一个任务
```

1.  **引用建立** ： `local.set(...)` 执行时，会在当前线程的 `ThreadLocalMap` 中创建一个 `Entry` 。此时的引用关系如上图所示：

   -  `local` 变量（栈上）强引用 `ThreadLocal` 实例

   -  `Entry` 的Key（弱引用）指向 `ThreadLocal` 实例

   -  `Entry` 的Value（强引用）指向 `MyBigObject` 实例

   -  `ThreadLocalMap` 的 `table` 数组强引用 `Entry` 实例

   - 当前 `Thread` 对象强引用 `ThreadLocalMap` 实例。

2.  **外部强引用消失** ： `processTask()` 方法执行完毕，栈帧销毁。局部变量 `local` 被回收。此时， **再也没有任何强引用指向 `ThreadLocal` 实例了** 

3.  **GC 执行** ：下一次GC发生时，垃圾回收器扫描到 `ThreadLocal` 实例只有一个来自 `Entry` 的弱引用。根据弱引用的规则， **这个 `ThreadLocal` 实例被回收** 

4.  **内存泄漏形成** ： `ThreadLocal` 实例被回收后， `Entry` 内部的弱引用会自动变为 `null` 。此时， `ThreadLocalMap` 中就出现了一个 **Key为 `null` ，但Value依然强引用着 `MyBigObject` 实例的 `Entry`** 

   由于该线程是线程池中的工作线程，它不会死亡，会一直存活。因此，这条强引用链将一直存在：
    **`Thread` -> `ThreadLocalMap` -> `Entry[]` -> `Entry(key=null, value=MyBigObject)` -> `MyBigObject`** 

   这个 `MyBigObject` 实例再也无法通过任何有效途径被访问到，但它却实实在在地占着内存无法被回收。这就是内存泄漏的本质

---

## 四、 源码佐证： `set` 方法剖析

`ThreadLocal` 的设计者显然意识到了这个问题，并在 `set` , `get` , `remove` 等方法中加入了一些“被动”的清理机制。我们以 `set` 方法为例：

```java
// ThreadLocalMap.java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        // ... (省略部分代码) ...

        // 如果在遍历过程中发现一个key为null的"脏"Entry
        if (e.refersTo(null)) {
            // 就尝试替换它，并清理附近的其它"脏"Entry
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    // ... (省略部分代码) ...
}
```

代码中的 `e.refersTo(null)` 就是在检查 `Entry` 的Key是否已经被回收。如果发现了这种“脏” `Entry` ，它会尝试进行一次清理。但这并不可靠，因为它依赖于你是否会再次操作这个 `Map` 的相同哈希槽位，不具有普适性

---

## 五、 最佳实践：如何优雅地避免泄漏

既然被动清理不可靠，我们就必须采取主动措施

**最佳且唯一的推荐做法是：在使用完 `ThreadLocal` 后，务必在 `finally` 块中调用 `remove()` 方法** 

```java
ThreadLocal<MyBigObject> local = ...;
try {
    local.set(new MyBigObject());
    // ... 你的业务逻辑 ...
} finally {
    local.remove(); // 确保无论是否发生异常，都能执行清理
}
```

`remove()` 方法会根据当前 `ThreadLocal` 实例作为Key，精准地找到 `ThreadLocalMap` 中对应的 `Entry` ，并将其从 `table` 数组中移除。一旦 `Entry` 被移除，它对 `Value` 的强引用就断开了， `Value` 对象在下一次GC时就能被正常回收

---

## 总结

回到问题的起点，用最精炼的语言总结 `ThreadLocal` 内存泄漏的本质

1.  **根源** ： `ThreadLocalMap` 的 `Entry` 设计中，Key（ `ThreadLocal` 实例）是 **弱引用** ，而 Value 是 **强引用** 

2.  **诱因** ：当 `ThreadLocal` 实例在外部的强引用被回收后， `Entry` 的 Key 会在GC后变为 `null` 

3.  **结果** ：由于 `Entry` 及其 Value 依然被强引用链（ `Thread` -> `ThreadLocalMap` -> `Entry` -> `Value` ）持有着，导致 Value 无法被回收，形成内存泄漏

4.  **条件** ：此问题在 **线程池** 等长生命周期的线程中尤为突出

5.  **解法** ：必须手动调用 `remove()` 方法，在 `finally` 块中执行，以确保万无一失

