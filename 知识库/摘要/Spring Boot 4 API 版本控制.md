---
type: source
title: Spring Boot 4 API 版本控制
tldr: Spring Boot 4 起 API 版本控制原生支持：Header 声明版本 + 方法级 version 属性
source_title: Spring Boot 4 终于原生支持 API 版本控制了，我把项目里那套 -v1、-v2 重新改了一遍
source: https://mp.weixin.qq.com/s/c_bGmCQ6ear4lTmOV3EMLw
source_type: 微信文章
author: []
published: null
captured: 2026-09-17
tags: [clippings, Java, Spring, API]
raw: "[[原始资料/剪藏/Spring Boot 4 API 版本控制]]"
status: draft
created: 2026-09-18
updated: 2026-09-18
---

## 一句话主旨

一位开发者把项目里"整套复制 `/v1`、`/v2` Controller"的旧做法，换成 Spring Framework 7 原生 API Versioning 的实践记录：**版本从目录结构变成请求映射的一部分**，并给出迁移期的取舍判断。

## 关键论点

1. **API 版本控制已进入 Spring MVC 官方能力。** 原文：「Spring Framework 7 已经把 API Versioning 正式做进 Spring MVC 了」；「版本怎么从请求里拿、版本是否合法、最终应该匹配哪个 Handler，这些事情交给框架。」

2. **旧方案的根本问题不是"丑"，而是把两种版本绑死。** 原文：「以前按照 /v1、/v2 整套复制 Controller，本质上把『应用版本』和『单个接口版本』绑得太死。」这是全篇的选型起点——**接口版本的变化节奏与应用部署版本本不必一致**。

3. **新方案的实质变化是版本成为请求映射的一部分**：「接口版本终于从『Controller 目录结构』变成了『请求映射的一部分』」——两个方法可以同名同路径，仅靠版本属性区分。

4. **⚠️ 最关键的选型判据（原文明确反对的做法）**：DTO 要分版本，**Service 不要跟版本走**。原文：「虽然两个版本可以放在同一个 Controller，但 DTO 不要为了省事继续共用」；「Service 层反而尽量不要跟版本走。」并明确反对 `OrderV1Service / OrderV2Service / OrderV3Service` 这类切分——「除非两个版本的业务规则真的已经不同」。DTO 侧的具体做法是 `record OrderV1VO(id, orderNo, amount, status)` 与 `record OrderV2VO(id, orderNo, originalAmount, discountAmount, payAmount, orderStatus, logisticsStatus)` 各写各的，并反对在老对象上追加字段「然后希望老客户端『忽略它不认识的字段』」。

5. **默认版本是迁移窗口的关键机制**：「没有 Header，就继续按 1.0 处理。……这给我们留出了一个很舒服的迁移窗口。」原文给出的风险量化很清楚：「如果升级后强制要求每个请求都必须带版本号，这批客户端当天就全挂了。」

6. **不必为了用新功能去改存量。** 原文：「如果 API 已经公开出去，而且大量第三方正在使用 /v1、/v2，为了换一个 Spring 新功能把 URL 全改掉，反而没有必要。」推荐的渐进策略是「旧接口如果已经稳定，而且基本不会再改，就继续保留。真正需要开发 V2、V3 的接口，才逐步迁到新的 API Versioning 上」，并以"业务流程是否已完全不同"作为拆 Controller/Facade 的阈值。

7. **版本来源不止 Header 一种**：「实际上 Spring Boot 4 并没有限制只能使用 Header。如果公司原来的 API 就是 `/api/v1/orders`，也可以继续采用 URL Path Segment；如果以前约定的是 `/api/orders?version=2`，也可以从 Query Parameter 获取。」——**存量约定不必推翻**。

8. **`supported` 白名单用于拒绝未知版本**：原文：「比如请求：X-API-Version: 9.9，不应该莫名其妙落到某个 Controller，而应该直接告诉调用方这个版本不支持。」

9. **版本区间后缀 `+` 表达"向后兼容的小版本"**：`version = "2.0+"` 可覆盖 2.0/2.1/2.2 这类响应结构相同的小版本，避免复制三个方法；「等到 3.0 真正发生不兼容变化的时候，再单独加」`version = "3.0"`。原文称合法写法包括「1、1.1、1.1.2、2.0」。

10. **客户端侧同样可声明版本**：`@HttpExchange` / `@GetExchange` 的 `version` 属性——「Server 端可以按 API Version 路由，HTTP Interface Client 同样可以声明自己调用哪个版本。」

## 机制与配置（原文给出的全部代码）

