# Sa-Token Redis 数据结构（调试用）

排查问题时直接看框架往 Redis 写了什么，往往比读代码快。

`{tokenName}` 是配置的 `sa-token.token-name`（默认 `satoken`）；`{loginType}` 是账号体系标识（默认 `login`）。

## 登录会话

| Key 格式 | 值 | TTL |
|---|---|---|
| `{tokenName}:{loginType}:token:{tokenValue}` | `{loginId}` | token 的 `timeout` |
| `{tokenName}:{loginType}:last-active:{tokenValue}` | 13 位时间戳，或 `时间戳,activeTimeout` | token 的 `timeout` |
| `{tokenName}:{loginType}:session:{loginId}` | SaSession（**Account-Session**） | |
| `{tokenName}:{loginType}:token-session:{loginId}` | SaSession（**Token-Session**） | |
| `{tokenName}:custom:session:{sessionId}` | SaSession（**Custom-Session**） | |
| `{tokenName}:{loginType}:safe:{service}:{tokenValue}` | 常量 `SAFE_AUTH_SAVE_VALUE`（二级认证） | |
| `{tokenName}:{loginType}:disable:{service}:{loginId}` | 封禁级别（int） | |
| `{tokenName}:var:{变量名}` | SaApplication 全局变量 | |

示例：`satoken:login:token:47ab0105-2be1-400c-b517-82f81a0cfcf8 --> 10001`

> 注：官方文档中 Token-Session 的 key 格式写作 `token-session:{loginId}`，但示例里展示的是 `satoken:login:session:47ab0105-...`（`session:` 下挂 token 值）。调试时以实际 key 为准，直接扫 Redis 确认。

## ⚠️ token→loginId 位置的哨兵值

这是排查鉴权失败最有用的信息 —— 如果那个 key 的值是负数而不是 loginId，说明：

```
-1  未能从请求中读取到有效 token
-2  已读取到 token，但 token 无效
-3  已读取到 token，但 token 已过期
-4  已读取到 token，但 token 已被顶下线 (replaced)
-5  已读取到 token，但 token 已被踢下线 (kicked)
-6  已读取到 token，但 token 已被冻结 (frozen / active-timeout)
-7  未按照指定前缀提交 token
```

## ⚠️ active-timeout 不是靠 Redis TTL 实现的

判定方式是 `当前时间 - token最后访问时间 > active-timeout` ⇒ 冻结。`last-active` key 自身的 TTL 等于 token 的 `timeout`。

**所以续签时看到 Redis TTL 没变化是正常的，不是 bug。**

## SaSession 的值结构

JSON，包含 `@class`、`id`、`type`（`Account-Session`/`Token-Session`/`Custom-Session`）、`loginType`、`loginId`、`token`、`createTime`、`dataMap`，以及 `terminalList`（仅 Account-Session）。

`terminalList` 里的 `SaTerminalInfo` 含 `index`、`tokenValue`、`deviceType`（默认 `DEF`）、`deviceId`、`extraData`、`createTime`。

## 非 Redis 存储（SaStorage，每请求级）

```
JUST_CREATED_ -> {token}
JUST_CREATED_NOT_PREFIX_ -> {token}
SWITCH_TO_SAVE_KEY_{loginType} -> {loginId}     # 临时身份切换
```

## SSO

```
{tokenName}:ticket:{ticket}                       -->  {loginId}
{tokenName}:ticket-client:{ticket}                -->  {client}
{tokenName}:ticket-index:{client}:{loginId}       -->  {ticket}
```

## OAuth2

格式为 `{tokenName}:oauth2:{类型}:{值}`，外加一个 `-index` 反查 key：

```
{tokenName}:oauth2:code:{code}                              --> CodeModel
{tokenName}:oauth2:code-index:{clientId}:{loginId}          --> code
{tokenName}:oauth2:access-token:{accessToken}               --> AccessTokenModel
{tokenName}:oauth2:access-token-index:{clientId}:{loginId}  --> access_token
{tokenName}:oauth2:refresh-token:{refreshToken}             --> RefreshTokenModel
{tokenName}:oauth2:refresh-token-index:{clientId}:{loginId} --> refresh_token
{tokenName}:oauth2:client-token:{clientToken}               --> ClientTokenModel
{tokenName}:oauth2:client-token-index:{clientId}            --> client_token
{tokenName}:oauth2:lower-client-token-index:{clientId}      --> client_token
{tokenName}:oauth2:grant-scope:{clientId}:{loginId}         --> 逗号分隔的 scope
```

## 插件

```
# 临时 token（namespace 默认 "temp-token"）
{tokenName}:{namespace}:{temp-token}          --> {value}
{tokenName}:raw-session:{namespace}:{value}   --> Raw#SaSession   （value → token 索引）
   索引挂在 SaSession 的 key "__HD_TEMP_TOKEN_MAP"，Map<temp-token, 到期时间戳>

# Same-Token —— 排查网关 `无效Same-Token` 的第一现场
{tokenName}:var:same-token        --> same-token 值
{tokenName}:var:past-same-token   --> 次级（旧）same-token 值

# API 签名
{tokenName}:sign:nonce:{nonce}    --> nonce 值（默认 32 位随机字符）

# API Key（namespace 默认 "apikey"，全小写；ttl = 剩余有效期）
{tokenName}:{namespace}:{apikey}              --> ApiKeyModel
{tokenName}:raw-session:{namespace}:{value}   --> Raw#SaSession   （value → apikey 索引）
   索引挂在 SaSession 的 key "__HD_API_KEY_LIST"，List<apiKey>
```

示例：`satoken:apikey:AK-XCoJLP2E7Q9GXyPiiZWMM8Sqi6Fm0JoFC41R`

## 调试要诀

⚠️ **验证"是不是连错 Redis"必须用框架自己的 DAO，不要用自己的 RedisUtil：**

```java
SaManager.getSaTokenDao().set("name", "value", 100000);
```

⚠️ **改了 `sa-token.token-name` 等于换掉了所有 key 的前缀**，效果相当于清空了 Redis —— 所有人都会掉线。

⚠️ **多个项目共用一个 Redis** 时用不同 database 索引、不同 `token-name`，或 `sa-token-three-redis-jackson-add-prefix` 插件。
