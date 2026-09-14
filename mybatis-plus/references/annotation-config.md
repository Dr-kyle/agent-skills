# MyBatis-Plus 3.5.17 注解与配置参考

覆盖 `com.baomidou.mybatisplus.annotation.*` 全部实体注解，以及 `mybatis-plus.*` 配置树。注解决定单实体行为、配置决定全局默认值，**注解优先级永远高于全局配置**——排查"字段没插进去/没查出来"时先看注解再看 yml。

---

## PART 1 注解

### @TableName

作用于类，指定实体对应的表名。实体名与表名一致（或表名是实体名的下划线形式，且 `table-underline: true`）时可省略。

```java
@TableName("sys_user")
public class User {
    private Long id;
    private String name;
    private Integer age;
    private String email;
}
```

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| value | String | `""` | 数据库表名。实体名与表名不一致时必填 |
| schema | String | `""` | 数据库 Schema 名。库未用 Schema 组织表时不填 |
| keepGlobalPrefix | boolean | `false` | 全局配置了 `tablePrefix` 时，是否仍然叠加全局前缀。`true` 时即使注解里写了表名，也会自动拼上全局前缀 |
| resultMap | String | `""` | XML 中已定义的 ResultMap 的 id，用于把查询结果映射到实体 |
| autoResultMap | boolean | `false` | 是否自动构建 resultMap。**已设置 `resultMap` 时本属性不生效** |
| excludeProperty | String[] | `{}` | 映射时需排除的属性名，不参与生成的 SQL。@since 3.3.1 |

#### autoResultMap 什么时候必须开

⚠️ **只要实体上有字段依赖 `typeHandler` 或 `jdbcType` 参与"查询结果"的封装，就必须 `@TableName(autoResultMap = true)`。**

MyBatis 只允许在两处写 typeHandler：

1. 定义在 `resultMap` 中——作用于**查询结果的封装**；
2. 定义在 insert/update 语句的 `#{property}` 后面，如 `#{property,typehandler=xxx.xxx.xxx}`——**只作用于当前设置的值**。

MyBatis-Plus 生成的通用 `insert/update` 属于第 2 种，所以 `@TableField(typeHandler=...)` / `jdbcType` **在不开 autoResultMap 时只在 update（写入）方向生效**，查询回来会是原始类型或报错。开了 autoResultMap 后 MP 自动构建 resultMap 并注入 MyBatis，读方向才正常。

典型必开场景：JSON 字段（`JacksonTypeHandler` / `FastjsonTypeHandler`）、PG 数组、加密字段的自定义 handler。

⚠️ 注入后生成的 resultMap 是**静态**的（等价于写死在 XML 里），运行期改不了。

另外：如果字段类型和 typeHandler 处理的值类型是 equals 关系，只要保证 typeHandler 被 MyBatis 加载到（`type-handlers-package`），可以不用注解。

### @TableId

标记主键字段。主键字段名恰好为 `id` 时可省略该注解（但显式写上更好，尤其要指定 `type`）。

```java
@TableName("sys_user")
public class User {
    @TableId
    private Long id;
    private String name;
    private Integer age;
    private String email;
}
```

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| value | String | `""` | 主键列名。不设置则用实体字段名作为主键列名 |
| type | Enum | `IdType.NONE` | 主键生成策略 |

#### IdType 全枚举

| 枚举值 | 版本 | 语义 | 适用主键类型 | 生成时机 |
|---|---|---|---|---|
| `AUTO` | — | 数据库自增 ID | 数值型 | DB 侧生成，插入后回写 |
| `NONE` | — | 无策略，**跟随全局 `global-config.db-config.id-type`** | 视全局而定 | 视全局而定 |
| `INPUT` | — | 插入前由用户自行设置主键值 | 任意 | 应用侧手动 / 序列 |
| `ASSIGN_ID` | 3.3.0 | 自动分配 ID，默认雪花算法，走 `IdentifierGenerator#nextId` | Long / Integer / String | 应用侧，插入前 |
| `ASSIGN_UUID` | 3.3.0 | 自动分配 UUID，走 `IdentifierGenerator#nextUUID` | String | 应用侧，插入前 |
| ~~`ID_WORKER`~~ | 已弃用 | 旧雪花 | Long | 用 `ASSIGN_ID` 替代 |
| ~~`ID_WORKER_STR`~~ | 已弃用 | 旧雪花字符串 | String | 用 `ASSIGN_ID` 替代 |
| ~~`UUID`~~ | 已弃用 | 旧 UUID | String | 用 `ASSIGN_UUID` 替代 |

