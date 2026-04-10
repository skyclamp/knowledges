# GitHub Copilot API 协议参考

本文基于 opencode 当前实现（截至 2026-04-10）整理，覆盖 GitHub Copilot 对接中的认证、请求头、模型发现、Chat Completions、Responses、SSE 事件以及模型路由策略。

---

## 0. 适用范围与结论速览

- GitHub.com Base URL: `https://api.githubcopilot.com`
- Enterprise Base URL: `https://copilot-api.{domain}`
- 认证：`Authorization: Bearer {github_oauth_token}`（OAuth device flow 取到的 access token）
- 通用头（由 auth fetch 包装注入）：
  - `User-Agent: opencode/{version}`
  - `Authorization: Bearer {token}`
  - `Openai-Intent: conversation-edits`
  - `x-initiator: user|agent`
  - `Copilot-Vision-Request: true`（仅视觉输入时）

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:26`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:133`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:137`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:141`

---

## 1. API 基础信息

### 1.1 Base URL

- GitHub.com:
  - `https://api.githubcopilot.com`
- Enterprise:
  - `https://copilot-api.{normalized_enterprise_domain}`
  - `enterpriseUrl` 会去掉 `http://`/`https://` 和末尾 `/`。

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:15`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:26`

### 1.2 认证方式

- 所有 Copilot API 调用通过 Bearer Token：
  - `Authorization: Bearer {ctx.auth.refresh}`
- 同时会删除 SDK 可能附带的 `x-api-key` 与小写 `authorization`，避免冲突。

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:136`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:144`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:145`

### 1.3 通用请求头

默认注入：

- `User-Agent: opencode/{Installation.VERSION}`
- `Openai-Intent: conversation-edits`
- `x-initiator: user|agent`（按消息上下文判断）
- `Copilot-Vision-Request: true`（有图像输入时）

补充：`chat.headers` hook 还可能额外设置：

- `anthropic-beta: interleaved-thinking-2025-05-14`（仅 Copilot + Anthropic SDK）
- `x-initiator: agent`（compaction/subagent 场景强制覆盖）

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:133`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:135`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:137`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:141`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:324`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:341`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:358`

---

## 2. Models API

## 2.1 Endpoint

```http
GET {baseURL}/models
```

- 超时：5 秒（`AbortSignal.timeout(5_000)`）

代码引用：
- `packages/opencode/src/plugin/github-copilot/models.ts:115`

## 2.2 请求头

- `Authorization: Bearer {github_oauth_token}`
- `User-Agent: opencode/{version}`

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:53`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:54`

