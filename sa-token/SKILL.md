---
name: sa-token
description: "Use this skill for any task involving Sa-Token — the Java (Spring Boot / WebFlux / Solon) authentication & authorization framework (cn.dev33). Covers dependency selection, login auth, permission & role auth, annotation & route interception, SaSession, token config, Redis integration, frontend-backend separation, JWT, multi-account, kickout/ban/second-factor auth, microservice gateway auth, Same-Token, SSO, OAuth2, and troubleshooting. Triggers include adding login/logout, protecting endpoints with @SaCheckLogin/@SaCheckPermission, implementing StpInterface, configuring sa-token in application.yml, gateway unified auth, building an SSO or OAuth2 server, or debugging errors like 未能读取到有效Token / Token无效 / SaTokenContext 上下文尚未初始化."
---

# Sa-Token 开发指南

Sa-Token 是一个轻量级 Java 权限认证框架，主要解决 **登录认证、权限认证、单点登录、OAuth2、微服务网关鉴权** 等一系列权限相关问题。

> 官方文档：<https://sa-token.com/doc.html> · 源码：<https://github.com/dromara/sa-token>
> 本 skill 基于 **v1.45.0** 文档整理。API 以项目实际依赖版本为准，跨大版本升级时先查[更新日志](https://sa-token.com/doc.html#/more/update-log)。

## 核心心智模型（先理解这一点，能避免 80% 的困惑）

**Sa-Token 不接管登录流程。** 它没有 Shiro 的 `Realm`，也没有 Spring Security 的 `UserDetailsService` / 过滤器链 DSL。你自己校验账号密码，校验通过后调用 `StpUtil.login(id)` 通知框架"这个会话属于这个 id"。框架负责的是：生成 token、维护会话、后续请求的鉴权。

```java
// 完整的登录接口就是这样
@RequestMapping("doLogin")
public SaResult doLogin(String username, String password) {
    // 1. 你自己比对密码（查库、加盐哈希等）
    if("zhang".equals(username) && "123456".equals(password)) {
        // 2. 通知框架：会话登录
        StpUtil.login(10001);
        // 3. 前后端分离时把 token 返给前端
        return SaResult.data(StpUtil.getTokenInfo());
    }
    return SaResult.error("登录失败");
}
```

三个必须理解的要点：
1. `StpUtil.login()` 只接受 **简单类型**（`long` / `int` / `String`），不要传 `User`、`Admin` 等复杂对象。用户信息存到 `StpUtil.getSession().set("user", user)`。
2. **注解鉴权默认关闭**，必须手动注册 `SaInterceptor`（见下文），这是新手最高频的坑。
3. **权限数据由你提供** —— 实现 `StpInterface` 接口。它不会在项目启动时执行，只在实际调用鉴权代码时才触发（启动时没日志是正常的）。

## 依赖选择（选错会直接导致启动失败或 InvalidContextException）

`groupId` 统一为 `cn.dev33`，所有 `sa-token-*` 依赖**必须保持同一版本**（版本错配会报 `ClassNotFoundException: cn.dev33.satoken.same.SaSameTemplate` 之类的错）。

| 场景 | artifactId |
|---|---|
| SpringBoot 2.x + SpringMVC（Servlet） | `sa-token-spring-boot-starter` |
| SpringBoot 3.x + SpringMVC | `sa-token-spring-boot3-starter` |
| SpringBoot 4.x + SpringMVC | `sa-token-spring-boot4-starter` |
| WebFlux / SpringCloud Gateway（Reactor），Boot 2.x | `sa-token-reactor-spring-boot-starter` |
| WebFlux / Gateway，Boot 3.x / 4.x | `sa-token-reactor-spring-boot3-starter` / `-boot4-starter` |
| Solon | `sa-token-solon-plugin` |
| 纯 Servlet 容器（无 Spring） | `sa-token-servlet` |
| 非 SpringMVC/WebFlux 且非 Servlet | `sa-token-core` |

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-spring-boot-starter</artifactId>
    <version>1.45.0</version>
</dependency>
```

**硬性约束：**
- ⚠️ **绝不可在同一个项目里同时引入 Servlet starter 和 Reactor starter**，会导致项目无法启动。
- ⚠️ SpringBoot 3.x/4.x 必须用对应的 `boot3`/`boot4` starter，用错会报错。
- ⚠️ 微服务架构下，**网关和各业务服务分别引入**，不要在顶层父 pom 里引入 Sa-Token。
- ⚠️ WebFlux/Gateway 环境**没有拦截器机制**，必须注册 `SaReactorFilter`，注解鉴权那套在这里不适用。

## 最小可用配置

零配置即可启动，以下均为可选项。完整配置项见 `references/config-reference.md`。

```yaml
sa-token:
  # token 名称（同时也是 cookie 名称、Redis key 前缀）
  token-name: satoken
  # token 有效期（秒），默认30天，-1 代表永久有效
  timeout: 2592000
  # token 最低活跃频率（秒），超过此时间未访问会被冻结，-1 代表不冻结
  active-timeout: -1
  # 是否允许同一账号多地同时登录（false 时新登录挤掉旧登录）
  is-concurrent: true
  # 多人登录同一账号时是否共用一个 token
  is-share: false
  # token 风格：uuid、simple-uuid、random-32、random-64、random-128、tik
  token-style: uuid
  # 是否输出操作日志
  is-log: true
```

⚠️ 配置必须是**顶级** `sa-token:`，不能嵌套在 `spring:` 下面。改配置不生效时，在 `SpringApplication.run(...)` 之后打印 `SaManager.getConfig()` 验证。注意：**代码配置（`@Bean @Primary`）会覆盖 yml 配置**。

## 登录认证常用 API

```java
StpUtil.login(10001);                    // 会话登录
StpUtil.login(10001, "PC");              // 指定设备类型（用于同端互斥登录）
StpUtil.login(10001, false);             // 非[记住我]模式，关浏览器即失效
StpUtil.login(10001, 86400);             // 指定本次 token 有效期（秒）

StpUtil.isLogin();                       // 是否已登录 → true/false
StpUtil.checkLogin();                    // 校验，未登录抛 NotLoginException
StpUtil.getLoginId();                    // 获取账号id，未登录抛异常
StpUtil.getLoginIdDefaultNull();         // 未登录返回 null
StpUtil.getLoginIdAsLong();              // 转型获取
StpUtil.getTokenValue();                 // 当前请求提交的 token
StpUtil.getTokenInfo();                  // token 详细信息（SaTokenInfo）

StpUtil.logout();                        // 当前会话注销
StpUtil.logout(10001);                   // 强制指定账号下线（所有端）
StpUtil.kickout(10001);                  // 踢指定账号下线
```

`logout` 与 `kickout` 的区别：`logout` 是干净移除（下次访问提示"Token无效"）；`kickout` 保留 token 并打标记（下次访问提示"Token已被踢下线"），便于给前端明确原因。

## 权限认证

### 第一步：实现 `StpInterface` 提供权限数据

```java
@Component  // 必须被 SpringBoot 扫描到
public class StpInterfaceImpl implements StpInterface {

    /** 返回一个账号所拥有的权限码集合 */
    @Override
    public List<String> getPermissionList(Object loginId, String loginType) {
        // 实际项目中根据 loginId 查库
        return Arrays.asList("user.add", "user.update", "art.*");
    }

    /** 返回一个账号所拥有的角色标识集合 */
    @Override
    public List<String> getRoleList(Object loginId, String loginType) {
        return Arrays.asList("admin", "super-admin");
    }
}
```

⚠️ 这两个方法在**每次鉴权时都会调用**，直接查库会有性能问题。生产环境需按 `[账号id → 角色id → 权限列表]` 的 RBAC 结构缓存，**不要**按 `[账号id → 权限列表]` 缓存（改一个角色的权限要失效十万个账号缓存，会引发缓存雪崩）。缓存代码见 `references/advanced.md`。

### 第二步：校验

```java
StpUtil.hasPermission("user.add");                          // → true/false
StpUtil.checkPermission("user.add");                        // 失败抛 NotPermissionException
StpUtil.checkPermissionAnd("user.add", "user.delete");      // 必须全部具备
StpUtil.checkPermissionOr("user.add", "user.delete");       // 具备其一即可

StpUtil.hasRole("admin");
StpUtil.checkRole("super-admin");                           // 失败抛 NotRoleException
StpUtil.checkRoleAnd("admin", "shop-admin");
StpUtil.checkRoleOr("admin", "shop-admin");
```

通配符：拥有 `art.*` 时 `art.add`/`art.update` 均通过；拥有 `*.delete` 时 `art.delete`/`user.delete` 均通过；拥有 `"*"` 即**上帝权限**，任何权限码都通过（角色同理）。

## 注解鉴权（⚠️ 必须先注册拦截器）

**为了不带来不必要的性能负担，Sa-Token 的全局拦截器默认关闭。** 不注册就用注解 = 注解完全不生效，这是最高频的坑。

```java
@Configuration
public class SaTokenConfigure implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 注册 Sa-Token 拦截器，打开注解式鉴权功能
        registry.addInterceptor(new SaInterceptor())
                .addPathPatterns("/**")
                .excludePathPatterns("/error");   // 建议排除 /error，见下文说明
    }
}
```

⚠️ 必须 `implements WebMvcConfigurer`，**不要** `extends WebMvcConfigurationSupport` 或加 `@EnableWebMvc`（常见于为 Swagger 加 `addResourceHandlers` 时），否则拦截器会失效、且 `SpringMVCUtil.getRequest()` 会报"非Web上下文"。

可用注解：

```java
@SaCheckLogin                                   // 登录校验
@SaCheckRole("admin")                           // 角色校验
@SaCheckPermission("user:add")                  // 权限校验
@SaCheckPermission(value = {"a","b"}, mode = SaMode.OR)   // OR 模式（默认 AND）
@SaCheckPermission(value = "user.add", orRole = "admin")  // 权限不足时角色兜底
@SaCheckSafe                                    // 二级认证校验
@SaCheckSafe("art")                             // 指定业务标识
@SaCheckDisable("comment")                      // 账号服务封禁校验
@SaCheckHttpBasic(account = "sa:123456")        // HttpBasic 校验
@SaCheckHttpDigest("sa:123456")                 // HttpDigest 校验
@SaCheckSign                                    // API 参数签名校验
@SaIgnore                                       // 忽略校验（优先级最高）
@SaCheckOr(login = @SaCheckLogin, role = @SaCheckRole("admin"))  // OR 组合
```

要点：
- 所有注解都可以**加在类上**，等效于加在该类所有方法上。
- **多个注解并存是 AND 关系**（所以没有 `@SaCheckAnd`）；需要 OR 用 `@SaCheckOr`。
- `@SaIgnore` 优先级最高，与其他鉴权注解共存时其他注解全部失效；它同时也能跳过 `SaInterceptor` 的路由鉴权，但**对自定义拦截器/过滤器无效**。
- `orRole` 的字符串切分语义有坑：`orRole = {"admin","manager"}` 是"任一即可"；`orRole = {"admin, manager"}`（单字符串带逗号）是"必须同时具备"。

**拦截器模式只能在 Controller 层用注解。** 想在 Service 等任意层级用注解，改用 AOP 模式：引入 `sa-token-spring-aop`。⚠️ **拦截器模式和 AOP 模式不可同时集成**，否则 Controller 层的注解会被校验两次。

## 路由拦截鉴权

适合按模块批量鉴权（粒度最粗）。注解是方法级，代码校验最细，三者按需搭配。

```java
@Configuration
public class SaTokenConfigure implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new SaInterceptor(handler -> {
            // 登录校验：拦截所有路由，排除登录接口
            SaRouter.match("/**").notMatch("/user/doLogin").check(r -> StpUtil.checkLogin());

            // 按模块分别鉴权
            SaRouter.match("/user/**", r -> StpUtil.checkPermission("user"));
            SaRouter.match("/admin/**", r -> StpUtil.checkRoleOr("admin", "super-admin"));
        })).addPathPatterns("/**");
    }
}
```

`SaRouter` 支持链式自由匹配（`SaRouterUtil` 是旧名，已重命名为 `SaRouter`）：

```java
SaRouter.match("/user/**", "/goods/**", "/art/get/{id}")  // 多路径 + restful
        .notMatch("*.html", "*.css", "*.js")               // 排除
        .match(SaHttpMethod.GET)                           // 按请求类型
        .match(r -> StpUtil.isLogin())                     // 按 boolean 条件
        .check(r -> StpUtil.checkLogin());                 // 全部匹配才执行
