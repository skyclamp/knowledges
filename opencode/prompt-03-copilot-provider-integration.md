# Prompt 03: 分析 Copilot Provider 在 Provider 系统中的集成

## 任务

阅读以下源代码文件，分析 GitHub Copilot provider 如何与 opencode 的整体 Provider 系统集成。包括模型发现、Provider 注册、模型实例化、请求参数变换等。输出一份详细的技术文档。

## 需要阅读的文件

1. `packages/opencode/src/provider/provider.ts` (1733行) — Provider 核心服务，**重点关注以下部分**：
   - `BUNDLED_PROVIDERS` 映射中 `"@ai-sdk/github-copilot"` 的注册（约第127-150行）
   - `custom()` 函数中 `"github-copilot"` 的自定义加载器（约第222-230行）
   - `shouldUseCopilotResponsesApi()` 函数（约第63-67行）
   - `wrapSSE()` SSE 超时包装（约第69-115行）
   - Provider 初始化和 SDK 实例创建的完整流程
   - `getModel()` 函数中如何为 copilot 创建模型实例
   - `cheapModel()` 中 copilot 的优先级（约第1595行）
   - Provider 的 `fetch` 包装和 header 注入
2. `packages/opencode/src/provider/transform.ts` (1051行) — Provider 请求/响应变换
3. `packages/opencode/src/plugin/github-copilot/models.ts` (144行) — Copilot 模型发现（从远程 API 获取可用模型列表）
4. `packages/opencode/src/plugin/github-copilot/copilot.ts` — 重点关注 `provider` hook 和 `chat.params`、`chat.headers` hook
5. `packages/opencode/src/provider/schema.ts` — ProviderID 和 ModelID 的定义
6. `packages/opencode/src/provider/models.ts` — models.dev 模型数据源
7. `packages/opencode/src/provider/error.ts` — 错误处理中与 Copilot 相关的部分

## 分析要求

请产出一份文档，包含以下部分：

### 1. Provider 注册和初始化
- `ProviderID.githubCopilot` 的定义（值为 `"github-copilot"`）
- `BUNDLED_PROVIDERS` 中如何将 `"@ai-sdk/github-copilot"` 映射到自定义的 `createGitHubCopilotOpenAICompatible`
- Provider 初始化的完整生命周期：配置加载 → auth 获取 → plugin loader 调用 → SDK 实例创建
- `custom["github-copilot"]` loader 的作用：`getModel` 如何选择 `sdk.responses()` vs `sdk.chat()`

### 2. 模型发现机制
- `CopilotModels.get()` 的工作流程：
  - 请求 `${baseURL}/models` 端点
  - 响应 schema（`CopilotModels.schema`）的完整结构
  - `model_picker_enabled` 过滤
  - `build()` 函数如何将远程模型数据转换为内部 `Model` 格式
  - 如何与 `models.dev` 的数据合并（`existing` 参数）
  - 模型能力推断：reasoning、vision、tool_calls
- `provider` hook 中模型获取的两条路径：
  - 有 OAuth 认证：从远程 API 获取
  - 无 OAuth 认证：使用本地 models.dev 数据（`fix()` 函数的作用）

### 3. 请求参数变换
- `chat.params` hook：为什么 GPT 模型要清除 `maxOutputTokens`
- `chat.headers` hook：
  - Anthropic 模型的 `anthropic-beta` header
  - compaction 消息的检测和 `x-initiator` 设置
  - 子 agent session 的 `x-initiator: agent` 标记
- `ProviderTransform` 中与 copilot 相关的变换（如果有）

### 4. Model 数据结构
- 内部 `Model` 类型的完整字段说明
- Copilot 模型特有的属性：
  - `api.npm: "@ai-sdk/github-copilot"` 的含义
  - `api.url` 和 `api.id` 的区别
  - `cost: { input: 0, output: 0 }` — Copilot 模型免费
  - `status: "active"` — 远程 API 总是返回 active
- `variants` 和 `options` 的用途

### 5. SSE 超时保护
- `wrapSSE()` 函数的实现原理
- 为什么需要 SSE 读取超时（防止 hang）
- 超时值的来源和配置

### 6. 错误处理
- `ProviderError` 中 Copilot 相关的 overflow pattern：`/exceeds the limit of \d+/i`
- API 错误的分类：context_overflow vs api_error
- 重试逻辑

### 7. cheap model 选择
- `cheapModel()` 函数中 copilot 的优先级列表：`["gpt-5-mini", "claude-haiku-4.5", ...]`
- 与其他 provider 的对比

## 输出格式

- 文档标题：`Copilot Provider 系统集成详解`
- 使用中文
- 每个代码引用标注文件路径和行号
- 包含数据流图（mermaid 语法）展示：用户请求 → Provider 系统 → Copilot SDK → GitHub API
- 保存到 `/Users/wenkai/workspace/knowledges/opencode/doc-03-copilot-provider-integration.md`
