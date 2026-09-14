# Sa-Token 进阶功能

## 同端互斥登录 / 踢人下线

QQ 式行为（手机和电脑可同时在线，但两个手机互斥）= **`is-concurrent: false` + 登录时声明设备类型**。

```java
StpUtil.login(10001, "PC");             // 同设备类型的会话会被顶下线，其它设备类型不受影响
StpUtil.getLoginDeviceType();           // 当前 token 的登录设备类型
StpUtil.getTokenValueByLoginId(10001, "APP");   // 取指定账号指定端的 token
```

相关配置：

| 配置 | 默认 | 说明 |
|---|---|---|
| `is-concurrent` | `true` | `false`=新登录挤掉旧登录 |
| `is-share` | `false` | `true`=多次登录共用一个 token |
| `replaced-login-exit-mode` | `OLD_DEVICE` | `is-concurrent=false` 时谁下线：旧设备 / `NEW_DEVICE`(新设备登录失败) |
| `replaced-range` | `CURR_DEVICE_TYPE` | 顶下线范围：当前设备类型端 / `ALL_DEVICE_TYPE` |
| `max-login-count` | `12` | 最大登录数量，溢出按先进先出注销。仅 `is-concurrent=true`+`is-share=false` 时有意义 |
| `overflow-logout-mode` | `LOGOUT` | 溢出时的下线方式 |

被顶下线的客户端下次请求抛 `NotLoginException` 场景值 `-4`；被强制注销的抛 `-2`；被踢下线的抛 `-5`。

⚠️ `is-concurrent=false` 时 Redis 里残留旧 token 是**故意的** —— 为了能告诉旧客户端"你被顶下线了"，它们会随 TTL 自然过期。

## 二级认证

用于敏感操作前的二次确认（如删除仓库前重新输密码），不必重新登录。

```java
StpUtil.openSafe(120);                 // 开启，120 秒有效
StpUtil.isSafe();
StpUtil.checkSafe();                   // 未通过抛 NotSafeException
StpUtil.getSafeTime();                 // 剩余秒数，-2=尚未通过
StpUtil.closeSafe();
```

按业务标识隔离（各标识完全独立，`openSafe("client")` 不满足 `checkSafe("shop")`）：

```java
StpUtil.openSafe("client", 600);
StpUtil.isSafe("client");
StpUtil.isSafe(tokenValue, "client");
StpUtil.checkSafe("client");
StpUtil.closeSafe("client");
```

典型流程：

```java
@RequestMapping("openSafe")
public SaResult openSafe(String password) {
    if("123456".equals(password)) {          // 实际项目校验真实密码
        StpUtil.openSafe(120);
        return SaResult.ok("二级认证成功");
    }
    return SaResult.error("二级认证失败");
}

@RequestMapping("deleteProject")
public SaResult deleteProject(String projectId) {
    if(!StpUtil.isSafe()) {
        return SaResult.error("请完成二级认证后再访问");
    }
    // 业务逻辑 ...
    return SaResult.ok();
}
```

注解形式：`@SaCheckSafe` / `@SaCheckSafe("art")`（需注册 `SaInterceptor`）。

## 账号封禁

### 整体封禁

```java
StpUtil.disable(10001, 86400);        // 封禁 1 天，-1=永久封禁
StpUtil.isDisable(10001);
StpUtil.checkDisable(10001);          // 抛 DisableServiceException
StpUtil.getDisableTime(10001);        // -1=永久封禁, -2=未被封禁
StpUtil.untieDisable(10001);
```

⚠️ **封禁不会让用户下线。** 要立即生效需先踢再封：

```java
StpUtil.kickout(10001);
StpUtil.disable(10001, 86400);
```

⚠️ **v1.31.0 起 `StpUtil.login()` 不再自动校验封禁**，必须自己先校验：

```java
StpUtil.checkDisable(10001);
StpUtil.login(10001);
```

### 分类封禁（v1.31.0+）

只封禁某项业务能力（如禁止评论但可正常登录）。业务标识是任意字符串。

