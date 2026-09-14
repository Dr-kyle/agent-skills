# Sa-Token API 参考

## StpUtil 完整方法目录

`StpUtil` 本身没有任何逻辑，只是对成员变量 `stpLogic` 的转发包装 —— 这是多账号认证的基础。

### 常规 / token 值操作

```java
StpUtil.getStpLogic();                              // 获取底层 StpLogic 对象
StpUtil.setStpLogic(newStpLogic);                   // 安全重置底层 StpLogic 引用
StpUtil.getLoginType();                             // 获取账号类型 (login、user、admin...)
StpUtil.getTokenName();                             // 获取 token 名称
StpUtil.getTokenValue();                            // 获取本次请求前端提交的 token
StpUtil.getTokenValueNotCut();                      // 同上，但不裁剪前缀
StpUtil.setTokenValue(tokenValue);                  // 在当前会话写入 token 值
StpUtil.setTokenValue(tokenValue, timeout);         // 并指定 Cookie 有效期
StpUtil.setTokenValue(tokenValue, loginParameter);
StpUtil.setTokenValueToStorage(tokenValue);         // 写入当前请求的 Storage 存储器
StpUtil.getTokenInfo();                             // 获取 token 详细信息 (SaTokenInfo)
StpUtil.createSaLoginParameter();                   // 按当前配置创建 SaLoginParameter
```

### 登录

```java
StpUtil.login(10001);
StpUtil.login(10001, "APP");                   // 指定设备类型
StpUtil.login(10001, true);                    // 指定是否[记住我]
StpUtil.login(10001, 86400);                   // 指定本次 token 有效期（秒）
StpUtil.login(10001, loginParameter);          // 指定全部登录参数
StpUtil.createLoginSession(10001);             // 创建登录会话但不注入上下文，返回 token 串
StpUtil.createLoginSession(10001, loginParameter);
StpUtil.getOrCreateLoginSession(10001);        // 获取，不存在则创建
```

### 注销 / 踢人 / 顶人

```java
StpUtil.logout();                                  // 当前会话注销
StpUtil.logout(logoutParameter);
StpUtil.logout(10001);                             // 指定账号注销（所有端）
StpUtil.logout(10001, "PC");                       // 指定账号指定端
StpUtil.logout(10001, logoutParameter);
StpUtil.logoutByTokenValue(token);
StpUtil.logoutByTokenValue(token, logoutParameter);

StpUtil.kickout(10001);                            // 踢下线 → KICK_OUT (-5)
StpUtil.kickout(10001, "PC");
StpUtil.kickout(10001, logoutParameter);
StpUtil.kickoutByTokenValue(token);
StpUtil.kickoutByTokenValue(token, logoutParameter);

StpUtil.replaced(10001);                           // 顶下线 → BE_REPLACED (-4)
StpUtil.replaced(10001, "PC");                     // 框架内部在登录时自动调用，一般无需手动调
StpUtil.replaced(10001, logoutParameter);
StpUtil.replacedByTokenValue(token);
StpUtil.replacedByTokenValue(token, logoutParameter);
```

语义区别：`logout` 是干净移除，下次访问是 `NOT_TOKEN`/`INVALID_TOKEN`；`kickout`/`replaced` 故意保留旧 token 并打标记，让下次请求能明确知道"为什么"下线的。

### 登录状态查询

```java
StpUtil.isLogin();                       // 当前会话是否已登录
StpUtil.isLogin(10001);                  // 指定账号是否已登录
StpUtil.checkLogin();                    // 未登录抛 NotLoginException
StpUtil.getLoginId();                    // 未登录抛异常
StpUtil.getLoginId(defaultValue);        // 未登录返回默认值（任意类型）
StpUtil.getLoginIdDefaultNull();         // 未登录返回 null
StpUtil.getLoginIdAsString();
StpUtil.getLoginIdAsInt();
StpUtil.getLoginIdAsLong();
StpUtil.getLoginIdByToken(token);                  // 无效/被踢/被顶/被冻结 → null
StpUtil.getLoginIdByTokenNotThinkFreeze(token);    // 不考虑冻结状态
StpUtil.getExtra(key);                             // ⚠️ 仅 jwt 模式生效
StpUtil.getExtra(token, key);                      // ⚠️ 仅 jwt 模式生效
```

### Session

