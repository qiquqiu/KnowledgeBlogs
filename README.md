# Java 后端技术核心博客知识库 🚀

<p align="center">
  <img src="https://img.shields.io/badge/Java-17+-orange?style=for-the-badge&logo=openjdk" alt="Java">
  <img src="https://img.shields.io/badge/Spring_Boot-3.x-green?style=for-the-badge&logo=spring" alt="Spring Boot">
  <img src="https://img.shields.io/badge/MySQL-8.0-blue?style=for-the-badge&logo=mysql" alt="MySQL">
  <img src="https://img.shields.io/badge/Redis-7.x-red?style=for-the-badge&logo=redis" alt="Redis">
  <img src="https://img.shields.io/badge/Articles-100+-brightgreen?style=for-the-badge&logo=read-the-docs" alt="Articles">
  <img src="https://img.shields.io/badge/CSDN%20Views-11W%2B-blueviolet?style=for-the-badge&logo=csdn" alt="CSDN Views">
</p>

## 📖 项目简介

欢迎来到我的个人技术知识库。

本项目是我在学习和实践Java后端开发学习过程中的知识沉淀与总结，共包含 **一百多篇** 原创技术文章。内容覆盖了从 **Java 核心**、**JVM 底层**，到 **Spring 全家桶**、**数据库**、**缓存**，再到 **数据结构与算法**、**计算机基础知识** 以及 **AI框架应用**
等多个方面。

创建这个知识库的目的是为了：

1. **系统化梳理**：将零散的知识点整理成体系，构建完整的技术知识图谱。
2. **深度化思考**：对核心原理进行深入剖析，不仅仅停留在“会用”的层面。
3. **分享与交流**：记录学习过程中的思考与踩坑经验，希望能对他人有所帮助。

推荐人群：
面试找工作、需要背记理解八股的，喜欢知识分享探讨等

## ✨ 核心主题概览

<p align="left">
  <strong>编程语言 & 平台:</strong>
  <br/>
  <code>&nbsp;Java Core&nbsp;</code>
  <code>&nbsp;JVM&nbsp;</code>
  <code>&nbsp;Concurrency&nbsp;</code>
  <code>&nbsp;Collections&nbsp;</code>
</p>
<p align="left">
  <strong>框架 & 生态:</strong>
  <br/>
  <code>&nbsp;Spring Framework&nbsp;</code>
  <code>&nbsp;Spring Boot&nbsp;</code>
  <code>&nbsp;Spring AI&nbsp;</code>
  <code>&nbsp;MyBatis&nbsp;</code>
  <code>&nbsp;LangChain4j&nbsp;</code>
</p>
<p align="left">
  <strong>数据库 & 缓存:</strong>
  <br/>
  <code>&nbsp;MySQL&nbsp;</code>
  <code>&nbsp;Redis&nbsp;</code>
  <code>&nbsp;ACID&nbsp;</code>
  <code>&nbsp;MVCC&nbsp;</code>
  <code>&nbsp;Distributed Lock&nbsp;</code>
</p>
<p align="left">
  <strong>数据结构 & 算法:</strong>
  <br/>
  <code>&nbsp;LeetCode&nbsp;</code>
  <code>&nbsp;Sliding Window&nbsp;</code>
  <code>&nbsp;Two Pointers&nbsp;</code>
  <code>&nbsp;Tree&nbsp;</code>
  <code>&nbsp;BFS/DFS&nbsp;</code>
</p>
<p align="left">
  <strong>计算机专业知识:</strong>
  <br/>
  <code>&nbsp;Computer Organization&nbsp;</code>
  <code>&nbsp;Operating Systems&nbsp;</code>
  <code>&nbsp;Networking&nbsp;</code>
  <code>&nbsp;Principles of Compilation&nbsp;</code>
</p>


---

## 📚 文章专栏目录

以下是根据专栏分类整理的文章目录，你可以根据感兴趣的方向进行查阅。

### ☕ JDK 核心

这是整个知识库的基石，深入探讨了Java语言的底层实现和核心概念。