```java
StpUtil.disable(10001, "comment", 86400);
StpUtil.isDisable(10001, "comment");
StpUtil.checkDisable(10001, "comment");       // 支持传多个业务标识
StpUtil.getDisableTime(10001, "comment");
StpUtil.untieDisable(10001, "comment");       // 支持传多个
```

异常里可取 `e.getService()`。

### 阶梯封禁（v1.31.0+）

数字越大封禁越重。**判定规则：`Level >= LimitLevel` 时抛异常。**

```java
StpUtil.disableLevel(10001, 3, 10000);          // 账号、级别、时间
StpUtil.getDisableLevel(10001);                 // 未封禁返回 -2
StpUtil.isDisableLevel(10001, 3);
StpUtil.checkDisableLevel(10001, 2);            // 已被 3 级封禁，校验是否达到 2 级 → 抛异常

// 分类 + 阶梯组合
StpUtil.disableLevel(10001, "comment", 3, 10000);
StpUtil.checkDisableLevel(10001, "comment", 2);
```

异常里可取 `e.getLevel()`（实际级别）和 `e.getLimitLevel()`（要求低于的级别）。

### 注解

```java
@SaCheckDisable                                          // 默认服务
@SaCheckDisable("comment")
@SaCheckDisable({"comment", "place-order", "open-shop"})  // 任一被封禁即拦截
@SaCheckDisable(level = 5)
@SaCheckDisable(value = "comment", level = 5)
```

### 封禁数据持久化

封禁默认存缓存，重启丢失。用 `StpInterface#isDisabled` 实现懒查库（仅 `checkDisable` 时触发）：

```java
@Component
public class StpInterfaceImpl implements StpInterface {
    /**
     * 返回指定账号 id 是否被封禁
     * @param loginId 账号id
     * @param service 业务标识符
     */
    public SaDisableWrapperInfo isDisabled(Object loginId, String service) {
        // 查库 ...
        return SaDisableWrapperInfo.createDisabled(86400, 1);
    }
}
```

`SaDisableWrapperInfo` 构造方式：

```java
new SaDisableWrapperInfo(true, 86400, 1);            // 是否封禁、封禁秒数、封禁等级
SaDisableWrapperInfo.createDisabled(86400, 1);       // 已封禁
SaDisableWrapperInfo.createNotDisabled();            // 未封禁
SaDisableWrapperInfo.createNotDisabled(86400);       // 未封禁，且缓存该结果 86400 秒
```

## 身份切换 / 模拟他人

```java
StpUtil.switchTo(10044);              // 本次请求内临时切换
StpUtil.getLoginId();                 // 10044
StpUtil.isSwitch();
StpUtil.endSwitch();

StpUtil.switchTo(10044, () -> {       // 代码段内切换，自动结束
    StpUtil.getLoginId();             // 10044
});
```

对他人账号的操作（无需对方在线）：

```java
StpUtil.getTokenValueByLoginId(10001);
StpUtil.logout(10001);
StpUtil.getSessionByLoginId(10001);
StpUtil.getSessionByLoginId(10001, false);   // 不存在返回 null
StpUtil.hasRole(10001, "super-admin");
StpUtil.hasPermission(10001, "user:add");
```

## 会话查询

```java
// 单账号的终端列表
List<SaTerminalInfo> list = StpUtil.getTerminalListByLoginId(10001);
for (SaTerminalInfo ter : list) {
    System.out.println("index=" + ter.getIndex() + ", type=" + ter.getDeviceType()
            + ", token=" + ter.getTokenValue() + ", time=" + ter.getCreateTime());
}

// 遍历系统全部会话
List<String> sessionIdList = StpUtil.searchSessionId("", 0, -1, false);
for (String sessionId : sessionIdList) {
    SaSession session = StpUtil.getSessionBySessionId(sessionId);
    List<SaTerminalInfo> terminalList = session.terminalListCopy();
    System.out.println("会话id：" + sessionId + "，共在 " + terminalList.size() + " 台设备登录");
}
```

⚠️ 遍历实现，百万会话下 Redis 模式取 10 条约 3.3s。要实时精确在线数请用 WebSocket。