```

- `.stop()` —— 停止后续匹配，**继续进入 Controller**。
- `.back("内容")` —— 停止匹配，**直接返回给前端**，不进 Controller。
- `.free(r -> {...})` —— 独立作用域，内部 `stop()` 只跳出 free 块。
- `new SaInterceptor(...).isAnnotation(false)` —— 只做路由鉴权，关闭注解鉴权。
- 注解鉴权**总是先于**路由鉴权执行；要调整顺序就注册两个拦截器。

**动态鉴权规则必须写在 lambda 内部**，写在外面的代码只在启动时执行一次，永远不会动态：

```java
// ✅ 正确
new SaInterceptor(handle -> SaRouter.match("/**").notMatch(excludePaths()).check(r -> StpUtil.checkLogin()))
// ❌ 错误 —— excludePaths() 只在启动时求值一次
registry.addInterceptor(...).excludePathPatterns(excludePaths())
```

## 全局异常处理（必做）

鉴权失败靠抛异常表达，不接就是 500。

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotLoginException.class)
    public SaResult handlerNotLogin(NotLoginException e) {
        // e.getType() 可细分场景，见下表
        return SaResult.error("未登录：" + e.getMessage()).setCode(401);
    }

    @ExceptionHandler(NotRoleException.class)
    public SaResult handlerNotRole(NotRoleException e) {
        return SaResult.error("缺少角色：" + e.getRole()).setCode(403);
    }

    @ExceptionHandler(NotPermissionException.class)
    public SaResult handlerNotPermission(NotPermissionException e) {
        return SaResult.error("缺少权限：" + e.getPermission()).setCode(403);
    }
}
```

