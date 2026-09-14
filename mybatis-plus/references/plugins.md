# MyBatis-Plus 插件体系（3.5.17）

插件全部挂在 `MybatisPlusInterceptor` 这一个 MyBatis `Interceptor` 之下。它代理了 `Executor#query`、`Executor#update`、`StatementHandler#prepare`，内部维护一个 `List<InnerInterceptor>`，按**注册顺序**依次执行。所有功能插件（分页、多租户、乐观锁、动态表名、数据权限、防全表攻击等）都是 `InnerInterceptor` 实现，不是独立的 MyBatis 插件。

⚠️ 版本要求：`MybatisPlusInterceptor` 架构为 3.4.0+。3.4.0 以下的 `PaginationInterceptor`（老式 MyBatis Interceptor）已废弃，不要混用。

⚠️ 3.5.9 起，依赖 JSQLParser 的插件（`PaginationInnerInterceptor`、`TenantLineInnerInterceptor`、`DataPermissionInterceptor`、`DynamicTableNameInnerInterceptor`、`IllegalSQLInnerInterceptor`、`BlockAttackInnerInterceptor` 等）从主包分离，必须额外引入 `mybatis-plus-jsqlparser` 依赖，否则启动即 `ClassNotFoundException` / 插件静默不生效。

---

## 1. 注册机制与添加顺序（最高频的坑）

只需要**一个** `MybatisPlusInterceptor` `@Bean`，在其中多次 `addInnerInterceptor(...)`。不要声明多个 `MybatisPlusInterceptor` Bean —— Spring Boot starter 只会装配其中一个，另一个的插件会全部失效（且无任何报错）。

### 顺序规则（记死）

```
┌─────────────────────────────────────────────────────────┐
│ 顺序规则：对 SQL 做"单次改造"的插件先放，                  │
│           不改造 SQL 的插件最后放。                        │
│                                                          │
│  1) 多租户 TenantLineInnerInterceptor                     │
│     动态表名 DynamicTableNameInnerInterceptor             │
│     数据权限 DataPermissionInterceptor                    │
│  2) 分页   PaginationInnerInterceptor                     │
│     乐观锁 OptimisticLockerInnerInterceptor               │
│  3) SQL 规范 IllegalSQLInnerInterceptor                   │
│     防全表   BlockAttackInnerInterceptor                  │
│                                                          │
│ ⚠️ 分页插件必须放在"改写 SQL 的插件"之后。                 │
│    否则 count SQL 在租户/数据权限条件拼接之前生成，         │
│    total 会算成全库总数 —— 分页数据对、总数错，            │
│    且不抛异常，线上极难发现。                              │
└─────────────────────────────────────────────────────────┘
```

官方原话：「使用多个插件时，需要注意它们的顺序。建议的顺序是：多租户、动态表名 → 分页、乐观锁 → SQL 性能规范、防止全表更新与删除。总结：对 SQL 进行单次改造的插件应优先放入，不对 SQL 进行改造的插件最后放入。」以及「在使用多个插件时，请将分页插件放到插件执行链的最后面，以避免 COUNT SQL 执行不准确的问题。」

### 完整多插件配置（Spring Boot）

```java
@Configuration
@MapperScan("com.yourpackage.mapper")
public class MybatisPlusConfig {

    @Autowired
    private CustomTenantHandler customTenantHandler;

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 1. 多租户（改写 SQL，最先）
        TenantLineInnerInterceptor tenantInterceptor = new TenantLineInnerInterceptor();
        tenantInterceptor.setTenantLineHandler(customTenantHandler);
        interceptor.addInnerInterceptor(tenantInterceptor);

        // 2. 动态表名（改写 SQL）
        DynamicTableNameInnerInterceptor dynamicTableNameInnerInterceptor = new DynamicTableNameInnerInterceptor();
        dynamicTableNameInnerInterceptor.setTableNameHandler((sql, tableName) -> tableName);
        interceptor.addInnerInterceptor(dynamicTableNameInnerInterceptor);

        // 3. 分页（必须在所有改写 SQL 的插件之后）
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));

        // 4. 乐观锁
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        // 5. 防止全表更新与删除（不改写 SQL，最后）
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());

        return interceptor;
    }
}
```

### Spring XML 配置

