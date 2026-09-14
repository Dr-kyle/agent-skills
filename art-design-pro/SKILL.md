---
name: art-design-pro
description: "Use this skill for any task involving development with Art Design Pro — a Vue 3 + TypeScript + Vite + Element Plus enterprise mid-back-office template. Covers project structure, routing & menu config, system & theme settings, env variables, i18n, permission control, the useTable composable, ArtSearchBar & ArtSvgIcon components, build & deploy, and code conventions. Triggers include requests to add a page/route, configure menus, set up tables with pagination/search, build search forms, manage permissions/roles, customize themes, or scaffold code following the project's conventions."
---

# Art Design Pro 开发指南

Art Design Pro 是一款专注于用户体验和视觉设计的企业级中后台解决方案。技术栈：**Vue 3、TypeScript、Vite、Element Plus、Tailwind CSS、Sass**。

> 官方文档：<https://www.artd.pro/docs/zh/> · 源码：<https://github.com/Daymychen/art-design-pro>
> 注意：文档可能随版本更新，关键 API 以项目实际代码为准。

## 环境要求与快速启动

- Node.js **20.19.0+**，推荐使用 **pnpm** 包管理器。
- 安装依赖：`pnpm install`（失败可改用 `pnpm install --ignore-scripts`）。
- 启动：`pnpm dev`，默认访问 `http://localhost:3006`。
- 构建：`pnpm build`（先跑 `vue-tsc --noEmit` 类型检查再构建），产物在 `dist/`。
- 精简演示项目：`pnpm clean:dev`（**不可逆**，会删除演示页面、Mock 数据、多语言文件等，执行前先备份）。

## 项目结构（src/ 关键目录）

```
src/
├── api/                    # API 接口定义（auth.ts 认证、system-manage.ts 系统管理）
├── assets/
│   ├── styles/             # 全局样式（core 系统级 / custom 业务级 / index.scss 入口 / el-ui.scss 组件调优）
│   └── svg/loading.ts      # 加载动画
├── components/
│   ├── business/           # 业务组件
│   └── core/               # 系统级通用组件（base/forms/tables/charts/layouts/widget 等）
├── config/
│   ├── index.ts            # 全局配置入口（系统名称、主题、菜单布局、主色等）
│   └── setting.ts          # 系统设置配置
├── directives/             # 自定义指令（core/auth.ts、core/roles.ts、business/）
├── enums/                  # 枚举（appEnum.ts 主题/语言、formEnum.ts 表单）
├── hooks/core/             # 核心 Hooks（useTable、useTheme、useAuth 等）
├── locales/                # 国际化（langs/zh.json、langs/en.json）
├── mock/                   # Mock 数据
├── router/
│   ├── core/               # 组件加载器、菜单处理器、路由转换器等
│   ├── guards/             # 全局路由守卫
│   ├── modules/            # 各业务模块路由
│   └── routes/
│       ├── asyncRoutes.ts  # 动态路由（业务页面）
│       └── staticRoutes.ts # 静态路由（登录/404 等）
├── store/modules/          # Pinia（menu、setting、table、user、worktab）
├── types/                  # TS 类型（common/response.ts 响应格式等）
├── utils/
│   ├── http/               # HTTP 请求封装（error.ts、status.ts）
│   ├── form/               # 表单工具
│   ├── table/              # 表格工具（tableConfig.ts 字段映射）
│   └── ui/                 # UI 工具（colors.ts 颜色、iconify-loader.ts 图标）
└── views/                  # 页面组件（新增页面放这里）
```

## 新增页面与注册路由

### 1. 创建页面

在 `src/views/` 下创建 `.vue` 文件，根元素使用 `class="page-content"` 可将盒子最小高度撑满屏幕剩余高度：

```vue
<template>
  <div class="page-content">
    <h1>test page</h1>
  </div>
</template>
```

### 2. 注册动态路由（带菜单）

编辑 `src/router/routes/asyncRoutes.ts`。一级路由：

```ts
export const asyncRoutes: MenuListType[] = [
  {
    path: "/test/index",
    name: "Test",
    component: "/test/index",          // 对应 src/views/test/index.vue
    meta: {
      title: "测试页",
      keepAlive: true,
    },
  },
];
```

