# MyBatis-Plus 报错速查

按**错误信息 / 现象**定位。先在这里找，再去改代码。每条按 现象 → 原因 → 修复 组织。

---

## `Invalid bound statement (not found)`

**最高频的报错**，九成不是 MP 的问题，而是 Mapper 没被扫描到 / XML 没被打进包。逐条排查：

1. **检查 jar 包冲突。** ⚠️ 引入 MyBatis-Plus 后**不要再引入** `mybatis`、`mybatis-spring-boot-starter`、`mybatis-spring`（starter 已传递）。

2. **检查 `Mapper.java` 的扫描路径。** 两种方式二选一：

```java
@Configuration
@MapperScan("com.yourpackage.*.mapper")
public class YourConfigClass{
  ...
}
```

```java
@Bean
public MapperScannerConfigurer mapperScannerConfigurer(){
    MapperScannerConfigurer scannerConfigurer = new MapperScannerConfigurer();
    // 可以通过环境变量获取你的 mapper 路径，这样 mapper 扫描可以通过配置文件配置了
    scannerConfigurer.setBasePackage("com.yourpackage.*.mapper");
    return scannerConfigurer;
}
```

3. **检查是否指定了主键。** 未指定会导致 `selectById` 相关 ID 无法操作。用 `@TableId` 标注；⚠️ 注解可以省略，但**主键字段必须叫 `id`**（忽略大小写）。

4. **不要使用原生的 `SqlSessionFactory`，请使用 `MybatisSqlSessionFactory`。**

5. **检查是否自定义了 `SqlInjector`** 并复写了 `getMethodList()` —— 该方法里是否注入了你需要的方法（参考 `DefaultSqlInjector`）。

6. **IDEA 的默认 build 不会把 `src/main/java` 下的 XML 拷到 `resources`。** 检查 build 产物里有没有对应 XML。修复：把 build 方式调整为 Maven / Gradle。

## Kotlin 调用接口 default 方法出现 `Invalid bound statement (not found)`

**原因**：Kotlin 1.4+ 生成 interface default 方法的方式变了（见 JetBrains 官方博客）。

**修复**：配置编译插件参数 `-Xjvm-default=all`。

```xml
<plugin>
     <groupId>org.jetbrains.kotlin</groupId>
     <artifactId>kotlin-maven-plugin</artifactId>
     <version>${kotlin.version}</version>
     <configuration>
        <args>
            <arg>-Xjsr305=strict</arg>
            <arg>-Xjvm-default=all</arg>
        </args>
     </configuration>
</plugin>
```

```groovy
compileKotlin {
    kotlinOptions {
        freeCompilerArgs = ['-Xjvm-default=all']
    }
}
```

## 自定义 SQL（XML 里写的方法）无法执行

**原因**：和原生 MyBatis 一样需要配置 XML 扫描路径，MP 不会帮你猜。

Spring Boot：

```yaml
mybatis-plus:
  mapper-locations: classpath*:/mapper/**/*.xml
```

Spring MVC：

```xml
<bean id="sqlSessionFactory" class="com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource" />
    <property name="typeAliasesPackage" value="xxx.entity" />
    <property name="mapperLocations" value="classpath*:/mybatis/*/*.xml"/>
    ...
</bean>
```

⚠️ **IDEA 系列编辑器不会编译 `java` 源码文件夹里的 XML。** 要么把 XML 放进 `resources`，要么在 POM 里显式声明 resource：

```xml
<build>
  <resources>
      <resource>
          <!-- xml放在java目录下-->
          <directory>src/main/java</directory>
          <includes>
              <include>**/*.xml</include>
          </includes>
      </resource>
      <!--指定资源的位置（xml放在resources下，可以不用指定）-->
      <resource>
          <directory>src/main/resources</directory>
      </resource>
  </resources>
</build>
```

⚠️ **Maven 多模块项目的扫描路径必须以 `classpath*:` 开头**（才能加载多个 jar 包下的 XML）。

## `java.lang.ClassCastException: sun.reflect.generics.reflectiveObjects.TypeVariableImpl cannot be cast to java.lang.Class`

**原因**：`MapperScan` 扫到了 `BaseMapper` 本身或你的自定义公共 Mapper 父接口。

