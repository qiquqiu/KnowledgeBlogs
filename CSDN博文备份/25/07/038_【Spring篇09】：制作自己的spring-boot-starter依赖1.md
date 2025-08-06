# 【Spring篇09】：制作自己的spring-boot-starter依赖1

> 原创 已于 2025-07-05 22:31:04 修改 · 公开 · 1.1k 阅读 · 35 · 22 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149095589

**文章目录**

[TOC]


在企业级应用开发中，我们经常会遇到一些通用的功能模块，比如统一的日志处理、RPC 调用客户端、特定的安全认证逻辑等如果每个项目都重复编写这些配置和代码，效率低下且容易出错

Spring Boot Starter 的出现，正是为了解决这个问题。Starter 是一种特殊的 Maven 或Gradle依赖，它能够将相关的依赖和自动化配置打包在一起，让其他项目只需要简单地引入一个 Starter 依赖，就能自动获得所需的功能，无需手动配置

本文将基于 [Spring Boot 的自动化装配原理](https://blog.csdn.net/lyh2004_08/article/details/149095382) ，手把手教你如何制作自己的 Spring Boot Starter 依赖

## 1. Spring Boot Starter 的本质

Spring Boot Starter 的本质是一个 **依赖聚合器** 和 **自动化配置提供者** 

-  **依赖聚合:** 一个 Starter 通常会引入实现某个功能所需的所有其他依赖例如， `spring-boot-starter-web` 会引入 Spring MVC、Tomcat 等依赖

-  **自动化配置:** Starter 中包含了自动化配置类，这些类会根据当前项目的环境（依赖、配置等）自动配置相关的 Bean，使得用户无需手动编写繁琐的配置

制作自己的 Starter，就是将你的通用功能封装在一个或多个模块中，并按照 Spring Boot 的规范提供自动化配置

---

## 2. Starter 的模块结构（推荐）

为了清晰起见，通常会将一个 Starter 分为两个模块：

-  **`xxx-spring-boot-starter`** : 这是提供给用户引入的模块它本身不包含实际的代码，只作为 **依赖的入口** ，主要依赖于 `xxx-spring-boot-autoconfigure` 模块

-  **`xxx-spring-boot-autoconfigure`** : 这是包含实际自动化配置逻辑的模块你所有的配置类、条件注解、以及 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件都放在这个模块中

这种分离的好处是：

-  **职责分离:** Starter 模块只负责依赖管理，autoconfigure 模块负责自动化配置逻辑

-  **清晰:** 用户引入 Starter 模块即可，无需关心内部实现

当然，对于简单的 Starter，你也可以将所有内容放在一个模块中，但这不符合最佳实践

---

## 3. 制作 `xxx-spring-boot-autoconfigure` 模块

这是实现自动化配置的核心模块

### 3.1 添加必要的依赖

首先，在 `xxx-spring-boot-autoconfigure` 模块的 `pom.xml` (Maven) 或 `build.gradle` (Gradle) 中添加 Spring Boot 的自动化配置依赖：

**Maven:** 

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
<!-- 如果你的自动化配置依赖于某个第三方库，也需要在这里添加 -->
<!-- <dependency>
    <groupId>com.example</groupId>
    <artifactId>some-library</artifactId>
</dependency> -->
```

**Gradle:** 

```gradle
implementation 'org.springframework.boot:spring-boot-autoconfigure'
// 如果你的自动化配置依赖于某个第三方库，也需要在这里添加
// implementation 'com.example:some-library'
```

`spring-boot-autoconfigure` 模块包含了 `@ConditionalOn...` 等条件注解，以及 Spring Boot 自动化配置所需的基础设施

### 3.2 编写具体功能的配置类

创建包含你通用功能具体Bean 定义的配置类这些类通常使用 `@Configuration` 和 `@Bean` 注解

```java
// src/main/java/com/example/starter/autoconfigure/MyServiceConfig.java
@Configuration
public class MyServiceConfig {

    // 定义一个通用的服务 Bean
    @Bean
    public MyService myService() {
        return new MyService();
    }

    // 如果你的服务需要配置属性，可以使用 @ConfigurationProperties
    @Bean
    @ConfigurationProperties(prefix = "my.service")
    public MyServiceProperties myServiceProperties() {
        return new MyServiceProperties();
    }
}
```

### 3.3 编写自动化配置类 ( `@AutoConfiguration` )

创建标记为 `@AutoConfiguration` 的自动化配置类这个类将作为自动化配置的入口，并使用相应的条件注解来控制何时生效

```java
// src/main/java/com/example/starter/autoconfigure/MyServiceAutoConfiguration.java
@AutoConfiguration // 标记为自动化配置类 (Spring Boot 2.7+)
// @Configuration // 兼容早期版本，但推荐使用 @AutoConfiguration
@ConditionalOnClass(MyService.class) // 条件：只有当 MyService.class 存在时才生效
@EnableConfigurationProperties(MyServiceProperties.class) // 启用配置属性绑定
@Import(MyServiceConfig.class) // 导入包含具体 Bean 定义的配置类
public class MyServiceAutoConfiguration {

    // 这个类本身可以不定义 Bean，主要通过 @Import 导入其他配置类
    // 也可以在这里直接定义 Bean，并使用条件注解
    // @Bean
    // @ConditionalOnMissingBean // 如果没有 MyService 的 Bean，则创建默认的
    // public MyService defaultMyService() {
    //     return new MyService();
    // }
}
```

**重要提示：** 

- 使用 `@AutoConfiguration` 注解（Spring Boot 2.7+）

- 使用 `@ConditionalOnClass` 等条件注解来控制自动化配置的生效条件通常会判断某个核心类是否存在，或者某个配置属性是否设置

- 使用 `@Import` 注解导入包含具体 Bean 定义的配置类

- 如果你的功能需要配置属性，使用 `@EnableConfigurationProperties` 和 `@ConfigurationProperties` 来绑定属性

### 3.4 注册自动化配置类 ( `.imports` 或 `spring.factories` )

这是最关键的一步，让 Spring Boot 能够找到你的自动化配置类

**推荐方式 (Spring Boot 2.7+):** 

在 `xxx-spring-boot-autoconfigure` 模块的 `src/main/resources/META-INF/spring/` 目录下创建 `org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件

在该文件中，每行填写一个你的自动化配置类的 **全限定名** ：

```properties
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.starter.autoconfigure.MyServiceAutoConfiguration
```

**兼容早期版本方式:** 

在 `xxx-spring-boot-autoconfigure` 模块的 `src/main/resources/META-INF/` 目录下创建 `spring.factories` 文件

在该文件中，添加如下内容：

```properties
# src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.starter.autoconfigure.MyServiceAutoConfiguration
```

**在 Spring Boot 2.7+ 中，你可以同时使用这两种方式，但推荐优先使用 `.imports` 文件** 

---

## 4. 制作 `xxx-spring-boot-starter` 模块

这个模块非常简单，它的主要作用就是依赖 `xxx-spring-boot-autoconfigure` 模块，并遵循 Starter 的命名规范

在 `xxx-spring-boot-starter` 模块的 `pom.xml` (Maven) 或 `build.gradle` (Gradle) 中添加依赖：

**Maven:** 

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-service-spring-boot-autoconfigure</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

**Gradle:** 

```gradle
implementation 'com.example:my-service-spring-boot-autoconfigure:1.0.0-SNAPSHOT'
```

这个模块通常不需要编写任何 Java 代码

---

## 5. 使用你的 Starter 依赖

现在，其他 Spring Boot 项目只需要在 `pom.xml` 或 `build.gradle` 中引入你的 Starter 依赖即可：

**Maven:** 

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-service-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

**Gradle:** 

```gradle
implementation 'com.example:my-service-spring-boot-starter:1.0.0-SNAPSHOT'
```

当用户引入这个依赖后，你的 `xxx-spring-boot-autoconfigure` 模块就会被添加到项目的运行时类路径中。Spring Boot 启动时， `AutoConfigurationImportSelector` 会找到并加载 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件（或 `spring.factories` ），发现并处理你的 `MyServiceAutoConfiguration` 如果条件满足， `MyServiceAutoConfiguration` 中导入的 `MyServiceConfig` 生效， `MyService` Bean 就会被自动创建并注册到 Spring 容器中

用户可以直接在他们的代码中注入并使用 `MyService` ：

```java
@Service
public class SomeBusinessLogic {

    @Autowired
    private MyService myService; // 直接注入，无需手动配置

    // ... 使用 myService
}
```

---

## 6. 最佳实践和注意事项

-  **命名规范:** 遵循 Spring Boot Starter 的命名规范 `xxx-spring-boot-starter` 

-  **版本管理:** 合理管理你的 Starter 版本

-  **文档:** 为你的 Starter 提供清晰的文档，说明它的功能、如何使用、支持哪些配置属性等

-  **条件注解:** 充分利用条件注解，确保你的自动化配置只在需要时生效，避免不必要的 Bean 创建和冲突

-  **配置属性:** 如果你的功能需要用户进行配置，使用 `@ConfigurationProperties` 提供类型安全的配置

-  **日志:** 在自动化配置类中使用日志，方便用户排查问题

-  **测试:** 编写自动化测试来验证你的 Starter 是否按预期工作

---

## 总结

制作自己的 Spring Boot Starter 依赖，是提升代码复用性和开发效率的有效手段通过理解 Spring Boot 的自动化装配原理，特别是 `META-INF/spring.factories` 和 `.imports` 文件以及条件注解的作用，你就能轻松地将你的通用功能封装成易于使用的 Starter，让其他项目能够享受到自动化装配带来的便利
**一个详细具体的 starter 依赖制作教程： [【Spring篇10】：制作自己的spring-boot-starter依赖2](https://blog.csdn.net/lyh2004_08/article/details/149095898)** 