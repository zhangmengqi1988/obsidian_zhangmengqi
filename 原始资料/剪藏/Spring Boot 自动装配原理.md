https://mp.weixin.qq.com/s/tfucgckSk2T1uw4878wp_w

# 吃透 Spring Boot 自动装配的核心原理（约定大于配置，3.0+版本）

 

> **摘要**：深入讲解 Spring Boot 自动装配的核心原理，从 @SpringBootApplication 注解到 SpringFactoriesLoader 机制，从条件注解到自定义 Starter，从源码分析到实战测试。通过 What-Why-How 的逻辑，帮助理解自动装配的设计思想与实现细节。
> 
> 
> 
> 
> **标签**：Spring Boot、自动装配、SPI、@EnableAutoConfiguration、条件注解、SpringFactoriesLoader

## 目录一览

- 目录一览
- 一、从一个实际问题说起
- 二、什么是自动装配？
- 三、为什么要自动装配？
- 四、自动装配的核心注解
- 五、自动装配的完整流程
- 六、四步流程源码深度分析
- 七、常见自动配置类深度解析
- 八、实战：查看自动配置报告
- 九、单元测试
- 十、自定义自动配置（Starter 开发）
- 十一、排除自动配置
- 十二、常见误区
- 十三、最佳实践
- 十四、面试自测
- 十五、总结
- 参考资料

## 一、从一个实际问题说起

小 A 在学习 Spring Boot 时，发现了一个让人困惑的现象：

```
// 只是引入了 spring-boot-starter-data-redis 依赖// 没有配置任何 Redis 相关的 Bean// 但是 RedisTemplate 可以直接注入使用！@Autowiredprivate RedisTemplate<String, Object> redisTemplate;  // 居然不为 null！
```

小 A 产生了一连串疑问：

1. 1. Spring Boot 是怎么知道要配置 Redis 的？它没有看到任何显式配置啊。
2. 2. 这些 Bean 是从哪里来的？是谁创建了它们？
3. 3. 如果我想自己写一个"开箱即用"的 Starter，该怎么做？
4. 4. 为什么有时候自动配置不生效？排错的时候怎么看原因？

这就是 Spring Boot **自动装配（Auto-Configuration）**机制在背后发挥作用。理解了自动装配，就理解了 Spring Boot 的"灵魂"。

## 二、什么是自动装配？

### 2.1 概念定义

**自动装配**：Spring Boot 根据 classpath 中的依赖、应用环境（Web/非 Web）、配置属性等条件，自动向 IoC 容器中注册 Bean 的机制。

> **类比**：自动装配就像一个"智能管家"——你不需要告诉它每一步怎么做，它会根据你的"家当"（classpath 依赖）和"生活习惯"（配置属性），自动帮你把 things 安排妥当。
> 
> - • 发现你引入了 MySQL 驱动 → 自动配置 DataSource
> - • 发现你引入了 Redis Starter → 自动配置 RedisTemplate
> - • 发现你引入了 Spring MVC → 自动配置 DispatcherServlet、视图解析器等
> - • 发现你自己配置了 Bean → 自动退让，不重复装配

### 2.2 传统 Spring vs Spring Boot

**传统 Spring：配置繁琐**

```
// 传统 Spring：需要手动配置大量 Bean@Configurationpublic class AppConfig {    @Bean    public DataSource dataSource() {        HikariConfig config = new HikariConfig();        config.setJdbcUrl("jdbc:mysql://localhost:3306/db");        config.setUsername("root");        config.setPassword("password");        return new HikariDataSource(config);    }    @Bean    public SqlSessionFactory sqlSessionFactory() throws Exception {        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();        factoryBean.setDataSource(dataSource());        return factoryBean.getObject();    }    @Bean    public RedisTemplate redisTemplate() {        RedisTemplate template = new RedisTemplate();        template.setConnectionFactory(redisConnectionFactory());        return template;    }    @Bean    public ViewResolver viewResolver() {        InternalResourceViewResolver resolver = new InternalResourceViewResolver();        resolver.setPrefix("/WEB-INF/views/");        resolver.setSuffix(".jsp");        return resolver;    }    // ... 可能还有几十个 Bean 要手动配置}
```

**Spring Boot：几乎零配置**

```
// Spring Boot：一个注解搞定@SpringBootApplicationpublic class Application {    public static void main(String[] args) {        SpringApplication.run(Application.class, args);    }}// application.properties 简单配置spring.datasource.url=jdbc:mysql://localhost:3306/dbspring.datasource.username=rootspring.datasource.password=passwordspring.redis.host=localhostspring.redis.port=6379
```

**对比一目了然**：传统 Spring 需要几十行甚至上百行配置，Spring Boot 只需几行配置文件，因为Spring Boot 帮你做了，够贴心，开箱即用。

### 2.3 生活类比

把自动装配比作一家**智能餐厅**的运营流程：

```
┌─────────────────────────────────────────────────────┐│                  智能餐厅操作系统                      ││                                                     ││  1. 检查食材（classpath 依赖）                         ││     - 有面粉 → 提供面包菜单                            ││     - 有牛排 → 提供牛排菜单                            ││     - 有海鲜 → 提供海鲜菜单                            ││                                                     ││  2. 检查顾客偏好（配置属性）                            ││     - 顾客说"不要辣" → 跳过辣味菜品                     ││     - 顾客说"素食" → 只展示素食                        ││                                                     ││  3. 检查顾客自带食物（自定义 Bean）                      ││     - 顾客带了蛋糕 → 餐厅不再提供蛋糕                    ││     - 顾客没带 → 餐厅自动提供默认蛋糕                    ││                                                     ││  4. 上菜顺序（排序）                                   ││     - 前菜 → 主菜 → 甜点                              ││     - 某些菜必须在另一些菜之前上                         │└─────────────────────────────────────────────────────┘
```

这就是自动装配的核心思想：**约定大于配置，按需装配，用户优先**。

## 三、为什么要自动装配？

### 3.1 Spring 时代的痛点

在 Spring Boot 出现之前，使用 Spring 框架开发应用面临几个突出问题：

**问题 1：配置爆炸**

一个典型的企业应用需要配置几十个 Bean——数据源、事务管理器、ORM 框架、Web MVC、安全框架、缓存等。每个框架都有自己的配置方式，开发者需要逐一学习。

**问题 2：版本兼容**

Spring、Spring MVC、Spring Data、Hibernate 等组件之间有严格的版本对应关系。选错版本组合会导致难以排查的运行时错误。

**问题 3：重复劳动**

大多数应用的配置模式是相似的——DataSource 配连接池、Redis 配连接工厂、MVC 配视图解析器。这些配置在不同项目中反复出现，几乎没有变化。

**问题 4：入门门槛高**

新手需要理解大量 Spring 概念（BeanFactory、ApplicationContext、BeanPostProcessor 等）才能开始写业务代码。

### 3.2 自动装配的解决思路