**修复**：`MapperScan` 需排除 `com.baomidou.mybatisplus.mapper.BaseMapper` 类及其子类，比如：

```java
import com.baomidou.mybatisplus.core.mapper.BaseMapper;


public interface SuperMapper<T> extends BaseMapper<T> {
    // your methods
}
```

## 启动报 `Injection of autowired`

低版本 Spring 不支持泛型注入。升级 Spring 到 **4+**。

## `java.lang.NoSuchMethodError: org.apache.ibatis.session.Configuration.getDefaultScriptingLanguageInstance()`

版本引入问题：该方法 **3.4.1 里没有，3.4.2 才有**。对齐 MyBatis 版本。

## 实体里有非表字段，报字段不存在

三种方式选一即可：

```java
private transient String noColumn;
```

```java
private static String noColumn;
```

```java
@TableField(exist=false)
private String noColumn;
```

**排除实体父类属性**：用 `transient` 修饰需要排除的父类属性。

```java
/**
 * 忽略父类 createTime 字段映射
 */
private transient String createTime;
```

## `MybatisPlusException: Your property named "xxx" cannot find the corresponding database column name!`

**版本**：3.1.1+。**现象**：单元测试正常，启动服务器调试时抛异常。

**原因**：3.1.1 起字段缓存优化，用 `.class` 作键替代了类名（className）。而 **dev-tools** 会用不同的类加载器加载 `.class`，导致找不到属性。

**修复**：**移除 `spring-boot-devtools` 插件。**

## Long 型主键 / 主键填充不生效

**检查是不是用了 `long` 而不是 `Long`。** `long` 默认值为 `0`，而 MP 只判断是否为 `null`。

## 生成的主键太长，前端 JS 精度丢失（末两位永远是 0）

JavaScript 无法处理 Java 的长整型 `Long`。思路统一是 **Long 转 String 返回**。

FastJson：

```java
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    FastJsonHttpMessageConverter fastJsonConverter = new FastJsonHttpMessageConverter();
    FastJsonConfig fjc = new FastJsonConfig();
    // 配置序列化策略
    fjc.setSerializerFeatures(SerializerFeature.BrowserCompatible);
    fastJsonConverter.setFastJsonConfig(fjc);
    converters.add(fastJsonConverter);
}
```

Jackson 方式一（注解，可配在公共 baseEntity 上）：

```java
// 注解处理，这里可以配置公共 baseEntity 处理
@JsonSerialize(using=ToStringSerializer.class)
public long getId() {
    return id;
}
```

Jackson 方式二（全局）：

```java
// 全局配置序列化返回 JSON 处理
final ObjectMapper objectMapper = new ObjectMapper();
SimpleModule simpleModule = new SimpleModule();
simpleModule.addSerializer(Long.class, ToStringSerializer.instance);
objectMapper.registerModule(simpleModule);
```

比较一般的处理方式：增加一个 `public String getIdStr()` 方法，前台获取 `idStr`。

## 更新 / 插入 null 或空字符串不生效（字段被静默跳过）

**原因**：`FieldStrategy` 默认 `NOT_NULL`，null 值字段不会进 SQL。三种策略：`IGNORED`（忽略）、`NOT_NULL`（非 NULL，默认）、`NOT_EMPTY`（非空）。

**修复方式一**：调整全局策略 —— 注入 `GlobalConfiguration` 的 `fieldStrategy` 属性。

**修复方式二**：调整字段验证注解：

```java
@TableField(strategy=FieldStrategy.NOT_EMPTY)
```

**修复方式三（推荐）**：用 `UpdateWrapper` 显式 `set`：

```java
//updateAllColumnById(entity) // 全部字段更新: 3.0已经移除
mapper.update(
   new User().setName("mp").setAge(3),
   Wrappers.<User>lambdaUpdate()
           .set(User::getEmail, null) //把email设置成null
           .eq(User::getId, 2)
);
//也可以参考下面这种写法
mapper.update(
    null,
    Wrappers.<User>lambdaUpdate()
       .set(User::getAge, 3)
       .set(User::getName, "mp")
       .set(User::getEmail, null) //把email设置成null
       .eq(User::getId, 2)
);
```

## `Cause: org.apache.ibatis.type.TypeException: Error setting null for parameter #1 with JdbcType OTHER`

