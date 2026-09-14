# MyBatis-Plus 3.5.17 核心特性参考

按特性组织，每节：用途 → 配置/代码 → ⚠️坑。代码块均取自官方文档原文。

---

## 1. 逻辑删除

**用途**：把物理 `DELETE` 改写成 `UPDATE`，查询自动追加过滤条件，保留数据痕迹。

工作方式（MP 自动改写）：

- 插入：逻辑删除字段的值不受限制
- 查找：自动追加条件，过滤掉标记为已删除的记录
- 更新：防止更新已删除的记录
- 删除：删除转换为更新

```
删除：update user set deleted=1 where id = 1 and deleted=0
查找：select id,name,deleted from user where deleted=0
```

### 方式一：全局配置（推荐，表结构统一时）

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted # 全局逻辑删除字段名(deleted为实体类属性名称)
      logic-delete-value: 1 # 逻辑已删除值。可选，默认值为 1
      logic-not-delete-value: 0 # 逻辑未删除值。可选，默认值为 0
```

⚠️ `logic-delete-field` 填的是**实体类属性名**（`deleted`），不是列名（`is_deleted`）。填错不会报错，只是静默不生效。

### 方式二：注解单表配置

```java
import com.baomidou.mybatisplus.annotation.TableLogic;

public class User {
    // 其他字段...

    @TableLogic
    private Integer deleted;
}
```

默认未删除 `0`、已删除 `1`，可通过 `@TableLogic(value = "...", delval = "...")` 覆盖。注解优先于全局配置。

### ⚠️ 坑

1. **只对 MP 自带方法生效**。`selectById` / `selectList` / `update` / `deleteById` 等注入方法会自动改写；你手写的 `@Select` 注解 SQL、XML 里的自定义 SQL **一律不生效**，必须自己在 SQL 里写 `AND deleted = 0`。这是最高频的线上事故来源。
2. **唯一索引冲突**：`user(name)` 上有唯一索引时，逻辑删一条 `name='a'` 后再插入 `name='a'` 会直接违反唯一约束——因为旧行物理仍在。两种解法：
   - 唯一索引改为 `(name, deleted)` 联合索引，但 `deleted` 只有 0/1 时**只能删一次**（第二次删除又变成 `(a,1)` 重复）。
   - **推荐**：`deleted` 用 `bigint`，未删除值 `0`，已删除值用函数如 `UNIX_TIMESTAMP()` 取当前时间戳。每次删除值都不同，`(name, deleted)` 联合唯一索引可以无限次逻辑删除。`datetime` 类型同理：未删除值配 `null`（长度为 4 的字符串，yaml 中需用单引号转义包裹 `'null'`），已删除用 `now()`。
3. **字段类型选择**：支持所有类型，推荐 `Integer`、`Boolean`、`LocalDateTime`。想让删除字段参与唯一索引就选 `bigint` + 时间戳方案。
4. **插入时逻辑删除字段为 null**：三种解法——数据库列设默认值 / 插入前手动赋值 / 用自动填充（见第 2 节）。
5. **删除接口自动填充失效**（想在删除时同时写 `updateTime`/`updateBy`）：
   - 用 `deleteById` 方法
   - 用 `update` + `UpdateWrapper.set(column, value)`
   - 用 `update` + `UpdateWrapper.setSql("column=value")`
   - `LogicDeleteByIdWithFill` 注入器（3.5.0 已废弃，别用，改用 `deleteById`）
6. **业务判断**：如果"已删除"数据还需要频繁查询，说明你要的不是逻辑删除而是**状态字段**。逻辑删除的语义应等同于物理删除。

---

## 2. 自动填充

**用途**：insert/update 时自动写入 `createTime` / `updateTime` / `createBy` 等审计字段。

### 实体侧：必须声明 `@TableField(fill = ...)`

```java
public class User {
    @TableField(fill = FieldFill.INSERT)
    private String createTime;

    @TableField(fill = FieldFill.UPDATE)
    private String updateTime;

    // 其他字段...
}
```

`FieldFill` 枚举：

```java
public enum FieldFill {
    DEFAULT,       // 默认不处理
    INSERT,        // 插入填充字段
    UPDATE,        // 更新填充字段
    INSERT_UPDATE  // 插入和更新填充字段
}
```

### 处理器：实现 `MetaObjectHandler`

```java
@Slf4j
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        log.info("开始插入填充...");
        this.strictInsertFill(metaObject, "createUserId", Long.class, 123456L)
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        log.info("开始更新填充...");
        this.strictInsertFill(metaObject, "updateUserId", Long.class, 123456L)
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

Kotlin 版本注意类型装箱：

```kotlin
@Component
class MyMetaObjectHandler : MetaObjectHandler {

    private val log = LoggerFactory.getLogger(MyMetaObjectHandler::class.java)

    // 注意将kotlin类型转为java类型请使用 xxx::class.javaObjectType，防止部分类型使用xxx::class.java转换为基本类型导致类型不一致无法填充
    override fun insertFill(metaObject: MetaObject) {
        log.info("开始插入填充...");
        this.strictInsertFill(metaObject, "createUserId", Long::class.javaObjectType, 123456L)
        this.strictInsertFill(metaObject, "createTime", LocalDateTime::class.javaObjectType, LocalDateTime.now())
    }

    override fun updateFill(metaObject: MetaObject) {
        log.info("开始更新填充...");
        this.strictUpdateFill(metaObject, "updateUserId", Long::class.javaObjectType, 123456L)
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime::class.javaObjectType, LocalDateTime.now())
    }
}
```

