# MyBatis-Plus 3.5.17 核心 API 参考

## 1. 依赖引入

按 Spring Boot 大版本选 starter，artifactId **不通用**，选错会出现 starter 自动装配不生效（Mapper 注入失败）：

| 运行环境 | artifactId |
| --- | --- |
| Spring Boot 2 | `mybatis-plus-boot-starter` |
| Spring Boot 3 | `mybatis-plus-spring-boot3-starter` |
| Spring Boot 4（3.5.13+） | `mybatis-plus-spring-boot4-starter` |
| 纯 Spring（无 Boot） | `mybatis-plus` |
| Solon | `mybatis-plus-solon-plugin` |

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.17</version>
</dependency>
```

Spring Boot 2：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.17</version>
</dependency>
```

⚠️ **引入 MyBatis-Plus 之后不要再引入 `mybatis`、`mybatis-spring-boot-starter`、`mybatis-spring`**。MP 的 starter 已经传递了经过验证的 MyBatis 版本，重复引入会因版本差异产生 `NoSuchMethodError` / 映射注册异常这类难排查的问题。

⚠️ 自 3.5.4 起，**没有**使用上述三个 boot starter 时（例如纯 Spring + `mybatis-plus`），需要自行按项目情况引入 `mybatis-spring`。

⚠️ 自 3.5.9 起，**插件部分改为可选依赖**。jsqlparser 5.0+ 不再支持 JDK 8，MP 把 jsqlparser 解耦成独立模块，用到分页插件、动态表名、多租户等需要 SQL 解析的能力时必须显式引入：

- `mybatis-plus-jsqlparser`：跟随 jsqlparser 最新版更新，**JDK 11+**
- `mybatis-plus-jsqlparser-4.9`：锁定 jsqlparser 4.9，**JDK 8+**

推荐用 BOM 统一版本：

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

<!-- spring boot3 引入可选模块 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
</dependency>

<!-- jdk 11+ 引入可选模块 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-jsqlparser</artifactId>
</dependency>
```

JDK 8 项目对应换成 `mybatis-plus-boot-starter` + `mybatis-plus-jsqlparser-4.9`。

测试依赖（提供 `@MybatisPlusTest` 切片测试注解）：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter-test</artifactId>
    <version>3.5.17</version>
</dependency>
```

快照版仓库自 3.5.11-SNAPSHOT 起迁移到 `https://central.sonatype.com/repository/maven-snapshots/`，有效期 90 天；用阿里云镜像时需 `<mirrorOf>*,!snapshots</mirrorOf>`，否则拉不到快照。

## 2. 最小可用工程

三件套：启动类 `@MapperScan` + 实体 + Mapper 继承 `BaseMapper`，XML 一行都不用写。

```java
@SpringBootApplication
@MapperScan("com.baomidou.mybatisplus.samples.quickstart.mapper")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

```java
@Data
@TableName("`user`")
public class User {
    private Long id;
    private String name;
    private Integer age;
    private String email;
}
```

```java
public interface UserMapper extends BaseMapper<User> {

}
```

```java
@SpringBootTest
public class SampleTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    public void testSelect() {
        System.out.println(("----- selectAll method test ------"));
        List<User> userList = userMapper.selectList(null);
        Assert.isTrue(5 == userList.size(), "");
        userList.forEach(System.out::println);
    }

}
```

要点与坑：

- `selectList(null)` 的参数是 Wrapper，传 `null` = 无条件全表扫。生产代码禁止裸写，至少带 `Wrappers.lambdaQuery()` 明确意图。
- `@TableName("`user`")` 的反引号是因为 `user` 是 MySQL/H2 保留字。表名撞关键字必须手动转义，MP 不会替你加。
- 没有 `@MapperScan` 时必须给每个 Mapper 打 `@Mapper`，二选一，别两个都不写。
- ⚠️ MP 不推荐复合主键，约定每张表都有唯一 `id` 主键；`BaseMapper` 里所有 `Serializable id` 参数都基于这个假设。

切片测试写法：

```java
@MybatisPlusTest
class MybatisPlusSampleTest {

