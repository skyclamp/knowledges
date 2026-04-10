# Copilot SDK 层：Chat & Responses API 实现详解

## 0. 范围与结论

本文分析 `packages/opencode/src/provider/sdk/copilot/` 下的 Copilot 定制 SDK 兼容层。该层是一个仅用于 Copilot 的 OpenAI-compatible fork，核心能力是：

1. 同时实现 Chat Completions (`/chat/completions`) 与 Responses (`/responses`) 两套协议。
2. 将两套协议统一适配到 Vercel AI SDK 的 `LanguageModelV3` 接口（`doGenerate` / `doStream`）。
3. 加入 Copilot/OpenAI 新特性（reasoning、built-in provider tools、Responses 流式事件族）。

相关定位可见：
- 仅用于 Copilot 的声明：`packages/opencode/src/provider/sdk/copilot/README.md:1`、`packages/opencode/src/provider/sdk/copilot/README.md:5`
- 入口导出：`packages/opencode/src/provider/sdk/copilot/index.ts:1`

---

## 1. 架构总览

### 1.1 Provider 工厂模式

`createOpenaiCompatible()` 在工厂中创建三类模型入口：

- `chat(modelId)`：返回 `OpenAICompatibleChatLanguageModel`
- `responses(modelId)`：返回 `OpenAIResponsesLanguageModel`
- `languageModel(modelId)`：当前默认回落到 `chat(modelId)`

关键实现：
- 工厂定义：`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:52`
- Chat 实例化：`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:68`、`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:69`
- Responses 实例化：`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:77`、`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:78`
- `languageModel` 回落 chat：`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:86`
- provider 方法挂载：`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:92`-`94`

### 1.2 URL 构建逻辑

统一 URL 构建是：

`url: ({ path }) => `${baseURL}${path}``

关键点：
- `baseURL` 来自配置并去掉尾部 `/`：`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:53`
- Chat URL 拼接：`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:72`
- Responses URL 拼接：`packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:81`

因此 path 层面由模型实现决定（`/chat/completions` 或 `/responses`），baseURL 只负责 host + `/v1` 前缀。

### 1.3 Headers 注入机制

Headers 采用两层策略：

1. 工厂初始化时合并默认头与用户头：
   - `Authorization: Bearer <apiKey>`（若提供）
   - 用户自定义 `headers` 覆盖默认项
   - 见 `packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:60`
2. 每次请求再通过 `withUserAgentSuffix` 追加 SDK UA 后缀：
   - `packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:66`

请求发送时再与调用级 `options.headers` 合并：
- Chat：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:206`
- Responses：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:401`、`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:784`

### 1.4 与标准 `@ai-sdk/openai-compatible` 的主要区别（为何 fork）

从代码行为看，这个 fork 的目标不是“泛 OpenAI-compatible”，而是“Copilot 专用双协议适配”：

1. 一套 provider 同时提供 chat+responses 双模型入口（标准 openai-compatible 通常聚焦 chat/completions 形态）。
   - `packages/opencode/src/provider/sdk/copilot/copilot-provider.ts:40`-`42`
2. Chat 协议里显式支持 Copilot 扩展字段：`reasoning_text` / `reasoning_opaque`。
   - 类型：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-api-types.ts:47`-`48`
   - 生成/流式处理：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:230`、`468`
3. 实现 Responses 全量语义：input item、built-in provider tools、复杂 SSE 事件族、reasoning summary 分片。
   - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1554`
4. Copilot provider 侧模型分流逻辑（GPT-5+ 用 Responses）。
   - `packages/opencode/src/provider/provider.ts:63`、`227`

---

## 2. Chat Completions API 协议详解

### 2.1 请求格式（`/chat/completions`）

Chat 请求在 `getArgs()` 构造，`doGenerate`/`doStream` 复用。

构造入口：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:87`