完整报错常伴随 `Cause: java.sql.SQLException: 无效的列类型: 1111`（典型 Oracle）。

**原因**：字段为 `null` 时 MP 默认映射成 `JdbcType.OTHER`，而某些 JDBC 驱动（如 Oracle）不支持。

**修复**：设置 `jdbcTypeForNull`（任选其一）。

```java
MybatisConfiguration configuration = new MybatisConfiguration();
configuration.setDefaultScriptingLanguage(MybatisXMLLanguageDriver.class);
configuration.setJdbcTypeForNull(JdbcType.NULL);
sqlSessionFactory.setConfiguration(configuration);
```

```yaml
mybatis-plus:
  configuration:
    jdbc-type-for-null: 'null'
```

```properties
mybatis-plus.configuration.jdbc-type-for-null=NULL
```

```xml
<configuration>
    <settings>
        <setting name="jdbcTypeForNull" value="NULL" />
    </settings>
</configuration>
```

⚠️ 如果你在项目中**自定义了 `SqlSessionFactory`，`jdbc-type-for-null` 配置可能失效**。

## SQL 里出现 2 个 limit 语句

**原因**：配了 2 个分页拦截器。检查配置文件和代码，**只留一个**。

## 分页 / 自定义 SQL：无法通过 `Page` 对象传递参数

**原因**：`Page` 继承自 `RowBounds`，在 Mapper 中无法直接获取其属性。

**修复**：
- 使用自定义的 Map 对象或普通 Java 对象来传递参数；
- 或在方法参数中用 `@Param("page") int page, @Param("size") int size` 传页码和每页大小。

## `Page` 或 `List` 返回的结果集为 null / 含部分 null 元素

**原因**：MyBatis 默认在返回行的所有列都为空时（含属性无法自动映射）返回 `null`。

```yaml
mybatis-plus:
    configuration:
        return-instance-for-empty-row: true
```

## 返回 `Map` / `List<Map>` 时，null 值对应的 key 丢失

**原因**：MyBatis 默认在值为 null 时不调用 `put`。

```yaml
mybatis-plus:
    configuration:
        call-setters-on-nulls: true
```

## `resultType="java.util.Map"` 时下划线不转驼峰

```java
@Configuration
public class MybatisConfigurationCustomizer {
    @Bean
    public ConfigurationCustomizer configurationCustomizer() {
        return configuration -> configuration.setObjectWrapperFactory(new MybatisMapWrapperFactory());
    }
}
```

## `tinyint(1)` / `bit` 字段被映射成 `boolean`

**原因**：**MySQL 驱动**默认把 `tinyint(1)` 映射为 boolean（`0=false`，非 0 `=true`），MyBatis 不会自动处理该映射。

**修复**（二选一）：
- 把类型 `tinyint(1)` 改为 `tinyint(2)` 或 `int`；
- 连接串加参数：`jdbc:mysql://127.0.0.1:3306/mp?tinyInt1isBit=false`

## `Error attempting to get column "create_time" from result set. Cause: java.sql.SQLFeatureNotSupportedException`

**版本**：MP 3.1.1+ 且使用 **Druid**。3.1.0 之前无此问题。

**原因**：3.1.1 起采用新版 JDBC，对新日期类型（`LocalDateTime` 等）处理方式升级；Druid 在 **1.1.21 之前**不支持。

**修复**：Druid 升级至 1.1.21+；或保持 MP ≤ 3.1.0；或更换数据源。

## 升级到 3.1.1+ 后新日期类型（`LocalDateTime`）无法映射

**原因**：MP 3.1.0 及之前依赖 MyBatis 3.5.0，3.1.1 升到了 3.5.1。而 MyBatis 3.5.1 中 `LocalDateTypeHandler` / `LocalTimeTypeHandler` / `LocalDateTimeTypeHandler` **要求 JDBC 驱动支持 JDBC 4.2 API**。

**修复**：升级 JDBC 驱动至支持 JDBC 4.2 的版本；或回滚 MP 到 3.1.0 及之前。

## `Failed to bind properties under "mybatis-plus.configuration.incomplete-result-maps[0]..." to org.apache.ibatis.mapping.ParameterMapping`

**现象**：本地启动正常，打成 war 部署到服务器报错。