Spring Boot 给出的答案：**约定大于配置（Convention over Configuration）**。

```
核心原则：├── 绝大多数应用使用相同的默认配置├── 自动检测 classpath 中的依赖，推断需要装配的组件├── 提供合理的默认值，用户可覆盖└── 用户自定义 Bean 优先级高于自动配置
```

**核心思想**：

1. 1. **推断式装配**：根据 classpath 中的 jar 包推断需要装配什么（有 RedisTemplate 类 → 装配 Redis 相关 Bean）
2. 2. **条件式装配**：只有条件满足时才装配，不满足时自动跳过（有 @ConditionalOnClass → 类存在才装配）
3. 3. **用户优先**：用户自定义的 Bean 会覆盖自动配置的 Bean（@ConditionalOnMissingBean → 用户没配才自动配）

## 四、自动装配的核心注解

### 4.1 @SpringBootApplication 拆解

```
@SpringBootConfiguration@EnableAutoConfiguration@ComponentScanpublic @interface SpringBootApplication {}
```

注解作用```
@SpringBootConfiguration
```

标记为配置类（元注解，等价于 ```
@Configuration
```

）```
@EnableAutoConfiguration
```

**自动装配的核心入口**```
@ComponentScan
```

扫描当前包及子包下的 ```
@Component
```

、```
@Service
```

 等组件```
@EnableAutoConfiguration
```

 是自动装配的关键，下面深入分析它的内部结构。

### 4.2 @EnableAutoConfiguration 的内部结构

```
@AutoConfigurationPackage          // 将主类所在包注册为自动扫描包@Import(AutoConfigurationImportSelector.class)  // 导入自动配置选择器public @interface EnableAutoConfiguration {    // exclude 和 excludeName 用于排除指定的自动配置类    Class<?>[] exclude() default {};    String[] excludeName() default {};}
```

**关键设计**：```
@Import(AutoConfigurationImportSelector.class)
```

 通过 ```
@Import
```

 注解导入了 ```
AutoConfigurationImportSelector
```

。这是自动装配的真正入口——Spring 容器在处理 ```
@Import
```

 时，会调用 ```
AutoConfigurationImportSelector
```

 的 ```
selectImports()
```

 方法，返回需要导入的自动配置类的全限定名数组。【Spring Boot 用上了 Spring 留下的扩展点（批量导入注册bean），然后就有了自动装配】

## 五、自动装配的完整流程

### 5.1 整体流程图

![](wechat_img_1788246811448_607.jpg)

### 5.2 完整时序图

![](wechat_img_1788246811785_586.jpg)

## 六、四步流程源码深度分析

### 6.1 AutoConfigurationImportSelector.selectImports()

这是自动装配的入口方法，负责 orchestrating 整个自动装配流程。

```
// AutoConfigurationImportSelector.java@Overridepublic String[] selectImports(AnnotationMetadata annotationMetadata) {    // 0. 前置检查：如果 @EnableAutoConfiguration 的 enable 属性为 false，直接返回空数组    if (!isEnabled(annotationMetadata)) {        return NO_IMPORTS;    }    // 1. 获取所有自动配置候选类（从 spring.factories / .imports 文件）    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(            annotationMetadata);    // 2. 将 AutoConfigurationEntry 中的配置类名称转为数组返回    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());}
```

```
getAutoConfigurationEntry()
```

 是核心方法，内部完成了四步流程：

排除、去重、过滤、排序。和我们的业务处理类似的步骤。

```
// 简化后的核心逻辑protected AutoConfigurationEntry getAutoConfigurationEntry(        AnnotationMetadata annotationMetadata) {    // Step 0: 检查是否启用    if (!isEnabled(annotationMetadata)) {        return EMPTY_ENTRY;    }    // 获取 exclude 和 excludeName 参数    AnnotationAttributes attributes = getAttributes(annotationMetadata);    // Step 1: 获取候选配置类    List<String> configurations = getCandidateConfigurations(            annotationMetadata, attributes);    // Step 2: 去重    configurations = removeDuplicates(configurations);    // 获取 exclude 的类    Set<String> exclusions = getExclusions(annotationMetadata, attributes);    // Step 3: 过滤（条件注解 + 排除列表）    configurations = filter(configurations, getAutoConfigurationImportListener(),            annotationMetadata, exclusions);    // Step 4: 排序    configurations = sort(configurations);    return new AutoConfigurationEntry(configurations, exclusions);}
```

### 6.2 Step 1: getCandidateConfigurations — 加载候选配置类

这一步负责从 classpath 中读取所有声明的自动配置类。

```
// AutoConfigurationImportSelector.javaprotected List<String> getCandidateConfigurations(        AnnotationMetadata metadata, AnnotationAttributes attributes) {    // 调用 SpringFactoriesLoader 加载    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(            EnableAutoConfiguration.class,            this.beanClassLoader);    // 断言：至少要有配置类，否则抛出异常    Assert.notEmpty(configurations,            "No auto configuration classes found in META-INF/spring/" +            "org.springframework.boot.autoconfigure.AutoConfiguration.imports" +            ". If you are using a custom packaging, make sure that file is correct.");    return configurations;}
```

#### 6.2.1 SpringFactoriesLoader 双文件加载机制

```
SpringFactoriesLoader
```

 是 Spring Boot 的核心工具类，负责加载 ```
META-INF
```

 下的工厂定义文件。在 Spring Boot 2.7 到 3.x 的演进中，加载机制发生了变化。

**Spring Boot 3.x 的加载逻辑**：

```
// SpringFactoriesLoader.java (Spring Boot 3.x)public static List<String> loadFactoryNames(Class<?> factoryType,        @Nullable ClassLoader classLoader) {    ClassLoader classLoaderToUse = classLoader;    if (classLoaderToUse == null) {        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();    }    // Spring Boot 3.x：只读取新格式文件    String factoryTypeName = factoryType.getName();    // 路径：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports    return loadSpringFactories(classLoaderToUse)            .getOrDefault(factoryTypeName, Collections.emptyList());}private static Map<String, List<String>> loadSpringFactories(        ClassLoader classLoader) {    // 1. 检查缓存（避免重复加载）    Map<String, List<String>> result = cache.get(classLoader);    if (result != null) {        return result;    }    result = new HashMap<>();    try {        // Spring Boot 3.x：只扫描新格式文件        Enumeration<URL> urls = classLoader.getResources(                "META-INF/spring/" +                "org.springframework.boot.autoconfigure.AutoConfiguration.imports");        // 2. 遍历所有 jar 包中的 .imports 文件        while (urls.hasMoreElements()) {            URL url = urls.nextElement();            // 3. 读取文件内容            String resource = ResourceReader.readAll(url);            // 按行分割，去除空行和注释            for (String line : resource.split("\n")) {                line = line.trim();                if (!line.isEmpty() && !line.startsWith("#")) {                    String factoryTypeName = line;                    result.computeIfAbsent(                            "org.springframework.boot.autoconfigure.EnableAutoConfiguration",                            k -> new ArrayList<>()).add(factoryTypeName);                }            }        }    } catch (IOException ex) {        throw new IllegalArgumentException(                "Unable to load spring.factories", ex);    }    // 4. 放入缓存    result.replaceAll((factoryType, implementations) ->            implementations.stream().distinct().collect(Collectors.toList()));    cache.put(classLoader, result);    return result;}
```

