https://mp.weixin.qq.com/s/qh7AIqOjRXOwh-k6IcgvoQ

# 请设计一个 Java 项目组件！Spring Boot Starter 全解析与案例应用（建议收藏！）

 

> **摘要**：从 Spring Boot Starter 的设计初衷出发，深入分析官方 Starter 的依赖传递机制、```
> @ConfigurationProperties
> ```
> 
>  绑定原理、```
> @EnableConfigurationProperties
> ```
> 
>  的工作方式，以及 ```
> spring-boot-configuration-processor
> ```
> 
>  的编译时处理，深刻理解Starter的原理与应用细节。
> 
> 
> 
> 
> **标签**：Spring Boot、Starter、自动装配、@ConfigurationProperties、条件注解、源码分析

- 一、从一个实际问题说起
- 二、概念定义：什么是 Starter
- 三、背景：为什么需要 Starter
- 四、生活类比：理解 Starter 的设计思想
- 五、源码级分析
- 六、实战案例：从零构建一个自定义 Starter
- 七、分场景讨论
- 八、常见误区
- 九、单元测试
- 十、面试自测
- 十一、总结
- 参考资料

## 一、从一个实际问题说起

小 A 所在团队有多个微服务，每个服务都需要接入一个内部的"消息发送 SDK"。

**最初的方案**：每个服务都手动编写配置类。

```
// 每个项目都要写一遍的配置@Configurationpublic class MessageConfig {    @Bean    public MessageSender messageSender() {        MessageSender sender = new MessageSender();        sender.setAppId("xxx");        sender.setAppSecret("yyy");        sender.setEndpoint("https://msg.internal.com");        sender.setPoolSize(10);        return sender;    }}
```

很快问题就暴露出来：

- • **重复劳动**：每个新项目都要复制粘贴这套配置
- • **版本同步**：SDK 新增了一个配置项（比如 ```
retryTimes
```

），所有服务的 ```
MessageConfig
```

 都要改
- • **配置漂移**：时间一长，不同项目的配置出现差异，排查问题变得困难

**更好的方案**：把配置封装成一个 Starter，其他项目只需要：

```
<!-- 1. 引入依赖 --><dependency>    <groupId>com.example</groupId>    <artifactId>msg-spring-boot-starter</artifactId>    <version>1.0.0</version></dependency>
```

```
# 2. 配置属性（有默认值，可选）msg.app-id=xxxmsg.app-secret=yyy
```

```
// 3. 直接注入使用@Autowiredprivate MessageSender messageSender;
```

这就是 **Starter** 的核心价值：**一次封装，处处可用**。

## 二、概念定义：什么是 Starter

### 2.1 定义

**Starter** 是 Spring Boot 提供的**依赖描述符**（dependency descriptor），它将某个功能模块所需的所有依赖和自动配置打包在一起。引入 Starter 后，Spring Boot 会根据 classpath 中的类自动装配相应的 Bean，无需手动配置。

### 2.2 类比

> Starter 就像餐厅里的**"套餐"**：
> 
> - • **传统方式**（点菜）：你需要自己点主食、配菜、饮料，还要告诉厨师怎么搭配（引入依赖 + 手动配置 Bean + 手动组装）
> - • **Starter 方式**（套餐）：选一个套餐编号，餐厅自动把主食、配菜、饮料配好端上来（引入一个 Starter，所有 Bean 自动装配）
> 
> 不同的套餐对应不同的场景：
> 
> - • "web 套餐"（```
> spring-boot-starter-web
> ```
> 
> ）：Tomcat + Jackson + Spring MVC，拿来就能写 Web 接口
> - • "Redis 套餐"（```
> spring-boot-starter-data-redis
> ```
> 
> ）：Lettuce + RedisTemplate，拿来就能操作 Redis
> - • "Actuator 套餐"（```
> spring-boot-starter-actuator
> ```
> 
> ）：健康检查 + 指标端点，拿来就能监控

### 2.3 没有 Starter vs 有 Starter

**没有 Starter** 时，要使用 Redis：

```
<!-- 手动引入多个依赖 --><dependencies>    <dependency>        <groupId>org.springframework.data</groupId>        <artifactId>spring-data-redis</artifactId>    </dependency>    <dependency>        <groupId>io.lettuce</groupId>        <artifactId>lettuce-core</artifactId>    </dependency></dependencies>
```

```
// 手动创建连接工厂@Beanpublic RedisConnectionFactory connectionFactory() {    return new LettuceConnectionFactory("localhost", 6379);}@Beanpublic RedisTemplate<String, Object> redisTemplate() {    RedisTemplate template = new RedisTemplate();    template.setConnectionFactory(connectionFactory());    return template;}
```