**原因**：**Spring Boot 2.2.0** 的构造器注入 bug，导致 MyBatis 的私有构造器无法正确绑定属性。已在 2.2.1 修复。

**修复**：降级到 2.1.x 或升级到 2.2.1+（建议直接升到 **2.2.2**）。

## 分离打包部署后执行 Lambda 出现 `ClassNotFoundException`

**现象**：IDE 里跑没问题，打包部署到服务器后执行 Lambda 表达式报错。

**原因**：MP **3.3.2 以下**版本反序列化时类加载器错误。

**修复**：去掉 `spring-boot-maven-plugin` 打包；或升级至 **3.3.2**。

## `Encountered unexpected token: "\n\n\n"` 或 `Could not set parameters for mapping`

**原因**：jsqlparser 4.9 中**连续换行**会被认为是语句结束，抛解析错误；5.0 起碰到连续 `\n\n` 会**截断后面的语句**，进而在 mybatis 层报 `Could not set parameters for mapping`（JSQLParser issue #1988）。

MP 自 **3.5.3.2** 起处理了框架内置注入 SQL 的换行，但**项目里自行编写的语句需要自己处理**。⚠️ 3.5.10 / 3.5.10.1 虽处理了项目里的换行，但**无法支持单行注释（`--` / `#`）**；**3.5.11 后不再处理**。

**修复方式一**（推荐，兼容单行注释）：

```java
static {
    JsqlParserGlobal.setParserMultiFunc((sql)-> {
        String formatSql = CCJSqlParserUtil.sanitizeSingleSql(sql);
        return CCJSqlParserUtil.parseStatements(formatSql, JsqlParserGlobal.getExecutorService(), null);
    });
    JsqlParserGlobal.setParserSingleFunc((sql)-> {
        String formatSql = CCJSqlParserUtil.sanitizeSingleSql(sql);
        return CCJSqlParserUtil.parse(formatSql, JsqlParserGlobal.getExecutorService(), null);
    });
}


// 如果jsqlParser低于5.0，请把如下方法复制到自己工程里，代码来源JsqlParser5.0版本。
public static String sanitizeSingleSql(String sqlStr) {
        Pattern SQL_DELIMITER_SPLIT = Pattern.compile("((?:'[^']*+'|[^\\n])*+)");
        StringBuilder builder = new StringBuilder();
        Matcher matcher = SQL_DELIMITER_SPLIT.matcher(sqlStr);
        while(matcher.find()) {
            for(int i = 1; i <= matcher.groupCount(); ++i) {
                if (!matcher.group(i).isEmpty()) {
                    builder.append("\n").append(matcher.group(i));
                }
            }
        }
        return builder.toString();
}
```

**修复方式二**（通用性没那么好，⚠️ 无法处理 `--` / `#` 单行注释）：

```yaml
mybatis-plus:
    configuration:
        shrink-whitespaces-in-sql: true
```

## 3.5.7+ 版本 `Db.saveBatch` 返回值一直为 `false`

**原因**：驱动配置不兼容。

**修复**：驱动连接串**去掉 `rewriteBatchedStatements=true`**。

## 项目启动速度缓慢

1. **初始化默认雪花 ID 导致** —— 查看本机 hostname，把它写进系统 hosts 文件：

```
# 示例,例如我的hostname名字为nieqiurong-PC，那我在hosts文件配置如下即可
127.0.0.1       localhost nieqiurong-PC
```

2. **数据库连接池初始化** —— 碰到过 hikari 在 linux 下启动缓慢。在 java 启动命令中把随机数来源从 `/dev/random` 改为 `/dev/urandom`：

```
java -Djava.security.egd=file:/dev/urandom -jar xxxx.jar
```

## 逻辑删除：插入时删除标记为空 / 删除接口自动填充失效

**插入**：三选一 —— 数据库为逻辑删除字段设默认值；插入前手动设值；用自动填充功能。

**删除接口自动填充失效**：
- 方法一：使用 `deleteById` 方法；
- 方法二：使用 `update` + `UpdateWrapper.set(column, value)`；
- 方法三：使用 `update` + `UpdateWrapper.setSql("column=value")`；
- 方法四：注入 `com.baomidou.mybatisplus.extension.injector.methods.LogicDeleteByIdWithFill`（**3.5.0 已废弃，推荐 `deleteById`**）。