多级路由（多级菜单）：

```ts
export const asyncRoutes: MenuListType[] = [
  {
    name: "Form",
    path: "/form",
    component: "/index/index",
    meta: { title: "表单", icon: "", keepAlive: false },
    children: [
      {
        path: "basic",                  // 子路径不需要前导 /
        name: "Basic",
        component: "/form/basic",
        meta: { title: "基础表单", keepAlive: true },
      },
    ],
  },
];
```

### 3. 静态路由（无需权限，登录页/404 等）

编辑 `src/router/routes/staticRoutes.ts`：

```ts
export const staticRoutes: AppRouteRecordRaw[] = [
  {
    path: "/test",
    name: "Test",
    component: () => import("@views/test/index.vue"),
    meta: { title: "测试页面", isHideTab: true, setTheme: true },
  },
];
```

> 若静态路由同时在动态路由表配置过，需移除静态路由中的配置，因为动态路由会自动注册。

### 4. 主页与 iframe 配置

- 主页路径：`src/router/index.ts` 中 `export const HOME_PAGE_PATH = ""`（默认取菜单第一个有效路径）。
- iframe 内嵌外部页面：设置 `meta.link` + `meta.isIframe: true`，`component` 留空：

```ts
{
  path: '/outside/iframe/elementui',
  name: 'ElementUI',
  component: '',
  meta: {
    title: 'menus.widgets.elementUI',
    keepAlive: false,
    link: 'https://element-plus.org/zh-CN/component/overview.html',
    isIframe: true
  }
}
```

## 路由 meta 字段说明

```ts
meta: {
  title: string;            // 路由标题（必需）
  icon?: string;            // 路由图标（Iconify，如 "ri:home-line"）
  showBadge?: boolean;      // 是否显示徽章
  showTextBadge?: string;   // 文本徽章
  isHide?: boolean;         // 是否在菜单中隐藏
  isHideTab?: boolean;      // 是否在标签页中隐藏
  link?: string;            // 外部链接
  isIframe?: boolean;       // 是否为 iframe
  keepAlive?: boolean;      // 是否缓存页面
  authList?: { title: string; authMark: string }[];  // 操作权限列表
  isFirstLevel?: boolean;   // 是否一级菜单（自动识别，无需配置）
  roles?: string[];         // 角色权限（前端权限模式）
  fixedTab?: boolean;       // 是否固定标签页
  isFullPage?: boolean;     // 是否为全屏页面
  activePath?: string;      // 手动指定当前激活菜单路径
}
```

## 权限管理

权限模式通过根目录 `.env` 的 `VITE_ACCESS_MODE` 切换：
- `frontend`：前端控制模式，基于 `asyncRoutes.ts` 中 `meta.roles` 与登录接口返回的角色匹配过滤；未设置 `roles` 则所有用户可访问。
- `backend`：后端控制模式，登录后由后端返回菜单列表，前端校验后动态注册路由。**后端菜单结构必须与前端 `MenuListType` 一致**，否则路由注册失败。

### 按钮权限

- `useAuth().hasAuth(authMark)` 控制按钮显示（前端模式需登录接口返回权限码，后端模式需菜单带 `authList`）：

```vue
<ElButton v-if="hasAuth('add')">添加</ElButton>
```

- 自定义指令 `v-auth="'add'"`：后端模式下基于 `authList` 的 `authMark` 控制按钮。
- 自定义指令 `v-roles="['R_SUPER', 'R_ADMIN']"` 或 `v-roles="'R_ADMIN'"`：基于用户接口返回的 `roles` 控制。

## 表格开发（useTable）

`useTable` 是项目最核心的组合式函数，封装数据获取、智能缓存（LRU）、分页、防抖搜索、多种刷新策略。位置：`src/hooks/core/useTable.ts`。

### 基础用法

