# MyBatis-Plus 条件构造器 Wrapper（3.5.17）

## 1. Wrapper 家族与选型

| 类 | 作用 | 字段引用方式 | 何时用 |
|---|---|---|---|
| `AbstractWrapper` | 抽象基类，定义 column / value / condition 的构造逻辑，所有 Wrapper 的父类 | - | 仅作方法参数类型 `Wrapper<T>` |
| `QueryWrapper` | 查询条件 + `select()` 指定列 | 字符串数据库字段名 | 需要动态列名、聚合函数别名时 |
| `UpdateWrapper` | 更新条件 + `set()` / `setSql()` | 字符串数据库字段名 | 需要 `setSql` 表达式更新 |
| `LambdaQueryWrapper` | 查询条件，方法引用取字段 | `User::getName` | **默认首选** |
| `LambdaUpdateWrapper` | 更新条件 + set，方法引用取字段 | `User::getName` | **默认首选**；`setIncrBy/setDecrBy` 仅此处有 |
| `QueryChainWrapper` / `LambdaQueryChainWrapper` | 链式调用，末尾直接出结果（`.list()` / `.one()` / `.page()`） | 同上 | Service 层一行搞定查询 |
| `UpdateChainWrapper` / `LambdaUpdateChainWrapper` | 链式更新/删除（`.update()` / `.remove()`） | 同上 | Service 层一行搞定更新 |
| `KtQueryWrapper` / `KtUpdateWrapper` | Kotlin 专用 Lambda 风格 | `User::name` | Kotlin 项目 |

**为什么默认选 Lambda 版**：字符串字段名是硬编码，字段改名后编译期无感知，运行期才炸；而且 `R` 为 `String` 时表示的是**数据库字段名而非实体属性名**，写错方向是高频事故。Lambda 版由方法引用解析出列名，重构安全、IDE 可追踪。

```java
// 静态工厂 Wrappers，省去 new 的泛型噪音
QueryWrapper<User> queryWrapper = Wrappers.query();
LambdaQueryWrapper<User> lambdaQueryWrapper = Wrappers.lambdaQuery();
UpdateWrapper<User> updateWrapper = Wrappers.update();
LambdaUpdateWrapper<User> lambdaUpdateWrapper = Wrappers.lambdaUpdate();
```

`QueryWrapper` / `UpdateWrapper` 可以通过 `lambda()` 转成 Lambda 版：

```java
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
LambdaQueryWrapper<User> lambdaQueryWrapper = queryWrapper.lambda();
lambdaQueryWrapper.eq(User::getName, "张三");
```

⚠️ **Wrapper 实例不是线程安全的**，每次使用都新建实例，不要做成 Spring 单例字段或静态常量。

⚠️ **禁止在 RPC 中传输 Wrapper**：对象体积大、序列化不可控。正确做法是定义 DTO 传输，在被调用方根据 DTO 组装 Wrapper。官方明确「不接受任何关于 RPC 传输 Wrapper 报错相关的 issue 或 pr」。

⚠️ Wrapper 里 entity 属性也会参与生成 where 条件，但「entity 生成的 where 条件与使用各个 api 生成的 where 条件没有任何关联行为」——两者是拼接关系，不会互相覆盖或去重。

---

## 2. condition 参数机制（最实用的特性）

几乎所有条件方法都有一个首位 `boolean condition` 的重载：

```java
eq(R column, Object val)                    // 等价于 condition = true
eq(boolean condition, R column, Object val) // condition 为 false 时该条件整个不进 SQL
```

语义：`condition == false` 时，这一段条件**完全不拼接**到最终 SQL，等同于这行代码不存在。这是 MP 处理动态查询的标准姿势，替代 `if (x != null) { wrapper.eq(...) }` 的面条式写法。

### 动态查询标准写法

```java
queryWrapper.like(StringUtils.isNotBlank(name), Entity::getName, name)
            .eq(age != null && age >= 0, Entity::getAge, age);
```

不带 condition 参数时默认为 `true`，条件总是加入 SQL。

### 集合类参数的隐式短路

⚠️ 如果方法参数是 `Map` 或 `List`，当它们为空时，对应的 SQL 条件**不会**被加入最终 SQL。即 `in(User::getId, emptyList)` 不会生成 `IN ()` 语法错误，但也不会生成「查不到任何数据」的语义——这一点极易误判：业务上想表达「空集合 = 无结果」时，必须自己前置判空并短路返回，不能指望 Wrapper。

