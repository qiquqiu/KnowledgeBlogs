# 【Spring篇08】：理解自动装配，从spring.factories到.imports剖析

> 原创 已于 2025-07-03 15:40:35 修改 · 公开 · 1.1k 阅读 · 29 · 12 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149095382

**文章目录**

[TOC]


Spring Boot 自动化装配是其最引人注目的特性之一，它极大地简化了 Spring 应用的配置，让开发者能够更专注于业务逻辑本文将带深入探讨 Spring Boot 自动化装配的原理，并回顾其从早期版本到现代版本的演进过程

## 1. 自动化装配的起点： `@SpringBootApplication` 

每个 Spring Boot 应用的入口点通常都有一个 `@SpringBootApplication` 注解这个复合注解是自动化装配的起点，它集成了三个核心注解：

-  **`@SpringBootConfiguration`** : 标记当前类为配置类，等同于 `@Configuration` 

-  **`@EnableAutoConfiguration`** : **开启 Spring Boot 自动化装配的关键注解** 

-  **`@ComponentScan`** : 开启组件扫描，用于发现和注册应用内的 Bean

其中， `@EnableAutoConfiguration` 是自动化装配的核心

---

## 2. 自动化装配的核心机制： `@EnableAutoConfiguration` 和 `AutoConfigurationImportSelector` 

`@EnableAutoConfiguration` 注解内部通过 `@Import` 注解导入了一个名为 `AutoConfigurationImportSelector` 的类

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class}) // 核心在这里！
public @interface EnableAutoConfiguration {
    // ...
}
```

`AutoConfigurationImportSelector` 的主要职责是在应用启动时， **找到并加载所有符合条件的自动化配置类** 

---

## 3.自动化配置的注册方式： `spring.factories` 与 `.imports` 

Spring Boot 自动化配置类并不是随便放在项目中的，它们需要被“注册”起来，以便 `AutoConfigurationImportSelector` 能够发现它们在 Spring Boot 的不同版本中，注册方式有所演进

### 3.1 早期版本： `META-INF/spring.factories` 

在 Spring Boot 2.7 版本之前，自动化配置类主要通过 `META-INF/spring.factories` 文件进行注册这个文件位于 JAR 包的 `META-INF` 目录下，遵循 Java 的 ServiceProviderInterface (SPI) 机制

`spring.factories` 是一个属性文件，其中 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 键对应的值就是自动化配置类的全限定名列表，用逗号分隔

```properties
# 示例：早期版本 Starter 中的 META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

`AutoConfigurationImportSelector` 在早期版本中主要依赖 `SpringFactoriesLoader` 工具类来读取这些 `spring.factories` 文件 `SpringFactoriesLoader` 会扫描当前应用的 **整个运行时类路径** ，查找所有 JAR 包中的 `META-INF/spring.factories` 文件，并加载指定键对应的值

### 3.2 现代版本： `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 

从 Spring Boot 2.7 版本开始，为了使自动化配置的注册更加清晰和规范，引入了 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 这个专门用于注册自动化配置类的文件

这个文件每行包含一个自动化配置类的全限定名

```properties
# 示例：现代版本 Starter 中的 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

在 Spring Boot 2.7+ 版本中， `AutoConfigurationImportSelector` 会同时从 `spring.factories` 和 `.imports` 文件中加载自动化配置类如我们之前分析的 `getCandidateConfigurations` 方法所示：

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    // 从 spring.factories 加载
    List<String> configurations = new ArrayList(SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader()));
    // 从 .imports 文件加载 (Spring Boot 2.7+ 新增)
    ImportCandidates.load(AutoConfiguration.class, this.getBeanClassLoader()).forEach(configurations::add);
    // ... 检查是否找到配置类
    return configurations;
}
```

**在 Spring Boot 3.x 版本中， `.imports` 文件成为了主要和推荐的自动化配置注册方式** 

---

## 4. 运行时类路径：自动化配置的基础

无论是 `spring.factories` 还是 `.imports` 文件， `AutoConfigurationImportSelector` 能够找到它们的前提是它们所在的 JAR 包存在于应用的 **运行时类路径 (Runtime Classpath)** 中

运行时类路径是JVM在运行的 Java 应用程序时，用来查找和加载 `.class` 文件的路径集合它包括：

> 

- 自己的项目编译输出的 `.class` 文件

- 项目依赖的所有 JAR 包

- JDK 自带的类库

`SpringFactoriesLoader` 和 `ImportCandidates` 工具类正是利用了 JVM 的类加载机制，遍历整个运行时类路径，查找指定位置的资源文件当通过 Maven 或 Gradle 引入一个 Spring Boot Starter 或其他包含自动化配置的 JAR 包时，这个 JAR 包就会被添加到的运行时类路径中，从而使得其中的 `spring.factories` 或 `.imports` 文件能够被 Spring Boot 发现

---

## 5. 条件判断：决定配置是否生效

找到自动化配置类只是第一步Spring Boot 的自动化装配之所以智能，是因为它会根据当前应用程序的环境来决定哪些配置类应该真正生效这个决策过程依赖于一系列的 **条件注解** 

这些条件注解通常加在自动化配置类或其中的 `@Bean` 方法上常见的条件注解包括：

-  **`@ConditionalOnClass`** : 当指定的类存在于运行时类路径中时生效

-  **`@ConditionalOnMissingBean`** : 当 Spring 容器中不存在指定类型的 Bean 时生效

-  **`@ConditionalOnProperty`** : 当指定的配置属性存在且符合指定值时生效

-  **`@ConditionalOnResource`** : 当指定的资源文件存在于类路径中时生效

-  **`@ConditionalOnWebApplication`** : 当应用程序是 Web 应用时生效

- 等等…

`AutoConfigurationImportSelector` 在加载了候选的自动化配置类名后，会进一步检查这些类上的条件注解只有当所有条件注解都满足时，这个自动化配置类才会被激活，其中的 Bean 定义才会被注册到 Spring 容器中

**举例来说：** 

> 如果引入了 `spring-boot-starter-data-redis` 依赖，那么Redis相关的 JAR 包会出现在的运行时类路径中，其中包含 `RedisOperations.class` 等类 `spring-boot-autoconfigure` 模块中的 `RedisAutoConfiguration` 类上带有 `@ConditionalOnClass({ RedisOperations.class, RedisConnectionFactory.class })` 注解因为这些类存在，条件满足， `RedisAutoConfiguration` 生效，Spring Boot 自动为配置 Redis 相关的 Bean

如果没有引入该依赖，这些类不存在，条件不满足， `RedisAutoConfiguration` 被忽略

---

## 6. 总结

Spring Boot 自动化装配是一个强大而智能的特性，它通过以下机制实现：

-  **`@EnableAutoConfiguration` :** 开启自动化装配

-  **`AutoConfigurationImportSelector` :** 核心选择器，负责查找和加载自动化配置类

-  **注册文件 ( `spring.factories` 和 `.imports` ):** 声明哪些类是自动化配置类，位于 JAR 包的特定位置

-  **运行时类路径:** 这些注册文件必须存在于应用的运行时类路径中才能被发现

-  **条件注解 ( `@ConditionalOnClass` 等):** 根据当前环境决定自动化配置是否生效

