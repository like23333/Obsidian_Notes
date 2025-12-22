# MyBatis & Spring MVC 核心题目整理（共 50 题）

## 第一部分：题目与选项


### 题目 1：MyBatis 中注解方式与 XML 映射文件方式的适用场景，说法正确的是？
选项：
A. 注解方式适合复杂 SQL 查询
B. 注解和 XML 在复杂查询场景下灵活性无差异
C. 复杂查询场景下，注解方式比 XML 更优，仅取决于需求和喜好
D. XML 映射文件的动态 SQL 标签更适合复杂查询的灵活拼接

### 题目 2：MyBatis 在未指定返回类型（如未配置 resultType 或 resultMap）时，默认查询结果封装为？
选项：
A. JavaBean
B. List<Map<String, Object>>
C. Map<String, Object>
D. List<JavaBean>

### 题目 3：关于 MyBatis 中 # 和 $ 的说法，错误的是？
选项：
A. $ 会直接将参数拼接为字符串，不做数据类型转换
B. # 会不加数据类型地将参数代入 SQL
C. # 会将参数转换为对应数据类型后代入 SQL
D. $ 会把字符串原样拼接进 SQL

### 题目 4：Spring MVC 的复杂 POJO 绑定中，请求参数的正确格式是？
选项：
A. 属性名：属性值
B. 属性名 = 属性值（嵌套对象为 “嵌套对象.属性名 = 属性值”）
C. 属性名 - 属性值
D. 属性名 [键]= 属性值

### 题目 5：关于 HTTP 的 GET 与 POST 请求，说法错误的是？
选项：
A. GET 通过 URL 传参有长度限制，POST 通过请求体传参无数据量限制
B. POST 请求的页面可以用邮件发送，GET 则不可以
C. GET 参数暴露在 URL 中，POST 参数在请求体中更安全（需配合 HTTPS）
D. GET 请求的 URL 可作为书签，POST 无法直接保存

### 题目 6：Spring MVC 中 @RequestMapping("/empl/{emplId}")，获取路径参数 emplId 的正确方式是？
选项：
A. @RequestParam String emplId
B. @PathVariable("emplId") String id
C. 方法参数直接写 String emplId
D. @RequestParam(required = false) String emplId

### 题目 7：关于 MyBatis 的 @Insert、@Update、@Delete 注解，说法错误的是？
选项：
A. 支持传入单个字符串或字符串数组作为 SQL 内容
B. 传入字符串数组时，MyBatis 会将数组元素按空格拼接为完整 SQL
C. 注解用于直接指定要执行的真实 SQL 语句
D. 注解通过指定类名和方法来返回 SQL 语句

### 题目 8：MyBatis 的 association 标签用于表示哪种关系？
选项：
A. 一对多关系
B. 一对一和一对多关系
C. 多对多关系
D. 一对一关系

### 题目 9：关于 MyBatis 的适用场景，说法错误的是？
选项：
A. 跨数据库兼容性弱，需多数据库支持时不如 Hibernate
B. 对象持久化并非完全透明，需要手动编写 SQL
C. 不适合完全动态的 SQL 场景
D. 追求极致性能时，可直接使用 JDBC 而非 MyBatis

### 题目 10：MyBatis 中可实现动态生成 SQL 的标签有？
选项：
A. choose 标签
B. if 标签
C. foreach 标签
D. 以上都是

### 题目 11：关于依赖注入（DI）的说法，错误的是？
选项：
A. 降低组件耦合，便于维护升级
B. 依赖关系通过配置管理，而非硬编码
C. 促进面向接口编程，易构建大规模程序
D. 强制要求定义大量接口，增加了开发复杂度

### 题目 12：以下哪个不是 MyBatis 的操作注解？
选项：
A. @Update
B. @Insert
C. @Request
D. @Delete

### 题目 13：Spring MVC 中用于 JSON 数据绑定的注解是？
选项：
A. @DateTimeFormat
B. @RequestBody
C. @PathVariable
D. @ResponseBody

### 题目 14：MyBatis 中绑定输入 / 输出参数的语法是？
选项：
A. $paramName
B. :paramName
C. #{paramName}
D. 通过 Mapper 配置文件直接指定

### 题目 15：Spring MVC 中接收请求路径 “/logic/findById” 的正确方式是？
选项：
A. 仅在 Controller 类上添加 @RequestMapping("/logic/findById")
B. 在 Controller 方法上添加 @RequestMapping("/logic/findById")
C. 在方法体内使用 @RequestMapping("/logic/findById")
D. 通过 @Controller("/logic/findById") 指定路径