⚠️ 已弃用的 `ID_WORKER` / `ID_WORKER_STR` / `UUID` 一律不要再用，新策略在灵活性和兼容性上都更好。

选型建议：

- **`ASSIGN_ID`（默认，推荐）**：分布式友好、无需回表、可提前拿到 ID 组装关联数据。Long 主键 + 雪花是绝大多数业务的正确答案。⚠️ 雪花 ID 是 19 位 long，传给 JS 前端会丢精度，需序列化为 String（Jackson `@JsonSerialize(using = ToStringSerializer.class)` 或全局 Long→String）。
- **`AUTO`**：只在单库单表、强依赖 DB 自增语义（如需要 ID 严格递增做游标分页）时用。
- **`INPUT`**：Oracle 序列（配 `@KeySequence`）、或业务自带主键（订单号、身份证号）。
- **`ASSIGN_UUID`**：只在必须无序、无中心化 ID、且主键为 String 时用。⚠️ 无序 UUID 做聚簇索引主键会导致 InnoDB 页分裂，写入性能差，能用 `ASSIGN_ID` 就别用它。
- **`NONE`**：实体上不写，等于跟随全局。生产建议全局设 `ASSIGN_ID`，实体不写 type。

⚠️ **AUTO 的坑**：该策略下**保存前手动设置的 ID 无效，且会被数据库自动生成的 ID 回写覆盖**。正确用法是 `insert` 成功后从实体上读回自动生成的 ID 再去关联业务。另外 `AUTO` 要求 DB 列真的是 `AUTO_INCREMENT`，否则插入报主键不能为空；而全局 `id-type: auto` + 某张表没有自增列，会在运行时才暴露。

### @TableField

标记非主键字段。字段名为驼峰且与数据库下划线列名对应（`map-underscore-to-camel-case: true`）时可省略。

```java
@TableName("sys_user")
public class User {
    @TableId
    private Long id;
    @TableField("nickname") // 映射到数据库字段 "nickname"
    private String name;
    private Integer age;
    private String email;
}
```

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| value | String | `""` | 数据库列名 |
| exist | boolean | `true` | 该字段是否存在于表中。`false` 时 MP 生成 SQL 完全忽略它 |
| condition | String | `""` | EntityQuery 时该字段在 WHERE 中的条件表达式。有值以设置值为准，无值用全局 `%s=#{%s}`。写法见 `SqlCondition` |
| update | String | `""` | UPDATE 的 SET 子句中该字段的表达式，**优先级高于 el** |
| insertStrategy | Enum | `FieldStrategy.DEFAULT` | INSERT 时字段验证策略 |
| updateStrategy | Enum | `FieldStrategy.DEFAULT` | UPDATE SET 子句的字段验证策略 |
| whereStrategy | Enum | `FieldStrategy.DEFAULT` | UPDATE 的 WHERE 子句（Wrapper 内部 Entity 生成）字段验证策略 |
| fill | Enum | `FieldFill.DEFAULT` | 自动填充策略，需配合 `MetaObjectHandler` |
| select | boolean | `true` | 查询时是否包含在 SELECT 列表中 |
| jdbcType | JdbcType | `JdbcType.UNDEFINED` | 显式指定 JDBC 类型 |
| typeHandler | Class<? extends TypeHandler> | `UnknownTypeHandler.class` | 字段类型处理器 |
| numericScale | String | `""` | 小数点后保留位数，**仅 update 生效** |
| keepGlobalFormat | boolean | `false` | 是否保持全局 `DbConfig.columnFormat` 的列格式化规则 |

#### FieldStrategy 全枚举

| 枚举值 | 语义 |
|---|---|
| `DEFAULT` | 遵循全局配置策略；全局未指定时，默认仅在字段值不为 NULL 时处理该字段 |
| `ALWAYS` | 总是处理该字段，无论值是否为 NULL |
| `NOT_NULL` | 仅在值不为 NULL 时处理 |
| `NOT_EMPTY` | 字符串不为空、其他类型不为 NULL 时处理 |
| `NEVER` | 从不处理该字段，即使值不为 NULL |
| ~~`IGNORED`~~ | 忽略判断，效果等同 `ALWAYS`。**@Deprecated，用 `ALWAYS`** |