### 对比三种动态写法

```java
// 推荐：condition 参数，链式不中断
lambdaQuery().eq(status != null, User::getStatus, status).list();

// 可用：func 保持链式，内部写 if/else
queryWrapper.func(i -> {
    if (true) {
        i.eq("id", 1);
    } else {
        i.ne("id", 1);
    }
});

// 不推荐：链式被打断，可读性差
if (status != null) { wrapper.eq(User::getStatus, status); }
```

---

## 3. 完整方法表

### 3.1 等值 / 比较

| 方法 | 说明 | 示例 → 生成 SQL |
|---|---|---|
| `allEq(Map params)` | Map 批量等值，key 为**数据库字段名** | `allEq(Map.of("id",1,"name","老王","age",null))` → `WHERE id = 1 AND name = '老王' AND age IS NULL` |
| `allEq(Map, boolean null2IsNull)` | `null2IsNull=false` 时忽略 value 为 null 的键值对；`true` 时转 `IS NULL` | — |
| `allEq(BiPredicate filter, Map)` | filter 过滤哪些字段参与 | `allEq((f,v) -> f.contains("a"), Map.of("id",1,"name","老王","age",null))` → `WHERE name = '老王' AND age IS NULL` |
| `eq` | 等于 `=` | `eq(User::getName, "老王")` → `WHERE name = '老王'` |
| `eqOrIsNull` (3.5.17) | 值为 null 时自动转 `IS NULL` | `eqOrIsNull(Product::getName, null)` → `WHERE name is null` |
| `ne` | 不等于 `<>` | `ne(User::getName, "老王")` → `WHERE name <> '老王'` |
| `gt` | 大于 `>` | `gt(User::getAge, 18)` → `WHERE age > 18` |
| `ge` | 大于等于 `>=` | `ge(User::getAge, 18)` → `WHERE age >= 18` |
| `lt` | 小于 `<` | `lt(User::getAge, 18)` → `WHERE age < 18` |
| `le` | 小于等于 `<=` | `le(User::getAge, 18)` → `WHERE age <= 18` |
| `between` | `BETWEEN v1 AND v2` | `between(User::getAge, 18, 30)` → `WHERE age BETWEEN 18 AND 30` |
| `notBetween` | `NOT BETWEEN` | `notBetween(User::getAge, 18, 30)` → `WHERE age NOT BETWEEN 18 AND 30` |

⚠️ `gt/ge/lt/le/between` 面向数值型字段；用于字符串字段会走字符序比较，结果通常不符预期。务必保证 `val` 类型与列类型匹配，避免隐式类型转换（既可能出错，也会让索引失效）。

### 3.2 模糊匹配

| 方法 | 说明 | 示例 → 生成 SQL |
|---|---|---|
| `like` | 全模糊 | `like(User::getName, "王")` → `WHERE name LIKE '%王%'` |
| `notLike` | 全模糊排除 | `notLike(User::getName, "王")` → `WHERE name NOT LIKE '%王%'` |
| `likeLeft` | 左侧加 `%`（匹配以此结尾） | `likeLeft(User::getName, "王")` → `WHERE name LIKE '%王'` |
| `likeRight` | 右侧加 `%`（匹配以此开头，可走索引） | `likeRight(User::getName, "王")` → `WHERE name LIKE '王%'` |
| `notLikeLeft` | `NOT LIKE '%值'` | `notLikeLeft(User::getName, "王")` → `WHERE name NOT LIKE '%王'` |
| `notLikeRight` | `NOT LIKE '值%'` | `notLikeRight(User::getName, "王")` → `WHERE name NOT LIKE '王%'` |

⚠️ `likeLeft` / `likeRight` 的命名反直觉：**Left/Right 指的是 `%` 加在哪一侧**，不是匹配方向。`likeLeft` 生成 `'%王'`，`likeRight` 生成 `'王%'`。想要走索引的前缀匹配用 `likeRight`。

### 3.3 NULL 与集合