### strictXxxFill vs fillStrategy

- `strictInsertFill` / `strictUpdateFill`：**三重校验** —— 注解 `FieldFill.xxx` 匹配 + 字段名匹配 + 字段类型匹配，任一不符则跳过。字段类型写错（如实体是 `Date`，你传 `LocalDateTime.class`）会静默不填充。
- `fillStrategy`：不区分 `FieldFill` 策略，只按字段名填。需要绕过注解校验时用。

### ⚠️ 坑

1. **默认策略：属性有值则不覆盖，填充值为 null 则不填充**。想强制覆盖已有值必须用 `setFieldValByName` 自己写逻辑。
2. **自动填充是直接给实体类属性 setValue**，不是在 SQL 层做的。属性没值入库就是 `null`。
3. **`@TableField(fill=...)` 必须配套**。少了注解，`strictXxxFill` 不填；处理器写了也白写。
4. **`update(Wrapper<T> updateWrapper)` 单参版本完全不触发 updateFill**，必须手动 `.set("update_time", now)`。
5. **`update(T entity, Wrapper<T> updateWrapper)` 时 `entity` 不能为 null**，否则自动填充失效。
6. **自定义 Mapper 方法要填充，参数名必须符合约定**：实体参数需命名为 `et`。

```java
// 插入填充示例
insertFillByCustomMethod1(H2User h2User);
insertFillByCustomMethod8(H2User[] h2Users);
insertFillByCustomMethod4(Collection<H2User> h2User);

// 更新填充示例
updateFillByCustomMethod2(@Param("coll") Collection<Long> ids, @Param("et") H2User h2User);
updateFillByCustomMethod4(@Param("colls") Collection<Long> ids, @Param("et") H2User h2User);
```

```java
// 方法参数名不满足填充条件，需要将 user 换成 et 才能正确填充
updateFillByCustomMethod3(@Param("coll") Collection<Long> ids, @Param("user") H2User h2User);
```

7. **处理器必须是 Spring Bean**（`@Component` 或 `@Bean`），否则 MP 拿不到。

---

## 3. 通用枚举

**用途**：枚举 ↔ 数据库列自动互转，用 `MybatisEnumTypeHandler`（基于枚举常量属性），而非 MyBatis 原生的 `EnumOrdinalTypeHandler`（序号）/ `EnumTypeHandler`（常量名）。

```java
public class User {
    private String name; // 名字
    private AgeEnum age; // 年龄
    private GradeEnum grade; // 年级
}
```

### 方式一：`@EnumValue` 注解（推荐，侵入最小）

```java
@Getter
@AllArgsConstructor
public enum GradeEnum {
    PRIMARY(1, "小学"),
    SECONDARY(2, "中学"),
    HIGH(3, "高中");

    @EnumValue // 标记数据库存的值是code
    private final int code;
    // 其他属性...
}
```

### 方式二：实现 `IEnum`

```java
@Getter
@AllArgsConstructor
public enum AgeEnum implements IEnum<Integer> {
    ONE(1, "一岁"),
    TWO(2, "二岁"),
    THREE(3, "三岁");

    private final int value;
    private final String desc;

    @Override
    public Integer getValue() {
        return this.value;
    }
}
```

### 未声明的枚举

走 MyBatis `defaultEnumTypeHandler` 默认值 `EnumTypeHandler`（存常量名）。可全局改：

```yaml
mybatis-plus:
  configuration:
    default-enum-type-handler: xx.xx.xx.MyEnumTypeHandler
```

或配置类：

```java
@Configuration
public class MybatisPlusAutoConfiguration {

    @Bean
    public MybatisPlusPropertiesCustomizer mybatisPlusPropertiesCustomizer() {
        return properties -> {
            GlobalConfig globalConfig = properties.getGlobalConfig();
            globalConfig.setBanner(false);
            MybatisPlusProperties.CoreConfiguration configuration = new MybatisPlusProperties.CoreConfiguration();
            configuration.setDefaultEnumTypeHandler(MyEnumTypeHandler.class);
            properties.setConfiguration(configuration);
        };
    }
}
```

### 与 Jackson 序列化配合

方式 A：重写 `toString()` + 开启 `WRITE_ENUMS_USING_TO_STRING`

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer customizer() {
    return builder -> builder.featuresToEnable(SerializationFeature.WRITE_ENUMS_USING_TO_STRING);
}
```

独立使用：

```java
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.configure(SerializationFeature.WRITE_ENUMS_USING_TO_STRING, true);
```

方式 B：`@JsonValue` 注解（推荐，与 `@EnumValue` 打在同一字段上，入库值与出参值一致）

```java
public enum GradeEnum {
    PRIMARY(1, "小学"),
    SECONDORY(2, "中学"),
    HIGH(3, "高中");

