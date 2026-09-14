# Sa-Token 报错速查

按错误信息定位。**先在这里找，再去改代码。**

## 注解完全不生效

1. **注解鉴权默认关闭** —— 必须注册 `SaInterceptor`，或引入 `sa-token-spring-aop`。这是第一大高频坑。
2. 同时配置了 `WebMvcConfigurer` **和** `WebMvcConfigurationSupport`（常见于为 Swagger 加 `addResourceHandlers`）会让拦截器失效。**二选一，永远选 `implements WebMvcConfigurer`。**

## `SaTokenContext 上下文尚未初始化`

1. 在异步/响应式上下文里调了同步 Sa-Token API → 见 `advanced.md` 的异步/Mock 上下文。
2. **访问了不存在的路由，而 `SaInterceptor` 里有鉴权代码。** 流程是：写入上下文 → 拦截器（通过）→ 检测到 404 → 清除上下文 → 转发 `/error` → 再次进拦截器（已无上下文）→ 报错。修复：

```java
registry.addInterceptor(new SaInterceptor(handle -> { /* 鉴权代码 */ }))
        .addPathPatterns("/**")
        .excludePathPatterns("/error");
```

3. SpringMVC 里返回 `SseEmitter` / `Flux`（流式响应）但没用 WebFlux。

## `非 web 上下文无法获取 HttpServletRequest`

在这些地方调了需要 Web 上下文的 API：`main()`、`@Async`、丢失上下文的子线程（如 MyBatis-Plus `insertFill`）、非 HTTP 的 RPC（Dubbo）、`@PostConstruct`、定时任务。

## `未能获取有效的上下文处理器`（InvalidContextException）

starter 选错或缺失。SpringBoot 2.x → `sa-token-spring-boot-starter`；3.x → `sa-token-spring-boot3-starter`；4.x → `-boot4-starter`；WebFlux → `sa-token-reactor-spring-boot-starter`（及 boot3/boot4 变体）。**WebFlux 下必须注册 `SaReactorFilter`。**

如果开了 `spring.main.lazy-initialization=true`：

```java
@Configuration
class MyConfiguration {
    @Bean
    LazyInitializationExcludeFilter integrationLazyInitExcludeFilter() {
        return LazyInitializationExcludeFilter.forBeanTypes(SaBeanInject.class);
    }
}
```

## NotLoginException 按消息排查

### 未能读取到有效Token

1. 前端没提交 token。
2. 参数名不对（默认 `satoken`，见 `sa-token.token-name`）。
3. 读取被关闭（`is-read-header=false`）。
4. token 前缀不对。
5. 前后端分离，浏览器不会自动提交 Cookie。
6. **Nginx 默认丢弃含下划线的 header 参数**（如 `shop_token`）。
7. 跨域把提交拦掉了。

### Token无效: \<uuid\>

1. 伪造的 / 跨项目的 token。
2. `timeout` 已过期。
3. 没接 Redis 且应用重启了。
4. Redis 重启了。
5. **提交的 token ≠ 框架读到的 token** —— 读取顺序是 `body → header → cookie`，如 `is-read-header=false` 或 `token-name` 不匹配，会穿透到读一个陈旧的 Cookie。
6. jwt Mixin/Stateless 模式下改了 `jwtSecretKey`。
7. 该账号登录数超过 `maxLoginCount`（默认 12），最早的被强制注销了。
8. `is-concurrent=true, is-share=true` 时别人注销了这个共享 token。
9. **多账号体系错配** —— `StpUserUtil.login()` 发的 token 用 `StpUtil.checkLogin()` 校验，永远无效。

### Token已过期

1. `active-timeout` 把它冻结了。
2. jwt Mixin/Stateless 模式且 `timeout` 到期。

### Token已被顶下线

1. `is-concurrent=false` 且别人登录了。
2. 有人调了 `StpUtil.replaced(loginId, device)`。

### Token已被踢下线

有人调了 `StpUtil.kickout(loginId)`。

## 拦截器"不拦截" / 注册了但进不去

用两个标记调试：注册处打印 `flag 1`，lambda 内部打印 `flag 2` 和 `SaHolder.getRequest().getRequestPath()`。

