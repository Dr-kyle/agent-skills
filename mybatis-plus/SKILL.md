---
name: mybatis-plus
description: "Use this skill for any task involving MyBatis-Plus — the Java (Spring Boot / Solon) MyBatis enhancement framework (com.baomidou). Covers dependency & starter selection, BaseMapper/IService CRUD, 条件构造器 Wrapper（QueryWrapper/LambdaQueryWrapper), 分页/乐观锁/多租户/数据权限等插件, 实体注解（@TableName/@TableId/@TableField/@TableLogic/@Version), application.yml 配置, 逻辑删除, 自动填充, 通用枚举, JSON 字段 TypeHandler, 雪花算法主键, 多数据源 @DS, SQL 注入器, 代码生成器 FastAutoGenerator, and troubleshooting. Triggers include writing a Mapper/Service, building dynamic queries, adding pagination, configuring MybatisPlusInterceptor, mapping an entity to a table, generating CRUD code, or debugging errors like Invalid bound statement (not found) / 更新 null 不生效 / 分页 total 不对 / 主键精度丢失."
---

# MyBatis-Plus 开发指南

MyBatis-Plus（MP）是 MyBatis 的**增强工具**，在不改变 MyBatis 任何既有行为的前提下，为单表操作提供一套零 SQL 的通用能力。

> 官方文档：<https://baomidou.com> · 源码：<https://github.com/baomidou/mybatis-plus>
> 本 skill 基于 **3.5.17** 文档整理。API 以项目实际依赖版本为准。

⚠️ **不要用 `mybatis.plus` 这个域名查文档。** 它停留在 VuePress 时代的 3.4.x 内容，且页面已被注入大量博彩 SEO 垃圾链接。当前唯一官方文档站是 **baomidou.com**。

## 核心心智模型（先理解这一点，能避免 80% 的困惑）

**MP 只做"增强"，不做"替代"。** 引入 MP 后，MyBatis 的 XML、`@Select`、动态 SQL、TypeHandler 全部照常工作；MP 额外做的事只有一件——在启动时**向每个继承 `BaseMapper<T>` 的接口注入一批通用方法**（`insert`/`selectById`/`selectList`...），这些方法的 SQL 由实体类的注解和全局配置推导生成。

由此推出三个必须记住的推论：

1. **实体注解就是 SQL 的真相来源。** 字段没插进去、没查出来、更新被忽略，先看 `@TableField`/`@TableId`，再看 yml，不要去翻 SQL 日志猜。
2. **MP 的特性只作用于 MP 注入的方法。** 逻辑删除、自动填充、乐观锁对你手写的 XML / `@Select` 一律**不生效**——手写 SQL 就得手写 `AND deleted = 0`。这是新手最高频的认知错误。
3. **插件是 SQL 改写器，注册顺序即执行顺序。** 分页、多租户、数据权限都靠改写 SQL 实现，顺序错了不会报错，只会悄悄给出错误结果。

### BaseMapper vs IService 怎么选

| | `BaseMapper<T>` | `IService<T>` / `ServiceImpl` |
|---|---|---|
| 定位 | 物理数据访问层 | 逻辑数据访问层 |
| 一次调用 | 恰好一条 SQL | 可能隐含多条 SQL |
| 返回值 | 影响行数 `int` / 实体 | 多为 `boolean` |
| 独有能力 | —— | `saveBatch`、`saveOrUpdate`、链式 `lambdaQuery()` |

**需要知道"到底改了几行"时，退回 `BaseMapper`**——`IService` 的 `boolean` 会把"更新了 0 行"和"更新成功"都算作成功语义之外的信息丢掉。

## 依赖引入（选错 starter 会直接启动失败）

`groupId` 统一 `com.baomidou`。按 Spring Boot 大版本选 starter：