```java
// Account-Session
StpUtil.getSession();                               // 无则新建（需已登录）
StpUtil.getSession(true);                           // isCreate
StpUtil.getSessionByLoginId(10001);
StpUtil.getSessionByLoginId(10001, false);          // 不存在返回 null

// Token-Session
StpUtil.getTokenSession();
StpUtil.getTokenSessionByToken(token);
StpUtil.getAnonTokenSession();                      // 匿名，未登录可用

// 其它
StpUtil.getSessionBySessionId("xxxx-xxxx");         // 不存在返回 null
StpUtil.isTrustDeviceId(10001, "xxxxxxxx");
```

### 有效期

```java
// active-timeout（最低活跃频率）
StpUtil.getTokenActiveTimeout();     // 距离被冻结还剩多少秒
StpUtil.getTokenLastActiveTime();    // 最后活跃时间
StpUtil.checkActiveTimeout();        // 已冻结则抛异常
StpUtil.updateLastActiveToNow();     // 续签（⚠️ 对已冻结的 token 也会成功，即解冻）

// timeout（长久有效期）
StpUtil.getTokenTimeout();           // 当前 token 剩余秒数，-1=永久
StpUtil.getTokenTimeout(token);
StpUtil.getSessionTimeout();         // Account-Session 剩余
StpUtil.getTokenSessionTimeout();    // Token-Session 剩余
StpUtil.renewTimeout(timeout);       // 续期（v1.29.0+）
StpUtil.renewTimeout(token, timeout);
```

### 角色认证

```java
StpUtil.getRoleList();                     StpUtil.getRoleList(10001);
StpUtil.hasRole(role);                     StpUtil.hasRole(loginId, role);
StpUtil.hasRoleAnd(...roleArray);          StpUtil.hasRoleOr(...roleArray);
StpUtil.checkRole(role);                   // 抛 NotRoleException
StpUtil.checkRoleAnd(...roleArray);        StpUtil.checkRoleOr(...roleArray);
```

### 权限认证

```java
StpUtil.getPermissionList();                    StpUtil.getPermissionList(10001);
StpUtil.hasPermission(permission);              StpUtil.hasPermission(loginId, permission);
StpUtil.hasPermissionAnd(...permissionArray);   StpUtil.hasPermissionOr(...permissionArray);
StpUtil.checkPermission(permission);            // 抛 NotPermissionException
StpUtil.checkPermissionAnd(...permissionArray); StpUtil.checkPermissionOr(...permissionArray);
```

`NotRoleException.getRole()` / `NotPermissionException.getPermission()` 可取到缺失的标识；两者都有 `getLoginType()` 可判断是哪个账号体系抛的。

### id 反查 token / 终端信息

```java
StpUtil.getTokenValueByLoginId(10001);
StpUtil.getTokenValueByLoginId(10001, "PC");
StpUtil.getTokenValueListByLoginId(10001);
StpUtil.getTokenValueListByLoginId(10001, "APP");
StpUtil.getTerminalListByLoginId(10001);
StpUtil.getTerminalListByLoginId(10001, "PC");
StpUtil.forEachTerminalList(10001, (session, terminal) -> {});
StpUtil.getTerminalInfo();
StpUtil.getTerminalInfoByToken(token);
StpUtil.getLoginDeviceType();                   // ✅ 当前 API
StpUtil.getLoginDeviceTypeByToken(token);
StpUtil.getLoginDeviceId();
StpUtil.getLoginDeviceIdByToken(token);
```

⚠️ 旧文档章节里出现过 `StpUtil.getLoginDevice()`，**当前规范名是 `getLoginDeviceType()`**。

### 会话治理

```java
StpUtil.searchTokenValue(keyword, start, size, sortType);       // 所有已登录 token
StpUtil.searchSessionId(keyword, start, size, sortType);        // 所有 Account-Session
StpUtil.searchTokenSessionId(keyword, start, size, sortType);   // 所有 Token-Session
```

参数：`keyword`=token 值子串过滤，`start`=起始下标，`size`=数量（**`-1`=到末尾**），`sortType`=`true` 升序（最早登录在前）/ `false` 降序。

`searchTokenValue` 返回**每次登录产生的每个 token**；`searchSessionId` 每个已登录账号只返回一个 id。`is-concurrent=true`+`is-share=false` 时，A 登录 3 次、B 登录 2 次 → 前者 5 条，后者 2 条。

⚠️ **性能警告：搜索是遍历实现的。** 单机模式 100 万会话取 10 个 token 平均 0.255s；Redis 模式 3.322s。且基于活跃 token 的统计比实际略有延迟，要精确实时在线数请用 WebSocket。