`NotLoginException.getType()` 的七种场景值 —— 前端据此区分"重新登录"还是"提示被踢"：

| 值 | 常量 | 含义 |
|---|---|---|
| `-1` | `NOT_TOKEN` | 未能从请求中读取到有效 token |
| `-2` | `INVALID_TOKEN` | 读取到 token 但无效 |
| `-3` | `TOKEN_TIMEOUT` | token 已过期 |
| `-4` | `BE_REPLACED` | token 已被顶下线 |
| `-5` | `KICK_OUT` | token 已被踢下线 |
| `-6` | `TOKEN_FREEZE` | token 已被冻结（active-timeout） |
| `-7` | `NO_PREFIX` | 未按照指定前缀提交 token |

⚠️ **`timeout` 到期时实际抛的是 `-2` 而不是 `-3`** —— token 已从 Redis 删除，框架无法区分"过期"和"从未存在"。只有集成 `sa-token-jwt` 时才能区分并抛 `-3`。

所有 Sa-Token 异常都继承 `SaTokenException`，都可通过 `e.getCode()` 拿到细分状态码，便于统一按码处理。

## Session 与三大作用域

三种 Session 都返回 `SaSession` 对象：

| 类型 | 键 | 获取方式 | 用途 |
|---|---|---|---|
| **Account-Session** | 账号id | `StpUtil.getSession()` | 同一账号 PC/APP 共享，跨端同步数据 |
| **Token-Session** | token | `StpUtil.getTokenSession()` | 每个 token 独立，各端隔离数据 |
| **Custom-Session** | 自定义 | `SaSessionCustomUtil.getSessionById("goods-10001")` | 给任意对象挂 Session |