```xml
<bean id="sqlSessionFactory" class="com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean">
    <!-- 其他属性 略 -->
    <property name="configuration" ref="configuration"/>
    <property name="plugins">
        <array>
            <ref bean="mybatisPlusInterceptor"/>
        </array>
    </property>
</bean>


<bean id="mybatisPlusInterceptor" class="com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor">
    <property name="interceptors">
        <list>
            <ref bean="paginationInnerInterceptor"/>
        </list>
    </property>
</bean>


<bean id="paginationInnerInterceptor" class="com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor">
    <!-- 对于单一数据库类型来说,都建议配置该值,避免每次分页都去抓取数据库类型 -->
    <constructor-arg name="dbType" value="H2"/>
</bean>
```

### mybatis-config.xml 配置

```xml
<plugins>
  <plugin interceptor="com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor">
    <property name="@page" value="com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor"/>
    <property name="page:dbType" value="h2"/>
  </plugin>
</plugins>
```

---

## 2. 分页 PaginationInnerInterceptor

### 最小配置

```java
@Configuration
@MapperScan("scan.your.mapper.package")
public class MybatisPlusConfig {

    /**
     * 添加分页插件
     */
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL)); // 如果配置多个插件, 切记分页最后添加
        // 如果有多数据源可以不配具体类型, 否则都建议配上具体的 DbType
        return interceptor;
    }
}
```

### 插件属性

| 属性 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `overflow` | boolean | false | 页码溢出总页数后是否处理（true = 回到首页，false = 返回空列表） |
| `maxLimit` | Long | — | 单页条数硬上限，防止 `size=99999999` 拖垮数据库 |
| `dbType` | DbType | — | 数据库类型 |
| `dialect` | IDialect | — | 自定义方言实现类 |

⚠️ **`dbType` 必须显式配置**（单数据源场景）。不配时插件每次分页都要从 `Connection` 反查数据库类型，既有性能开销，也可能在连接池代理/多数据源下识别错误，生成错误的 limit 语法。只有真正的多数据源混合方言场景才留空。

⚠️ 生产环境务必设置 `maxLimit`，否则 `size` 由前端控制 = 内存溢出入口。

### Page 属性

| 属性 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `records` | List\<T\> | emptyList | 数据列表 |
| `total` | Long | 0 | 总记录数 |
| `size` | Long | 10 | 每页条数 |
| `current` | Long | 1 | 当前页 |
| `orders` | List\<OrderItem\> | emptyList | 排序字段 |
| `optimizeCountSql` | boolean | true | 自动优化 COUNT SQL |
| `optimizeJoinOfCountSql` | boolean | true | COUNT SQL 是否移除 join 部分 |
| `searchCount` | boolean | true | 是否执行 count 查询 |
| `maxLimit` | Long | — | 单页条数限制 |
| `countId` | String | — | XML 自定义 count 查询的 statementId |

### 用法

```java
// 构造方式：new 或 Page.of(current, size)
Page<User> page = Page.of(1, 10);
Page<User> page2 = new Page<>(1, 10);

// 不需要总数时关闭 count 查询，直接省掉一次 SQL
Page<User> page3 = Page.of(1, 10, false);   // 第三参即 searchCount
page.setSearchCount(false);

IPage<User> result = userMapper.selectPage(page, new LambdaQueryWrapper<User>().eq(User::getStatus, 1));
```

⚠️ `setSearchCount(false)` 后 `getTotal()` 恒为 0，前端分页器不要依赖它（用「是否还有下一页」判断）。瀑布流/滚动加载场景应默认关掉 count。

### 自定义 Mapper 方法分页

```java
IPage<UserVo> selectPageVo(IPage<?> page, Integer state);
// 或者自定义分页类
MyPage selectPageVo(MyPage page);
// 或者返回 List
List<UserVo> selectPageVo(IPage<UserVo> page, Integer state);
```

```xml
<select id="selectPageVo" resultType="xxx.xxx.xxx.UserVo">
    SELECT id,name FROM user WHERE state=#{state}
</select>
```

规则：

- **`IPage` 参数必须是第一个入参**，插件靠位置识别。
- 返回 `IPage` 时，入参 `IPage` **不能为 null**。想临时不分页，把 `size` 传小于 0 的值。
- 返回 `List` 时，入参 `IPage` 可为 null，但需要**手动** `page.setRecords(返回的 List)`。
- XML 中取 page 内字段用 `page.属性`（如 `#{page.current}`），不能直接 `#{current}`。

### countSql 的坑

⚠️ 生成 countSql 时，如果 `left join` 的表不参与 where 条件，会被 `optimizeJoinOfCountSql` 优化掉。若 join 会导致行数膨胀（一对多），优化后的 count 值与实际记录数不一致。解决办法二选一：