### 账号封禁

```java
StpUtil.disable(10001, 86400);        // 封禁指定秒数，-1=永久封禁
StpUtil.isDisable(10001);
StpUtil.checkDisable(10001);          // 抛 DisableServiceException
StpUtil.getDisableTime(10001);        // -1=永久封禁, -2=未被封禁
StpUtil.untieDisable(10001);          // 解封
```

分类封禁（v1.31.0+）与阶梯封禁见 `advanced.md`。

### 身份切换

```java
StpUtil.switchTo(10044);              // 临时切换身份（本次请求内有效）
StpUtil.getLoginId();                 // 此时返回 10044
StpUtil.isSwitch();
StpUtil.endSwitch();

StpUtil.switchTo(10044, () -> {       // 代码段内切换，无需手动 endSwitch
    StpUtil.getLoginId();             // 10044
});
```

### 二级认证

```java
StpUtil.openSafe(120);                        // 开启，120 秒
StpUtil.isSafe();
StpUtil.checkSafe();                          // 未通过抛 NotSafeException (code 11071)
StpUtil.getSafeTime();                        // -2 = 尚未通过二级认证
StpUtil.closeSafe();

// 带业务标识（各标识完全隔离）
StpUtil.openSafe("client", 600);
StpUtil.isSafe("client");
StpUtil.isSafe(tokenValue, "client");
StpUtil.checkSafe("client");
StpUtil.getSafeTime("client");
StpUtil.closeSafe("client");
```

## SaTokenInfo

`StpUtil.getTokenInfo()` 返回：

```js
{
  "tokenName": "satoken",
  "tokenValue": "e67b99f1-3d7a-4a8d-bb2f-e888a0805633",
  "isLogin": true,
  "loginId": "10001",              // 未登录时为 null
  "loginType": "login",
  "tokenTimeout": 2591977,         // token 剩余有效期（秒）
  "sessionTimeout": 2591977,       // Account-Session 剩余
  "tokenSessionTimeout": -2,       // Token-Session 剩余，-2=系统中不存在该缓存
  "tokenActiveTimeout": -1,        // 距离被冻结还剩的时间
  "loginDevice": "DEF"             // 默认设备类型是 DEF
}
```

## SaSession

### 常量

```java
SaSession.USER = "USER";                       // 存储用户对象时建议用的 key
SaSession.ROLE_LIST = "ROLE_LIST";
SaSession.PERMISSION_LIST = "PERMISSION_LIST";
```

### 取值 / 存值

```java
session.get(key);
session.get(key, defaultValue);
session.get(key, () -> queryFromDb());   // 值为 null 时执行并写入缓存
session.getString(key);
session.getInt(key);
session.getLong(key);
session.getDouble(key);
session.getFloat(key);
session.getModel(key, User.class);
session.getModel(key, User.class, defaultValue);
session.has(key);
session.set(key, value);
session.setByNull(key, value);           // 仅在原本无值时写入
session.delete(key);
session.keys();                          // Set<String>
session.clear();
session.getDataMap();                    // ⚠️ 直接改 map 后需调 session.update() 避免脏数据
session.setDataMap(dataMap);
session.refreshDataMap(dataMap);
```

⚠️ 旧章节里有 `session.setDefaultValue(key, value)`，API 参考中的规范名是 **`setByNull`**。

### 元信息 / 生命周期

```java
session.getId();          session.setId(id);
session.getCreateTime();  session.setCreateTime(t);
session.getType();        session.setType(type);       // Account-Session / Token-Session / Custom-Session
session.getLoginType();   session.getLoginId();  session.getToken();

session.update();                            // 从持久库刷新
session.logout();                            // 从持久库删除
session.logoutByTerminalCountToZero();       // 终端数为零时注销会话
session.timeout();                           // 剩余存活秒数
session.updateTimeout(timeout);
session.updateMinTimeout(minTimeout);        // 仅当当前过期时间低于该值时才改
session.updateMaxTimeout(maxTimeout);        // 仅当当前过期时间高于该值时才改
```

### 终端列表（多设备）

```java
session.setTerminalList(list);
session.getTerminalList();
session.terminalListCopy();
session.getTerminalListByDeviceType("PC");
session.getTokenValueListByDeviceType("PC");
session.getTerminal(tokenValue);
session.addTerminal(terminal);
session.removeTerminal(tokenValue);
session.forEachTerminalList((session, terminal) -> {});
session.getHistoryTerminalCount();
session.setHistoryTerminalCount(count);
session.isTrustDeviceId("xxxxxxxx");
```