**有了 Starter** 之后：

```
<!-- 一行依赖搞定 --><dependency>    <groupId>org.springframework.boot</groupId>    <artifactId>spring-boot-starter-data-redis</artifactId></dependency>
```

```
# application.propertiesspring.redis.host=localhostspring.redis.port=6379
```

```
// RedisTemplate 自动创建，直接注入！@Autowiredprivate RedisTemplate<String, Object> redisTemplate;
```

## 三、背景：为什么需要 Starter

### 3.1 Spring 时代的问题

在 Spring Boot 之前，使用 Spring 框架开发一个 Web 项目，需要：

1. 1. **手动管理依赖**：确定 Spring 版本、Spring MVC 版本、Jackson 版本、Servlet API 版本，确保它们之间兼容
2. 2. **编写大量 XML 配置**：```
web.xml
```

、```
applicationContext.xml
```

、```
spring-mvc.xml
```
3. 3. **手动组装 Bean**：DataSource、SessionFactory、TransactionManager、ViewResolver，一个一个写

对于小 A 这样的开发者来说，一个项目还没开始写业务逻辑，光是搭框架就要花半天。

### 3.2 Starter 的诞生

Spring Boot 2014 年发布后，Starter 是其核心创新之一。它的出现解决了三个问题：

- • **依赖管理**：Starter 封装了"最佳依赖组合"，用户不需要关心版本兼容性
- • **自动装配**：基于条件注解（```
@ConditionalOnClass
```

、```
@ConditionalOnMissingBean
```

），Spring Boot 在运行时判断是否需要装配某个 Bean
- • **约定优于配置**：提供合理的默认值，绝大多数情况下不需要额外配置

### 3.3 Starter 的本质

```
Starter = 依赖描述（pom.xml 中的传递依赖） + 自动配置代码（AutoConfiguration）
```

- • **依赖描述**：告诉 Maven/Gradle "我要用到哪些 jar"
- • **自动配置代码**：告诉 Spring "在什么条件下创建哪些 Bean"

两者解耦是 Starter 设计的关键——**Starter 模块通常没有代码（纯 pom），真正的配置代码在 autoconfigure 模块中**。

## 四、生活类比：理解 Starter 的设计思想

### 4.1 类比一：家电"套装"

想象你刚搬进新家，需要添置家电：

- • **没有 Starter**：你要分别去挑洗衣机、冰箱、空调、电视，每个品牌规格不同，还要自己研究怎么接水管、接电线
- • **有 Starter**：你选了"家装套装 A"，厂家把客厅和卧室需要的家电全配好，插电就能用

### 4.2 类比二：开发环境的"一键安装"

- • **没有 Starter**：配置开发环境需要装 JDK、配置 Maven、安装 IDE 插件、下载依赖、配置 Tomcat……
- • **有 Starter**：引入 ```
spring-boot-starter-web
```

，JDK + Maven + Tomcat + Spring MVC + Jackson 全部自动配好

### 4.3 类比三：乐高套装

- • Starter 就像乐高的**主题套装**（城市系列、星球大战系列）
- • 每个套装里包含了搭建场景所需的所有积木块（依赖）
- • 你不需要自己从散装积木里挑，打开套装按说明书拼就行
- • 如果你不喜欢某个积木块，也可以替换成自己的（```
@ConditionalOnMissingBean
```

）

## 五、源码级分析

### 5.1 官方 Starter 内部结构深度分析

#### 5.1.1 spring-boot-starter-web 的依赖树分析

```
spring-boot-starter-web
```

 是 Spring Boot 最核心的 Starter 之一。它的 pom.xml 大致如下：

```
<!-- spring-boot-starter-web（简化版） --><project>    <modelVersion>4.0</modelVersion>    <parent>        <groupId>org.springframework.boot</groupId>        <artifactId>spring-boot-starters</artifactId>        <version>3.2.4</version>    </parent>    <artifactId>spring-boot-starter-web</artifactId>    <name>Spring Boot Web Starter</name>    <dependencies>        <!-- spring-boot-starter：通用 Starter（日志 + YAML 支持） -->        <dependency>            <groupId>org.springframework.boot</groupId>            <artifactId>spring-boot-starter</artifactId>        </dependency>        <!-- Spring MVC：核心 Web 框架 -->        <dependency>            <groupId>org.springframework</groupId>            <artifactId>spring-web</artifactId>        </dependency>        <dependency>            <groupId>org.springframework</groupId>            <artifactId>spring-webmvc</artifactId>        </dependency>        <!-- Tomcat：嵌入式 Servlet 容器 -->        <dependency>            <groupId>org.springframework.boot</groupId>            <artifactId>spring-boot-starter-tomcat</artifactId>        </dependency>        <!-- Jackson：JSON 序列化 -->        <dependency>            <groupId>com.fasterxml.jackson.core</groupId>            <artifactId>jackson-databind</artifactId>        </dependency>        <!-- Jakarta Validation：参数校验 -->        <dependency>            <groupId>org.springframework.boot</groupId>            <artifactId>spring-boot-starter-validation</artifactId>        </dependency>    </dependencies></project>
```