**Spring Boot 2.7 的兼容加载逻辑**：

```
// Spring Boot 2.7 同时支持两种格式Enumeration<URL> urls = classLoader.getResources(        "META-INF/spring.factories");  // 旧格式// 如果旧格式没有，再找新格式Enumeration<URL> newUrls = classLoader.getResources(        "META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports");
```

**关键区别**：

特性Spring Boot 2.7Spring Boot 3.x文件格式```
META-INF/spring.factories
```

（旧）+ ```
.imports
```

（新）仅 ```
.imports
```

文件路径```
META-INF/spring.factories
```

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

文件格式```
key=value1,value2
```

 的 Properties 格式每行一个类名的纯文本格式兼容性双格式兼容不兼容 ```
spring.factories
```

 中的 AutoConfiguration**Spring Boot 3.x .imports 文件示例**：

```
# spring-boot-autoconfigure-3.x.x.jar# 路径：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.importsorg.springframework.boot.autoconfigure.admin.SpringApplicationAdminAutoConfigurationorg.springframework.boot.autoconfigure.aop.AopAutoConfigurationorg.springframework.boot.autoconfigure.cache.CacheAutoConfigurationorg.springframework.boot.autoconfigure.data.redis.RedisAutoConfigurationorg.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfigurationorg.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration...
```

可以看到，新格式比旧格式更简洁——每行一个类名，不需要 key 前缀，也不需要使用反斜杠续行。

#### 6.2.2 classpath 扫描机制

```
classLoader.getResources()
```

 会从所有 jar 包的 classpath 中查找匹配的资源文件。如果有多个 jar 包都提供了 ```
.imports
```

 文件，Spring Boot 会将它们**合并**，而不是覆盖。

```
应用 classpath 中有多个 jar 包：├── spring-boot-autoconfigure-3.2.4.jar│   └── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports│       → DataSourceAutoConfiguration│       → RedisAutoConfiguration│       → WebMvcAutoConfiguration│       → ...├── my-spring-boot-starter-1.0.0.jar│   └── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports│       → MyAutoConfiguration└── 项目本身    └── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports        → 无（项目通常不提供此文件）合并结果：DataSourceAutoConfiguration, RedisAutoConfiguration, WebMvcAutoConfiguration, MyAutoConfiguration, ...
```

### 6.3 Step 2: removeDuplicates — 去重

去重策略使用 ```
LinkedHashSet
```

 保持插入顺序的同时去除重复项：

```
// AutoConfigurationImportSelector.javaprotected final <T> List<T> removeDuplicates(List<T> list) {    // LinkedHashSet：保持插入顺序 + 去重    return new ArrayList<>(new LinkedHashSet<>(list));}
```

**为什么需要去重**：

不同 jar 包可能声明了相同的自动配置类。例如，某个 starter 可能显式依赖了 ```
spring-boot-starter-web
```

，而 ```
spring-boot-autoconfigure
```

 本身也声明了 ```
WebMvcAutoConfiguration
```

。去重后只保留一份，避免重复注册。

**为什么用 LinkedHashSet**：

保持配置类的原始加载顺序很重要，因为后续的排序（Step 4）需要基于这个顺序进行拓扑排序。```
LinkedHashSet
```

 在去重的同时保留了插入顺序。

### 6.4 Step 3: filter — 条件注解过滤（核心）

这是自动装配的"过滤网"——从数百个候选配置类中，筛选出当前环境真正需要的。

```
// AutoConfigurationImportSelector.javaprivate List<String> filter(List<String> configurations,        AutoConfigurationImportListener importListener,        AnnotationMetadata metadata, Set<String> exclusions) {    // 3.1 处理排除列表（exclude / excludeName / spring.autoconfigure.exclude）    List<String> copy = new ArrayList<>(configurations);    copy.removeAll(exclusions);    // 3.2 通过 AutoConfigurationImportFilter 进行条件注解过滤    AutoConfigurationImportFilter filter = getAutoConfigurationImportFilter();    // 调用 filter.match() 评估每个配置类的所有条件注解    boolean[] match = filter.match(copy, AutoConfigurationMetadataLoader            .loadMetadata(this.beanClassLoader));    // 3.3 记录匹配结果并移除未匹配的配置类    List<String> result = new ArrayList<>();    for (int i = 0; i < match.length; i++) {        if (match[i]) {            result.add(copy.get(i));        }    }    // 触发 AutoConfigurationImportListener 事件    importListener.onAutoConfigurationImportEvent(            new AutoConfigurationImportEvent(metadata, result, copy, match));    return result;}
```

#### 6.4.1 AutoConfigurationImportFilter 的排除机制

Spring Boot 通过 ```
AutoConfigurationImportFilter
```

 接口实现条件注解过滤：

```
public interface AutoConfigurationImportFilter {    /**     * 评估自动配置类是否应该被导入。     * @param autoConfigurationClasses 自动配置类的全限定名列表     * @param autoConfigurationMetadata 自动配置元数据（缓存了条件注解信息）     * @return 布尔数组，true 表示应该导入，false 表示应该排除     */    boolean[] match(List<String> autoConfigurationClasses,            AutoConfigurationMetadata autoConfigurationMetadata);}
```

Spring Boot 提供了 ```
OnBeanCondition
```

 和 ```
OnClassCondition
```

 等实现，统称为 ```
OutcomesFilter
```

：

```
// OutcomesFilter.java（简化）class OutcomesFilter implements AutoConfigurationImportFilter {    private final AutoConfigurationConditions conditions;    @Override    public boolean[] match(List<String> autoConfigurationClasses,            AutoConfigurationMetadata metadata) {        boolean[] match = new boolean[autoConfigurationClasses.size()];        for (int i = 0; i < autoConfigurationClasses.size(); i++) {            String className = autoConfigurationClasses.get(i);            match[i] = isRequiredMatch(className, metadata);        }        return match;    }}
```

#### 6.4.2 条件注解的底层实现

所有条件注解的底层都依赖于 Spring Framework 的 ```
Condition
```

 接口和 Spring Boot 的 ```
SpringBootCondition
```

 抽象类。

**Spring Framework 的 Condition 接口**：

```
// org.springframework.context.annotation.Condition.java@FunctionalInterfacepublic interface Condition {    /**     * 判断条件是否满足。     * @param context 条件评估上下文（可访问 BeanFactory、Environment 等）     * @param metadata 当前被注解的类的元数据     * @return true 表示条件满足，组件应该被注册；false 表示不满足，跳过     */    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);}
```