1. 给所有 join 的表和字段**都加别名**（官方建议：任何带 `left join` 的 SQL 都加别名）；
2. `page.setOptimizeJoinOfCountSql(false)` 关闭该优化，或用 `countId` 指定手写的 count statement。

⚠️ 排序：`OrderItem.column` **不要让前端直接传**。必须走白名单校验，否则是标准 SQL 注入入口（该字段是拼接进 SQL 的，不走占位符）。

---

## 3. 乐观锁 OptimisticLockerInnerInterceptor

```java
@Configuration
@MapperScan("按需修改")
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        return interceptor;
    }
}
```

```java
import com.baomidou.mybatisplus.annotation.Version;


public class YourEntity {
    @Version
    private Integer version;
    // 其他字段...
}
```

原理：读取时取 `version`，更新时 `SET version = newVersion WHERE ... AND version = oldVersion`，版本不匹配则影响行数为 0（**不抛异常**，必须自己判断返回值）。

### 硬性约束

- 支持类型：`int`、`Integer`、`long`、`Long`、`Date`、`Timestamp`、`LocalDateTime`。其它类型（如 `String`、`BigDecimal`）不生效。
- 整数类型 `newVersion = oldVersion + 1`；`newVersion` 会**自动回写到实体对象**（同一个实体连续 update 不用重新查库，但跨请求必须重查）。
- ⚠️ **只对 `updateById(entity)`、`update(entity, wrapper)`、`saveOrUpdate(entity)`、`insertOrUpdate(entity)`（3.5.7+）生效**。`update(null, wrapper)`、`LambdaUpdateWrapper.set(...)` 这类"无实体"更新**完全不走乐观锁**，是最常见的"乐观锁没生效"根因。
- ⚠️ **实体必须带上版本值**。从前端 DTO 转出的实体若 `version` 为 null，插件不会拼接条件，乐观锁静默失效。
- 自定义方法若参数形态与内置一致（如 `myUpdate(entity)` 等价于 `updateById(entity)`），也会提取参数执行乐观锁填充，但**更新语句本身要自己写**。
- ⚠️ `update(entity, wrapper)` 中 **wrapper 不能复用**。第二次使用时上一轮拼接的 `version = ?` 条件还在，条件会叠加导致更新不到数据。

---

## 4. 多租户 TenantLineInnerInterceptor

```java
public interface TenantLineHandler {

    /**
     * 获取租户 ID 值表达式，只支持单个 ID 值
     */
    Expression getTenantId();

    /**
     * 获取租户字段名，默认字段名叫: tenant_id
     */
    default String getTenantIdColumn() {
        return "tenant_id";
    }

    /**
     * 根据表名判断是否忽略拼接多租户条件
     * true:表示忽略，false:需要解析并拼接多租户条件
     */
    default boolean ignoreTable(String tableName) {
        return false;
    }

    /**
     * 忽略插入租户字段逻辑
     */
    default boolean ignoreInsert(List<Column> columns, String tenantIdColumn) {
        return columns.stream().map(Column::getColumnName).anyMatch(i -> i.equalsIgnoreCase(tenantIdColumn));
    }
}
```

```java
@Component
public class CustomTenantHandler implements TenantLineHandler {

    @Override
    public Expression getTenantId() {
        // 假设有一个租户上下文，能够从中获取当前用户的租户
        Long tenantId = TenantContextHolder.getCurrentTenantId();
        // 返回租户ID的表达式，LongValue 是 JSQLParser 中表示 bigint 类型的 class
        return new LongValue(tenantId);
    }

    @Override
    public String getTenantIdColumn() {
        return "tenant_id";
    }

    @Override
    public boolean ignoreTable(String tableName) {
        // 根据需要返回是否忽略该表
        return false;
    }
}
```

```java
@Configuration
@MapperScan("com.yourpackage.mapper")
public class MybatisPlusConfig {

    @Autowired
    private CustomTenantHandler customTenantHandler;

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        TenantLineInnerInterceptor tenantInterceptor = new TenantLineInnerInterceptor();
        tenantInterceptor.setTenantLineHandler(customTenantHandler);
        interceptor.addInnerInterceptor(tenantInterceptor);
        return interceptor;
    }
}
```

### 约束与坑