| 方法 | 说明 | 示例 → 生成 SQL |
|---|---|---|
| `isNull` | `IS NULL` | `isNull(User::getName)` → `WHERE name IS NULL` |
| `isNotNull` | `IS NOT NULL` | `isNotNull(User::getName)` → `WHERE name IS NOT NULL` |
| `in(column, Collection)` / `in(column, Object...)` | `IN` | `in(User::getAge, Arrays.asList(1,2,3))` → `WHERE age IN (1, 2, 3)` |
| `notIn(column, Collection)` / `notIn(column, Object...)` | `NOT IN` | `notIn(User::getAge, Arrays.asList(1,2,3))` → `WHERE age NOT IN (1, 2, 3)` |
| `inSql(column, String)` | `IN` 后跟原始 SQL 片段 | `inSql("id", "select id from other_table where id < 3")` → `WHERE id IN (select id from other_table where id < 3)` |
| `notInSql(column, String)` | `NOT IN` 后跟原始 SQL 片段 | `notInSql("age", "1,2,3,4,5,6")` → `WHERE age NOT IN (1, 2, 3, 4, 5, 6)` |

### 3.4 字段 vs 子查询结果比较（SQL 系列）

| 方法 | Since | 示例 → 生成 SQL |
|---|---|---|
| `eqSql(column, sql)` | 3.5.6 | `eqSql("id", "select MAX(id) from table")` → `WHERE id = (select MAX(id) from table)` |
| `gtSql(column, sql)` | 3.4.3.2 | `gtSql("id", "select id from table where name = 'xx'")` → `WHERE id > (...)` |
| `geSql(column, sql)` | 3.4.3.2 | → `WHERE id >= (...)` |
| `ltSql(column, sql)` | 3.4.3.2 | → `WHERE id < (...)` |
| `leSql(column, sql)` | 3.4.3.2 | → `WHERE id <= (...)` |

⚠️ 整个 `*Sql` 系列的字符串参数会**直接嵌入**生成的 SQL，不做参数绑定。「应避免 SQL 由前端动态参数传入并直接引用」。

### 3.5 分组、排序、HAVING

| 方法 | 说明 | 示例 → 生成 SQL |
|---|---|---|
| `groupBy(R... columns)` | 分组 | `groupBy(User::getId, User::getName)` → `GROUP BY id, name` |
| `orderByAsc(R... columns)` | 升序 | `orderByAsc(User::getId, User::getName)` → `ORDER BY id ASC, name ASC` |
| `orderByDesc(R... columns)` | 降序 | `orderByDesc(User::getId, User::getName)` → `ORDER BY id DESC, name DESC` |
| `orderBy(boolean condition, boolean isAsc, R... columns)` | 指定方向 | `orderBy(true, true, User::getId, User::getName)` → `ORDER BY id ASC, name ASC` |
| `having(String sqlHaving, Object... params)` | HAVING 子句，`{0}` 占位 | `groupBy(User::getAge).having("sum(age) > {0}", 10)` → `GROUP BY age HAVING sum(age) > 10` |

⚠️ `orderBy` 唯一的签名是三参形式 `orderBy(condition, isAsc, columns)`，没有 `orderBy(columns)` 重载，第一个 boolean 必须显式传。

### 3.6 逻辑与嵌套

| 方法 | 说明 |
|---|---|
| `or()` / `or(boolean condition)` | 将**紧接着的下一个**条件改为 OR 连接 |
| `or(Consumer<Param> consumer)` | OR 连接一个括号嵌套块 |
| `and(Consumer<Param> consumer)` | AND 连接一个括号嵌套块（无无参 `and()`） |
| `nested(Consumer<Param> consumer)` | 独立括号块，**不带默认 AND/OR**，由外层决定连接符 |
| `not(Consumer<Param> consumer)` | NOT 取反嵌套块 |
| `func(Consumer<Children> consumer)` | 在链式中插入任意构建逻辑，不中断链式 |

### 3.7 原始 SQL 拼接