```java
StpUtil.getSession().set("user", user);
User user = StpUtil.getSession().getModel("user", User.class);
StpUtil.getSession().get("name", () -> queryFromDb());  // 无值时执行并缓存
```

⚠️ **`SaSession` ≠ `HttpSession`**，两者数据不互通。⚠️ `getSession()` 要求已登录；未登录场景用 `StpUtil.getAnonTokenSession()` 或配置 `token-session-check-login: false`。

三大作用域：`SaHolder.getStorage()`（请求级，无需登录）、`StpUtil.getSession()`（会话级，需登录）、`SaHolder.getApplication()`（全局级）。

## timeout 与 active-timeout

- **`timeout`** —— 绝对有效期，到期 token 被删除。续期用 `StpUtil.renewTimeout(100)`。
- **`active-timeout`** —— 最低活跃频率。如 `1800` 表示 30 分钟无操作就冻结；有操作则自动续签。冻结的 token 不删除，只是冻结。

`active-timeout > 0` 时，每次直接/间接调用 `getLoginId()`、`getTokenSession()` 都会触发冻结检查+自动续签（`autoRenew: false` 可关闭）。手动续签：

```java
StpUtil.checkActiveTimeout();      // 先检查是否已冻结（否则会把冻结的 token 解冻）
StpUtil.updateLastActiveToNow();   // 再续签
```

⚠️ `active-timeout` **不是靠 Redis TTL 实现的**（存的是最后访问时间戳，判断 `now - 时间戳 > active-timeout`）。所以看到 Redis TTL 续签时没变化是正常的，不是 bug。

## 前后端分离

```yaml
sa-token:
  is-read-cookie: false   # 关闭 Cookie 读写，同时也能防 CSRF
```

后端登录时返回 `StpUtil.getTokenInfo()`（含 `tokenName` 和 `tokenValue`），前端存 `localStorage`，之后每个请求放到 header：

```js
// 登录后保存
localStorage.setItem('tokenName', res.data.tokenName)
localStorage.setItem('tokenValue', res.data.tokenValue)

// 请求拦截器里统一注入（推荐动态取 tokenName，比硬编码 'satoken' 灵活）
config.headers[localStorage.getItem('tokenName')] = localStorage.getItem('tokenValue')
```

token 读取顺序是 **body → header → cookie**。若前端提交了 header 但后端读到的是旧 Cookie，检查 `is-read-header` 和 `token-name` 是否匹配。

