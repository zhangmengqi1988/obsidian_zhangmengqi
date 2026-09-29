---
type: source
title: Spring Boot 自动装配原理
tldr: Spring Boot 自动装配四步：加载候选、去重、条件过滤、拓扑排序；约定大于配置、用户 Bean 优先
source_title: 吃透 Spring Boot 自动装配的核心原理（约定大于配置，3.0+版本）
source: https://mp.weixin.qq.com/s/tfucgckSk2T1uw4878wp_w
source_type: 微信文章
author: []
published: null
captured: 2026-09-17
tags: [clippings, Java, Spring]
raw: "[[原始资料/剪藏/Spring Boot 自动装配原理]]"
status: draft
created: 2026-09-18
updated: 2026-09-18
---

## 一句话主旨

把 Spring Boot 自动装配拆成一条可复述的四步链路（加载候选 → 去重 → 条件过滤 → 拓扑排序），并围绕它讲清**什么时候装配、装配哪个、按什么顺序、出问题怎么看**——落点是排错与迁移判断。

## 关键论点

1. **自动装配的定义是按条件注册 Bean，而非"帮你写配置"。** 原文：「**自动装配**：Spring Boot 根据 classpath 中的依赖、应用环境（Web/非 Web）、配置属性等条件，自动向 IoC 容器中注册 Bean 的机制。」

2. **核心判据是"用户优先"。** 原文：「这就是自动装配的核心思想：**约定大于配置，按需装配，用户优先**」；「用户自定义的 Bean 会覆盖自动配置的 Bean（@ConditionalOnMissingBean → 用户没配才自动配）」。这条决定了两件事的取舍：想覆盖自动配置，**定义一个同类型 Bean 即可**，不必去改自动配置类；反之，若自己声明了 Bean 却发现自动配置没生效，先查是不是撞上了 `@ConditionalOnMissingBean`。

3. **入口不是注解本身，而是 `@Import` 引入的选择器。** `@SpringBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`；其中 `@EnableAutoConfiguration` 内部是 `@AutoConfigurationPackage` + `@Import(AutoConfigurationImportSelector.class)`。原文点出机制来源：「Spring Boot 用上了 Spring 留下的扩展点（批量导入注册 bean），然后就有了自动装配」。

4. **四步流程**（代码注释口径）：`getCandidateConfigurations`（加载候选）→ `removeDuplicates`（去重，用 `LinkedHashSet`，**保持原始加载顺序**以便后续拓扑排序）→ `filter`（排除列表 + 条件注解过滤）→ `sort`（拓扑排序）。

5. **⚠️ 版本边界判据——Spring Boot 3.x 不再从 `spring.factories` 加载 AutoConfiguration。** 原文：「Spring Boot 3.x **移除了**对 `spring.factories` 中 AutoConfiguration 的支持，必须使用新格式文件」；「如果只配置 `spring.factories`，自动配置将不生效。」两种格式对比：旧格式是 `key=value1,value2` 的 Properties 格式（2.7 双格式兼容），新格式 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 是**每行一个类名的纯文本**（3.x 只读它）。这是全文最有实战价值的一条：**自定义 Starter 在 3.x 下写错注册文件，症状是"静默不生效"**。

6. **多个条件注解之间是 AND 而非 OR。** 原文：「多个条件注解之间是 **AND 关系**——**所有条件都必须满足**，Bean 才会被装配。」

7. **条件注解检查的不是同一个维度**：`@ConditionalOnClass` 看 classpath 有没有指定类，`@ConditionalOnBean` 看 IoC 容器有没有指定 Bean。前者「尝试加载类，不初始化（initialize=false）」以避免副作用；后者「即使 Bean 还没有被创建（Lazy），只要有定义就会被检测到」，且 `@ConditionalOnMissingBean` 的搜索优先级是「当前 BeanFactory → 祖先工厂」。

8. **顺序由拓扑排序保证，不是"写得越靠前越先执行"。** 依赖关系用 `@AutoConfigureOrder`（数值）、`@AutoConfigureBefore`、`@AutoConfigureAfter` 三者声明。原文举例：「配置类：A(@AutoConfigureOrder(10)) B(20) C(after A, 30) D(before B, 25) …拓扑排序结果：A → D → B → C」。

9. **排错路径是自动配置报告，而不是读源码猜。** `debug=true` 或 `--debug` 输出 CONDITIONS EVALUATION REPORT，分 **Positive matches**（条件满足）与 **Negative matches**（条件不满足并说明原因）；Actuator 侧用 `management.endpoints.web.exposure.include=conditions` 后访问 `GET /actuator/conditions`，等价于 `debug=true`。

10. **排除自动配置的三种方式效果是合并的**：注解 `exclude` / `excludeName` / 配置 `spring.autoconfigure.exclude`。原文给的优先级为「`exclude` 属性 > `excludeName` 属性 > `spring.autoconfigure.exclude` 配置」，且「所有被排除的类都不会被加载」。

11. **典型自动配置类的条件写法可直接作为阅读范式**：`DataSourceAutoConfiguration` 用 `@ConditionalOnClass({DataSource.class, EmbeddedDatabaseType.class})` + `@ConditionalOnMissingBean(type = "jakarta.sql.DataSource")`，默认 HikariCP（`matchIfMissing = true`）；`WebMvcAutoConfiguration` 的关键判据是「如果用户继承了 `WebMvcConfigurationSupport`（完全自定义 MVC 配置），则自动配置不生效」而「用户只需实现 `WebMvcConfigurer` 接口（扩展 MVC 配置），自动配置仍然生效」——**这是"扩展"与"接管"的分界线**。

