# Sa-Token SSO 与 OAuth2

## 先做选型

| 能力 | SSO | OAuth2 |
|---|---|---|
| 统一认证 | 高 | 高 |
| 统一注销（单点注销） | **高** | **低** |
| 跨系统会话一致性 | 强 | 弱 |
| 第三方应用授权管理 | **不支持** | **高** |
| 自有系统授权管理 | 高 | 低 |
| client 级别权限校验 | **不支持** | **高** |
| 集成难度 | 简单 | 中等 |
| 适用 | **内部/企业系统整合** | **对外开放平台** |

**判断规则**：自己拥有的内部系统 → SSO。要把 API 开放给第三方公司（他们拿 `client_id`/`client_secret`、scope、`access_token`）→ OAuth2。

---

# 一、单点登录 SSO

## 依赖

server 和 client **用同一个 artifact**，没有单独的 sso-server/sso-client 包：

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-spring-boot-starter</artifactId>
    <version>1.45.0</version>
</dependency>
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-sso</artifactId>
    <version>1.45.0</version>
</dependency>
```

按模式追加：
- `sa-token-redis-template` + `commons-pool2` —— 模式一、二必需，模式三可选。
- `sa-token-alone-redis` —— 模式一、二时装在**客户端**，让 Sa-Token 认证缓存指向 SSO 的 Redis，业务缓存仍用本地。推荐但非必需。**模式三要移除它。**
- `sa-token-forest`（或 `sa-token-okhttps`）—— HTTP 请求处理器，模式三的客户端必需（服务端做单点注销推送时也要）。两个插件都是 v1.43.0 新增。

## 三种模式选型

| 模式 | 前提 | 机制 | 关键配置 |
|---|---|---|---|
| **模式一** | 前端共享父域名（`s1.stp.com`、`s2.stp.com`）**且**同一 Redis | 共享 Cookie → 共享 Session | **服务端**配 `sa-token.cookie.domain=stp.com` |
| **模式二** | 前端**不同**域名，**同一** Redis | URL 重定向 + ticket，客户端直接从 Redis 读 | 客户端 `alone-redis` 指向服务端 Redis（**database 索引也要相同**） |
| **模式三** | 不同域名**且**不同 Redis | 客户端通过 HTTP 调服务端校验 ticket | `sa-token.sso-client.is-http=true` + `secret-key` |

限制：
- 模式一：所有子系统必须在同一父域名下。测模式二/三前**必须把 `cookie.domain` 注释掉**，否则会破坏流程。
- 模式二：server 与 client 必须连**完全相同**的 Redis 和 database 索引。且服务端写入 Session 的实体类必须在客户端存在，**包名完全一致**，否则反序列化失败。
- 模式三：客户端有自己独立的会话，单点注销依赖消息推送（HTTP）机制，需要 HTTP 插件。

⚠️ 模式二、三下，在客户端 1 登录后，客户端 2、3 **仍需点一次登录按钮**（只是跳过了输凭证）。这是正常的；要零点击需注册过滤器（见 troubleshooting）。

## SSO-Server 搭建（当前 API）

> **版本提示（重要）**：当前 API 是 `SaSsoServerProcessor.instance.dister()`，策略配在 **`SaSsoServerTemplate.strategy`** 上。历史变更：
> - v1.32.0：`SaSsoHandle`（静态）→ `SaSsoProcessor`（实例），**不向下兼容**；服务端单点注销路径 `/sso/logout` → **`/sso/signout`**（避免与客户端的 `/sso/logout` 冲突），**不向下兼容**。
> - **v1.38.0**：整个 `sa-token-sso` 模块拆成服务端/客户端两侧（`SaSsoServerProcessor`/`SaSsoClientProcessor`、`SaSsoServerConfig`/`SaSsoClientConfig`），**不向下兼容**。旧的单一 `SaSsoConfig` 已不存在。
> - v1.35.0：`sa-token.sso.secretkey` → `sa-token.sign.secret-key`；`SaSsoUtil.checkSign(req)` → `SaSignUtil.checkRequest(req)`，均**不向下兼容**。
> - **v1.43.0**：新增 `SaSsoServerTemplate.strategy` / `SaSsoClientTemplate.strategy` 策略类、`clients` map 配置、`allowAnonClient`、消息推送机制、单设备注销。以下示例基于 ≥ v1.43.0。

```java
@RestController
public class SsoServerController {

