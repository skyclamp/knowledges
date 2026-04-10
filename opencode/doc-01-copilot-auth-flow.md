# GitHub Copilot Provider 认证流程详解

本文基于以下代码实现分析：
- packages/opencode/src/auth/index.ts
- packages/opencode/src/provider/auth.ts
- packages/opencode/src/plugin/github-copilot/copilot.ts
- packages/opencode/src/plugin/index.ts

并补充引用了两个调用链定义文件：
- packages/opencode/src/provider/provider.ts
- packages/plugin/src/index.ts

## 1. Auth 数据模型

### 1.1 三种 Auth 类型与字段

Auth 服务把认证信息定义为带 `type` 判别字段的 Union（oauth/api/wellknown）。

1. `Auth.Oauth`
- 字段：`type`=`oauth`、`refresh`、`access`、`expires`、可选 `accountId`、可选 `enterpriseUrl`
- 用途：存储 OAuth 令牌与扩展上下文（账号、企业域）
- 引用：packages/opencode/src/auth/index.ts:15-22

2. `Auth.Api`
- 字段：`type`=`api`、`key`、可选 `metadata`
- 用途：存储 API Key 型认证
- 引用：packages/opencode/src/auth/index.ts:24-28

3. `Auth.WellKnown`
- 字段：`type`=`wellknown`、`key`、`token`
- 用途：存储“已知键位”的 token（通常用于特定 provider 的固定键语义）
- 引用：packages/opencode/src/auth/index.ts:30-34

Union 与 Zod 适配：
- `Info = Union([Oauth, Api, WellKnown])`，并通过 `zod(_Info)` 暴露 zod schema。
- 引用：packages/opencode/src/auth/index.ts:36-38

### 1.2 持久化方式（文件路径、权限）

- 持久化文件：`path.join(Global.Path.data, "auth.json")`
- 引用：packages/opencode/src/auth/index.ts:10

- 写入权限：`0o600`（仅文件所有者可读写）
- `set()` 写入时使用 0o600
- `remove()` 重写文件时也使用 0o600
- 引用：packages/opencode/src/auth/index.ts:69-77,79-85

- 读取容错与解码：
- `all()` 读取 JSON，失败时回退 `{}`
- 通过 `Schema.decodeUnknownOption(Info)` 过滤非法条目
- 引用：packages/opencode/src/auth/index.ts:58,60-63

- key 规范化：`set/remove` 会去掉 provider key 末尾 `/`，并清理同义 key（`key`、`norm`、`norm/`）
- 引用：packages/opencode/src/auth/index.ts:70-74,80-83

### 1.3 Copilot 使用哪种类型

GitHub Copilot 在授权成功后写入的是 `type: "oauth"`，字段包括 `access`、`refresh`、`expires`，并可能附带 `enterpriseUrl`。
- 写入逻辑：packages/opencode/src/provider/auth.ts:216-224
- Copilot callback 返回 OAuth 结果：packages/opencode/src/plugin/github-copilot/copilot.ts:258-277

## 2. OAuth Device Flow 完整流程

### 2.1 总体时序

```mermaid
flowchart TD
  A[用户选择 GitHub Copilot 登录方法] --> B[ProviderAuth.authorize]
  B --> C[调用 CopilotAuthPlugin.methods[oauth].authorize]
  C --> D[POST /login/device/code 获取 device_code+user_code]
  D --> E[返回 verification_uri + Enter code 指令]
  E --> F[用户在浏览器输入 user_code 完成授权]
  F --> G[ProviderAuth.callback]
  G --> H[调用 OAuthResult.callback 开始轮询]
  H --> I[POST /login/oauth/access_token]
  I -->|authorization_pending| H
  I -->|slow_down| H
  I -->|access_token| J[构造 success(refresh/access/expires)]
  J --> K[Auth.set(providerID, oauth信息)]
  K --> L[运行时 provider 通过 auth.loader 注入请求头]
```

关键入口：
- OAuth 方法定义与 authorize/callback 实现：packages/opencode/src/plugin/github-copilot/copilot.ts:154-308
- ProviderAuth 编排 authorize/callback：packages/opencode/src/provider/auth.ts:165-190,192-226