⚠️ 最高频的坑：`updateById` 想把某列**更新成 NULL**，默认 `NOT_NULL` 策略会直接跳过该字段，SQL 里根本没有它。解法二选一——该字段加 `@TableField(updateStrategy = FieldStrategy.ALWAYS)`（推荐，影响面可控），或改用 `UpdateWrapper.set("col", null)`。不要为此把全局 `update-strategy` 调成 `ALWAYS`，否则所有未赋值字段都会被写成 NULL。

`condition` 示例（EntityQuery = 用实体实例构建 Wrapper 条件，而非手写 SQL 片段）：

```java
@TableField(condition = "%s > #{%s}") // 自定义 age 字段的条件表达式
private Integer age;
```

```java
User queryEntity = new User();
queryEntity.setAge(18);
QueryWrapper<User> queryWrapper = new QueryWrapper<>(queryEntity);
List<User> userList = userMapper.selectList(queryWrapper);
// SELECT * FROM sys_user WHERE age > 18;
```

`update` 示例（乐观锁之外的自增列）：

```java
@TableField(update="%s+1") // 自定义更新时的表达式
private Integer version;
// UPDATE sys_user SET name=..., version = version + 1 WHERE id = 1;
```

`insertStrategy` 示例：

```java
@TableField(insertStrategy = FieldStrategy.NOT_EMPTY) // 仅在 nickname 不为空时插入
private String nickname;
```

等价于手写：

```xml
<insert id="insertUser" parameterType="com.example.entity.User">
    INSERT INTO sys_user (
        <if test="nickname != null and nickname != ''">
            nickname,
        </if>
        age,
        email
    ) VALUES (
        <if test="nickname != null and nickname != ''">
            #{nickname},
        </if>
        #{age},
        #{email}
    )
</insert>
```

`whereStrategy` 只影响 `new UpdateWrapper<>(whereEntity)` 这种由实体推导出的 WHERE 条件，不影响手写 `.eq()`。

#### FieldFill 枚举

| 枚举值 | 语义 |
|---|---|
| `DEFAULT` | 不填充，依赖数据库默认值或手动设置 |
| `INSERT` | 插入时自动填充 |
| `UPDATE` | 更新时自动填充 |
| `INSERT_UPDATE` | 插入和更新都自动填充 |

```java
@TableField(fill = FieldFill.INSERT)
private LocalDateTime createTime;

@TableField(fill = FieldFill.UPDATE)
private LocalDateTime updateTime;
```

⚠️ 必须同时注册 `MetaObjectHandler` Bean，否则注解形同虚设、无任何报错。`updateTime` 用 `INSERT_UPDATE` 而非 `UPDATE` 更符合直觉（插入时也应有值）。

#### select = false

```java
@TableField(select = false)
private String password;
```

生成的 SELECT 列表中不含 `password`。⚠️ 只影响 MP 生成的语句，自己写的 XML / `selectMaps` 自定义 SQL 不受约束；且该字段查询结果恒为 null，若同一实体拿去做 `updateById` 且策略是 `ALWAYS`，会把密码刷成 NULL。

#### numericScale / jdbcType / typeHandler 生效范围（务必记住）

- `numericScale` **仅在 update 操作时生效**。
- `jdbcType` 和 `typeHandler` 如果不与 `@TableName(autoResultMap = true)` 一起使用，也**仅在 update 时生效**。
- `typeHandler`：若字段类型与设置的值类型是 equals 关系，只需确保 typeHandler 被 MyBatis 加载到，无需注解。

### @Version — 乐观锁

```java
@Version // 标记为乐观锁版本号字段
private Integer version;
```

更新时自动带上 `WHERE version = #{oldVersion}` 并 `SET version = oldVersion + 1`；影响行数为 0 说明被其他事务改过。⚠️ 必须注册 `OptimisticLockerInnerInterceptor`；仅对 `updateById` / `update(entity, wrapper)` 生效，且 entity 的 version 字段必须有值，为 null 时跳过乐观锁。支持类型：int/Integer/long/Long/Date/Timestamp/LocalDateTime。

### @TableLogic — 逻辑删除

```java
@TableLogic(value = "0", delval = "1") // 逻辑删除字段
private Integer deleted;
```