    @Autowired
    private SampleMapper sampleMapper;

    @Test
    void testInsert() {
        Sample sample = new Sample();
        sampleMapper.insert(sample);
        assertThat(sample.getId()).isNotNull();
    }
}
```

## 3. BaseMapper 方法表

物理数据访问层，返回 `int` 影响行数或实体，**不做任何业务判断**。

| 方法 | 说明 | 注意点 |
| --- | --- | --- |
| `int insert(T entity)` | 插入一条记录 | 策略插入，null 字段默认不进 SQL；主键回填到 entity 上，插入后直接读 `entity.getId()` |
| `int deleteById(Serializable id)` | 按主键删 | 配 `@TableLogic` 时实际执行 UPDATE 而非 DELETE |
| `int deleteByMap(Map<String,Object> columnMap)` | 按 columnMap 等值条件删 | key 是**数据库列名**不是属性名；map 为空会删全表 |
| `int delete(Wrapper<T> wrapper)` | 按条件构造器删 | wrapper 可为 null → **全表删除**，务必校验 |
| `int deleteBatchIds(Collection<? extends Serializable> idList)` | 按主键集合批删，生成 `IN (...)` | idList 不能为 null/empty；超大集合会撑爆 SQL 长度，自行分片 |
| `int updateById(T entity)` | 按主键更新 | 只更新非 null 字段；想把字段更新成 null 得用 `UpdateWrapper.set(col, null)` 或选装件 `alwaysUpdateSomeColumnById` |
| `int update(T updateEntity, Wrapper<T> whereWrapper)` | entity 提供 SET，wrapper 提供 WHERE | updateEntity 可为 null（此时 SET 全部来自 `UpdateWrapper.set`）；wrapper 内的 entity 也会参与 where 生成 |
| `T selectById(Serializable id)` | 按主键查 | 查不到返回 null |
| `List<T> selectBatchIds(Collection<? extends Serializable> idList)` | 按主键集合查 | 同 `deleteBatchIds`，不能为 null/empty |
| `List<T> selectByMap(Map<String,Object> columnMap)` | 按 columnMap 等值查 | key 为列名；只支持等值，复杂条件用 Wrapper |
| `T selectOne(Wrapper<T> queryWrapper)` | 查一条 | ⚠️ 结果多于一条会抛异常，不确定时加 `wrapper.last("LIMIT 1")` |
| `Integer selectCount(Wrapper<T> queryWrapper)` | 条件计数 | 3.4.3.2 之后统一为 `Long`，升级时注意拆箱/赋值编译错误 |
| `List<T> selectList(Wrapper<T> queryWrapper)` | 条件查全部 | 无上限，大表会 OOM，配合分页或流式 |
| `List<Map<String,Object>> selectMaps(Wrapper<T> queryWrapper)` | 结果映射为 Map | 适合 `select` 指定聚合列 / group by 场景 |
| `List<Object> selectObjs(Wrapper<T> queryWrapper)` | ⚠️ **只返回第一个字段的值** | 必须用 `select()` 明确限定列，否则拿到的是主键 |
| `IPage<T> selectPage(IPage<T> page, Wrapper<T> queryWrapper)` | 实体分页 | ⚠️ 必须注册分页插件（`PaginationInnerInterceptor`），否则分页参数被静默忽略、返回全表 |
| `IPage<Map<String,Object>> selectMapsPage(IPage<T> page, Wrapper<T> queryWrapper)` | Map 分页 | 同上；`page` 亦可传 `RowBounds.DEFAULT` |

3.5.4 新增的**流式重载**（第 7 节详述）：`selectList`、`selectByMap`、`selectBatchIds`、`selectMaps`、`selectObjs` 均新增接受 `ResultHandler<T>` 的重载，其中 `selectList` 还有 `(IPage, Wrapper, ResultHandler)` 三参版本。

### 选装件（需配置 SQL 注入器 + 在 Mapper 中显式声明）

位于 `com.baomidou.mybatisplus.extension.injector.methods`：

| 方法 | 用途 |
| --- | --- |
| `int alwaysUpdateSomeColumnById(T entity)` | 强制更新指定字段，绕过「null 不更新」策略，典型场景：更新时间戳、版本号 |
| `int insertBatchSomeColumn(List<T> entityList)` | 真·单条 SQL 多 VALUES 批量插入，只插部分字段 |
| `int logicDeleteByIdWithFill(T entity)` | 逻辑删除同时填充删除时间/删除人等字段 |

⚠️ `insertBatchSomeColumn` 只支持 MySQL 系（依赖多 VALUES 语法），且不会走主键回填。

## 4. IService / ServiceImpl 方法表

逻辑数据访问层，返回 `boolean` 语义化结果。方法命名刻意与 Mapper 区分：`get` 查单条、`list` 查集合、`page` 分页、`remove` 删除、`save` 插入。

⚠️ v3.5.9 之后 service 模块被重构抽象为 repository，**新项目建议用 `IRepository` / `CrudRepository`**，用法与 `IService` 完全一致，只是语义更清晰：

```java
public class UserRepository extends CrudRepository<UserMapper, User> implements IRepository<User> {
  // TODO
}
```

| 方法 | 说明 | 注意点 |
| --- | --- | --- |
| `boolean save(T entity)` | 策略插入一条 | 等价 `mapper.insert() > 0` |
| `boolean saveBatch(Collection<T> entityList)` | 批量插入 | 默认 batchSize 1000，见第 6 节 |
| `boolean saveBatch(Collection<T> entityList, int batchSize)` | 指定批次大小批量插入 | 按批次拆分成多条 SQL |
| `boolean saveOrUpdate(T entity)` | `@TableId` 有值则 UPDATE，否则 INSERT | ⚠️ 每条要先 SELECT 判存在，N 条 = N 次查询，热点路径别用 |
| `boolean saveOrUpdateBatch(Collection<T> entityList)` | 批量存在则更新否则插入 | 同上，混合场景性能最差；能确定行为时拆成 saveBatch / updateBatchById |
| `boolean saveOrUpdateBatch(Collection<T> entityList, int batchSize)` | 指定批次 | |
| `boolean remove(Wrapper<T> queryWrapper)` | 条件删除 | wrapper 为空条件 = 删全表 |
| `boolean removeById(Serializable id)` | 按主键删 | |
| `boolean removeByMap(Map<String,Object> columnMap)` | 按 columnMap 删 | key 为列名 |
| `boolean removeByIds(Collection<? extends Serializable> idList)` | 按主键集合删 | 生成 `IN (...)` |
| `boolean update(Wrapper<T> updateWrapper)` | 仅 wrapper 更新 | ⚠️ **必须在 wrapper 里 `.set(...)`**，否则无 SET 子句、SQL 报错 |
| `boolean update(T updateEntity, Wrapper<T> whereWrapper)` | entity 给 SET，wrapper 给 WHERE | |
| `boolean updateById(T entity)` | 按主键选择性更新 | null 字段不更新 |
| `boolean updateBatchById(Collection<T> entityList)` | 批量按主键更新 | 生成 N 条 UPDATE，不是一条 |
| `boolean updateBatchById(Collection<T> entityList, int batchSize)` | 指定批次 | |
| `T getById(Serializable id)` | 按主键查 | |
| `T getOne(Wrapper<T> queryWrapper)` | 查一条 | ⚠️ 多结果抛异常；要随机取一条须自行 `wrapper.last("LIMIT 1")` |
| `T getOne(Wrapper<T> queryWrapper, boolean throwEx)` | `throwEx=false` 时多结果不抛异常 | |
| `Map<String,Object> getMap(Wrapper<T> queryWrapper)` | 查一条并映射为 Map | |
| `<V> V getObj(Wrapper<T> queryWrapper, Function<? super Object,V> mapper)` | 查一条并转换 | |
| `List<T> list()` | 全表查 | 大表禁用 |
| `List<T> list(Wrapper<T> queryWrapper)` | 条件查 | |
| `Collection<T> listByIds(Collection<? extends Serializable> idList)` | 按主键集合查 | |
| `Collection<T> listByMap(Map<String,Object> columnMap)` | 按 columnMap 查 | |
| `List<Map<String,Object>> listMaps()` / `listMaps(Wrapper<T>)` | 结果映射为 Map | |
| `<V> List<V> listObjs(Wrapper<T> queryWrapper, Function<? super Object,V> mapper)` | ⚠️ **只返回第一个字段的值** | 必须配 `.select(...)` 限定列 |
| `IPage<T> page(IPage<T> page)` / `page(IPage<T>, Wrapper<T>)` | 实体分页 | 依赖分页插件 |
| `IPage<Map<String,Object>> pageMaps(IPage<T> page)` / `pageMaps(IPage<T>, Wrapper<T>)` | Map 分页 | |
| `long count()` / `long count(Wrapper<T> queryWrapper)` | 计数 | ⚠️ 自 3.4.3.2 返回值由 `int` 改为 `long`，老代码 `int total = service.count()` 直接编译失败 |

`listObjs` 的正确写法（必须 select 限定列，否则拿到主键）：

```java
// 设置查询条件为 age > 25，并将结果转换为 String 列表
List<String> userNames = userService.listObjs(Wrappers.<SysUser>lambdaQuery().select(SysUser::getName)
.gt(SysUser::getAge, 25), obj -> (String) obj);
```

⚠️ 如果有自定义通用 Service 方法的可能，**先建自己的 `IBaseService extends IService`**，后期不用改动上百个接口签名。

### 链式调用

`IService` 内置四个链式入口：

```java
// 链式查询 普通
QueryChainWrapper<T> query();
// 链式查询 lambda 式。注意：不支持 Kotlin
LambdaQueryChainWrapper<T> lambdaQuery();
// 链式更改 普通
UpdateChainWrapper<T> update();
// 链式更改 lambda 式。注意：不支持 Kotlin
LambdaUpdateChainWrapper<T> lambdaUpdate();
```

查询：

```java
// 普通链式查询示例
query().eq("name", "John").list(); // 查询 name 为 "John" 的所有记录