### 2.2 Device Code 申请阶段

Copilot 插件在 `authorize(inputs)` 中执行：

1. 解析部署类型
- 默认 `github.com`
- 如果 `deploymentType === "enterprise"`，读取 `enterpriseUrl` 并归一化为 domain
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:195-202

2. 生成 OAuth 端点
- `getUrls(domain)` =>
  - `https://${domain}/login/device/code`
  - `https://${domain}/login/oauth/access_token`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:19-23,204

3. 请求 Device Code
- 端点：`POST /login/device/code`
- Header：`Accept: application/json`、`Content-Type: application/json`、`User-Agent`
- Body：`{ client_id: CLIENT_ID, scope: "read:user" }`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:206-217

4. Client ID（硬编码）
- `CLIENT_ID = "Ov23li8tweQw6odWQebz"`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:11

5. 响应结构（代码中显式声明）
- `verification_uri`、`user_code`、`device_code`、`interval`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:223-228

6. 返回给上层的授权信息
- `url = verification_uri`
- `instructions = Enter code: ${user_code}`
- `method = auto`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:230-234

### 2.3 Access Token 轮询阶段

在返回对象的 `callback()` 里进入 `while(true)`：

1. 轮询请求
- 端点：`POST /login/oauth/access_token`
- Body：`client_id`、`device_code`、`grant_type = urn:ietf:params:oauth:grant-type:device_code`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:236-248

2. 轮询响应结构
- `access_token?`、`error?`、`interval?`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:252-256

3. 成功条件
- 当 `data.access_token` 存在，返回 `type: success`
- `refresh = access_token`
- `access = access_token`
- `expires = 0`
- enterprise 时附带 `enterpriseUrl = domain`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:258-277

4. Polling 机制
- 安全边距常量：`OAUTH_POLLING_SAFETY_MARGIN_MS = 3000`
- 常规 pending：睡眠 `deviceData.interval * 1000 + safety_margin`
- `slow_down`：
  - 默认按 RFC 8628 增加 5 秒：`(deviceData.interval + 5) * 1000`
  - 若服务端返回 `interval`，优先使用该值（秒）
  - 最终都再加 safety margin
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:12-14,280-299,303

5. 失败处理
- HTTP 非 2xx：直接 `{ type: "failed" }`
- 有其它 `error`：`{ type: "failed" }`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:250,301

### 2.4 token 最终存储语义

ProviderAuth 在 `callback()` 中接收插件返回值并入库：
- 若返回包含 `refresh` 字段，写入 Auth.Oauth
- 字段：`access`、`refresh`、`expires` + 额外扩展字段（如 `enterpriseUrl`）
- 引用：packages/opencode/src/provider/auth.ts:204-224

对于 Copilot：
- `refresh` 与 `access` 被设置为同一个 `access_token`
- `expires = 0` 表示当前实现未使用过期时间驱动刷新流程（可理解为“非时间驱动续签”）
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:268-270

## 3. GitHub Enterprise 支持

### 3.1 部署类型选择

Copilot 登录表单提供：
- `GitHub.com`（value=`github.com`）
- `GitHub Enterprise`（value=`enterprise`）
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:160-173

### 3.2 Enterprise URL 如何影响 OAuth 端点

- 当用户选 enterprise，`enterpriseUrl` 由文本输入提供
- 经过 `normalizeDomain()` 后赋值给 `domain`
- `getUrls(domain)` 用该 domain 生成 `device/code` 与 `access_token` 端点
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:177-202,19-23

### 3.3 normalizeDomain 与 getUrls 逻辑

- `normalizeDomain(url)`：
  - 去掉前缀 `http://` 或 `https://`
  - 去掉末尾 `/`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:15-17

- `getUrls(domain)`：
  - `DEVICE_CODE_URL = https://${domain}/login/device/code`
  - `ACCESS_TOKEN_URL = https://${domain}/login/oauth/access_token`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:19-23

### 3.4 base() 如何确定 Copilot API Base URL