| 属性 | 类型 | 说明 |
|---|---|---|
| value | String | 逻辑**未**删除值 |
| delval | String | 逻辑**已**删除值 |

行为：查询自动追加 `AND deleted = 0`；`deleteById` 转为 `UPDATE ... SET deleted = 1`；更新同样只作用于未删除行。

⚠️ 只对 MP 自动注入的方法和 Wrapper 生效，**自定义 XML / `@Select` 的 SQL 必须自己写删除条件**。另外逻辑删除会让唯一索引失效（同一业务键的多条已删记录冲突），要么把 `deleted` 加进唯一索引，要么删除时把唯一列打上时间戳后缀。

### @EnumValue — 通用枚举

标记枚举类中"存库的那个属性"，可标在字段或 getter 上。

```java
public enum Gender {
    MALE("M", "男"),
    FEMALE("F", "女");

    private String code;
    private String description;

    Gender(String code, String description) {
        this.code = code;
        this.description = description;
    }

    @EnumValue // 指定存储到数据库的枚举值为 code
    public String getCode() {
        return code;
    }
}
```

3.5.2 起通用枚举**无需任何配置**即可使用（默认处理器为 `CompositeEnumTypeHandler`，检测到实现 `IEnum` 或带 `@EnumValue` 的枚举就走 `MybatisEnumTypeHandler`，其余枚举走 `defaultEnumTypeHandler`）。`type-enums-package` 从 3.5.2 起**已失效**，写了也没用。

### @OrderBy — 默认排序

```java
@OrderBy(asc = false, sort = 10) // 指定默认排序为倒序，优先级为10
private Integer age;
```

| 属性 | 类型 | 说明 |
|---|---|---|
| asc | boolean | `true` 升序，`false` 降序 |
| sort | int | 优先级，**数字越小越先应用** |

⚠️ 优先级低于 Wrapper 中显式指定的排序——Wrapper 里写了 `orderBy`，注解的默认排序被覆盖。别在大表的非索引列上加 `@OrderBy`，它会污染所有查询。

### @KeySequence — 序列主键（Oracle/DB2/PG 等）

```java
@TableName("sys_user")
@KeySequence("SEQ_USER_ID") // 指定序列名称为 "SEQ_USER_ID"
public class User {
    @TableId(type = IdType.INPUT) // 使用序列生成主键
    private Long id;
}
```

| 属性 | 类型 | 说明 |
|---|---|---|
| value | String | 序列名称 |
| dbType | DbType | 数据库类型。未指定时使用注入的 `IKeyGenerator` 实现；⚠️ **存在多个 `IKeyGenerator` 实现时必须指定 dbType** |

必须配 `@TableId(type = IdType.INPUT)` 与一个 `IKeyGenerator` Bean（如 `OracleKeyGenerator`）。

### @InterceptorIgnore — 跳过插件

标在 Mapper 接口上（作用于全部方法）或某个方法上。

```java
// @InterceptorIgnore(tenantLine = "1") // 忽略多租户拦截器
public interface UserMapper extends BaseMapper<User> {

    @InterceptorIgnore(tenantLine = "1") // 忽略多租户拦截器
    List<User> selectUsers();
}
```

MP 每个插件在该注解上都有对应属性（多租户为 `tenantLine`，还有 `dataPermission`、`blockAttack`、`illegalSql`、`dynamicTableName` 等）。取值 `"1"`、`"yes"`、`"on"` 表示**忽略**该插件；`"0"`、`"false"`、`"off"` 或为空表示插件正常执行。

### 字段忽略的三种方式对比

| 方式 | 写法 | 对 MP 生成 SQL | 可被序列化(JSON/RPC) | 备注 |
|---|---|---|---|---|
| 注解忽略 | `@TableField(exist = false)` | 完全忽略该字段 | ✅ 可以 | **推荐**。DTO 式扩展字段（如 `roleNames`）用这个 |
| transient | `private transient String token;` | 忽略 | ❌ Java 原生序列化丢失，部分框架也会跳过 | 需要跨进程传输时别用 |
| static | `private static String TYPE;` | 忽略 | ❌ 非实例状态 | 只适合常量，不能承载每行数据 |

⚠️ 结论：业务上只是"表里没这列但要返回给前端"的字段一律用 `@TableField(exist = false)`；`transient`/`static` 是副作用型手段，别当作忽略机制使用。

---