## 2.3 响应体 JSON Schema（由 CopilotModels.schema 提取）

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["data"],
  "properties": {
    "data": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["model_picker_enabled", "id", "name", "version", "capabilities"],
        "properties": {
          "model_picker_enabled": { "type": "boolean" },
          "id": { "type": "string" },
          "name": { "type": "string" },
          "version": { "type": "string" },
          "supported_endpoints": {
            "type": "array",
            "items": { "type": "string" }
          },
          "capabilities": {
            "type": "object",
            "required": ["family", "limits", "supports"],
            "properties": {
              "family": { "type": "string" },
              "limits": {
                "type": "object",
                "required": [
                  "max_context_window_tokens",
                  "max_output_tokens",
                  "max_prompt_tokens"
                ],
                "properties": {
                  "max_context_window_tokens": { "type": "number" },
                  "max_output_tokens": { "type": "number" },
                  "max_prompt_tokens": { "type": "number" },
                  "vision": {
                    "type": "object",
                    "required": [
                      "max_prompt_image_size",
                      "max_prompt_images",
                      "supported_media_types"
                    ],
                    "properties": {
                      "max_prompt_image_size": { "type": "number" },
                      "max_prompt_images": { "type": "number" },
                      "supported_media_types": {
                        "type": "array",
                        "items": { "type": "string" }
                      }
                    }
                  }
                }
              },
              "supports": {
                "type": "object",
                "required": ["streaming", "tool_calls"],
                "properties": {
                  "adaptive_thinking": { "type": "boolean" },
                  "max_thinking_budget": { "type": "number" },
                  "min_thinking_budget": { "type": "number" },
                  "reasoning_effort": {
                    "type": "array",
                    "items": { "type": "string" }
                  },
                  "streaming": { "type": "boolean" },
                  "structured_outputs": { "type": "boolean" },
                  "tool_calls": { "type": "boolean" },
                  "vision": { "type": "boolean" }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

代码引用：
- `packages/opencode/src/plugin/github-copilot/models.ts:5`

## 2.4 字段语义

- `data[]`：可用模型列表。
- `model_picker_enabled`：是否允许在 UI/模型列表中展示；opencode 仅保留 `true` 项。
- `id`：模型标识（API id）。
- `name`：展示名称。
- `version`：版本串，常见格式 `{id}-YYYY-MM-DD`。
- `supported_endpoints`：模型支持的 endpoint 列表（可选）。
- `capabilities.family`：模型家族。
- `capabilities.limits.*`：窗口和 token 限制。
- `capabilities.limits.vision`：视觉能力限制（可选）。
- `capabilities.supports.*`：功能支持矩阵（流式、工具、结构化输出、推理预算等）。

opencode 额外处理：
- 将响应映射到内部模型能力（context/input/output/token limit、toolcall、reasoning、image 支持等）。
- 删除服务端已下线模型，补充新模型。

代码引用：
- `packages/opencode/src/plugin/github-copilot/models.ts:47`
- `packages/opencode/src/plugin/github-copilot/models.ts:124`

## 2.5 curl 示例（伪代码）

```bash
curl -sS "${BASE_URL}/models" \
  -H "Authorization: Bearer ${GITHUB_OAUTH_TOKEN}" \
  -H "User-Agent: opencode/${VERSION}" \
  --max-time 5
```

---

## 3. Chat Completions API

## 3.1 Endpoint

```http
POST {baseURL}/chat/completions
```

代码引用：
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:203`
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:320`

## 3.2 请求体结构（完整）

顶层字段（`doGenerate` 与 `doStream` 共用；流式增加 `stream=true`）：

- `model` (string, 必需)
- `user` (string, 可选)
- `max_tokens` (number, 可选)
- `temperature` (number, 可选)
- `top_p` (number, 可选)
- `frequency_penalty` (number, 可选)
- `presence_penalty` (number, 可选)
- `response_format` (object, 可选)
  - JSON Schema 模式：
    - `type: "json_schema"`
    - `json_schema.schema` (object)
    - `json_schema.name` (string)
    - `json_schema.description` (string, 可选)
  - 普通 JSON：`{ "type": "json_object" }`
- `stop` (string[], 可选)
- `seed` (number, 可选)
- `reasoning_effort` (string, 可选)
- `verbosity` (string, 可选)
- `messages` (array, 必需)
- `tools` (array, 可选)
- `tool_choice` ("auto"|"none"|"required"|object, 可选)
- `thinking_budget` (number, 可选)
- 额外透传字段（object, 可选）：来自 providerOptions 中未被 schema 消费的键

流式专有附加字段：
- `stream` (boolean, 必需 for 流式，固定 `true`)
- `stream_options` (object, 可选)
  - `include_usage` (boolean, 必需 if 出现)

代码引用：
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:87`
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:142`
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:179`
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:313`

### 3.2.1 messages 项结构

`messages[]` 是以下联合：

1) System message
- `role` ("system", 必需)
- `content` (string | {type:"text",text:string}[], 必需)

2) User message
- `role` ("user", 必需)
- `content` (string | part[], 必需)
- `part`:
  - 文本：`{ type:"text", text:string }`
  - 图片：`{ type:"image_url", image_url:{ url:string } }`
    - `url` 可为远程 URL 或 data URL（本地二进制自动 base64）

3) Assistant message
- `role` ("assistant", 必需)
- `content` (string|null, 可选)
- `tool_calls` (array, 可选)
- `reasoning_text` (string, 可选，Copilot 扩展)
- `reasoning_opaque` (string, 可选，Copilot 扩展)

4) Tool message
- `role` ("tool", 必需)
- `tool_call_id` (string, 必需)
- `content` (string, 必需)

代码引用：
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-api-types.ts:3`
- `packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts:13`
- `packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts:50`
- `packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts:117`
- `packages/opencode/src/provider/sdk/copilot/chat/convert-to-openai-compatible-chat-messages.ts:153`

## 3.3 Tool 调用格式

### 3.3.1 请求中的 tools

`tools[]`：
- `type` ("function", 必需)
- `function.name` (string, 必需)
- `function.description` (string, 可选)
- `function.parameters` (object, 必需)

`tool_choice`：
- 字符串：`auto | none | required`
- 或指定函数：
  - `{"type":"function","function":{"name":"..."}}`

代码引用：
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-prepare-tools.ts:3`

### 3.3.2 assistant tool_calls

`message.tool_calls[]`：
- `type` ("function", 必需)
- `id` (string, 必需)
- `function.name` (string, 必需)
- `function.arguments` (string, 必需，JSON 字符串)

代码引用：
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-api-types.ts:40`

## 3.4 响应体结构（非流式）

- `id` (string, 可选)
- `created` (number, 可选)
- `model` (string, 可选)
- `choices` (array, 必需)
  - `message` (object, 必需)
    - `role` ("assistant", 可选)
    - `content` (string, 可选)
    - `reasoning_text` (string, 可选)
    - `reasoning_opaque` (string, 可选)
    - `tool_calls` (array, 可选)
  - `finish_reason` (string, 可选)
- `usage` (object, 可选)

代码引用：
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:748`

## 3.5 SSE 事件格式（流式）

Chat Completions 流式由 SSE 承载，数据帧为 JSON chunk（默认 message event）。SDK 解析的 chunk 联合：

1) 正常 chunk：
- `id` (string, 可选)
- `created` (number, 可选)
- `model` (string, 可选)
- `choices` (array, 必需)
  - `delta` (object, 可选)
    - `role` ("assistant", 可选)
    - `content` (string, 可选)
    - `reasoning_text` (string, 可选)
    - `reasoning_opaque` (string, 可选)
    - `tool_calls` (array, 可选)
      - `index` (number, 必需)
      - `id` (string, 可选)
      - `function.name` (string, 可选)
      - `function.arguments` (string, 可选)
  - `finish_reason` (string, 可选)
- `usage` (object, 可选)

2) 错误 chunk：
- `error.message` (string, 必需)
- `error.type` (string, 可选)
- `error.param` (any, 可选)
- `error.code` (string|number, 可选)