**传递依赖机制**（Maven transitive dependency）：

```
spring-boot-starter-web├── spring-boot-starter│   ├── spring-boot-autoconfigure  ← 自动配置核心│   ├── spring-boot                ← Spring Boot 基础│   └── logback-classic            ← 默认日志├── spring-web + spring-webmvc     ← Spring MVC 框架├── spring-boot-starter-tomcat     ← 嵌入式 Tomcat├── jackson-databind               ← JSON 序列化└── spring-boot-starter-validation ← Jakarta Validation
```

这就是为什么引入一个 ```
spring-boot-starter-web
```

 就能获得完整的 Web 开发能力——Maven 的传递依赖机制会自动拉入所有间接依赖。

#### 5.1.2 spring-boot-autoconfigure 模块结构

自动配置的"大脑"在 ```
spring-boot-autoconfigure
```

 模块中。该模块包含大量 ```
AutoConfiguration
```

 类，每个类负责特定场景的自动装配。

```
spring-boot-autoconfigure/├── org/springframework/boot/autoconfigure/│   ├── AutoConfiguration.imports          ← 入口清单│   ├── web/│   │   └── ServletWebServerFactoryAutoConfiguration.java  ← Web 容器│   ├── data/│   │   ├── redis/│   │   │   └── RedisAutoConfiguration.java ← Redis│   │   └── jpa/│   │       └── HibernateJpaAutoConfiguration.java ← JPA│   ├── jdbc/│   │   └── DataSourceAutoConfiguration.java  ← 数据源│   └── ...
```

**spring-boot-autoconfigure 的入口文件**（Spring Boot 3.x）：

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.importsorg.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfigurationorg.springframework.boot.autoconfigure.data.redis.RedisAutoConfigurationorg.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfigurationorg.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration...（共 200+ 个 AutoConfiguration 类）
```

Spring Boot 启动时会读取这个文件，加载所有自动配置类，然后根据每个类上的 ```
@Conditional
```

 注解决定是否生效。

### 5.2 @ConfigurationProperties 绑定原理

#### 5.2.1 Binder 的绑定流程

```
@ConfigurationProperties
```

 的核心是 ```
Binder
```

，它负责将配置文件中的键值对绑定到 Java 对象。绑定流程如下：

```
配置文件（application.properties）         │         ▼┌─────────────────────────┐│ ConfigurationPropertySource │ ← PropertySource 适配层└─────────────────────────┘         │         ▼┌─────────────────────────┐│ ConfigurationPropertyName   │ ← 属性名规范化│   "greeting.prefix"         │   解析、格式化、标准化└─────────────────────────┘         │         ▼┌─────────────────────────┐│ Binder                      │ ← 核心绑定器│   bind(prefix, target)      │└─────────────────────────┘         │         ▼┌─────────────────────────┐│ BindHandler 链            │ ← 回调链│   → Validator             │   校验、转换、命名策略│   → Converter             ││   → NumberCreator         │└─────────────────────────┘         │         ▼  目标对象（GreetingProperties）
```

**第一步：属性源提取**。Spring Boot 将 ```
application.properties
```

 / ```
application.yml
```

 转化为 ```
ConfigurationPropertySource
```

 接口，统一不同来源的属性。

**第二步：属性名解析**。```
ConfigurationPropertyName
```

 对属性名进行规范化处理：

```
GREETING_PREFIX       → greeting.prefixgreeting.prefix       → greeting.prefixgreeting.prefix       → greeting.prefixgreeting-prefix       → greeting.prefixGreeting.Prefix       → greeting.prefix
```

这就是为什么 ```
greeting.prefix
```

、```
greeting-prefix
```

、```
GREETING_PREFIX
```

 都能被正确识别——它们在规范化后指向同一个属性名。

**第三步：Binder.bind() 执行绑定**。```
Binder
```

 遍历目标对象的所有字段（通过反射），将属性值绑定到对应字段。如果字段类型不是 String，还会调用 ```
