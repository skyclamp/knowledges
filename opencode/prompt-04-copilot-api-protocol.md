# Prompt 04: 分析 Copilot API 协议细节和请求头规范

## 任务

综合所有之前读过的文件，整理出 opencode 与 GitHub Copilot API 通信的完整协议规范。这是一份面向开发者的 API 参考文档。

## 需要阅读/回顾的文件

1. `packages/opencode/src/plugin/github-copilot/copilot.ts` — auth loader 中的 fetch 包装
2. `packages/opencode/src/plugin/github-copilot/models.ts` — models API
3. `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-api-types.ts` — Chat API 类型
4. `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-api-types.ts` — Responses API 类型
5. `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts` — 实际发送请求的代码
6. `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts` — 实际发送请求的代码
7. `packages/opencode/src/cli/cmd/github.ts` — GitHub CLI 命令（可能包含额外的 API 调用）

## 分析要求

请产出一份 API 协议参考文档，包含：

### 1. API 基础信息
- Base URL：
  - GitHub.com：`https://api.githubcopilot.com`
  - Enterprise：`https://copilot-api.{domain}`
- 认证方式：`Authorization: Bearer {github_oauth_token}`
- 通用请求头：
  - `User-Agent: opencode/{version}`
  - `Openai-Intent: conversation-edits`
  - `x-initiator: user | agent`
  - `Copilot-Vision-Request: true`（条件）

### 2. Models API
```
GET {baseURL}/models
```
- 请求头
- 响应体的完整 JSON Schema（从 `CopilotModels.schema` 提取）
- 每个字段的含义
- 5秒超时

### 3. Chat Completions API
```
POST {baseURL}/chat/completions
```
- 完整请求体结构（从 api-types 和实际构建请求的代码提取）
- 流式 vs 非流式
- Tool 调用格式
- 响应体结构
- SSE 事件格式

### 4. Responses API
```
POST {baseURL}/responses
```
- 完整请求体结构
- 流式 vs 非流式
- Tool 调用格式
- 响应体结构
- SSE 事件格式和事件类型列表

### 5. OAuth Device Flow API
```
POST https://github.com/login/device/code
POST https://github.com/login/oauth/access_token
```
- 请求参数
- 响应结构
- 错误处理（`authorization_pending`、`slow_down`）

### 6. 请求头决策树
用决策树或表格展示什么情况下设置什么请求头：
- 何时设置 `x-initiator: agent`（三种情况：subagent session、compaction message、非 user-initiated message）
- 何时设置 `Copilot-Vision-Request: true`
- 何时设置 `anthropic-beta`
- Chat API 特有的头 vs Responses API 特有的头

### 7. 模型路由逻辑
- GPT-5+（非 mini）→ Responses API
- 其他模型 → Chat Completions API
- fallback：`useLanguageModel()` → `sdk.languageModel()`

## 输出格式

- 文档标题：`GitHub Copilot API 协议参考`
- 使用中文
- 包含 curl 示例（伪代码形式，展示实际请求的样子）
- 所有字段标注类型和是否必需
- 每个 API 包含代码引用
- 保存到 `/Users/wenkai/workspace/knowledges/opencode/doc-04-copilot-api-protocol.md`