### 题目 16：MyBatis 中配置日志功能的正确方式是？
选项：
A. 在 SQL 语句前添加 “LOG:” 标识
B. 在 mybatis-config.xml 中通过 <settings> 配置日志实现
C. 使用 @Log 注解标注 Mapper 接口
D. 手动编写日志输出语句

### 题目 17：MyBatis 中通过 @Insert 注解插入数据后，获取新生主键的方式是？
选项：
A. 调用数据库存储过程获取
B. 配合 @Options(useGeneratedKeys = true, keyProperty = "主键属性名")
C. 在 SQL 语句中手动编写获取主键的逻辑
D. 以上都不对

### 题目 18：MyBatis 的 <foreach> 标签主要用于处理？
选项：
A. 结果集的映射关系
B. 数据库连接的创建与释放
C. 动态 SQL 的拼接（如遍历集合构建 IN 条件）
D. 仅用于批量插入、更新操作

### 题目 19：MyBatis 支持的参数类型有？
选项：
A. 单一值参数、JavaBean、Map
B. 仅支持 JavaBean 参数
C. 仅支持 Map 参数
D. 仅支持单一值参数

### 题目 20：Spring 中 Bean 的作用域默认是？
选项：
A. prototype
B. singleton
C. request
D. session

### 题目 21：MyBatis 中 resultMap 标签的主要作用是？
选项：
A. 定义 SQL 语句
B. 映射查询结果到 Java 对象
C. 配置数据库连接
D. 开启缓存

### 题目 22：Spring MVC 中 @ResponseBody 注解的作用是？
选项：
A. 接收请求体中的数据
B. 将方法返回值转为 JSON/XML 等响应给客户端
C. 获取路径参数
D. 标识控制器

### 题目 23：MyBatis 的一级缓存作用域是？
选项：
A. 全局
B. Session
C. Statement
D. 应用级

### 题目 24：Spring 中实现自动装配的注解是？
选项：
A. @Component
B. @Autowired
C. @Repository
D. @Service

### 题目 25：MyBatis 中 @Select 注解的作用是？
选项：
A. 定义插入 SQL
B. 定义更新 SQL
C. 定义查询 SQL
D. 定义删除 SQL

### 题目 26：Spring MVC 中处理文件上传的注解是？
选项：
A. @RequestParam
B. @RequestPart
C. @PathVariable
D. @RequestBody

### 题目 27：MyBatis 中开启二级缓存的关键配置是？
选项：
A. 在 mybatis-config.xml 中设置 cacheEnabled=true，并在 Mapper 中添加 <cache/>
B. 仅在 mybatis-config.xml 中设置 cacheEnabled=true
C. 仅在 Mapper 中添加 <cache/>
D. 无需配置，默认开启

### 题目 28：Spring 中 @Component 注解的作用是？
选项：
A. 标识控制器类
B. 标识服务类
C. 标识数据访问类
D. 通用的 Bean 标识，将类纳入 Spring 容器管理

### 题目 29：MyBatis 中 trim 标签的 prefix 属性作用是？
选项：
A. 去除前缀字符
B. 为拼接的 SQL 片段添加前缀
C. 去除后缀字符
D. 为拼接的 SQL 片段添加后缀

### 题目 30：Spring MVC 的核心前端控制器是？
选项：
A. DispatcherServlet
B. ContextLoaderListener
C. HandlerAdapter
D. HandlerMapping

### 题目 31：MyBatis 中 typeAlias 标签的作用是？
选项：
A. 为数据库表名起别名
B. 为 Java 类型起别名
C. 为 SQL 语句起别名
D. 为 Mapper 接口起别名

### 题目 32：Spring 中 @Scope 注解用于设置 Bean 的？
选项：
A. 生命周期
B. 作用域
C. 初始化方法
D. 销毁方法

### 题目 33：MyBatis 中 collection 标签用于表示哪种关系？
选项：
A. 一对一
B. 一对多
C. 多对多
D. 以上都不对

### 题目 34：Spring MVC 中 @RequestParam 的 defaultValue 属性作用是？
选项：
A. 设置参数是否必填
B. 设置参数名
C. 设置参数的默认值
D. 转换参数类型

### 题目 35：MyBatis 中配置分页插件的核心步骤是？
选项：
A. 在 mybatis-config.xml 中配置插件拦截器
B. 仅在 Mapper 中编写分页 SQL
C. 通过 Java 代码手动分页
D. 无需配置，MyBatis 自带分页功能

### 题目 36：Spring 中 @PostConstruct 注解的作用是？
选项：
A. 标注 Bean 的初始化方法
B. 标注 Bean 的销毁方法
C. 实现自动装配
D. 标识 Bean

