# 【Mybatis篇01】：Mybatis执行流程 从配置文件到结果输出

> 原创 于 2025-07-02 11:37:38 发布 · 公开 · 559 阅读 · 17 · 17 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149066101

**文章目录**

[TOC]



## MyBatis执行流程

MyBatis 的执行流程可以概括为以下几个主要步骤：

> 

1.  **加载配置文件并构建会话工厂** 

2.  **创建会话** 

3.  **执行数据库操作** 

4.  **处理结果并返回** 

### 完整流程图示

![](./assets/035_1.svg)

---

下面我们将一步步深入了解每个环节。

### 1. 加载配置文件并构建会话工厂

一切的起点都始于 **`mybatis-config.xml`** 配置文件。这个文件包含了 MyBatis 的核心配置信息，例如：

- 数据库连接信息

- Mapper XML 文件的位置

- 类型别名（typeAliases）

- 插件（plugins）

- 环境配置（environments）

MyBatis 启动时，会加载并解析 `mybatis-config.xml` 文件，并将配置信息读取到内存中。

基于这些配置信息，MyBatis 会创建一个全局唯一的 **`SqlSessionFactory`** 对象。 `SqlSessionFactory` 的作用是创建 `SqlSession` 。你可以将 `SqlSessionFactory` 类比成一个 **生产汽车的工厂** ，它是一个重量级对象，在应用程序的整个生命周期中通常只需要一个实例，并且是线程安全的。

---

### 2. 创建会话

通过 `SqlSessionFactory` ，我们可以创建 **`SqlSession`** 对象。

`SqlSession` 代表了与数据库的一次会话。它是一个轻量级对象，包含了执行SQL语句的所有方法，例如 `selectOne` 、 `selectList` 、 `insert` 、 `update` 、 `delete` 等。每次与数据库进行交互（执行 SQL）时，都需要创建一个新的 `SqlSession` 。

可以把 `SqlSession` 类比成一次 **开车出门** ，每次出门都需要一辆车（SqlSession），用完后要记得“停车”（关闭 `SqlSession` ），以释放数据库连接资源。

---

### 3. 执行数据库操作

`SqlSession` 内部会创建一个 **`Executor`** 执行器。 `Executor` 是 MyBatis 真正执行数据库操作的核心组件。它负责：

- 根据 Mapper 接口方法或 XML Mapper 文件中的 statement ID，找到对应的 SQL 语句。

- 设置 SQL 语句的参数。

- 通过 JDBC 与数据库进行交互，执行 SQL 语句。

- 维护一级缓存（会话级别缓存）。

当 `Executor` 准备执行 SQL 语句时，它会查找并使用 **`MappedStatement`** 对象。 `MappedStatement` 对象封装了 MapperXML 文件中 `<select>` 、 `<insert>` 、 `<update>` 、 `<delete>` 等标签中的所有信息，包括：

- 要执行的 SQL 语句

- 参数类型

- 结果类型

- 缓存配置

`Executor` 利用 `MappedStatement` 对象中的信息，构建最终的 SQL 语句，并通过 JDBC 驱动将 SQL 发送到数据库执行。

---

### 4. 处理结果并返回

数据库执行完 SQL 语句后，会将结果集返回给 `Executor` 。

`Executor` 根据 `MappedStatement` 对象中定义的结果类型（例如 `resultType` 或 `resultMap` ），将数据库返回的结果集映射成 Java 对象。这些 Java 对象可以是：

- 基本数据类型（如 `Integer` 、 `String` ）

- 集合（如 `List` ）

- Map

- 自定义的 POJO 对象

最终，这些映射后的 Java 对象会作为 `SqlSession` 方法的返回值，返回给调用调用 MyBatis 的业务代码。

---

### 总结

理解 MyBatis 的执行流程，有助于我们更好地理解其内部工作原理，优化 SQL 语句，以及在开发过程中排查问题。从加载配置文件到构建会话工厂，再到创建会话、通过执行器和 `MappedStatement` 与数据库交互，最终将结果映射为 Java 对象，MyBatis 巧妙地将这些步骤串联起来，为我们提供了便捷的数据访问层解决方案。

