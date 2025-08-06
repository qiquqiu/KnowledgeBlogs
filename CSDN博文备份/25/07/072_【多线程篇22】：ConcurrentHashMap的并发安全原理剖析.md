# 【多线程篇22】：ConcurrentHashMap的并发安全原理剖析

> 原创 于 2025-07-22 12:15:23 发布 · 公开 · 1.1k 阅读 · 32 · 18 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149532881

**文章目录**

[TOC]


在 Java 面试和日常开发中， `HashMap` 和 `ConcurrentHashMap` (以下简称 CHM) 是我们绕不开的两个核心类。我们都知道它们的关键区别是“线程安全”： `HashMap` 是非线程安全的，而 CHM 是线程安全的。

那么，这种“安全”是如何实现的？CHM到底比 `HashMap` 高明在哪里？为什么 JDK 1.8 的 CHM 性能远超早期版本？

本文将带你深入 JDK 1.8+ 的源码，彻底搞懂 CHM 并发安全的底层奥秘。

## 一、HashMap的“不安全”：问题的根源

在分析 CHM 的精妙设计之前，我们必须先理解 `HashMap` 在并发环境下为什么会“翻车”。

### 1. 数据结构回顾 (JDK 1.8)

`HashMap` 的底层是 **“数组 + 链表 / 红黑树”** 。

```java
// HashMap.java
transient Node<K,V>[] table;
```

`table` 是一个 `Node` 数组。当多个元素的 `hash` 值冲突时，它们会以链表的形式存放在同一个数组索引（桶）下；当链表长度超过 `TREEIFY_THRESHOLD` (默认为8) 且数组长度大于 64 时，链表会转化为红黑树以提高查询效率。

### 2. 并发下的致命缺陷： `put` 操作

`HashMap` 的所有操作都没有任何同步措施。我们以 `putVal` 方法为例，看看在并发下会发生什么。

```java
// HashMap.java -> putVal() 核心逻辑简化
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 1. 检查 table 是否为空，如果为空则初始化 (resize)
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 2. 计算索引 i，检查该位置是否为空
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 3. 如果为空，直接创建一个新节点放在这里
        tab[i] = newNode(hash, key, value, null);
    else {
        // ... 省略桶内已有元素时的处理逻辑
    }
    // ...
    return null;
}
```

**经典的“数据覆盖”竞争条件 (Race Condition):** 

假设两个线程 T1 和 T2 同时向一个 **空桶** `tab[i]` 中 `put` 数据。

1.  **T1** 执行到第 2 步， `p = tab[i]` ，发现 `p` 是 `null` 。

2. 此时发生线程切换， **T2** 开始执行。

3.  **T2** 也执行到第 2 步， `p = tab[i]` ，发现 `p` 同样是 `null` 。

4.  **T2** 继续执行第 3 步，成功将自己的新节点放入 `tab[i]` 。

5. 线程切换回 **T1** 。由于 T1 之前已经判断过 `p` 是 `null` ，它不会再检查一次，而是直接执行第 3 步，将自己的新节点放入 `tab[i]` 。

6.  **结果** ：T1 的数据 **覆盖** 了 T2 的数据，导致 T2 的写入操作 **丢失** 。

除了数据覆盖，并发下的 `resize()` 操作在 JDK 1.7 中甚至会因头插法导致链表形成 **环形结构** ，造成 `get` 操作时死循环。虽然 JDK 1.8 改为尾插法修复了此问题，但数据丢失等并发问题依然存在。

**结论** ： `HashMap` 的不安全，源于其所有操作都是“非原子性”的，且缺乏内存可见性保障。

---

## 二、ConcurrentHashMap 的安全之道 (JDK 1.8+)

JDK 1.8 对 CHM 进行了革命性的重构，摒弃了 JDK 1.7 的分段锁（ `Segment` ）机制，采用了更为精细的 **`CAS` + `synchronized` + `volatile`** 方案，实现了更高的并发性能。

### 1. 核心数据结构

```java
// ConcurrentHashMap.java
transient volatile Node<K,V>[] table;

// Node 定义 (部分)
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;       // value 是 volatile 的
    volatile Node<K,V> next; // next 指针是 volatile 的
}
```

**关键点** ：

-  `table` 数组被 `volatile` 修饰：保证其可见性，当一个线程对 `table` 进行扩容或修改后，其他线程能立刻看到。

-  `Node` 的 `val` 和 `next` 指针被 `volatile` 修饰：保证了节点值或链表结构的修改对其他线程的可见性。这是无锁读（ `get` 操作）的关键基础。

### 2. 安全的 `put` 操作：分场景精细化加锁

CHM 的 `put` 方法是其并发设计的精髓所在。它通过 `CAS` 和 `synchronized` 的配合，将 **锁的粒度** 降到了最低。

我们来分析其核心方法 `putVal` 的源码：

```java
// ConcurrentHashMap.java -> putVal() 核心逻辑简化
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ... key/value 不允许为 null
    int hash = spread(key.hashCode());
    int binCount = 0;
    
    // (1) 无限循环，直到成功插入或找到已有key
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        
        // (2) 场景一：table 未初始化，CAS 初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable(); // CAS操作，只有一个线程能成功
        
        // (3) 场景二：目标桶为空，CAS 插入节点 (无锁)
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break; // CAS 成功，直接跳出循环
        }
        
        // (4) 场景三：检测到正在扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f); // 帮助其他线程一起扩容
            
        // (5) 场景四：目标桶有值，锁住头节点
        else {
            V oldVal = null;
            synchronized (f) { // f 是桶的头节点
                if (tabAt(tab, i) == f) { // 再次确认头节点没变
                    // 在同步块内，遍历链表或红黑树
                    if (fh >= 0) {
                        // ... 链表遍历逻辑 ...
                    } else if (f instanceof TreeBin) {
                        // ... 红黑树处理逻辑 ...
                    }
                }
            }
            // ...
            break;
        }
    }
    addCount(1L, binCount); // 更新 size
    return null;
}
```