`SaTerminalInfo` 字段（旧名 `TokenSign` 已废弃）：

```java
terminal.getIndex();       // 登录会话索引（该账号第几个登录的设备）
terminal.getDeviceType();  // 设备类型：PC、WEB、HD、MOBILE、APP，默认 DEF
terminal.getTokenValue();
terminal.getCreateTime();  // 13 位时间戳
terminal.getDeviceId();
terminal.getExtra("key");  // 来自 SaLoginParameter.setTerminalExtra
```

### 三种 Session 对比

| 类型 | 键 | 获取方式 | 说明 |
|---|---|---|---|
| **Account-Session** | loginId | `StpUtil.getSession()` / `getSessionByLoginId(10001)` | 同账号跨端共享，适合跨设备同步数据。**需已登录** |
| **Token-Session** | token | `StpUtil.getTokenSession()` / `getTokenSessionByToken(t)` / `getAnonTokenSession()` | 各端隔离。登录前使用需 `token-session-check-login: false` 或用匿名版 |
| **Custom-Session** | 任意 SessionId | `SaSessionCustomUtil.getSessionById("goods-10001")` | 同 key 同 Session。默认 TTL 取全局 `timeout`，可 `updateTimeout` |

```java
SaSessionCustomUtil.isExists("goods-10001");
SaSessionCustomUtil.getSessionById("goods-10001");          // 没有则新建
SaSessionCustomUtil.getSessionById("goods-10001", false);   // 没有则返回 null
SaSessionCustomUtil.deleteSessionById("goods-10001");
```

⚠️ **`SaSession` 与 `HttpSession` 完全不互通。** 往 `HttpSession` 写的值在 `SaSession` 里读不到，反之亦然。

## 三大作用域

| 作用域 | 生命周期 | 需登录 | API |
|---|---|---|---|
| `SaStorage`（请求作用域） | 单次请求，结束自动清理 | 否 | `SaHolder.getStorage()` |
| `SaSession`（会话作用域） | 会话期间 | **是** | `StpUtil.getSession()` |
| `SaApplication`（全局作用域） | 应用生命周期 | 否 | `SaHolder.getApplication()` |

三者都支持 `get(key)` / `set(key, value)` / `delete(key)`。

## SaLoginParameter 全量参数

⚠️ 当前类名是 **`SaLoginParameter`**（v1.41.0 起 `SaLoginModel` 已重命名，不向下兼容）。

```java
StpUtil.login(10001, new SaLoginParameter()
    .setDeviceType("PC")                 // 客户端设备类型，用于[同端互斥登录]
    .setDeviceId("xxxxxxxxx")            // 客户端设备ID，登录后标记为可信任设备
    .setIsLastingCookie(true)            // 是否持久 Cookie
    .setTimeout(60 * 60 * 24 * 7)        // 本次 token 有效期（秒），-1=永久
    .setActiveTimeout(60 * 60 * 24 * 7)  // 本次最低活跃频率（秒），-1=不检查
    .setIsConcurrent(true)               // 是否允许同账号多地同时登录
    .setIsShare(false)                   // 多人登录同账号时是否共用一个 token
    .setMaxLoginCount(12)                // 同账号最大登录数量，-1=不限
    .setMaxTryTimes(12)                  // 创建 token 的最大循环次数
    .setExtra("key", "value")            // ⚠️ 记录在 token 上的扩展参数，仅 jwt 模式生效
    .setToken("xxxx-xxxx")               // 预定本次生成的 token
    .setIsWriteHeader(false)             // 是否把 token 写入响应头
    .setTerminalExtra("key", "value")    // 挂到 SaTerminalInfo 的自定义扩展数据
    .setReplacedRange(SaReplacedRange.CURR_DEVICE_TYPE)   // 顶人下线范围
    .setOverflowLogoutMode(SaLogoutMode.LOGOUT)           // 溢出客户端下线方式
    .setRightNowCreateTokenSession(true) // 是否立即创建 Token-Session
    .setupCookieConfig(cookie -> {
        cookie.setDomain("sa-token.com");
        cookie.setPath("/shop");
        cookie.setSecure(true);
        cookie.setHttpOnly(true);
        cookie.setSameSite("Lax");
        cookie.addExtraAttr("aa", "bb");
    })
);
```