主要字段：
- 模型与采样：`model`、`max_tokens`、`temperature`、`top_p`、`frequency_penalty`、`presence_penalty`
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:142`-`151`
- 响应格式：
  - 普通 JSON object：`{ type: "json_object" }`
  - 结构化 JSON schema：`{ type: "json_schema", json_schema: {...} }`
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:153`
- 停止与随机项：`stop`、`seed`
- Copilot 选项：`reasoning_effort`、`verbosity`、`thinking_budget`
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:175`、`186`
- 消息和工具：`messages`、`tools`、`tool_choice`
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:179`、`182`、`183`

请求路径：
- 非流式：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:203`
- 流式：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:320`

### 2.2 消息转换：`convertToOpenAICompatibleChatMessages`

入口：`packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts:13`

转换规则：
- system：直接映射为 `{ role: "system", content }`
- user：
  - 单 text part 时降级为字符串 content（兼容旧格式）
  - 多 part 时转数组，支持 `text` 与 `image_url`
  - 图片文件转 `data:` URL 或原 URL
  - 非图片 file 抛 `UnsupportedFunctionalityError`
  - 见 `packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts:27`、`50`、`60`
- assistant：
  - 聚合 text
  - reasoning part -> `reasoning_text`
  - 从 part/providerOptions 抽 `reasoningOpaque` -> `reasoning_opaque`
  - tool-call -> `tool_calls[]`
  - 见 `packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts:73`、`119`-`121`
- tool：tool result 统一压平成 `role: "tool"` + `tool_call_id` + 字符串化 content
  - 见 `packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts:128`、`154`

### 2.3 Tool 调用：定义与 tool_choice

Chat tools 适配器：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-prepare-tools.ts:3`

- 函数工具统一映射为 OpenAI function tool：
  - `{ type: "function", function: { name, description, parameters } }`
- provider 类型工具在 Chat 协议下视为 unsupported warning
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-prepare-tools.ts:43`
- `toolChoice` 映射：
  - `auto`/`none`/`required` 直传
  - `tool` -> `{ type: "function", function: { name } }`
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-prepare-tools.ts:63`-`72`

### 2.4 流式响应：SSE 事件解析与 delta 处理

流式入口：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:305`

关键行为：
- 请求体设置：`stream: true`，可选 `stream_options.include_usage`
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:310`-`313`
- SSE handler：`createEventSourceResponseHandler(this.chunkSchema)`
- 首块输出 `response-metadata`
- usage 增量累积（含 reasoning/cached/prediction token）
- finish_reason 每块更新，最终在 `finish` 事件输出
- delta 处理顺序：
  - `reasoning_text` -> `reasoning-start` / `reasoning-delta`
  - `content` -> `text-start` / `text-delta`
  - `tool_calls` -> `tool-input-start` / `tool-input-delta` / `tool-input-end` + `tool-call`
- 若同块混合 reasoning 与 text/tool，会先补 `reasoning-end`
- `reasoning_opaque` 只允许单值，多值直接 `InvalidResponseDataError`
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:468`-`477`

收尾阶段：
- 补发未闭合 `reasoning-end` / `text-end` / tool 输入结束
- 输出统一 `finish`（含 usage + providerMetadata）
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:692`

### 2.5 非流式响应：完整响应解析

非流式入口：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:192`

响应 schema（裁剪版）定义：
- `OpenAICompatibleChatResponseSchema`：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:748`
- usage schema：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:726`

解析产物：
- content: `text` + `reasoning` + `tool-call`
- finishReason: 统一映射
- usage: prompt/completion/reasoning/cache
- providerMetadata: prediction tokens + 可选 extractor 元数据

### 2.6 finish_reason 映射

映射函数：`packages/opencode/src/provider/sdk/copilot/chat/map-openai-compatible-finish-reason.ts:3`

映射关系：
- `stop -> stop`
- `length -> length`
- `content_filter -> content-filter`
- `function_call` / `tool_calls -> tool-calls`
- 其他 -> `other`

### 2.7 元数据提取

