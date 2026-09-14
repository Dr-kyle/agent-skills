# Sa-Token 插件

所有插件 `groupId` 均为 `cn.dev33`，**版本必须与 starter 一致**。插件都是在已引入 Sa-Token 的基础上追加。

## JWT 集成

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-jwt</artifactId>
    <version>1.45.0</version>
</dependency>
```

⚠️ **版本兼容硬约束：**
1. `sa-token-jwt` 明确依赖 `hutool-jwt 5.7.14`。项目要么不引 hutool，要么引 **>= 5.7.14**。
2. **hutool 5.8.13 和 5.8.14 会导致类型转换问题**，避开这两个版本。

```yaml
sa-token:
  jwt-secret-key: asdasdasifhueuiwyurfewbfjsdafjk   # 别直接抄这个串
```

### 三种模式：只能注入一个

```java
@Configuration
public class SaTokenConfigure {
    @Bean
    public StpLogic getStpLogicJwt() {
        return new StpLogicJwtForSimple();      // Simple 简单模式：仅替换 token 风格
     // return new StpLogicJwtForMixin();       // Mixin 混入模式：混入部分逻辑
     // return new StpLogicJwtForStateless();   // Stateless 无状态模式：完全舍弃 Redis
    }
}
```

### 功能支持矩阵（前提：jwt + Redis 都在）

| 功能点 | Simple | Mixin | Stateless |
|---|---|---|---|
| 登录数据存储 | Redis | Token 中 | Token 中 |
| Session 存储 | Redis | Redis | **无 Session** |
| 注销下线 | 前后端双清 | 前后端双清 | **仅前端清除** |
| 踢人下线 kickout | 支持 | **不支持** | **不支持** |
| 顶人下线 replaced | 支持 | **不支持** | **不支持** |
| 登录/角色/权限认证 | 支持 | 支持 | 支持 |
| `timeout` | 支持 | 支持 | 支持 |
| `active-timeout` | 支持 | 支持 | **不支持** |
| id 反查 Token | 支持 | 支持 | **不支持** |
| 会话管理 | 支持 | **部分支持** | **不支持** |
| 注解 / 路由鉴权 | 支持 | 支持 | 支持 |
| 账号封禁 | 支持 | 支持 | **不支持** |
| 身份切换 / 二级认证 | 支持 | 支持 | 支持 |

**Stateless 模式的代价**：完全没有 SaSession；注销只是前端清 token（token 在密码学上仍有效直到过期）；kickout/replaced 不可用；`active-timeout` 不可用；id 反查不可用；账号封禁不可用。**Mixin 模式已经失去 kickout/replaced，会话管理只部分支持。**

### 各模式会静默强制的配置（真实 bug 来源）

1. **jwt-simple**：`is-share` 被**强制为 `false`**（`true` 意为每次登录同一 token，与 per-token 的 `setExtra` 数据不兼容）。
2. **jwt-mixin**：`is-concurrent` **必须为 `true`**。因为 mixin 的 token 不记录在持久层，技术上无法被踢，`false` 会让 replaced/kickout 不可用。
3. **jwt-mixin**：`max-try-times` 被**强制为 `-1`**，防止框架误判 token 唯一性。

### 在 token 里挂载额外数据

```java
StpUtil.login(10001, new SaLoginParameter()
        .setExtra("name", "zhangsan")
        .setExtra("age", 18));

String name = StpUtil.getExtra("name");
String name2 = StpUtil.getExtra("tokenValue", "name");
```

### 多账号模式需手动注入

`sa-token-jwt` 只自动注入 `StpUtil`，自定义的 `StpUserUtil` 不会被注入：

```java
@PostConstruct
public void setUserStpLogic() {
    StpUserUtil.setStpLogic(new StpLogicJwtForSimple(StpUserUtil.TYPE));
}
```

### 自定义 jwt 生成算法

```java
@PostConstruct
public void setSaJwtTemplate() {
    SaJwtUtil.setSaJwtTemplate(new SaJwtTemplate() {
        @Override
        public String generateToken(JWT jwt, String keyt) {
            return super.generateToken(jwt, keyt);
        }
    });
}
```

## AOP 注解鉴权

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-spring-aop</artifactId>
    <version>1.45.0</version>
</dependency>
```

**为什么需要它**：默认拦截器模式**只能在 Controller 层用注解**。AOP 模式可以在**任意层级**（如 Service）用注解。

⚠️ **拦截器模式和 AOP 模式不可同时集成**，否则 Controller 层的注解会被校验两次。引入 AOP 后要移除注解拦截器的注册。

## 临时 Token