未指定的参数回落到全局配置。⚠️ `setExtra` 只在 jwt 模式生效；非 jwt 模式调 `StpUtil.getExtra(key)` 会抛 `this api is disabled`，改用 `StpUtil.getSession().set("name","zhangsan")`。

## SaLogoutParameter

```java
StpUtil.logout(new SaLogoutParameter()
    // TOKEN=只注销当前 token；ACCOUNT=注销该 loginId 的所有客户端会话
    // 仅对 StpUtil.logout() 生效
    .setRange(SaLogoutRange.TOKEN)
);

StpUtil.logoutByTokenValue("xxxx", new SaLogoutParameter()
    // 已冻结的 token 是否保留调用注销 API 的权限（默认 false）
    // 仅对 [logout/kickout/replaced]ByTokenValue 生效
    .setIsKeepFreezeOps(false)
    .setIsKeepTokenSession(true)      // 是否保留 Token-Session（默认 false）
);

StpUtil.logout(10001, new SaLogoutParameter()
    .setDeviceType("PC")              // 不指定则注销所有客户端
    .setIsKeepTokenSession(true)
    .setMode(SaLogoutMode.REPLACED)   // LOGOUT / KICKOUT / REPLACED（默认 LOGOUT）
);
```

参数表达不了的策略，手动遍历终端：

```java
StpUtil.forEachTerminalList(10001, (session, ter) -> {
    if(ter.getIndex() % 2 == 0) {
        StpUtil.removeTerminalByLogout(session, ter);
        // StpUtil.removeTerminalByKickout(session, ter);
        // StpUtil.removeTerminalByReplaced(session, ter);
    }
});
```

## SaTokenDao

自定义存储层时实现此接口（默认内存实现，官方插件如 `sa-token-redis-template`）。

```java
SaTokenDao.NEVER_EXPIRE = -1;      // 永不过期
SaTokenDao.NOT_VALUE_EXPIRE = -2;  // 系统中不存在这个缓存

// 字符串读写
dao.get(key);  dao.set(key, value, timeout);  dao.update(key, value);  dao.delete(key);
dao.getTimeout(key);  dao.updateTimeout(key, timeout);

// 对象读写
dao.getObject(key);  dao.getObject(key, classType);  dao.setObject(key, value, timeout);
dao.updateObject(key, value);  dao.deleteObject(key);
dao.getObjectTimeout(key);  dao.updateObjectTimeout(key, timeout);

// Session 读写
dao.getSession(sessionId);  dao.setSession(session, timeout);  dao.updateSession(session);
dao.deleteSession(sessionId);  dao.getSessionTimeout(sessionId);  dao.updateSessionTimeout(sessionId, timeout);

// 会话管理
dao.searchData(prefix, keyword, start, size, sortType);
```

访问：`SaManager.getSaTokenDao()` / `SaManager.setSaTokenDao(dao)`。

**排查"是不是连错 Redis"的最佳工具**：`SaManager.getSaTokenDao().set("name","value",100000);`（不要用自己的 RedisUtil 测）。

## SaStrategy 全局策略

```java
/** 创建 Token 的策略，参数 [账号id, 账号类型] */
public BiFunction<Object, String, String> createToken;

/** 创建 Session 的策略，参数 [SessionId] */
public Function<String, SaSession> createSession;

/** 反序列化 SaSession 时默认指定的类型 */
public Class<? extends SaSession> sessionClassType = SaSession.class;

/** 判断集合是否包含指定元素（模糊匹配），参数 [集合, 元素] */
public BiFunction<List<String>, String, Boolean> hasElement;

/** 生成唯一式 token 的算法 */
public SaGenerateUniqueTokenFunction generateUniqueToken;

/** 是否自动续期，参数为当前 stpLogic 实例，返回 true 则续期 */
public Function<StpLogic, Boolean> autoRenew;

/** 创建 StpLogic 的算法，参数为账号体系标识 */
public SaCreateStpLogicFunction createStpLogic;

/** 路由匹配策略，参数 [pattern, path] */
public SaRouteMatchFunction routeMatcher;

/** CORS 策略处理函数，参数 [请求包装对象, 响应包装对象, 数据读写对象] */
public SaCorsHandleFunction corsHandle;
```

重写方式（`routeMatcher` / `corsHandle` 直接赋值，其余也有 fluent setter）：