    /*  /sso/auth     -- SSO 授权入口
     *  /sso/doLogin  -- RestAPI 登录（参数：name、pwd）
     *  /sso/signout  -- 单点注销（仅 isSlo=true 时开放）
     *  /sso/pushS    -- 消息推送接收端点  */
    @RequestMapping("/sso/*")
    public Object ssoRequest() {
        return SaSsoServerProcessor.instance.dister();
    }

    @Autowired
    private void configSso(SaSsoServerTemplate ssoServerTemplate) {
        // 未登录时返回的视图
        ssoServerTemplate.strategy.notLoginView = () -> "...html 或 ModelAndView...";
        // 登录处理函数
        ssoServerTemplate.strategy.doLoginHandle = (name, pwd) -> {
            if("sa".equals(name) && "123456".equals(pwd)) {
                StpUtil.login(10001);
                return SaResult.ok("登录成功！").setData(StpUtil.getTokenValue());
            }
            return SaResult.error("登录失败！");
        };
    }
}
```

`name`/`pwd` 之外的额外参数用 `SaHolder.getRequest().getParam("xxx")` 取。务必加 `@RestControllerAdvice` 全局异常处理。

配置：

```yml
sa-token:
  # cookie: { domain: stp.com }   # 仅模式一需要，模式二/三务必注释掉
  sso-server:
    ticket-timeout: 300        # ticket 有效期，默认 300 秒
    home-route: /home          # /sso/auth 无 redirect 参数时的落地页
    is-slo: true               # 默认 true，控制 /sso/signout 是否注册
    auto-renew-timeout: false  # 每次签发 ticket 时是否续期 token
    max-reg-client: 32         # Access-Session 上记录的最大 client 数，-1=不限，FIFO 淘汰
    is-check-sign: true        # ⚠️ 生产环境必须保持 true
    allow-anon-client: false   # 是否允许无 client 标识的客户端
    secret-key: ""             # 全局默认秘钥
    clients:
      sso-client1: { client: sso-client1, allow-url: "http://sa-sso-client1.com:9003/sso/login" }
      sso-client3:
        client: sso-client3
        allow-url: "http://sa-sso-client3.com:9003/sso/login"
        is-push: true                                        # 接收消息推送
        push-url: http://sa-sso-client3.com:9003/sso/pushC   # 默认 /sso/pushC
        secret-key: SSO-C3-xxx                               # 未配则回落全局
        is-slo: true
```

`SaSsoClientModel` 字段：`client`、`allowUrl`、`isPush`(false)、`isSlo`(true)、`secretKey`、`serverUrl`、`pushUrl`(`/sso/pushC`)。

⚠️ **安全红线：`allow-url: "*"` 会导致 ticket 劫持** —— 攻击者构造 `/sso/auth?client=x&redirect=https://evil.com/` 就能从 URL 里收割 ticket。安全等级：`*` 生产环境禁用；域名级（`http://host/*`）不推荐；**精确 URL 才安全**。另外 `*` 只能出现在字符串末尾，且 `@` 字符被禁止（均为 v1.38.0 的安全修复）。

客户端信息存数据库：继承 `SaSsoServerTemplate` 并加 `@Component`，重写 `getClient(String client)` / `getClients()` 返回 `SaSsoClientModel`。

## SSO-Client 搭建

```java
@RestController
public class SsoClientController {
    /*  /sso/login   -- 客户端登录入口（参数：back、ticket）
     *  /sso/logout  -- 客户端注销（isSlo=true）
     *  /sso/pushC   -- 消息推送接收  */
    @RequestMapping("/sso/*")
    public Object ssoRequest() {
        return SaSsoClientProcessor.instance.dister();
    }

    // 仅注销本系统
    @RequestMapping("/sso/logoutByAlone")
    public Object logoutByAlone() {
        StpUtil.logout();
        return SaSsoClientProcessor.instance._ssoLogoutBack(SaHolder.getRequest(), SaHolder.getResponse());
    }
}
```

前端链接：`/sso/login?back=/`、`/sso/logoutByAlone?back=/`、`/sso/logout?back=self`（全系统注销）。