## 全局侦听器

内置 `SaTokenListenerForLog`（配 `is-log: true` 生效）；只想覆盖部分方法就继承 `SaTokenListenerForSimple`。

```java
@Component   // 加了 @Component，SpringBoot 会自动注册
public class MySaTokenListener implements SaTokenListener {

    /** 每次登录时触发 */
    public void doLogin(String loginType, Object loginId, String tokenValue, SaLoginParameter loginParameter) { }

    /** 每次注销时触发 */
    public void doLogout(String loginType, Object loginId, String tokenValue) { }

    /** 每次被踢下线时触发 */
    public void doKickout(String loginType, Object loginId, String tokenValue) { }

    /** 每次被顶下线时触发 */
    public void doReplaced(String loginType, Object loginId, String tokenValue) { }

    /** 每次被封禁时触发 */
    public void doDisable(String loginType, Object loginId, String service, int level, long disableTime) { }

    /** 每次被解封时触发 */
    public void doUntieDisable(String loginType, Object loginId, String service) { }

    /** 每次二级认证时触发 */
    public void doOpenSafe(String loginType, String tokenValue, String service, long safeTime) { }

    /** 每次退出二级认证时触发 */
    public void doCloseSafe(String loginType, String tokenValue, String service) { }

    /** 每次创建 Session 时触发 */
    public void doCreateSession(String id) { }

    /** 每次注销 Session 时触发 */
    public void doLogoutSession(String id) { }

    /** 每次 Token 续期时触发（两个重载都要实现） */
    public void doRenewTimeout(String tokenValue, Object loginId, long timeout) { }
    public void doRenewTimeout(String loginType, Object loginId, String tokenValue, long timeout) { }
}
```

⚠️ **侦听器里的风险代码必须包 try-catch**，否则抛异常会强制中断 Sa-Token 的整个登录流程。

事件中心 API：

```java
SaTokenEventCenter.registerListener(listener);
SaTokenEventCenter.registerListenerList(list);
SaTokenEventCenter.removeListener(listener);
SaTokenEventCenter.removeListener(cls);
SaTokenEventCenter.getListenerList();
SaTokenEventCenter.setListenerList(list);
SaTokenEventCenter.clearListener();
SaTokenEventCenter.hasListener(listener);
```

## 全局过滤器

### 过滤器 vs 拦截器

**过滤器优点：** 更底层、执行时机更靠前（利于防渗透扫描）；能拦截静态资源；几乎所有 Web 框架都提供过滤器机制。

**过滤器缺点：**
1. 太底层，拿不到 `HandlerMethod` 对象 → **注解鉴权在过滤器里无法工作**。
2. 拦截太全面，需要额外处理 `/favicon.ico` 之类的特殊路由。
3. **Spring 中过滤器抛出的异常进不了全局 `@ExceptionHandler`，必须自己写异常处理。**

两者并存、按需选择。**过滤器默认关闭**，需手动注册。WebFlux 没有拦截器机制，**过滤器是唯一选择**。

```java
@Configuration
public class SaTokenConfigure {
    @Bean
    public SaServletFilter getSaServletFilter() {
        return new SaServletFilter()
            .addInclude("/**").addExclude("/favicon.ico")
            // 认证函数：每次请求执行
            .setAuth(obj -> {
                SaRouter.match("/**", "/user/doLogin", () -> StpUtil.checkLogin());
            })
            // 异常处理函数：必须写，因为过滤器异常进不了 @ExceptionHandler
            .setError(e -> {
                SaHolder.getResponse().setHeader("Content-Type", "application/json;charset=UTF-8");
                return JSONUtil.toJsonStr(SaResult.error(e.getMessage()));
            })
            // 前置函数：不受 include/exclude 限制，所有请求都会进入
            .setBeforeAuth(r -> {
                SaHolder.getResponse()
                    .setServer("sa-server")
                    .setHeader("X-Frame-Options", "SAMEORIGIN")
                    .setHeader("X-XSS-Protection", "1; mode=block")
                    .setHeader("X-Content-Type-Options", "nosniff");
            });
    }
}
```

