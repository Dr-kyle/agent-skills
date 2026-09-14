# 代码生成器（New，3.5.1+）

⚠️ **全新的代码生成器添加于 3.5.1 版本，且对历史版本不兼容。** 3.5.1 以下请参考旧版「代码生成器」文档。旧版是 `new GlobalConfig().setXxx()` 的 setter 风格，新版全部改成 **Builder 模式**，两套 API 不能混用。

本文基于 **3.5.17**，方法名取自该版本源码。

## 依赖引入

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.5.17</version>
</dependency>
```

⚠️ **模板引擎是 optional 依赖，必须自行引入**，否则运行时报 `NoClassDefFoundError` / `ClassNotFoundException`。

| 引擎类 | 需引入的依赖 | 报错时缺的类 |
|---|---|---|
| `VelocityTemplateEngine`（**默认**） | `org.apache.velocity:velocity-engine-core`（3.5.17 对应 2.4.1） | `org.apache.velocity.app.VelocityEngine` |
| `FreemarkerTemplateEngine` | `org.freemarker:freemarker`（2.3.34） | `freemarker.template.Configuration` |
| `BeetlTemplateEngine` | `com.ibeetl:beetl`（3.17.0.RELEASE） | `org.beetl.core.GroupTemplate` |
| `EnjoyTemplateEngine` | `com.jfinal:enjoy`（5.2.2） | `com.jfinal.template.Engine` |

⚠️ **不调用 `.templateEngine(...)` 时会 fallback 到 `VelocityTemplateEngine`** —— 也就是说即便你什么都没配，也必须有 velocity 依赖在 classpath 上。

还想适配其他模板引擎，继承 `AbstractTemplateEngine` 并参考已有实现。

另外别忘了 **JDBC 驱动** —— 生成器要真连数据库读元数据。

## 最小可用示例

官方首屏示例，一眼看懂整体形状：

```java
FastAutoGenerator.create("url", "username", "password")
        .globalConfig(builder -> builder
                .author("Baomidou")
                .outputDir(Paths.get(System.getProperty("user.dir")) + "/src/main/java")
                .commentDate("yyyy-MM-dd")
        )
        .packageConfig(builder -> builder
                .parent("com.baomidou.mybatisplus")
                .entity("entity")
                .mapper("mapper")
                .service("service")
                .serviceImpl("service.impl")
                .xml("mapper.xml")
        )
        .strategyConfig(builder -> builder
                .entityBuilder()
                .enableLombok()
        )
        .templateEngine(new FreemarkerTemplateEngine())
        .execute();
```

## 快速生成（完整示例）

在 `CodeGenerator` 的 `main` 方法里直接写生成器代码，配好后直接运行。

```java
public static void main(String[] args) {
    FastAutoGenerator.create("url", "username", "password")
            .globalConfig(builder -> {
                builder.author("baomidou") // 设置作者
                        .enableSwagger() // 开启 swagger 模式
                        .outputDir("D://"); // 指定输出目录
            })
            .dataSourceConfig(builder ->
                    builder.typeConvertHandler((globalConfig, typeRegistry, metaInfo) -> {
                        int typeCode = metaInfo.getJdbcType().TYPE_CODE;
                        if (typeCode == Types.SMALLINT) {
                            // 自定义类型转换
                            return DbColumnType.INTEGER;
                        }
                        return typeRegistry.getColumnType(metaInfo);
                    })
            )
            .packageConfig(builder ->
                    builder.parent("com.baomidou.mybatisplus.samples.generator") // 设置父包名
                            .moduleName("system") // 设置父包模块名
                            .pathInfo(Collections.singletonMap(OutputFile.xml, "D://")) // 设置mapperXml生成路径
            )
            .strategyConfig(builder ->
                    builder.addInclude("t_simple") // 设置需要生成的表名
                            .addTablePrefix("t_", "c_") // 设置过滤表前缀
            )
            .templateEngine(new FreemarkerTemplateEngine()) // 使用Freemarker引擎模板，默认的是Velocity引擎模板
            .execute();
}
```

## 交互式生成

每个 `xxxConfig` 都有一个 `BiConsumer<Function<String,String>, XxxBuilder>` 重载，第一个参数 `scanner` 就是控制台输入函数。运行后按提示输入即可。

```java
public static void main(String[] args) {
    FastAutoGenerator.create("url", "username", "password")
            // 全局配置
            .globalConfig((scanner, builder) -> builder.author(scanner.apply("请输入作者名称？")))
            // 包配置
            .packageConfig((scanner, builder) -> builder.parent(scanner.apply("请输入包名？")))
            // 策略配置
            .strategyConfig((scanner, builder) -> builder.addInclude(getTables(scanner.apply("请输入表名，多个英文逗号分隔？所有输入 all")))
                    .entityBuilder()
                    .enableLombok()
                    .addTableFills(
                            new Column("create_time", FieldFill.INSERT)
                    )
                    .build())
            // 使用Freemarker引擎模板，默认的是Velocity引擎模板
            .templateEngine(new FreemarkerTemplateEngine())
            .execute();
}