ConversionService
```

 进行类型转换。

**第四步：BindHandler 链处理**。绑定过程中会经过多个处理器：

- • ```
NoOpBindHandler
```

：默认空操作
- • ```
Validator
```

：执行 ```
@Validated
```

 校验
- • ```
Converter
```

：类型转换
- • ```
IgnoreTopLevelConverterBindHandler
```

：忽略顶层类型转换

#### 5.2.2 ConfigurationPropertiesBean 元数据提取

```
ConfigurationPropertiesBean
```

 封装了配置属性类的元数据：

```
ConfigurationPropertiesBean├── prefix: "greeting"         ← @ConfigurationProperties(prefix="greeting")├── targetType: GreetingProperties.class├── fields:│   ├── prefix: String = "Hello"   ← 字段名、类型、默认值│   └── smart: boolean = false├── validator: Optional<Validator>└── validationAnnotation: @Validated / null
```

Spring Boot 通过 JavaBean 的 getter/setter 对来识别字段，而不是直接访问字段本身。这就要求配置类必须有标准的 getter/setter 方法（或使用记录类）。

#### 5.2.3 Spring Boot 3.x 的 Relaxed Binding 变化

**Spring Boot 2.x** 使用"宽松绑定"（Relaxed Binding），支持大量格式：

```
# Spring Boot 2.x - 这些都能识别greeting.prefix=HellogreetingPrefix=HelloGREETING_PREFIX=Hellogreeting-prefix=Hello
```

**Spring Boot 3.x** 收紧了规则，推荐使用**短横线命名**（kebab-case）：

```
# Spring Boot 3.x - 推荐格式greeting.prefix=Hellogreeting.smart=true
```

Spring Boot 3.x 仍保留部分宽松支持，但不再支持全大写+下划线格式（```
GREETING_PREFIX
```

）。如果使用 IDE 的自动补全功能，编译期就能发现不兼容的属性名。

### 5.3 @EnableConfigurationProperties 的工作机制

#### 5.3.1 注册流程

```
@EnableConfigurationProperties
```

 注解告诉 Spring Boot："把这个 ```
@ConfigurationProperties
```

 类注册为 Bean"。

```
@EnableConfigurationProperties(GreetingProperties.class)              │              ▼┌─────────────────────────────────────────┐│ ConfigurationPropertiesBeanRegistrar        ││  → 扫描 @EnableConfigurationProperties    ││  → 注册 BeanDefinition                     │└─────────────────────────────────────────┘              │              ▼┌─────────────────────────────────────────┐│ ConfigurationPropertiesBindingPostProcessor ││  → Bean 初始化前回调                       ││  → 绑定配置属性到 Bean                     ││  → 校验 Bean（如果有 @Validated）          │└─────────────────────────────────────────┘              │              ▼   GreetingProperties Bean 就绪
```

#### 5.3.2 ConfigurationPropertiesBindingPostProcessor 的后处理

```
@EnableConfigurationProperties
```

 的完整注册流程：

![](wechat_img_1788177250284_1.jpg)

对比 ASCII 流程：

```
@Configuration 类被解析    │    ├── 发现 @EnableConfigurationProperties(GreetingProperties.class)    │    ├── ConfigurationPropertiesBeanRegistrar 注册 BeanDefinition    │    ├── Spring 创建 GreetingProperties 实例    │    ├── ConfigurationPropertiesBindingPostProcessor 拦截    │   ├── 识别 @ConfigurationProperties 注解    │   ├── 创建 ConfigurationPropertiesBean（封装元数据）    │   ├── Binder.bind() —— 从 PropertySource 提取属性值    │   ├── Converter —— 类型转换（String → boolean/int/...）    │   └── Validator —— 校验（如果标注了 @Validated）    │    └── GreetingProperties Bean 就绪，可被注入
```

#### 5.3.3 ConfigurationPropertiesBindingPostProcessor 的后处理

```
ConfigurationPropertiesBindingPostProcessor
```

 实现了 ```
BeanPostProcessor
```

 接口，在 Bean 初始化前后执行处理：

```
// ConfigurationPropertiesBindingPostProcessor 核心逻辑（简化）public Object postProcessBeforeInitialization(Object bean, String beanName) {    ConfigurationProperties annotation = AnnotationUtils.findAnnotation(            bean.getClass(), ConfigurationProperties.class);    if (annotation != null) {        // 1. 创建 ConfigurationPropertiesBean        ConfigurationPropertiesBean propertiesBean =                ConfigurationPropertiesBean.get(applicationContext, bean, annotation);        // 2. 执行绑定        binder.bind(propertiesBean);        // 3. 执行校验        if (propertiesBean.isValidated()) {            validator.validate(propertiesBean);        }    }    return bean;}
```

这就是为什么加上 ```
@EnableConfigurationProperties(GreetingProperties.class)
```

 后，```
