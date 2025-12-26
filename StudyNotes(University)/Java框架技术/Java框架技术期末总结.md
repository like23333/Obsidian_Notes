# Java Spring/SSM 复习大纲

## 一、Spring基础部分

1. **Spring框架的核心概念**
   - IoC（控制反转）
   - DI（依赖注入）
   - AOP（面向切面编程）

2. **Spring核心容器**
   - Core
   - Beans
   - Context

3. **Spring框架支持编程模式**
   - MVC 「（Model-View-Controller）模型-视图-控制器」
   - ORM 「（Object-Relational Mapping）对象关系映射」
   - AOP 「（Aspect Oriented Programming）面向切面编程」
   
3. **Spring框架的优点**
   - 非侵入式设计
   - 支持声明式事务
   - 方便程序测试

5. **Spring框架的常见说法**
   - Spring是一个轻量级JAVA EE的框架集合
   - Spring通过IOC实现松耦合
   - Spring是一个包含且管理系统对象生命周期和配置的容器

6. **关于Spring特性中IoC描述**
   - **概念**：“控制反转”是指控制权由应用代码转到外部容器，即控制权的转移。
   - **职责**：IoC将控制创建的职责搬进了框架中，从应用代码脱离开来。
   - **使用**：使用Spring的IoC容器时只需指出组件需要的对象，在运行时Spring的IoC容器会根据XML配置数据提供给它。

7. **“依赖注入”常见的知识点**
   - 将组件间的依赖关系采取配置文件的方式管理，而不是硬编码在代码中。
   - 降低了组件间的耦合，使程序更容易维护和升级。
   - 促进了“面向接口”编程，使构建大规模程序更轻松。

8. **注入方式区别**
   - **Set注入**与**构造方法注入**的区别主要在于：**注入依赖关系时机不同**。

9. **Spring JDBC模块组成**
   - `object` 包
   - `core` 包
   - `support` 包
   - `dataSource` 包

10. **Spring事务管理的核心接口**
    - `PlatformTransactionManager`
    - `TransactionDefinition`
    - `TransactionStatus`

11. **自动装配**
    - Spring中 `@AutoWired` 注解默认的装配方式为 **byType**。

12. **Bean的初始化方法**
    - `init-method` 属性
    - `@PostConstruct` 注解
    - `InitializingBean` 接口
    - `@Bean` 注解的 `initMethod` 属性

13. **IoC容器类型**
    - 包括 `BeanFactory` 和 `ApplicationContext`。
    - `ApplicationContext` 在启动时自动加载并初始化所有Bean。

14. **概念辨析**
    - **控制反转 (IoC)**: 应用本身不负责依赖对象的创建及维护，由外部容器负责。控制权由应用转移到了外部容器。
    - **依赖注入 (DI)**: 在运行期，由外部容器动态地将依赖对象注入到组件中。

15. **@Component注解**
    - 表示泛化的概念，仅仅表示一个组件，可用于任何层次。

16. **关于BeanFactory**
    - 是一个接口。
    - 负责创建和管理Bean。
    - `ApplicationContext` 是 `BeanFactory` 的子接口。

17. **SpringMVC视图配置**
    - 前缀属性：`prefix`
    - 后缀属性：`suffix`

18. **@AutoWired 和 @Resource 的区别**
    - `@Autowired` 默认按照 **byType** 方式进行bean匹配，是 **Spring** 的注解。
    - `@Resource` 默认按照 **byName** 方式进行bean匹配，是 **J2EE** 的注解。

19. **Spring七大模块**
    1. **Core Container**：核心容器，负责Bean管理（BeanFactory）。
    2. **Context**：上下文，提供国际化等支持。
    3. **AOP**：面向切面编程。
    4. **DAO**：数据库访问。
    5. **ORM**：对象关系映射集成。
    6. **Web**：Web开发基础。
    7. **Web MVC**：MVC实现。