⚠️ 全局配置 `logic-delete-field` 填的是**实体类属性名**，不是数据库列名。

## 数据库关键字（`status`、`order` 等）导致 SQL 语法错误

3.x **不再自动识别关键字**。两种处理：

单字段加反引号：

```java
@TableField(value = "`status`")
private Boolean status;
```

全局处理（MySQL 为例）：

```yaml
mybatis-plus:
  global-config:
    db-config:
      column-format: "`%s`"
```

⚠️ 用了 `@TableField` 注解又想保持全局格式化，需设置 `keepGlobalFormat=true`；也可在注解里直接写固定格式，如 `@TableField("'status'")`。

**根本建议：数据库设计时避免使用关键字作为字段名或表名。**

## 二级缓存相关

**建议把缓存放到 service 层**，自定义 `BaseServiceImpl` 重写注解父类方法。

**Mapper 层二级缓存刷新问题**（按 mybatis 方式配了第三方二级缓存且版本 > 2.0.9，自带方法无法更新缓存内容），二选一：

```java
@CacheNamespace(implementation=MybatisRedisCache.class,eviction=MybatisRedisCache.class)
public interface DataResourceMapper extends BaseMapper<DataResource>{}
```

```xml
<cache-ref namespace="com.mst.cms.dao.DataResourceMapper"></cache-ref>
```

## 看不到执行的 SQL / 需要开启日志

```yaml
mybatis-plus:
  configuration:
    # 如果项目无日志框架，可以考虑指定为 org.apache.ibatis.logging.stdout.StdOutImpl (请勿在实际生产中使用).
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

或按包设级别：

```yaml
logging:
  level:
    com.baomidou.example.mapper: debug
```

## 想重写 `BaseMapper` 的方法但不生效

⚠️ 必须区分 `default` 方法和抽象接口方法，**重写的方法要以最终实际调用的方法为准**。

- **抽象接口方法**：直接在 XML 重写即可。
- **`default` 方法**：直接重写真实调用的方法，或把原 `default` 重写为真实接口方法，再用 XML / 注解重写执行语句。

```java
// 方式一: 注解式重写BaseMapper中selectPage方法(低版本下可能为接口方法,这里以新版default为例)
@Override
@Select("select * from h2user")
<P extends IPage<H2User>> P selectPage(P page, @Param(Constants.WRAPPER) Wrapper<H2User> queryWrapper);


// 方式二: XML式重写BaseMapper中selectPage方法(低版本下可能为接口方法,这里以新版default为例)
@Override
<P extends IPage<H2User>> P selectPage(P page, @Param(Constants.WRAPPER) Wrapper<H2User> queryWrapper);
// 这里自己去XML里面重写selectPage的查询语句


// 方式三: 重写BaseMapper中selectPage方法
@Override
default <P extends IPage<User>> P selectPage(P page, @Param(Constants.WRAPPER) Wrapper<User> queryWrapper) {
  return xxxx(); //实际调用的自己定义的真实接口方法
}
```

---

## 常见"这怎么做" 速答

- **insert 后如何返回主键** —— 主键会自动 set 到实体的 ID 字段，直接 `getId()`。
- **只查指定的几个字段**：

```java
//3.x
mapper.selectList(
    Wrappers.<User>lambdaQuery()
    .select(User::getId, User::getName)
);
//或者使用QueryWrapper
mapper.selectList(
    new QueryWrapper<User>()
    .select("id","name")
);
```

- **Wrapper 里加 limit**：`wrapper.last("limit 1");`
- **update 时字段自增**：

```java
UpdateWrapper<Entity> wrapper = new UpdateWrapper<>();
wrapper.setSql("column = column + 1");