// lambda 链式查询示例
lambdaQuery().eq(User::getAge, 30).one(); // 查询年龄为 30 的单条记录
```

更新：

```java
// 普通链式更新示例
update().set("status", "inactive").eq("name", "John").update(); // 将 name 为 "John" 的记录 status 更新为 "inactive"

// lambda 链式更新示例
User updateUser = new User();
updateUser.setEmail("new.email@example.com");
lambdaUpdate().set(User::getEmail, updateUser.getEmail()).eq(User::getId, 1).update(); // 更新 ID 为 1 的用户的邮箱
```

⚠️ 链式调用**必须以终结方法收尾**（`list()` / `one()` / `count()` / `page()` / `update()` / `remove()`），只构建不调用等于什么都没执行，且不报错——这是最常见的「代码没生效」原因。

⚠️ `lambdaQuery()` / `lambdaUpdate()` **不支持 Kotlin**（方法引用无法反序列化出列名），Kotlin 项目用 `query()` / `update()` 字符串版或 KtQueryWrapper。

### Db Kit（静态门面）

用于规避 Service 循环注入：

```java
User user = Db.getById(1L, User.class);
List<User> userList = Db.listByIds(Arrays.asList(1L, 2L, 3L), User.class);

LambdaQueryWrapper<User> queryWrapper = Wrappers.lambdaQuery(User.class)
    .eq(User::getStatus, "active");