GreetingProperties
```

 不仅被注册为 Bean，还会自动从配置文件中绑定属性。

### 5.4 spring-boot-configuration-processor 的编译时处理

#### 5.4.1 生成 additional-spring-configuration-metadata.json

```
spring-boot-configuration-processor
```

 是一个**注解处理器**（Annotation Processor），在编译时扫描 ```
@ConfigurationProperties
```

 类，生成元数据 JSON 文件：

```
编译时（javac）    │    ├── spring-boot-configuration-processor 运行    │   │    │   ├── 扫描 @ConfigurationProperties 类    │   ├── 提取字段名、类型、默认值、Javadoc    │   └── 生成 META-INF/additional-spring-configuration-metadata.json    │    └── 编译完成运行时    └── IDE 读取 JSON 文件 → 提供自动补全
```

#### 5.4.2 生成的 JSON 文件示例

```
{  "groups": [    {      "name": "greeting",      "type": "com.example.starter.GreetingProperties",      "sourceType": "com.example.starter.GreetingProperties"    }  ],  "properties": [    {      "name": "greeting.prefix",      "type": "java.lang.String",      "defaultValue": "Hello",      "description": "问候语前缀。"    },    {      "name": "greeting.smart",      "type": "boolean",      "defaultValue": false,      "description": "是否启用智能问候。"    }  ]}
```

这个 JSON 文件被 IDE（IDEA、VS Code）读取后，在编辑 ```
application.properties
```

 时就能提供：

- • **自动补全**：输入 ```
greeting.
```

 后提示可用属性
- • **类型提示**：显示每个属性的类型
- • **默认值显示**：鼠标悬停显示默认值
- • **描述信息**：显示 Javadoc 注释

#### 5.4.3 Maven 配置

```
<dependency>    <groupId>org.springframework.boot</groupId>    <artifactId>spring-boot-configuration-processor</artifactId>    <optional>true</optional>  <!-- 标记为 optional，不会传递给使用方 --></dependency>
```

注意 ```
optional=true
```

——这说明它是编译时工具，不参与运行时。

## 六、实战案例：从零构建一个自定义 Starter

### 6.1 场景描述

假设公司内部需要一个通用的"问候服务"：

- • 默认问候：```
Hello, {name}!
```
- • 支持时间感知：```
Good morning, {name}!
```

（根据当前时间自动切换）
- • 可通过配置文件自定义前缀
- • 用户可以用自己的实现覆盖默认行为

### 6.2 第一步：定义服务接口

```
public interface GreetingService {    String greet(String name);}
```

### 6.3 第二步：实现服务

**默认实现**：

```
public class DefaultGreetingService implements GreetingService {    private String prefix;    public void setPrefix(String prefix) {        this.prefix = prefix;    }    @Override    public String greet(String name) {        return prefix + ", " + name + "!";    }}
```

**智能实现（时间感知）**：

```
public class SmartGreetingService implements GreetingService {    @Override    public String greet(String name) {        int hour = LocalTime.now().getHour();        String greeting;        if (hour >= 6 && hour < 12) {            greeting = "Good morning";        } else if (hour >= 12 && hour < 18) {            greeting = "Good afternoon";        } else if (hour >= 18 && hour < 24) {            greeting = "Good evening";        } else {            greeting = "Good night";        }        return greeting + ", " + name + "!";    }}
```

### 6.4 第三步：定义配置属性

```
@ConfigurationProperties(prefix = "greeting")public class GreetingProperties {    /** 问候语前缀，默认 Hello */    private String prefix = "Hello";    /** 是否启用智能问候，默认 false */    private boolean smart = false;    public String getPrefix() { return prefix; }    public void setPrefix(String prefix) { this.prefix = prefix; }    public boolean isSmart() { return smart; }    public void setSmart(boolean smart) { this.smart = smart; }}
```

### 6.5 第四步：实现自动配置类

```
@Configuration@EnableConfigurationProperties(GreetingProperties.class)public class GreetingAutoConfiguration {    @Bean    @ConditionalOnMissingBean(GreetingService.class)    @ConditionalOnProperty(prefix = "greeting", name = "smart", havingValue = "true")    public GreetingService smartGreetingService() {        return new SmartGreetingService();    }    @Bean    @ConditionalOnMissingBean(GreetingService.class)    @ConditionalOnProperty(prefix = "greeting", name = "smart",                           havingValue = "false", matchIfMissing = true)    public GreetingService defaultGreetingService(GreetingProperties properties) {        DefaultGreetingService service = new DefaultGreetingService();        service.setPrefix(properties.getPrefix());        return service;    }}
```

**装配逻辑**：

```
GreetingAutoConfiguration├── @EnableConfigurationProperties — 注册 GreetingProperties 为 Bean├── @ConditionalOnMissingBean — 用户自定义了就不装配└── 根据 greeting.smart 选择实现：    ├── smart=true → SmartGreetingService（时间感知）    └── smart=false 或未设置 → DefaultGreetingService（默认前缀）
```

### 6.6 第五步：注册自动配置类

**Spring Boot 3.x 的文件格式**：

文件位置：```
src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```



文件内容：

```
com.example.springbootstartup.starter.GreetingAutoConfiguration
```

> **注意**：Spring Boot 2.7 之前使用 ```
> META-INF/spring.factories
> ```
> 
> ，Spring Boot 3.x 已迁移到新的 ```
> .imports
> ```
> 
>  文件。旧文件格式仍然兼容但已被废弃。

### 6.7 自动装配流程图

![](wechat_img_1788177250447_229.jpg)

## 七、分场景讨论

### 7.1 场景一：最简单的 Starter（无配置）

如果 Starter 不需要任何配置，可以省略 ```
@ConfigurationProperties
```

：

```
@Configurationpublic class SimpleAutoConfiguration {    @Bean    @ConditionalOnMissingBean    public SimpleService simpleService() {        return new SimpleService();    }}
```

适用场景：无状态服务、只需要一个默认实现的工具类。

### 7.2 场景二：有配置的 Starter

最常见的场景，包含 ```
@ConfigurationProperties
```

 和条件装配：

```
@Configuration@EnableConfigurationProperties(MyProperties.class)public class MyAutoConfiguration {    @Bean    @ConditionalOnMissingBean    @ConditionalOnProperty(prefix = "my", name = "enabled", havingValue = "true",                           matchIfMissing = true)    public MyService myService(MyProperties props) {        return new MyService(props.getTimeout(), props.getRetryTimes());    }}
```

适用场景：大多数需要外部配置的组件（数据库连接池、消息队列客户端等）。

### 7.3 场景三：依赖 classpath 的 Starter

如果 Starter 的功能依赖于某个第三方库（比如 Redisson），可以用 ```
@ConditionalOnClass
```

 控制：

```
@Configuration@ConditionalOnClass(RedissonClient.class)@EnableConfigurationProperties(RedissonProperties.class)public class RedissonAutoConfiguration {    @Bean    @ConditionalOnMissingBean(RedissonClient.class)    public RedissonClient redissonClient(RedissonProperties props) {        return Redisson.create(props.toConfig());    }}
```

适用场景：Starter 的某个功能仅在 classpath 存在特定类时才启用。

### 7.4 场景四：分层 Starter（官方推荐）

大型项目建议拆分为两个模块：

```
my-spring-boot-starter (pom，无代码)└── my-spring-boot-autoconfigure (jar，有代码)    ├── MyAutoConfiguration.java    ├── MyProperties.java    └── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