```vue
<template>
  <ArtTable
    :loading="loading"
    :data="data"
    :columns="columns"
    :pagination="pagination"
    @pagination:size-change="handleSizeChange"
    @pagination:current-change="handleCurrentChange"
  />
</template>

<script setup lang="ts">
import { useTable } from "@/hooks/core/useTable";
import { fetchGetUserList } from "@/api/system-manage";

const {
  data, loading, columns, pagination,
  handleSizeChange, handleCurrentChange,
} = useTable({
  core: {
    apiFn: fetchGetUserList,
    apiParams: { current: 1, size: 20 },
    columnsFactory: () => [
      { prop: "id", label: "ID" },
      { prop: "name", label: "姓名" },
    ],
  },
});
</script>
```

### 常用配置项

| 分组 | 配置 | 默认 | 说明 |
|---|---|---|---|
| core | `apiFn` | 必填 | API 请求函数 |
| core | `apiParams` | `{}` | 默认请求参数 |
| core | `excludeParams` | `[]` | 排除的参数字段 |
| core | `immediate` | `true` | 是否立即加载数据 |
| core | `columnsFactory` | - | 列配置工厂函数 |
| transform | `dataTransformer` | - | 数据转换函数 |
| transform | `responseAdapter` | 默认适配器 | 自定义后端响应格式 |
| performance | `enableCache` | `false` | 是否启用缓存 |
| performance | `cacheTime` | `300000` | 缓存时间(ms) |
| performance | `debounceTime` | `300` | 防抖延迟(ms) |
| performance | `maxCacheSize` | `50` | 最大缓存条数 |
| hooks | `onSuccess` / `onError` / `onCacheHit` / `resetFormCallback` | - | 生命周期回调 |

### 常用返回值/方法

- 数据：`data`、`loading`、`error`、`hasData`、`isEmpty`
- 分页：`pagination`、`handleSizeChange`、`handleCurrentChange`
- 搜索：`searchParams`（配合 `Object.assign(searchParams, formData)` 赋值）、`resetSearchParams`
- 数据操作：`getData`（回到第一页）、`getDataDebounced`（防抖）、`fetchData`（手动，配合 `immediate: false`）、`clearData`
- **刷新策略**（CRUD 后使用）：`refreshCreate()` 新增后回到第一页；`refreshUpdate()` 编辑后保持当前页；`refreshRemove()` 删除后智能处理页码；`refreshData()` 手动刷新按钮（清空所有缓存）；`refreshSoft()` 定时刷新（仅清空当前搜索条件缓存）
- 缓存：`cacheInfo`、`clearCache(strategy)`、`clearExpiredCache`
- 动态列：`addColumn`、`removeColumn`、`toggleColumn`、`updateColumn`、`resetColumns`

`clearCache` 策略来自 `CacheInvalidationStrategy` 枚举：`CLEAR_ALL` / `CLEAR_CURRENT` / `CLEAR_PAGINATION` / `KEEP_ALL`。

### 表格后端字段映射

后端返回的列表字段名可通过 `src/utils/table/tableConfig.ts` 自定义映射：

```ts
export const tableConfig = {
  recordFields: ["list", "data", "records", "items", "result", "rows"],  // 列表数据
  totalFields: ["total", "count"],                                        // 总条数
  currentFields: ["current", "page", "pageNum"],                          // 当前页码
  sizeFields: ["size", "pageSize", "limit"],                              // 每页大小
  paginationKey: { current: "current", size: "size" },                    // 请求分页参数名
};
```

> 项目约定：useTable 传递分页参数时用 `current` 和 `size`。后端字段不在列表中时，在对应数组追加自定义字段名即可。

## 搜索表单（ArtSearchBar）

`ArtSearchBar` 是高度可配置的表单搜索组件，支持 20+ 种控件、动态显隐、表单验证、插槽、响应式。

### 基础用法

```vue
<template>
  <ArtSearchBar
    v-model="formData"
    :items="formItems"
    @search="handleSearch"
    @reset="handleReset"
  />
</template>

<script setup>
const formData = ref({ name: "", status: "" });

const formItems = [
  { label: "用户名", key: "name", type: "input", placeholder: "请输入用户名" },
  {
    label: "状态",
    key: "status",
    type: "select",
    props: { options: [{ label: "启用", value: "1" }, { label: "禁用", value: "0" }] },
  },
];
</script>
```

### 常用表单项 type