20. **Spring Boot自动配置机制**
    - 通过 `@EnableAutoConfiguration` 触发。
    - 核心文件：`META-INF/spring.factories`。
    - 扫描类路径jar包，使用 `@Conditional` 系列注解（如 `@ConditionalOnClass`）判断加载。
    - 遵循"约定优于配置"，可通过 `application.properties` 调整，或通过 `@SpringBootApplication` 排除特定配置。

21. **Spring Data JPA Repository进阶**
    - 核心接口：`CrudRepository`, `PagingAndSortingRepository`, `JpaRepository`。
    - 查询方式：方法名派生（`findByName`）、`@Query`、Querydsl。
    - 实体映射：`@Entity`, `@Table`, `@OneToMany`, `@ManyToOne`。
    - 性能优化：解决N+1问题（`@EntityGraph` 或 `FetchType.EAGER`）。

22. **配置属性绑定与类型安全配置**
    - 使用 `@ConfigurationProperties` 实现类型安全绑定（支持前缀匹配）。
    - 结合 `@EnableConfigurationProperties` 或 `@Component` 使用。
    - 验证：JSR-303注解（`@NotNull` 等）。
    - 特性：支持随机值 `${random.int}`，占位符解析，`@DurationUnit` 等特殊类型处理。

---

## 二、SpringMVC相关

1. **默认跳转模式**
   - 控制器默认返回值作为跳转地址时，模式是 `RequestDispatcher`（转发）。

2. **参数解析核心**
   - 负责请求参数解析和类型转换的核心组件是 `DataBinder`。

3. **复杂POJO绑定**
   - 请求参数名称格式要求：`属性名=属性值`。

4. **前端控制器**
   - `DispatcherServlet`。

5. **SpringMVC描述**
   - 基于Java实现的MVC设计模型的请求驱动类型轻量级WEB框架。
   - 融合在Spring Web Flow里，提供全功能MVC模块。
   - 使用可插入的MVC架构。

6. **常用注解**
   - `@SessionAttributes`, `@ModelAttribute`, `@CookieValue`, `@RequestHeader`
   - `@Controller`, `@RequestMapping`, `@PostMapping`, `@GetMapping`
   - `@ResponseBody`, `@RequestParam`, `@RequestBody`, `@PathVariable`

7. **日期处理注解**
   - `DateTimeFormat`
   - `JsonFormat`

8. **拦截器 (HandlerInterceptorAdapter)**
   - 方法：`preHandle`, `postHandle`, `afterCompletion`。
   - 注册：`InterceptorRegistry` 的 `addInterceptor` 方法。
   - 路径：`addPathPatterns`。

9. **核心类**
   - `org.springframework.web.servlet.DispatcherServlet`

10. **拦截器(Interceptor)与过滤器(Filter)的区别**
    | 特性 | 拦截器 (Interceptor) | 过滤器 (Filter) |
    | :--- | :--- | :--- |
    | **机制** | 基于反射机制 | 基于函数回调 |
    | **依赖** | 不依赖Servlet容器 | 依赖Servlet容器 |
    | **作用域** | 只对Action起作用 | 对几乎所有请求起作用 |
    | **上下文** | 可访问Action上下文/值栈 | 不行 |
    | **调用次数** | Action生命周期中可多次调用 | 容器初始化时调用一次 |
    | **Bean获取** | 可获取IOC容器中的Bean | 不行 |

11. **JSON绑定**
    - 常用注解：`@RequestBody`。

12. **重定向**
    - 前缀 `"redirect:"` 表示客户端重定向。

13. **@RequestMapping作用位置**
    - **类上**：第一级访问目录。
    - **方法上**：第二级访问目录。

14. **Spring Web模块常用组件**
    - WebSocket, Spring Web, Portlet, Servlet。

15. **注解功能**
    - `@Controller`：标识控制器。
    - `@RequestMapping`：映射URL。
    - `@RequestParam`：绑定参数（支持默认值）。