List<User> activeUsers = Db.list(queryWrapper);

boolean isBatchInserted = Db.saveBatch(batchUsers);
```

⚠️ 传 Wrapper 时**必须在 Wrapper 中带上 Entity 或 EntityClass**（`Wrappers.lambdaQuery(User.class)`），否则 MP 找不到对应 Mapper 直接抛异常。
⚠️ 不要在循环里调 Db Kit，先把数据构造好再 `Db.saveBatch(...)`。

## 5. Mapper vs Service 选型

- **Mapper（BaseMapper）**：物理数据访问层。返回影响行数 / 原始实体，无语义包装，单次调用 = 单条 SQL，行为可预测。⚠️ 任何分层方案下，**Mapper 层都不应写业务逻辑处理代码**。
- **Service（IService / IRepository）**：逻辑数据访问层。返回 `boolean`，提供 `saveBatch`、`saveOrUpdate`、链式 API 等组合能力，但一次调用可能隐含多条 SQL。

选型口径：

| 场景 | 选择 |
| --- | --- |
| 单条精确 CRUD、要影响行数做乐观判断 | Mapper |
| 需要知道「更新了几行」 | Mapper（Service 的 boolean 丢失了行数信息） |
| 批量插入 / 批量更新 | Service `saveBatch` / `updateBatchById` 或 `MybatisBatchUtils` |
| 链式条件构造、代码可读性优先 | Service `lambdaQuery()` / `lambdaUpdate()` |
| 复杂多表 SQL | 自定义 Mapper 方法 + XML/注解，别硬套 Wrapper |
| 避免 Service 相互注入 | `Db` 静态工具 |

完整分层：Controller → Service（业务逻辑） → Repository（逻辑数据访问） → Mapper（物理数据访问）。追求简洁可把 Service 与 Repository 合并为一层，但 Mapper 层的纯粹性不能破。

## 6. 批量操作

### saveBatch 的真实行为

`saveBatch` **不是**把 N 条数据拼成一条 `INSERT ... VALUES (..),(..)`，而是在同一个 SqlSession 里用 `ExecutorType.BATCH` 逐条 `addBatch`，按 `batchSize`（默认 1000）分批 `flushStatements`。

```java
List<User> users = Arrays.asList(
    new User("Alice", "alice@example.com"),
    new User("Bob", "bob@example.com"),
    new User("Charlie", "charlie@example.com")
);
boolean result = userService.saveBatch(users);
```

指定批次大小：

```java
boolean result = userService.saveBatch(users, 2); // 调用 saveBatch 方法，指定批次大小
```

⚠️ **MySQL 下必须在 JDBC URL 加 `rewriteBatchedStatements=true`**，否则驱动仍然一条条发往服务端，batch 只省了解析开销，性能提升极其有限（常见「用了 saveBatch 还是慢」的根因）：

```
jdbc:mysql://host:3306/db?rewriteBatchedStatements=true
```

⚠️ `updateBatchById` 生成的是 N 条独立 UPDATE（`UPDATE user SET email=? WHERE id=1`、`... WHERE id=2`），不会合并；`saveOrUpdateBatch` 每条还要先判存在，代价最高——能确定是纯新增就用 `saveBatch`。

⚠️ 批次大小不是越大越好：太大占用连接内存、单次 flush 阻塞时间长、失败回滚代价高。经验值 500~1000。

### MybatisBatch / MybatisBatchUtils（3.5.4+）

更底层、更可控的批量 API，支持自定义 Mapper 方法与非实体类型数据。

- 事务控制：**需手动管理（默认关闭）**
- 执行结果：`List<BatchResult>`，按「MappedStatement + SQL」分组。例如批量按 ID 更新，10 条里 5 条更新一个字段、5 条更新两个字段，返回的 List 容量为 2
- 数据写入取决于代码是否正确执行到 `flushStatements`
- 异常类型：`PersistenceException`
- 支持 Spring 与非 Spring 项目

类结构：`MybatisBatch<?>`（绑定 `sqlSessionFactory` + 非空 `dataList`）→ `MybatisBatch.Method<?>`（绑定具体 Mapper 类）→ execute。

实体类型数据：

```java
List<H2User> userList = Arrays.asList(new H2User(2000L, "测试"), new H2User(2001L, "测试"));
MybatisBatch<H2User> mybatisBatch = new MybatisBatch<>(sqlSessionFactory, userList);
MybatisBatch.Method<H2User> method = new MybatisBatch.Method<>(H2UserMapper.class);
mybatisBatch.execute(method.insert());
```

非实体类型数据（用 `parameterConvert` 转换）：

```java
List<Long> ids = Arrays.asList(120000L, 120001L);
MybatisBatch<Long> mybatisBatch = new MybatisBatch<>(sqlSessionFactory, ids);
MybatisBatch.Method<H2User> method = new MybatisBatch.Method<>(H2UserMapper.class);
mybatisBatch.execute(method.insert(id -> {
    H2User h2User = new H2User();
    h2User.setTestId(id);
    return h2User;
}));
```

自定义方法（无 `@Param`）：

```java
// mapper方法定义
@Insert("insert into h2user(name,version) values( #{name}, #{version})")
int myInsertWithoutParam(H2User user1);