**Spring Boot 的 SpringBootCondition 抽象类**：

```
// SpringBootCondition.java（简化）public abstract class SpringBootCondition implements Condition {    private final Log logger;    @Override    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {        // 1. 记录匹配追踪（用于 debug 日志和 auto-configuration report）        String className = metadata.getClassName();        try {            // 2. 调用子类实现的 getMatchOutcome() 进行实际判断            ConditionOutcome outcome = getMatchOutcome(context, metadata);            // 3. 记录匹配结果（Positive/Negative）            if (outcome.isMatch()) {                logOutcome(className, outcome, true);            } else {                logOutcome(className, outcome, false);            }            // 4. 记录到 ConditionEvaluationReport（用于 debug=true 的输出）            ConditionEvaluationReport.get(context.getBeanFactory())                    .recordOutcome(metadata, outcome);            return outcome.isMatch();        } catch (NoClassDefFoundError ex) {            // 某些类加载错误可能导致条件评估失败            logger.trace("Did not match because of missing class " + ex.getClassName());            return false;        }    }    /**     * 子类实现：具体的条件判断逻辑。     */    public abstract ConditionOutcome getMatchOutcome(            ConditionContext context, AnnotatedTypeMetadata metadata);}
```

**ConditionOutcome 返回值**：

```
// ConditionOutcome.javapublic final class ConditionOutcome {    private final boolean match;         // 是否匹配    private final ConditionMessage message;  // 匹配/不匹配的原因说明    // 工厂方法    public static ConditionOutcome match(ConditionMessage message) { ... }    public static ConditionOutcome noMatch(ConditionMessage message) { ... }}
```

#### 6.4.3 核心条件注解的源码分析

**OnClassCondition — classpath 检查**：

```
// OnClassCondition.java（简化）final class OnClassCondition extends SpringBootCondition implements ConfigurationCondition {    @Override    public ConditionOutcome getMatchOutcome(ConditionContext context,            AnnotatedTypeMetadata metadata) {        // 获取所有 @ConditionalOnClass 和 @ConditionalOnMissingClass 注解属性        ClassLoader classLoader = context.getClassLoader();        ConditionOutcome matchOutcome = findOutcomesClassMatch(                metadata, classLoader);        if (matchOutcome != null) {            return matchOutcome;        }        // @ConditionalOnMissingClass 检查        matchOutcome = findOutcomesMissingClassMatch(metadata, classLoader);        return matchOutcome;    }    private ConditionOutcome findOutcomesClassMatch(            AnnotatedTypeMetadata metadata, ClassLoader classLoader) {        // 获取 @ConditionalOnClass 的 value 属性        AnnotationAttributes onClass = getAnnotationAttributes(                metadata, ConditionalOnClass.class.getName());        if (onClass != null) {            // 逐个检查 classpath 中是否存在指定类            String[] classNames = onClass.getStringArray("value");            for (String className : classNames) {                if (!isPresent(className, classLoader)) {                    return ConditionOutcome.noMatch(                        ConditionMessage.forCondition(ConditionalOnClass.class)                            .didNotFind("class")                            .items(Style.QUOTE, className));                }            }        }        return null;  // 继续检查其他条件    }    private boolean isPresent(String className, ClassLoader classLoader) {        try {            // 尝试加载类，不初始化（initialize=false）            // 只检查类是否存在，不触发类的静态初始化            ResourceUtils.forName(className, classLoader);            return true;        } catch (Throwable ex) {            return false;        }    }}
```

**关键点**：

- • 使用 ```
Class.forName()
```

 加载类，但 **不初始化**（```
initialize=false
```

），避免副作用
- • 嵌套的 ```
@Conditional
```

 也会被递归检查
- • 匹配结果会输出到 debug 日志

**OnBeanCondition — Bean 存在性检查**：

```
// OnBeanCondition.java（简化）final class OnBeanCondition extends SpringBootCondition        implements ConfigurationCondition, BeanFactoryAware {    @Override    public ConditionOutcome getMatchOutcome(ConditionContext context,            AnnotatedTypeMetadata metadata) {        // 获取 @ConditionalOnBean / @ConditionalOnMissingBean 的注解属性        BeanSearchSpec spec = new BeanSearchSpec(context, metadata);        // 搜索容器中的 Bean        List<String> matching = getMatchingBeans(context, spec);        if (matching.isEmpty()) {            // @ConditionalOnMissingBean：没有找到匹配的 Bean → 条件满足            return ConditionOutcome.match(                ConditionMessage.forCondition(ConditionalOnMissingBean.class, spec)                    .didNotFind("any beans")            );        } else {            // @ConditionalOnMissingBean：找到了匹配的 Bean → 条件不满足            return ConditionOutcome.noMatch(                ConditionMessage.forCondition(ConditionalOnMissingBean.class, spec)                    .found("beans")                    .items(Style.QUOTE, matching)            );        }    }    /**     * Bean 搜索策略：     * 1. 按类型（byType）：在 BeanFactory 中查找指定类型的 Bean     * 2. 按名称（byName）：按 Bean 名称查找     * 3. 按注解（byAnnotation）：查找带有指定注解的 Bean     *     * 搜索范围：     * - 当前 BeanFactory（优先）     * - 如果当前工厂找不到，搜索父/祖先工厂（可配置）     */    private List<String> getMatchingBeans(ConditionContext context,            BeanSearchSpec spec) {        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();        List<String> matching = new ArrayList<>();        // 1. 按类型搜索        if (!spec.getTypes().isEmpty()) {            for (Class<?> type : spec.getTypes()) {                String[] names = beanFactory.getBeanNamesForType(type);                matching.addAll(filterBeanNames(names, spec, beanFactory));            }        }        // 2. 按名称搜索        for (String name : spec.getNames()) {            if (beanFactory.containsBean(name)) {                matching.add(name);            }        }        // 3. 按注解搜索        for (Class<? extends Annotation> annotation : spec.getAnnotations()) {            String[] names = beanFactory.getBeanNamesForAnnotation(annotation);            matching.addAll(filterBeanNames(names, spec, beanFactory));        }        return matching;    }}
```

**关键设计**：

- • ```
@ConditionalOnMissingBean
```

 的搜索优先级：当前 BeanFactory → 祖先工厂
- • 会检查 ```
BeanDefinition
```

 级别的定义，而不仅仅是已实例化的 Bean
- • 即使 Bean 还没有被创建（Lazy），只要有定义就会被检测到

**OnPropertyCondition — 属性匹配**：

