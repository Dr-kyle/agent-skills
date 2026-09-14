# Sa-Token 微服务

## 依赖引入规则（官方说：大多数微服务 bug 报告都源于引错依赖）

### 硬性规则

> **微服务架构下，网关和内部服务必须分别引入 Sa-Token 依赖。不要在顶层父 pom 里引入。**

| 基础框架 / Web 模型 | artifactId |
|---|---|
| SpringMVC（Servlet 模型）—— 内部业务服务的常规情况 | `sa-token-spring-boot-starter` |
| Servlet 模型的网关（如 **Zuul**） | `sa-token-spring-boot-starter` |
| Reactor 模型的网关（**Spring Cloud Gateway**、**ShenYu** 等） | `sa-token-reactor-spring-boot-starter` **且必须注册全局过滤器** |

SpringBoot 版本对应：2.x → 上表原名；3.x → `sa-token-spring-boot3-starter` / `sa-token-reactor-spring-boot3-starter`；4.x → `-boot4-` 变体。

### 必须避开的冲突

- ⚠️ **切不可在一个项目里同时引入 Servlet starter 和 Reactor starter，会造成项目无法启动。**
- 推论：把任一 starter 放进共享父 pom，会把错误的 Web 模型泄漏给网关或业务服务之一。
- Zuul 理论上不支持 SpringBoot 3。

### Redis 在微服务拓扑下是必需的

> 各服务之间需要通过 Redis 同步数据。

网关和每个下游服务都要加：

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

## 网关统一鉴权（Spring Cloud Gateway）

### 第一步：在网关实现权限数据源

```java
@Component
public class StpInterfaceImpl implements StpInterface {
    @Override
    public List<String> getPermissionList(Object loginId, String loginType) {
        return ...;   // 返回此 loginId 拥有的权限列表
    }
    @Override
    public List<String> getRoleList(Object loginId, String loginType) {
        return ...;   // 返回此 loginId 拥有的角色列表
    }
}
```

数据来源三种推荐策略（任选）：
1. 网关集成 ORM，直接查库。
2. 先读 Redis，miss 时回落 ORM/DB。
3. 先读 Redis 缓存，miss 时 RPC 调专门的权限数据服务。

### 第二步：注册全局过滤器（完整可用代码）

```java
/**
 * [Sa-Token 权限认证] 配置类
 */
@Configuration
public class SaTokenConfigure {
    @Bean
    public SaReactorFilter getSaReactorFilter() {
        return new SaReactorFilter()
            // 拦截地址
            .addInclude("/**")
            // 开放地址
            .addExclude("/favicon.ico")
            // 鉴权方法：每次访问进入
            .setAuth(obj -> {
                // 登录校验 —— 拦截所有路由，并排除 /user/doLogin 用于开放登录
                SaRouter.match("/**", "/user/doLogin", r -> StpUtil.checkLogin());

                // 权限认证 —— 不同模块校验不同权限
                SaRouter.match("/user/**", r -> StpUtil.checkPermission("user"));
                SaRouter.match("/admin/**", r -> StpUtil.checkPermission("admin"));
                SaRouter.match("/goods/**", r -> StpUtil.checkPermission("goods"));
                SaRouter.match("/orders/**", r -> StpUtil.checkPermission("orders"));
            })
            // 异常处理方法：每次 setAuth 出现异常时进入
            .setError(e -> SaResult.error(e.getMessage()));
    }
}
```

要点：
- `SaRouter.match` 的**第二个参数是排除路径** —— `match("/**", "/user/doLogin", fn)` 意为"拦截全部但放行登录接口"。这是开放登录端点的规范写法；`/favicon.ico` 则在过滤器层面排除。
- 响应式环境**没有拦截器**，注解鉴权那套不适用，过滤器是唯一路径。
- `setError` 是必需的 —— 过滤器抛的异常进不了 `@ExceptionHandler`。

Kotlin 里需要显式 SAM 包装：`SaRouter.match("/test/test", SaFunction { StpUtil.checkLogin() })`。

