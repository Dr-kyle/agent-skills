# Sa-Token 配置参考

配置有两种写法，语义不同：

```java
// 模式一：@Bean @Primary —— 会【覆盖】 application.yml 中的配置
@Configuration
public class SaTokenConfigure {
    @Bean
    @Primary
    public SaTokenConfig getSaTokenConfigPrimary() {
        SaTokenConfig config = new SaTokenConfig();
        config.setTokenName("satoken");
        config.setTimeout(30 * 24 * 60 * 60);
        config.setTokenStyle("uuid");
        return config;
    }
}
```

```java
// 模式二：@Autowired 方法 —— 与 application.yml 【合并】（代码配置优先）
@Configuration
public class SaTokenConfigure {
    @Autowired
    public void configSaToken(SaTokenConfig config) {
        config.setTokenName("satoken");
        config.setTimeout(30 * 24 * 60 * 60);
    }
}
```

验证配置是否生效：`System.out.println("Sa-Token 配置：" + SaManager.getConfig());`

⚠️ yml 中必须是**顶级** `sa-token:`，不可嵌套在 `spring:` 下。

## 核心配置项全表

下表左列为 `SaTokenConfig` 字段名（camelCase），yml 中使用 kebab-case（如 `activeTimeout` → `active-timeout`）。

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `tokenName` | String | `satoken` | token 名称（同时是 Cookie 名称、数据持久化前缀） |
| `timeout` | long | `2592000` | token 有效期（秒），默认 30 天，`-1`=永久 |
| `activeTimeout` | long | `-1` | token 最低活跃频率（秒），超时冻结，`-1`=不限制 |
| `dynamicActiveTimeout` | Boolean | `false` | 是否启用动态 activeTimeout；false 可省缓存请求 |
| `isConcurrent` | Boolean | `true` | 是否允许同账号多地同时登录（false=新登录挤掉旧登录） |
| `isShare` | Boolean | `false` | 多人登录同账号时是否共用一个 token。⚠️ 若 login 时提供了 Extra 数据，即使为 true 也会新建 token |
| `replacedLoginExitMode` | `SaReplacedLoginExitMode` | `OLD_DEVICE` | `isConcurrent=false` 时谁下线：`OLD_DEVICE`（旧设备下线）/ `NEW_DEVICE`（新设备登录失败） |
| `replacedRange` | `SaReplacedRange` | `CURR_DEVICE_TYPE` | 顶人下线范围：`CURR_DEVICE_TYPE` / `ALL_DEVICE_TYPE` |
| `maxLoginCount` | int | `12` | 同账号最大登录数量，`-1`=不限。**仅在 `isConcurrent=true` 且 `isShare=false` 时有意义**，溢出按先进先出注销 |
| `overflowLogoutMode` | `SaLogoutMode` | `LOGOUT` | 溢出客户端的下线方式：`LOGOUT` / `KICKOUT` / `REPLACED` |
| `maxTryTimes` | int | `12` | 创建 token 时的最大循环次数（保证唯一性），`-1`=不重试 |
| `isReadBody` | Boolean | `true` | 是否尝试从请求体读取 token |
| `isReadHeader` | Boolean | `true` | 是否尝试从 header 读取 token |
| `isReadCookie` | Boolean | `true` | 是否尝试从 Cookie 读取 token。**为 false 时 `StpUtil.login()` 也不再注入 Cookie** |
| `isLastingCookie` | Boolean | `true` | 是否为持久 Cookie（临时 Cookie 关浏览器即失效） |
| `isWriteHeader` | Boolean | `false` | 登录后是否把 token 写入响应头 |
| `logoutRange` | `SaLogoutRange` | `TOKEN` | `TOKEN`=只注销当前 token，`ACCOUNT`=注销该账号所有会话。仅对 `StpUtil.logout()` 生效 |
| `isLogoutKeepFreezeOps` | Boolean | `false` | 已冻结的 token 是否保留调用注销 API 的权限。仅对 `[logout/kickout/replaced]ByTokenValue` 生效 |
| `isLogoutKeepTokenSession` | Boolean | `false` | 注销后是否保留 Token-Session |
| `rightNowCreateTokenSession` | Boolean | `false` | 登录时立即创建 Token-Session，还是首次 `getTokenSession()` 时懒创建 |
| `tokenStyle` | String | `uuid` | token 风格，见下 |
| `dataRefreshPeriod` | int | `30` | 默认 DAO 的过期数据清理间隔（秒），`-1`=不清理 |
| `tokenSessionCheckLogin` | Boolean | `true` | 获取 Token-Session 时是否校验登录态（防伪造 token 拿到 Session） |
| `autoRenew` | Boolean | `true` | 是否自动续期（每次直接/间接 `getLoginId()` 都做过期检查+续签） |
| `tokenPrefix` | String | `null` | token 提交前缀，如 `Bearer` |
| `cookieAutoFillPrefix` | Boolean | `false` | Cookie 模式下自动填充 token 前缀 |
| `isPrint` | Boolean | `true` | 启动时是否打印字符画 |
| `isLog` | Boolean | `false` | 是否输出操作日志（注意：官方示例 yml 里写的是 `true`） |
| `logLevel` | String | `trace` | `trace/debug/info/warn/error/fatal`，与 `logLevelInt` 联动 |
| `logLevelInt` | int | `1` | 1=trace … 6=fatal |
| `isColorLog` | Boolean | `null` | true/false/null(自动检测终端) |
| `jwtSecretKey` | String | `null` | JWT 秘钥（`sa-token-jwt` / `sa-token-temp-jwt` 模块生效） |
| `sameTokenTimeout` | long | `86400` | Same-Token 有效期（秒） |
| `basic` | String | `""` | Http Basic 认证的 `账号:密码` |
| `currDomain` | String | `null` | 当前项目的网络地址 |
| `checkSameToken` | Boolean | `false` | 是否校验 Same-Token（部分 RPC 插件用） |
| `cookie` | Object | `new SaCookieConfig()` | Cookie 子配置 |
| `sign` | Object | `new SaSignConfig()` | API 签名子配置 |