- 通用响应元数据（id/model/timestamp）：`packages/opencode/src/provider/sdk/copilot/chat/get-response-metadata.ts:1`
- 可插拔 metadata 提取接口：
  - 非流式 `extractMetadata`
  - 流式 `createStreamExtractor().processChunk/buildMetadata`
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-metadata-extractor.ts:17`、`26`、`33`、`42`

---

## 3. Responses API 协议详解

### 3.1 请求格式（`/responses`）

构造入口：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:152`

请求核心：
- 基础字段：`model`、`input`、`temperature`、`top_p`、`max_output_tokens`
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:255`
- 文本控制：`text.format`（json schema/json_object）与 `text.verbosity`
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:261`
- provider 选项：
  - `max_tool_calls`、`parallel_tool_calls`、`previous_response_id`、`service_tier`、`include`、`top_logprobs` 等
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:281`-`292`
- reasoning 模型专属：`reasoning.effort` / `reasoning.summary`
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:295`-`302`
- truncation：特定模型自动要求 `truncation: "auto"`
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:306`
- tools：`tools` + `tool_choice`
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:386`-`387`

请求路径：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:396`、`782`

### 3.2 输入转换：`convertToOpenAIResponsesInput`

入口：`packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:21`

关键能力：
- system message 模式切换：`system` / `developer` / `remove`
  - `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:44`
- file 输入支持：
  - image -> `input_image`（URL / file_id / data URL）
  - pdf -> `input_file`（URL / file_id / base64）
  - `isFileId` 按前缀判定 file_id
  - `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:16`、`81`、`94`
- assistant 工具调用：
  - 常规工具 -> `function_call`
  - local_shell -> `local_shell_call`
  - provider executed 工具结果可转 `item_reference`
  - `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:141`、`161`、`174`
- reasoning：
  - store=true 时倾向 `item_reference`
  - store=false 时组装 `reasoning` + `summary_text`
  - `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:185`-`236`
- tool role 输出：
  - approval -> `mcp_approval_response`
  - local_shell json 输出 -> `local_shell_call_output`
  - 其他 -> `function_call_output`
  - `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:269`、`287`、`311`

### 3.3 Tool 定义（与 Chat API 的格式差异）

Responses tool type 联合：
- `function`
- `web_search` / `web_search_preview`
- `code_interpreter`
- `file_search`
- `image_generation`
- `local_shell`

定义见：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-api-types.ts:138`-`203`

适配器 `prepareResponsesTools`：
- 普通函数工具 -> `type: "function"`，可带 `strict`
- provider 工具按 id 映射到内建 tool
  - `openai.file_search`
  - `openai.local_shell`
  - `openai.web_search_preview`
  - `openai.web_search`
  - `openai.code_interpreter`
  - `openai.image_generation`
- `toolChoice.tool` 对内建工具映射到 `{ type: <builtin> }`，其余映射到 `{ type: "function", name }`

关键行：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-prepare-tools.ts:44`、`53`、`55`-`111`、`153`-`163`

### 3.4 流式响应：事件类型与处理

流式入口：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:777`

事件总 schema：`openaiResponsesChunkSchema`，包含：
- `response.output_item.added`
- `response.output_item.done`
- `response.output_text.delta`
- `response.function_call_arguments.delta`
- `response.reasoning_summary_part.added`
- `response.reasoning_summary_text.delta`
- `response.completed` / `response.incomplete`
- `response.created`
- `error`

定义位置：
- `response.output_item.added`：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1398`
- `response.output_text.delta`：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1364`
- `response.reasoning_summary_text.delta`：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1548`
- completed/incomplete：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1379`

处理策略亮点：
- `output_item.added` 建立状态（text/tool/reasoning）并发出 start 事件
- `output_item.done` 发出 end/tool-result
- function call 参数走增量拼接
- code_interpreter 代码流做 JSON 字符串转义拼接
- reasoning 使用 `output_index` 做稳定关联（应对 Copilot item_id 旋转）
  - 注释与状态结构：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:835`、`846`
- `response.finished`（completed/incomplete）阶段统一写 finishReason + usage

### 3.5 非流式响应解析

非流式入口：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:393`