- 输入类：`input`、`number`、`input`(textarea)
- 选择类：`select`（props.options）、`cascader`、`treeselect`
- 日期时间：`datetime`（props.type: `date`/`daterange`，`valueFormat: 'YYYY-MM-DD'`）、`timepicker`
- 其他：`switch`、`radiogroup`、`checkboxgroup`、`rate`、`slider`
- 自定义：`type: () => h(CustomComponent, {...})` 渲染函数，或用 `#插槽名` 插槽

### 验证与动态控制

```ts
// 表单验证（rules 格式与 Element Plus Form 一致）
const rules = { name: [{ required: true, message: "请输入用户名", trigger: "blur" }] };
await searchBarRef.value.validate();   // 返回 Promise<boolean>

// 动态显示隐藏
{ label: "高级选项", key: "advanced", type: "input", hidden: !showAdvanced.value }
```

### 主要 Props

| 参数 | 说明 | 默认 |
|---|---|---|
| `modelValue` | 表单数据对象 | `{}` |
| `items` | 表单项配置数组 `SearchFormItem[]` | `[]` |
| `span` | 每项栅格数 | `6` |
| `gutter` | 栅格间隔 | `12` |
| `labelPosition` | 标签位置 `'left'|'right'|'top'` | `'right'` |
| `labelWidth` | 标签宽度 | `'70px'` |
| `defaultExpanded` | 默认展开 | `false` |
| `showExpand` / `showReset` / `showSearch` / `disabledSearch` | 显示控制 | `true` |

Events：`search`、`reset`。Methods：`validate`、`reset`。Slots：`[key]`（参数 `{ item, modelValue }`）。

> 表单项 `key` 必须唯一；`props` 会直接透传给对应 Element Plus 组件。

## 图标（Iconify，v3.0）

- 全部采用 **Remix Icon**（`ri:` 前缀），通过 `ArtSvgIcon` 组件使用，样式用 Tailwind 类控制：

```vue
<ArtSvgIcon icon="ri:home-line" />
<ArtSvgIcon icon="ri:user-line" class="text-2xl" />          <!-- 大小 -->
<ArtSvgIcon icon="ri:heart-fill" class="text-red-500" />     <!-- 颜色 -->
<ArtSvgIcon icon="ri:home-line" class="text-theme" />        <!-- 主题色 -->
```

- 离线图标（内网部署）：安装 `@iconify-json/ri` 等图标集包 → 在 `src/utils/ui/iconify-loader.ts` 中 `addCollection()` 注册 → 在 `src/main.ts` 中 `import "@utils/ui/iconify-loader"`。新增图标集后需重启开发服务器。

## 主题与样式

主题基于 **Tailwind CSS v4**，配置文件：`src/assets/styles/core/tailwind.css`。核心原则：**优先使用 CSS 变量和 Tailwind 工具类，避免硬编码颜色**。

- 语义化 CSS 变量（自动适配 Light/Dark）：

```css
color: var(--art-gray-100/500/900);      /* 文字色 9 个层级 */
border: 1px solid var(--default-border);  /* 边框 */
background: var(--default-bg-color);      /* 页面底色 */
background: var(--default-box-color);     /* 卡片/容器背景 */
background: var(--art-hover-color);       /* 悬停 */
color: var(--art-primary/success/warning/error/info/danger); /* 主题色 */
background: var(--el-color-primary-light-1..9);  /* Element 主题色变体 */
```

- Tailwind 快捷工具类：`text-g-900` / `bg-box` / `bg-primary` / `flex-c`(flex+居中) / `flex-cb`(flex+两端对齐) / `tad-200`(transition-all 200ms) / `border-full-d` / `c-p`(cursor-pointer) / `rounded-custom-sm`。
- 主题切换（JS）：`useTheme().switchThemeStyles(SystemThemeEnum.DARK | LIGHT | AUTO)`。
- 主题色配置：`src/config/index.ts` 的 `systemMainColor` 数组；动态设置 `setElementThemeColor("#5D87FF")`（来自 `@/utils/ui/colors`）。颜色工具还有 `hexToRgba`、`getLightColor`、`getDarkColor`、`colourBlend` 等。

## 系统配置（src/config/index.ts）