**没注册成功**：(1) `SaTokenConfigure` 不在启动类的包/子包下；(2) 启动类上的 `@ComponentScan("com.xxx")` 缩小了扫描范围；(3) 多模块 Maven 里配置类所在模块没被引入。

**注册了但没进入**：路径没被 `addPathPatterns` 匹配到，或被排除了；或者你访问的是另一个正在运行的实例。

## 明明排除了路径还是被拦截

1. **跨域。**
2. 404 转发到 `/error` 被二次拦截。
3. 是别的东西在拦 —— 注释掉这个拦截器看堆栈。
4. 后端实际路径 ≠ 前端路径（网关重写）。打印 `SaHolder.getRequest().getRequestPath()` 确认。
5. 浏览器自动发的其它请求（如 `/favicon.ico`）。
6. **`context-path` 不能出现在拦截路径里。** 配了 `server.servlet.context-path=/shop` 时，写 `.excludePathPatterns("/user/login")`，**不是** `"/shop/user/login"`。
7. 多条规则互相兜底 —— 被规则 1 放行的请求被规则 2 拦下了：

```java
SaRouter.match("/**").notMatch("/user/doLogin").check(r -> StpUtil.checkLogin());     // 规则1
SaRouter.match("/**").notMatch("/article/getList").check(r -> StpUtil.checkLogin());  // 规则2 ← 拦下了 /user/doLogin
```

8. 自定义的 `excludePaths()` 返回值和你以为的不一样 —— 先硬编码一个字面量列表隔离问题。

## 配置不生效

在 `SpringApplication.run(...)` 之后验证：

```java
System.out.println("\n启动成功：Sa-Token配置如下：" + SaManager.getConfig());
```

原因：代码式配置（`@Bean @Primary`）覆盖了 `application.yml`；文件名不对；前缀/缩进不对（**必须是顶级 `sa-token:`，不能嵌在 `spring:` 下**）。

## 自定义组件不生效

打印验证：`System.out.println(SaManager.getStpInterface());`

原因：缺 `@Component`；类不在启动类的包树下；`@ComponentScan` 缩小了范围。

另外：**`StpInterface` 是授权钩子，登录时不会触发**，只在权限/角色校验时才调用 —— 启动时或登录时没日志是正常的。

## Redis 里明明有值但 token 无效

**九成是在看另一个 Redis。** 用框架自己的 DAO 证明：

```java
SaManager.getSaTokenDao().set("name", "value", 100000);
```

⚠️ **不要**用自己的 `RedisUtil` 测。

## `无效Same-Token：xxx`（以及 `无效ticket`、`无效 Access-Token`、分布式 Session 的 `无效 Token`）

这几个都要求多个服务共用**同一个** Redis。在两个服务里都执行 `SaManager.getSaTokenDao().set(...)` 验证。若确实共用一个 Redis，再核对**所有** `sa-token` 配置是否跨服务一致 —— **单是 `token-name` 不同就会导致互相认证失败。**

## 30 天的 timeout 提前失效了

逐条排查：没接 Redis（内存实现，重启即丢）· Redis 重启了 · `is-concurrent=false` 且别人登录了 · `is-concurrent=true, is-share=false` 且超过 12 个（`maxLoginCount`）设备 · 别人调了注销（`StpUtil.logout()` 是单 token，`StpUtil.logout(10001)` 是该账号全部 token）· `active-timeout` 设得太短 · 浏览器/缓存变了 · **你改了 `sa-token.token-name`** —— 它是存储 key 前缀，改名等于清空 Redis。

## 序列化报错

- JDK 与 Jackson 序列化互切 → `SerializationException`。**必须先清空 Redis 历史数据。**
- 换 SpringBoot 版本后报 `SerializationException: Could not read JSON: Cannot deserialize value of type java.lang.Long from Array value` → Redis 里有陈旧的 Session 数据，清 Redis。
- `无法从 User 类型转换成 User 类型` → `User` 类换了包名，或者开了热重载（关掉）。

## `this api is disabled`（调 `StpUtil.getExtra("name")` 时）

`getExtra` 是 jwt 插件专属。普通模式存到 Session：

```java
StpUtil.getSession().set("name", "zhangsan");
String name = StpUtil.getSession().getString("name");
```

## `SaServletFilter` 里 `SpringMVCUtil.getRequest()` 报 `非Web上下文无法获取Request`