    GradeEnum(int code, String descp) {
        this.code = code;
        this.descp = descp;
    }

    @EnumValue
    @JsonValue // 标记响应json值
    private final int code;
}
```

Fastjson 对应：全局 `config.setSerializerFeatures(SerializerFeature.WriteEnumUsingToString)`，或字段级 `@JSONField(serialzeFeatures = SerializerFeature.WriteEnumUsingToString)`。

### ⚠️ 坑

- `type-enums-package` 老配置在新版本已非必需（`@EnumValue`/`IEnum` 会被自动扫描），但显式配置更稳：`mybatis-plus.type-enums-package: com.xx.enums`。
- 改 `default-enum-type-handler` **对已用 `@EnumValue`/`IEnum` 声明的枚举无效**，它们始终走 `MybatisEnumTypeHandler`。
- 只加 `@EnumValue` 不加 `@JsonValue`，接口返回的是枚举**常量名**字符串，前端拿到的和库里存的不是一个值——前后端联调高频踩坑。

---

## 4. 字段类型处理器（TypeHandler）

**用途**：JavaType ↔ JdbcType 转换。最常见场景是 JSON 列 ↔ POJO。

内置：`AbstractJsonTypeHandler` 及子类 `Fastjson2TypeHandler`、`FastjsonTypeHandler`、`GsonTypeHandler`、`JacksonTypeHandler`。

### 实体配置

```java
@Data
@Accessors(chain = true)
@TableName(autoResultMap = true)
public class User {
    private Long id;

    ...

    /**
     * 必须开启映射注解
     *
     * @TableName(autoResultMap = true)
     *
     * 选择对应的 JSON 处理器，并确保存在对应的 JSON 解析依赖包
     */
    @TableField(typeHandler = JacksonTypeHandler.class)
    // 或者使用 FastjsonTypeHandler
    // @TableField(typeHandler = FastjsonTypeHandler.class)
    private OtherInfo otherInfo;
}
```

⚠️ **`@TableName(autoResultMap = true)` 是硬性前提**。不加的话：写入能正常序列化，但**查询回来永远是 null**（MP 不生成带 typeHandler 的 resultMap）。这是 JSON 字段第一大坑。

### XML 侧写法（自定义 SQL / VO 映射同样要显式声明）

```xml
<!-- 单个字段的类型处理器配置 -->
<result column="other_info" jdbcType="VARCHAR" property="otherInfo" typeHandler="com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler" />

<!-- 多个字段中某个字段的类型处理器配置 -->
<resultMap id="departmentResultMap" type="com.baomidou...DepartmentVO">
    <result property="director" column="director" typeHandler="com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler" />
</resultMap>
<select id="selectPageVO" resultMap="departmentResultMap">
   select id,name,director from department ...
</select>
```

### Wrapper 中使用 TypeHandler（3.5.3.2+）

```java
Wrappers.<H2User>lambdaQuery()
    .apply("name={0,typeHandler=" + H2userNameJsonTypeHandler.class.getCanonicalName() + "}", "{\"id\":101,\"name\":\"Tomcat\"}"))
```

### 自定义 TypeHandler（PostgreSQL JSONB 示例）

```java
import com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.MappedJdbcTypes;
import org.apache.ibatis.type.MappedTypes;
import org.postgresql.util.PGobject;

import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

@MappedTypes({Object.class})
@MappedJdbcTypes(JdbcType.VARCHAR)
public class JsonbTypeHandler<T> extends JacksonTypeHandler<T> {

    private final Class<T> clazz;

    public JsonbTypeHandler(Class<T> clazz) {
        if (clazz == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.clazz = clazz;
    }

    // 自3.5.6版本开始支持泛型,需要加上此构造.
    public JsonbTypeHandler(Class<?> type, Field field) {
        super(type, field);
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
        PGobject jsonbObject = new PGobject();
        jsonbObject.setType("jsonb");
        jsonObject.setValue(toJson(parameter));
        ps.setObject(i, jsonbObject);
    }
}
```

使用同内置处理器：

```java
@Data
@Accessors(chain = true)
@TableName(autoResultMap = true)
public class User {
    private Long id;

    ...