## PART 2 配置

### 使用方式

Spring Boot（`application.yml`）：

```yaml
mybatis-plus:
  configuration:
    # MyBatis 配置
    map-underscore-to-camel-case: true
  global-config:
    # 全局配置
    db-config:
      # 数据库配置
      id-type: auto
```

Spring MVC（XML）——注意 SqlSessionFactory 必须换成 MP 的：

```xml
<bean id="sqlSessionFactory" class="com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath*:mapper/**/*.xml"/>
    <property name="typeAliasesPackage" value="com.your.domain"/>
    <!-- 其他配置 -->
</bean>
```

Mapper 扫描：

```java
@SpringBootApplication
@MapperScan("com.baomidou.mybatisplus.samples.quickstart.mapper")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### mybatis-plus.*（Base）

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| config-location | String | `null` | MyBatis 配置文件位置，如 `classpath:/mybatis-config.xml`。有独立 MyBatis 配置文件时才用 |
| mapper-locations | String[] | `["classpath*:/mapper/**/*.xml"]` | Mapper XML 位置。Mapper 里有自定义方法时需要配 |
| type-aliases-package | String | `null` | 别名包扫描路径，注册后 XML 中可直接用类名而非全限定名 |
| type-aliases-super-type | Class<?> | `null` | 与 `type-aliases-package` 配合，仅扫描指定父类的子类 |
| type-handlers-package | String | `null` | TypeHandler 扫描路径，注册自定义类型转换器 |
| type-enums-package | String | `null` | ⚠️ **3.5.2 起该配置无效**，通用枚举无需配置即可使用 |
| check-config-location | boolean | `false` | 启动时是否检查 MyBatis XML 是否存在（Spring Boot Only） |
| executor-type | ExecutorType | `simple` | 执行器类型：`SIMPLE` / `REUSE` / `BATCH`（Spring Boot Only） |
| configuration-properties | Properties | `null` | 外部化 MyBatis Properties，如 `classpath:/mybatis-properties.properties` |
| configuration | Configuration | `null` | 原生 MyBatis 配置块 |
| global-config | GlobalConfig | `GlobalConfig::new` | MP 全局策略配置块 |

### mybatis-plus.configuration.*

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| map-underscore-to-camel-case | boolean | `true` | 下划线列名→驼峰属性名映射。MP 还用它生成最终 SQL 的 select body；库命名规范时无需 `@TableField` |
| default-enum-type-handler | Class<? extends TypeHandler> | `EnumTypeHandler`（3.5.2 起外层为 `CompositeEnumTypeHandler`） | 默认枚举处理器 |
| cache-enabled | boolean | `true` | 是否开启 MyBatis 二级缓存 |
| local-cache-scope | String | `SESSION` | 一级缓存：`SESSION` 同 Session 相同查询不再查库；`STATEMENT` 关闭一级缓存 |
| call-setters-on-nulls | boolean | `false` | 结果集值为 null 时是否调用 setter（Map 为 put）。用于依赖 `Map.keySet()` 或需要 null 初始化的场景 |
| aggressive-lazy-loading | boolean | `true` | 懒加载对象是否被任一懒属性触发全量加载，需配合 `lazy-loading-enabled` |
| auto-mapping-behavior | AutoMappingBehavior | `partial` | `NONE` 不自动映射 / `PARTIAL` 仅非嵌套 resultMap / `FULL` 全部 resultMap |
| auto-mapping-unknown-column-behavior | AutoMappingUnknownColumnBehavior | `NONE` | 遇未知列/属性：`NONE` 不处理 / `WARNING` 打日志 / `FAILING` 抛异常 |
| configuration-factory | Class<?> | `null` | 提供 Configuration 实例的工厂类，须有 `static Configuration getConfiguration()` |
| log-impl | Class | — | 日志实现。开发期常用 `org.apache.ibatis.logging.stdout.StdOutImpl` 打印 SQL |
| jdbc-type-for-null | JdbcType | `OTHER` | 传 null 时使用的 JDBC 类型。Oracle 需设为 `NULL` |

枚举处理器可选值：

| 值 | 语义 |
|---|---|
| `org.apache.ibatis.type.EnumTypeHandler` | 存储枚举的**名称** |
| `org.apache.ibatis.type.EnumOrdinalTypeHandler` | 存储枚举的**索引** |
| `com.baomidou.mybatisplus.extension.handlers.MybatisEnumTypeHandler` | 枚举需实现 `IEnum` 或字段标 `@EnumValue`（3.1.2 以下为 `EnumTypeHandler`） |
| `com.baomidou.mybatisplus.extension.handlers.EnumAnnotationTypeHandler` | 枚举字段需标 `@EnumValue` |

⚠️ 3.5.2 起默认为 `CompositeEnumTypeHandler`：通用枚举内部用 `MybatisEnumTypeHandler`，其他枚举用 `defaultEnumTypeHandler`。配 `default-enum-type-handler` **只改变 `CompositeEnumTypeHandler#defaultEnumTypeHandler` 的值**，不会影响通用枚举。

