# Prompt 02: 分析 Copilot 自定义 SDK 层（Chat & Responses API）

## 任务

阅读以下源代码文件，分析 opencode 为 GitHub Copilot 定制的 AI SDK 兼容层。这是一个 fork/定制版的 OpenAI Compatible provider，专门用于 Copilot API。输出一份详细的技术文档。

## 背景

`packages/opencode/src/provider/sdk/copilot/` 目录下的代码是一个独立的 SDK 包，只用于 Copilot provider。它实现了两种 API 协议：
- **Chat Completions API** (`/chat/completions`) — 传统的 OpenAI chat 接口
- **Responses API** (`/responses`) — OpenAI 新的 Responses API

## 需要阅读的文件

### 入口和 Provider 工厂
1. `packages/opencode/src/provider/sdk/copilot/index.ts` — 导出入口
2. `packages/opencode/src/provider/sdk/copilot/copilot-provider.ts` — Provider 工厂，创建 chat/responses model 实例
3. `packages/opencode/src/provider/sdk/copilot/openai-compatible-error.ts` — 错误处理

### Chat Completions API 实现（8个文件）
4. `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts` (815行) — Chat 模型核心实现
5. `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-options.ts` — Chat 配置选项
6. `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-api-types.ts` — API 类型定义
7. `packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts` — 消息格式转换
8. `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-prepare-tools.ts` — Tool 调用准备
9. `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-metadata-extractor.ts` — 元数据提取
10. `packages/opencode/src/provider/sdk/copilot/chat/get-response-metadata.ts` — 响应元数据
11. `packages/opencode/src/provider/sdk/copilot/chat/map-openai-compatible-finish-reason.ts` — 结束原因映射

### Responses API 实现（8+个文件）
12. `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts` (1769行) — Responses 模型核心实现
13. `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-api-types.ts` — API 类型定义
14. `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts` — 输入格式转换
15. `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-prepare-tools.ts` — Tool 调用准备
16. `packages/opencode/src/provider/sdk/copilot/responses/map-openai-responses-finish-reason.ts` — 结束原因映射
17. `packages/opencode/src/provider/sdk/copilot/responses/openai-config.ts` — 配置
18. `packages/opencode/src/provider/sdk/copilot/responses/openai-error.ts` — 错误处理
19. `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-settings.ts` — 设置
20. 检查 `packages/opencode/src/provider/sdk/copilot/responses/tool/` 目录下是否有更多文件

## 分析要求

请产出一份文档，包含以下部分：

### 1. 架构总览
- Provider 工厂模式：`createOpenaiCompatible()` 如何创建 `chat()`、`responses()`、`languageModel()` 方法
- 与标准 `@ai-sdk/openai-compatible` 的区别（为什么需要 fork）
- URL 构建逻辑：`url: ({ path }) => \`${baseURL}${path}\``
- Headers 注入机制

### 2. Chat Completions API 协议详解
- **请求格式**：`/chat/completions` 的完整请求体结构
- **消息转换**：`convertToOpenAICompatibleChatMessages` 如何将 AI SDK 的标准消息格式转为 OpenAI 格式
- **Tool 调用**：tool 定义和 tool_choice 的处理
- **流式响应**：SSE 事件的解析和 delta 处理
- **非流式响应**：完整响应的解析
- **finish_reason 映射**：`stop`、`tool_calls`、`length`、`content_filter` 等
- **元数据提取**：usage、model、id 等信息的提取

### 3. Responses API 协议详解
- **请求格式**：`/responses` 的完整请求体结构
- **输入转换**：`convertToOpenAIResponsesInput` 如何转换输入
- **Tool 定义**：tool 的格式差异（与 Chat API 对比）
- **流式响应**：事件类型（`response.output_item.added`、`response.output_text.delta` 等）
- **非流式响应**：完整响应的解析
- **reasoning/thinking 支持**：如何处理模型的 reasoning 输出

### 4. 两种 API 的选择逻辑
- 引用 `provider.ts` 中 `shouldUseCopilotResponsesApi()` 函数
- GPT-5+ 使用 Responses API，其他使用 Chat API 的逻辑
- `useLanguageModel()` fallback 的含义

### 5. 错误处理
- `openai-compatible-error.ts` 的错误格式化
- `openai-error.ts` 的 Responses API 错误处理
- HTTP 状态码到 AI SDK 错误类型的映射

### 6. 与 Vercel AI SDK 的集成
- 实现了 `LanguageModelV3` 接口的哪些方法（`doGenerate`、`doStream` 等）
- `supportsUrl`、`supportsStructuredOutputs` 等能力声明
- 如何与 AI SDK 的 `generateText`/`streamText` 配合工作

## 输出格式

- 文档标题：`Copilot SDK 层：Chat & Responses API 实现详解`
- 使用中文
- 每个代码引用标注文件路径和行号
- 对比表格展示 Chat API vs Responses API 的差异
- 保存到 `/Users/wenkai/workspace/knowledges/opencode/doc-02-copilot-sdk-layer.md`