⚠️ `setError` 的返回值是**以字符串写给前端的**，格式转换要自己做。
⚠️ `setBeforeAuth` 不受 include/exclude 限制 —— 这是设置安全响应头和 CORS 头的规范位置。

过滤器默认 order 是 `-100`，需要改就用 `FilterRegistrationBean`（SpringBoot 中 order 越小越先执行）：

```java
@Bean
public FilterRegistrationBean<SaServletFilter> getSaServletFilter() {
    FilterRegistrationBean<SaServletFilter> frBean = new FilterRegistrationBean<>();
    frBean.setFilter(new SaServletFilter().addInclude("/**").setAuth(obj -> { /* ... */ }));
    frBean.setOrder(-101);
    return frBean;
}
```

⚠️ `includeList`/`excludeList` 已改为 public 字段，对应的 getter 方法已移除。

## 多账号认证

`StpUtil` 只是对 `stpLogic` 的转发，换一个 `loginType` 就是一套全新的账号体系。

### 方式一：Kit 模式（推荐，无代码重复）

```java
/**
 * StpLogic 门面类，管理项目中所有的 StpLogic 账号体系
 */
public class StpKit {
    /** 默认原生会话对象 */
    public static final StpLogic DEFAULT = StpUtil.stpLogic;
    /** Admin 会话对象 */
    public static final StpLogic ADMIN = new StpLogic("admin");
    /** User 会话对象 */
    public static final StpLogic USER = new StpLogic("user");
}
```

```java
StpKit.ADMIN.login(10001);
StpKit.USER.login(10001);
StpKit.ADMIN.checkPermission("article:add");
StpKit.USER.getSession().set("name", "zhang");
```

### 方式二：复制 StpUtil 成 StpUserUtil

新建 `StpUserUtil.java`，复制 `StpUtil` 全部代码，改 `LoginType`：

```java
public class StpUserUtil {
    /** 账号体系标识 */
    public static final String TYPE = "user";   // 从 login 改为 user
    // 其它代码 ...
}
```

### 注解指定账号体系

默认注解只校验原生 `StpUtil`。`StpUserUtil.login()` 建立的会话**永远通不过**裸的 `@SaCheckLogin`。

```java
@SaCheckLogin(type = StpUserUtil.TYPE)   // type 默认 ""，代表原生 StpUtil
@RequestMapping("info")
public String info() { return "查询用户信息"; }
```

避免每处都写 `type=`，两种做法。其一是 Spring 注解合并：

```java
@PostConstruct
public void rewriteSaStrategy() {
    SaAnnotationStrategy.instance.getAnnotation = (element, annotationClass) ->
            AnnotatedElementUtils.getMergedAnnotation(element, annotationClass);
}
```

```java
@SaCheckLogin(type = "user")
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface SaUserCheckLogin { }
```

其二是自定义注解处理器（不依赖 Spring，性能更好，见下节）。

### 一个客户端同时登录两套账号

两个体系的 token 名都是 `satoken`，后者会**覆盖**前者，静默杀掉第一个会话。重写 `splicingKeyTokenName()`：

```java
public class StpUserUtil {
    public static StpLogic stpLogic = new StpLogic("user") {
        @Override
        public String splicingKeyTokenName() {
            return super.splicingKeyTokenName() + "-user";   // → satoken-user
        }
    };
}
```

### 拦截器里混合鉴权

```java
registry.addInterceptor(new SaInterceptor(handle -> {
    // 要求登录 Admin
    SaRouter.match("/art/getInfo").check(r -> StpUtil.checkLogin());
    // 要求同时登录 Admin 和 User
    SaRouter.match("/art/add").check(r -> { StpUtil.checkLogin(); StpUserUtil.checkLogin(); });
    // 要求任一登录即可
    SaRouter.match("/art/list").check(r -> {
        if(!StpUtil.isLogin() && !StpUserUtil.isLogin()) {
            throw new SaTokenException("请登录后再访问接口");
        }
    });
})).addPathPatterns("/**");
```

### ⚠️ LoginType 运行时不可更改