代码引用：
- `packages/opencode/src/provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts:780`
- `packages/opencode/src/provider/sdk/copilot/openai-compatible-error.ts:3`

## 3.6 curl 示例（伪代码）

非流式：

```bash
curl -sS "${BASE_URL}/chat/completions" \
  -H "Authorization: Bearer ${GITHUB_OAUTH_TOKEN}" \
  -H "User-Agent: opencode/${VERSION}" \
  -H "Openai-Intent: conversation-edits" \
  -H "x-initiator: user" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4.1",
    "messages": [
      {"role":"system","content":"You are helpful."},
      {"role":"user","content":"hello"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "read_file",
          "description": "Read file",
          "parameters": {"type":"object","properties":{},"required":[]}
        }
      }
    ],
    "tool_choice": "auto"
  }'
```

流式：

```bash
curl -N "${BASE_URL}/chat/completions" \
  -H "Authorization: Bearer ${GITHUB_OAUTH_TOKEN}" \
  -H "User-Agent: opencode/${VERSION}" \
  -H "Openai-Intent: conversation-edits" \
  -H "x-initiator: agent" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4.1",
    "messages": [{"role":"user","content":"stream please"}],
    "stream": true,
    "stream_options": {"include_usage": true}
  }'
```

---

## 4. Responses API

## 4.1 Endpoint

```http
POST {baseURL}/responses
```

代码引用：
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:396`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:782`

## 4.2 请求体结构（完整）

顶层字段：

- `model` (string, 必需)
- `input` (array, 必需)
- `temperature` (number, 可选)
- `top_p` (number, 可选)
- `max_output_tokens` (number, 可选)
- `text` (object, 可选)
  - `format` (object, 可选)
    - JSON Schema：`{type:"json_schema", strict:boolean, name:string, description?:string, schema:object}`
    - JSON Object：`{type:"json_object"}`
  - `verbosity` ("low"|"medium"|"high", 可选)
