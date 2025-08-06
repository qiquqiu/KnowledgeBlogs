# 【Spring篇03】：@Transcational事务失效几种常见场景及原因简析

> 原创 于 2025-07-02 08:44:54 发布 · 公开 · 699 阅读 · 20 · 15 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149050288

**文章目录**

[TOC]



## 为什么会事务失效？核心在于 AOP 和代理

在深入了解具体的失效情况之前，我们首先要明白Spring 事务管理的底层原理。Spring 的事务管理是通过 **AOP（面向切面编程）** 实现的。当我们为方法添加 `@Transactional` 注解时，Spring 会为这个 Bean 创建一个 **代理对象** 。

当我们通过 Spring 容器获取这个 Bean 并调用带有 `@Transactional` 注解的方法时，实际上调用的是这个代理对象的方法。代理对象会在调用目标方法之前和之后插入事务相关的逻辑，比如：

-  **方法调用前：** 开启事务。

-  **方法调用后（正常返回）：** 提交事务。

-  **方法调用后（抛出异常）：** 回滚事务（根据异常类型和配置）。

>  **总结： 事务失效，很多时候就是因为绕过了这个代理对象，直接调用了原始对象的方法，导致 Spring 事务的增强逻辑没有被应用。** 

---

## Spring 事务失效的常见情况及原理简析

### 概览对比

|  **场景**  |  **原因**  |  **解决方案**  |
|:---:|:---:|:---:|
|  **1. 异常被捕获未抛出**  | Spring通过异常触发回滚，内部 `try-catch` **吞掉** 异常导致事务失效 | 在 `catch` 中抛出新异常（如 `throw new RuntimeException(e)` ） |
|  **2. 异常类型不匹配**  | 默认只回滚 `RuntimeException` 和 `Error` ，不处理检查异常（如 `IOException` ） |  `@Transactional(rollbackFor = Exception.class)` 指定回滚异常类型 |
|  **3. 非public方法**  |  **CGLIB代理** 无法重写 `private` 方法； **JDK代理** 要求接口方法（默认public） |  **强制规范** ：事务方法必须为 `public`  |
|  **4. 类内方法调用**  |  `this.xxx()` 调用 **绕过代理** 对象，直接访问原始方法（ **高频面试题** ） | 方案1：方法拆分到不同Service
方案2： `AopContext.currentProxy()` 获取代理对象调用（需要先暴露代理） |
|  **5. 传播行为配置不当**  | 嵌套事务中 `REQUIRED` / `REQUIRES_NEW` 使用错误导致事务隔离失效 | 理解7种传播行为，慎用嵌套事务 |
|  **6. 类未被Spring管理**  |  `new UserService()` 而非 `@Autowired` ，导致无法生成代理对象 | 确保类标注 `@Component` / `@Service` 等注解 |


### 1. 异常被方法内部捕获并处理

这是最常见的一种事务失效情况。如果在带有 `@Transactional` 注解的方法内部，你使用了 `try-catch` 块捕获了异常，并且没有将异常重新抛出（或者抛出的异常类型不是 Spring 默认会回滚的），那么 Spring 事务管理器就无法感知到异常的发生，自然也就不会触发回滚。

**示例：** 

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Transactional
    public void updateUserAndLog(User user) {
        try {
            userRepository.update(user);
            // 模拟一个可能抛出异常的操作
            int result = 10 / 0; // 会抛出 ArithmeticException (运行时异常)
            logService.saveLog("User updated successfully");
        } catch (Exception e) {
            // 捕获了异常，但没有重新抛出
            System.err.println("Error updating user: " + e.getMessage());
            // 事务不会回滚
        }
    }
}
```

**解决方案：** 

- 不要在事务方法内部捕获应该导致事务回滚的异常。

- 如果确实需要捕获异常进行一些处理（比如记录日志），在处理完后一定要将异常重新抛出，或者抛出一个新的运行时异常。

- 如果只希望对特定类型的异常进行回滚，可以通过 `rollbackFor` 属性指定。

---

### 2. 抛出的异常类型不是 Spring 默认回滚的

Spring 事务默认只对 **运行时异常（ `RuntimeException` 及其子类）** 和 **`Error`** 进行回滚。对于 **检查异常（ `Checked Exception` ）** ，Spring 默认是不会触发回滚的。

**示例：** 

```java
@Service
public class FileService {

    @Transactional
    public void processFile(String filePath) throws IOException {
        // 模拟文件操作，可能抛出 IOException (检查异常)
        readFile(filePath);
        // 如果这里抛出 IOException，事务默认不会回滚
    }