由 `extends WebMvcConfigurationSupport` 或 `@EnableWebMvc` 引起。改用 `implements WebMvcConfigurer`。若确实必须继承 `WebMvcConfigurationSupport`：

```java
@Bean
@ConditionalOnMissingBean({ RequestContextListener.class, RequestContextFilter.class })
@ConditionalOnMissingFilterBean(RequestContextFilter.class)
public static RequestContextFilter requestContextFilter() {
    return new OrderedRequestContextFilter();
}
```

## `未能获取对应StpLogic，type=xxx`（多账号注解鉴权）

1. 注解里的 `loginType` 拼写错了 —— 用常量（`StpUserUtil.TYPE`）而不是字面量。
2. 自定义的 `StpUtil` 静态类从未被初始化 —— 在 `main` 里调用一次，或给它加 `@Component`。

## `ClassNotFoundException: cn.dev33.satoken.same.SaSameTemplate`

版本错配。所有 `sa-token-*` 依赖必须**同一版本**。全局搜索 `"sa-token"` 检查每个 pom（不要靠肉眼扫）。

## `NoClassDefFoundError: PolymorphicTypeValidator`（SpringBoot < 2.2.0）

手动补 `jackson-core`、`jackson-annotations`、`jackson-databind`（如 `2.17.3`）。

## SpringBoot 3.x：`No more pattern data allowed after {*...} or ** pattern element`

SpringBoot 3.x 默认 `path_pattern_parser`，禁止 `**` 之后再有内容（如 `/admin/**/info`）。三个选择：避免在 `**` 后写内容；或切回 ant 匹配（**两处都要改**）：

```yml
spring:
  mvc:
    pathmatch:
      matching-strategy: ant_path_matcher
```

```java
@PostConstruct
public void customRouteMatcher() {
    SaStrategy.instance.routeMatcher = (pattern, path) ->
            SaPatternsRequestConditionHolder.match(pattern, path);
}
```

WebFlux / Gateway（2.x）下 `SaPatternsRequestConditionHolder` 会抛 `NoClassDefFoundError: PatternsRequestCondition`，改用：

```java
SaStrategy.instance.routeMatcher = (pattern, path) ->
        SaPathMatcherHolder.getPathMatcher().match(pattern, path);
```

## WebFlux / Gateway：`NoSuchFieldError: defaultInstance`

出现在 `SaPathPatternParserUtil.match`。SpringBoot 版本太老，升级到 ≥ `2.3.x`，或用上面的 `SaPathMatcherHolder` 方案重写 `routeMatcher`。

## 404 时堆栈噪音很大

```properties
spring.web.resources.add-mappings=false
spring.mvc.throw-exception-if-no-handler-found=true
```

## Swagger 被过滤器/拦截器挡住

排除这些路径：

```java
"/swagger-resources/**", "/webjars/**", "/v2/**", "/swagger-ui.html/**",
"/doc.html/**", "/error", "/favicon.ico"
```

注解鉴权不需要排除（Swagger 不用 Sa-Token 注解）。

## SSO 相关

### `{"msg":"not handle"}`

路由不对，或该路由未启用（`is-slo=true` 才会注册 signout/logout 路由）。

### `Ticket 无效`

- 模式二：客户端和服务端并非真的连着同一个 Redis。不要肉眼看 YAML，在两端启动时执行 `SaManager.getSaTokenDao().set("name","value",100000)` 确认（**注意 database 索引也要相同**）。
- 模式三：同一个 ticket 校验了两次（ticket 是一次性的）。

### `No serializer found for class com.pj.sso.SysUser`

服务端写进 Session 的实体在客户端不存在。把这个类拷过去，**包名必须完全一致**。

### 模式一排查

服务端已登录但客户端显示未登录 → `sa-token.cookie.domain` 没配。登录页不跳回 → `cookie.domain` 配错，或 `sa-token.is-read-cookie` 关了。页面一直闪 → 测模式二/三时留下的 Cookie/Redis 残留，清 Redis、清 Cookie，或换个浏览器。

⚠️ 测模式二/三前**必须把 `sa-token.cookie.domain` 注释掉**，它会破坏那两种模式的流程。

### 模式二/三下第二个客户端还要点一次登录按钮

这是**正常的**（只是跳过了输密码环节）。要做到零点击，注册一个 `SaServletFilter`：