**无需额外依赖**（`sa-token-temp` 已内嵌到核心包）。用于短时效授权，如邀请链接用 `?token=oEwQ...` 代替 `?id=10014`。

```java
String token = SaTempUtil.createToken("10014", 200);      // value, 有效期(秒)
String value = SaTempUtil.parseToken(token, String.class);
SaTempUtil.getTimeout(token);                             // 剩余有效期
SaTempUtil.deleteToken(token);
```

多业务线加前缀（前缀不对时即使 token 有效也返回 `null`）：

```java
String token = SaTempUtil.createToken("shop_1001", 1200);
SaTempUtil.parseToken(token, "shop_", Long.class);
```

反查（value → tokens）需在创建时开启索引记录：

```java
String token1 = SaTempUtil.createToken(10004, 1200, true);
String token3 = SaTempUtil.createToken(10004, -1, true);   // -1=永久
List<String> list = SaTempUtil.getTempTokenList(10004);
```

改用 jwt 承载（上层 API 不变）：

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-temp-jwt</artifactId>
    <version>1.45.0</version>
</dependency>
```

```yml
sa-token:
  jwt-secret-key: JfdDSgfCmPsDfmsAaQwnXk   # 该模块下【必填】
```

## Alone-Redis（权限缓存与业务缓存分离）

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-alone-redis</artifactId>
    <version>1.45.0</version>
</dependency>
```

⚠️ SpringBoot 4.x 需用 `sa-token-alone-redis-by-spring-boot4`，v1.45.0 时该包**尚未发布到中央仓库**，需自行构建或拷源码。

```yaml
sa-token:
  alone-redis:      # Sa-Token 专用的 Redis
    database: 2
    host: 127.0.0.1
    port: 6379
    timeout: 10s
spring:
  redis:            # 业务用的 Redis
    database: 0
    host: 127.0.0.1
    port: 6379
```

支持 cluster 和 sentinel 模式，参数与 Spring Redis 一致。

⚠️ **只对 `sa-token-redis-template` 和 `sa-token-redis-template-jdk-serializer` 生效。**

## API 参数签名

解决伪造请求身份、参数篡改、抓包重放。**调用方和接收方都要引入**。

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-sign</artifactId>
    <version>1.45.0</version>
</dependency>
```

```yml
sa-token:
  sign:
    secret-key: kQwIOrYvnXmSDkwEiFngrKidMcdrgKor   # 两端必须一致
```

调用方签名：

```java
Map<String, Object> paramMap = new LinkedHashMap<>();
paramMap.put("userId", 10001);
paramMap.put("money", 1000);

// 自动补全 timestamp、nonce、sign 并序列化为 kv 字符串
String paramStr = SaSignUtil.addSignParamsAndJoin(paramMap);
String res = HttpUtil.request("http://b.com/api/addMoney?" + paramStr);
```

接收方校验：

```java
@RequestMapping("addMoney")
public SaResult addMoney(long userId, long money) {
    SaSignUtil.checkRequest(SaHolder.getRequest());
    return SaResult.ok();
}
```

注解形式：

```java
@SaCheckSign
@SaCheckSign(verifyParams = {"id", "name"})
@SaCheckSign(appid = "xm-shop")        // 多应用模式
```

**签名算法**：追加 `timestamp`（13 位毫秒）、`nonce`（32 位随机串）、`sign`。摘要输入是**所有参数按字典序排序、最后追加 `key={secretKey}`**：`md5("money=x&nonce=y&timestamp=z&userId=w&key=秘钥")`。接收方依次校验 timestamp 偏差（**必须取绝对值**以容忍时钟偏移）、nonce 未用过、sign 匹配，并以 `偏差×2` 为 ttl 记录 nonce（2 倍是必需的，1 倍会留出重放窗口）。

⚠️ **`@SaCheckSign` 和 `SaSignUtil.checkRequest` 只能读 Query / Form 参数，读不了 `application/json` 请求体。** JSON body 签名参考社区方案 `https://github.com/ly-chn/sa-token-check-sign-from-body`。

多应用模式：

```java
String paramStr = SaSignMany.getSignTemplate("xm-shop").addSignParamsAndJoin(paramMap);
SaSignMany.getSignTemplate("xm-shop").checkRequest(SaHolder.getRequest());
```

## API Key

"可控的部分授权" —— 用户可以把它交给第三方插件，而不必交出密码或会话 token。特点：随机串、绑定到特定用户、一个用户可有多个、各自携带 scope 集合、有独立过期时间、可随时吊销。

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-apikey</artifactId>
    <version>1.45.0</version>