MybatisBatch<H2User> mybatisBatch = new MybatisBatch<>(sqlSessionFactory, h2UserList);
MybatisBatch.Method<H2User> method = new MybatisBatch.Method<>(H2UserMapper.class);
mybatisBatch.execute(method.get("myInsertWithoutParam"));
```

自定义方法（带 `@Param`，必须手动拼 Map）：

```java
// 带注解的mapper方法定义
@Insert("insert into h2user(name,version) values( #{user1.name}, #{user1.version})")
int myInsertWithParam(@Param("user1") H2User user1);

mybatisBatch.execute(method.get("myInsertWithParam", (user) -> {
    Map<String, Object> map = new HashMap<>();
    map.put("user1", user);
    return map;
}));
```

saveOrUpdate（三参：insert 方法、判断条件、update 方法）：

```java
MybatisBatch.Method<H2User> mapperMethod = new MybatisBatch.Method<>(H2UserMapper.class);

new MybatisBatch<>(sqlSessionFactory, h2UserList).saveOrUpdate(
    mapperMethod.insert(), // 指定insert方法
    ((sqlSession, h2User) -> userMapper.selectById(h2User.getTestId()) == null), // 判断条件
    mapperMethod.updateById()); // 指定update方法
```

⚠️ 上例中判断条件走的是**另一个 sqlSession**（注入的 `userMapper`），批量写入还没 flush 时查不到本批数据，会导致重复插入。共用 sqlSession 的写法才能感知本批未提交数据：

```java
new MybatisBatch<>(sqlSessionFactory, h2UserList).saveOrUpdate(
    mapperMethod.insert(), // 指定insert方法
    ((sqlSession, h2User) -> sqlSession.selectList(mapperMethod.get("selectById").getStatementId(), h2User.getTestId()).isEmpty()), // 判断条件
    mapperMethod.updateById()); // 指定update方法