16. **视图解析与多模板**
    - `ViewResolver` 链支持混合视图。
    - `ContentNegotiatingViewResolver` 根据 Content-Type 选择视图（JSP/Thymeleaf/FreeMarker）。

17. **全局异常处理**
    - 使用 `@ControllerAdvice` 和 `@ExceptionHandler`。

18. **文件上传**
    - 配置 `MultipartResolver`。
    - 使用 `@RequestPart` 注解。

19. **跨域请求 (CORS)**
    - 注解：`@CrossOrigin`。
    - 注意：预检请求（OPTIONS），凭证模式与通配符冲突问题。

20. **配置方式对比**
    - **Java配置**：`@EnableWebMvc` + `WebMvcConfigurer`（类型安全）。
    - **XML配置**：`<mvc:annotation-driven>`。
    - **Boot配置**：`WebMvcAutoConfiguration`（约定优于配置）。

21. **性能优化**
    - 视图缓存（`setCache`）。
    - 结果缓存（`@Cacheable` + Redis）。
    - 静态资源缓存头（Cache-Control）。

---

## 三、SpringAOP相关

1. **常用注解**
   - `@Before`, `@Around`, `@AfterThrowing`

2. **全称**
   - Aspect Oriented Programming（面向切面编程）。

3. **XML切点定义**
   - 标签：`<aop:pointcut>`。

4. **Spring AOP优点**
   - 分离业务逻辑与横切关注点。
   - 提高代码可重用性。
   - 简化事务管理配置。
   - 降低系统耦合度，增强可维护性。
   - 自动生成代理对象。

5. **代理机制**
   - **JDK动态代理**：目标对象实现接口时默认使用。
   - **CGLIB代理**：目标对象无接口时使用。
   - 开启方式：`ProxyFactory` 或 `@EnableAspectJAutoProxy`。

6. **切点表达式 (execution)**
   - 支持通配符 `*`，类型模式，参数匹配 `(..)`，逻辑组合 `&&`, `||`。
   - 注意 `within` 和 `@annotation` 用法。

7. **环绕通知 (@Around)**
   - 参数：`ProceedingJoinPoint`。
   - 能力：修改参数、替换返回值、阻止执行、处理异常。

8. **切面顺序**
   - 控制：`@Order` 或 `Ordered` 接口。
   - 同一连接点：`@Before` 优先级升序执行，`@After` 优先级降序执行。

9. **加载时织入 (LTW)**
   - 机制：`-javaagent` + `META-INF/aop.xml`。
   - 适用：构造函数、静态方法等无法代理的场景。

10. **事务切面**
    - 实现：`@Transactional` + `TransactionAspectSupport`。
    - 关注：传播行为（如 `REQUIRES_NEW`）和回滚规则（`NoRollbackFor`）。

11. **最佳实践**
    - 避免过度使用；仅用于日志、安全、事务等横切点；注意切面粒度和性能。

---

## 四、MyBatis相关

1. **定义**
   - MyBatis是一种基于Java的 **O/R Mapping** 框架。

2. **多对多关联**
   - 需要借助 **第三张表** 来实现。

3. **主键生成 (MySQL)**
   - 使用 `<insert>` 标签中的 `useGeneratedKeys` 属性。

4. **主键生成 (Oracle/Select方式)**
   - 使用 `<selectKey>` 标签。

5. **支持数据库**
   - Oracle, MongoDB, MySQL, Access, SQLServer 等。

6. **连接池配置**
   - `url`, `username`, `password`, `poolMaximumActiveConnections`, `poolPingQuery` 等。

7. **动态SQL场景**
   - 动态查询条件拼接、批量插入/更新、条件更新字段选择。

8. **关联查询类型**
   - 一对一、一对多、多对多。