- `max_tool_calls` (number, 可选)
- `metadata` (any, 可选)
- `parallel_tool_calls` (boolean, 可选)
- `previous_response_id` (string, 可选)
- `store` (boolean, 可选)
- `user` (string, 可选)
- `instructions` (string, 可选)
- `service_tier` ("auto"|"flex"|"priority", 可选)
- `include` (string[], 可选)
- `prompt_cache_key` (string, 可选)
- `safety_identifier` (string, 可选)
- `top_logprobs` (number, 可选)
- `reasoning` (object, 可选；推理模型)
  - `effort` (string, 可选)
  - `summary` (string, 可选)
- `truncation` ("auto", 可选；部分模型必加)
- `tools` (array, 可选)
- `tool_choice` (string|object, 可选)

流式专有附加字段：
- `stream` (boolean, 必需 for 流式，固定 `true`)

代码引用：
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:253`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:373`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:788`

### 4.2.1 input 项结构（联合）

来自 `openai-responses-api-types.ts` 与转换器，常见类型：

- System/Developer message
  - `role` ("system"|"developer", 必需)
  - `content` (string, 必需)
- User message
  - `role` ("user", 必需)
  - `content` (array, 必需)
    - `{type:"input_text",text:string}`
    - `{type:"input_image",image_url:string}`
    - `{type:"input_image",file_id:string}`
    - `{type:"input_file",file_url:string}`
    - `{type:"input_file",filename:string,file_data:string}`
    - `{type:"input_file",file_id:string}`
- Assistant message
  - `role` ("assistant", 必需)
  - `content` (`[{type:"output_text",text:string}]`, 必需)
  - `id` (string, 可选)
- 函数调用
  - `type:"function_call"` / `type:"function_call_output"`
- 本地 shell
  - `type:"local_shell_call"` / `type:"local_shell_call_output"`
- 其他
  - `computer_call`、`reasoning`、`item_reference`、`mcp_approval_response`

代码引用：
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-api-types.ts:3`
- `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:21`
- `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:74`
- `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:161`
- `packages/opencode/src/provider/sdk/copilot/responses/convert-to-openai-responses-input.ts:311`

## 4.3 Tool 调用格式

### 4.3.1 请求中的 tools

`tools[]` 支持：

- Function tool
  - `type:"function"`
  - `name` (string, 必需)
  - `description` (string, 可选)
  - `parameters` (JSON Schema, 必需)
  - `strict` (boolean, 可选)

- Provider 内建工具
  - `file_search`
  - `local_shell`
  - `web_search`
  - `web_search_preview`
  - `code_interpreter`
  - `image_generation`

`tool_choice`：
- `auto | none | required`
- `{type:"file_search"|"web_search"|"web_search_preview"|"code_interpreter"|"image_generation"}`
- `{type:"function",name:"..."}`

代码引用：
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-prepare-tools.ts:9`

## 4.4 响应体结构（非流式）

- `id` (string, 必需)
- `created_at` (number, 必需)
- `model` (string, 必需)
- `error` (object|null, 可选)
  - `code` (string, 必需)
  - `message` (string, 必需)
- `output` (array, 必需)
  - `message` / `reasoning` / `function_call` / `web_search_call` / `file_search_call` / `code_interpreter_call` / `image_generation_call` / `local_shell_call` / `computer_call`
- `service_tier` (string|null, 可选)
- `incomplete_details` (object|null, 可选)
  - `reason` (string, 必需)
- `usage` (object, 必需)
  - `input_tokens` (number, 必需)
  - `input_tokens_details.cached_tokens` (number|null, 可选)
  - `output_tokens` (number, 必需)
  - `output_tokens_details.reasoning_tokens` (number|null, 可选)

代码引用：
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:417`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:488`

## 4.5 SSE 事件格式与事件类型列表

Responses 流式事件由 `openaiResponsesChunkSchema` 定义，事件 `type` 列表：

- `response.created`
- `response.output_item.added`
- `response.output_item.done`
- `response.output_text.delta`
- `response.function_call_arguments.delta`
- `response.image_generation_call.partial_image`
- `response.code_interpreter_call_code.delta`
- `response.code_interpreter_call_code.done`
- `response.output_text.annotation.added`
- `response.reasoning_summary_part.added`
- `response.reasoning_summary_text.delta`
- `response.completed`
- `response.incomplete`
- `error`
- 兜底：`{ type: string, ... }`（未知事件）

代码引用：
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1554`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1364`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1379`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1388`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1398`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1460`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1494`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1501`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1508`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1515`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1522`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1542`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1548`