```

事务需自己套（默认不开）：

```java
@Autowired
private TransactionTemplate transactionTemplate;

transactionTemplate.execute((TransactionCallback<List<BatchResult>>) status -> {
    MybatisBatch.Method<H2User> mapperMethod = new MybatisBatch.Method<>(H2UserMapper.class);
    // 执行批量插入
    MybatisBatchUtils.execute(sqlSessionFactory, h2UserList, mapperMethod.insert());
    throw new RuntimeException("出错了");
});
```

超大数据量导入（百万级）优先考虑数据库原生能力，MySQL：

```sql
LOAD DATA INFILE '/path/to/your/file.csv'
INTO TABLE your_table_name
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

## 7. 流式查询（3.5.4+）

本质是 MyBatis 原生 `ResultHandler` 能力，逐条回调而非一次性装进 List，用于跑批 / 大数据处理，避免 `selectList` 全量加载 OOM。

`ResultContext` 三个常用方法：

| 方法 | 说明 |
| --- | --- |
| `getResultObject()` | 获取数据库中的每一条记录 |
| `getResultCount()` | 当前已处理的结果集条数，每处理一条 +1，**从 1 开始计数** |
| `stop()` | 停止继续处理结果集，相当于循环里的 `break` |

结合分页按批拉取：

```java
// 结合分页，按批次从数据库拉取数据出来跑批，例如从数据库获取10万记录，做数据处理
Page<H2User> page = new Page<>(1, 100000);
baseMapper.selectList(page, Wrappers.emptyWrapper(), new ResultHandler<H2User>() {
    int count = 0;
    @Override
    public void handleResult(ResultContext<? extends H2User> resultContext) {
        H2User h2User = resultContext.getResultObject();
        System.out.println("当前处理第" + (++count) + "条记录: " + h2User);
        // 在这里进行你的业务处理，比如分发任务
    }
});
```