- 系统名称：`appConfig.systemInfo.name`。
- Logo：`src/components/core/base/ArtLogo.vue` 中 `src="@imgs/common/logo.png"`，只改资源路径即可。
- 菜单布局：`menuLayoutList`（Left 左侧 / Top 顶部 / Mixed 混合 / Dual Column 双列）。
- 菜单主题：`themeList`、`darkMenuStyles`（各色值配置）。
- 主题列表：`settingThemeList`、`systemThemeStyles`。
- 系统主色：`systemMainColor`。

## 环境变量（.env 系列）

- `.env` 通用、`.env.development` 开发、`.env.production` 生产；自定义变量需以 **`VITE_`** 开头，代码中用 `import.meta.env.VITE_XXX` 访问。
- 关键变量：`VITE_PORT`（端口 3006）、`VITE_BASE_URL`（部署前缀）、`VITE_ACCESS_MODE`（权限模式 frontend/backend）、`VITE_WITH_CREDENTIALS`、`VITE_OPEN_ROUTE_INFO`、`VITE_LOCK_ENCRYPT_KEY`。

## 构建与部署

- `pnpm build` 生成 `dist/`，完整版约 8.4MB、精简版约 4.7MB，默认开启 gzip。
- 部署到非根目录：改 `.env.production` 的 `VITE_BASE_URL = /art-design-pro/`，并在 nginx 配置 alias：

```nginx
server {
  location /art-design-pro {
    alias /usr/local/nginx/html/art-design-pro;
    index index.html index.htm;
  }
}
```

## 国际化

- 基于 `vue-i18n`，配置在 `src/locales/index.ts`，语言包 `langs/zh.json`、`langs/en.json`。
- 模板中使用 `{{ $t('setting.color.title') }}`；获取当前语言 `const { locale } = useI18n()`。
- 新增语言：在 `index.ts` 的 `messages` 增加配置，在 `langs/` 新建语言文件。

## API 响应格式

默认响应结构定义在 `src/types/common/response.ts`：

```ts
interface BaseResponse<T = unknown> {
  code: number;  // 状态码
  msg: string;   // 消息
  data: T;       // 数据
}
```

网络请求**默认返回 `data` 中的数据**而非整个响应体。错误处理通过 `HttpError` 捕获：

```ts
try {
  const { token, refreshToken } = await fetchLogin({ userName, password });
} catch (error) {
  if (error instanceof HttpError) {
    // error.code 区分状态码处理
  }
}
```

## 常见坑

1. **页面切换空白**：开启了路由切换动画时，Vue `<Transition>` 要求页面组件必须有**单个根元素**（注释节点也算根）。将内容包在一个容器 div 内。
2. **点击菜单页面自动刷新**：Vite 首次使用某些组件库子模块（如 `element-plus/es/components/tooltip/style/index`）会触发依赖预构建刷新。把提示的依赖加到 `vite.config.ts` 的 `optimizeDeps.include` 后重启开发服务器。仅开发环境出现。
3. **路由配置错误**：浏览器控制台会给出详细错误提示，F12 查看。
4. **精简脚本不可逆**：`pnpm clean:dev` 前先提交或备份。

## 常用脚本命令

| 命令 | 说明 |
|---|---|
| `pnpm dev` | 启动开发服务器（自动开浏览器） |
| `pnpm build` | 类型检查 + 构建 |
| `pnpm serve` | 预览构建产物 |
| `pnpm lint` / `pnpm fix` | ESLint 检查 / 自动修复 |
| `pnpm lint:prettier` | Prettier 格式化 |
| `pnpm lint:stylelint` | Stylelint 检查修复 |
| `pnpm commit` | Commitizen 规范化提交（配合 git-cz） |

## 提交规范（cz-git）

提交类型：`feat` 新增功能、`fix` 修复缺陷、`docs` 文档变更、`style` 代码格式、`refactor` 重构、`perf` 性能优化、`test` 测试、`build` 构建/依赖变更、`ci` CI 配置、`revert` 回滚、`chore` 工具变更、`wip` 进行中。

## 同步上游更新

```bash
git remote add upstream https://github.com/Daymychen/art-design-pro
git fetch upstream
git checkout main
git merge upstream/main   # 解决冲突后提交
```