### tokenStyle 可选值

| 值 | 样例 |
|---|---|
| `uuid`（默认） | `623368f0-ae5e-4475-a53f-93e4225f16ae` |
| `simple-uuid` | `6fd4221395024b5f87edd34bc3258ee8`（去掉横线） |
| `random-32` | `qEjyPsEA1Bkc9dr8YP6okFr5umCZNR6W` |
| `random-64` | 64 位随机串 |
| `random-128` | 128 位随机串 |
| `tik` | `gr_SwoIN0MC1ewxHX_vfCW3BothWDZMMtx__` |

完全自定义 token 生成算法：

```java
@Configuration
public class SaTokenConfigure {
    @PostConstruct
    public void rewriteSaStrategy() {
        SaStrategy.instance.createToken = (loginId, loginType) -> {
            return SaFoxUtil.getRandomString(60);
        };
    }
}
```

⚠️ 改了 token 风格不生效？**清掉 Redis 里的旧数据再试。**

## Cookie 子配置

| 键 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `domain` | String | `null` | Cookie 作用域（二级域名 SSO 共享时用） |
| `path` | String | `/` | 路径 |
| `secure` | Boolean | `false` | 是否仅 https 有效 |
| `httpOnly` | Boolean | `false` | 是否禁止 js 操作 |
| `sameSite` | String | `Lax` | `Strict` / `Lax` / `None` |
| `extraAttrs` | Map | `{}` | 额外属性（如 `Priority`、`Partitioned`） |

```yaml
sa-token:
  cookie:
    domain: stp.com
    path: /
    secure: false
    httpOnly: true
    sameSite: Lax
    extraAttrs:
      Priority: Medium
      Partitioned: ""
```

## API 签名子配置

| 键 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `secretKey` | String | `null` | 签名秘钥 |
| `timestampDisparity` | long | `900000` | 允许的时间戳偏差（毫秒），`-1`=不校验，默认 15 分钟 |
| `digestAlgo` | String | `md5` | 摘要算法 |

```yaml
sa-token:
  sign:
    secret-key: kQwIOrYvnXmSDkwEiFngrKidMcdrgKor
```

多应用模式：

```yaml
sa-token:
  sign-many:
    xm-shop:
      secret-key: 0123456789abcdefg
      digest-algo: md5
    xm-forum:
      secret-key: 0123456789hijklmnopq
      digest-algo: sha256
```

## API Key 子配置

| 键 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `prefix` | String | `AK-` | API Key 前缀 |
| `timeout` | long | `2592000` | 有效期，`-1`=永久。改动不影响已创建的 key |
| `isRecordIndex` | Boolean | `true` | 是否记录索引信息 |

yml 路径为 `sa-token.api-key.*`（官方 properties 示例里曾误写为 `sa-token.pi-key`，以 `api-key` 为准）。

## Redis 配置

```yaml
spring:
  redis:          # ⚠️ SpringBoot 3.x 起改为 spring.data.redis
    database: 1
    host: 127.0.0.1
    port: 6379
    # password:
    timeout: 10s
    lettuce:
      pool:
        max-active: 200
        max-wait: -1ms
        max-idle: 10
        min-idle: 0
```

独立 Redis（权限缓存与业务缓存分离，需 `sa-token-alone-redis`）：

```yaml
sa-token:
  alone-redis:
    database: 2
    host: 127.0.0.1
    port: 6379
    timeout: 10s
spring:
  redis:
    database: 0
    host: 127.0.0.1
    port: 6379
```

⚠️ `alone-redis` 仅对 `sa-token-redis-template` 和 `sa-token-redis-template-jdk-serializer` 生效。SpringBoot 4.x 需用 `sa-token-alone-redis-by-spring-boot4`（v1.45.0 时尚未发布到中央仓库，需自行构建）。

## 多账号体系分别配置

```java
@Configuration
public class SaTokenConfigure {
    @PostConstruct
    public void setSaTokenConfig() {
        SaTokenConfig config1 = new SaTokenConfig();
        config1.setTokenName("satoken1");
        config1.setTimeout(1000);
        StpUtil.stpLogic.setConfig(config1);

        SaTokenConfig config2 = new SaTokenConfig();
        config2.setTokenName("satoken2");
        config2.setTimeout(2000);
        StpUserUtil.stpLogic.setConfig(config2);
    }
}
```

## SpringBoot 3.x 路径匹配

SpringBoot 3.x 默认 `path_pattern_parser`，禁止 `**` 之后再有内容（如 `/admin/**/info`），会报 `No more pattern data allowed after {*...} or ** pattern element`。两个解法：

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

WebFlux / SpringCloud Gateway 下 `SaPatternsRequestConditionHolder` 会抛 `NoClassDefFoundError`，改用：

```java
SaStrategy.instance.routeMatcher = (pattern, path) ->
        SaPathMatcherHolder.getPathMatcher().match(pattern, path);
```