- **多线程与并发编程**：`JMM`、`volatile`、`synchronized`、`ReentrantLock`、`AQS` 核心原理、`ThreadLocal`、`线程池`、`CAS` 等。
- **JVM 深入剖析**：内存区域划分（`JMM`）、垃圾回收机制（`GC`）、类加载机制、性能调优等。
- **集合框架源码**：`HashMap` 扩容与树化、`ArrayList` 与 `LinkedList` 对比、`ConcurrentHashMap` 并发安全原理等。
- **Java 基础**：`final` 关键字、`equals` & `hashCode`、`String` 不可变性等。

### 🌱 Spring 框架

全面覆盖了 Spring 生态的核心组件和原理。

- **核心思想**：`IoC` 容器、`AOP` 实现原理。
- **Bean 生命周期**：从实例化到销毁的全过程，`BeanPostProcessor` 的作用。
- **事务管理**：`@Transactional` 注解的生效与失效场景。
- **Spring Boot**：自动装配原理（`@EnableAutoConfiguration`）、`starter` 制作流程。

### 🗄️ MySQL 篇

从原理到实践，系统讲解了MySQL数据库的核心知识。

- **索引**：`B+树` 索引原理、覆盖索引、最左前缀原则。
- **事务与锁**：`ACID` 特性、`MVCC` 多版本并发控制、`undo/redo log`、行锁与表锁。
- **架构**：`binlog` 与主从复制、分库分表策略。
- **SQL 优化**：执行计划分析、慢查询定位。

### 🚀 Redis 篇

涵盖了 Redis 作为高性能缓存和中间件的常用技术点。

- **持久化机制**：`RDB` 与 `AOF` 的对比和选择。
- **高可用架构**：哨兵模式（`Sentinel`）与集群（`Cluster`）。
- **缓存核心问题**：缓存穿透、击穿、雪崩及其解决方案。
- **分布式锁**：`SETNX` 与 `Redisson` 实现方案对比。
- **数据同步**：Redis 与 MySQL 的数据一致性保证方案。

### 🧠 数据结构与算法刷题

本部分整合了 **“数据结构与算法刷题”** 和 **“滑动窗口与双指针”** 两个专栏，以 LeetCode 题目为载体，锻炼编程思维。

- **核心思想/技巧**：
    - **滑动窗口**：定长/不定长窗口问题模板。
    - **双指针**：快慢指针、左右指针、链表操作。
    - **树与图**：二叉树的遍历（前/中/后/层序）、深度优先（`DFS`）、广度优先（`BFS`）、平衡二叉树判断。
- **典型问题**：数组、链表、二叉树、字符串等相关的高频面试题。

### 🤖 AI框架应用

本部分整合了 **“SpringAI”** 和 **“langchain4j”** 两个专栏，探索 Java 与大语言模型（LLM）的结合。

- **核心功能**：实现连续对话、上下文记忆管理、流式响应。
- **框架集成**：与 `Spring Boot` 快速集成，调用大模型 API。
- **应用实践**：构建简单的聊天机器人应用。

### 🖥️ 计算机专业知识

讲解了后端开发必备的计算机底层知识。

- **计算机组成原理**：`主存与Cache的地址映射` 等。
- **编译原理(TODO)**：词法分析、语法分析、NFA、DFA 等。
- **操作系统(TODO)**：进程与线程、内存管理、I/O 模型等。
- **计算机网络(TODO)**：`TCP/IP` 协议栈、`HTTP` 协议等。

### 🔩 其他专栏

- **Mybatis框架**：探讨了 MyBatis 的核心执行流程和四大组件。
- **项目解决方案**：记录了具体的项目开发中遇到的问题和解决方案。

## ✍️ 持续更新中

- 保持一日一到两更博客

## 🚀 如何查阅

1. **按需查阅**：通过上方的目录，找到你感兴趣的专栏进行系统性学习。
2. **面试准备**：本知识库覆盖了 Java 后端面试的高频考点，可作为面试复习的参考资料。
3. **问题索引**：当你遇到具体的技术问题时（如“Redis 分布式锁如何实现？”），可以尝试在这里寻找答案和原理剖析。

## 🤝 贡献与交流

这个知识库是我个人学习路径的记录，难免有疏漏或错误之处。如果你发现任何问题，或者有更好的见解，非常欢迎通过 [Issue](https://github.com/qiquqiu/KnowledgeBlogs/issues)
或直接在 [CSDN](https://blog.csdn.net/lyh2004_08) 评论区提出。

🤓😁😋