    /**
     * 使用自定义的 JSONB 类型处理器
     */
    @TableField(typeHandler = JsonbTypeHandler.class)
    private OtherInfo otherInfo;
}
```

⚠️ 3.5.6+ 支持泛型字段（`List<Foo>`），**必须补上 `(Class<?> type, Field field)` 构造**，否则反射实例化失败。另：选了哪个处理器就要引对应 JSON 解析依赖（Jackson/Fastjson/Gson）。

---

## 5. 主键策略与自定义 ID 生成器

### 5.1 `IdentifierGenerator`（MP 自己的主键生成）

3.3.0 起默认雪花算法（`ASSIGN_ID`）+ 无中划线 UUID（`ASSIGN_UUID`）。

| 方法 | 主键生成策略 | 主键类型 | 说明 |
|---|---|---|---|
| nextId | ASSIGN_ID | Long,Integer,String | 支持自动转换为 String 类型，但数值类型不支持自动转换，需精准匹配，例如返回 Long，实体主键就不支持定义为 Integer |
| nextUUID | ASSIGN_UUID | String | 默认不含中划线的 UUID 生成 |

自定义（接入分布式发号器）：

```java
@Component
public class CustomIdGenerator implements IdentifierGenerator {
    @Override
    public Long nextId(Object entity) {
        // 使用实体类名作为业务键，或者提取参数生成业务键
        String bizKey = entity.getClass().getName();
        // 根据业务键调用分布式ID生成服务
        long id = ...; // 调用分布式ID生成逻辑
        // 返回生成的ID值
        return id;
    }
}
```

另两种注册方式：

```java
@Bean
public IdentifierGenerator idGenerator() {
    return new CustomIdGenerator();
}
```

```java
@Bean
public MybatisPlusPropertiesCustomizer plusPropertiesCustomizer() {
    return plusProperties -> plusProperties.getGlobalConfig().setIdentifierGenerator(new CustomIdGenerator());
}
```

⚠️ **雪花算法集群重复 ID**：MP 默认的 `DefaultIdentifierGenerator` 从本机 **网卡 MAC + 进程 PID** 推导 `workerId`/`datacenterId`。容器化部署（K8s、Docker）下多个 Pod 极易算出相同 workerId，导致**跨实例 ID 重复**。生产必须显式指定：

```java
@Bean
public IdentifierGenerator idGenerator() {
    // workerId(0~31), datacenterId(0~31)，从环境变量/配置中心下发，保证全局唯一
    return new DefaultIdentifierGenerator(workerId, datacenterId);
}
```

⚠️ 其他雪花坑：
- 主键类型必须**精准匹配**，`nextId` 返回 `Long` 时实体主键定义成 `Integer` 会直接炸。
- 雪花 ID 是 19 位 Long，超过 JS `Number.MAX_SAFE_INTEGER`，**接口返回给前端必须序列化为 String**（Jackson 加 `@JsonSerialize(using = ToStringSerializer.class)` 或全局 Long→String）。
- 依赖系统时钟，**时钟回拨会抛异常**。

### 5.2 `IKeyGenerator`（数据库序列，Oracle/PG/DB2/H2/Kingbase）

内置：`DB2KeyGenerator`、`H2KeyGenerator`、`KingbaseKeyGenerator`、`OracleKeyGenerator`、`PostgreKeyGenerator`。不够用就实现 `IKeyGenerator`。

```java
@KeySequence(value = "SEQ_ORACLE_STRING_KEY", clazz = String.class)
public class YourEntity {

    @TableId(value = "ID_STR", type = IdType.INPUT)
    private String idStr;

    // 其他字段和方法...
}
```

⚠️ **必须 `IdType.INPUT`**，其他类型序列不生效。3.3.0 起主键类型自动识别，`clazz` 可省。`@KeySequence` 支持定义在父类，子类继承。

Spring Boot 注册：

```java
@Bean
public IKeyGenerator keyGenerator() {
    return new H2KeyGenerator();
}
```

```java
@Bean
public MybatisPlusPropertiesCustomizer plusPropertiesCustomizer() {
    return plusProperties -> plusProperties.getGlobalConfig().getDbConfig().setKeyGenerator(new H2KeyGenerator());
}
```

Spring XML：

```xml
<bean id="globalConfig" class="com.baomidou.mybatisplus.core.config.GlobalConfig">
   <property name="dbConfig" ref="dbConfig"/>
</bean>

<bean id="dbConfig" class="com.baomidou.mybatisplus.core.config.GlobalConfig.DbConfig">
   <property name="keyGenerator" ref="keyGenerator"/>
</bean>

<bean id="keyGenerator" class="com.baomidou.mybatisplus.extension.incrementer.H2KeyGenerator"/>
```

### 5.3 两者区别

`IdentifierGenerator` 是 MP 的接口，专注主键 ID 生成，在 Java 侧生成后塞进实体；`KeyGenerator` 是 MyBatis 框架接口，更通用（自增主键回填、INSERT 后取 ID）。MP 项目优先用 `IdentifierGenerator`。

---

## 6. 多数据源（dynamic-datasource）

**用途**：读写分离、一主多从、跨库操作。

### 依赖

SpringBoot 2：

```xml
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
  <version>${version}</version>
</dependency>
```

SpringBoot 3：

```xml
<dependency>
 <groupId>com.baomidou</groupId>
 <artifactId>dynamic-datasource-spring-boot3-starter</artifactId>
 <version>${version}</version>
</dependency>
```

### 配置

```yaml
spring:
  datasource:
    dynamic:
      primary: master
      strict: false
      datasource:
        master:
          url: jdbc:mysql://xx.xx.xx.xx:3306/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        slave_1:
          url: jdbc:mysql://xx.xx.xx.xx:3307/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        slave_2:
          url: ENC(xxxxx)
          username: ENC(xxxxx)
          password: ENC(xxxxx)
          driver-class-name: com.mysql.jdbc.Driver
```

### `@DS` 切换

```java
@Service
@DS("slave")
public class UserServiceImpl implements UserService {