    private void readFile(String path) throws IOException {
        throw new IOException("File not found");
    }
}
```

**解决方案：** 

- 如果需要对检查异常进行回滚，可以使用 `@Transactional` 的 `rollbackFor` 属性指定需要回滚的异常类型，例如：```java
  @Transactional(rollbackFor = IOException.class)
  public void processFile(String filePath) throws IOException {
      readFile(filePath);
  }
  ```

- 如果希望对所有异常（包括检查异常）都进行回滚，可以将 `rollbackFor` 设置为 `Exception.class` ：```java
  @Transactional(rollbackFor = Exception.class)
  public void processFile(String filePath) throws IOException {
      readFile(filePath);
  }
  ```

---

### 3. 事务方法不是 Public 的

Spring AOP默认使用 JDK 动态代理或 CGLIB 代理来实现事务。

-  **JDK 动态代理** 只能代理接口方法，接口方法默认就是 `public` 的。

-  **CGLIB 代理** 通过继承实现，可以代理非接口方法，但无法代理 `private` 方法，对 `protected` 方法的代理也可能存在问题。

因此，为了确保事务能够正确应用，带有 `@Transactional` 注解的方法必须是 `public` 的。

**解决方案：** 

- 确保所有带有 `@Transactional` 注解的方法都是 `public` 的。

---

### 4. 同一个类内部方法互相调用（最常见！）

这是最容易犯错，也是最常见的事务失效情况之一。当同一个类中的一个方法调用另一个带有 `@Transactional` 注解的方法时，比如 `methodA` 调用 `methodB` ：

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder() {
        // ... 创建订单逻辑 ...
        updateInventory(); // 内部调用另一个事务方法
    }

    @Transactional
    public void updateInventory() {
        // ... 更新库存逻辑 ...
        // 如果这里发生异常，createOrder 的事务不会回滚
    }
}
```

在这种情况下， `createOrder` 方法内部调用 `updateInventory` 时，使用的是 `this` 关键字，直接调用了原始 `OrderService` 对象的方法，绕过了 Spring 为 `updateInventory` 生成的代理对象。因此， `updateInventory` 方法上的 `@Transactional` 注解不会生效。

**通俗类比：** 就像你有一个“超能力披风”（代理对象），穿上它你就能拥有事务的能力。但如果你在家里（同一个类内部）直接通过你的名字（ `this` ）喊自己，你穿的是你自己的普通衣服，而不是超能力披风，所以超能力（事务）不会生效。

**解决方案：** 

-  **推荐做法：** 将需要事务的方法拆分到不同的 Service 类中，然后通过 Spring 注入的方式进行调用。这样，每个 Service 类的 Bean 都会被 Spring 代理，调用时自然会走代理逻辑。```java
  @Service
  public class OrderService {
      @Autowired
      private InventoryService inventoryService;

      @Transactional
      public void createOrder() {
          // ... 创建订单逻辑 ...
          inventoryService.updateInventory(); // 调用另一个 Service 的事务方法
      }
  }

  @Service
  public class InventoryService {
      @Transactional
      public void updateInventory() {
          // ... 更新库存逻辑 ...
      }
  }
  ```

-  **备选方案（不推荐）：** 在启动类上添加 `@EnableAspectJAutoProxy(exposeProxy = true)` 注解，然后通过 `AopContext.currentProxy()` 获取当前对象的代理对象，再使用代理对象来调用内部方法。这种方法增加了代码的复杂性，不推荐常用。

---

### 5. 事务的传播行为设置不当

Spring 提供了多种事务传播行为（ `Propagation` ），用于控制嵌套事务的行为。如果对传播行为理解不当或设置错误，也可能导致事务失效或行为不符合预期。

例如， `Propagation.NOT_SUPPORTED` 表示当前方法不应该运行在事务中，如果当前存在事务，则将其挂起。如果在这样的方法中进行了数据库操作，这些操作将不会被事务管理。

**解决方案：** 

- 深入理解各种事务传播行为的含义和作用。

- 在实际开发中，优先使用默认的 `REQUIRED` 传播行为，保持事务结构的简单性。

- 对于复杂的嵌套事务场景，谨慎使用其他传播行为，并进行充分测试。

---

### 6. 事务方法所在的类没有被 Spring 管理

如果带有 `@Transactional` 注解的类没有被 Spring 容器管理（例如，你通过 `new` 关键字手动创建了一个对象，而不是通过 `@Autowired` 或 `@Resource` 注入获取），那么 Spring 就无法为其创建代理对象，自然也无法应用事务增强。

**解决方案：** 

- 确保所有带有 `@Transactional` 注解的类都是 Spring Bean，例如使用 `@Service` , `@Component` , `@Repository` 等注解标记，并确保 Spring 能够扫描到这些类。

---

## 生产环境最佳实践总结

为了避免 Spring 事务失效，并在生产环境中更好地应用和管理事务，建议遵循以下最佳实践：

1.  **明确事务边界：** 仔细分析业务需求，确定哪些操作应该包含在一个事务中。

2.  **使用 `public` 方法：** 确保所有带有 `@Transactional` 注解的方法都是 `public` 的。

3.  **避免内部调用：** 将事务方法拆分到不同的 Service 类中，通过注入调用。

4.  **谨慎处理异常：** 在事务方法内部捕获异常后，如果需要回滚，务必重新抛出或抛出新的运行时异常。对于检查异常，使用 `rollbackFor` 明确指定回滚。

5.  **优先使用 `REQUIRED` 传播行为：** 保持事务结构的简单性。

6.  **确保 Bean 被 Spring 管理：** 带有 `@Transactional` 注解的类必须是 Spring Bean。