9. **标签对应关系**
   - 一对一：`<association>`
   - 一对多：`<collection>`
   - 查询：`<select>`

10. **Mapper XML标签**
    - `<sql>`, `<select>`, `<insert>`, `<update>`, `<delete>`

11. **注解映射**
    - `@Results` 定义多个映射规则，包含多个 `@Result`。

12. **插入注解**
    - `@Insert`，使用 `#{}` 占位符。

13. **@Options属性**
    - `useGeneratedKeys`, `keyProperty`, `timeout`, `flushCache`。

14. **Mybatis-config.xml 常用配置**
    - 延迟加载：`lazyLoadingEnabled` (true/false)。
    - 激进懒加载：`aggressiveLazyLoading` (false为按需加载)。
    - 二级缓存全局开关：`cacheEnabled`。
    - 事务管理器：`transactionManager` (type="JDBC")。

15. **二级缓存配置要点**
    1. `SqlSessionFactory` 级共享。
    2. 基于 `Mapper` 命名空间独立。
    3. 全局开启 (`cacheEnabled=true`) 且 Mapper 中声明 (`<cache/>`)。
    4. 配置回收策略、刷新间隔等。

16. **$ 和 # 的区别 (重点)**
    - **# (PreparedStatement)**: 转换成数据类型，防止SQL注入（占位符）。
    - **$ (Statement)**: 字符串原样拼接，不加数据类型（常用于表名/字段名）。

17. **操作标签**
    - 更新：`<update>`
    - 删除：`<delete>`

18. **注解**
    - 方法参数需用 `@Param` 指定占位符名称。

19. **结果映射**
    - `@Result`: `property` (实体属性) vs `column` (库字段)。

20. **逆向工程**
    - 生成：Java实体类、Mapper接口、XML映射文件。

21. **持久化与ORM**
    - **持久化**：内存数据与存储模型（DB/XML）的双向转换。
    - **ORM**：对象关系映射，解决对象模型与关系数据库模型不匹配的问题。

22. **一级缓存完全相同查询的判断条件**
    1. `statementId` 相同。
    2. 结果集范围相同。
    3. 生成的 SQL 语句字符串 (`boundSql.getSql()`) 相同。
    4. 传递给 Statement 的参数值相同。

23. **循环遍历标签 (<foreach>) 参数**
    - `collection`: 集合类型。
    - `item`: 元素变量名。
    - `open`/`close`: 开始/结束符号。
    - `separator`: 分隔符。

---

## 五、SSM框架整合相关

1. **整合步骤**
   2. 引入依赖
   3. 配置数据源
   4. 扫描Mapper
   5. 配置视图解析器
   6. 设置事务管理器
   7. 编写测试

8. **常用注解**
   - 指定配置文件：`@PropertySource`
   - 声明配置类：`@Configuration`
   - 扫描组件：`@ComponentScan`
   - 加载XML：`@ImportResource`

3. **Spring Boot整合优势**
   - 简化配置（自动配置）。
   - 内嵌容器（Tomcat等）。
   - 依赖管理（Starter）。
   - 生产就绪特性（监控等）。

4. **全局配置文件**
   - `application.properties`
   - `application.yml`

5. **常用Starter**
   - `spring-boot-starter-web`
   - `spring-boot-starter-data-jpa`
   - `spring-boot-starter-data-redis`
   - `spring-boot-starter-data-solr`
   - `mybatis-spring-boot-starter`

6. **分工**
   - **MyBatis**: 数据库交互。
   - **Spring**: Bean管理和事务。
   - **Spring MVC**: 处理HTTP请求。

---

## 六、编程范围

1. **文件操作**
   - 使用SSM技术实现文件的上传和下载。

2. **业务逻辑**
   - Service层的编写以及与Mapper的联系。

3. **邮件功能**
   - 使用 `JavaMail` 进行邮件发送。

4. **动态SQL**
   - 在注解方式中调用MyBatis动态SQL。