| 方法 | 说明 | 示例 → 生成 SQL |
|---|---|---|
| `apply(String applySql, Object... params)` | 拼接任意 WHERE 片段，`{index}` 占位 | `apply("date_format(dateColumn, '%Y-%m-%d') = {0}", "2008-08-08")` → `WHERE date_format(dateColumn, '%Y-%m-%d') = '2008-08-08'` |
| `last(String lastSql)` | 拼到 SQL 最末尾 | `last("limit 1")` → `SELECT * FROM user LIMIT 1` |
| `exists(String existsSql)` | `EXISTS` 子查询 | `exists("select id from table where age = 1")` → `WHERE EXISTS (select id from table where age = 1)` |
| `notExists(String notExistsSql)` | `NOT EXISTS` 子查询 | `notExists("select id from table where age = 1")` → `WHERE NOT EXISTS (...)` |

### 3.8 列选择（仅 QueryWrapper / LambdaQueryWrapper）

| 方法 | 说明 | 示例 → 生成 SQL |
|---|---|---|
| `select(String... sqlSelect)` | 指定查询列 | `select("id", "name", "age")` → `SELECT id, name, age FROM user` |
| `select(SFunction... columns)` | Lambda 版指定列 | `select(User::getId, User::getName, User::getAge)` → 同上 |
| `select(Predicate<TableFieldInfo>)` | 过滤字段（主键除外） | `select(i -> i.getProperty().startsWith("test"))` → `SELECT testField1, testField2 FROM user` |
| `select(Class<T> entityClass, Predicate<TableFieldInfo>)` | 同上但显式给实体类 | — |

⚠️ `select` 的两类方法**重复调用时以最后一次为准**，不会累加。不带 `entityClass` 的 Predicate 版本要求 wrapper 内的 entity 属性有值，否则无字段元信息可过滤。

### 3.9 更新专用（UpdateWrapper / LambdaUpdateWrapper）

| 方法 | 使用范围 | 说明 | 示例 → 生成 SQL |
|---|---|---|---|
| `set(R column, Object val)` | 两者 | 设置 SET 字段 | `set(User::getName, "老李头")` → `UPDATE user SET name = '老李头'` |
| `set(R column, Object val, String mapping)` | 两者 | 带 typeHandler/jdbcType 映射 | `mapping = "javaType=int,jdbcType=NUMERIC,typeHandler=xxx.MyTypeHandler"` |
| `set(boolean condition, ...)` | 两者 | 条件式 SET | `set(true, "name", "")` → `UPDATE user SET name = ''` |
| `setSql(String setSql, Object... params)` | 两者 | 原始 SET 片段 | `setSql("dateColumn={0}", LocalDate.now())` |
| `setIncrBy(SFunction column, Number val)` | LambdaUpdateWrapper (3.5.6) | 字段自增 | `setIncrBy(Product::getNum, 1)` → `UPDATE product SET num = num + 1` |
| `setDecrBy(SFunction column, Number val)` | LambdaUpdateWrapper (3.5.6) | 字段自减 | `setDecrBy(Product::getNum, 1)` → `UPDATE product SET num = num - 1` |

---

## 4. and / or 嵌套与优先级（高频 bug 来源）

### 默认连接是 AND

不调用 `or()` 时，所有条件之间用 AND 连接。

### `or()` 只影响紧接着的下一个条件

```java
LambdaQueryWrapper<User> lambdaQueryWrapper = new LambdaQueryWrapper<>();
lambdaQueryWrapper.eq(User::getId, 1).or().eq(User::getName, "老王");
```

```sql
SELECT * FROM user WHERE id = 1 OR name = '老王'
```

「主动调用 or 方法表示紧接着下一个查询条件方法不是用 and 连接，而是用 or 连接。」

⚠️ **这是最经典的坑**：`.eq(A).or().eq(B).eq(C)` 生成的是 `A OR B AND C`，由于 SQL 中 AND 优先级高于 OR，实际语义为 `A OR (B AND C)`——`or()` 的作用域只覆盖它后面的**一个**条件，不会把后续所有条件都变成 OR，也不会自动加括号。想表达 `(A OR B) AND C` 必须显式嵌套。

### 用 Consumer 嵌套产生括号

```java
// 错误期望：以为能得到 (name = '李白' OR name = '杜甫') AND status = 'alive'
wrapper.eq(User::getName, "李白").or().eq(User::getName, "杜甫").eq(User::getStatus, "alive");
// 实际:  WHERE name = '李白' OR name = '杜甫' AND status = 'alive'
//       == WHERE name = '李白' OR (name = '杜甫' AND status = 'alive')

// 正确：把 OR 组包进 and(Consumer)
wrapper.and(i -> i.eq(User::getName, "李白").or().eq(User::getName, "杜甫"))
       .eq(User::getStatus, "alive");
// WHERE (name = '李白' OR name = '杜甫') AND status = 'alive'
```