- ⚠️ **多租户 ≠ 权限过滤**。租户之间是完全隔离的，不要拿它当行级权限用（那是数据权限插件的活）。
- ⚠️ 启用后 **所有 method 的 SQL 都会被处理**。字典表、配置表、系统表这类全局共享表必须在 `ignoreTable` 里列出来，否则会被拼上 `tenant_id = ?` 查不到数据。`ignoreTable` 的 `tableName` 是解析出的原始表名，注意大小写与 schema 前缀。
- ⚠️ `getTenantId()` 返回的是 **JSQLParser 的 `Expression`**，不是 Long。数值用 `new LongValue(id)`，字符串用 `new StringValue(id)`。上下文取不到租户时返回 null 会导致 NPE 或拼出非法 SQL —— 要么抛业务异常，要么走 `ignoreTable`。
- ⚠️ 自定义 SQL 必须规范书写：**多表必须每个表都给别名**，`inner join` 要写成标准的 `inner join`（不能省略成逗号连表），否则插件无法正确定位表并拼条件。
- 插入时默认也会判断租户条件，**需配合自动填充（`MetaObjectHandler`）填充 `tenant_id` 字段**，否则租户字段不落库。
- 顺序上租户插件必须在分页插件**之前**，否则 count SQL 不带租户条件 → `total` 是全租户总数。

### 本地缓存 SQL 解析

分页、租户等 JSQLParser 插件可开启解析缓存提升性能：

```java
static {
    // 默认支持序列化 FstSerialCaffeineJsqlParseCache，JdkSerialCaffeineJsqlParseCache
    JsqlParserGlobal.setJsqlParseCache(new JdkSerialCaffeineJsqlParseCache(
      (cache) -> cache.maximumSize(1024)
      .expireAfterWrite(5, TimeUnit.SECONDS))
    );
}
```

### 解析线程池

3.5.6 起 JSQLParser(4.9) 支持线程池解析复用。默认固定线程池核心数 `(Runtime.getRuntime().availableProcessors() + 1) / 2`。自定义线程池需自行关闭：

```java
static {
  // 3.5.6 ~ 3.5.11 使用静态变量进行赋值
  JsqlParserGlobal.executorService = xxxxx;
  // 3.5.11 开始使用 setExecutorService 方法进行设置
  JsqlParserGlobal.setExecutorService(....);
}
```

需要在解析前加工 SQL 字符串：

```java
static {
        JsqlParserGlobal.setParserMultiFunc((sql)-> {
            System.out.println("解析SQL:" + sql);
            return CCJSqlParserUtil.parseStatements(sql, JsqlParserGlobal.getExecutorService(), null);
        });
        JsqlParserGlobal.setParserSingleFunc((sql)-> {
            System.out.println("解析SQL:" + sql);
            return CCJSqlParserUtil.parse(sql, JsqlParserGlobal.getExecutorService(), null);
        });
}
```

---

## 5. 数据权限 DataPermissionInterceptor

原理与租户插件一致：SQL 执行前拦截，按用户权限动态往 where 里拼片段。区别在于它是**按 mappedStatementId + 表名**决定拼什么，粒度更细，适合"只能看本部门数据"这类行级权限。