### 题目 37：MyBatis 中 where 标签的作用是？
选项：
A. 替代 SQL 中的 WHERE 关键字，自动处理多余的 AND/OR
B. 仅添加 WHERE 关键字
C. 定义查询条件
D. 过滤结果集

### 题目 38：Spring MVC 中 HandlerMapping 的作用是？
选项：
A. 执行处理器方法
B. 映射请求到对应的处理器
C. 转换请求参数
D. 处理视图渲染

### 题目 39：MyBatis 中 jdbcType 属性的作用是？
选项：
A. 指定 Java 类型
B. 指定数据库字段类型
C. 指定参数的 JDBC 类型
D. 指定结果集类型

### 题目 40：Spring 中 @Qualifier 注解的作用是？
选项：
A. 按类型装配 Bean
B. 按名称装配 Bean，解决同类型多个 Bean 的冲突
C. 标识 Bean 的名称
D. 开启自动装配

### 题目 41：MyBatis 中 set 标签的作用是？
选项：
A. 用于 INSERT 语句的字段设置
B. 用于 UPDATE 语句，自动处理多余的逗号
C. 定义查询条件
D. 拼接 SQL 片段

### 题目 42：Spring MVC 中 ViewResolver 的作用是？
选项：
A. 解析请求路径
B. 解析视图名，返回对应的视图对象
C. 处理请求参数
D. 执行处理器

### 题目 43：MyBatis 中 plugins 标签的作用是？
选项：
A. 配置数据库连接池
B. 配置插件，如分页、拦截器等
C. 配置缓存
D. 配置类型别名

### 题目 44：Spring 中 @Value 注解的作用是？
选项：
A. 注入 Bean 的属性值，支持读取配置文件
B. 标识 Bean 的名称
C. 设置 Bean 的作用域
D. 实现自动装配

### 题目 45：MyBatis 中 association 标签的 select 属性作用是？
选项：
A. 指定关联查询的 SQL 语句
B. 指定关联查询的 Mapper 方法
C. 定义关联对象的类型
D. 映射关联对象的属性

### 题目 46：Spring MVC 中 @GetMapping 注解是哪种请求方式的简化？
选项：
A. POST
B. GET
C. PUT
D. DELETE

### 题目 47：MyBatis 中 cache-ref 标签的作用是？
选项：
A. 引用其他 Mapper 的缓存配置
B. 开启自身缓存
C. 关闭缓存
D. 设置缓存过期时间

### 题目 48：Spring 中 @Transactional 注解的作用是？
选项：
A. 开启事务管理
B. 标注事务方法，实现声明式事务
C. 配置事务传播行为
D. 配置事务隔离级别

### 题目 49：MyBatis 中 parameterType 属性的作用是？
选项：
A. 指定返回值类型
B. 指定参数的 Java 类型
C. 指定数据库类型
D. 指定结果集类型

### 题目 50：Spring MVC 中 @ExceptionHandler 注解的作用是？
选项：
A. 处理请求参数异常
B. 全局处理异常
C. 处理控制器中的异常，实现局部异常处理
D. 配置异常页面

---

## 第二部分：答案与解析

### 答案 1：D
解析：XML 映射文件支持动态 SQL 标签，比注解更适合处理复杂的逻辑拼接。

### 答案 2：B
解析：MyBatis 在未配置返回类型时，默认会将结果集封装为 `List<Map<String, Object>>`。

### 答案 3：B
解析：`#` 会进行预编译并根据 Java 类型进行数据转换，防止 SQL 注入；而 `$` 是直接拼接。

### 答案 4：B
解析：Spring MVC 绑定复杂 POJO 或嵌套对象时，使用 `.` 语法。

### 答案 5：B
解析：GET 请求参数包含在 URL 中，方便分享和收藏；POST 依赖请求体，无法直接通过简单 URL 分发。

### 答案 6：B
解析：从 URL 路径占位符中获取值必须使用 `@PathVariable`。

### 答案 7：D
解析：Provider 类注解（如 `@InsertProvider`）才通过指定类名和方法返回 SQL，基础注解直接写 SQL。

### 答案 8：D
解析：`association` 对应一对一关系映射，`collection` 对应一对多关系。

### 答案 9：C
解析：MyBatis 特别适合动态 SQL，这是其相比 Hibernate 等框架的主要优势之一。

### 答案 10：D
解析：`choose`, `if`, `foreach` 都是 MyBatis 核心的动态 SQL 生成标签。

### 答案 11：D
解析：依赖注入（DI）是为了解耦，并不是强制要求增加接口数量，也不会必然增加开发复杂度。

