# agent-skills

个人 **Agent Skills** 集合，遵循 [open agent skills 生态](https://github.com/vercel-labs/skills) 约定，可通过 `npx skills` 安装到任意支持的编码 Agent（Claude Code、Codex、Cursor、OpenCode、Gemini CLI 等 79 个）。

| Skill | 说明 |
|---|---|
| `art-design-pro` | Art Design Pro（Vue 3 + TS + Vite + Element Plus 中后台模板）开发指南：项目结构、路由菜单、主题与系统配置、环境变量、i18n、权限控制、`useTable`、`ArtSearchBar`/`ArtSvgIcon` |
| `mybatis-plus` | MyBatis-Plus 3.5.17 开发指南：CRUD 接口、条件构造器、插件体系、注解与配置、代码生成器、报错速查 |
| `sa-token` | Sa-Token 认证鉴权框架开发指南：登录鉴权、注解与路由拦截、Redis 集成、微服务网关鉴权、SSO/OAuth2、报错速查 |

## 安装

```bash
# 全部安装（交互式选择目标 Agent）
npx skills add Dr-kyle/agent-skills

# 只装某一个
npx skills add Dr-kyle/agent-skills --skill mybatis-plus

# 指定 Agent + 免交互（CI 友好）
npx skills add Dr-kyle/agent-skills --skill '*' -a claude-code -y

# 装到用户全局目录，跨项目可用
npx skills add Dr-kyle/agent-skills --skill '*' -g

# 先看有哪些 skill，不安装
npx skills add Dr-kyle/agent-skills --list
```

多个 Agent 一次装：

```bash
npx skills add Dr-kyle/agent-skills --skill '*' -a claude-code -a codex -a cursor
```

各 Agent 的安装路径不同（Claude Code 为 `.claude/skills/`，Codex/Cursor/OpenCode 等为 `.agents/skills/`），由 CLI 自动处理。

## 不安装直接用

```bash
npx skills use Dr-kyle/agent-skills@mybatis-plus | claude
```

## 本地开发

已克隆到本地时，可直接从路径安装，无需联网：

```bash
# 在目标项目里执行
npx skills add ../agent-skills --skill '*' -a claude-code -y
```

⚠️ `skills add` 是**复制**而非软链接，改完本仓库的内容后需要重新执行安装命令，目标项目才会同步。

⚠️ 安装会在目标项目根目录生成 `skills-lock.json`（记录来源与内容哈希），建议在目标项目里把它和 `.claude/skills/` 一起加入 `.gitignore`——本仓库才是唯一事实来源。

## 新增一个 skill

```bash
npx skills init my-skill        # 生成 SKILL.md 模板
```

目录结构：

```
my-skill/
├── SKILL.md          # 必需，YAML frontmatter 需含 name 与 description
└── references/       # 可选，大块参考资料拆到这里
```

`SKILL.md` 的 frontmatter 至少包含：

```yaml
---
name: my-skill
description: "一句话说明覆盖什么、什么情况下触发。"
---
```

两条经验：

⚠️ `description` 是 Agent 选择 skill 的**唯一依据**。必须写明**触发场景**（"Triggers include..."、"用于……时"），而不只是罗列功能——只写功能会导致该触发时不触发。

⚠️ `SKILL.md` 会被**完整载入上下文**，过长会挤占预算。内容多时拆进 `references/`，在 `SKILL.md` 末尾用一张索引表指明"需要 X 时读 references/y.md"，让 Agent 按需加载。