```yaml
spring:
  mvc:
    apiversion:
      use:
        header: X-API-Version
      default: 1.0
      supported:
        - 1.0
        - 2.0
        - 2.1
```

```java
@GetMapping(value = "/{orderId}", version = "1.0")
public OrderV1VO getOrderV1(@PathVariable Long orderId) { ... }

@GetMapping(value = "/{orderId}", version = "2.0+")
public OrderV2VO getOrderV2(@PathVariable Long orderId) { ... }

// 未发生版本分化的方法不写 version
@GetMapping("/{orderId}/logistics")
public LogisticsVO getLogistics(@PathVariable Long orderId) { ... }
```

原文对最后一种的说法是：「这个方法没有指定 version，可以作为未发生版本分化的处理方法存在。当出现一个带明确版本的更具体映射时，Spring 会优先选择对应版本。」请求侧 `X-API-Version: 1.0` / `2.0`，URL 保持 `GET /api/orders/10001` 不变。

## 与其他资料的关系

**本页与本库已有页面无原文层面的关联。** 原文是 Spring/Web 技术实践文，未提及清结算、支付、跨库 Join、分库分表、分布式系统或 LLM Wiki 的任何主题。

唯一可报告的是**同作者前作的线索**：原文自述「我顺手又把昨天刚改完的 @HttpExchange Client 拿过来试了一下」「这次改完以后，我最大的感受其实跟昨天换 @HttpExchange 差不多」，表明存在一篇同作者的 `@HttpExchange` 前作，**但该文不在本库中**（既不在剪藏目录，也未 ingest）。是否值得追加入库，留待用户判断。

与同批另外两篇 Spring 文章**无概念重叠**：自动装配、Starter、条件注解等概念在本文**完全未出现**，故本文不参与 [[自动装配]]、[[Starter]]、[[条件注解]] 三页的来源支撑。

## 值得追问的问题

1. **URL Path Segment 与 Query Parameter 两种版本来源的具体配置键是什么**——原文只给了 `use.header` 一种 YAML 示例。
2. **不支持的版本返回什么 HTTP 状态码与错误体**——「直接告诉调用方这个版本不支持」的默认行为与可定制性，原文未讲。
3. **`2.0+` 的边界语义不完整**：是否覆盖 2.10？是否覆盖 3.x？当 `3.0` 已单独声明时两者优先级如何裁决？原文只给了正向例子，**未说明区间上界与歧义消解规则**。
4. **未写 `version` 的方法与写了 `version` 的方法同 URL 冲突时的完整优先级规则**——原文只有一句「Spring 会优先选择对应版本」，机制细节（Order 语义、精确匹配 vs 版本特异性）未展开。
5. **`supported` 白名单与 `version = "2.0+"` 组合时的校验顺序**，以及该功能对应 Framework 7 的哪个 API（原文明言未给出任何类名，如 `ApiVersionResolver` 之类）。

## 存疑之处（引用前必读）

本页内容**属于单一来源、且处于版本敏感区**，以下各点必须随引用一并带出：

- **⚠️ 机制断言均无出处。** 「默认版本会按照语义版本的方式解析」与「当出现一个带明确版本的更具体映射时，Spring 会优先选择对应版本」都是强断言，但**全文无文档链接、无 API 类名佐证**，属个人实践观察。**若 Framework 7 后续版本改变解析规则，这两条最先失效。**
- **⚠️ 版本信息不足。** 原文只给出"Spring Boot 4 / Spring Framework 7"两个大版本，**未给任何具体小版本号、最低版本要求、废弃或破坏性变更说明**（如 Boot 4.0.0、Framework 7.0 等精确版本均未讲）。据此无法判断该能力从哪个 patch 起可用。
- **`+` 的语义证据弱于其建议力度**：原文说 `2.0+` 覆盖 2.0/2.1/2.2，但未说明这是框架定义还是作者推断。
- **标题与正文用词不一致（轻微）**：标题写「那套 -v1、-v2」，正文方案均为 `/v1`、`/v2` 路径形式，疑为排版或剪藏转写误差，非观点矛盾。
- **⚠️ 剪藏转写失真（本页最需注意的一点）**：原文**全文没有任何小节标题**，为连续散文加代码块；且代码块中的引号**全部为中文全角引号**，部分代码的换行已丢失——例如原文代码块中的 `11.11.1.22.0` 实为 `1 / 1.1 / 1.1.2 / 2.0` 四个版本被挤成一行，`12` / `1.01.12.0` 属同类问题。**本页列出的代码已按上下文拆分还原，但无法保证与原文逐字一致**；引用示例前建议回溯原文链接。