| 环境 | artifactId |
|---|---|
| Spring Boot 2.x | `mybatis-plus-boot-starter` |
| Spring Boot 3.x | `mybatis-plus-spring-boot3-starter` |
| Spring Boot 4.x | `mybatis-plus-spring-boot4-starter` |
| Solon | `mybatis-plus-solon-plugin` |

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.17</version>
</dependency>
```

**硬性约束：**

- ⚠️ **引入 MP 后，不要再引入 `mybatis`、`mybatis-spring`、`mybatis-spring-boot-starter`。** starter 已传递依赖它们，重复引入会导致版本错配，报各种诡异的 `NoSuchMethodError`。这是 `Invalid bound statement` 的第一大成因。
- ⚠️ **3.5.9 起，依赖 JSQLParser 的插件（分页、多租户、动态表名、数据权限、防全表）被拆成了可选模块。** 只引 starter 的话这些插件用不了，必须额外引入：JDK 11+ 用 `mybatis-plus-jsqlparser`，JDK 8 用 `mybatis-plus-jsqlparser-4.9`（JSQLParser 5.0+ 已不支持 JDK 8）。
- 自 3.5.4 起，**不使用**上述任一 starter 时，需自行引入 `mybatis-spring`。

推荐用 BOM 统一版本，避免 starter 与 jsqlparser 版本漂移：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-bom</artifactId>
            <version>3.5.17</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 最小可用工程

```java
// 1. 实体
@TableName("sys_user")
public class User {
    @TableId(type = IdType.ASSIGN_ID)
    private Long id;
    private String name;
    private Integer age;
    @TableLogic
    private Integer deleted;
}

// 2. Mapper：继承即拥有全部单表 CRUD
public interface UserMapper extends BaseMapper<User> {}

// 3. 启动类或配置类上扫描
@MapperScan("com.example.**.mapper")
```

即可直接使用 `userMapper.selectById(1L)`。`BaseMapper` 与 `IService` 的完整方法目录见 `references/core-api.md`。

## 条件构造器 Wrapper（最高频 API）

**默认用 Lambda 版**（`LambdaQueryWrapper` / `lambdaQuery()`），用方法引用代替字符串列名，重构和改字段名时编译期就能发现问题。

### condition 参数：动态查询的标准写法

每个条件方法都有一个 `boolean condition` 首参重载，为 `false` 时该条件**完全不进入 SQL**。用它代替 `if` 包裹：

```java
queryWrapper.like(StringUtils.isNotBlank(name), Entity::getName, name)
            .eq(age != null && age >= 0, Entity::getAge, age);
```

⚠️ 附带效应：传入的 `Collection`/`Map` 为空时，条件同样被**静默丢弃**——`in(User::getId, emptyList())` 不会返回空结果，而是**该条件消失**，可能导致全表返回。

### and / or 嵌套：最高频的逻辑 bug

`or()` 只改变**紧随其后的一个**条件的连接符，**不加括号**。而 SQL 中 `AND` 优先级高于 `OR`：

```java
// ❌ 错误
wrapper.eq(User::getName, "李白").or().eq(User::getName, "杜甫").eq(User::getStatus, "alive");
// WHERE name = '李白' OR name = '杜甫' AND status = 'alive'
// 实际语义：name='李白' OR (name='杜甫' AND status='alive')

// ✅ 正确：显式嵌套
wrapper.and(i -> i.eq(User::getName, "李白").or().eq(User::getName, "杜甫"))
       .eq(User::getStatus, "alive");