```yml
sa-token:
  sso-client:
    client: sso-client2                            # 本客户端标识
    server-url: http://sa-sso-server.com:9000      # auth/signout/push/getData 的 url 前缀
    is-http: false        # true = 模式三（HTTP 校验 ticket / 注销 / 取数据）
    is-slo: true
    secret-key: SSO-C2-xxx
    is-check-sign: true
    reg-logout-call: false  # 登录时注册 /sso/logoutCall（匿名 client 参与 SLO 时必需）
    # auth-url: /sso/auth      signout-url: /sso/signout
    # push-url: /sso/pushS     get-data-url: /sso/getData
  # 仅模式一、二：
  alone-redis: { database: 1, host: 127.0.0.1, port: 6379, timeout: 10s }
```

`server-url` 会拼到 `auth-url`、`signout-url`、`push-url`、`get-data-url` 前面；若某个子 url 以 `http` 开头则以它为准、忽略 `server-url`。

模式一的客户端首页用 `SaSsoManager.getClientConfig().splicingAuthUrl()` + `?mode=simple&client=...&redirect=...` 构造授权链接（`mode=simple` 表示登录后直接跳回、不走 ticket）。

模式三相对模式二的差异：去掉 `sa-token-alone-redis`、设 `is-http: true`、设 `secret-key`、`spring.redis` 指向**不同的** Redis database。

模式三跨域取数据（userinfo 等）：客户端调 `SaSsoUtil.getData(map)` 或 `SaSsoUtil.getData("/sso/getFansList", map)`；服务端暴露端点并用 `SaSignUtil.checkRequest(SaHolder.getRequest())` 校验。若各 client 有独立秘钥，改用 `SaSsoServerProcessor.instance.ssoServerTemplate.getSignTemplate(client).checkRequest(req)`（client 从请求头读）。v1.34.0 之前的 `getUserinfo` url 配置方式已废弃。

## 单点注销

三种范围：**单系统**（`/sso/logoutByAlone`）、**全系统 / SLO**（`/sso/logout`）、**单浏览器**（`/sso/logout?singleDeviceIdLogout=true`）。

全注销链路（框架内置，你只在前端调用）：
1. 客户端前端 → 客户端后端 `/sso/logout`
2. 客户端 → 服务端 `/sso/pushS?msgType=signout`
3. 服务端遍历 client 列表，推送 `/sso/pushC?msgType=logoutCall`
4. 服务端注销；5–7 响应逐级回传。

必需配置：`sa-token.sso-server.is-slo=true` 和 `sa-token.sso-client.is-slo=true`。⚠️ **不配这两个，`/sso/signout` 和 `/sso/logout` 路由根本不会注册，会得到 `{"msg":"not handle"}`。** 模式三还需两侧都装 HTTP 插件且 `secret-key` 一致。客户端必须在服务端 `clients` map 里配 `is-push: true` + `push-url` 才会被通知。匿名 client 需 `reg-logout-call: true`。

单浏览器注销要按 device id 分组登录，服务端：

```java
String deviceId = SaHolder.getRequest().getParam("deviceId", SaFoxUtil.getRandomString(32));
StpUtil.login(10001, new SaLoginParameter().setDeviceId(deviceId));
```

然后客户端传 `singleDeviceIdLogout=true`（`SaLogoutParameter.deviceId` 为 v1.43.0 新增）。测试提示：同一浏览器的两个窗口共享会话，要用第二个浏览器或隐私模式测。

Ajax 注销也支持（不传 `back` 参数，需提交会话 token），返回 `{code:200,msg:"单点注销成功"}`。

## 前后端分离（h5）

思路：把 `/sso/login` 的重定向中转逻辑从后端搬到前端。客户端后端额外暴露三个端点：

```java
@RequestMapping("/sso/isLogin")
public Object isLogin() {
    return SaResult.data(StpUtil.isLogin()).set("loginId", StpUtil.getLoginIdDefaultNull());
}

@RequestMapping("/sso/getSsoAuthUrl")
public SaResult getSsoAuthUrl(String clientLoginUrl) {
    return SaResult.data(SaSsoClientUtil.buildServerAuthUrl(clientLoginUrl, ""));
}

@RequestMapping("/sso/doLoginByTicket")
public SaResult doLoginByTicket(String ticket) {
    SaCheckTicketResult ctr = SaSsoClientProcessor.instance.checkTicket(ticket);
    StpUtil.login(ctr.loginId, new SaLoginParameter()
            .setTimeout(ctr.remainTokenTimeout)
            .setDeviceId(ctr.deviceId));
    return SaResult.data(StpUtil.getTokenValue());
}
```

