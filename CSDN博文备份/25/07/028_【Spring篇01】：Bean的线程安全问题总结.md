# 【Spring篇01】：Bean的线程安全问题总结

> 原创 于 2025-07-01 09:58:57 发布 · 公开 · 1.1k 阅读 · 37 · 25 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149040172

**文章目录**

[TOC]



## 1. 核心问题：Spring 框架中的 Bean 是线程安全的吗？

**核心点：** 

> 

- Spring 中的 Bean 默认是 **单例** （ `singleton` ）的。

-  **默认不线程安全：** Spring 框架本身不对单例 Bean 进行线程安全封装。

-  **线程安全取决于使用方式：** 如果 Bean 的状态是不可变的，或者不包含可变状态，则在某种程度上是线程安全的。如果 Bean 包含可变状态且多线程同时访问，就需要考虑线程安全问题。

-  **解决方案：** 改变作用域（ `prototype` ，可多例）或自行处理线程同步。

让我们用一个通俗的类比来理解：

**类比：一个公共图书馆的书籍** 

-  **Spring 容器：** 想象一下 Spring 容器是一个大型的公共图书馆。

-  **Bean：** 图书馆里的每一本书都是一个 Bean。

-  **单例 Bean：** 想象一下图书馆里只有 **一本** 特别重要的参考书（比如《Spring 官方文档》），所有想查阅这本书的人（线程）都必须共用这一本。这就是单例 Bean。

-  **多例 Bean (Prototype)：** 想象一下图书馆里有很多本同一本书（比如《Java 编程入门》），每个人都可以拿一本自己的去读。这就是多例 Bean。

-  **Bean 的状态：** 书籍的内容就是 Bean 的状态。

-  **可变状态：** 如果这本书允许你在上面做笔记、划线、修改内容，那么这本书就是具有可变状态的。

-  **不可变状态：** 如果这本书不允许任何修改，只能阅读，那么这本书就是具有不可变状态的。

-  **线程：** 来图书馆查阅书籍的每个人就是一个线程。

**现在，我们来套用类比来理解线程安全问题：** 

-  **单例 Bean (只有一本参考书):** 如果多个读者（线程）同时想要在同一本参考书（单例 Bean）上做笔记（修改状态），就会产生冲突。第一个读者写了一半，第二个读者也开始写，内容就会混乱。这就是 **线程不安全** 。

-  **多例 Bean (很多本入门书):** 如果每个读者（线程）都拿一本自己的入门书（多例 Bean），他们可以在自己的书上随意做笔记（修改状态），互不影响。这就是 **线程安全** 。

-  **不可变状态的单例 Bean (只能阅读的参考书):** 如果这本参考书不允许做任何修改，多个读者（线程）同时阅读（访问不可变状态），他们不会互相干扰。尽管是同一本书，但因为内容不可修改，所以是 **线程安全** 的。

-  **可变状态的单例 Bean (允许做笔记的参考书):** 如果这本参考书允许做笔记，多个读者（线程）同时做笔记，就会产生线程不安全问题。

---

## 2. 最佳实践与解决方案

### 禁止方案：滥用 `prototype` 作用域

-  **问题** ：每次请求创建新Bean实例，导致内存飙升、GC压力增大，违背单例设计初衷。

-  **适用场景** ：仅当Bean需要持有 **请求级状态** （如用户会话数据）时使用。

### 推荐方案（按优先级排序）

|  **方案**  |  **适用场景**  |  **实现方式**  |  **案例**  |
|:---:|:---:|:---:|:---:|
|  **无状态设计**  | 绝大多数业务逻辑 | 移除成员变量，用局部变量/参数传递数据 | Service层业务方法 |
|  **ThreadLocal**  | 线程绑定的数据（如用户身份） |  `ThreadLocal<UserContext>`  | 权限校验、数据库路由 |
|  **同步锁（synchronized）**  | 低并发场景的简单状态 |  `synchronized` 方法/代码块 | 本地计数器 |
|  **并发工具类**  | 复杂状态管理 |  `AtomicInteger` , `ConcurrentHashMap`  | 分布式ID生成、缓存 |
|  **不可变对象**  | 配置类等只需初始化的数据 | 用 `final` 修饰字段，无setter方法 | 系统参数配置Bean |


---

## 3. 生产环境中的典型案例

### Case 1：订单服务统计

```java
@Service
public class OrderService {
    // 错误！多线程下totalOrders可能少加
    private long totalOrders = 0; 

    // 正确方案：使用AtomicLong
    private final AtomicLong totalOrders = new AtomicLong(0);

    public void placeOrder() {
        // 业务逻辑...
        totalOrders.incrementAndGet();
    }
}
```

### Case 2：用户会话存储

```java
@Service
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSession {  // 使用request作用域替代单例
    private String userId;
    // getter/setter...
}

// 更优方案：ThreadLocal（避免创建过多对象）
public class UserContext {
    private static final ThreadLocal<String> USER_HOLDER = new ThreadLocal<>();
    public static void setUserId(String id) { USER_HOLDER.set(id); }
    public static String getUserId() { return USER_HOLDER.get(); }
}
```

---

## 4. 关键总结

1.  **默认规则** ：Spring单例Bean **非线程安全** ，安全与否取决于 **开发者的设计** 。

2.  **黄金准则** ：优先设计 **无状态Bean** ，必须维护状态时用 **并发工具** 或 **ThreadLocal** 。

3.  **避坑指南** ：

   - 避免在单例Bean中定义 `非final` 成员变量

   - 慎用 `prototype` 作用域

   - 同步锁范围要 **最小化** （锁方法不如锁代码块）

>  **延伸思考** ：为什么Spring MVC的 `Controller` 默认单例却安全？
>  **答** ：Controller中处理的 `HttpServletRequest` 和响应对象本质是 **每个请求独享** （由 `Tomcat` 线程池分配），与单例Controller实例无关。