```java
// ❌ 严重错误
StpUtil.getStpLogic().setLoginType("user");
```

`LoginType` 只能在项目启动时指定。启动后修改会造成**线程安全问题和严重的逻辑问题**。

## 自定义注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface CheckAccount {
    String name();
    String pwd();
}
```

```java
@Component
public class CheckAccountHandler implements SaAnnotationHandlerInterface<CheckAccount> {

    /** 指定这个处理器要处理哪个注解 */
    @Override
    public Class<CheckAccount> getHandlerAnnotationClass() {
        return CheckAccount.class;
    }

    /** 每次请求校验注解时执行 */
    @Override
    public void checkMethod(CheckAccount at, AnnotatedElement element) {
        String name = SaHolder.getRequest().getParamNotNull("name");
        String pwd = SaHolder.getRequest().getParamNotNull("pwd");
        if(!(name.equals(at.name()) && pwd.equals(at.pwd()))) {
            throw new SaTokenException("账号或密码错误，未通过校验");
        }
    }
}
```

非 IOC 环境需手动注册：`SaAnnotationStrategy.instance.registerAnnotationHandler(new CheckAccountHandler());`

多账号体系专用注解（比 Spring 注解合并更好：不依赖 Spring、无递归性能开销、还能扩展行为）：

```java
@Component
public class SaUserCheckLoginHandler implements SaAnnotationHandlerInterface<SaUserCheckLogin> {
    @Override
    public Class<SaUserCheckLogin> getHandlerAnnotationClass() { return SaUserCheckLogin.class; }
    @Override
    public void checkMethod(SaUserCheckLogin at, AnnotatedElement element) {
        SaCheckLoginHandler._checkMethod(StpUserUtil.TYPE);
    }
}
```

## 权限缓存

`StpInterface` 每次鉴权都会调用，直接查库会打爆数据库。**按 `[账号id → 角色id → 权限列表]` 缓存，不要按 `[账号id → 权限列表]`** —— 后者改一个角色的权限要失效十万个账号的缓存，会引发缓存雪崩。

```java
@Component
public class StpInterfaceImpl implements StpInterface {

    @Override
    @SuppressWarnings("unchecked")
    public List<String> getPermissionList(Object loginId, String loginType) {
        List<String> list = new ArrayList<>();
        for (String roleId : getRoleList(loginId, loginType)) {
            List<String> permissionList = (List<String>) SaManager.getSaTokenDao()
                    .getObject("satoken:role-find-permission:" + roleId);
            if (permissionList == null) {
                permissionList = ...;   // 查库：该角色拥有的权限列表
                SaManager.getSaTokenDao().setObject(
                        "satoken:role-find-permission:" + roleId, permissionList, 60 * 60 * 24 * 30);
            }
            list.addAll(permissionList);
        }
        return list;
    }

    @Override
    @SuppressWarnings("unchecked")
    public List<String> getRoleList(Object loginId, String loginType) {
        List<String> roleList = (List<String>) SaManager.getSaTokenDao()
                .getObject("satoken:loginId-find-role:" + loginId);
        if (roleList == null) {
            roleList = ...;   // 查库：该账号拥有的角色列表
            SaManager.getSaTokenDao().setObject(
                    "satoken:loginId-find-role:" + loginId, roleList, 60 * 60 * 24 * 30);
        }
        return roleList;
    }
}
```

## 动态路由鉴权

**动态逻辑必须写在 lambda 内部** —— 外面的代码只在启动时执行一次。

```java
// ✅ 正确
registry.addInterceptor(new SaInterceptor(handle -> {
    SaRouter.match("/**").notMatch(excludePaths()).check(r -> StpUtil.checkLogin());
})).addPathPatterns("/**");

// ✅ 动态 path→权限 规则
registry.addInterceptor(new SaInterceptor(handle -> {
    Map<String, String> rules = getAuthRules();
    for (String path : rules.keySet()) {
        SaRouter.match(path, () -> StpUtil.checkPermission(rules.get(path)));
    }
})).addPathPatterns("/**");