输出 `response.output[]` 被映射为 AI SDK content：
- `message` -> `text` + `source(url/document)`
- `reasoning` -> reasoning content（携带 itemId/encryptedContent）
- `function_call` -> client-side tool-call
- provider built-in 调用（web_search/file_search/code_interpreter/image_generation/computer/local_shell）
  -> `tool-call` + 可选 `tool-result`，并标记 `providerExecuted`

映射区间：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:517`-`731`

### 3.6 reasoning/thinking 支持

Responses reasoning 支持分两层：

1. 请求层
- 对 reasoning model 才下发 `reasoning.effort/summary`
- 对非 reasoning model 发 unsupported warning
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:295`、`311`、`332`、`340`

2. 响应层
- 非流式：读取 `output.type === "reasoning"`，输出 reasoning parts，并保存 `reasoningEncryptedContent`
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:517`-`530`
- 流式：通过 summary part 事件增量输出 `reasoning-start`/`reasoning-delta`/`reasoning-end`
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1229`-`1258`

另外，模型能力判定中 `gpt-5-chat*` 被视为非 reasoning 模型：
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1690`-`1693`

---

## 4. 两种 API 的选择逻辑

此逻辑不在 copilot SDK 子目录，而在 provider 总调度：`packages/opencode/src/provider/provider.ts`。

### 4.1 `shouldUseCopilotResponsesApi()`

函数定义：`packages/opencode/src/provider/provider.ts:63`

逻辑：
- 用正则提取 `gpt-<n>` 的 `n`
- `n >= 5` 且 **不是** `gpt-5-mini` 时，返回 true（用 Responses）
- 否则走 Chat

关键判断：`packages/opencode/src/provider/provider.ts:66`

### 4.2 GitHub Copilot provider 的分流

`github-copilot` 的 `getModel()`：
- 若 SDK 不提供 `chat/responses`（旧 provider 形态），回落 `sdk.languageModel(modelID)`
- 否则按 `shouldUseCopilotResponsesApi(modelID)` 二选一：
  - true -> `sdk.responses(modelID)`
  - false -> `sdk.chat(modelID)`

关键行：`packages/opencode/src/provider/provider.ts:222`、`226`、`227`

### 4.3 `useLanguageModel()` fallback 含义

`useLanguageModel()` 定义：`packages/opencode/src/provider/provider.ts:168`

语义：当某个 provider SDK 不暴露 `responses` 与 `chat` 子方法时，统一走 `languageModel()`，避免因为新接口缺失导致崩溃。这是面向多 provider/多版本 SDK 的兼容分支。

---

## 5. 错误处理

### 5.1 Chat 侧：`openai-compatible-error.ts`

定义通用错误 schema：
- `error.message` 必填
- `type/param/code` 宽松可选
- `packages/opencode/src/provider/sdk/copilot/openai-compatible-error.ts:3`

默认错误结构：
- `errorSchema`
- `errorToMessage`
- 可选 `isRetryable`
- `packages/opencode/src/provider/sdk/copilot/openai-compatible-error.ts:19`-`21`

在 Chat 模型中注入：
- `createJsonErrorResponseHandler(errorStructure)`
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:70`

### 5.2 Responses 侧：`openai-error.ts`

与 Chat 类似，但单独定义为 OpenAI Responses 错误处理器：
- schema：`packages/opencode/src/provider/sdk/copilot/responses/openai-error.ts:4`
- handler：`packages/opencode/src/provider/sdk/copilot/responses/openai-error.ts:19`

在 Responses 模型中用于 generate/stream：
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:408`、`790`

### 5.3 HTTP 状态码到 AI SDK 错误类型映射

本层没有手写“状态码 -> 业务错误枚举”的大映射表，主要依赖 `@ai-sdk/provider-utils` 的 `createJsonErrorResponseHandler` 标准行为。

可见的显式映射点：
- Responses `doGenerate` 若 body 内有 `response.error`，主动抛 `APICallError`，并将 `statusCode` 固定为 400：
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:493`-`501`
- 其余 HTTP 错误由统一 handler 解析后转 `APICallError`（状态码由底层响应携带），并沿 AI SDK 错误语义上抛。

