# 【多线程篇15】：JMM之happens-before规则清晰详解

> 原创 于 2025-07-19 08:15:00 发布 · 公开 · 757 阅读 · 10 · 24 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149455339

**文章目录**

[TOC]


上文我们详解了 **[JMM 内存模型](https://blog.csdn.net/lyh2004_08/article/details/149433419)** ，JMM（Java Memory Model）基于 `happens-before` 规则来定义和保证线程间的内存可见性

现在，我们来理解 `happens-before` 原则，我们先来明确两个概念：

> 

-  **操作（Operation）** ：你可以简单理解为一行代码，或者一个单独的动作，比如 `a = 1;` ，或者 `System.out.println(b);` 

-  **线程（Thread）** ：执行这些操作的独立执行流

---

## 1. `happens-before` 原则到底是什么？

`happens-before` 原则，是 Java 内存模型（JMM）为我们提供的一套 **规则** 。这套规则的唯一目的，就是帮助我们判断在多线程环境下：

1.  **可见性 (Visibility)** ：一个线程对共享变量的修改， **是否能被** 另一个线程“看到”（即从主内存中读取到最新值）

2.  **有序性 (Ordering)** ：两个操作的执行顺序， **是否是固定和可预测的** ，尤其是在存在指令重排序的情况下

**最核心的理解** ：
`happens-before` 关系，就像一条 **单向的、不可逆转的时间线** 
如果操作 A `happens-before` 操作 B，那么就意味着：

-  **A 的结果对 B 可见** ：操作 A 对共享变量所做的任何修改，在操作 B 执行之前，都保证已经写入到主内存中，并且操作 B 在读取时能从主内存中获取到这个最新值

-  **A 在 B 之前执行** ：从 JMM 的角度看，操作 A 必须在操作 B 之前完成。编译器和处理器不能对它们进行重排序，导致 B 在 A 之前执行

**反之** ：
如果操作 A 和操作 B **没有** `happens-before` 关系，那么它们之间的可见性和有序性是 **无法保证的** 。A 的修改可能对 B 不可见，A 和 B 的执行顺序也可能被重排序

---

## 2. 为什么需要 `happens-before` 原则？

想象一下，你和你的朋友在共享一张纸条（共享变量）

-  **没有 `happens-before` 规则** ：

  - 你写了一句话，但可能你写完后，你的朋友立即去看，却没看到你写的内容（ **可见性问题** ，就像你的修改还在你的“草稿纸”上，没放到“纸条”上）

  - 你先写了“你好”，然后写了“再见”。但你的朋友可能先看到了“再见”，然后才看到“你好”（ **有序性问题** ，指令重排序）

  - 这是因为你们没有约定好“写完就放桌上”、“看之前先检查桌上是否有新内容”这样的规则

-  **有了 `happens-before` 规则** ：

  - JMM 定义了一系列“约定”（就是下面的具体规则）

  - 例如，如果你的写操作 A `happens-before` 你朋友的读操作 B，那么 JMM 保证你写完的内容一定在桌上，并且你朋友去读的时候一定能看到最新内容

  - 这样，你们就可以 **可靠地** 进行交流了

`happens-before` 原则就是 Java 为了 **简化并发编程** 而提供的 **保证** 。它让 **我们不用去关心底层** 的 CPU 缓存一致性协议、指令重排序等复杂细节，只需要理解这些简单的规则，就能编写出正确的并发程序

---

## 3. `happens-before` 原则的具体规则

> 这些规则是 JMM **天然提供的保证** 。我们不需要额外做什么，只要我们的代码符合这些情况，JMM 就会自动为我们建立 `happens-before` 关系

1.  **程序顺序规则 (Program Order Rule)** ：

   -  **描述** ：在一个线程内部，你代码里先写的操作， `happens-before` 后写的操作

   -  **例子** ：```java
     // 线程 T1
     int a = 1; // 操作 A
     int b = 2; // 操作 B
     ```

     操作 A `happens-before` 操作 B。这意味着，在线程 T1 内部， `a=1` 的结果一定对 `b=2` 可见，并且 `a=1` 逻辑上在 `b=2` 之前执行

   -  **注意** ：这只是 **单线程内部的逻辑顺序保证** ，编译器和处理器仍然可能为了优化而重排指令，但它们会保证重排后的结果与未重排时一致（ `as-if-serial` 语义）。但从 `happens-before` 角度看，关系依然成立

2.  **监视器锁规则 (Monitor Lock Rule)** ：

   -  **描述** ：对一个 `synchronized` 锁的 **解锁操作** ， `happens-before` 后续对这个锁的 **加锁操作** 

   -  **例子** ：

     ```java
       // 线程 T1
       synchronized (lock) {
             sharedVar = 10; // 操作 A：对共享变量的修改
       } // 操作 B：解锁
      
       // 线程 T2
       synchronized (lock) { // 操作 C：加锁
             System.out.println(sharedVar); // 操作 D：读取共享变量
       }
     ```

     操作 B (T1 解锁) `happens-before` 操作 C (T2 加锁)
      **因此** ：操作 A (T1 对 `sharedVar` 的修改) `happens-before` 操作 C (T2 加锁)，进而 `happens-before` 操作 D (T2 读取 `sharedVar` )
      **结果** ：T2 线程在进入 `synchronized` 块后， **一定能看到** T1 线程在退出 `synchronized` 块前对 `sharedVar` 所做的修改（即 `sharedVar` 为 10）。这是 `synchronized` 保证可见性的原理

3.  **volatile 变量规则 (Volatile Variable Rule)** ：

   -  **描述** ：对一个 `volatile` 变量的 **写操作** ， `happens-before` 后续对这个 `volatile` 变量的 **读操作** 

   -  **例子** ：

     ```java
     volatile int flag = 0;

     // 线程 T1
     flag = 1; // 操作 A：对 volatile 变量的写

     // 线程 T2
     if (flag == 1) { // 操作 B：对 volatile 变量的读
         // ...
     }
     ```

     操作 A (T1 写 `flag` ) `happens-before` 操作 B (T2 读 `flag` )
      **结果** ：当 T2 线程读取 `flag` 发现它是 1 时，它 **一定能看到** T1 线程对 `flag` 所做的修改。并且， `volatile` 变量的写操作会强制将修改刷新到主内存，读操作会强制从主内存获取最新值，从而避免了缓存导致的问题

4.  **线程启动规则 (Thread Start Rule)** ：

   -  **描述** ： `Thread.start()` 方法的调用， `happens-before` 该线程的任何操作

   -  **例子** ：```java
     int data = 10; // 操作 A
     Thread t = new Thread(() -> {
         System.out.println(data); // 操作 B
     });
     t.start(); // 操作 C
     ```

     操作 C ( `t.start()` ) `happens-before` 操作 B (新线程中的打印)
      **因此** ：操作 A ( `data=10` ) `happens-before` 操作 C，进而 `happens-before` 操作 B
      **结果** ：新线程 T 启动后， **一定能看到** 主线程在调用 `start()` 之前对 `data` 所做的修改（即 `data` 为 10）

5.  **线程终止规则 (Thread Termination Rule)** ：

   -  **描述** ：一个线程的所有操作， `happens-before` 其他线程检测到这个线程已经终止

   -  **例子** ：

     ```java
      // 线程 T1
      // ...执行一些操作，修改共享变量... // 操作 A
     // T1 线程终止
   
      // 线程 T2
      t1.join(); // 操作 B：等待 T1 终止
      // ...读取共享变量... // 操作 C
     ```

     操作 A (T1 的所有操作) `happens-before` 操作 B ( `t1.join()` )
      **因此** ：操作 A `happens-before` 操作 C
      **结果** ：T2 线程在 `join()` 方法返回后， **一定能看到** T1 线程在终止前对共享变量所做的所有修改

6.  **传递性 (Transitivity)** ：

   -  **描述** ：如果 A `happens-before` B，并且 B `happens-before` C，那么 A `happens-before` C

   -  **例子** ：
     如果 `synchronized` 解锁 (B) `happens-before` `volatile` 写 ©，且 `volatile` 写 © `happens-before` `volatile` 读 (D)，那么 `synchronized` 解锁 (B) `happens-before` `volatile` 读 (D)

---

## 4. 总结

-  **理解并发基石** ： `happens-before` 是 JMM 的核心，是理解 Java 并发编程的 **最底层、最根本的原理** 

-  **解释并发工具** ：所有 Java 并发工具（ `synchronized` , `volatile` , `Lock` , `Atomic` 类等）都是围绕 `happens-before` 原则来设计的。理解了它，你就能深入解释这些工具为什么能保证线程安全

>  `happens-before` 就是 JMM 给你的一套“承诺”，让我们知道在什么情况下，多线程操作是可预测和安全的