### 答案 12：C
解析：MyBatis 的核心 CRUD 注解是 `@Select`, `@Insert`, `@Update`, `@Delete`。

### 答案 13：B
解析：`@RequestBody` 用于将请求体中的 JSON 数据反序列化为 Java 对象。

### 答案 14：C
解析：`#{}` 是预编译参数绑定的标准语法。

### 答案 15：B
解析：`@RequestMapping` 标注在方法上用于指定具体访问的逻辑路径。

### 答案 16：B
解析：日志设置属于全局配置，应在 `mybatis-config.xml` 的 `<settings>` 中配置。

### 答案 17：B
解析：配合 `@Options(useGeneratedKeys = true, keyProperty = "...")` 实现主键回填。

### 答案 18：C
解析：`<foreach>` 用于遍历集合拼接 SQL（如 `IN` 查询）。

### 答案 19：A
解析：MyBatis 对输入参数类型支持非常广泛，包括基本类型、Bean 及 Map。

### 20：B
解析：Spring 的默认 Bean 作用域是单例（Singleton）。

### 答案 21：B
解析：`resultMap` 主要解决数据库字段名与实体类属性名不一致及关联查询问题。

### 答案 22：B
解析：`@ResponseBody` 将方法返回的对象转化为 JSON/XML 等格式直接写入响应体。

### 答案 23：B
解析：一级缓存是 `SqlSession` 级别的，二级缓存是 `Namespace` 级别的。

### 答案 24：B
解析：`@Autowired` 是 Spring 实现自动依赖注入的标准注解。

### 答案 25：C
解析：`@Select` 对应查询，`@Insert` 对应插入，以此类推。

### 答案 26：B
解析：`@RequestPart` 用于处理文件上传或复杂的 Multipart 请求数据。

### 答案 27：A
解析：开启二级缓存需在全局配置启用 `cacheEnabled`（默认开）并在 Mapper.xml 中加入 `<cache/>`。

### 答案 28：D
解析：`@Component` 是泛指组件，当类不好归类时使用此注解。

### 答案 29：B
解析：`trim` 标签的 `prefix` 属性用于在内容不为空时添加前缀。

### 答案 30：A
解析：`DispatcherServlet` 负责拦截并分发所有的 HTTP 请求。

### 答案 31：B
解析：`typeAlias` 用于在配置文件中简化 Java 类型的全限定类名引用。

### 答案 32：B
解析：`@Scope` 用于设置 Bean 是单例、多例还是其他 Web 相关作用域。

### 答案 33：B
解析：`collection` 用于映射 Java 对象中的集合属性（一对多）。

### 答案 34：C
解析：`defaultValue` 当请求中未携带该参数名时，自动使用设置的默认值。

### 答案 35：A
解析：分页插件（如 PageHelper）需要在 MyBatis 配置文件中注册拦截器。

### 答案 36：A
解析：`@PostConstruct` 标注的方法会在依赖注入完成后、Bean 初始化时执行。

### 答案 37：A
解析：`where` 标签能智能地在需要时插入 `WHERE` 并删除多余的 `AND/OR`。

### 答案 38：B
解析：`HandlerMapping` 负责根据请求的 URL 找到对应的处理器（Handler）。

### 答案 39：C
解析：`jdbcType` 在参数为 null 时尤为重要，帮助数据库识别字段类型。

### 答案 40：B
解析：当存在多个相同类型的 Bean 时，`@Qualifier` 按名称区分要注入的 Bean。

### 答案 41：B
解析：`set` 标签用于更新操作，能自动去除最后一个字段后多余的逗号。

### 答案 42：B
解析：`ViewResolver` 负责将逻辑视图名解析为具体的视图实现。

### 答案 43：B
解析：`plugins` 标签用于注册 MyBatis 的拦截器插件。

### 答案 44：A
解析：`@Value` 用于注入配置文件中的值或 SpEL 表达式结果。

### 答案 45：B
解析：`association` 的 `select` 属性用于触发另一个 SQL 查询来获取关联对象（嵌套查询）。

### 答案 46：B
解析：`@GetMapping` 是专门处理 HTTP GET 请求的快捷注解。

### 答案 47：A
解析：`cache-ref` 使得不同的 Mapper 能够共享同一个二级缓存。

### 答案 48：B
解析：`@Transactional` 通过 AOP 实现声明式事务，控制业务逻辑的原子性。

### 答案 49：B
解析：`parameterType` 显式声明传入参数的 Java 类型。

### 答案 50：C
解析：`@ExceptionHandler` 用于捕获该控制器中抛出的异常并进行统一处理。