⚠️ 一级缓存：单服务架构开启只提升性能；**微服务架构应关闭**（`local-cache-scope: statement`）——Service1 查完后 Service2 改了数据，Service1 再查会拿到过期数据。

⚠️ `call-setters-on-nulls` 无法让基本类型（int、boolean）变成 null。

### mybatis-plus.global-config.*

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| banner | boolean | `true` | 是否在控制台打印 MP LOGO |
| enable-sql-runner | boolean | `false` | 是否初始化 `SqlRunner` |
| sql-injector | ISqlInjector | `DefaultSqlInjector` | SQL 注入器，注入通用方法。Starter 下支持 `@Bean` 注入 |
| super-mapper-class | Class | `com.baomidou.mybatisplus.core.mapper.Mapper.class` | 通用 Mapper 父类，只有其子类 Mapper 才注入 sqlInjector 的方法 |
| meta-object-handler | MetaObjectHandler | `null` | 字段自动填充控制器。Starter 下支持 `@Bean` 注入 |
| identifier-generator | IdentifierGenerator | `DefaultIdentifierGenerator` | ID 生成器（@since 3.3.0）。Starter 下支持 `@Bean` 注入 |
| db-config | GlobalConfig$DbConfig | `null` | DB 策略配置块 |

推荐用 `@Bean` 而非 yml 字符串配置这些扩展点：

```java
@Bean
public MetaObjectHandler metaObjectHandler() {
    return new MyMetaObjectHandler();
}

@Bean
public IdentifierGenerator identifierGenerator() {
    return new CustomIdentifierGenerator();
}

@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
    return interceptor;
}
```

### mybatis-plus.global-config.db-config.*

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| id-type | IdType | `ASSIGN_ID` | 全局默认主键类型 |
| table-prefix | String | `null` | 表名前缀，如 `tbl_` |
| schema | String | `null` | 数据库 Schema，通常不用设 |
| column-format | String | `null` | 生成 SQL 时格式化字段名（加前后缀），**对主键无效**，例 `%s` |
| table-format | String | `null` | 生成 SQL 时格式化表名，例 `tbl_%s`（@since 3.5.3.2） |
| property-format | String | `null` | Entity 字段映射到数据库字段时格式化，**仅 `column as property` 场景生效**，对主键无效（@since 3.3.0） |
| table-underline | boolean | `true` | 表名是否驼峰转下划线 |
| capital-mode | boolean | `false` | 表名与字段名是否使用大写命名 |
| key-generator | IKeyGenerator | `null` | 自定义表主键生成器。Starter 下支持 `@Bean` 注入 |
| logic-delete-field | String | `null` | 全局逻辑删除字段的**实体属性名**，仅逻辑删除开启时有效 |
| logic-delete-value | String | `1` | 逻辑已删除值 |
| logic-not-delete-value | String | `0` | 逻辑未删除值 |
| insert-strategy | FieldStrategy | `NOT_NULL` | Insert 时字段验证策略 |
| update-strategy | FieldStrategy | `NOT_NULL` | Update SET 子句字段验证策略 |
| where-strategy | FieldStrategy | `NOT_NULL` | Update 时 Wrapper 根据内部 Entity 生成的 WHERE 条件的字段验证策略 |

⚠️ 配了 `logic-delete-field: deleted` 后，实体上就不必再加 `@TableLogic`；但注解仍优先，两处不一致时以注解为准。`logic-delete-value` 支持 `true`/`false` 这类字面量（对应 `tinyint(1)`/`boolean` 列），也支持 `now()` 之类的函数（删除时间戳方案）。

### ⚠️ 高频坑