```
<!-- my-spring-boot-starter 的 pom.xml --><dependencies>    <dependency>        <groupId>com.example</groupId>        <artifactId>my-spring-boot-autoconfigure</artifactId>        <version>1.0.0</version>    </dependency></dependencies>
```

这种拆分让用户可以只引入 autoconfigure（如果需要更细粒度控制），也可以直接引入 starter。

## 八、常见误区

### 误区 1：Starter 就是自动配置

**错误理解**：Starter 和自动配置是一回事。

**正确理解**：

- • **Starter** 是依赖描述符（pom.xml 中的传递依赖），负责"引入哪些 jar"
- • **自动配置** 是代码逻辑（```
AutoConfiguration
```

 类），负责"在什么条件下创建哪些 Bean"

两者是**分离**的。Starter 模块通常没有代码（纯 pom），真正的配置代码在 autoconfigure 模块中。引入 Starter 后，Maven 会传递依赖 autoconfigure，其中的自动配置类才会生效。

```
starter 模块（xxx-spring-boot-starter）    └── pom.xml（只声明依赖，无代码）         └── 依赖 → xxx-spring-boot-autoconfigure.jar              └── GreetingAutoConfiguration.java（真正的自动配置代码）
```

### 误区 2：@ConfigurationProperties 不需要 setter