### or 嵌套（官方示例）

```java
LambdaQueryWrapper<User> lambdaQueryWrapper = new LambdaQueryWrapper<>();
lambdaQueryWrapper.or(i -> i.and(j -> j.eq(User::getName, "李白").eq(User::getStatus, "alive"))
                              .or(j -> j.eq(User::getName, "杜甫").eq(User::getStatus, "alive")));
```

```sql
SELECT * FROM user WHERE (name = '李白' AND status = 'alive') OR (name = '杜甫' AND status = 'alive')
```

### and 嵌套（官方示例）

```java
LambdaQueryWrapper<User> lambdaQueryWrapper = new LambdaQueryWrapper<>();
lambdaQueryWrapper.and(i -> i.and(j -> j.eq(User::getName, "李白").eq(User::getStatus, "alive"))
                              .and(j -> j.eq(User::getName, "杜甫").eq(User::getStatus, "alive")));
```

```sql
SELECT * FROM user WHERE ((name = '李白' AND status = 'alive') AND (name = '杜甫' AND status = 'alive'))
```

### nested：不带连接符的独立块

```java
LambdaQueryWrapper<User> lambdaQueryWrapper = new LambdaQueryWrapper<>();
lambdaQueryWrapper.nested(i -> i.eq(User::getName, "李白").ne(User::getStatus, "活着"));
```

```sql
SELECT * FROM user WHERE (name = '李白' AND status <> '活着')
```

`and(Consumer)` 与 `nested(Consumer)` 的区别：`and` 强制以 AND 连接前面的条件，`nested` 只产生一个独立括号块，连接符由上下文决定（通常作为 `or()` 之后的那一个条件使用）。

### 逻辑组合速查

| 目标 SQL | 写法 |
|---|---|
| `A AND B` | `.eq(A).eq(B)` |
| `A OR B` | `.eq(A).or().eq(B)` |
| `A AND (B OR C)` | `.eq(A).and(i -> i.eq(B).or().eq(C))` |
| `(A AND B) OR (C AND D)` | `.and(i -> i.eq(A).eq(B)).or(i -> i.eq(C).eq(D))` |
| `A AND (B AND C)` | `.eq(A).and(i -> i.eq(B).eq(C))` |
| `A OR (B AND C)` | `.eq(A).or(i -> i.eq(B).eq(C))` |

---

## 5. UpdateWrapper / LambdaUpdateWrapper 的 set 用法

### 只用 Wrapper 更新，不传实体

「在更新操作中使用 UpdateWrapper 或 LambdaUpdateWrapper 时，可以省略实体对象，直接在 Wrapper 中设置更新字段。」

```java
LambdaUpdateWrapper<User> uw = Wrappers.<User>lambdaUpdate()
        .set(User::getName, "老李头")
        .set(User::getStatus, 0)
        .eq(User::getId, 1);
userMapper.update(null, uw);   // 第一个实体参数传 null
```

这是把某个字段更新为 `null` 的**唯一正确姿势**——`updateById(entity)` 默认按 `NOT NULL` 策略更新，实体上的 null 字段会被跳过，而 `set(col, null)` 会实打实生成 `SET col = null`。

- `val` 为空字符串 → 数据库字段变为空字符串
- `val` 为 `null` → 数据库字段变为 `null`

### 条件式 SET

```java
UpdateWrapper<User> updateWrapper = new UpdateWrapper<>();
updateWrapper.set(true, "name", "");
```

```sql
UPDATE user SET name = ''
```

### 自增 / 自减（避免读改写竞态）

```java
LambdaUpdateWrapper<Product> lambdaUpdateWrapper = new LambdaUpdateWrapper<>();
lambdaUpdateWrapper.setIncrBy(Product::getNum, 1);
```

```sql
UPDATE product SET num = num + 1
```

⚠️ `setIncrBy` / `setDecrBy` 的「使用范围」只写了 `LambdaUpdateWrapper`；库存扣减等场景务必配合 `gt(Product::getNum, 0)` 之类的条件，靠数据库做原子性，不要先查后改。