// 处理 all 情况
protected static List<String> getTables(String tables) {
    return "all".equals(tables) ? Collections.emptyList() : Arrays.asList(tables.split(","));
}
```

⚠️ 注意上面 `strategyConfig` 里**链到了 `.entityBuilder()` 之后必须 `.build()` 收尾**才能回到 `StrategyConfig.Builder`。

## 生成方式：DefaultQuery vs SQLQuery

| | DefaultQuery（元数据查询，**默认**） | SQLQuery（SQL 查询） |
|---|---|---|
| 优点 | 按通用接口读数据库元数据，通用性好 | 精确可控 |
| 缺点 | 依赖数据库厂商驱动实现，部分类型处理可能不理想 | 通用性不强，同厂商不同版本可能有兼容问题（如 H2 只支持 1.X） |
| 维护 | 默认方式 | ⚠️ **后期不再维护** |

已知数据库（无版本兼容问题）可继续用 SQL 查询方式，需同时指定 `dbQuery` 与 `typeConvert`：

```java
// MYSQL 示例 切换至SQL查询方式,需要指定好 dbQuery 与 typeConvert 来生成
FastAutoGenerator.create("url", "username", "password")
                .dataSourceConfig(builder ->
                        builder.databaseQueryClass(SQLQuery.class)
                                .typeConvert(new MySqlTypeConvert())
                                .dbQuery(new MySqlQuery())
                )
                // Other Config ...
```

### 元数据查询的已知问题（⚠️ 高频踩坑）

- **不支持使用 `NotLike` 的方式反向生成表。**
- **无法读取表注释** —— 生成出来的实体没有注释。解法是在连接串上加参数：
  - MySQL：`remarks=true&useInformationSchema=true`
  - Oracle：`remarks=true` 或 `remarksReporting=true`（某些驱动版本）
  - SqlServer：**驱动不支持**
- **部分 PostgreSQL 类型处理不佳**（`json`、`jsonb`、`uuid`、`xml`、`money`）。解法：转成自定义类型 + 自定义 `TypeHandler`；或扩展 `typeConvertHandler` 处理（3.5.3.3 后增加了 `typeName` 获取）。
- **MySQL `tinyint` 转换问题**：
  - 字段长度为 1 时无法转成 `Boolean`，建议连接串加 `&tinyInt1isBit=true`；
  - 字段长度 > 1 时默认转成 `Byte`，想继续转成 `Integer`：

```java
FastAutoGenerator.create("url", "username", "password")
        .dataSourceConfig(builder ->
                builder.typeConvertHandler((globalConfig, typeRegistry, metaInfo) -> {
                    // 兼容旧版本转换成Integer
                    if (JdbcType.TINYINT == metaInfo.getJdbcType()) {
                        return DbColumnType.INTEGER;
                    }
                    return typeRegistry.getColumnType(metaInfo);
                })
        );