```java
.addInclude("/**").addExclude("/sso/*", "/favicon.ico")
```

⚠️ **必须排除 `/sso/*`，否则会无限重定向。**

## OAuth2 相关

- `{"msg":"not handle"}` → 路由路径不对。
- `code 无效` → 一个 code 被用了多次（一次性）。
- 多账号 → 在 `configOAuth2Server` 里 `SaOAuth2Manager.setStpLogic(...)`。

## 其它常见答案

- **过滤器/拦截器顺序**：Sa-Token 全局过滤器默认 order `-100`；用 `@Order(xxx)` 或 `FilterRegistrationBean.setOrder(-101)` 覆盖。
- **`SaInterceptor` 里注解鉴权总是先于路由鉴权。** 要换顺序就注册两个拦截器：先 `new SaInterceptor(r -> {...}).isAnnotation(false)` 做路由，再 `new SaInterceptor()` 做注解。
- **登录前拿 Session**：`StpUtil.getSession()` 要求已登录。用 `token-session-check-login=false` + `StpUtil.getTokenSession()`，或 `StpUtil.getAnonTokenSession()`。
- **非并发登录后 Redis 里残留旧 token 是故意的**（为了能告知"被顶下线"），会随 TTL 过期。或者设 `is-concurrent=true, is-share=true`，或登录前先 `StpUtil.logout(10001)`。
- **多个项目共用一个 Redis**：用不同 database 索引、不同 `sa-token.token-name`，或 `sa-token-three-redis-jackson-add-prefix` 插件。
- **Redis 集群**：只需把 `RedisTemplate` 配成集群，`sa-token-redis-template` 自动跟随。
- **改密码后强制下线**：`StpUtil.logout()`。
- **关闭启动字符画**：`is-print: false`。**关闭 Cookie 操作**：`is-read-cookie: false`。
- **鉴权粒度选择**：路由拦截最粗（模块级）；注解方法级；代码校验最细（条件级）。

## 从 Shiro / Spring Security / JWT 迁移

| 关注点 | Shiro | Spring Security | Sa-Token |
|---|---|---|---|
| 登录 | `subject.login(...)` + 自定义 `Realm` + 一堆 Bean | `SecurityFilterChain` 配置 | `StpUtil.login(user.getId())` —— 密码你自己校验，**框架完全不拦截登录流程** |
| 注销 | `SecurityUtils.getSubject().logout()` | `httpSecurity.logout(...)` | `StpUtil.logout()` |
| 当前用户 | `subject.getPrincipal()` | `SecurityContextHolder...` | `StpUtil.getLoginId()` + `StpUtil.getSession().get("user")` |
| Session | `subject.getSession()` | `request.getSession()` | `StpUtil.getSession()` |
| 权限数据源 | `Realm#doGetAuthorizationInfo` | `UserDetailsService` | `implements StpInterface` |
| 注解 | `@RequiresRoles("admin")` | `@PreAuthorize("hasRole('admin')")` | `@SaCheckRole("admin")` —— **需注册 `SaInterceptor`** |
| 路由鉴权 | `filterChainDefinitionMap` | `authorizeHttpRequests(...)` | `SaInterceptor` 里用 `SaRouter.match(...).check(...)` |
| 失败处理 | 重定向到 `/401` `/403` | `AccessDeniedHandler` | `@RestControllerAdvice` 接 `NotLoginException` 等 |
| Redis | `shiro-redis` + 配置 Bean | 手写 | 加 `sa-token-redis-template`，**其余代码零改动** |

迁移要点：
- **Sa-Token 最大的结构性差异是它从不拥有登录流程** —— 没有 `Realm`、没有 `UserDetailsService`、没有过滤器链 DSL。你校验凭证，然后 `StpUtil.login(id)` 通知框架。
- Shiro 的 `session.getAttribute` 与 `SaSession.get/set` 一一对应，但 Sa-Token 的 Session 默认按 loginId 划分（Account-Session）而非按客户端 —— 需要按端隔离时切到 Token-Session。
- 从 JWT 迁过来会**首次获得** kickout/replaced/冻结语义，这也意味着 `NotLoginException` 的场景值开始有意义了。

示例仓库：`https://gitee.com/sa-tokens/auth-framework-function-test`