### setSql 与 TypeHandler

```java
setSql("name = '老李头'")
setSql("dateColumn={0}", LocalDate.now())
setSql("type={0,javaType=int,jdbcType=NUMERIC,typeHandler=xxx.xxx.MyTypeHandler}", "待处理字符串");
```

在 wrapper 中使用 typeHandler 需要特殊处理，利用 `formatSqlMaybeWithParam` 方法：

```java
// 查询
queryWrapper.apply("type={0,typeHandler="+ MyTypeHandler.class.getCanonicalName()+ "}", "待处理字符串");

// 更新
updateWrapper.setSql("type={0,javaType=string,jdbcType=VARCHAR,typeHandler=xxx.xxx.MyTypeHandler}", "待处理字符串");
```

---

## 6. SQL 注入安全

有两类方法：一类做参数绑定（安全），一类原样拼接（危险）。

| 类别 | 方法 | 安全性 |
|---|---|---|
| 参数绑定 | `eq/ne/gt/ge/lt/le/like/between/in/notIn/set` 等 | 安全，值走 `?` 占位 |
| 占位符拼接 | `apply(sql, params)`、`having(sql, params)`、`setSql(sql, params)` | **仅当值通过 `{index}` 传入时安全** |
| 纯拼接 | `last`、`inSql`、`notInSql`、`exists`、`notExists`、`eqSql/gtSql/geSql/ltSql/leSql` | **危险**，字符串原样进 SQL |

### apply 的正确 / 错误写法

```java
// ✅ 正确：动态值走 {0} 占位，由 MyBatis 参数绑定
queryWrapper.apply("date_format(dateColumn, '%Y-%m-%d') = {0}", "2008-08-08");

// ❌ 错误：字符串拼接，前端可注入
queryWrapper.apply("date_format(dateColumn, '%Y-%m-%d') = '" + userInput + "'");
```

源文档原文：「动态入参的 params 对应 applySql 内部的 {index} 部分，这样是不会有 SQL 注入风险的。如果直接将参数拼接到 SQL 中，则会有 SQL 注入风险，故应避免 SQL 由前端动态参数传入并直接引用。」

### last

```java
queryWrapper.last("limit 1");
```

```sql
SELECT * FROM user LIMIT 1
```

原文警告：

- 「last 方法只能调用一次，多次调用将以最后一次为准。」
- 「使用 last 方法会绕过 MyBatis-Plus 的查询优化规则，可能会导致查询效率降低。」
- 「因为 lastSql 参数直接拼接到 SQL 中，所以必须确保输入的 SQL 片段是安全的，即保障 lastSql 应该是后端自行控制，而不是动态参数由前端传入。」

⚠️ `last("limit 1")` 与分页插件冲突：分页插件也要往末尾加 LIMIT，两者叠加会产生非法 SQL 或错误结果。单条查询用 `Page(1,1)` 或 `selectOne`，不要用 `last`。

### inSql / notInSql / exists / notExists / *Sql

「sqlValue 参数应该是一个有效的 SQL 语句，它将直接嵌入到生成的 SQL 中，因此需要确保其安全性和正确性，应避免 SQL 由前端动态参数传入并直接引用。」

### 全局原则

「在使用 QueryWrapper UpdateWrapper 查询 设置 条件字段部分 一定要保证安全，避免 SQL 注入攻击。任何前端传入的 SQL 片段都要严格过滤。」

⚠️ 尤其注意：`QueryWrapper` 的 column 参数、`orderByAsc/Desc` 的列名、`select()` 的列名，都是字符串直拼。**排序字段来自前端时必须用白名单校验**，这是 MP 项目最常见的注入口子——它不在 value 侧而在 column 侧，参数绑定救不了。

---

## 7. Lambda 引用失效的坑

Lambda 版靠解析 `SFunction` 的序列化信息反推属性名，再映射到列名。以下情况会失效：