全表流式处理：

```java
// 从数据库获取表所有记录，做数据处理
baseMapper.selectList(Wrappers.emptyWrapper(), new ResultHandler<H2User>() {
    int count = 0;
    @Override
    public void handleResult(ResultContext<? extends H2User> resultContext) {
        H2User h2User = resultContext.getResultObject();
        System.out.println("当前处理第" + (++count) + "条记录: " + h2User);
        // 在这里进行你的业务处理，比如分发任务
    }
});
```

约束与坑：

- ⚠️ **必须在事务中执行**。流式查询依赖游标持有数据库连接，脱离事务时连接可能被提前归还连接池，导致 `ResultHandler` 处理到一半连接失效。方法上加 `@Transactional`（只读即可）。
- ⚠️ **MySQL 需要开启真正的流式拉取**：`useCursorFetch=true` + `setFetchSize`，或 `setFetchSize(Integer.MIN_VALUE)`。否则驱动仍会把整个结果集读进客户端内存，ResultHandler 只是「换了个遍历方式」，OOM 照旧。
- ⚠️ **低版本 MP 中自定义 `ResultHandler` 结合分页查询可能出错，需要手动关闭 count 查询**（`page.setSearchCount(false)`）。
- ⚠️ 流式回调过程中**不要用同一连接做其他查询/写入**，容易死锁或游标失效。要写回结果，收集到一定量再用独立事务批量写。
- ⚠️ 回调内抛异常会中断整个结果集处理并回滚事务；需要「跳过坏数据」时自行 try-catch，需要提前终止时用 `resultContext.stop()` 而非抛异常。
- 支持流式重载的方法：`selectList`（含 `IPage` 三参重载）、`selectByMap`、`selectBatchIds`、`selectMaps`、`selectObjs`。

### SimpleQuery：非流式的结果集后处理

不解决内存问题（底层仍是 `selectList`），只是把结果直接收成 Map/List：

```java
Map<A, E> keyMap(LambdaQueryWrapper<E> wrapper, SFunction<E, A> sFunction, Consumer<E>... peeks);
Map<A, P> map(LambdaQueryWrapper<E> wrapper, SFunction<E, A> keyFunc, SFunction<E, P> valueFunc, Consumer<E>... peeks);
Map<K, List<T>> group(LambdaQueryWrapper<T> wrapper, SFunction<T, K> sFunction, Consumer<T>... peeks);
List<A> list(LambdaQueryWrapper<E> wrapper, SFunction<E, A> sFunction, Consumer<E>... peeks);
```

每个方法都有多一个 `boolean isParallel` 的重载用于并行流。⚠️ 使用前需确保项目中已注入对应实体的 `BaseMapper`；⚠️ `peeks` 是副作用回调，并行流下必须线程安全（示例里往 `ArrayList` 里 add 在 `isParallel=true` 时会丢数据）。