**错误理解**：```
@ConfigurationProperties
```

 只需要字段定义即可。

**正确理解**：

- • ```
@ConfigurationProperties
```

 通过 JavaBean 的 **getter/setter 对** 来绑定属性
- • 如果没有 setter，属性无法被绑定（字段值不会改变）
- • 例外：Java 16+ 的 ```
record
```

 类可以作为不可变的配置属性类（Spring Boot 2.6+ 支持）

```
// ❌ 错误：没有 setter，属性无法绑定@ConfigurationProperties(prefix = "greeting")public class GreetingProperties {    private String prefix = "Hello";    public String getPrefix() { return prefix; }    // 缺少 setPrefix()}// ✅ 正确：有 getter/setter@ConfigurationProperties(prefix = "greeting")public class GreetingProperties {    private String prefix = "Hello";    public String getPrefix() { return prefix; }    public void setPrefix(String prefix) { this.prefix = prefix; }}// ✅ 也可以：使用 record（Spring Boot 2.6+）@ConfigurationProperties(prefix = "greeting")public record GreetingProperties(String prefix, boolean smart) {    // record 自动生成 getter，Spring Boot 会自动匹配}
```

### 误区 3：自定义 Starter 不需要 @ConditionalOnMissingBean

**错误理解**：自动配置类里的 Bean 反正只有没有时才创建，不用加 ```
@ConditionalOnMissingBean
```

。

**正确理解**：

```
@ConditionalOnMissingBean
```

 是 Starter 设计的关键——它允许用户通过定义自己的 Bean 来覆盖自动配置的默认行为。没有它，用户就无法替换 Starter 的默认实现。

```
// ✅ 正确的 Starter 设计@Bean@ConditionalOnMissingBean(GreetingService.class)  // 关键注解public GreetingService greetingService() {    return new DefaultGreetingService();}// 用户可以这样覆盖@Beanpublic GreetingService greetingService() {    return new MyCustomGreetingService();  // 自动配置不会再生成默认 Bean}
```

### 误区 4：spring-boot-configuration-processor 是运行时依赖

**错误理解**：需要把 ```
spring-boot-configuration-processor
```

 打包进最终应用。

**正确理解**：

```
spring-boot-configuration-processor
```

 是一个**编译时注解处理器**，它在编译阶段生成元数据 JSON 文件，运行时完全不需要它。必须标记为 ```
optional=true
```

：

```
<dependency>    <groupId>org.springframework.boot</groupId>    <artifactId>spring-boot-configuration-processor</artifactId>    <optional>true</optional>  <!-- 必须加！ --></dependency>
```

如果不标记 ```
optional=true
```

，引入该 Starter 的其他项目也会被传递这个依赖，导致不必要的传递。

### 误区 5：Starter 可以无限叠加

**错误理解**：引入的 Starter 越多越好，功能越全。

**正确理解**：

过多的 Starter 会带来问题：

- • **启动变慢**：每个 Starter 的 AutoConfiguration 都要被评估，Spring Boot 3.x 默认加载 200+ 个 AutoConfiguration 类
- • **配置冲突**：多个 Starter 可能依赖同一个库的不同版本
- • **隐式依赖**：某些 Starter 可能引入了你不需要的传递依赖

**建议**：

- • 按需引入 Starter，不需要的不要加
- • 定期用 ```
mvn dependency:tree
```

 检查依赖树，发现不必要的传递依赖
- • 对于不需要的自动配置，可以用 ```
@SpringBootApplication(exclude = ...)
```

 排除

```
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})public class MyApplication { ... }
```

## 九、单元测试

具体单元测试：Spring Boot Starter 全解析与案例应用（单元测试）

结果：

![](wechat_img_1788177250653_306.jpg)

测试覆盖：

测试方法验证目标使用的测试工具```
greetingServiceAutoConfigured()
```

GreetingService 自动装配```
@SpringBootTest
```

```
greetingPropertiesBound()
```

```
@ConfigurationProperties
```

 属性绑定```
@SpringBootTest
```

```
configurationPropertiesBindWithCustomPrefix()
```

自定义 prefix 绑定```
ApplicationContextRunner
```

```
configurationPropertiesBindWithSmartTrue()
```

smart 属性绑定```
ApplicationContextRunner
```

```
userDefinedBeanOverridesAutoConfiguration()
```

```
@ConditionalOnMissingBean
```

 覆盖```
ApplicationContextRunner
```