```
// OnPropertyCondition.java（简化）class OnPropertyCondition extends SpringBootCondition {    @Override    public ConditionOutcome getMatchOutcome(ConditionContext context,            AnnotatedTypeMetadata metadata) {        // 1. 获取 @ConditionalOnProperty 的注解属性        MultiValueMap<String, Object> allAttributes = metadata.getAllAnnotationAttributes(                ConditionalOnProperty.class.getName(), false);        // 2. 逐个属性组进行检查        for (Map<String, Object> attributes : allAttributes.asMultiValueMap().values()) {            ConditionOutcome outcome = getMatchOutcome(                    context,                    (String) attributes.get("prefix"),       // 属性前缀                    (String[]) attributes.get("name"),        // 属性名                    (String) attributes.get("havingValue"),   // 期望值                    (Boolean) attributes.get("matchIfMissing"));  // 缺失时默认匹配            if (outcome != null) {                return outcome;            }        }        return null;    }    private ConditionOutcome getMatchOutcome(ConditionContext context,            String prefix, String[] names, String havingValue,            boolean matchIfMissing) {        Environment environment = context.getEnvironment();        for (String name : names) {            // 拼接完整属性名：prefix.name            String key = prefix + name;            // 3. 解析 placeholder（如 ${spring.app.name}）            String resolvedKey = environment.resolvePlaceholders(key);            String value = environment.getProperty(resolvedKey);            // 4. 检查 havingValue 匹配            if (havingValue != null) {                if (value == null) {                    // 属性缺失                    if (matchIfMissing) {                        continue;  // matchIfMissing=true → 视为匹配                    }                    return ConditionOutcome.noMatch("property " + key + " not found");                }                // 比较属性值                if (!havingValue.equals(value)) {                    return ConditionOutcome.noMatch(                        "property " + key + " expected='" + havingValue +                        "' actual='" + value + "'");                }            } else {                // 没有指定 havingValue → 只检查属性是否存在                if (value == null && !matchIfMissing) {                    return ConditionOutcome.noMatch("property " + key + " not found");                }            }        }        return ConditionOutcome.match();    }}
```

**关键参数组合**：

prefixnamehavingValuematchIfMissing行为```
spring.redis
```

```
enabled
```

```
true
```

```
false
```

属性必须存在且值为 "true"```
spring.redis
```

```
enabled
```

```
true
```

```
true
```

属性为 "true" 或属性缺失时匹配```
spring.redis
```

```
host
```

(空)```
false
```

属性必须存在（任意值）```
spring.redis
```

```
host
```

(空)```
true
```

属性存在或属性缺失时都匹配### 6.5 Step 4: sort — 拓扑排序

自动配置类之间有严格的依赖关系——DataSource 必须在 JdbcTemplate 之前，Redis 连接工厂必须在 RedisTemplate 之前。

#### 6.5.1 排序策略

Spring Boot 提供三种排序注解：

```
// @AutoConfigureOrder — 数值排序（类似 @Order）@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)public class SomeAutoConfiguration { ... }// @AutoConfigureBefore — 在指定配置之前@AutoConfigureBefore(WebMvcAutoConfiguration.class)public class SomeOtherAutoConfiguration { ... }// @AutoConfigureAfter — 在指定配置之后@AutoConfigureAfter(DataSourceAutoConfiguration.class)public class JdbcTemplateAutoConfiguration { ... }
```

#### 6.5.2 拓扑排序算法

```
// AutoConfigurationSorter.java（简化）public List<String> getInPriorityOrder(Collection<String> classNames) {    // 1. 读取每个配置类的 @AutoConfigureOrder 值    // 2. 读取每个配置类的 @AutoConfigureBefore 和 @AutoConfigureAfter    // 3. 构建依赖图    Map<String, Set<String>> dependencies = new HashMap<>();    Map<Integer, List<String>> ordered = new TreeMap<>();  // 按 order 值排序    for (String className : classNames) {        // 读取排序注解信息        Integer order = getOrder(className);  // @AutoConfigureOrder 值        List<String> before = getBefore(className);  // @AutoConfigureBefore        List<String> after = getAfter(className);    // @AutoConfigureAfter        // 4. 建立依赖关系        for (String beforeClass : before) {            // className 必须在 beforeClass 之前            dependencies.computeIfAbsent(className, k -> new HashSet<>())                .add(beforeClass);        }        for (String afterClass : after) {            // className 必须在 afterClass 之后            dependencies.computeIfAbsent(afterClass, k -> new HashSet<>())                .add(className);        }    }    // 5. 拓扑排序（消除环检测）    return process(configurations, dependencies);}private List<String> process(Collection<String> classNames,        Map<String, Set<String>> dependencies) {    // Inverter：反转依赖关系    // 将 "A before B" 转换为 "B after A"    Set<String> processed = new LinkedHashSet<>();    List<String> result = new ArrayList<>();    // 6. 按 order 值分组排序    // 先处理 @AutoConfigureOrder 数值最小的（优先级最高）    // 然后按 @AutoConfigureBefore/@After 建立拓扑排序    // 7. 拓扑排序（ Kahn 算法）    // - 找出没有前置依赖的节点    // - 将其加入结果集    // - 从依赖图中移除该节点    // - 重复直到所有节点处理完毕    // - 如果有环，抛出 IllegalArgumentException    return result;}
```

**排序示例**：

```
配置类：  A (@AutoConfigureOrder(10))  B (@AutoConfigureOrder(20))  C (@AutoConfigureAfter(A), @AutoConfigureOrder(30))  D (@AutoConfigureBefore(B), @AutoConfigureOrder(25))排序过程：  1. 按 @AutoConfigureOrder 初步排序：A(10), B(20), C(30), D(25)  2. 解析 @AutoConfigureBefore/@After：     - C 必须在 A 之后（C after A）     - D 必须在 B 之前（D before B）  3. 拓扑排序结果：A → D → B → C验证：  A(10) 最前 ✓  D before B ✓ (D 在 B 之前)  C after A ✓ (C 在 A 之后)  B(20) before C(30) ✓ (order 值小的在前)
```

#### 6.5.3 拓扑排序的 mermaid 流程图

![](wechat_img_1788246811980_991.jpg)

### 6.6 AutoConfigurationImportListener 事件机制

Spring Boot 还提供了事件监听机制，允许外部代码在自动装配导入时获得通知：

```
public interface AutoConfigurationImportListener {    /**     * 自动装配导入事件。     * 在 auto-configuration 类被导入到 ApplicationContext 之前触发。     * 可以用来记录日志、修改配置类列表、收集统计信息等。     */    void onAutoConfigurationImportEvent(            AutoConfigurationImportEvent event);}
```

Spring Boot 内部有一个实现用于记录自动配置报告（```
AutoConfigurationReport
```

），当 ```
debug=true
```

 时输出 CONDITIONS EVALUATION REPORT。

## 七、常见自动配置类深度解析

### 7.1 DataSourceAutoConfiguration