- 非 enterprise：`https://api.githubcopilot.com`
- enterprise：`https://copilot-api.${normalizeDomain(enterpriseUrl)}`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:26-28

此 base URL 被两处使用：
- provider model 拉取：`CopilotModels.get(base(...), headers, provider.models)`
- auth.loader 返回 `baseURL` 给运行时 provider 配置
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:50-56,69-73

## 4. Auth Loader（运行时 token 使用）

### 4.1 auth.loader 如何被调用

`provider` 服务构建 provider 状态时，会遍历 plugins：
- 要求 plugin 有 `auth`、有已存储 auth、且定义了 `auth.loader`
- 调用形态：
  - 第一个参数：`getAuth` 函数（实时读取 Auth.get(providerID)）
  - 第二个参数：provider 配置对象 `database[plugin.auth.provider]`
- loader 返回对象被合并到 provider `options`
- 引用：packages/opencode/src/provider/provider.ts:1206-1224

类型定义也说明了 loader 签名：
- `loader?: (auth: () => Promise<Auth>, provider: Provider) => Promise<Record<string, any>>`
- 引用：packages/plugin/src/index.ts:56-59

### 4.2 loader 返回的三个字段作用

Copilot `auth.loader` 在 OAuth 存在时返回：
- `baseURL`：覆盖 SDK 请求基址（GitHub.com 或 enterprise copilot-api 域）
- `apiKey`：设为空字符串，避免走默认 API key 方案
- `fetch`：自定义请求函数，统一改写 Header
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:65-76,69-74

### 4.3 自定义 fetch 的请求头改造

核心 header 注入：
- `Authorization: Bearer ${info.refresh}`
- `x-initiator: agent|user`
- `Openai-Intent: conversation-edits`
- vision 请求附加：`Copilot-Vision-Request: true`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:132-142

删除头：
- 删除 `x-api-key`
- 删除小写 `authorization`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:144-145

删除原因（实现意图）：
1. 避免与上面显式设置的 `Authorization` 冲突（尤其大小写并存时代理层行为不一致）。
2. 防止客户端默认 `x-api-key` 鉴权路径干扰 Copilot 的 Bearer token 路径。

### 4.4 `x-initiator`（user vs agent）判断逻辑

fetch 内根据请求体推断 `isAgent`：

1. Completions API
- 条件：`body.messages` 且 URL 包含 `completions`
- 规则：最后一条消息 `role !== user` 则 `agent`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:83-93

2. Responses API
- 条件：`body.input` 存在
- 规则：最后一条 input `role !== user` 则 `agent`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:95-105

3. Messages API
- 条件：`body.messages`（且未命中 completions 分支）
- 规则：
  - 计算 `hasNonToolCalls`：最后消息 content 中是否存在非 `tool_result` 项
  - `isAgent = !(last.role === "user" && hasNonToolCalls)`
  - 即：只有“用户角色且包含真实非工具结果内容”才归类 user，其余归 agent
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:107-126

### 4.5 Vision 请求检测逻辑

1. Completions API
- 任意消息 content 中包含 `type === image_url`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:87-90

2. Responses API
- 任意 input item.content 中包含 `type === input_image`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:99-102

3. Messages API
- 任意 content part 为 `type === image`
- 或 `tool_result` 内嵌 content 出现 `type === image`
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:113-123

## 5. ProviderAuth 服务层

### 5.1 methods()：列出可用认证方法

- ProviderAuth 初始化时从所有插件中提取 `x.auth`，按 `auth.provider` 建立 `hooks` map
- `methods()` 将插件定义的 `methods` 映射成对外 schema（type/label/prompts/when/options）
- 引用：packages/opencode/src/provider/auth.ts:119-133,135-163

要点：
- 这里仅暴露 prompt 的结构化字段，不负责执行 authorize。
- `validate` 函数不会直接出现在 methods 返回结构中（只在 authorize 时执行）。
- 引用：packages/opencode/src/provider/auth.ts:142-159,174-179

### 5.2 authorize()：发起 OAuth 授权