再加一个 `SaCorsHandleFunction` Bean 处理跨域（Sa-Token 过滤器风格，非原生 CORS 过滤器 —— v1.41.0 起文档改用此法）。

前端需要一个中转页（`sso-login.html`）：URL 里有 `ticket` 就 POST 给 `/sso/doLoginByTicket`，把返回的 token 存 `localStorage.satoken` 后跳回 `back`；否则请求 `/sso/getSsoAuthUrl` 并 `location.href` 过去。之后每个请求带 `satoken` 头。

若**服务端**也是分离的：用 `sa-token.sso-client.auth-url` 指向前端授权页 —— ⚠️ **`server-url` 也要保留，两个都得有。**

参考 demo：`sa-token-demo-sso-client-h5` / `-vue2` / `-vue3`、`sa-token-demo-sso-server-h5`。

## SSO 端点清单（STS 协议）

**服务端**：`/sso/auth`（`redirect`、`mode`∈{simple,ticket} 默认 ticket、`client`）· `/sso/doLogin`（`name`、`pwd`）· `/sso/signout`（`back`，可为 `self`）· `/sso/pushS`（`client`、`timestamp`、`msgType`、`nonce`、`sign`）。
**客户端**：`/sso/login`（`back` 必填、`ticket`）· `/sso/logout`（`back`）· `/sso/logoutCall`（仅 `reg-logout-call=true` 时）· `/sso/pushC`。

签名算法：参数按字典序排序后，最后追加 `&key={secretKey}` 再取 md5。内置 msgType：服务端 `checkTicket`、`signout`；客户端 `logoutCall`。

Ticket 语义：每次授权唯一、**一次性使用**、默认 5 分钟；签发新 ticket 会立即作废旧的。

## 其它 SSO 提示

- 改造建议：**不要把现有系统改造成 SSO 服务端**，而是新建一个干净的 SSO-Server 项目，让所有旧系统都做客户端。
- server + client 在同一项目：拆分聚合路由避免冲突 —— `@RequestMapping({"/sso/auth","/sso/doLogin","/sso/signout","/sso/pushS"})` 走 server dister；`@RequestMapping({"/sso/login","/sso/logout","/sso/logoutCall","/sso/pushC"})` 走 client dister。
- 判断遗留项目用的是哪种模式：有 `sa-token.cookie.domain` → 模式一；`sso-client.is-http=true` → 模式三；否则模式二。⚠️ `sso-client.mode` / `sso-server.mode` 只是纯文档标记，**没有任何逻辑作用**。
- 非 Java / 无 SDK 客户端：NoSdk 模式（直接调 STS 端点）或 ReSdk 模式（重写少量 SDK 钩子）。demo：`sa-token-demo-sso3-client-nosdk`、`-resdk`。

---

# 二、OAuth2

## 依赖

```xml
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-oauth2</artifactId>
    <version>1.45.0</version>
</dependency>
```

外加 `sa-token-spring-boot-starter`；`sa-token-redis-template` 可选（仅用于查看底层数据）。

## 支持的授权模式及开关

| 模式 | `response_type` / `grant_type` | 全局开关 | `SaClientModel` 授权类型 |
|---|---|---|---|
| 授权码 | `response_type=code` → `grant_type=authorization_code` | `enable-authorization-code`（默认 true） | `GrantType.authorization_code` |
| 隐式 | `response_type=token` | `enable-implicit`（true） | `GrantType.implicit` |
| 密码 | `grant_type=password` | `enable-password`（true） | `GrantType.password` |
| 客户端 | `grant_type=client_credentials` | `enable-client-credentials`（true） | `GrantType.client_credentials` |
| 刷新 | `grant_type=refresh_token` | （无全局开关） | `GrantType.refresh_token` |

⚠️ **两层都要放行**：全局 `enable-*` 开关 **且** 该 client 的 `allowGrantTypes`。