```

---

# 各配置项

## DataSourceConfig（`.dataSourceConfig(builder -> ...)`）

url / username / password 通常在 `FastAutoGenerator.create(url, username, password)` 里给，Builder 里配的是"怎么读"。

| 方法 | 说明 |
|---|---|
| `driverClassName(String)` | 驱动类名 |
| `schema(String)` | 数据库 schema name（PostgreSQL 等需要，如 `public`） |
| `dbQuery(IDbQuery)` | 数据库信息查询类，默认按 dbType 自动选；实现 `IDbQuery` 可自定义查询 SQL |
| `typeConvert(ITypeConvert)` | 字段类型 → Java 类型转换，默认按 dbType 自动选 |
| `typeConvertHandler(ITypeConvertHandler)` | **更推荐的类型转换扩展点**（见上文 tinyint 示例），3.5.3.3 后可取 `typeName` |
| `keyWordsHandler(IKeyWordsHandler)` | 数据库关键字处理器 |
| `databaseQueryClass(Class<? extends AbstractDatabaseQuery>)` | 切换生成方式：`DefaultQuery.class`（默认）/ `SQLQuery.class` |
| `addConnectionProperty(String key, String value)` | 追加 JDBC 连接属性（如 `remarks=true`），比拼 URL 干净 |

## GlobalConfig（`.globalConfig(builder -> ...)`）

| 方法 | 说明 |
|---|---|
| `outputDir(String)` | 生成文件输出目录（⚠️ 默认是 **D 盘根目录**，几乎一定要改） |
| `author(String)` | 作者名，写进类注释 |
| `commentDate(String pattern)` / `commentDate(Supplier<String>)` | 注释日期格式 / 自定义日期供给 |
| `dateType(DateType)` | 时间类型策略：`ONLY_DATE`（`java.util.Date`）、`SQL_PACK`（`java.sql` 包）、`TIME_PACK`（`java.time`，**默认**） |
| `enableKotlin()` | 开启 Kotlin 模式，生成 `.kt` |
| `enableSwagger()` | 生成 Swagger2 注解 |
| `enableSpringdoc()` | 生成 springdoc（OpenAPI 3）注解 |
| `disableOpenDir()` | 生成后**不要**自动打开输出目录（默认会打开，脚本化执行时必关） |
| `disableServiceInterface()` | 不生成 Service 接口，只生成实现类 |

⚠️ `enableSwagger()` 与 `enableSpringdoc()` **按项目实际用的那套选一个**，别同时开。

## PackageConfig（`.packageConfig(builder -> ...)`）

| 方法 | 说明 |
|---|---|
| `parent(String)` | 父包名（顶级包） |
| `moduleName(String)` | 父包模块名，拼在 parent 之后 |
| `entity(String)` | Entity 子包名 |
| `service(String)` | Service 接口子包名 |
| `serviceImpl(String)` | Service 实现类子包名 |
| `mapper(String)` | Mapper 接口子包名 |
| `xml(String)` | Mapper XML 子包名 |
| `controller(String)` | Controller 子包名 |
| `pathInfo(Map<OutputFile, String>)` | **物理路径**覆盖，key 为 `OutputFile` 枚举 |

`OutputFile` 枚举值：`entity`、`service`、`serviceImpl`、`mapper`、`xml`、`controller`、`parent`。

⚠️ **包名 ≠ 输出路径。** `xml("mapper.xml")` 只是包名；要把 XML 放到 `src/main/resources/mapper` 下必须用 `pathInfo`：

```java
.packageConfig(builder ->
        builder.parent("com.example")
                .pathInfo(Collections.singletonMap(OutputFile.xml, "D://")) // 设置mapperXml生成路径
)
```

## StrategyConfig（`.strategyConfig(builder -> ...)`）

表级别的过滤与开关。

| 方法 | 说明 |
|---|---|
| `addInclude(String... / List<String> / String)` | 需要生成的表名（不配 = 全部表） |
| `addExclude(String... / List<String>)` | 需要排除的表名 |
| `likeTable(LikeTable)` / `notLikeTable(LikeTable)` | 模糊匹配 / 模糊排除表名，**二选一** |
| `addTablePrefix(String...)` / `addTableSuffix(String...)` | 过滤表名前缀 / 后缀（如 `t_`、`c_`） |
| `addFieldPrefix(String...)` / `addFieldSuffix(String...)` | 过滤字段前缀 / 后缀 |
| `enableCapitalMode()` | 大写命名模式 |
| `enableSkipView()` | 跳过视图 |
| `enableSchema()` | 启用 schema |
| `disableSqlFilter()` | 关闭 SQL 模糊表名匹配，改为**内存过滤**；此时 `include` / `exclude` 允许正则，`likeTable` / `notLikeTable` 失效 |
| `outputFile(IOutputFile)` | 自定义输出文件策略 |
| `entityBuilder()` / `mapperBuilder()` / `serviceBuilder()` / `controllerBuilder()` | 进入四个子策略 |

⚠️ **`disableSqlFilter()` 的使用场景**：`enableSqlFilter` 默认 `true`，会把表名过滤下推成 SQL。若有 SQL 语法兼容性问题（**已知 MyCat 中间件无法使用**）就关掉它。

⚠️ 从子 Builder 链回 `StrategyConfig.Builder` 要调 `.build()`；`FastAutoGenerator` 的 lambda 里最后一个子 Builder 可以省略（会自动收尾），但**链式连用多个子 Builder 时必须显式 `.build()`**。

### entityBuilder() —— 实体策略

| 方法 | 说明 |
|---|---|
| `enableLombok()` / `enableLombok(ClassAnnotationAttributes...)` | 生成 Lombok 注解，可指定具体注解 |
| `enableChainModel()` | 链式模型（生成 `@Accessors(chain=true)` 风格的 setter） |
| `enableActiveRecord()` | ActiveRecord 模式（实体继承 `Model`） |
| `enableColumnConstant()` | 生成字段常量 |
| `enableTableFieldAnnotation()` | 生成字段上的 `@TableField` 注解 |
| `enableRemoveIsPrefix()` | 移除 Boolean 字段的 `is` 前缀 |
| `enableSerialAnnotation()` | 生成 `@Serial` 注解 |
| `disableSerialVersionUID()` | 不生成 `serialVersionUID` |
| `superClass(Class<?> / String)` | 父类 |
| `addSuperEntityColumns(String...)` | 父类已有的公共字段（生成时跳过） |
| `addIgnoreColumns(String...)` | 忽略的字段 |
| `naming(NamingStrategy)` / `columnNaming(NamingStrategy)` | 表名 / 列名映射策略：`underline_to_camel`（默认）、`no_change` |
| `idType(IdType)` | 主键生成策略 |
| `versionColumnName(String)` / `versionPropertyName(String)` | 乐观锁列名 / 属性名 |
| `logicDeleteColumnName(String)` / `logicDeletePropertyName(String)` | 逻辑删除列名 / 属性名 |
| `addTableFills(IFill... / List<IFill>)` | 自动填充字段，如 `new Column("create_time", FieldFill.INSERT)` |
| `convertFileName(ConverterFileName)` / `formatFileName(String)` | 文件名转换函数 / 格式（如 `%sEntity`） |
| `enableFileOverride()` | 覆盖已有文件（⚠️ **默认不覆盖**，重新生成时"没生效"多半是这个原因） |
| `javaTemplate(String)` / `kotlinTemplatePath(String)` | 自定义模板路径 |
| `fieldUseJavaDoc(boolean)` / `toString(boolean)` | 字段使用 JavaDoc / 生成 toString |
| `addClassAnnotation(ClassAnnotationAttributes)` | 追加类注解 |
| `tableFieldAnnotationHandler(...)` / `tableAnnotationHandler(...)` / `tableFieldMetaInfoCustomizer(...)` | 注解与元信息的高级定制钩子 |
| `disable()` | 不生成实体 |

⚠️ **`enableChainModel()`**：3.3.2 以下版本默认生成链式模型，3.3.2 以后默认**不**生成，需要就显式开启。

### mapperBuilder() —— Mapper 策略

| 方法 | 说明 |
|---|---|
| `superClass(Class<?> / String)` | Mapper 父接口 |
| `enableMapperAnnotation()` | 生成 `@Mapper` 注解 |
| `mapperAnnotation(Class<? extends Annotation>)` | 指定别的注解类 |
| `enableBaseResultMap()` | XML 里生成 `BaseResultMap` |
| `enableBaseColumnList()` | XML 里生成 `Base_Column_List` |
| `cache(Class<? extends Cache>)` | XML 二级缓存实现类 |
| `convertMapperFileName(...)` / `formatMapperFileName(String)` | Mapper 文件名，如 `%sDao` |
| `convertXmlFileName(...)` / `formatXmlFileName(String)` | XML 文件名 |
| `mapperTemplate(String)` / `mapperXmlTemplate(String)` | 自定义模板 |
| `enableFileOverride()` | 覆盖已有文件 |
| `disable()` / `disableMapper()` / `disableMapperXml()` | 全部不生成 / 只不生成接口 / 只不生成 XML |
| `generateMapperMethodHandler(...)` | 自定义生成的 Mapper 方法 |

⚠️ **加了 `@Mapper` 注解就不要再 `@MapperScan` 重复扫**（不会报错，但语义重复）；反过来，两者**至少要有一个**，否则运行时 `Invalid bound statement (not found)`。

### serviceBuilder() —— Service 策略

| 方法 | 说明 |
|---|---|
| `superServiceClass(Class<?> / String)` | Service 接口父类，默认 `IService` |
| `superServiceImplClass(Class<?> / String)` | ServiceImpl 父类，默认 `ServiceImpl` |
| `convertServiceFileName(ConverterFileName)` | 接口文件名转换函数 |
| `convertServiceImplFileName(ConverterFileName)` | 实现类文件名转换函数 |
| `formatServiceFileName(String)` / `formatServiceImplFileName(String)` | 格式串，如 `%sService` / `%sServiceImpl` |
| `serviceTemplate(String)` / `serviceImplTemplate(String)` | 自定义模板 |
| `enableFileOverride()` | 覆盖已有文件 |
| `disable()` / `disableService()` / `disableServiceImpl()` | 不生成 |

⚠️ 默认生成的接口名是 `IUserService`（带 `I` 前缀）。不想要就 `formatServiceFileName("%sService")`。

### controllerBuilder() —— Controller 策略

| 方法 | 说明 |
|---|---|
| `superClass(Class<?> / String)` | Controller 父类 |
| `enableRestStyle()` | 生成 `@RestController`（默认是 `@Controller`） |
| `enableHyphenStyle()` | 请求映射路径驼峰转连字符（`userInfo` → `user-info`） |
| `convertFileName(...)` / `formatFileName(String)` | 文件名，如 `%sAction` |
| `template(String)` | 自定义模板 |
| `enableFileOverride()` | 覆盖已有文件 |
| `disable()` | 不生成 Controller |

## TemplateConfig（`.templateConfig(builder -> ...)`）

指定各类型模板文件路径，或**关掉某类文件的生成**。

| 方法 | 说明 |
|---|---|
| `entity(String)` | Java 实体类模板 |
| `entityKt(String)` | Kotlin 实体类模板 |
| `service(String)` / `serviceImpl(String)` | Service 接口 / 实现类模板 |
| `mapper(String)` / `xml(String)` | Mapper 接口 / XML 模板 |
| `controller(String)` | Controller 模板 |
| `disable(TemplateType...)` | 禁用指定类型模板 |
| `disable()` | 禁用全部模板 |

`TemplateType` 枚举：`ENTITY`、`SERVICE`、`SERVICE_IMPL`、`CONTROLLER`、`MAPPER`、`XML`。

```java
.templateConfig(builder -> builder.disable(TemplateType.ENTITY, TemplateType.XML))
```

⚠️ **模板文件后缀要和引擎匹配** —— Velocity 是 `.vm`，Freemarker 是 `.ftl`，Beetl 是 `.btl`。配了 `.vm` 路径又用 `FreemarkerTemplateEngine` 会找不到模板。

⚠️ 用 `enableKotlin()` 时要配的是 `entityKt`，不是 `entity`。

## InjectionConfig（`.injectionConfig(builder -> ...)`）

自定义参数与自定义输出文件。

| 方法 | 说明 |
|---|---|
| `customMap(Map<String, Object>)` | 自定义参数，模板中通过 `cfg.xxx` 引用 |
| `customFile(Map<String, String>)` | 自定义文件：文件名 → 模板路径 |
| `customFile(CustomFile)` / `customFile(List<CustomFile>)` / `customFile(Consumer<CustomFile.Builder>)` | 自定义文件的三种写法 |
| `beforeOutputFile(BiConsumer<TableInfo, Map<String, Object>>)` | **输出文件前的钩子**，可按表动态往 map 里塞参数 |

`beforeOutputFile` 是最有用的扩展点 —— 一次生成多张表时，每张表的参数可以不同。

---

## 实践建议

- **生成器代码放 test 目录**，别打进生产包；跑完把 `CodeGenerator` 留着，表结构变更时重跑。
- ⚠️ **默认不覆盖已有文件。** 改完策略重新跑却"没变化"，先看是不是缺 `enableFileOverride()`。反过来，开了覆盖就会**冲掉你手写的业务代码** —— 实际项目里只对 Entity / Mapper / XML 开覆盖，Service 与 Controller 保持不覆盖。
- **脚本化执行必加 `disableOpenDir()`**，否则会去调用系统文件管理器。
- MySQL 想要表注释，连接串务必带 `remarks=true&useInformationSchema=true`，否则生成的实体一行注释都没有，事后补比重新生成贵得多。
- 官方更多示例见 `mybatis-plus-generator` 的 test 包：`FastAutoGeneratorTest`、`TableField 元数据扩展处理`。