## 4.6 curl 示例（伪代码）

非流式：

```bash
curl -sS "${BASE_URL}/responses" \
  -H "Authorization: Bearer ${GITHUB_OAUTH_TOKEN}" \
  -H "User-Agent: opencode/${VERSION}" \
  -H "Openai-Intent: conversation-edits" \
  -H "x-initiator: user" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5",
    "input": [
      {"role":"developer","content":"You are helpful."},
      {"role":"user","content":[{"type":"input_text","text":"hello"}]}
    ],
    "tools": [
      {"type":"function","name":"read_file","description":"Read file","parameters":{"type":"object"},"strict":false}
    ],
    "tool_choice": "auto"
  }'
```

流式：

```bash
curl -N "${BASE_URL}/responses" \
  -H "Authorization: Bearer ${GITHUB_OAUTH_TOKEN}" \
  -H "User-Agent: opencode/${VERSION}" \
  -H "Openai-Intent: conversation-edits" \
  -H "x-initiator: agent" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5",
    "input": [{"role":"user","content":[{"type":"input_text","text":"stream"}]}],
    "stream": true
  }'
```

---

## 5. OAuth Device Flow API

## 5.1 Endpoint

```http
POST https://github.com/login/device/code
POST https://github.com/login/oauth/access_token
```

Enterprise 时域名替换为企业域（`https://{domain}/login/...`）。

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:19`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:21`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:22`

## 5.2 第一步：device code

请求：
- Header:
  - `Accept: application/json`
  - `Content-Type: application/json`
  - `User-Agent: opencode/{version}`
- Body:
  - `client_id` (string, 必需)
  - `scope` (string, 必需，固定 `read:user`)

响应（成功）：
- `verification_uri` (string, 必需)
- `user_code` (string, 必需)
- `device_code` (string, 必需)
- `interval` (number, 必需，秒)

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:206`

## 5.3 第二步：轮询 access token

请求：
- Header 同上
- Body:
  - `client_id` (string, 必需)
  - `device_code` (string, 必需)
  - `grant_type` (string, 必需，固定 `urn:ietf:params:oauth:grant-type:device_code`)

响应（成功）：
- `access_token` (string, 可选；存在即成功)
- `error` (string, 可选)
- `interval` (number, 可选)

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:236`

## 5.4 错误处理

- `authorization_pending`：
  - 继续轮询，等待 `deviceData.interval * 1000 + 3000ms`。
- `slow_down`：
  - 默认等待 `(deviceData.interval + 5) * 1000`。
  - 若服务端返回 `interval`，优先用它（秒）再加 `3000ms` safety margin。
- 其他 `error`：
  - 视为失败。

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:280`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:285`

## 5.5 curl 示例（伪代码）

```bash
# step 1
curl -sS "https://github.com/login/device/code" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "User-Agent: opencode/${VERSION}" \
  -d '{"client_id":"Ov23li8tweQw6odWQebz","scope":"read:user"}'

# step 2 (poll)
curl -sS "https://github.com/login/oauth/access_token" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "User-Agent: opencode/${VERSION}" \
  -d '{
    "client_id":"Ov23li8tweQw6odWQebz",
    "device_code":"${DEVICE_CODE}",
    "grant_type":"urn:ietf:params:oauth:grant-type:device_code"
  }'