流程：
1. 根据 `providerID + method index` 找到插件 auth method
2. 非 oauth 方法直接返回（当前逻辑仅处理 oauth）
3. 遍历 text prompts，执行 `validate`；若失败抛 `ValidationFailed(field, message)`
4. 调用插件 method.authorize(inputs)
5. 把返回的 `AuthOAuthResult` 缓存到 `pending` map
6. 向上层返回 `{url, method, instructions}`

引用：packages/opencode/src/provider/auth.ts:165-190

### 5.3 callback()：处理 OAuth 回调

流程：
1. 从 `pending` 取对应 provider 的授权上下文；不存在则 `OauthMissing`
2. 若 method=code 且缺少 code，抛 `OauthCodeMissing`
3. 执行插件返回的 `callback()`（auto）或 `callback(code)`（code）
4. 非 success -> `OauthCallbackFailed`
5. success 且含 `key` -> 存为 `Auth.Api`
6. success 且含 `refresh` -> 存为 `Auth.Oauth`

引用：packages/opencode/src/provider/auth.ts:192-226

### 5.4 表单验证与 when 条件

- `when` 是 prompt 展示条件（`key + op(eq/neq) + value`）的一部分，随 methods 暴露给前端/调用层
- `validate` 只在 authorize 阶段执行（text prompt）
- Copilot 的 `enterpriseUrl`：
  - 仅当 `deploymentType == enterprise` 时显示
  - validate 要求可解析 URL/域名
- 引用：
  - packages/opencode/src/provider/auth.ts:24-30,43-49,174-179
  - packages/opencode/src/plugin/github-copilot/copilot.ts:181-191

## 6. Plugin 注册机制

### 6.1 CopilotAuthPlugin 如何注册为内置插件

- `CopilotAuthPlugin` 在 `INTERNAL_PLUGINS` 数组中注册
- Plugin 服务初始化时顺序遍历 `INTERNAL_PLUGINS`，执行 `plugin(input)` 并收集返回的 hooks
- 引用：packages/opencode/src/plugin/index.ts:50-57,137-146

### 6.2 Hooks 结构中 auth/provider 字段用途

在插件类型系统中：
- `hooks.auth?: AuthHook`
  - 声明 provider 认证入口（methods、authorize、loader）
- `hooks.provider?: ProviderHook`
  - 声明 provider 级能力（如模型列表动态加载）
- 引用：packages/plugin/src/index.ts:56-60,181-184,189-196

Copilot 插件同时实现了两者：
- `provider.id = github-copilot`，并在有 oauth 时用 token 拉取动态模型
- `auth.provider = github-copilot`，提供完整 OAuth Device Flow 与运行时 loader
- 引用：packages/opencode/src/plugin/github-copilot/copilot.ts:43-63

---

## 附：端到端调用链（简版）

1. Plugin 初始化加载内置 Copilot 插件 -> 得到 hooks
- packages/opencode/src/plugin/index.ts:50-57,137-146

2. ProviderAuth 从 plugins 构建 auth hooks 映射
- packages/opencode/src/provider/auth.ts:119-133

3. 用户发起 `authorize` -> Copilot `authorize` 请求 `/login/device/code`
- packages/opencode/src/provider/auth.ts:165-190
- packages/opencode/src/plugin/github-copilot/copilot.ts:206-234

4. 用户完成网页输入 code 后，系统触发 `callback` -> 轮询 `/login/oauth/access_token`
- packages/opencode/src/provider/auth.ts:192-226
- packages/opencode/src/plugin/github-copilot/copilot.ts:236-305

5. 成功后 `Auth.set(...oauth...)` 落盘 `auth.json (0600)`
- packages/opencode/src/provider/auth.ts:216-224
- packages/opencode/src/auth/index.ts:10,75-76

6. Provider 状态构建时调用 `auth.loader`，把 baseURL/apiKey/fetch 注入 provider options，后续请求全部走 Bearer + header 策略
- packages/opencode/src/provider/provider.ts:1206-1224
- packages/opencode/src/plugin/github-copilot/copilot.ts:65-151