---

**关于生产环境的最佳实践（针对 MyBatis）：** 

-  **单例的 `SqlSessionFactory` ：** 在应用程序启动时创建 `SqlSessionFactory` ，并将其作为单例对象管理。不要在每次数据库操作时都创建新的 `SqlSessionFactory` 。

-  **线程安全的 `SqlSession` 管理：** `SqlSession` 不是线程安全的，每次数据库操作都应该获取一个新的 `SqlSession` ，并在操作完成后及时关闭。在 Spring 等框架中，通常通过事务管理器来管理 `SqlSession` 的生命周期，确保线程安全和资源释放。

-  **合理使用缓存：** MyBatis 提供了一级缓存（SqlSession 级别）和二级缓存（SqlSessionFactory 级别）。理解它们的区别和适用场景，合理配置缓存可以提高查询性能。对于更新频繁的数据，谨慎使用二级缓存。

-  **SQL 语句的可读性和维护性：** 将 SQL 语句写在 XML Mapper 文件中，保持 SQL 语句的清晰和规范。使用 `<sql>` 标签提取重复的 SQL 片段。

-  **使用 ResultMap：** 对于复杂的查询结果映射，优先使用 `ResultMap` 而不是 `ResultType` 。 `ResultMap` 提供了更灵活和强大的映射能力，可以处理复杂的关联关系和字段映射。

-  **异常处理：** 合理捕获和处理数据库操作可能抛出的异常，例如 `DataAccessException` 。

-  **日志记录：** 配置 MyBatis 的日志，以便在开发和生产环境中查看执行的 SQL 语句、参数和结果，方便排查问题。

-  **集成 Spring：** 在实际项目中，通常会将 MyBatis 与 Spring 框架集成。Spring 提供了方便的事务管理和 Mapper 接口扫描等功能，简化了 MyBatis 的使用。

---

**深度拓展：** 

MyBatis 的一级缓存和二级缓存，你能简单解释一下它们的作用和区别吗？

-  **一级缓存（Local Cache）：** 也称为会话级别缓存。它是 `SqlSession` 级别的缓存。同一个 `SqlSession` 中执行相同的查询语句（SQL 和参数都相同），第一次查询会从数据库获取数据并放入一级缓存，后续相同的查询会直接从一级缓存中获取，而不再访问数据库。一级缓存的生命周期与 `SqlSession` 相同，当 `SqlSession` 关闭时，一级缓存也会被清空。

-  **二级缓存（Cache）：** 也称为全局缓存。它是 `SqlSessionFactory` 级别的缓存。同一个 `SqlSessionFactory` 下的不同 `SqlSession` 都可以共享二级缓存。二级缓存是跨 `SqlSession` 的。当一个 `SqlSession` 提交或关闭时，它查询的数据会存入二级缓存。其他 `SqlSession` 再次查询相同的数据时，可以从二级缓存中获取。二级缓存默认是关闭的，需要在 Mapper XML 文件中进行配置开启。

区别在于缓存的范围和生命周期：

-  **范围：** 一级缓存是 `SqlSession` 范围，二级缓存是 `SqlSessionFactory` 范围。

-  **生命周期：** 一级缓存随 `SqlSession` 关闭而失效，二级缓存随 `SqlSessionFactory` 关闭而失效（通常是应用程序关闭）。

一级缓存默认开启，用于提高同一个会话内的查询性能。二级缓存需要手动开启，可以跨会话共享数据，进一步减少数据库访问，但需要注意缓存一致性问题，特别是对于更新频繁的数据。
范围。

-  **生命周期：** 一级缓存随 `SqlSession` 关闭而失效，二级缓存随 `SqlSessionFactory` 关闭而失效（通常是应用程序关闭）。

一级缓存默认开启，用于提高同一个会话内的查询性能。二级缓存需要手动开启，可以跨会话共享数据，进一步减少数据库访问，但需要注意缓存一致性问题，特别是对于更新频繁的数据。