---

## 6. 与 Vercel AI SDK 的集成

### 6.1 `LanguageModelV3` 接口实现

Chat 与 Responses 两个类都实现 `LanguageModelV3`：
- Chat 类定义：`packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:49`
- Responses 类定义：`packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:126`

两者都实现：
- `specificationVersion = "v3"`
- `provider` getter
- `doGenerate(options)`
- `doStream(options)`

### 6.2 能力声明：`supportsUrl`、`supportsStructuredOutputs`

- Chat：
  - `supportsStructuredOutputs` 由构造配置注入，默认 false
  - `supportedUrls` 可由 config 回调注入
  - `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:45`、`56`、`72`、`79`
- Responses：
  - 内置声明支持 URL 型 `image/*`、`application/pdf`
  - `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:137`

### 6.3 与 `generateText` / `streamText` 协作方式

Vercel AI SDK 在运行时调用模型对象的 `doGenerate` 或 `doStream`。该 Copilot 层保证两种协议最终都归一化为 `LanguageModelV3Content` 与 `LanguageModelV3StreamPart`：

- 非流式：返回 `content`、`finishReason`、`usage`、`providerMetadata`、`request/response`
- 流式：输出 `stream-start`、`response-metadata`、`text-*`、`reasoning-*`、`tool-*`、`finish`

因此上层 `generateText`/`streamText` 不需要关心底层是 Chat 还是 Responses，只消费统一 V3 事件语义。

---

## 7. Chat API vs Responses API 对比

| 维度 | Chat Completions | Responses |
|---|---|---|
| 端点 | `/chat/completions` (`...chat-language-model.ts:203`, `:320`) | `/responses` (`...openai-responses-language-model.ts:396`, `:782`) |
| 输入主字段 | `messages` (`...chat-language-model.ts:179`) | `input` (`...responses-language-model.ts:255`) |
| 消息模型 | role-based message + tool_calls | item-based input/output（message/function_call/reasoning/...） |
| 工具定义 | 仅 function tools，provider tools 记 warning (`...prepare-tools.ts:43`) | function + 多种 provider built-in tools (`...openai-responses-api-types.ts:138`) |
| tool_choice | `auto/none/required/function(name)` | `auto/none/required` + built-in tool 类型枚举 |
| reasoning 字段 | Copilot 扩展：`reasoning_text`/`reasoning_opaque` | 原生 `reasoning` item + summary 事件 |
| 流式事件 | OpenAI chat chunk delta（content/tool_calls/reasoning_text） | Responses 事件族（output_item.added、output_text.delta、reasoning_summary_text.delta...） |
| source/citation | 基本不强调 citation item | 支持 output annotation -> `source(url/document)` |
| service tier | 无专门验证 | 有 `flex/priority` 能力校验与 warning |
| 模型策略 | 通用 chat | 更适合 GPT-5+ 与 reasoning/工具编排场景 |

---

## 8. 附录：文件清单与角色

### 入口与工厂
- `packages/opencode/src/provider/sdk/copilot/index.ts`
- `packages/opencode/src/provider/sdk/copilot/copilot-provider.ts`
- `packages/opencode/src/provider/sdk/copilot/openai-compatible-error.ts`

### Chat 实现
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts`
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-options.ts`
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-api-types.ts`
- `packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts`
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-prepare-tools.ts`
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-metadata-extractor.ts`
- `packages/opencode/src/provider/sdk/copilot/chat/get-response-metadata.ts`
- `packages/opencode/src/provider/sdk/copilot/chat/map-openai-compatible-finish-reason.ts`

### Responses 实现
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-api-types.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-prepare-tools.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/map-openai-responses-finish-reason.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-config.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-error.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-settings.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/tool/code-interpreter.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/tool/file-search.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/tool/image-generation.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/tool/local-shell.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/tool/web-search-preview.ts`
- `packages/opencode/src/provider/sdk/copilot/responses/tool/web-search.ts`