// ❌ 反模式：excludePathPatterns 里调方法，永远不会动态
registry.addInterceptor(...).excludePathPatterns(excludePaths());
```

## CORS 跨域

官方 `cors-filter` 章节只有外链，规范代码在 SSO 章节里，用 `SaCorsHandleFunction` 策略 Bean：

```java
@Configuration
public class SaTokenConfigure {
    /** CORS 跨域处理策略 */
    @Bean
    public SaCorsHandleFunction corsHandle() {
        return (req, res, sto) -> {
            res.setHeader("Access-Control-Allow-Origin", "*")
               .setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE")
               .setHeader("Access-Control-Max-Age", "3600")
               .setHeader("Access-Control-Allow-Headers", "*");

            // 预检请求直接返回前端
            SaRouter.match(SaHttpMethod.OPTIONS)
                    .free(r -> System.out.println("--------OPTIONS预检请求，不做处理"))
                    .back();
        };
    }
}
```

⚠️ **跨域问题常伪装成鉴权问题**：「明明排除了路径却还是被拦截」「未能读取到有效Token」「vue 不开 proxy 时 token 不注入 Cookie」都可能是跨域导致的。

## 防火墙

内置防火墙在鉴权之前拦截危险请求。如 `/test//login`（双斜杠，经典的鉴权绕过向量）会被硬拦，返回纯文本 `非法请求：/test//login`。

默认 hook：

- `SaFirewallCheckHookForWhitePath` —— path 白名单放行
- `SaFirewallCheckHookForBlackPath` —— path 黑名单
- `SaFirewallCheckHookForPathDangerCharacter` —— path 危险字符
- `SaFirewallCheckHookForPathBannedCharacter` —— path 禁止字符
- `SaFirewallCheckHookForDirectoryTraversal` —— 目录遍历符检测
- `SaFirewallCheckHookForHost` —— Host 检测
- `SaFirewallCheckHookForHttpMethod` —— 请求 Method 检测
- `SaFirewallCheckHookForHeader` —— 请求头检测
- `SaFirewallCheckHookForParameter` —— 请求参数检测

```java
@PostConstruct
public void saTokenPostConstruct() {
    // 重设某个 hook 的配置
    SaFirewallCheckHookForBlackPath.instance.resetConfig("/abc");

    // 注册新 hook
    SaFirewallStrategy.instance.registerHook((req, res, extArg) -> {
        if(req.getParam("pwd") != null) {
            throw new FirewallCheckException("请求中不可包含 pwd 参数");
        }
    });

    // 移除默认 hook
    SaFirewallStrategy.instance.removeHook(SaFirewallCheckHookForHost.class);
}
```

IOC 方式注册（自动扫描）：`@Component public class XxxHook implements SaFirewallCheckHook { ... }`

防火墙拒绝响应默认不是格式化的 JSON，需自己指定：

```java
@PostConstruct
public void saTokenPostConstruct() {
    SaFirewallStrategy.instance.checkFailHandle = (e, req, res, extArg) -> {
        try {
            HttpServletResponse response = (HttpServletResponse) res.getSource();
            response.setContentType("application/json;charset=UTF-8");
            response.getWriter().print(SaResult.error(e.getMessage()).toString());
            response.getWriter().flush();
        } catch (IOException ex) {
            throw new RuntimeException(ex);
        }
    };
}
```

## 异步 / Mock 上下文

报错 `SaTokenContextException: SaTokenContext 上下文尚未初始化` 的根因：`StpUtil.isLogin()` 等 API 需要从 `HttpServletRequest` 读 token，而异步上下文没有 request。

异步上下文包括：`new Thread()`、`taskExecutor.execute()`、`@Async` 方法、`@Scheduled` 定时任务、MQ 消费者、MyBatis-Plus `insertFill`、非 HTTP 的 RPC（Dubbo）、`@PostConstruct`、`main()`。

```java
@RequestMapping("isLogin2")
public SaResult isLogin2() {
    String tokenValue = StpUtil.getTokenValue();     // 在主线程先取出来
    new Thread(() -> {
        SaTokenContextMockUtil.setMockContext(() -> {
            StpUtil.setTokenValueToStorage(tokenValue);
            System.out.println("是否登录：" + StpUtil.isLogin());
        });
    }).start();
    return SaResult.data(tokenValue);
}
```