```java
public class CustomDataPermissionHandler extends MultiDataPermissionHandler {
    @Override
    public Expression getSqlSegment(Table table, Expression where, String mappedStatementId) {
        // 在此处编写自定义数据权限逻辑
        try {
            String sqlSegment = "..."; // 数据权限相关的 SQL 片段
            return CCJSqlParserUtil.parseCondExpression(sqlSegment);
        } catch (JSQLParserException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

```java
// 在 MyBatis 配置中
Interceptor dataPermissionInterceptor = new DataPermissionInterceptor(new CustomDataPermissionHandler());
mybatisConfiguration.addInterceptor(dataPermissionInterceptor);
```

按 statementId + 表名分派的完整写法：

```java
new DataPermissionInterceptor(new MultiDataPermissionHandler() {

    @Override
    public Expression getSqlSegment(final Table table, final Expression where, final String mappedStatementId) {
        try {
            String sqlSegment = sqlSegmentMap.get(mappedStatementId, table.getName());
            if (sqlSegment == null) {
                logger.info("{} {} AS {} : NOT FOUND", mappedStatementId, table.getName(), table.getAlias());
                return null;
            }
            Expression sqlSegmentExpression = CCJSqlParserUtil.parseCondExpression(sqlSegment);
            logger.info("{} {} AS {} : {}", mappedStatementId, table.getName(), table.getAlias(), sqlSegmentExpression.toString());
            return sqlSegmentExpression;
        } catch (JSQLParserException e) {
            e.printStackTrace();
        }
        return null;
    }
});
```

要点：

- 返回 `null` 表示**该表不加任何权限条件**（放行），不是"拒绝全部"。写反了就是越权漏洞。
- 片段用 `CCJSqlParserUtil.parseCondExpression(sqlSegment)` 解析，只能是**条件表达式**，不能带 `WHERE` 关键字。
- 拼接时务必用 `table.getAlias()`（若有别名）限定列，否则多表 join 下会报 ambiguous column。
- 权限值来自当前用户上下文 → 不要把 `sqlSegment` 用字符串拼接用户输入，会 SQL 注入。
- ⚠️ 配置在 `MybatisPlusInterceptor` 内时，同样要放在分页插件**之前**。
- 配合 `@DataPermission` 注解可在 Mapper 方法/类上声明权限维度，并用 `@DataPermission(ignore = true)` 跳过该方法的数据权限处理。

---

## 6. 动态表名 DynamicTableNameInnerInterceptor

运行时替换 SQL 中的表名，典型用于按时间/业务维度分表。

```java
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.DynamicTableNameInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        DynamicTableNameInnerInterceptor dynamicTableNameInnerInterceptor = new DynamicTableNameInnerInterceptor();
        dynamicTableNameInnerInterceptor.setTableNameHandler((sql, tableName) -> {
            // 获取参数方法
            Map<String, Object> paramMap = RequestDataHelper.getRequestData();
            paramMap.forEach((k, v) -> System.err.println(k + "----" + v));

            String year = "_2018";
            int random = new Random().nextInt(10);
            if (random % 2 == 1) {
                year = "_2019";
            }
            return tableName + year;
        });
        interceptor.addInnerInterceptor(dynamicTableNameInnerInterceptor);
        return interceptor;
    }
}
```

Mapper 侧无需改动，正常继承 `BaseMapper<User>` 即可，表名在运行时被替换。

### 坑

- ⚠️ **ThreadLocal 泄漏**：上例中 `RequestDataHelper` 这类传参容器几乎都是 `ThreadLocal`。Tomcat/Undertow 线程是复用的，**用完必须在 finally 或拦截器 `afterCompletion` 中 remove()**，否则下一个请求会读到上一个请求的分表后缀，查到别人的数据。异步线程（`@Async`、线程池、`CompletableFuture`）不继承 ThreadLocal，分表参数会丢失，`tableName` 后缀算错 → `Table doesn't exist`。异步场景需显式透传参数（`TransmittableThreadLocal` 或方法入参）。
- ⚠️ 表名替换是**字符串匹配**，容易误伤。官方建议把动态表名定义得复杂一些（如统一前缀 `mp_dt_`），避免 `user` 这种短名把 `user_role`、字段名、字符串字面量一并替换掉。
- ⚠️ 后缀参与拼 SQL，**绝不能直接来自前端**，必须白名单/正则校验，否则是表名注入。
- 替换后的表名要符合目标数据库命名规则（长度、大小写敏感性）。

---

## 7. 安全类插件

### BlockAttackInnerInterceptor —— 防全表更新/删除

拦截**没有 where 条件**的 `update` / `delete`，抛 `MybatisPlusException: Prohibition of table update operation`。

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
        return interceptor;
    }
}
```

```java
/**
 * SQL：UPDATE user  SET name=?,email=?;
 */
@Test
public void testFullUpdate() {
    User user = new User();
    user.setId(999L);
    user.setName("custom_name");
    user.setEmail("xxx@mail.com");
    // 由于没有指定更新条件，插件将抛出异常
    // com.baomidou.mybatisplus.core.exceptions.MybatisPlusException: Prohibition of table update operation
    Assertions.assertThrows(MybatisPlusException.class, () -> {
        userService.saveOrUpdate(user, null);
    });
}
```

带条件的更新正常放行：

```java
/**
 * SQL：UPDATE user  SET name=?, email=? WHERE id = ?;
 */