```java
@PostConstruct
public void rewriteSaStrategy() {
    SaStrategy.instance.createToken = (loginId, loginType) -> SaFoxUtil.getRandomString(60);
}
```

注意注解处理策略在**另一个**持有者上：`SaAnnotationStrategy.instance`（`getAnnotation`、`registerAnnotationHandler`、`checkELRootMapExtendFunction`）。

## SaManager / SaHolder 全局入口

```java
SaManager.getConfig();  SaManager.setConfig(config);
SaManager.getSaTokenDao();  SaManager.setSaTokenDao(dao);
SaManager.getStpInterface();  SaManager.setStpInterface(si);
SaManager.getSaTokenContext();  SaManager.getSaTempTemplate();
SaManager.getSaJsonTemplate();  SaManager.getSaHttpTemplate();
SaManager.getSaSerializerTemplate();  SaManager.getSaSameTemplate();
SaManager.getSaTotpTemplate();  SaManager.getLog();  SaManager.setLog(log);
SaManager.getStpLogic("type");  SaManager.getStpLogic("type", false);  SaManager.putStpLogic(stpLogic);

SaHolder.getContext();  SaHolder.getRequest();  SaHolder.getResponse();
SaHolder.getStorage();  SaHolder.getApplication();
```

常用请求 / 响应操作：

```java
SaHolder.getRequest().getParam("key");
SaHolder.getRequest().getParamNotNull("key");
SaHolder.getRequest().getHeader("satoken");
SaHolder.getRequest().getRequestPath();     // 调试拦截路径问题时必用
SaHolder.getRequest().getUrl();

SaHolder.getResponse().setStatus(401);
SaHolder.getResponse().setHeader("Content-Type", "application/json;charset=UTF-8");
SaHolder.getResponse().redirect("/login?back=" + SaHolder.getRequest().getUrl());
```

## 异常体系

所有异常继承 `SaTokenException`，都可 `e.getCode()` 拿细分码（`-1` 表示抛出点未指定码）。

| 异常 | 访问器 / 含义 | 码 |
|---|---|---|
| `NotLoginException` | `getType()` → 场景值常量 | 11011–11016 |
| `NotRoleException` | `getRole()` | 11041 |
| `NotPermissionException` | `getPermission()` | 11051 |
| `NotSafeException` | 二级认证未通过 | 11071 |
| `DisableServiceException` | `getService()`、`getLevel()`、`getLimitLevel()` | 11061–11064 |
| `SameTokenInvalidException` | Same-Token 无效 | 10301 |
| `SaTokenContextException` | 上下文尚未初始化 | 10001/10002 |
| `InvalidContextException` | 未能获取有效的上下文处理器 | — |
| `FirewallCheckException` | 防火墙 hook 抛出 | — |
| `ApiDisabledException` | 如未集成 jwt 时调 `getExtra()` | 11031 |
| `SaSsoException` | SSO 错误 | 30001–30011 |
| `SaOAuth2Exception` | OAuth2 错误 | 30101–30191 |

按码统一处理：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(SaTokenException.class)
    public SaResult handler(SaTokenException e) {
        if(e.getCode() == 30001) return SaResult.error("redirect 重定向 url 是一个无效地址");
        if(e.getCode() == 30004) return SaResult.error("提供的 ticket 是无效的");
        return SaResult.error("服务器繁忙，请稍后重试...");
    }
}
```

抛出时带码：`throw new SaSsoException("无效redirect：" + url).setCode(SaSsoErrorCode.CODE_30001);`

### 常用状态码速查

`10001` 未能获取有效的上下文处理器 · `10002` 未能获取有效的上下文 · `10011` 未能从全局 StpLogic 集合中找到对应 type · `10301` 无效 Same-Token · `10311` Http Basic 认证失败 · `11001/11011` 未能读取到有效Token · `11002` 登录时账号id为空 · `11012` Token无效 · `11013` Token已过期 · `11014` Token已被顶下线 · `11015` Token已被踢下线 · `11016` Token已被冻结 · `11031` 未集成 sa-token-jwt 却调 `getExtra()` · `11041` 缺少角色 · `11051` 缺少权限 · `11061` 未通过服务封禁校验 · `11071` 二级认证未通过 · `12201–12203` 签名秘钥为空/签名无效/timestamp 超范围 · `20101` 企图在非 Web 上下文获取 Request/Response · jwt 插件 `30201–30206`。
