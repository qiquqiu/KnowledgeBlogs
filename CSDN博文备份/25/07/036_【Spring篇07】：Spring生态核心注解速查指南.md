# 【Spring篇07】：Spring生态核心注解速查指南

> 原创 于 2025-07-02 15:00:00 发布 · 公开 · 275 阅读 · 4 · 4 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149064900

**文章目录**

[TOC]



## 一、Spring基础注解

| 注解 | 说明 |
|:---:|:---:|
| @Component等 Stereotype | 标识不同层次的Bean（@Controller/@Service/@Repository） |
| @Autowired | 基于类型自动装配依赖 |
| @Qualifier | 配合@Autowired按名称精确注入 |
| @Scope | 定义Bean作用域（singleton/prototype等） |
| @Configuration | 标记配置类，替代XML配置 |
| @ComponentScan | 指定组件扫描包路径 |
| @Bean | 在配置类中声明Bean |
| @Import | 导入其他配置类 |
| AOP相关注解 | @Aspect/@Before/@After等实现面向切面编程 |


---

## 二、Spring MVC核心注解

| 注解 | 说明 |
|:---:|:---:|
| @RequestMapping | 映射请求路径，可作用于类/方法，类级定义父路径 |
| @RequestBody | 解析HTTP请求体JSON数据并转换为Java对象 |
| @RequestParam | 指定查询参数名称 |
| @PathVariable | 提取URL模板变量（如/user/{id}） |
| @ResponseBody | 将返回对象序列化为JSON响应 |
| @RequestHeader | 获取指定HTTP请求头 |
| @RestController | 组合@Controller + @ResponseBody，简化REST接口开发 |


---

## 三、Spring Boot特色注解

### 1. 应用启动相关

| 注解 | 说明 |
|:---:|:---:|
| @SpringBootApplication | 组合注解：@SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan |
| @EnableAutoConfiguration | 开启自动配置机制，可排除特定配置 |


### 2. Web开发增强

| 注解 | 说明 |
|:---:|:---:|
| @RestController | 同Spring MVC，但默认启用消息转换器 |
| @GetMapping等 | 简化版@RequestMapping（@PostMapping/@PutMapping等） |


### 3. 高级配置

| 注解 | 说明 |
|:---:|:---:|
| @Conditional | 条件化配置Bean（如JVM参数检测） |
| @PropertySource | 加载外部属性文件 |
| @Value | 注入配置属性值 |


---

## 四、使用建议

1.  **分层注解** ：坚持MVC分层原则，@Controller处理请求、@Service封装业务逻辑

2.  **自动装配** ：优先使用构造器注入 **加粗样式** 代替字段注入提升测试性

3.  **配置优先级** ：application.properties > 配置类 > 自动配置