  @Autowired
  private JdbcTemplate jdbcTemplate;

  @Override
  @DS("slave_1")
  public List selectByCondition() {
    return jdbcTemplate.queryForList("select * from user where age >10");
  }
}
```

### 约定

- 配置里以下划线 `_` 分割的数据源，**首部为组名**（`slave_1`、`slave_2` → 组名 `slave`），`@DS("slave")` 即组内轮询。
- 切换目标可以是组名或具体数据源名。
- 默认主数据源名 `master`，用 `spring.datasource.dynamic.primary` 改。
- **方法上的注解优先于类上的注解**。

### ⚠️ 坑

1. **`@DS` 与 `@Transactional` 的经典冲突**：Spring 事务在方法进入时就绑定并缓存了 Connection，事务开启后再切数据源**不会生效**（后续 SQL 仍走事务开启时那个库）。两条规则：
   - `@DS` 必须作用在**比 `@Transactional` 更外层或同层**，不要在事务方法内部调另一个 `@DS` 方法期望切库。
   - 跨库写操作**不是本地事务能保证的**，需要 `@DSTransactional`（dynamic-datasource 提供的弱一致本地多数据源事务）或 Seata 分布式事务。
2. **`strict: false` 时找不到数据源会静默回落到 primary**，排查困难。多库场景建议 `strict: true` 让它直接报错。
3. `@DS` 基于 AOP，**同类内部方法自调用不走代理，切换失效**。
4. 敏感配置可用 `ENC()` 加密。

（企业版 mybatis-mate 对应能力见第 11 节 `@Sharding`。）

---

## 7. p6spy（SQL 打印与分析）

**用途**：代理 JDBC 驱动，打印真实 SQL（参数已回填）+ 执行耗时 + 慢 SQL 检测。MP 3.1.0+。

### 依赖

```xml
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

### 数据源改走 p6spy 驱动

```yaml
spring:
  datasource:
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
    url: jdbc:p6spy:h2:mem:test
    # 其他数据库配置...
```

### `spy.properties`（放 classpath 根目录）

```properties
# 模块列表，根据版本选择合适的配置
modulelist=com.baomidou.mybatisplus.extension.p6spy.MybatisPlusLogFactory,com.p6spy.engine.outage.P6OutageFactory

# 自定义日志格式
logMessageFormat=com.baomidou.mybatisplus.extension.p6spy.P6SpyLogger

# 日志输出到控制台
appender=com.baomidou.mybatisplus.extension.p6spy.StdoutLogger

# 取消JDBC驱动注册
deregisterdrivers=true

# 使用前缀
useprefix=true

# 排除的日志类别
excludecategories=info,debug,result,commit,resultset

# 日期格式
dateformat=yyyy-MM-dd HH:mm:ss

# 实际驱动列表
# driverlist=org.h2.Driver

# 开启慢SQL记录
outagedetection=true

# 慢SQL记录标准（单位：秒）
outagedetectioninterval=2

# 过滤 flw_ 开头的表 SQL 打印
filter=true
exclude=flw_*
```

### Spring Boot Starter 方案（无需改 url）

```xml
<dependency>
    <groupId>com.github.gavlyukovskiy</groupId>
    <artifactId>p6spy-spring-boot-starter</artifactId>
    <version>2.0.1</version>
</dependency>
```

```yaml
decorator:
  datasource:
    p6spy:
      # 日志格式
      log-format: "\ntime:%(executionTime) || sql:%(sql)\n"
      # 自定义日志类
      logging: custom
      custom-appender-class: com.example.testinit.config.StdoutLogger
```

```java
public class StdoutLogger extends com.p6spy.engine.spy.appender.StdoutLogger {
    public void logText(String text) {
        System.err.println(text);
    }
}
```

### ⚠️ 坑

- `driver-class-name` 必须是 `com.p6spy.engine.spy.P6SpyDriver`，url 必须加 `jdbc:p6spy:` 前缀。
- 打印出来的 SQL 是 `null`？在 `excludecategories` 里**加上 `commit`**。
- 批量操作不打印？从 `excludecategories` 里**去掉 `batch`**。
- 批量操作重复打印？用 `MybatisPlusLogFactory`（3.2.1 新增）。
- ⚠️ **有性能损耗，不建议在生产环境使用**。只在 dev/test profile 挂载 `spy.properties`。

---

## 8. 数据安全保护

MP 3.3.2+ 提供配置加密与数据加密，目的是防止开发人员流动造成的敏感信息泄露。

### 8.1 YML 配置加密

以 `mpw:` 开头的配置项会被当作密文解密。

```yaml
spring:
  datasource:
    url: mpw:qRhvCwF4GOqjessEB3G+a5okP+uXXr96wcucn2Pev6Bf1oEMZ1gVpPPhdDmjQqoM
    password: mpw:Hzy5iliJbwDHhjLs1L0j6w==
    username: mpw:Xb+EgsyuYRXw7U7sBJjBpA==
```

生成密钥与密文：

```java
// 生成16位随机AES密钥
String randomKey = AES.generateRandomKey();

// 使用随机密钥加密数据
String encryptedData = AES.encrypt(data, randomKey);
```

启动时传密钥：

```
// Jar 启动参数示例（在IDEA中设置Program arguments，或在服务器上设置为启动环境变量）
--mpw.key=d1104d7c3b616f0b
```

3.5.10 起支持通过**系统属性与环境变量**传递密钥。

⚠️ 加密配置**必须以 `mpw:` 开头**；密钥由负责人保管，知晓人越少越好；密钥不要和密文放在同一个 git 仓库，否则等于没加密。

### 8.2 字段加解密与脱敏

属于 mybatis-mate 企业组件能力（`@FieldEncrypt` / `@FieldSensitive`），详见第 11 节。

### 8.3 SQL 注入检查

自动检查（Wrapper）：

```java
Wrappers.query()
// 开启自动检查 SQL 注入
.checkSqlInjection().orderByDesc("任意前端传入字段，我们推荐最好是白名单处理，因为可能存在检查覆盖不全情况")
```

手动校验：

```java
// 手动校验前端传入的字段是否存在 SQL 注入风险
SqlInjectionUtils.check("任意前端传入字段，我们推荐最好是白名单处理，因为可能存在检查覆盖不全情况")
```

⚠️ **`checkSqlInjection()` 覆盖不全，不能当成安全兜底**。`orderBy`、`apply`、`last` 这类能拼接原始 SQL 的入口，正确做法是**列名白名单映射**，最好根本不让前端传 SQL 片段。

---

## 9. SQL 注入器（自定义全局方法）

**用途**：给所有 Mapper 注入自定义通用方法（批量 insert、物理删除、数据权限 SQL 等），弥补 `BaseMapper` 不足。

### 步骤 1：继承 `AbstractMethod` 定义 SQL

```java
public class MysqlInsertAllBatch extends AbstractMethod {

    /**
     * @since 3.5.0
     */
    public MysqlInsertAllBatch() {
        super("mysqlInsertAllBatch");
    }

    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {
     // 定义SQL语句 (主键+普通字段)  insert into table(....) values(....),(....)
        String sql = "INSERT INTO " + tableInfo.getTableName() + "(" + tableInfo.getKeyColumn() + "," +
                tableInfo.getFieldList().stream().map(TableFieldInfo::getColumn).collect(Collectors.joining(",")) + ") VALUES ";
        String value = "(" + "#{" + ENTITY + DOT + tableInfo.getKeyProperty() + "}" + ","
                + tableInfo.getFieldList().stream().map(tableFieldInfo -> "#{" + ENTITY + DOT + tableFieldInfo.getProperty() + "}")
                .collect(Collectors.joining(",")) + ")";
        String valuesScript = SqlScriptUtils.convertForeach(value, "list", null, ENTITY, COMMA);
        SqlSource sqlSource = super.createSqlSource(configuration, "<script>" + sql + valuesScript + "</script>", modelClass);
        KeyGenerator keyGenerator = tableInfo.getIdType() == IdType.AUTO ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
        // 第三个参数必须和baseMapper的自定义方法名一致
        return this.addInsertMappedStatement(mapperClass, modelClass, this.methodName, sqlSource, keyGenerator,tableInfo.getKeyProperty(), tableInfo.getKeyColumn());
    }
}
```

⚠️ `super("mysqlInsertAllBatch")` 里的方法名、`addInsertMappedStatement` 的第三个参数、Mapper 接口方法名**三者必须完全一致**，否则运行期报 `Invalid bound statement`。

### 步骤 2：继承 `DefaultSqlInjector` 注册

```java
public class MyLogicSqlInjector extends DefaultSqlInjector {

    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass) {
        List<AbstractMethod> methodList = super.getMethodList(mapperClass);
        methodList.add(new DeleteAll());
        methodList.add(new MyInsertAll());
        methodList.add(new MysqlInsertAllBatch());
        return methodList;
    }
}
```

⚠️ 一定要先 `super.getMethodList(mapperClass)` 再 add，否则丢掉全部 BaseMapper 内置方法。

### 步骤 3：定义自己的 BaseMapper

```java
public interface MyBaseMapper<T> extends BaseMapper<T> {

    Integer deleteAll();

    int myInsertAll(T entity);

    int mysqlInsertAllBatch(@Param("list") List<T> batchList);
}
```

### 步骤 4：配置注入器

```yaml
mybatis-plus:
  global-config:
    sql-injector: com.example.MyLogicSqlInjector
```

```properties
mybatis-plus.global-config.sql-injector=com.example.MyLogicSqlInjector
```

（也可以直接 `@Bean ISqlInjector`。）

### ⚠️ 坑

- 批量方法参数必须用 `@Param` 且命名符合 MP 默认支持（`list`、`collection`、`array`），否则 `foreach` 取不到。
- 自定义注入方法**不会自动套用逻辑删除/自动填充**，逻辑得自己在 SQL 里写全。
- 注入器是全局的，所有 Mapper 都会被注入，SQL 里别写死表名/方言。

---

## 10. 自动维护 DDL

**用途**（3.5.3+）：Schema 初始化与升级 SQL 自动执行。相比 flyway 的优势：支持分库分表、可用代码控制脚本执行过程、可切换数据源。

首次运行会在库中生成 `ddl_history` 表，记录每次执行的脚本版本信息。

```java
@Component
public class MysqlDdl implements IDdl {

    /**
     * 获取要执行的SQL脚本文件列表
     */
    @Override
    public List<String> getSqlFiles() {
        return Arrays.asList(
                "db/tag-schema.sql",
                // 从`3.5.3.2`版本开始，支持执行存储过程。在文件名后追加`#$$`，其中`$$`是自定义的完整SQL分隔符。
                // 存储过程脚本以`END`结尾，并追加分隔符`END;$$`表示脚本结束。
                "db/procedure.sql#$$",
                "D:\\db\\tag-data.sql"
        );
    }
}