响应式（WebFlux / Gateway）—— 调同步 API 前先 set 上下文：

```java
@Component
public class MyFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        try {
            SaReactorSyncHolder.setContext(exchange);
            System.out.println(StpUtil.isLogin());
        } finally {
            SaReactorSyncHolder.clearContext();
        }
        return chain.filter(exchange);
    }
}
```

响应式 Controller 里：

```java
@RequestMapping("isLogin2")
public SaResult isLogin2(ServerWebExchange exchange) {
    return SaReactorSyncHolder.setContext(exchange, () -> SaResult.data(StpUtil.getTokenInfo()));
}
```

相关辅助：`SaReactorHolder.getMonoExchange()`、`SaReactorHolder.sync(() -> {...})`（返回 `Mono<R>`）、`SaReactorSyncHolder.getExchange()`。

**判断原则：任何需要先从前端请求读数据的 API 都是"需要 Web 上下文"的**（`getLoginId()`、`getTokenValue()`）；而 `StpUtil.getLoginType()`、`SaManager.getConfig()` 与上下文无关。

## 加密工具

```java
// 摘要
SaSecureUtil.md5("123456");
SaSecureUtil.sha1("123456");
SaSecureUtil.sha256("123456");
SaSecureUtil.sha384("123456");

// 对称加密 AES
String ciphertext = SaSecureUtil.aesEncrypt(key, text);
String text2 = SaSecureUtil.aesDecrypt(key, ciphertext);

// Base64 / Base32
SaBase64Util.encode(text);   SaBase64Util.decode(base64Text);
SaBase32Util.encode(text);   SaBase32Util.decode(base32Text);

// TOTP（动态口令，用于 2FA）
String secretKey = SaTotpUtil.generateSecretKey();
String qrString  = SaTotpUtil.generateGoogleSecretKey("zhangsan", secretKey);
String code      = SaTotpUtil.generateTOTP(secretKey);
boolean isValid  = SaTotpUtil.validateTOTP(secretKey, code, 1);
```

⚠️ **`SaSecureUtil` 的 RSA 相关 API（`rsaEncryptByPublic` 等）已被官方标注"已过时"，`BCrypt` 已标 `@Deprecated`** —— 新代码不要用。需要更多算法用 Hutool-crypto。

密码存储推荐：`SaSecureUtil.md5(salt + password)`（加盐哈希）。

## 其它实用模式

**CSRF 防护** —— 二选一：配 `is-read-cookie: false` + localStorage/header 提交；或在 Session 上放 csrf_token：

```java
String csrfToken = StpUtil.getSession().get("csrf_token", () -> SaFoxUtil.getRandomString(60));
// 校验
String submitted = SaHolder.getRequest().getHeader("csrf_token");
if (submitted == null || !submitted.equals(StpUtil.getSession().get("csrf_token"))) {
    throw new SaTokenException("csrf_token 不匹配");
}
```

**RefreshToken 模式**（框架无原生支持）：`StpUtil.login()` 的 token 设短 `timeout`，再用 `SaTempUtil.createToken(10001, 2592000)` 作长效刷新 token；刷新接口里 `SaTempUtil.parseToken(refreshToken)` 后 `StpUtil.createLoginSession(userId)`。

**自定义 token 读取方式**：

```java
@Component
public class MyStpLogic extends StpLogic {
    public MyStpLogic() { super("login"); }
    @Override
    public String getTokenValue() {
        return SaHolder.getRequest().getHeader("my-token");
    }
}
```

**日志**：`is-log: true` 输出登录/注销/二级认证步骤。接入 Slf4j 就实现 `SaLog` 接口（`trace/debug/info/warn/error/fatal`）并加 `@Component`。临时打点：

```java
SaManager.getLog().debug("----- path={} token={}",
        SaHolder.getRequest().getRequestPath(), StpUtil.getTokenValue());
```
