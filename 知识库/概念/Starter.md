---
type: concept
title: Starter
tldr: 把依赖描述符与自动配置代码打包成可复用模块；靠条件装配与属性绑定做到用户可覆盖、一次封装处处可用
aliases: [spring-boot-starter, 起步依赖, 自定义 Starter]
tags: [Java, Spring]
sources: ["[[摘要/Spring Boot Starter 全解析]]", "[[摘要/Spring Boot 自动装配原理]]"]
related: ["[[自动装配]]", "[[条件注解]]"]
confidence: medium
status: draft
created: 2026-09-18
updated: 2026-09-18
---

Starter 是 Spring Boot 的**依赖描述符**（dependency descriptor）——把某个功能模块所需的所有依赖与自动配置打包在一起，引入后框架按 classpath 自动装配，无需手动配置。本页记录它的内部构造、分场景选型判据，以及自建 Starter 时最容易做错的地方。

## 本质公式：两部分，且必须解耦

> **Starter = 依赖描述（pom.xml 中的传递依赖） + 自动配置代码（AutoConfiguration）**

关键在于"解耦"二字：「**Starter 模块通常没有代码（纯 pom），真正的配置代码在 autoconfigure 模块中**」。这不是风格偏好——合成一个模块会导致使用者**无法只要依赖、不要实现**。

## 分层结构

```
my-spring-boot-starter            ← pom，无代码
└── my-spring-boot-autoconfigure  ← jar，有代码
    ├── MyAutoConfiguration.java
    ├── MyProperties.java
    └── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

是否分层的判据是**"要不要给使用者一个只要依赖、不要实现的选项"**，而非规模大小。

## 核心价值：一次封装，处处可用

解决三个问题，它们同时也是使用者不再需要操心的三件事：

| 解决的问题 | Starter 的做法 |
|---|---|
| 依赖管理 | 封装"最佳依赖组合"，使用者不必关心版本兼容性 |
| 自动装配 | 基于条件注解决定是否装配（见 [[条件注解]]） |
| 约定优于配置 | 给合理默认值，绝大多数情况无需额外配置 |

## 设计要点：`@ConditionalOnMissingBean` 不是可选修饰

> 它允许用户通过定义自己的 Bean 来覆盖自动配置的默认行为。**没有它，用户就无法替换 Starter 的默认实现。**

这是 [[自动装配]] "用户优先"原则在交付层的落点：**自建 Starter 时漏掉它，等于把封装做成了硬编码**。凡是给使用者留的可替换点（服务实现、客户端、序列化器），都应挂上它。

## 分场景选型四档

| 档位 | 适用 | 关键手段 |
|---|---|---|
| ① 无配置 | 引入即用 | 只需 AutoConfiguration |
| ② 有配置项 | 需要暴露可调参数 | `@ConfigurationProperties` + `@EnableConfigurationProperties` |
| ③ 依赖可选 classpath | 功能依赖某个可选库（如 Redisson） | `@ConditionalOnClass` |
| ④ 分层 | 需要使用者拆开引入 | starter（pom）+ autoconfigure（jar） |

判据是三个问题：**这个功能有没有可配置项？有没有可选依赖？要不要让使用者拆开引入？** 都否，就用最简单的一档。

## 属性绑定链路

```
application.properties
  → ConfigurationPropertySource
  → ConfigurationPropertyName（规范化）
  → Binder.bind(prefix, target)
  → BindHandler 链（NoOpBindHandler / Validator / Converter / IgnoreTopLevelConverterBindHandler）
  → 目标对象
```

⚠️ **易踩的坑**：「Spring Boot 通过 JavaBean 的 **getter/setter 对**来识别字段，**而不是直接访问字段本身**」——所以"`@ConfigurationProperties` 不需要 setter"是错的（`record` 类型走构造器绑定，属另一条路径）。

`BindHandler` 链中 `Validator` 负责执行 `@Validated` 校验——**这是属性校验的落点**。

**`@EnableConfigurationProperties` 做的是两件事**（常被记成一件）：注册 Bean + 绑定属性。链路为 `ConfigurationPropertiesBeanRegistrar → 注册 BeanDefinition → ConfigurationPropertiesBindingPostProcessor（Bean 初始化前回调 → 绑定配置属性到 Bean → 校验 Bean）`。

## `spring-boot-configuration-processor` 的正确用法

它是一个**编译时注解处理器**，扫描 `@ConfigurationProperties` 类生成元数据 JSON，「运行时完全不需要它。**必须标记为 `optional=true`**」。

它的收益是**IDE 体验**而非运行时行为——生成的 JSON 被 IDEA／VS Code 读取后，编辑 `application.properties` 时提供自动补全、类型提示、默认值显示与描述信息。**标了 `optional` 却看不到提示，通常是 IDE 未重新导入**，与运行时无关。

## 代价：Starter 不是越多越好

「**启动变慢**：每个 Starter 的 AutoConfiguration 都要被评估，Spring Boot 3.x 默认加载 200+ 个 AutoConfiguration 类」——这是叠加 Starter 的成本侧。**引入一个 Starter 就是在启动路径上增加被评估的条件分支**，与前述收益构成选型权衡。

（该"200+"数字在原文中出现两次**均无出处**，引用时需带出这一点。）

## 官方 Starter 的传递依赖是"套餐"思路

`spring-boot-starter-web` 经 Maven 传递依赖拉入：`spring-boot-starter`（含 `spring-boot-autoconfigure`、`spring-boot`、`logback-classic`）、`spring-web + spring-webmvc`、`spring-boot-starter-tomcat`、`jackson-databind`、`spring-boot-starter-validation`。这是"引入一个就获得完整 Web 开发能力"的实现方式，也是前表"依赖管理"一行的具体含义。

## 与其他页面的关系

本页与 [[自动装配]]、[[条件注解]] 同由 [[摘要/Spring Boot Starter 全解析]] 与 [[摘要/Spring Boot 自动装配原理]] **两篇来源**支撑，故建为完整页。三页分工：本页讲**封装与交付**，[[自动装配]] 讲**装配如何发生**，[[条件注解]] 讲**判定维度**。

两来源在本页范围内**互补而非重复**：原文未讲的"多个 AutoConfiguration 之间的执行顺序"，由 [[自动装配]] 的拓扑排序一节补足；而 [[自动装配]] 未讲的"如何把一整套装配交付给别人"，由本页补足。

## 覆盖度说明

置信度 medium。以下为**现有来源未覆盖**的内容：

- **官方与第三方 Starter 的命名规范**——原文只在自测题中提问，正文通篇未给答案；
- **`matchIfMissing = true` 的确切语义**——正文仅出现在代码示例里，无解释；
- 除 `@SpringBootApplication(exclude = ...)` 外还有哪些排除方式（由 [[自动装配]] 补足）；
- 原文第 9、10 章（单元测试、面试题答案）**不在本剪藏内**，另行成文且未带完整 URL。

另有两项**原文自身的问题**，引用前须注意（详见 [[摘要/Spring Boot Starter 全解析]] 的"存疑之处"）：① Relaxed Binding 在 5.2.1 与 5.2.3 两节前后不一致，其中一节未标明适用版本；② 案例包名与产物文件名前后不一致，至少一处有误。

文内多张示意图（含注册流程图与自动装配流程图）在剪藏中仅存引用，**图内文字不在文字层**。