// WHERE (name = '李白' OR name = '杜甫') AND status = 'alive'
```

### 安全红线

⚠️ **`orderByAsc`/`orderByDesc`/`select()` 的列名、以及 `apply`/`last`/`inSql`/`setSql` 的内容，都是字符串直接拼接进 SQL，参数绑定保护不到。** 任何来自前端的排序字段、列名，**必须走白名单校验**。值传递一律用 `apply("name = {0}", value)` 的占位符形式，不要字符串拼接。

完整方法表、生成 SQL 对照、`likeLeft`/`likeRight` 的反直觉命名等见 `references/wrapper.md`。

## 插件注册：顺序决定正确性

所有功能插件都是 `InnerInterceptor`，挂在唯一的 `MybatisPlusInterceptor` 下，**按注册顺序执行**。

> **排序规则（官方原文）：对 SQL 进行单次改造的插件应优先放入，不对 SQL 进行改造的插件最后放入。**
>
> 1. 多租户 `TenantLine` / 动态表名 `DynamicTableName` / 数据权限 `DataPermission`
> 2. 分页 `Pagination` / 乐观锁 `OptimisticLocker`
> 3. SQL 规范 `IllegalSQL` / 防全表 `BlockAttack`
>
> 官方另有明确要求：**使用多个插件时，分页插件必须放在执行链最后**，否则 COUNT SQL 不准确。

```java
@Configuration
@MapperScan("com.example.**.mapper")
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL)); // 如果配置多个插件, 切记分页最后添加
        return interceptor;
    }
}
```

⚠️ 分页插件注册在多租户/数据权限**之前**是典型事故：records 正确但 total 是全表行数，**不报错、不告警**。

⚠️ 始终显式传 `DbType`，不要依赖自动推断。

各插件的完整配置项与坑见 `references/plugins.md`。

## 主键与基线配置

**主键策略选型：默认全局 `ASSIGN_ID`（雪花算法 Long）**，实体上就不用再写 `type`。

| 场景 | IdType |
|---|---|
| 绝大多数情况 | `ASSIGN_ID` |
| 单库、需要数据库自增单调 | `AUTO` |
| Oracle/PG sequence、或业务自己定主键 | `INPUT`（配 `@KeySequence`） |
| String 主键且必须去中心化 | `ASSIGN_UUID`（代价：InnoDB 页分裂） |

⚠️ `ID_WORKER` / `ID_WORKER_STR` / `UUID` 已废弃，新代码不要用。
⚠️ 雪花 ID 是 19 位 Long，**超出 JS `Number` 安全范围**，返回前端必须序列化为 String（Jackson 注册 `ToStringSerializer`），否则前端拿到的 ID 末两位变 `0`。
⚠️ 雪花算法的 `workerId` 默认由 MAC+PID 推导，**容器化多副本可能算出相同 workerId 导致跨实例 ID 重复**，集群务必显式 `new DefaultIdentifierGenerator(workerId, datacenterId)`。

推荐基线 `application.yml`：

```yaml
mybatis-plus:
  mapper-locations: classpath*:/mapper/**/*.xml
  type-aliases-package: com.example.domain.entity
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
    call-setters-on-nulls: true
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
  global-config:
    banner: false
    db-config:
      id-type: ASSIGN_ID
      table-underline: true
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

⚠️ `mapper-locations` 若要覆盖，**必须用 `classpath*:`（带星号）**，单星号在多模块/JAR 场景下加载不到，直接导致 `Invalid bound statement`。
⚠️ `map-underscore-to-camel-case` 在 `mybatis-plus.configuration` 下，不是 `global-config` 下；层级写错**不报错**，只是字段全 null。
⚠️ `configuration` 与 `config-location` 互斥，不能同时配置。

全部注解属性与配置项见 `references/annotation-config.md`。

## 高频特性速查

**逻辑删除** —— 全局配 `logic-delete-field`，或字段上加 `@TableLogic`。⚠️ 填的是**实体属性名，不是列名**。⚠️ 只对 MP 注入方法生效。⚠️ 与唯一索引冲突时（`deleted` 用 0/1 只能删一次），改用 `bigint` 存删除时间戳。