// 切换到mysql从库，执行SQL脚本 (开源版本无此功能)
ShardingKey.change("mysqlt2");
ddlScript.run(new StringReader("DELETE FROM user;\n" +
        "INSERT INTO user (id, username, password, sex, email) VALUES\n" +
        "(20, 'Duo', '123456', 0, 'Duo@baomidou.com');"));
```

### 自定义运行器

集成 MP starter 会自动实例化 `DdlApplicationRunner` 执行脚本，**默认自动提交事务、忽略错误继续执行**。要精细控制就自己注入：

```java
    @Bean
    public DdlApplicationRunner ddlApplicationRunner(List<IDdl> ddlList) {
          DdlApplicationRunner ddlApplicationRunner = new DdlApplicationRunner(ddlList);
        // 下面属性自 3.5.11 开始 ...
        // 设置是否自动提交 默认: true
        ddlApplicationRunner.setAutoCommit(false);
        // 设置脚本遇到错误的处理方式 默认: 忽略错误,打印异常 (如果设置为抛出异常,那会终止下一个sql文件处理)
        ddlApplicationRunner.setDdlScriptErrorHandler(DdlScriptErrorHandler.ThrowsErrorHandler.INSTANCE);
        //是否抛出异常中断下个处理器处理 默认: false
        ddlApplicationRunner.setThrowException(true);
        ddlApplicationRunner.setScriptRunnerConsumer(scriptRunner -> {
            scriptRunner.setLogWriter(null);   // 关闭执行日志打印 默认: System.out
            scriptRunner.setErrorLogWriter(null); // 关闭错误日志打印  默认:System.err
            scriptRunner.setStopOnError(true); // 遇到异常是否停止
            scriptRunner.setRemoveCRs(false); //  是否替换\r\n 为 \n 默认: false
        });
        return ddlApplicationRunner;
    }