1. **`mapper-locations` 默认值已经是 `classpath*:/mapper/**/*.xml`**。把它改写成 `classpath:/mapper/**.xml`（单星 + 单层）会在 Maven 多模块 / 打成 JAR 后加载不到 XML，报 `Invalid bound statement (not found)`。**多模块项目必须以 `classpath*:` 开头**才能加载多个 JAR 包中的 XML。不需要自定义路径时干脆别配这一项。
2. **驼峰映射位置错**。`map-underscore-to-camel-case` 属于 `mybatis-plus.configuration.*`，不是 `global-config` 下；写错层级 Spring Boot 不会报错，只是静默不生效，表现为查出来的实体某些字段全是 null。同时注意 `configuration.map-underscore-to-camel-case`（列→属性）与 `db-config.table-underline`（表名驼峰转下划线）是两件不同的事，别混。另外 ⚠️ **`configuration` 与 `config-location` 互斥**，同时配会启动报错。
3. **`banner: false` 的层级**。是 `mybatis-plus.global-config.banner`，不是 `mybatis-plus.banner`。生产环境建议关掉。
4. **Spring Boot 3 的 starter 名不一样**。Boot 2.x 用 `mybatis-plus-boot-starter`；**Boot 3.x 必须用 `mybatis-plus-spring-boot3-starter`**（3.5.9+ 后 core/extension 也拆成了 `mybatis-plus-jsqlparser` 等模块，用到分页/多租户等解析型插件时需额外引入 `mybatis-plus-jsqlparser`）。用错 starter 会因 `spring.factories` → `AutoConfiguration.imports` 的迁移而完全不自动装配。
5. **`type-aliases-package` 配错等于没配**。它必须指向**实体所在包**（`com.your.domain`），不是 mapper 包、不是 XML 目录。配错时 XML 里的 `resultType="User"` 会报 `Cause: java.lang.ClassNotFoundException: Cannot find class: User`。只有 XML 中用短类名时才需要它；用全限定名可以完全不配。顺带：`type-enums-package` 3.5.2 起彻底无效，见到老配置直接删。

---

## PART 3 生产基线 yml

```yaml
mybatis-plus:
  # 仅在有自定义 XML 时保留；多模块必须 classpath*:
  mapper-locations: classpath*:/mapper/**/*.xml
  # 指向实体包,让 XML 能用短类名
  type-aliases-package: com.example.domain.entity
  # 自定义 TypeHandler(JSON/加密字段)放这里
  type-handlers-package: com.example.infra.typehandler
  configuration:
    # 下划线列 -> 驼峰属性,默认已是 true,显式写出防误改
    map-underscore-to-camel-case: true
    # 生产关二级缓存,避免脏读
    cache-enabled: false
    # 微服务务必关一级缓存
    local-cache-scope: statement
    # 结果为 null 也调 setter,避免 Map 结果丢 key
    call-setters-on-nulls: true
    # Oracle 改成 NULL;MySQL 保持默认即可
    jdbc-type-for-null: 'null'
    # 生产用 Slf4j,开发期临时换 StdOutImpl 打印 SQL
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
  global-config:
    # 关闭 LOGO
    banner: false
    db-config:
      # 雪花 ID:分布式安全,插入前可拿到 ID
      id-type: ASSIGN_ID
      # 表名驼峰转下划线
      table-underline: true
      # 逻辑删除三件套(实体无需再写 @TableLogic)
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
      # 保持默认 NOT_NULL:置 NULL 的需求用字段级 ALWAYS 或 UpdateWrapper.set
      insert-strategy: NOT_NULL
      update-strategy: NOT_NULL
      where-strategy: NOT_NULL
```

配套 Java 侧（yml 管不了的部分）：

```java
@Configuration
@MapperScan("com.example.domain.mapper")
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 顺序有讲究:多租户 -> 分页 -> 乐观锁
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
        return interceptor;
    }

    @Bean
    public MetaObjectHandler metaObjectHandler() {
        return new MyMetaObjectHandler();
    }
}
```

⚠️ 用到 `PaginationInnerInterceptor` / `TenantLineInnerInterceptor` / `DynamicTableNameInnerInterceptor` 等依赖 SQL 解析的插件时，3.5.9+ 需显式引入 `mybatis-plus-jsqlparser`（或 `mybatis-plus-jsqlparser-4.9`），否则启动即 `NoClassDefFoundError`。