```
@Configuration(proxyBeanMethods = false)@ConditionalOnClass({DataSource.class, EmbeddedDatabaseType.class})@ConditionalOnMissingBean(type = "jakarta.sql.DataSource")@Conditional(EnumConditions::DataSourceAvailable)@EnableConfigurationProperties(DataSourceProperties.class)@Import({DataSourcePoolMetadataProvidersConfiguration.class,         DataSourceInitializationConfiguration.class,         DataSourceJmxConfiguration.class,         DataSourceConfiguration.class})public class DataSourceAutoConfiguration {    // 内部通过 @Import 引入多个子配置    // DataSourceConfiguration 中包含 Hikari、Tomcat、Dbcp2 等连接池的自动配置    @Configuration(proxyBeanMethods = false)    @ConditionalOnClass(HikariDataSource.class)    @ConditionalOnMissingBean    @ConditionalOnProperty(name = "spring.datasource.type",                           havingValue = "com.zaxxer.hikari.HikariDataSource",                           matchIfMissing = true)    static class Hikari {        @Bean        DataSource dataSource(DataSourceProperties properties) {            return properties.initializeDataSourceBuilder()                    .type(HikariDataSource.class).build();        }    }}
```

**解读**：

1. 1. ```
@ConditionalOnClass
```

：classpath 有 ```
DataSource
```

 和 ```
EmbeddedDatabaseType
```

 才生效
2. 2. ```
@ConditionalOnMissingBean(type = "...")
```

：用户自定义了 ```
jakarta.sql.DataSource
```

 就不装配
3. 3. ```
@Conditional(EnumConditions::DataSourceAvailable)
```

：检查是否有数据库驱动依赖
4. 4. ```
@EnableConfigurationProperties
```

：注册 ```
DataSourceProperties
```

，绑定 ```
spring.datasource.*
```

 配置
5. 5. 默认使用 HikariCP 连接池（```
matchIfMissing = true
```

）

### 7.2 WebMvcAutoConfiguration

```
@Configuration(proxyBeanMethods = false)@ConditionalOnWebApplication(type = Type.SERVLET)@ConditionalOnClass({Servlet.class, DispatcherServlet.class,                     WebMvcConfigurer.class})@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)@AutoConfigureAfter({DispatcherServletAutoConfiguration.class,                     ValidationAutoConfiguration.class})public class WebMvcAutoConfiguration {    @Bean    @ConditionalOnMissingBean    public HiddenHttpMethodFilter hiddenHttpMethodFilter() {        return new HiddenHttpMethodFilter();    }    @Bean    @ConditionalOnMissingBean    public FormContentFilter formContentFilter() {        return new FormContentFilter();    }    // ViewResolver、MessageConverter、ResourceHandler 等 Bean 配置...}
```

**解读**：

- • ```
@ConditionalOnWebApplication(type = Type.SERVLET)
```

：只在 Servlet Web 环境下生效
- • ```
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
```

：如果用户继承了 ```
WebMvcConfigurationSupport
```

（完全自定义 MVC 配置），则自动配置不生效
- • 用户只需实现 ```
WebMvcConfigurer
```

 接口（扩展 MVC 配置），自动配置仍然生效

### 7.3 JacksonAutoConfiguration

```
@Configuration(proxyBeanMethods = false)@ConditionalOnClass(ObjectMapper.class)@Conditional(JacksonMappingsAvailableCondition.class)public class JacksonAutoConfiguration {    @Configuration(proxyBeanMethods = false)    @ConditionalOnMissingBean    static class JacksonMixinModuleConfiguration {        @Bean        @ConditionalOnBean(Jackson2ObjectMapperBuilder.class)        @ConditionalOnMissingBean        public Jackson2ObjectMapperBuilder.Customizer jacksonMixinModuleCustomizer(                ObjectProvider<AbstractJsonMixinModule> modules,                @JacksonMixin AbstractJsonMixinModule... additionalModules) {            // 配置 Jackson Mixin        }    }    @Configuration(proxyBeanMethods = false)    @ConditionalOnMissingBean(ObjectMapper.class)    static class JacksonObjectMapperConfiguration {        @Bean        public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder) {            return builder.build();        }    }}
```

**解读**：

- • ```
@ConditionalOnClass(ObjectMapper.class)
```

：有 Jackson 库才生效
- • ```
@ConditionalOnMissingBean(ObjectMapper.class)
```

：用户自定义了 ObjectMapper 就不自动配
- • ```
@ConditionalOnBean(Jackson2ObjectMapperBuilder.class)
```

：有 Builder 才配置 Mixin

## 八、实战：查看自动配置报告

### 8.1 启用调试日志

Spring Boot 提供了内置的自动配置报告，可以查看每个自动配置类的匹配情况。

**方式 1：配置文件**

```
# application.propertiesdebug=true
```

**方式 2：启动参数**

```
java -jar app.jar --debug
```

**输出示例**：

```
============================CONDITIONS EVALUATION REPORT============================Positive matches:（匹配成功的配置）-----------------   DataSourceAutoConfiguration matched:      - @ConditionalOnClass found required class 'jakarta.sql.DataSource'      - @ConditionalOnMissingBean did not find any beans        of type jakarta.sql.DataSource   RedisAutoConfiguration matched:      - @ConditionalOnClass found required class        'org.springframework.data.redis.core.RedisOperations'      - @ConditionalOnProperty        spring.redis.url matched   JacksonAutoConfiguration matched:      - @ConditionalOnClass found required class        'com.fasterxml.jackson.databind.ObjectMapper'      - @ConditionalOnMissingBean did not find any beansNegative matches:（未匹配的配置）-----------------   SecurityAutoConfiguration did not match:      - @ConditionalOnClass did not find required class        'org.springframework.security.core.userdetails.UserDetailsService'   JmsAutoConfiguration did not match:      - @ConditionalOnClass did not find required class        'jakarta.jms.Message'   RedisCacheConfiguration did not match:      - @ConditionalOnProperty spring.cache.type=redis        (property not set)
```

**解读要点**：

- • **Positive matches**：条件满足，Bean 会被注册
- • **Negative matches**：条件不满足，Bean 被跳过，并说明原因
- • 每行都列出了具体的条件注解和匹配/不匹配的详细原因

### 8.2 使用 Spring Boot Actuator

对于运行中的应用，可以通过 Actuator 端点查看：

```
<dependency>    <groupId>org.springframework.boot</groupId>    <artifactId>spring-boot-starter-actuator</artifactId></dependency>
```

```
management.endpoints.web.exposure.include=conditions,beans,configprops
```

```
# 查看自动配置报告（等价于 debug=true 的输出）GET /actuator/conditions# 查看所有已注册的 BeanGET /actuator/beans# 查看所有配置属性绑定GET /actuator/configprops
```

## 九、单元测试

具体单元测试：[Spring Boot 自动装配原理全解析（单元测试）](./Spring Boot 自动装配原理全解析（单元测试）.md)

测试截图：

![](wechat_img_1788246812087_688.jpg)

## 十、自定义自动配置（Starter 开发）

### 10.1 场景：公司内部的通用服务

假设公司有一个内部 SDK，需要统一配置：

