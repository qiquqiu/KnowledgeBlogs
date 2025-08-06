# 【Spring篇10】：制作自己的spring-boot-starter依赖2

> 原创 于 2025-07-04 09:00:00 发布 · 公开 · 995 阅读 · 20 · 17 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149095898

**文章目录**

[TOC]


Spring Boot Starter 是 Spring Boot 生态中非常重要的组成部分通过 Starter，我们可以将一组相关的依赖、配置和自动化装配逻辑打包成一个独立的模块，供其他项目直接引入使用在上文 [制作SpringBootStarter依赖的基本流程](https://blog.csdn.net/lyh2004_08/article/details/149095589) 中，我们打通了整个过程的基本流程，本文我们具体制作一个属于自己的简单 Spring Boot Starter

## 1. 什么是 Spring Boot Starter？

Spring Boot Starter 是一个 Maven 或Gradle模块，它通常包含：

- 一组相关的依赖（如数据库驱动、消息队列客户端等）

- 自动化配置类，用于自动配置这些依赖所需的 Bean

- 条件注解，用于控制这些配置在什么情况下生效

- 可选的属性配置（通过 `application.properties` 或 `application.yml` 文件进行配置）

**Starter 的核心价值在于：** 

-  **简化依赖管理:** 用户只需要引入一个 Starter 依赖，就能获得所需的所有相关依赖

-  **自动配置:** Starter 提供的自动化配置类会根据当前环境自动配置所需的 Bean

-  **开箱即用:** 用户无需手动配置，直接使用 Starter 提供的功能

---

## 2. 制作 Starter 的标准流程

制作一个 Spring Boot Starter 通常需要以下步骤：

1.  **创建项目结构:** 通常包括一个 `xxx-spring-boot-starter` 模块和一个 `xxx-spring-boot-autoconfigure` 模块

2.  **编写自动化配置类:** 在 `autoconfigure` 模块中编写包含 `@Configuration` 和 `@Bean` 的配置类

3.  **使用 `@AutoConfiguration` 注解:** 将自动化配置类标记为 `@AutoConfiguration` ，并使用条件注解控制其生效条件

4.  **注册自动化配置类:** 在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件中列出自动化配置类的全限定名

5.  **打包发布:** 将 Starter 模块打包成 JAR 包，并发布到 Maven 仓库或本地仓库

下面我们通过一个具体的例子来演示如何制作一个简单的 Starter

---

## 3. 实战：制作一个 “Hello World” Starter

假设我们要制作一个名为 `hello-spring-boot-starter` 的 Starter，它提供一个 `HelloService` Bean，能够根据配置的 `hello.name` 属性返回问候语

### 3.1 创建项目结构

首先，创建一个 Maven 项目，包含两个模块：

-  `hello-spring-boot-starter` : 作为依赖的入口，只包含对 `hello-spring-boot-autoconfigure` 的依赖

-  `hello-spring-boot-autoconfigure` : 包含自动化配置逻辑

**项目结构如下：** 

```
hello-spring-boot-starter/
├── pom.xml
└── src
hello-spring-boot-autoconfigure/
├── pom.xml
└── src
    └── main
        ├── java
        │   └── com
        │       └── example
        │           └── hello
        │               ├── HelloAutoConfiguration.java
        │               ├── HelloProperties.java
        │               └── HelloService.java
        └── resources
            └── META-INF
                └── spring
                    └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

### 3.2 编写代码

**`hello-spring-boot-autoconfigure` 模块：** 

1.  **`HelloProperties.java` :** 定义配置属性

   ```java
   package com.example.hello;

   import org.springframework.boot.context.properties.ConfigurationProperties;

   @ConfigurationProperties(prefix = "hello")
   public class HelloProperties {
       private String name = "World";

       public String getName() {
           return name;
       }

       public void setName(String name) {
           this.name = name;
       }
   }
   ```

2.  **`HelloService.java` :** 提供问候服务

   ```java
   package com.example.hello;

   public class HelloService {
       private final String name;

       public HelloService(String name) {
           this.name = name;
       }

       public String sayHello() {
           return "Hello, " + name + "!";
       }
   }
   ```

3.  **`HelloAutoConfiguration.java` :** **自动化配置** 类

   ```java
   package com.example.hello;

   import org.springframework.boot.autoconfigure.AutoConfiguration;
   import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
   import org.springframework.boot.context.properties.EnableConfigurationProperties;
   import org.springframework.context.annotation.Bean;

   @AutoConfiguration
   @EnableConfigurationProperties(HelloProperties.class)
   public class HelloAutoConfiguration {

       @Bean
       @ConditionalOnMissingBean
       public HelloService helloService(HelloProperties properties) {
           return new HelloService(properties.getName());
       }
   }
   ```

4.  **`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` :** 注册自动化配置类

   ```properties
   com.example.hello.HelloAutoConfiguration
   ```

**`hello-spring-boot-starter` 模块：** 

1.  **`pom.xml` :** 只包含对 `hello-spring-boot-autoconfigure` 的依赖

   ```xml
   <dependency>
       <groupId>com.example</groupId>
       <artifactId>hello-spring-boot-autoconfigure</artifactId>
       <version>1.0.0</version>
   </dependency>
   ```

### 3.3 使用 Starter

在其他项目中引入 `hello-spring-boot-starter` 依赖后，就可以直接使用 `HelloService` Bean 了

**`application.properties` :** 

```properties
hello.name=Spring Boot
```

**`MyApp.java` :** 

```java
import com.example.hello.HelloService;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(MyApp.class, args);
        HelloService helloService = context.getBean(HelloService.class);
        System.out.println(helloService.sayHello()); // 输出: Hello, Spring Boot!
    }
}
```

---

## 4. 最佳实践

-  **命名规范:** Starter 通常命名为 `xxx-spring-boot-starter` ，自动化配置模块命名为 `xxx-spring-boot-autoconfigure` 

-  **条件注解:** 使用 `@ConditionalOn...` 注解控制配置的生效条件，避免不必要的 Bean 创建

-  **属性配置:** 使用 `@ConfigurationProperties` 提供可配置的属性，并通过 `application.properties` 或 `application.yml` 文件进行配置

-  **测试:** 为 Starter 编写自动化测试，确保其功能正常

-  **文档:** 提供清晰的文档，说明 Starter 的功能、配置和使用方式