// 调用 update 方法
baseMapper.update(null, wrapper);
```

- **XML 中按数据库类型选不同 SQL 片段** —— 配 `mybatis-plus.configuration.database-id: mysql`，然后用变量 `_databaseId` 做 `<choose>`，或用标签属性 `databaseId="mysql"` 写多份同 id 的语句。
- **为什么批量插入放在 Service 层** —— SQL 长度有限制（海量数据下单条 SQL 可能执行不了、易引起内存泄漏 / JDBC 连接超时），且各数据库批量语法不一致。MP 采用循环预处理批量提交。想用单条 SQL 方案，自行注入选装方法 `insertbatchsomecolumn`。
- **为什么不支持复合主键** —— 增加表间依赖、增加约束与规则、增加更新限制、易造成数据冗余与更新异常、查询单 ID 时无法走索引。用联合索引代替，坚持唯一 ID。
- **代码生成器生成 `XXXRepository`** —— ⚠️ 官方**后续不再维护 Service 与 Repository，建议不要继续使用**。确需生成：

```java
AutoGenerator generator = new AutoGenerator(DATA_SOURCE_CONFIG);
generator.packageInfo(new PackageConfig.Builder()
    //-- 转换开始----
    .service("repository")
    .serviceImpl("repository.impl")
    //-- 转换结束----
    .build());
generator.strategy(
    new StrategyConfig.Builder()
        //-- 转换开始----
        .serviceBuilder().convertServiceFileName(entityName -> "I" + entityName + "Repository")
        .superServiceClass(IRepository.class).convertServiceImplFileName(entityName -> entityName + "Repository")
        .superServiceImplClass(CrudRepository.class)
        //-- 转换结束----
        .build());
generator.execute();
```

---

# CVE / 安全

## 官方立场

MyBatis-Plus 相关代码和 Jar 包被提交过若干 CVE，官方**均不予认可**，认为是使用方式不当而非框架缺陷：

| CVE | 描述 | 官方回应 |
|---|---|---|
| **CVE-2024-35548** | 前端传入 SQL 片段导致注入 | `QueryWrapper` / `UpdateWrapper` 的字段部分**允许子查询**，因此不能人为允许前端传入 SQL 片段。需要时用 `SqlInjectionUtils.check()` 或 `xxWrapper.checkSqlInjection()`；推荐改用 `LambdaQueryWrapper` / `LambdaUpdateWrapper` |
| **CVE-2023-25330** | 称多租户插件引起 SQL 注入 | 提交者**把租户 ID 暴露给前端**、允许前端传入并保持在上下文中直接被插件读取。正确做法是登录后由后端自行查询用户所属租户并维护上下文；即便要切换租户，也必须校验能不能切 |
| **CVE-2022-25517** | 称 String 类型字段参数可拼接攻击脚本 | 把表字段作为前端可传入的一部分直接拿去拼接。`LambdaQueryWrapper` 可做 Type-Safe 查询；即便用普通 `QueryWrapper`，String 类型的字段也绝不该由前端传递 |

## 漏洞发生的两个位置

ORM 中 SQL 注入通常都因为这两部分允许从前台传参：

1. **表结构部分** —— 表字段、表名等固定内容。
2. **表字段参数 / 变量部分** —— 各种动态 SQL 参数。

## 正确的防注入实践

**表字段部分（由你负责）**：字段名应由后端控制。若系统为了灵活性允许前端动态传入字段名，**必须自行控制字段安全性** —— 通过**字段映射逻辑**阻断攻击，绝不能让前端字符串直接转成 SQL 字段进入编译阶段。

**字段参数部分（框架已负责）**：MyBatis 通过 `#` 占位符（而非 `$`）预编译防注入；MP 生成 SQL 时底层同样使用 `#`，这一步由 MP 自动完成。

**工具类兜底**：

```java
// 开启自动检查 SQL 注入 (3.5.3.2+ 版本支持
Wrappers.query().checkSqlInjection().orderByDesc("任意前端传入字段，我们推荐最好是白名单处理，因为可能存在检查覆盖不全情况")
​
// 手动校验方式 (3.4.3.2+ 版本支持)
SqlInjectionUtils.check("任意前端传入字段，我们推荐最好是白名单处理，因为可能存在检查覆盖不全情况")
```

⚠️ **`checkSqlInjection()` / `SqlInjectionUtils.check()` 只是兜底，存在检查覆盖不全的可能，白名单才是正解。**

⚠️ 分页插件的排序 **`OrderItem.column` 不要让前端传入**；若允许传入，必须做**白名单验证**。

⚠️ 最好的预防方式仍旧是**不允许任何 SQL 片段由前端传到后台**。任何 ORM 框架（包括 JDBC）都允许字符串直接拼接 SQL，安全责任在使用者。