```
// 公司内部 SDKpublic class MyService {    private String prefix;    public String sayHello(String name) {        return prefix + ", " + name;    }}
```

**目标**：让其他项目引入 Starter 后，自动配置 MyService，开箱即用。

### 10.2 创建配置属性类

```
@ConfigurationProperties(prefix = "my")public class MyProperties {    private String prefix = "Hello";    public String getPrefix() { return prefix; }    public void setPrefix(String prefix) { this.prefix = prefix; }}
```

### 10.3 创建自动配置类

```
@Configuration(proxyBeanMethods = false)@ConditionalOnClass(MyService.class)@EnableConfigurationProperties(MyProperties.class)public class MyAutoConfiguration {    @Autowired    private MyProperties properties;    @Bean    @ConditionalOnMissingBean    public MyService myService() {        MyService service = new MyService();        service.setPrefix(properties.getPrefix());        return service;    }}
```

### 10.4 注册配置类

**Spring Boot 3.x 新格式**（必须使用）：

```
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.importscom.example.MyAutoConfiguration
```

> ⚠️ **注意**：Spring Boot 3.x 不再从 ```
> spring.factories
> ```
> 
>  中加载 AutoConfiguration。如果只配置 ```
> spring.factories
> ```
> 
> ，自动配置将不生效。

### 10.5 打包发布

```
<project>    <groupId>com.example</groupId>    <artifactId>my-spring-boot-starter</artifactId>    <version>1.0.0</version>    <dependencies>        <dependency>            <groupId>org.springframework.boot</groupId>            <artifactId>spring-boot-autoconfigure</artifactId>        </dependency>        <dependency>            <groupId>com.example</groupId>            <artifactId>my-sdk</artifactId>        </dependency>    </dependencies></project>
```

### 10.6 使用自定义 Starter

```
<!-- 其他项目引入依赖 --><dependency>    <groupId>com.example</groupId>    <artifactId>my-spring-boot-starter</artifactId>    <version>1.0.0</version></dependency>
```

```
# 覆盖默认配置my.prefix=Welcome
```

```
@Servicepublic class UserService {    @Autowired    private MyService myService;  // 自动注入，无需额外配置！    public void greet(String name) {        System.out.println(myService.sayHello(name));    }}
```

## 十一、排除自动配置

### 11.1 三种排除方式

```
// 方式 1：@SpringBootApplication 注解排除@SpringBootApplication(exclude = {    DataSourceAutoConfiguration.class,    RedisAutoConfiguration.class})public class Application { }// 方式 2：配置文件排除// spring.autoconfigure.exclude=\//   org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\//   org.springframework.boot.autoconfigure.redis.RedisAutoConfiguration// 方式 3：测试时通过属性排除@SpringBootTest(properties = {    "spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.redis.RedisAutoConfiguration"})public class MyTest { }
```

**优先级**：```
exclude
```

 属性 > ```
excludeName
```

 属性 > ```
spring.autoconfigure.exclude
```

 配置。三种方式的效果是合并的——所有被排除的类都不会被加载。

## 十二、常见误区

### 误区 1：@ConditionalOnClass 和 @ConditionalOnBean 的区别

**错误认知**：两者都是"检查是否存在"，可以互换使用。

**正确理解**：它们的检查维度完全不同。

条件注解检查维度检查时机典型场景```
@ConditionalOnClass
```

classpath 是否有指定类编译/加载时有 Redis 库才装配 Redis 配置```
@ConditionalOnBean
```

IoC 容器是否有指定 Bean运行时有 DataSource 才装配 JdbcTemplate```
// 正确：检查 classpath 用 @ConditionalOnClass@ConditionalOnClass(RedisTemplate.class)  // classpath 有 Redis 库public class RedisAutoConfiguration { ... }// 正确：检查容器 Bean 用 @ConditionalOnBean@ConditionalOnBean(DataSource.class)  // 容器中有 DataSource Beanpublic class JdbcTemplateAutoConfiguration { ... }// 错误：混用！@ConditionalOnBean(RedisTemplate.class)  // ❌ 这会检查容器中是否有 RedisTemplate Bean// 但 RedisTemplate 是我们要自动装配的 Bean！这会导致循环检查
```

### 误区 2：spring.factories 还能用于 AutoConfiguration

**错误认知**：Spring Boot 3.x 仍然可以从 ```
spring.factories
```

 加载自动配置。

**正确理解**：Spring Boot 3.x **移除了**对 ```
spring.factories
```

 中 AutoConfiguration 的支持，必须使用新格式文件。

```
# Spring Boot 2.7 — 仍然可以工作# META-INF/spring.factoriesorg.springframework.boot.autoconfigure.EnableAutoConfiguration=\com.example.MyAutoConfiguration# Spring Boot 3.x — 不再加载上面的配置！# 必须使用新格式：# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.importscom.example.MyAutoConfiguration
```

> **迁移建议**：如果是 Spring Boot 3.x 项目，确保所有 Starter 都使用了 ```
> .imports
> ```
> 
>  格式。Spring Boot 官方 Starter 已全部迁移，但第三方 Starter 可能还没有。

### 误区 3：多个条件注解是 OR 关系

**错误认知**：多个 ```
@ConditionalOn*
```

 注解中只要有一个满足，Bean 就会被装配。

**正确理解**：多个条件注解之间是 **AND 关系**——**所有条件都必须满足**，Bean 才会被装配。

```
// ❌ 错误理解：以为只要满足任意一个条件就装配@Configuration@ConditionalOnClass(RedisTemplate.class)       // 条件 A@ConditionalOnProperty(prefix = "spring.redis", name = "enabled", havingValue = "true")  // 条件 B@ConditionalOnMissingBean(RedisTemplate.class)  // 条件 Cpublic class RedisAutoConfiguration {    // 实际行为：A AND B AND C 必须全部为 true 才装配    // 如果 A=true, B=false → 不装配！    // 如果 A=true, B=true, C=false → 不装配！}
```

### 误区 4：自动配置类的顺序可以随意修改

**错误认知**：自动配置类的加载顺序不重要，因为 Spring 会自动处理依赖。

**正确理解**：自动配置类之间有严格的依赖顺序，Spring Boot 通过拓扑排序保证顺序正确。手动修改顺序可能导致问题。

```
// 正确顺序（Spring Boot 保证）：// DataSourceAutoConfiguration (创建 DataSource)//     ↓ @AutoConfigureAfter// JdbcTemplateAutoConfiguration (依赖 DataSource)//     ↓ @AutoConfigureAfter// MyBatisAutoConfiguration (依赖 DataSource)// 如果手动打乱顺序：// JdbcTemplateAutoConfiguration 在 DataSourceAutoConfiguration 之前加载// → DataSource Bean 不存在 → JdbcTemplate 创建失败
```

```
// 正确写法@AutoConfigureAfter(DataSourceAutoConfiguration.class)public class JdbcTemplateAutoConfiguration {    // 依赖 DataSource，必须在 DataSource 之后加载}
```