## 分布式 Session

单机 Session 的问题：节点 1 上改的状态节点 2 看不到，用户在节点 1 登录后，下一个请求打到节点 2 就显示未登录。

四种方案与官方结论：

| 方案 | 结论 |
|---|---|
| 1. Session 同步（强制复制到所有节点） | 成本太高，不考虑 |
| 2. Session 粘滞（把用户固定到一个节点） | 网关层处理，与框架无关 |
| 3. **Session 中心（Redis）** | **推荐** —— Sa-Token 集成 Redis 极简单 |
| 4. 无状态 token（JWT），数据自描述在 token 里 | 复杂业务下功能受限 |

> 由于 jwt 模式不在服务端存储数据，对于比较复杂的业务可能会功能受限，因此更加推荐方案三。

只需引入 `sa-token-redis-template` + `commons-pool2` 并配置连接，上层 API 完全不变。

## Same-Token（内部服务外网隔离）

### 解决什么问题

子服务只允许通过网关访问。两种隔离风格：
1. **物理隔离** —— 服务在内网，只有网关公网可达。
2. **逻辑隔离** —— 服务和网关都公网可达，但服务带一层鉴权，只接受来自网关的请求，直连一律拒绝。

两个触点：**网关转发鉴权** 和 **服务间 RPC 调用鉴权**。

### API

```java
SaSameUtil.getToken();                      // 获取当前 Same-Token
SaSameUtil.isValid(token);                  // 判断是否有效
SaSameUtil.checkToken(token);               // 校验，无效则抛异常
SaSameUtil.checkCurrentRequestToken();      // 校验当前请求提供的 Same-Token
SaSameUtil.refreshToken();                  // 刷新（⚠️ 集群环境不要多个服务重复调用）
SaSameUtil.SAME_TOKEN;                      // 在 Request 上储存时建议使用的 key
```

包路径：`cn.dev33.satoken.same.SaSameUtil`。

### 网关侧：追加请求头

```java
/**
 * 全局过滤器，为请求添加 Same-Token
 */
@Component
public class ForwardAuthFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest newRequest = exchange.getRequest().mutate()
                .header(SaSameUtil.SAME_TOKEN, SaSameUtil.getToken())
                .build();
        ServerWebExchange newExchange = exchange.mutate().request(newRequest).build();
        return chain.filter(newExchange);
    }
}
```

注意这是普通的 Spring Cloud `GlobalFilter`，与鉴权用的 `SaReactorFilter` 是**两个独立的 Bean**。

### 下游服务侧：校验请求头

```java
@Configuration
public class SaTokenConfigure implements WebMvcConfigurer {
    @Bean
    public SaServletFilter getSaServletFilter() {
        return new SaServletFilter()
            .addInclude("/**").addExclude("/favicon.ico")
            .setAuth(obj -> {
                // 以下两句可简化为：SaSameUtil.checkCurrentRequestToken();
                String token = SaHolder.getRequest().getHeader(SaSameUtil.SAME_TOKEN);
                SaSameUtil.checkToken(token);
            })
            .setError(e -> SaResult.error(e.getMessage()));
    }
}
```

绕过网关直连会得到：`无效Same-Token：xxx`。

### 服务间调用（Feign）

```java
@Component
public class FeignInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate requestTemplate) {
        requestTemplate.header(SaSameUtil.SAME_TOKEN, SaSameUtil.getToken());

        // 如果希望被调用方有会话状态，还需把 satoken 也加到请求头
        // requestTemplate.header(StpUtil.getTokenName(), StpUtil.getTokenValue());
    }
}
```

```java
@FeignClient(
        name = "sp-home",
        configuration = FeignInterceptor.class,          // ⚠️ 关键代码
        fallbackFactory = SpCfgInterfaceFallback.class
)
public interface SpCfgInterface {
    @RequestMapping("/SpConfig/getConfig")
    String getConfig(@RequestParam("key") String key);
}
```