@Test
public void testPartialUpdate() {
    LambdaUpdateWrapper<User> wrapper = new LambdaUpdateWrapper<>();
    wrapper.eq(User::getId, 1);
    User user = new User();
    user.setId(10L);
    user.setName("custom_name");
    user.setEmail("xxx@mail.com");
    // 由于指定了更新条件，插件不会拦截此操作
    userService.saveOrUpdate(user, wrapper);
}
```

⚠️ 它只看"有没有 where"，不看 where 是否有意义。`where 1=1` 照样放行，防不住逻辑全表。另外合法的批量刷数脚本会被误杀，需用 `@InterceptorIgnore(blockAttack = "true")` 或 `InterceptorIgnoreHelper` 临时放行。

### IllegalSQLInnerInterceptor —— SQL 性能规范

检查项：全表更新/删除、强制走索引（防全表扫描）、`not` / `or` / 子查询的额外检查。

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 添加非法SQL拦截器
        interceptor.addInnerInterceptor(new IllegalSQLInnerInterceptor());
        return interceptor;
    }
}
```

```xml
<bean id="mybatisPlusInterceptor" class="com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor">
    <property name="interceptors">
        <list>
            <bean class="com.baomidou.mybatisplus.extension.plugins.inner.IllegalSQLInnerInterceptor"/>
        </list>
    </property>
</bean>
```

⚠️ 该插件规则**偏激进，不适用于所有企业环境**（会额外查询索引元数据，有性能开销；`or`/子查询正常业务也会被判违规）。官方明确建议按项目需求自行改造。推荐**只在 dev/test profile 启用**，生产关闭。

### DataChangeRecorderInnerInterceptor —— 数据变动记录 + 批量阈值

```java
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.DataChangeRecorderInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        DataChangeRecorderInnerInterceptor dataChangeRecorderInnerInterceptor = new DataChangeRecorderInnerInterceptor();
        // 配置安全阈值，限制批量更新或删除的记录数不超过 1000 条
        dataChangeRecorderInnerInterceptor.setBatchUpdateLimit(1000).openBatchUpdateLimitation();
        interceptor.addInnerInterceptor(dataChangeRecorderInnerInterceptor);
        return interceptor;
    }
}
```

批量更新/删除超过阈值直接抛异常。⚠️ 插件只负责**采集**变动数据，**日志落库逻辑要自己实现**（继承并重写处理方法）。开启后每次更新会额外查询变更前的数据，写密集场景有明显性能成本，按需启用。

---

## 8. 按方法跳过插件

### @InterceptorIgnore 注解

打在 Mapper 接口（类级）或 Mapper 方法上。每个属性对应一个插件，不设值默认 `false`（不忽略），设为 `"true"` 则忽略该插件。

```java
public interface UserMapper extends BaseMapper<User> {

    // 该方法跳过多租户条件拼接（跨租户统计场景）
    @InterceptorIgnore(tenantLine = "true")
    List<UserCountVo> countAllTenants();

    // 运维刷数：跳过全表更新拦截
    @InterceptorIgnore(blockAttack = "true")
    int resetAllFlag();

    // 同时跳过多个
    @InterceptorIgnore(tenantLine = "true", dataPermission = "true", illegalSql = "true")
    List<User> rawQuery();
}
```

常用属性：`tenantLine`、`dataPermission`、`blockAttack`、`illegalSql`、`dynamicTableName`、`others`。

⚠️ 属性类型是 **String** 不是 boolean，必须写 `"true"`。

### InterceptorIgnoreHelper（3.5.3+，更灵活）

注解是静态的，需要按运行时条件决定是否跳过时用这个。⚠️ **必须手动关闭**，否则 ThreadLocal 残留会让后续所有请求都跳过插件 —— 租户隔离直接失效，是严重数据泄漏。

```java
// 请尽量使用 try finally 的方式来保证能正确得到关闭
try {
    // 设置忽略租户插件
    InterceptorIgnoreHelper.handle(IgnoreStrategy.builder().tenantLine(true).build());
    // 执行逻辑 ..
} finally {
    // 关闭忽略策略
  InterceptorIgnoreHelper.clearIgnoreStrategy();
}
```

---

## 排查清单

插件"不生效"时按序检查：

1. 是否声明了**多个** `MybatisPlusInterceptor` Bean？只能有一个。
2. 3.5.9+ 是否引入 `mybatis-plus-jsqlparser`？
3. `@MapperScan` 路径是否覆盖到目标 Mapper？
4. 分页是否被放在了租户/数据权限**之前**（total 不对的典型症状）？
5. 乐观锁：用的是 `updateById(entity)` 还是 `update(null, wrapper)`？实体的 `version` 是不是 null？
6. 是否有残留的 `InterceptorIgnoreHelper` 策略未 clear？
7. 自定义 XML SQL 是否给所有表加了别名、join 是否写成标准形式？