```yml
sa-token:
  token-name: sa-token-oauth2-server
  oauth2-server:
    enable-authorization-code: true
    enable-implicit: true
    enable-password: true
    enable-client-credentials: true
    code-timeout: 300              # 5 分钟
    access-token-timeout: 7200     # 2 小时
    refresh-token-timeout: 2592000 # 30 天
    client-token-timeout: 7200
    max-access-token-count: 12     # 每 client 每用户
    max-refresh-token-count: 12
    max-client-token-count: 12
    is-new-refresh: false          # 刷新时是否轮换 refresh token
    higher-scope: ""               # 每次都需手动确认授权的 scope
    lower-scope: ""                # 总是静默通过的 scope
    mode4-return-access-token: false  # client_credentials 的规范兼容
    hide-status-field: false          # 隐藏 code/msg/data 包装
    oidc: { iss: xxx, idTokenTimeout: 600 }
```

Scope 等级：**higher** = 每次手动确认；**normal**（未列出）= 近期已授权则静默；**lower** = 总是静默。

## OAuth2-Server 搭建（当前 API）

> **版本提示**：`sa-token-oauth2` 在 **v1.39.0 被完全重写，官方明确"不向下兼容"**。`SaOAuth2Handle` → `SaOAuth2ServerProcessor`；`SaOAuth2Config` → **`SaOAuth2ServerConfig`**；`SaOAuth2Strategy`、`SaOAuth2Dao`、`SaOAuth2DataLoader`、`SaOAuth2DataGenerate` 等在此时抽出；`scope` 变为 `List<String>`；新增注解鉴权。
> **v1.43.0** 把授权流程回调（`notLoginView`、`confirmView`、`doLoginHandle`）**从 `SaOAuth2ServerConfig` 移到了 `SaOAuth2Strategy.instance`** —— 这是当前位置。任何把它们设在 config 对象上的代码都是 1.43 之前的写法。同时新增 `SaClientModel.isAutoConfirm`、多 token 共存、`currentAccessToken()`/`currentClientToken()`。

```java
@RestController
public class SaOAuth2ServerController {

    @RequestMapping("/oauth2/*")
    public Object request() {
        return SaOAuth2ServerProcessor.instance.dister();
    }

    @Autowired
    public void configOAuth2Server(SaOAuth2ServerConfig oauth2Server) {

        oauth2Server.addClient(
            new SaClientModel()
                .setClientId("1001")
                .setClientSecret("aaaa-bbbb-cccc-dddd-eeee")
                .addAllowRedirectUris("http://client.com/callback")   // ⚠️ 生产禁用 "*"
                .addContractScopes("openid", "userid", "userinfo")
                .addAllowGrantTypes(
                    GrantType.authorization_code, GrantType.implicit,
                    GrantType.refresh_token, GrantType.password,
                    GrantType.client_credentials)
        );

        SaOAuth2Strategy.instance.notLoginView = () -> "...登录页...";
        SaOAuth2Strategy.instance.doLoginHandle = (name, pwd) -> {
            if("sa".equals(name) && "123456".equals(pwd)) { StpUtil.login(10001); return SaResult.ok(); }
            return SaResult.error("账号名或密码错误");
        };
        SaOAuth2Strategy.instance.confirmView = (clientId, scopes) -> "...授权确认页...";
    }
}
```

也可拆分路由（更细粒度）：`SaOAuth2ServerProcessor.instance.authorize() / doLogin() / doConfirm() / token() / refresh() / revoke() / clientToken()`。端点路径可用 `SaOAuth2Consts.Api.authorize = "/oauth2/authorize2"` 改名。

多账号：在 `configOAuth2Server` 里 `SaOAuth2Manager.setStpLogic(StpUserUtil.stpLogic)`。

## SaOAuth2DataLoader

三种提供 client 信息的方式，真实度递增：代码 `addClient(...)`（内存）→ yml `sa-token.oauth2-server.clients.*`（内存）→ **实现 `SaOAuth2DataLoader`（数据库）**，官方后续章节都假定用最后这种。

```java
@Component
public class SaOAuth2DataLoaderImpl implements SaOAuth2DataLoader {

    @Override
    public SaClientModel getClientModel(String clientId) {
        // 查库；未知 client 返回 null
        return ...;
    }

    @Override
    public String getOpenid(String clientId, Object loginId) {
        return SaOAuth2DataLoader.super.getOpenid(clientId, loginId);
    }

    @Override
    public String getUnionid(String subjectId, Object loginId) {
        return SaOAuth2DataLoader.super.getUnionid(subjectId, loginId);
    }
}
```

三个方法都是 `default` 方法（所以有 `SaOAuth2DataLoader.super.xxx` 回落）。默认 unionid 算法：`md5(prefix + "_" + subjectId + "_" + loginId)`，共享 `subjectId` 的应用会得到相同 unionid。v1.44.0 起该接口还负责 higher/lower scope 的查询。