1. **不是标准 getter 方法引用**。必须是 `User::getName` 形式的实体 getter 方法引用，不能传普通 Lambda 表达式（`u -> u.getName()`）——后者拿不到序列化元信息。
2. **属性名推导依赖 JavaBean 规范**。`getXxx` 推导为 `xxx`，`isXxx`（boolean）推导为 `xxx`。用 Lombok `@Accessors(fluent = true)` 生成 `name()` 而非 `getName()` 时，Lambda 解析直接崩。
3. **泛型父类 / 跨实体引用**。在 `LambdaQueryWrapper<A>` 上误传 `B::getId`，编译期未必报错（继承体系下），运行期列名解析到错误的表字段。
4. **Kotlin 不支持** `LambdaQueryWrapper` / `LambdaUpdateWrapper`，必须用 `KtQueryWrapper` / `KtUpdateWrapper`：

```kotlin
val queryWrapper = KtQueryWrapper(User()).eq(User::name, "sss").eq(User::roleId, "sss2")
userMapper!!.selectList(queryWrapper)
```

5. **需要表别名/函数表达式时 Lambda 无能为力**（如 `t1.name`、`DATE(create_time)`），只能退回 `QueryWrapper` 字符串或 `apply`。

Kotlin 实体定义建议：所有成员变量定义为可空类型并赋初值 `null`，不推荐 `data class` 或全参构造，否则构造空对象时被迫填一堆 null。

---

## 8. 链式调用与 Wrapper 的关系

`ChainWrapper` 是「Wrapper + 执行入口」的组合：内部持有一个普通 Wrapper 收集条件，末端方法（`list/one/page/count/update/remove`）直接调 Service 执行，省掉 `mapper.selectList(wrapper)` 这一步。

```java
// 普通链式调用
UpdateChainWrapper<T> update();
// Lambda式链式调用（不支持Kotlin）
LambdaUpdateChainWrapper<T> lambdaUpdate();

// 等价示例：
query().eq("id", value).one();
lambdaQuery().eq(Entity::getId, value).one();

// 等价示例：
update().eq("id", value).remove();
lambdaUpdate().eq(Entity::getId, value).remove();
```

⚠️ `lambdaQuery()` / `lambdaUpdate()` 是 `IService` 的方法，只能在 Service 层用；Mapper 层只有 `Wrappers.lambdaQuery()` 这种纯构造器，需要自己调 `mapper.selectList(...)`。

⚠️ ChainWrapper 只能用一次，末端方法调用后不要复用实例。

### 自定义 SQL 中复用 Wrapper

版本要求 ≥ 3.0.7。参数名必须为 `ew` 或用 `@Param(Constants.WRAPPER)` 标注，SQL 中用 `${ew.customSqlSegment}` 引用条件片段。

```java
import com.baomidou.mybatisplus.core.conditions.Wrapper;
import com.baomidou.mybatisplus.core.toolkit.Constants;
import org.apache.ibatis.annotations.Param;

public interface UserMapper extends BaseMapper<User> {
    @Select("SELECT * FROM user ${ew.customSqlSegment}")
    List<User> selectByCustomSql(@Param(Constants.WRAPPER) Wrapper<User> wrapper);
}
```

XML 写法：

```xml
<select id="getAll" resultType="MysqlData">
    SELECT * FROM mysql_data ${ew.customSqlSegment}
</select>
```

⚠️ **自定义 SQL 时 Wrapper 不会基于 entity 自动生成 where 子句**，`${ew.customSqlSegment}` 只包含通过 API 添加的条件，实体属性上的值会被忽略——想复用实体条件必须手写。

---

## 9. 速查：高频踩坑清单

1. `or()` 只管一个条件，`A OR B AND C` ≠ `(A OR B) AND C`，要括号就写 `and(i -> ...)`。
2. `likeLeft` 生成 `'%值'`、`likeRight` 生成 `'值%'`，命名与直觉相反；走索引用 `likeRight`。
3. 空 `Collection` / `Map` 会让条件**消失**而非返回空结果，业务上要自己判空短路。
4. 排序/列名来自前端必须白名单，参数绑定只保护 value 不保护 column。
5. `last` 只能调一次且与分页插件冲突。
6. `select()` 重复调用以最后一次为准，不累加。
7. 把字段更新为 null 只能用 `set(col, null)` + `update(null, wrapper)`，`updateById` 会跳过。
8. Wrapper 非线程安全，不能做单例/静态字段。
9. `orderBy` 只有三参签名，第一个 boolean 必传。
10. 自定义 SQL 的 `${ew.customSqlSegment}` 不含 entity 条件。