⚠️ 常见坑：
- **Nginx 默认丢弃含下划线的 header**，所以 `token-name` 不要用 `shop_token` 这种名字。
- 前端发 `Bearer xxx` 时需配 `token-prefix: Bearer`，且**前缀与 token 值之间必须恰好一个空格**。配了前缀后 Cookie 模式会失效（Cookie 不能存空格），需 `cookie-auto-fill-prefix: true`。
- 跨域问题常伪装成鉴权问题。CORS 处理见 `references/advanced.md`。

## Redis 集成（生产必备）

不集成 Redis 时数据在内存里，**重启即丢失所有登录状态**。

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-redis-template</artifactId>
    <version>1.45.0</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

引入依赖后**还必须配置 Redis 连接**，且上层 API 完全不变：

```yaml
spring:
  redis:          # ⚠️ SpringBoot 3.x 要改成 spring.data.redis
    database: 1
    host: 127.0.0.1
    port: 6379
    timeout: 10s
```

⚠️ Redis 集成包版本要与 starter 版本一致。⚠️ 切换序列化方案（JDK ↔ Jackson）必须先清空 Redis 历史数据，否则报 `SerializationException`。

排查"Redis 里明明有值但 token 无效"：**九成是在看另一个 Redis**。用 `SaManager.getSaTokenDao().set("name","value",100000)` 验证，**不要**用自己的 `RedisUtil` 测。

## 参考文档索引

需要以下能力时，读对应的 reference 文件：

| 文件 | 内容 |
|---|---|
| `references/config-reference.md` | 全部配置项及默认值、Cookie/sign/api-key 子配置、代码式配置两种模式 |
| `references/api-reference.md` | `StpUtil` 完整方法目录、`SaSession`、`SaTokenDao`、`SaStrategy`、`SaLoginParameter`/`SaLogoutParameter` 全量参数 |
| `references/advanced.md` | 踢人下线/同端互斥、二级认证、账号封禁、身份切换、会话查询、全局侦听器、全局过滤器、多账号认证、自定义注解、权限缓存、CORS、防火墙、异步/Mock 上下文、加密工具 |
| `references/plugins.md` | JWT 三种模式（含各模式功能限制矩阵）、临时 token、alone-redis、API 签名、API Key、Thymeleaf/Freemarker、SpEL 注解、序列化扩展 |
| `references/microservice.md` | 网关统一鉴权完整代码、分布式 Session、Same-Token 内网隔离、Dubbo/gRPC 集成、依赖引入规则 |
| `references/sso-oauth2.md` | SSO 三种模式选型与搭建、单点注销、OAuth2 Server 搭建、授权码/密码/客户端模式、`SaOAuth2DataLoader`、端点清单 |
| `references/troubleshooting.md` | **报错速查表** —— 按错误信息定位原因与修复方案 |
| `references/data-structure.md` | Redis key 布局，调试时用来直接看框架写了什么 |

## 遇到报错时

先查 `references/troubleshooting.md`，它按错误信息组织，覆盖了官方文档"常见问题排查"整章。最高频的几个：

- **注解不生效** → 没注册 `SaInterceptor`；或用了 `WebMvcConfigurationSupport`/`@EnableWebMvc`。
- **`SaTokenContext 上下文尚未初始化`** → 在异步/定时任务/RPC 里调了需要 Web 上下文的 API；或访问了不存在的路由（404 转发 `/error` 被二次拦截，`excludePathPatterns("/error")` 可解）。
- **`未能获取有效的上下文处理器`** → starter 选错（Boot 版本不匹配），或 WebFlux 下没注册 `SaReactorFilter`。
- **`未能读取到有效Token`** → 前端没提交/参数名不对/Nginx 丢下划线 header/跨域拦掉了。
- **明明排除了路径还是被拦截** → 跨域、404 转 `/error`、`context-path` 不该写进拦截路径、或多条 `SaRouter` 规则互相兜底拦下了。

## 代码约定

- 统一用 `SaResult` 包装响应：`SaResult.ok(msg)` / `SaResult.error(msg)` / `SaResult.data(obj)`。
- 权限码建议用固定分隔符（`user.add` 或 `user:add` 二选一并统一），通配符按该分隔符切分生效。
- 登录 id 用简单类型；用户对象存 Session，key 建议用框架常量 `SaSession.USER`。
- 密码哈希用 `SaSecureUtil.md5(salt + password)`；`RSA` 相关 API 与 `BCrypt` 已标记过时，新代码别用。
- 鉴权粒度：路由拦截（模块级）→ 注解（方法级）→ 代码校验（条件级），按需选择最粗的那个。