`SaClientModel` 字段：`clientId`、`clientSecret`、`contractScopes`(List)、`allowRedirectUris`(List)、`allowGrantTypes`(List)、`subjectId`、`accessTokenTimeout`、`refreshTokenTimeout`、`clientTokenTimeout`、`maxAccessTokenCount`、`maxRefreshTokenCount`、`maxClientTokenCount`、`isNewRefresh`、`isAutoConfirm`（默认 false，⚠️ **高风险，绝不对不受信第三方开启**）。超时/数量字段默认取全局配置。

## 端点清单

| 端点 | 参数 |
|---|---|
| `GET /oauth2/authorize` | `response_type`（`code`\|`token`）、`client_id`、`redirect_uri` 必填；`scope`、`state` 可选（**必须随机、一次性、不重复**）。返回 `redirect_uri?code=..&state=..`；隐式模式返回 `redirect_uri#token=xxx`（**`#` 片段，不是 query**） |
| `POST /oauth2/doLogin` | `name`、`pwd` —— 非标准辅助端点，路由到 `doLoginHandle` |
| `POST /oauth2/doConfirm` | `client_id`、`scope` 必填；可加 `build_redirect_uri=true` + `response_type`+`redirect_uri`+`state`，直接在 JSON 里拿到最终 `redirect_uri`（前端 `location.href=res.redirect_uri`，省掉二次 `/oauth2/authorize`） |
| `/oauth2/token` | `grant_type=authorization_code` + `client_id` + `client_secret` + `code`；**或** `grant_type=password` + `client_id` + `client_secret` + `username` + `password` + `scope`。客户端凭证也可放 `Authorization: Basic base64(client_id:client_secret)` |
| `/oauth2/refresh` | `grant_type=refresh_token`、`client_id`、`client_secret`、`refresh_token` |
| `/oauth2/revoke` | `client_id`、`client_secret`、`access_token` |
| `/oauth2/client_token` | `grant_type=client_credentials`、`client_id`、`client_secret`、`scope` → 返回 `client_token`（**不是** `access_token`；`mode4-return-access-token` 可额外加 `access_token` 字段做规范兼容） |
| `/oauth2/userinfo` | **不是标准端点** —— 仅 demo，需自己写 |

token 响应字段：`token_type: "Bearer"`、`access_token`、`refresh_token`、`expires_in`、`refresh_expires_in`、`client_id`、`scope`、`openid`。

Code 语义：每次授权唯一、**一次性**、5 分钟；签发新 code 会立即作废旧的。

⚠️ **Client-Token 有延迟失效特性**：重新签发时旧的会保留为 `LowerClientToken`，新旧都能通过校验 —— 这是刻意设计，为了在高并发轮换时不中断服务。

## 校验 access-token

**注解**（需先注册框架拦截器）：

```java
@SaCheckAccessToken
@SaCheckAccessToken(scope = "userinfo")
@SaCheckAccessToken(scope = {"openid", "userinfo"})
@SaCheckClientToken
@SaCheckClientIdSecret
```

token 提交方式：query 参数 `access_token` / `client_token`，或 `Authorization: bearer xxx`；id/secret 用 `Authorization: Basic base64(client_id:client_secret)`。

**编码方式**：

```java
SaOAuth2Util.checkAccessToken(accessToken);                       // → AccessTokenModel，失败抛异常
SaOAuth2Util.getAccessToken(accessToken);                         // 无效返回 null
SaOAuth2Util.checkAccessTokenScope(accessToken, "userinfo");      // 抛异常
SaOAuth2Util.hasAccessTokenScope(accessToken, "userinfo");        // boolean
SaOAuth2Util.getLoginIdByAccessToken(accessToken);
SaOAuth2Util.getClientIdByAccessToken(accessToken);
SaOAuth2Util.revokeAccessToken(accessToken);
SaOAuth2Util.currentAccessToken();    // 从请求中读取，无效则抛异常（v1.43.0+）
SaOAuth2Util.currentClientToken();
```

读原始 token：`SaOAuth2Manager.getDataResolver().readAccessToken(SaHolder.getRequest())`。

## 把 userinfo 传给客户端 —— 两种做法