⚠️ 那行注释很重要：**Same-Token 只传递服务身份，不传递用户会话**。要同时携带用户登录态，必须额外转发 `StpUtil.getTokenName()` / `StpUtil.getTokenValue()`。

### 运维特性

- 与其它 Sa-Token 数据一起存 Redis，**默认有效期 1 天**，过期后下次获取自动重新生成。
- 刷新间隔越短越安全。
- ⚠️ **默认的自刷新机制不是高可用的** —— 多服务并发刷新会造成毫秒级服务失败，只适合开发环境或低并发。微服务下应主动刷新：

```java
/**
 * Same-Token 定时刷新
 */
@Configuration
public class SaSameTokenRefreshTask {
    // 从 0 分钟开始，每隔 5 分钟刷新一次
    @Scheduled(cron = "0 0/5 * * * ? ")
    public void refreshToken() {
        SaSameUtil.refreshToken();
    }
}
```

任何小于有效期（默认 1 天）的间隔都可以。

- **并发竞态的答案**：每次刷新时旧 token 会保留为**次级 token**，新旧任一都能通过校验，直到下次刷新再轮换。
- RPC 框架（Dubbo/gRPC）建议直接用配置开关 `sa-token.check-same-token: true`。

调试技巧：Redis 里 `{tokenName}:var:same-token` 和 `{tokenName}:var:past-same-token` 可直接确认轮换状态 —— 这通常是网关边界 `无效Same-Token` 的元凶。

## Dubbo 集成

解决两个问题：RPC 丢失**上下文环境**（被调方调任何 Sa-Token API 都抛 `无效上下文`）和丢失**上下文参数**。

依赖（**消费方和提供方都要引**）：`sa-token-dubbo`；**Dubbo 3 用 `sa-token-dubbo3`**。

收益：提供方可安全调用 Sa-Token API；登录态双向传播。

⚠️ **限制（必须绕开设计）**：两端**不共享 `SaStorage` 数据**；提供方侧的 `SaResponse.setHeader()` / `setStatus()` **是空操作**。

RPC 鉴权，**方式一（推荐）**：

```yaml
sa-token:
  check-same-token: true
```

方式二是手写 Dubbo 过滤器（可携带租户上下文等额外 attachment），注册在 `resources/META-INF/dubbo/org.apache.dubbo.rpc.Filter`：

- 消费方：`dubboConsumerFilter=com.pj.DubboConsumerFilter`，`@Activate(group = {CommonConstants.CONSUMER}, order = -10000)`，方法体 `RpcContext.getContext().setAttachment(SaSameUtil.SAME_TOKEN, SaSameUtil.getToken());`
- 提供方：`dubboProviderFilter=com.pj.DubboProviderFilter`，`@Activate(group = {CommonConstants.PROVIDER}, order = -10000)`，方法体 `SaSameUtil.checkToken(invocation.getAttachment(SaSameUtil.SAME_TOKEN));`

## gRPC 集成

依赖 `sa-token-grpc`（两端都引，**需要 Spring Boot 环境**）。收益与限制同 Dubbo（同样不共享 `SaStorage`，`SaResponse` 设置无效）。Same-Token 校验同样用 `sa-token.check-same-token: true` 开启。

## 响应式环境注意事项

- Reactor starter 单独引入是不够的 —— **必须注册全局过滤器**。
- 绝不与 Servlet starter 共存（项目起不来）。
- Redis 必需，网关与服务只通过它共享状态。
- 在 `WebFilter` 等地方调 Sa-Token **同步** API 前，要先 `SaReactorSyncHolder.setContext(exchange)`，并在 finally 里 `clearContext()`（详见 `advanced.md` 的异步/Mock 上下文一节）。
- SpringBoot 2.x 老版本在 WebFlux 下可能报 `NoSuchFieldError: defaultInstance`，升级到 ≥ 2.3.x，或重写 `routeMatcher`（见 `config-reference.md`）。