```
conditionalOnMissingBeanOnlyAppliesWhenBeanAbsent()
```

缺失时才装配```
ApplicationContextRunner
```

```
smartGreetingWhenSmartEnabled()
```

```
smart=true
```

 装配 SmartGreetingService```
ApplicationContextRunner
```

```
defaultGreetingWhenSmartDisabled()
```

```
smart=false
```

 装配 DefaultGreetingService```
ApplicationContextRunner
```

```
defaultPrefixWhenNoConfiguration()
```

默认前缀（未配置时）```
ApplicationContextRunner
```

```
enableConfigurationPropertiesRegistersBean()
```

```
@EnableConfigurationProperties
```

 注册```
ApplicationContextRunner
```

```
enableConfigurationPropertiesWithImportNotNeeded()
```

自动注册不需要手动导入```
ApplicationContextRunner
```

```
ApplicationContextRunner
```

 是 Spring Boot 推荐的单元测试工具，它可以在轻量级上下文中测试自动配置，无需启动完整的应用上下文。

## 十、面试自测

以下问题用于自测对 Starter 机制的理解程度：

- •1. Starter 和 AutoConfiguration 的区别是什么？为什么它们是分离的？
- •2. Maven 的传递依赖（transitive dependency）在 Starter 中起什么作用？
- •3. Spring Boot 3.x 注册自动配置类的文件是什么？和 2.x 有什么区别？
- •4. ```
@ConfigurationProperties
```

 绑定属性的底层原理是什么？Binder 是如何工作的？
- •5. Spring Boot 3.x 对 Relaxed Binding 做了什么变化？
- •6. ```
@EnableConfigurationProperties
```

 注解做了什么？它的底层是如何注册 Bean 的？
- •7. ```
ConfigurationPropertiesBindingPostProcessor
```

 在 Bean 生命周期中什么时候执行？
- •8. ```
spring-boot-configuration-processor
```

 是运行时依赖还是编译时依赖？如何证明？
- •9. ```
@ConditionalOnMissingBean
```

 为什么是 Starter 设计的必备注解？
- •10. 官方 Starter 和第三方 Starter 的命名规范分别是什么？
- •11. 如果用户想排除某个自动配置类，有哪些方式？
- •12. 过多的 Starter 会带来什么问题？如何排查？
- •13. 什么是 Starter 的"依赖描述 vs 自动配置分离"设计？为什么要这样设计？
- •14. ```
@ConditionalOnProperty
```

 的 ```
matchIfMissing = true
```

 是什么意思？
- •15. ```
ApplicationContextRunner
```

 和 ```
@SpringBootTest
```

 在测试自动配置时有什么区别？

👉 Spring Boot Starter全解析与案例应用（面试自测与答案）

## 十一、总结

### 11.1 核心要点

要点说明**核心价值**一次封装，处处可用——降低重复配置成本**Starter vs AutoConfigure**Starter 是依赖描述（pom），AutoConfigure 是配置代码（jar）**@ConfigurationProperties**通过 Binder 将配置文件属性绑定到 Java 对象**@EnableConfigurationProperties**注册 ```
@ConfigurationProperties
```

 类为 Bean，并通过后处理器绑定属性**@ConditionalOnMissingBean**允许用户覆盖默认实现，是 Starter 设计的必备注解**AutoConfiguration.imports**Spring Boot 3.x 的自动配置类注册文件（替代 spring.factories）**configuration-processor**编译时注解处理器，生成 IDE 自动补全所需的元数据 JSON### 11.2 最佳实践

1. 1. **按需引入**：不需要的 Starter 不要引入，避免启动变慢和配置冲突
2. 2. **提供合理默认值**：大部分情况下不需要额外配置即可工作
3. 3. **使用 ```
@ConditionalOnMissingBean
```

**：让用户可以覆盖默认实现
4. 4. **依赖与配置分离**：Starter 模块只声明依赖，autoconfigure 模块放配置代码
5. 5. **标记 configuration-processor 为 optional**：它是编译时工具，不参与运行时

## 参考资料

- • Spring Boot 官方文档 - Creating Your Own Starter
- • Spring Boot 源码 - spring-boot-project/spring-boot-autoconfigure
- • Spring Boot 源码 - spring-boot-project/spring-boot-samples
- • MyBatis Spring Boot Starter 源码：https://github.com/mybatis/spring-boot-starter
- • 《Spring Boot 揭秘与实战》- Starter 原理分析章节

 






---
*Source: [WeChat Article](https://mp.weixin.qq.com/s/qh7AIqOjRXOwh-k6IcgvoQ)*