```

### ⚠️ 坑

- **默认忽略错误继续执行**，脚本失败了应用照常启动，`ddl_history` 里却记了版本 —— 生产务必显式设 `ThrowsErrorHandler` + `setThrowException(true)`。
- 脚本内如果包含"切库建库"操作可能抛异常；正确做法是先切数据源，再动态执行脚本命令。
- 存储过程脚本需要 `文件名#$$` 指定分隔符，且脚本以 `END;$$` 结尾（3.5.3.2+）。
- `IDdlGenerator` 是方言层接口（`MysqlDdlGenerator`/`PostgreDdlGenerator` 等），负责生成 `ddl_history` 相关语句；一般不用自己实现，只在自研数据库方言时扩展。

---

## 11. 高级特性 / 工具类

### 11.1 `Db` 静态工具类（3.5.3+，开源版，强烈推荐）

**用途**：免注入 Service / Mapper 直接做 CRUD。彻底解决"为了查一张表而循环注入 Service 导致循环依赖"的问题。

```java
// 条件查询（Lambda）
List<User> users = Db.lambdaQuery(User.class)
        .eq(User::getStatus, 1)
        .like(User::getName, "张")
        .orderByDesc(User::getCreateTime)
        .list();

// 单条 / 数量 / 分页
User one   = Db.lambdaQuery(User.class).eq(User::getPhone, phone).one();
long count = Db.lambdaQuery(User.class).eq(User::getStatus, 1).count();
Page<User> page = Db.lambdaQuery(User.class).page(new Page<>(1, 10));

// 主键 / 批量主键
User u = Db.getById(id, User.class);
List<User> list = Db.listByIds(ids, User.class);

// 写入
Db.save(user);
Db.saveBatch(userList);
Db.saveOrUpdate(user);
Db.updateById(user);
Db.removeById(id, User.class);

// 更新（Lambda）
Db.lambdaUpdate(User.class)
        .eq(User::getId, id)
        .set(User::getStatus, 0)
        .update();
```

⚠️ 坑：
- `Db` 的每个静态方法内部**各自走一次 SqlSession**。要保证原子性必须外层有 `@Transactional`。
- 事务传播依赖 Spring 上下文，**在 Spring 容器初始化完成前调用会 NPE**（拿不到 SqlSessionFactory）。
- `Db.lambdaQuery(User.class)` 传的是 **Class**；`Db.lambdaQuery(userInstance)` 传实例时会把实例非空字段当成等值条件，两者语义不同，别混。
- `saveBatch` 默认按 1000 条分批，MySQL 需 url 加 `rewriteBatchedStatements=true` 才真正合并成一条 SQL，否则没有性能收益。