### 误区 5：exclude 排除不生效

**问题现象**：排除了某个自动配置类，但启动时仍然报错或仍然加载了该配置。

**可能原因**：

原因解决方案使用了 ```
spring.factories
```

 而非 ```
.imports
```

（Spring Boot 3.x）升级 Starter 使用新格式exclude 的类路径写错了检查类的全限定名被其他 Starter 隐式引入（传递依赖）使用 Maven ```
mvn dependency:tree
```

 排查多个配置类都引入了同一个 Bean需要逐个排查```
@ConditionalOnMissingBean
```

 的搜索范围问题确保用户 Bean 在正确的作用域**排查步骤**：

```
# 1. 启用 debug 模式，查看 CONDITIONS EVALUATION REPORTjava -jar app.jar --debug# 2. 查看自动配置报告中的 Negative matches# 找到被排除的配置类，确认是否真的被排除# 3. 检查依赖树mvn dependency:tree | grep xxx# 4. 使用 Actuator 端点GET /actuator/conditions
```

## 十三、最佳实践

### 13.1 自定义配置优先

```
// 用户自定义配置（优先级高于自动配置）@Configurationpublic class UserConfig {    // 这个 Bean 会覆盖自动配置的 Bean    // 因为自动配置使用了 @ConditionalOnMissingBean    @Bean    public MyService myService() {        return new CustomMyService();    }}
```

**原理**：```
@ConditionalOnMissingBean
```

 在容器刷新时检查是否有该类型的 Bean。如果用户定义的 Bean 先被处理，自动配置就会跳过。

### 13.2 避免的陷阱

```
// ❌ 避免：自动配置类没有条件注解@Configurationpublic class BadAutoConfiguration {    @Bean    public MyService myService() {        return new MyService();  // 无条件创建，可能和用户的 Bean 冲突    }}// ✅ 推荐：加上条件注解@Configuration@ConditionalOnClass(MyService.class)public class GoodAutoConfiguration {    @Bean    @ConditionalOnMissingBean    public MyService myService() {        return new MyService();    }}
```

### 13.3 使用 @ConditionalOnMissingClass

```
@Configurationpublic class BackupAutoConfiguration {    @Bean    @ConditionalOnMissingClass("com.example.AdvancedService")    public MyService backupService() {        // 当没有高级服务时，提供备用实现        return new BasicService();    }}
```

### 13.4 proxyBeanMethods = false 优化

```
// ✅ Spring Boot 2.7+ 推荐：proxyBeanMethods = false@Configuration(proxyBeanMethods = false)  // 使用 CGLIB 代理的开销public class MyAutoConfiguration {    // 如果 Bean 之间没有相互调用，设置 proxyBeanMethods = false    // 可以跳过 CGLIB 代理创建，提升启动速度}
```

## 十四、面试自测

- •**Q1**：什么是 Spring Boot 自动装配？用一句话概括核心原理。
- •**Q2**：@SpringBootApplication 由哪三个注解组成？各自的作用是什么？
- •**Q3**：@EnableAutoConfiguration 是如何触发自动装配的？（从 @Import 说起）
- •**Q4**：AutoConfigurationImportSelector.selectImports() 的四步流程是什么？
- •**Q5**：SpringFactoriesLoader 的加载机制是什么？Spring Boot 3.x 和 2.x 有什么区别？
- •**Q6**：.imports 文件的完整路径是什么？格式和 spring.factories 有什么区别？
- •**Q7**：Condition 接口的 matches() 方法签名是什么？返回值 ConditionOutcome 包含什么信息？
- •**Q8**：SpringBootCondition 抽象类的作用是什么？（日志记录、匹配追踪、FailureAnalysis）
- •**Q9**：@ConditionalOnClass 底层如何检查 classpath？（Class.forName + initialize=false）
- •**Q10**：@ConditionalOnBean 的 Bean 搜索策略是什么？（当前工厂 vs 祖先工厂）
- •**Q11**：@ConditionalOnProperty 的四个参数如何组合使用？（prefix, name, havingValue, matchIfMissing）
- •**Q12**：多个条件注解之间是什么关系？为什么？
- •**Q13**：@AutoConfigureOrder、@AutoConfigureBefore、@AutoConfigureAfter 的区别是什么？
- •**Q14**：AutoConfigurationSorter 的拓扑排序算法如何实现？（Inverter、Kahn 算法）
- •**Q15**：如何查看自动配置报告？正/负匹配分别代表什么？
- •**Q16**：如何排除自动配置？有哪三种方式？
- •**Q17**：如何自定义一个 Starter？需要哪些步骤？
- •**Q18**：Spring Boot 3.x 移除了 spring.factories 的 AutoConfiguration 支持，如何迁移？
- •**Q19**：@ConditionalOnClass 和 @ConditionalOnBean 的区别是什么？
- •**Q20**：proxyBeanMethods = false 的作用是什么？为什么推荐设置？

👉 [Spring Boot 自动装配原理全解析（面试自测与答案）]

## 十五、总结

**自动装配核心要点**：

**What（是什么）**：

- • 自动装配是 Spring Boot 根据 classpath 依赖、配置属性等条件，自动注册 Bean 的机制
- • 核心入口是 ```
@EnableAutoConfiguration
```

，通过 ```
@Import
```

 导入 ```
AutoConfigurationImportSelector
```

**Why（为什么）**：

- • 解决传统 Spring 配置繁琐、版本兼容困难、重复劳动多、入门门槛高的问题
- • 核心理念：约定大于配置，按需装配，用户优先

**How（怎么做）**：

- • 四步流程：```
getCandidateConfigurations
```

 → ```
removeDuplicates
```

 → ```
filter
```

 → ```
sort
```
- • 条件注解：```
@ConditionalOnClass
```

、```
@ConditionalOnMissingBean
```

、```
@ConditionalOnProperty
```

 等
- • 注册方式：```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```
- • 自定义 Starter：配置属性类 + 自动配置类 + ```
.imports
```

 注册

**核心思想**：约定大于配置，自动检测依赖，按需装配 Bean，用户自定义优先。

## 参考资料

- • Spring Boot 官方文档：Auto-configuration
- • Spring Boot 源码：```
AutoConfigurationImportSelector.java
```

、```
SpringFactoriesLoader.java
```
- • Spring Boot 源码：```
OnClassCondition.java
```

、```
OnBeanCondition.java
```

、```
OnPropertyCondition.java
```
- • Spring Boot 源码：```
AutoConfigurationSorter.java
```

（拓扑排序实现）
- • 《Spring Boot 实战》第 3 章：自动配置原理
- • 《Spring Boot 编程思想》第 8 章：Spring Boot 自动装配机制
- • Spring Boot 3.x 迁移指南：从 spring.factories 迁移到 .imports

 






---
*Source: [WeChat Article](https://mp.weixin.qq.com/s/tfucgckSk2T1uw4878wp_w)*