</dependency>
```

```java
// 创建
ApiKeyModel akModel = SaApiKeyUtil.createApiKeyModel(10001).setTitle("test");
System.out.println("API Key 值：" + akModel.getApiKey());
SaApiKeyUtil.saveApiKey(akModel);
SaApiKeyUtil.deleteApiKey(apiKey);

// 完整字段
ApiKeyModel m = new ApiKeyModel();
m.setLoginId(10001);
m.setApiKey("AK-NAO6u57zbOWCmLaiVQuVW2tyt3rHpZrXkaQp");
m.setTitle("commit");
m.setIntro("提交代码专用");
m.addScope("commit", "pull");
m.setExpiresTime(System.currentTimeMillis() + 2592000);   // 13位时间戳，-1=永不失效
m.setIsValid(true);
m.addExtra("name", "张三");
SaApiKeyUtil.saveApiKey(m);

// 查询
SaApiKeyUtil.getApiKey("AK-...");
SaApiKeyUtil.getLoginIdByApiKey("AK-...");
SaApiKeyUtil.getApiKeyList(10001);

// 校验
SaApiKeyUtil.checkApiKey("AK-...");                    // 无效抛 ApiKeyException
SaApiKeyUtil.checkApiKeyScope("AK-...", "userinfo");   // 无权限抛 ApiKeyScopeException
SaApiKeyUtil.hasApiKeyScope("AK-...", "userinfo");
SaApiKeyUtil.checkApiKeyLoginId("AK-...", 10001);
SaApiKeyUtil.currentApiKey();                          // 当前请求的 ApiKeyModel
```

注解：

```java
@SaCheckApiKey
@SaCheckApiKey(scope = "userinfo")
@SaCheckApiKey(scope = {"userinfo", "chat"})                       // AND
@SaCheckApiKey(scope = {"userinfo", "chat"}, mode = SaMode.OR)     // OR
```

**提交格式**：请求参数或 header，名称为 **`apikey`**（全小写），如 `/user/getInfo?apikey=AK-...`；也支持 Basic 风格 `http://AK-...@localhost:8081/user/getInfo`。

数据库模式实现 `SaApiKeyDataLoader`：

```java
@Component
public class SaApiKeyDataLoaderImpl implements SaApiKeyDataLoader {
    @Autowired SaApiKeyMapper apiKeyMapper;

    @Override public Boolean getIsRecordIndex() { return false; }

    @Override public ApiKeyModel getApiKeyModelFromDatabase(String namespace, String apiKey) {
        return apiKeyMapper.getApiKeyModel(apiKey);
    }
}
```

⚠️ 数据库模式的后果：不再维护索引数据，**`getApiKeyList(loginId)` 会失效**；`saveApiKey` 只写缓存（需自己持久化）；`deleteApiKey` 只删缓存（需双删）。

多账号：`new SaApiKeyTemplate("admin-apikey")` 用独立 namespace。

## SpEL 注解鉴权

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-spring-el</artifactId>
    <version>1.45.0</version>
</dependency>
```

底层依赖 Spring AOP，所以要单独引入。注解是 `@SaCheckEL`：

```java
@SaCheckEL("stp.checkLogin()")
@SaCheckEL("stp.checkPermission('user:edit')")
@SaCheckEL("NEED( #name.length() > 3 )")                            // #参数名 引用方法参数
@SaCheckEL("NEED( stp.getSession().get('name') == 'zhangsan' )")
@SaCheckEL("stp.checkPermission( this.permissionCode )")            // this.字段 引用当前类成员
```

语法要素：根对象 `stp` 即 `StpLogic`（可调它的任意方法）；`NEED( boolean表达式 )` 包装任意断言；`#paramName` 引用方法参数；`this.field` 引用当前类成员；字符串用单引号。

多账号注册额外根对象：

```java
@PostConstruct
public void rewriteSaStrategy() {
    SaAnnotationStrategy.instance.checkELRootMapExtendFunction = rootMap -> {
        rootMap.put("stpUser", StpUserUtil.getStpLogic());
    };
}
// 之后可用 @SaCheckEL("stpUser.checkLogin()")
```

可与 `@SaIgnore` 组合跳过校验。IDEA 插件 **SpEL Assistant** 提供语法提示。

## 缓存层扩展（SaTokenDao 实现）

全部持久化抽象到 `SaTokenDao` 接口。官方实现：

| artifactId | 说明 |
|---|---|
| （核心包内置） | 默认，内存实现 |
| `sa-token-redis-template` | RedisTemplate 集成（**推荐**），默认 JSON 格式 |
| `sa-token-redis-template-jdk-serializer` | RedisTemplate + JDK 默认序列化 |
| `sa-token-redis-jackson` | RedisTemplate + Jackson |
| `sa-token-hutool-timed-cache` | Hutool Timed-Cache（内存） |
| `sa-token-caffeine` | Caffeine（内存） |
| `sa-token-redisson` | Redisson |
| `sa-token-redisson-spring-boot-starter` | Redisson + SpringBoot 自动配置 |
| `sa-token-redisx` | Redisx |