**自动填充** —— 实现 `MetaObjectHandler`，配合 `@TableField(fill = FieldFill.INSERT_UPDATE)`：

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject mo) {
        strictInsertFill(mo, "createTime", LocalDateTime.class, LocalDateTime.now());
    }
    @Override
    public void updateFill(MetaObject mo) {
        strictUpdateFill(mo, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

⚠️ 单参数的 `update(Wrapper)` **完全不触发** `updateFill`。⚠️ `strictXxxFill` 的类型写错会静默跳过。

**`Db` 静态工具类** —— 免注入 Service，避免为了一次查询而循环依赖：

```java
List<User> list = Db.lambdaQuery(User.class).eq(User::getStatus, 1).list();
User u = Db.getById(id, User.class);
Db.saveBatch(list);
```

**JSON 字段** —— 用 `JacksonTypeHandler`，⚠️ **必须同时在实体上加 `@TableName(autoResultMap = true)`**，否则写入正常但查出来永远是 `null`。

**批量插入** —— ⚠️ MySQL 下 `saveBatch` 不加 JDBC 参数 `rewriteBatchedStatements=true` 等于白批，驱动仍逐条发送。

以上及多数据源、枚举、SQL 注入器、p6spy 等见 `references/features.md`。

## 参考文档索引

需要以下能力时，读对应的 reference 文件：

| 文件 | 内容 |
|---|---|
| `references/core-api.md` | `BaseMapper` / `IService` 完整方法目录、链式调用、批量操作、流式查询、安装与最小工程 |
| `references/wrapper.md` | Wrapper 全量方法表与生成 SQL 对照、and/or 嵌套、动态条件、注入风险 |
| `references/annotation-config.md` | 全部实体注解及属性、`IdType`/`FieldStrategy` 枚举对比、`mybatis-plus.*` 全量配置项 |
| `references/plugins.md` | `MybatisPlusInterceptor` 顺序规则、分页/乐观锁/多租户/数据权限/动态表名/防全表 各插件配置与坑 |
| `references/features.md` | 逻辑删除、自动填充、通用枚举、TypeHandler、主键生成器、多数据源、p6spy、SQL 注入器、DDL、`Db` 工具类 |
| `references/code-generator.md` | `FastAutoGenerator`（3.5.1+ Builder API）完整配置与模板引擎选择 |
| `references/troubleshooting.md` | **报错速查表** —— 按错误信息/现象定位原因与修复；含官方对 CVE 的回应 |

## 遇到报错时

先查 `references/troubleshooting.md`，它按错误信息组织。最高频的几个：

- **`Invalid bound statement (not found)`** → ①jar 冲突（重复引入了 mybatis/mybatis-spring）②`@MapperScan` 路径不对 ③实体没有主键（`@TableId`，或字段名叫 `id`）④用了原生 `SqlSessionFactory` 而非 `MybatisSqlSessionFactory` ⑤IDEA 默认不把 `src/main/java` 下的 XML 拷进产物，改用 Maven/Gradle 构建。
- **更新 `null` / 空串不生效** → `FieldStrategy` 默认 `NOT_NULL`。用 `UpdateWrapper.set(User::getEmail, null)`，或字段级 `@TableField(updateStrategy = FieldStrategy.ALWAYS)`。**不要全局改成 ALWAYS。**
- **主键末两位永远是 0** → JS `Number` 精度丢失，Long 主键序列化为 String 返回。
- **自动填充对 Long 字段不生效** → 用了 `long` 而非 `Long`；`long` 默认值 `0` 不是 `null`，MP 只在 `null` 时填充。
- **SQL 里出现两个 `limit`** → 注册了两个分页拦截器，只留一个。
- **分页 total 是全表行数** → 分页插件注册在多租户/数据权限之前了。
- **自定义 XML 里拿不到 `Page` 参数** → `Page` 继承 `RowBounds`，Mapper 中取不到；改传 `@Param("page") int page, @Param("size") int size`。

## 代码约定

- 一律用 Lambda 版 Wrapper（`lambdaQuery()` / `LambdaQueryWrapper`），禁止字符串列名硬编码。
- 动态条件用 `condition` 首参，不要用 `if` 包裹 Wrapper 调用。
- 需要"改了几行"就用 `BaseMapper`；纯业务编排用 `IService`；一次性查询用 `Db` 静态工具。
- 链式调用**必须以终结方法结尾**（`list()`/`one()`/`page()`/`update()`/`remove()`），漏写不报错且不执行。
- 前端传入的排序字段、列名走白名单；值一律用 `{0}` 占位符。
- 手写 XML/`@Select` 时，自行补齐逻辑删除条件与租户条件——MP 的插件不会管你手写的 SQL（除非走 JSQLParser 改写的插件）。
- 实体的 `Long` 主键返回前端前转 String。
- `delete(null)` / `remove(空 wrapper)` 会全表删除，写删除逻辑时显式断言条件非空。