- **自定义端点**：写 `/oauth2/userinfo`，解析 token 拿 loginId，`checkAccessTokenScope(accessToken, "userinfo")`，返回数据。多一次往返。
- **自定义 scope 处理器**（更优，无额外请求）：实现 `SaOAuth2ScopeHandlerInterface` 并加 `@Component`：

```java
String getHandlerScope();                      // 如 "userinfo"
void workAccessToken(AccessTokenModel at);     // at.extraData.putAll(map) → 合并进 token 响应
void workClientToken(ClientTokenModel ct);
boolean refreshAccessTokenIsWork();            // true = 刷新时重跑（数据保持新鲜）
```

`getHandlerScope()` 返回 `SaOAuth2Consts._FINALLY_WORK_SCOPE` 可让它成为"最终处理器" —— 在所有其它处理器之后总会执行，即使没请求任何 scope。

## 自定义 grant_type

- ⚠️ **`password` 模式生产环境必须重写处理器** —— 框架会打警告，因为它默认转发给 `SaOAuth2Strategy.instance.doLoginHandle`。继承 `PasswordGrantTypeHandler` 并重写 `PasswordAuthResult loginByUsernamePassword(String username, String password)`。
- 新增 grant type：实现 `SaOAuth2GrantTypeHandlerInterface` 的 `String getHandlerGrantType()` 和 `AccessTokenModel getAccessToken(SaRequest req, String clientId, List<String> scopes)`，构造 `RequestAuthModel`，再 `SaOAuth2Manager.getDataGenerate().generateAccessToken(ra, true, atm -> atm.grantType = "phone_code")`。别忘了把新类型加进该 client 的 `allowGrantTypes`。

## OAuth2 与登录会话数据互通

默认 `access_token` 无法访问 `StpUtil`（satoken）的接口，会报"无效Token"。要统一：

- 服务端：`SaOAuth2Strategy.instance.createAccessToken = (clientId, loginId, scopes) -> StpUtil.getOrCreateLoginSession(loginId);`
- 客户端：`StpUtil.login(uid, new SaLoginParameter().setToken(access_token));`

⚠️ 官方警示：这样做**丢弃了 OAuth2 的按 client 权限隔离**，等于隐式赋予每个 client 完整的会话权限。且两个 token 的有效期各自独立（`sa-token.timeout` vs `oauth2-server.access-token-timeout`）—— 互通是一次性拷贝，不是联动。

## OAuth2 安全要点

1. **`state` 防客户端假冒**：必须不可预测、与发起 client 一一对应、返回时校验相等、一次性使用。
2. **`allowRedirectUris="*"` → code 劫持。** 框架 `SaOAuth2Template.checkRedirectUri` 强制的硬规则：任何位置都**不允许 `@` 字符**（阻断 `http://sa-oauth-client.com@evil.com`），且 `*` 只能在最末尾（`http*://...` 和 `http://*.host/` 都会被拒）。v1.39.0 修复（github issue 529）。
3. `isAutoConfirm=true` 属高风险配置，不受信第三方绝不开启。

## 当前 API 类名对照表

| 关注点 | SSO | OAuth2 |
|---|---|---|
| 请求分发器 | `SaSsoServerProcessor.instance.dister()` / `SaSsoClientProcessor.instance.dister()` | `SaOAuth2ServerProcessor.instance.dister()` |
| 注入的配置对象 | `SaSsoServerTemplate` / `SaSsoClientTemplate` | `SaOAuth2ServerConfig` |
| 策略持有者 | `ssoServerTemplate.strategy` / `ssoClientTemplate.strategy` | `SaOAuth2Strategy.instance` |
| 配置 POJO | `SaSsoServerConfig`、`SaSsoClientConfig`、`SaSsoClientModel` | `SaOAuth2ServerConfig`、`SaClientModel`、`SaOAuth2OidcConfig` |
| 数据加载 | 重写 `SaSsoServerTemplate.getClient/getClients` | 实现 `SaOAuth2DataLoader` |
| 工具类 | `SaSsoServerUtil`、`SaSsoClientUtil`、`SaSsoUtil.getData` | `SaOAuth2Util`、`SaOAuth2Manager` |
| 签名校验 | `SaSignUtil.checkRequest(req)` | （基于 client_id/secret） |
| 配置读取 | `SaSsoManager.getServerConfig()` / `getClientConfig()` | `SaOAuth2Manager.getServerConfig()` |