### 11.2 ActiveRecord（`Model<T>`）

实体继承 `Model<T>`，直接在实体上调 CRUD：

```java
@TableName("sys_user")
public class User extends Model<User> {
    @TableId
    private Long id;
    private String name;
}

// 使用
User user = new User();
user.setName("张三");
user.insert();

new User().setId(1L).selectById();
new User().setId(1L).deleteById();
user.updateById();
```

⚠️ 前提是该实体对应的 Mapper 已被 Spring 扫描注册；同样要靠 Spring 上下文，脱离容器（如单测未启容器）直接报错。AR 模式会把持久化逻辑污染进领域对象，团队项目建议统一用 `Db` 工具类而非 AR。

### 11.3 mybatis-mate（企业级扩展模块，收费授权）

> Mybatis-Mate 是为 MyBatis-Plus 提供的企业级模块，属于扩展库，非 MyBatis-Plus 的收费版本。

- **数据审计（对账）**：对比两对象属性差异（银行流水对账）

```java
// 1，异步回调，注意 @EnableAsync 开启异步
applicationEventPublisher.publishEvent(new DataAuditEvent((t) -> {
    List<Change> changes = t.apply(newVersion, oldVersion);
    for (Change valueChange : changes) {
        ValueChange change = (ValueChange) valueChange;
        System.err.println(String.format("%s不匹配，期望值 %s 实际值 %s", change.getPropertyName(), change.getLeft(), change.getRight()));
    }
}));

// 2，手动调用对比
DataAuditor.compare(obj1, obj2);
```

- **数据敏感词过滤**：AC 算法，数据库自维护词库，支持嵌套关键词。
- **数据范围（数据权限）**：`@DataScope` + `@DataColumn`，行级粒度。

```java
// 测试 test 类型数据权限范围，混合分页模式
@DataScope(type = "test", value = {
        // 关联表 user 别名 u 指定部门字段权限
        @DataColumn(alias = "u", name = "department_id"),
        // 关联表 user 别名 u 指定手机号字段（自己判断处理）
        @DataColumn(alias = "u", name = "mobile")
})
@Select("select u.* from user u")
List<User> selectTestList(IPage<User> page, Long id, @Param("name") String username);

// 测试数据权限，最终执行 SQL 语句
SELECT u.* FROM user u WHERE (u.department_id IN ('1', '2', '3', '5')) AND u.mobile LIKE '%1533%' LIMIT 1,10
```

⚠️ 必须注入 `IDataScopeProvider` 实现类处理数据权限；传参两种方式：自定义 mapper 方法参数传递（在 `setWhere` 的 `Object[] args` 中获取），或用 ThreadLocal 在 controller/service 层拦截设置。

- **字段数据绑定（字典回写）**：`@FieldBind(type="user_sex", target="sexText")`，绑定属性需 `@TableField(exist = false)` 排除；业务类实现 `IDataBind` 注入容器。
- **虚拟属性绑定**：`@JsonBind("绑定类型")` + `IJsonBindStrategy` 注入虚拟 JSON 节点。
- **字段加密解密**：`@FieldEncrypt`，算法可选 `MD5_32/MD5_16/BASE64/AES/RSA/SM2/SM3/SM4/PBEWithMD5AndDES/...`。

```java
@FieldEncrypt
private String email;
```

⚠️ **需要模糊查询的字段务必用 AES**；MD5 不可逆（库里和查询结果都是密文）；SM4 依赖 bouncycastle；PBE 混合算法依赖 jasypt；**查询返回的必须是带加密注解的对象，单纯返回 String 或 List 集合无法解密**。

- **字段脱敏**：`@FieldSensitive`，内置手机号/邮箱/银行卡号等 9 种规则。

```java
@FieldSensitive("testStrategy")
private String username;

@Configuration
public class SensitiveStrategyConfig {

    /**
     * 注入脱敏策略
     */
    @Bean
    public ISensitiveStrategy sensitiveStrategy() {
        // 自定义 testStrategy 类型脱敏处理
        return new SensitiveStrategy().addStrategy("testStrategy", t -> t + "***test***");
    }
}

// 跳过脱密处理，用于编辑场景
RequestDataTransfer.skipSensitive();
```

- **多数据源分库分表（读写分离）**：`@Sharding`，组内节点默认随机选择（查从写主）。

```yaml
mybatis-mate:
  sharding:
    health: true # 健康检测
    primary: mysql # 默认选择数据源
    datasource:
      mysql: # 数据库组
        - key: node1
          ...
        - key: node2
          cluster: slave # 从库读写分离时候负责 sql 查询操作，主库 master 默认可以不写
          ...
      postgres:
        - key: node1 # 数据节点
          ...
```

```java
@Mapper
@Sharding("mysql")
public interface UserMapper extends BaseMapper<User> {

    @Sharding("postgres")
    Long selectByUsername(String username);

}
```

```java
// 切换到 mysql 从库 node2 节点
ShardingKey.change("mysqlnode2");
```

- 另支持多数据源动态加载卸载、多数据源事务（JTA Atomikos）。