⚠️ **已废弃的旧 artifactId，不要再用**：`sa-token-dao-redis` → `sa-token-redis`；`sa-token-dao-redis-jackson` → `sa-token-redis-jackson`；`sa-token-dao-redis-fastjson` → `sa-token-redis-fastjson`；`sa-token-dao-redisson-jackson` → `sa-token-redisson-jackson`；`sa-token-dao-redisx` → `sa-token-redisx`。

## JSON 序列化扩展

序列化层调用链是 `String 序列化` → `JSON 序列化`。JSON 层通过 **SPI 自动注入**，只加依赖、无需写代码。

| artifactId | 目标 |
|---|---|
| `sa-token-jackson` | Jackson（`com.fasterxml.jackson`），SpringBoot 2/3 |
| `sa-token-jackson3` | Jackson 3（`tools.jackson.core`），SpringBoot 4 / Java 17+ |
| `sa-token-fastjson` | Fastjson |
| `sa-token-fastjson2` | Fastjson2 |
| `sa-token-snack3` | Snack3 |
| `sa-token-snack4` | Snack4 |

`sa-token-spring-boot-starter`（含 boot3）会**自动带上 Jackson**，默认无需配置。要换就加对应插件依赖。

String 层需代码切换：

```java
@PostConstruct
public void rewriteComponent() {
    SaManager.setSaSerializerTemplate(new SaSerializerTemplateForJdkUseBase64());
    // 还有 SaSerializerTemplateForJdkUseHex()、SaSerializerTemplateForJdkUseISO_8859_1()
}
```

⚠️ **切换序列化方案（JDK ↔ Jackson）必须先清空 Redis 历史数据**，否则报 `SerializationException`。

`sa-token-serializer-features` 是娱乐向插件（天干地支/元素周期表/emoji 编码 base64），**官方明说不建议上生产**。

## Thymeleaf 集成

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-thymeleaf</artifactId>
    <version>1.45.0</version>
</dependency>
```

```java
@Bean
public SaTokenDialect getSaTokenDialect() { return new SaTokenDialect(); }
```

标签：`sa:login`、`sa:notLogin`、`sa:hasRole`、`sa:hasRoleAnd`、`sa:hasRoleOr`、`sa:notRole`、`sa:hasPermission`、`sa:hasPermissionAnd`、`sa:hasPermissionOr`、`sa:notPermission`（And/Or 接逗号分隔列表）。

想调任意 API：在注入的 `ThymeleafViewResolver` 上 `viewResolver.addStaticVariable("stp", StpUtil.stpLogic)`。

⚠️ 若 `SaTokenConfigure` 同时 `implements WebMvcConfigurer`，这样做可能造成**循环依赖** —— 放到单独的配置类里。

IDE 提示命名空间：`xmlns:sa="http://www.thymeleaf.org/extras/sa-token"`。

## Freemarker 集成

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-freemarker</artifactId>
    <version>1.45.0</version>
</dependency>
```

```java
@PostConstruct
public void init() {
    configurer.getConfiguration().setSharedVariable("sa", new SaTokenTemplateModel());
    configurer.getConfiguration().setSharedVariable("stp", StpUtil.stpLogic);
}
```

标签：`<@sa.login>`、`<@sa.notLogin>`、`<@sa.hasRole value="admin">`、`hasRoleAnd`、`hasRoleOr`、`notRole`、`hasPermission`、`hasPermissionAnd`、`hasPermissionOr`、`notPermission`；以及 `<#if stp.isLogin()>${stp.getSession().get('name')}</#if>`。

## Quick-Login 快速登录

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-quick-login</artifactId>
    <version>1.45.0</version>
</dependency>
```

零代码注入一个登录页。⚠️ 官方明确说明**不可定制、不适合正常项目**，只适合内部小工具（如服务器监控页）。默认账密 `sa / 123456`，也支持 HTTP Basic `http://sa:123456@localhost:8081/`。

配置前缀是顶级 **`sa:`**（不是 `sa-token:`），全部可选：`sa.name`、`sa.pwd`、`sa.auto`（每次启动随机账密，会覆盖 name/pwd）、`sa.auth`（false=停止拦截）、`sa.title`、`sa.copr`、`sa.include`、`sa.exclude`。运行时读取：`SaQuickManager.getConfig().getName()`。