## 关键机制（源码链路）

```
@SpringBootApplication
  └─ @EnableAutoConfiguration
       └─ @Import(AutoConfigurationImportSelector.class)
            └─ selectImports() → getAutoConfigurationEntry()
                 0. 前置检查 isEnabled()（enable=false 直接返回 NO_IMPORTS）
                 1. getCandidateConfigurations  ← SpringFactoriesLoader.loadFactoryNames
                 2. removeDuplicates            ← LinkedHashSet（保序）
                 3. filter                      ← removeAll(exclusions) + AutoConfigurationImportFilter
                 4. sort                        ← @AutoConfigureOrder/Before/After → 拓扑排序（Kahn）
```

条件判断的底层三件套：Spring Framework 的 `Condition` 接口 → Spring Boot 的 `SpringBootCondition` 抽象类 → 返回值 `ConditionOutcome`（含 `boolean match` 与 `ConditionMessage message`，即"匹配/不匹配的原因说明"）。`SpringBootCondition.matches()` 的流程为：记录匹配追踪（供 debug 日志与 auto-configuration report 使用）→ 调用子类 `getMatchOutcome()` 做实际判断 → 记录到 `ConditionEvaluationReport`。

## 与其他资料的关系

与同批的 [[摘要/Spring Boot Starter 全解析]] 构成**同一概念簇**：本篇讲"装配是怎么发生的"，那篇讲"如何把装配打包成 Starter 交付"。两者共享「自动配置」「`AutoConfiguration.imports` 」「条件注解」等概念，因此这些概念在本库中由**两篇来源**共同支撑，已建为完整概念页（见 [[自动装配]]、[[Starter]]、[[条件注解]]）。

除该篇外，**本页与本库其他页面无原文层面的关联**：原文未提及支付、清结算、跨库 Join、分布式系统或 LLM Wiki 的任何主题。

## 值得追问的问题

1. **配套文章未入库**。原文第九章「单元测试」正文只有一个链接与一张截图，并指向《Spring Boot 自动装配原理全解析（单元测试）》与《（面试自测与答案）》两篇独立文章（答案篇未带 URL）——**这两篇的内容不在本剪藏内**，是否另存待查。测试怎么写（如用 `ApplicationContextRunner` 验证条件装配）原文未展开。
2. **图内信息不可得**。5.1 整体流程图、5.2 完整时序图、6.5.3 mermaid 流程图均为 `wechat_img_*.jpg`，文字层未复述其内容；图里是否含正文没有的信息，原文未讲。
3. **`AutoConfigurationMetadataLoader.loadMetadata()` 加载的元数据文件是什么、如何生成**——原文只在 filter 代码里出现一次调用，未解释。
4. **失败分析机制未展开**。面试题 Q8 提到 `SpringBootCondition` 的作用含 `FailureAnalysis`，但正文 6.4.2 的简化代码里没有相应内容。
5. **`@AutoConfiguration` 类级注解（2.7+ 新格式配套）在 3.x 下与 `@Configuration(proxyBeanMethods=false)` 的关系**——第十章示例仍用 `@Configuration`，原文未讲。

## 存疑之处（引用前必读）

本页以下各点属于**原文自身的问题**，本库如实记录、不予抹平：

- **⚠️ 代码不是逐字源码。** 6.4.1 的 `OutcomesFilter`、6.4.2/6.4.3 各条件类、6.5.2 排序器**均被原作者标注为「简化」**；其类名与方法签名（如 `findOutcomesMatch`、`isRequiredMatch`）是作者的示意写法，「统称为 OutcomesFilter」这一说法原文也未给出对应真实类名的出处。**引用这些类名时应视为示意而非权威源码。**
- **四步流程文中存在两种说法。** 6.1 正文写作「排除、去重、过滤、排序」，而同节代码注释与总结章写「getCandidateConfigurations → removeDuplicates → filter → sort」。二者口径不同：按代码注释口径，"加载候选"是第一步，而"排除"实际发生在第三步 filter 内部。**本页采用代码注释口径**，此判断来自本库编译者，非原文明确声明。
- **拓扑排序节自相矛盾。** 6.5.2 一处注释写「拓扑排序（消除环检测）」，另一处写「如果有环，抛出 IllegalArgumentException」——「消除环检测」与「有环则抛异常」语义冲突，前者疑为笔误。
- **简化代码残留不一致的异常消息。** 3.x 的加载逻辑只扫描 `.imports`，但 catch 块的异常消息仍写 `"Unable to load spring.factories"`，与其自身声明的「3.x 只扫描新格式文件」不吻合，属示意代码的复制痕迹。
- **「编译时」表述存疑。** 误区 1 的表格称 `@ConditionalOnClass` 检查时机为「编译/加载时」、`@ConditionalOnBean` 为「运行时」，但原文未解释"编译时"如何成立——条件评估实际发生在容器启动的 BeanDefinition 注册阶段，此表述易误导。`@Conditional(EnumConditions::DataSourceAvailable)` 一行被解读为"检查是否有数据库驱动依赖"，原文亦未给出该 `Condition` 实现的任何代码佐证。
- **术语松散**：全文混用「自动装配」与「自动配置」未加区分（标题用"装配"，第七章配置类用"自动配置类"），属术语不统一而非实质矛盾。