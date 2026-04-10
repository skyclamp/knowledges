# Prompt 01: 分析 GitHub Copilot Provider 的认证流程

## 任务

阅读以下源代码文件，分析 opencode 中 GitHub Copilot provider 的完整认证（Auth）流程，输出一份详细的技术文档。

## 需要阅读的文件

按顺序阅读：

1. `packages/opencode/src/auth/index.ts` — Auth 基础服务，定义 OAuth/API/WellKnown 三种认证类型的 Schema 和 CRUD 操作
2. `packages/opencode/src/provider/auth.ts` — ProviderAuth 服务，定义了 provider 级别的认证方法注册、authorize/callback 流程
3. `packages/opencode/src/plugin/github-copilot/copilot.ts` — CopilotAuthPlugin，Copilot 的具体认证实现（OAuth Device Flow）
4. `packages/opencode/src/plugin/index.ts` — Plugin 系统，了解 CopilotAuthPlugin 如何被注册为内置插件

## 分析要求

请产出一份文档，包含以下部分：

### 1. Auth 数据模型
- `Auth.Oauth`、`Auth.Api`、`Auth.WellKnown` 三种类型的字段和用途
- Auth 数据的持久化方式（文件路径、权限等）
- Copilot 使用的是哪种类型

### 2. OAuth Device Flow 完整流程
- 从用户触发登录到获取 token 的全部步骤
- 涉及的 GitHub API 端点（`/login/device/code`、`/login/oauth/access_token`）
- OAuth Client ID（硬编码值）
- 请求参数和响应结构
- Polling 机制（interval、slow_down 处理、safety margin）
- 最终 token 如何存储（refresh/access/expires 的含义）

### 3. GitHub Enterprise 支持
- 部署类型选择（github.com vs enterprise）
- Enterprise URL 如何影响 API 端点
- `normalizeDomain` 和 `getUrls` 函数的逻辑
- `base()` 函数如何确定 Copilot API base URL

### 4. Auth Loader（运行时 token 使用）
- `auth.loader` 函数如何被调用
- 返回的 `baseURL`、`apiKey`、`fetch` 三个字段的作用
- 自定义 fetch 中的请求头改造：
  - `Authorization: Bearer ${info.refresh}` 的使用
  - `x-initiator`（user vs agent）的判断逻辑
  - `Openai-Intent: conversation-edits`
  - `Copilot-Vision-Request` header
  - 为什么删除 `x-api-key` 和 `authorization`（小写）
- Vision 请求和 Agent 请求的检测逻辑（Completions API / Responses API / Messages API 三种格式）

### 5. ProviderAuth 服务层
- `methods()` — 如何列出可用的认证方法
- `authorize()` — 如何发起 OAuth 授权
- `callback()` — 如何处理 OAuth 回调
- 表单验证逻辑（`prompts` 中的 `when` 条件和 `validate`）

### 6. Plugin 注册机制
- `CopilotAuthPlugin` 如何通过 `INTERNAL_PLUGINS` 注册
- `Hooks` 结构中 `auth` 和 `provider` 字段的用途

## 输出格式

- 文档标题：`GitHub Copilot Provider 认证流程详解`
- 使用中文
- 每个代码引用标注文件路径和行号，格式如 `copilot.ts:206-217`
- 包含流程图（用 mermaid 语法）
- 保存到 `/Users/wenkai/workspace/knowledges/opencode/doc-01-copilot-auth-flow.md`