```

---

## 6. 请求头决策树

## 6.1 决策总表

| 条件 | Header | 值 |
|---|---|---|
| 任意 Copilot API 请求 | Authorization | Bearer {token} |
| 任意 Copilot API 请求 | User-Agent | opencode/{version} |
| 任意 Copilot API 请求 | Openai-Intent | conversation-edits |
| 默认 | x-initiator | user 或 agent（按请求体推断） |
| 检测到视觉输入 | Copilot-Vision-Request | true |
| Copilot + Anthropic SDK | anthropic-beta | interleaved-thinking-2025-05-14 |
| compaction 消息 | x-initiator | agent（覆盖） |
| subagent session（session.parentID 存在） | x-initiator | agent（覆盖） |

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:133`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:137`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:141`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:324`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:341`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:358`

## 6.2 x-initiator 何时是 agent（3 类）

1) Subagent session
- 条件：`session.parentID` 存在。
- 行为：强制 `x-initiator=agent`。

2) Compaction message
- 条件：当前消息 parts 中含 `type === "compaction"`。
- 行为：强制 `x-initiator=agent`。

3) 非 user-initiated message
- fetch 包装根据 body 推断：
  - Chat Completions：最后一条 `messages[-1].role !== "user"`
  - Responses：最后一条 `input[-1].role !== "user"`
  - Messages API 分支（兼容逻辑）：最后消息并非“user + 非 tool_result 内容”
- 行为：`x-initiator=agent`。

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:84`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:99`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:109`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:341`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:358`

## 6.3 Copilot-Vision-Request 何时为 true

- Chat Completions：任意 message 的 content part 含 `type === "image_url"`
- Responses：任意 input item 的 content part 含 `type === "input_image"`
- Messages API 兼容逻辑：message/content 或 tool_result 嵌套 content 中含 image

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:88`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:103`
- `packages/opencode/src/plugin/github-copilot/copilot.ts:117`

## 6.4 何时设置 anthropic-beta

- 条件：当前模型 provider 为 `github-copilot` 且 `incoming.model.api.npm === "@ai-sdk/anthropic"`
- 值：`interleaved-thinking-2025-05-14`

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:324`

## 6.5 Chat API 特有头 vs Responses API 特有头

结论（基于当前实现）：

- Chat 与 Responses 没有“硬编码的独占请求头”。
- 两者共享同一套 Copilot fetch 包装头注入（Authorization/User-Agent/Openai-Intent/x-initiator/Vision）。
- `anthropic-beta` 实际上几乎只会出现在 Chat 路径（Anthropic 模型走 chat），但逻辑本身在通用 `chat.headers` hook，不按 endpoint 分支。
- Responses 特有的是 body 字段和 SSE 事件集合，不是 header。

---

## 7. 模型路由逻辑

## 7.1 Copilot 路由判定

`shouldUseCopilotResponsesApi(modelID)`：

- 正则提取 `^gpt-(\d+)`
- 若主版本号 `>= 5` 且 **不是** `gpt-5-mini*`，走 Responses API
- 否则走 Chat Completions API

即：
- GPT-5+（非 mini）→ `sdk.responses(modelID)`
- 其他模型 → `sdk.chat(modelID)`

代码引用：
- `packages/opencode/src/provider/provider.ts:63`
- `packages/opencode/src/provider/provider.ts:227`

## 7.2 fallback 逻辑

- 当 provider SDK 不提供 `responses/chat`（仅有 `languageModel`）时：
  - `useLanguageModel(sdk) === true`
  - 直接 `sdk.languageModel(modelID)`

代码引用：
- `packages/opencode/src/provider/provider.ts:168`
- `packages/opencode/src/provider/provider.ts:226`

---

## 8. 与 github.ts 的关系

`packages/opencode/src/cli/cmd/github.ts` 未直接调用 Copilot API（`/models`、`/chat/completions`、`/responses`、device flow 均未出现）。该文件主要是 GitHub App / Issues / PR / Actions 工作流相关调用。

代码引用：
- `packages/opencode/src/cli/cmd/github.ts:1`

---

## 9. 实现注意点（开发者常见坑）

- 请求头大小写清理：会删除 `authorization`（小写）和 `x-api-key`，避免与 Bearer 冲突。
- 视觉头是动态推断，不是根据模型能力静态判断。
- `x-initiator` 可能被后置 hook 覆盖（compaction/subagent），不要只看 fetch 包装初值。
- Models API 有 5 秒硬超时，网络抖动时会回退到本地静态模型。
- Responses SSE 事件种类远多于 Chat，需要按 `type` 做分发处理。

代码引用：
- `packages/opencode/src/plugin/github-copilot/copilot.ts:144`
- `packages/opencode/src/plugin/github-copilot/models.ts:115`
- `packages/opencode/src/plugin/github-copilot/models.ts:124`
- `packages/opencode/src/provider/sdk/copilot/responses/openai-responses-language-model.ts:1554`