**剖析其安全机制：** 

-  **无锁读（ `get` 操作）** ：得益于 `table` 、 `Node.val` 、 `Node.next` 都是 `volatile` 的， `get` 操作不需要加任何锁。 `volatile` 保证了读线程总能看到其他写线程对数据的最新修改，从而实现了高效的无锁读取。

-  **`put` 操作的安全性分析** ：

  1.  **初始化安全 (场景二)** ： `initTable()` 方法内部使用 `CAS` 操作来设置 `sizeCtl` 状态位，保证了在多线程同时初始化时，只有一个线程能成功执行，其他线程则会 `yield` 等待。

  2.  **空桶插入安全 (场景三)** ：当目标桶为空时，CHM 并未直接加锁，而是乐观地尝试使用 `CAS` ( `casTabAt` ) 操作将新节点放入。

     -  `casTabAt` 是一个原子操作，它会比较 `tab[i]` 的内存值是否为 `null` ，如果是，就将其更新为新节点。

     - 如果 `CAS` 成功，说明没有竞争，操作完成。

     - 如果失败，说明在它操作的瞬间，有另一个线程捷足先登了。此时 `for` 循环会继续，重新读取 `tab[i]` 的新值，进入下一个场景（通常是场景五）。

  3.  **非空桶插入安全 (场景五)** ：这是最关键的部分。当桶中已有节点时，不能再用 `CAS` 了（因为需要遍历链表）。此时，CHM 采用 `synchronized` 关键字，但它锁住的 **不是整个 `table`** ，而是这个 **桶的头节点 `f`** 。

     -  **锁粒度极小** ：只锁住当前要操作的桶，其他桶的操作完全不受影响，并发度极高。

     -  **为什么不锁 `String` 或 `Integer`** ： `synchronized` 锁的是对象。如果锁 `key` ，当 `key` 是常量池中的字符串时，可能会锁住一个全局共享的对象，造成意想不到的死锁。锁住桶的头节点 `Node` 对象是最安全和高效的选择。

  4.  **扩容安全 (场景四)** ：当一个线程在 `put` 时发现桶的头节点 `hash` 值为 `MOVED` ，表示 `table` 正在扩容。此时它不会等待，而是调用 `helpTransfer` 加入到扩容大军中，帮助移动数据，充分利用了 CPU 资源，实现并发扩容。

### 3. 安全的 `size()` 计算：并发计数

`HashMap` 的 `size` 只是一个简单的成员变量 `int size` 。多线程下 `++size` 是非原子操作，会导致计数不准。

`ConcurrentHashMap` 的 `size()` 实现非常巧妙，它避免了全局锁，以分散计数的方式实现。

-  **`baseCount`** ：一个基础计数值，在没有并发竞争时，优先通过 `CAS` 更新这个值。

-  **`CounterCell[]`** ：一个计数器单元数组。当 `CAS` 更新 `baseCount` 失败（说明存在竞争）时，线程会找到 `CounterCell` 数组中的一个槽位，对这个槽位里的计数值进行更新。

-  **`size()` 计算** ：最终的大小是 `baseCount` 加上所有 `CounterCell` 数组中计数的总和。

这种 **条带化计数（Striped Counter）** 的思想，将对单一 `size` 变量的竞争压力分散到多个 `CounterCell` 上，极大地提高了高并发下的计数性能。

---

## 三、表格总结

| 特性 | HashMap (JDK 1.8+) | ConcurrentHashMap (JDK 1.8+) |
|:---|:---|:---|
|  **线程安全**  | 否 |  **是**  |
|  **核心数据结构**  |  `Node[] table`  |  `volatile Node<K,V>[] table`  |
|  **加锁机制**  | 无锁 |  **`CAS` + `synchronized` (锁桶头节点)**  |
|  **`put` 操作**  | 非原子性，并发下有数据覆盖风险 | 1. 空桶： `CAS` 无锁插入
2. 非空桶： `synchronized` 锁住桶头节点 |
|  **`get` 操作**  | 非线程安全 |  **无锁** ，通过 `volatile` 保证可见性 |
|  **`size()` 计算**  |  `int size` 成员变量，非线程安全 |  **`baseCount` + `CounterCell[]`** ，并发安全高效 |
|  **扩容机制**  | 单线程扩容，并发下危险 |  **并发扩容** ，多线程可协同完成 |
|  **Key/Value**  | 可为 `null`  |  **均不可为 `null`** (避免二义性) |
|  **性能**  | 单线程下极高 | 高并发下性能优异，单线程下因 `volatile` 和 `CAS` 略逊于 `HashMap`  |


**结论：** 

- 用 `volatile` 作为 **基石** ，保证了多线程间的内存可见性，是无锁读和后续操作的基础。

- 用 `CAS` 作为 **先锋** ，处理无竞争或低竞争场景（如初始化、空桶插入），避免了不必要的加锁开销。

- 用 `synchronized` 锁住桶头节点作为 **主力** ，在必须同步的场景下，将锁的粒度降到最低，实现了极高的并发度。

- 用并发扩容和并发计数等 **辅助策略** ，将并发思想贯彻到每一个细节，榨干硬件性能。

