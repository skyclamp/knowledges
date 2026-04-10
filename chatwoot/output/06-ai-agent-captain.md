# Chatwoot AI Agent—自动化系统深度分析

*文档生成于代码审核。所有引用均指向 Chatwoot 主分支代码库。*

---

## 目录

1. [系统架构总览](#系统架构总览)
2. [核心概念与数据模型](#核心概念与数据模型)
3. [三种 AI Agent 接入方式](#三种-ai-agent-接入方式)
4. [完整事件流与消息转交机制](#完整事件流与消息转交机制)
5. [Enterprise Captain 助手系统](#enterprise-captain-助手系统)
6. [Webhook 与自动化规则引擎](#webhook-与自动化规则引擎)
7. [实现参考](#实现参考)

---

## 系统架构总览

### 架构核心特征

**Chatwoot 在会话管理上实现分层 AI 支持**：

| 层级 | 技术 | 特点 | 适用场景 |
|-----|------|------|---------|
| **Webhook AgentBot** | REST Webhook | 完全独立、支持任意 LLM | 通用、自主部署 |
| **Dialogflow** | Google 专有 API | 原生 NLU、多语言 | 快速 Intent 识别 |
| **Captain 助手** | 企业 LLM + 知识库 | 深度集成、工具链 | 复杂场景、需要上下文 |

**核心驱动机制**：

- **事件驱动系统** (`Rails.configuration.dispatcher`) → 触发 listeners → 并发执行（Sync vs Async）
- **会话状态机** (`:open/:pending/:resolved/:snoozed`) → 状态变更触发不同分支
- **双重分配模型** (`assignee_id` 用户 vs `assignee_agent_bot_id` Bot，互斥设计)
- **转交触发点** (Bot 返回特定 payload 或状态设置 `waiting_since` 字段)

---

## 核心概念与数据模型

### Conversation 状态与分配

会话有 4 个状态，每个状态触发不同的 agent 行为：

```ruby
# app/models/conversation.rb
enum status: { open: 0, resolved: 1, pending: 2, snoozed: 3 }

# Core Fields
- assignee_id           # 人工分配给某个 User
- assignee_agent_bot_id # Bot 分配，与 assignee_id 互斥（通过 reset_agent_bot_when_assignee_present）
- waiting_since         # Bot 转交标记，表示等待人工接入
- priority              # 0-3 (low/medium/high/urgent)
- team_id               # 团队限制 (可选)
```

**状态决策逻辑** (app/models/conversation.rb 第 249-258 行)：

```ruby
# determine_conversation_status (before_create 钩子)
if contact.blocked?
  status = :resolved  # 阻止的联系人不开启会话
elsif inbox.active_bot?
  status = :pending   # 设置为 pending，让 bot 先处理
else
  status = :open      # 正常情况下，等待人工分配
end
```

### Schema 关键字段对应表

| 表 | 字段 | 含义 | 使用点 |
|---|------|------|--------|
| `conversations` | `assignee_agent_bot_id` | 当前 Bot 分配 | AgentBotListener 投递消息、bot_handoff 清空 |
| `conversations` | `waiting_since` | Bot 转交标记 | `bot_handoff!()` 设置、自动分配触发 |
| `agent_bots` | `outgoing_url` | Webhook 端点 | `AgentBots::WebhookJob` 投递入口 |
| `agent_bots` | `bot_type` | Bot 类型 | 当前仅支持 webhook (enum: 0) |
| `agent_bot_inboxes` | `status` | active/inactive | 控制 bot 启用/禁用（app/models/inbox.rb 第 170 行 `active_bot?` 检查) |
| `webhooks` | `subscriptions` | 事件订阅数组 | 过滤投递（14 个事件） |
| `automation_rules` | `conditions` / `actions` | JSONB | 条件过滤 + 动作执行 |
| `integrations_hooks` | `app_id` | 'dialogflow' / 'slack' / 等 | HookJob 路由 |

---

## 三种 AI Agent 接入方式

### 方式 1：Webhook AgentBot（推荐通用）

**概述**：通过 REST Webhook 完全解耦实现，支持任意 LLM 后端与自定义逻辑。

**实现途径**：

1. **创建 AgentBot**：
   - API: `POST /api/v1/accounts/{id}/agent_bots`
   - 参数：`name, description, outgoing_url, bot_type='webhook', bot_config={}`
   - 生成 AccessToken（`access_tokens` 表，owner_type='AgentBot'） —— 但注意当前 AccessToken API 不支持该 Bot 的反向调用

2. **激活到 Inbox**：
   - API: `POST /api/v1/accounts/{id}/inboxes/{inbox_id}/set_agent_bot`
   - 参数：`agent_bot={id}`
   - 创建 `AgentBotInbox` 关联，status=`active`

3. **事件流**：
   - 用户消息 → `message_created` 事件
   - `AgentBotListener.message_created()` 执行（app/listeners/agent_bot_listener.rb 第 19-27 行）
   - 条件：`message.webhook_sendable?` ✓ (排除 activity/auto_reply)、`conversation.pending?` ✓ 或已分配给该 Bot
   - → `AgentBots::WebhookJob` 投递（高优先级队列）
   - → `Webhooks::Trigger.execute()` 通过 RestClient POST

4. **Webhook Payload 结构**：

   ```ruby
   {
     event: 'message_created',
     message: {
       id: 12345,
       content: 'User message',
       message_type: 'incoming',
       sender: { id: 999, name: 'John', type: 'contact', ... },
       conversation: {
         id: 567,
         display_id: '#42',
         status: 'pending',
         assignee_agent_bot_id: 42,  # 当前 Bot ID
         waiting_since: null,
         custom_attributes: {...},
         ...
       },
       inbox: {...},
       account: {...}
     }
   }
   ```

5. **Bot 回复方式**（核心差异）：

   **方式 A：返回回复或转交（推荐）**
   ```json
   {
     "action": "handoff",
     "handoff_message": "I need human assistance."
   }
   ```
   **内部处理**：
   - Chatwoot 接收 webhook 返回（需自己实现处理逻辑或通过 AutomationRule）
   - 或在 Bot 侧调用 Chatwoot 对话 API：
     - `POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/toggle_status`
     - 参数：`status='open'`（将 pending 转为 open，自动清空 `assignee_agent_bot_id`、设置 `waiting_since`）
   - 事件分发 `CONVERSATION_BOT_HANDOFF` → AutoAssignment 触发人工分配

   **方式 B：直接创建消息（需 Bot 的 AccessToken）**
   ```ruby
   POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/messages
   {
     "message": {
       "content": "Reply from bot",
       "private": false
     }
   }
   ```

**代码位置**：
- 模型: app/models/agent_bot.rb (第 29-35 行: 关系与字段)
- 监听: app/listeners/agent_bot_listener.rb (第 1-70 行)
- 投递: app/jobs/agent_bots/webhook_job.rb (第 1-13 行)
- API:  app/controllers/api/v1/accounts/agent_bots_controller.rb (第 1-50 行)

**优点**：
- ✅ 完全自主：任意 LLM、任意编程语言、完整业务逻辑
- ✅ 无 Chatwoot 代码修改：纯 API 消费方
- ✅ 支持并发：高优先级队列、重试机制（TooManyRequests/InternalServerError 重试 3 次，间隔 3 秒）

**缺点**：
- ❌ 需要维护外部服务
- ❌ 延迟：网络往返 + 处理时间
- ❌ 状态同步复杂：需要自己处理失败回退

---

### 方式 2：Dialogflow 集成（Google 专有）

**概述**：直接集成 Google Dialogflow V2 API，支持多轮对话与 Intent 提取。

**实现途径**：

1. **配置 Hook**：
   - 类型：`Integrations::Hook` (多对一，支持多个 Dialogflow hook per account)
   - `app_id='dialogflow'`, `hook_type='inbox'`（每个 Inbox 一个 hook）
   - settings：
     ```json
     {
       "credentials": "{ ... Service Account JSON ... }",
       "project_id": "my-project-123",
       "region": "global" || "eu" || "us-central1" // 地域端点
     }
     ```

2. **事件监听**：
   - `HookListener.message_created()` 和 `message_updated()` 事件触发（app/listeners/hook_listener.rb 第 1-65 行）
   - 条件：inbox 为 Dialogflow 接收方、消息可处理 (`message.webhook_sendable?`)
   - → `HookJob` 分发到 `Integrations::Dialogflow::ProcessorService`

3. **Dialogflow API 调用**（lib/integrations/dialogflow/processor_service.rb 第 1-100 行）：

   ```ruby
   detect_intent(
     session_path:   "projects/#{project_id}/locations/#{region}/agent/sessions/#{session_id}",
     query_input: {
       text: { text: message.content, language_code: 'en-US' }
     }
   )
   ```

4. **响应处理**：
   - 遍历 `response.query_result['fulfillment_messages']` 数组
   - 若包含 `action` 字段 → 调用 `process_action()` (隐式握手信号：`handoff` 或 `resolve`)
   - 否则 → 创建 outgoing message

5. **错误处理**：
   - Permission 错误 → `hook.prompt_reauthorization!()` + `hook.disable()`
   - 网络错误：纳入通用 webhook 错误处理

**代码位置**：
- Hook 模型: app/models/integrations/hook.rb (第 55-56 行: `dialogflow?()` 方法)
- 监听: app/listeners/hook_listener.rb (第 1-65 行)
- 处理: lib/integrations/dialogflow/processor_service.rb (第 1-100 行)
- 工作: app/jobs/hook_job.rb (第 12-39 行)

**优点**：
- ✅ 开箱即用：Google 的 NLU 模型质量高
- ✅ 多语言支持：内建语言代码
- ✅ 可视化配置：Google Console 中管理 Intent

**缺点**：
- ❌ 平台锁定：仅 Google Dialogflow V2
- ❌ 成本：Google Cloud 按 API 调用计费
- ❌ 事件支持有限：仅 message.created/updated
- ❌ 实时响应延迟：依赖 Google 服务可用性

---

### 方式 3：Captain 助手（Enterprise LLM）

**概述**：Chatwoot 内置 Enterprise LLM 系统，深度集成会话上下文、知识库、工具链与自动分流。

**实现途径**：

1. **创建 Captain 助手**：
   - API: `POST /api/v1/accounts/{id}/captain/assistants`
   - 参数：
     ```json
     {
       "name": "Support Bot",
       "description": "Handles customer inquiries",
       "config": {
         "product_name": "MyProduct",
         "feature_faq": true,          // 启用 FAQ 生成
         "feature_memory": true,        // 启用联系人记忆
         "feature_contact_attributes": false,  // 在 chat 上下文中包含联系人属性
         "instructions": "You are a helpful assistant...",
         "temperature": 0.7,
         "welcome_message": "Hello! How can I help?",
         "handoff_message": "Let me connect you with an agent...",
         "resolution_message": "Thank you for contacting us!"
       },
       "response_guidelines": [...],
       "guardrails": [...]
     }
     ```

2. **启用到 Inbox**：
   - API: `POST /api/v1/accounts/{id}/captain/assistants/{id}/inboxes`
   - 参数：`{"inbox_id": 123}`
   - 创建 `CaptainInbox` 关联：单个 Inbox 仅 1 个活跃 Assistant（唯一约束）

3. **消息处理流程**（enterprise/app/services/enterprise/message_templates/hook_execution_service.rb 第 1-50 行）：

   - 用户消息到达 → `message_created` 事件
   - 检查条件：`conversation.pending?` ✓ && `inbox.captain_assistant.present?` ✓
   - → 异步 `Captain::Conversation::ResponseBuilderJob` 入队（带可配置延迟）

4. **LLM 推理**（enterprise/app/jobs/captain/conversation/response_builder_job.rb 第 1-120 行）：

   - 收集会话历史消息（格式：使用者/助手角色 + content）
   - 调用 `Captain::Assistant::AgentRunnerService.generate_response(message_history: [...])` (V2 模式)
   - 或回退 `Captain::Llm::AssistantChatService` (V1 模式)

5. **响应决策**：

   ```ruby
   # response 返回格式
   {
     "response": "reply_text" || "conversation_handoff",
     "reasoning": "explanation",
     "agent_name": "scenario_name" (可选)
   }
   ```

   - 若 `response == 'conversation_handoff'` → 自动转接：
     - 创建私密备注（转接原因）
     - 调用 `conversation.bot_handoff!()` (设置 `waiting_since`, 状态变 `open`, 分发事件)
     - 发送 out-of-office 模板（如设置）
   - 否则 → 创建 outgoing message（来自 Captain::Assistant sender）

6. **响应后处理**（enterprise/app/listeners/captain_listener.rb 第 1-15 行）：

   - 监听 `conversation.resolved` 事件
   - 若启用 `feature_memory` → `Captain::Llm::ContactNotesService` 生成联系人笔记
   - 若启用 `feature_faq` → `Captain::Llm::ConversationFaqService` 生成 FAQ 条目

7. **自动收尾机制**（enterprise/app/jobs/captain/inbox_pending_conversations_resolution_job.rb 第 1-120 行）：

   - 后台定期检查 `inbox.conversations.pending` && `last_activity_at < 1.hour.ago`
   - 若启用 `captain_auto_resolve_evaluated` → 调用 `Captain::ConversationCompletionService` 评估是否完成
   - 返回 `{complete: bool, reason: string}`
   - 完成 → auto-resolve 或 auto-handoff

**內置工具链**（enterprise/lib/captain/tools/）：

| 工具 | 功能 | 触发方式 |
|-----|------|---------|
| `FAQ Lookup` | 向量检索知识库 | 自动包含 |
| `Handoff Tool` | 转接给人工 | Agent 调用 |
| `Resolve Conversation Tool` | 自动关闭会话 | Agent 调用 |
| `Add Private Note` | 添加内部备注 | Agent 调用 |
| `Update Priority` | 改变优先级 | Agent 调用 |
| `Add Label` | 添加标签 | Agent 调用 |
| `HTTP Tool` | 调用自定义 API | Agent 调用 |
| 自定义工具 | 用户定义的 Tool | 通过 `Captain::CustomTool` |

**场景分流**（enterprise/app/models/captain/scenario.rb 第 1-150 行）：

- 定义专属 Agent 处理特定场景（如技术支持、账单问题等）
- 通过 `handoff_to_scenario_{id}_agent` 工具自动转接
- 每个 Scenario 可配置专属工具、指令、响应指南

**知识库管理**（enterprise/app/models/captain/document.rb + enterprise/app/models/captain/assistant_response.rb）：

- 上传文档（URL 爬虫或 PDF 上传）
- 自动分割 + 向量化 Embedding
- 手工编辑 FAQ 或自动生成

**代码位置**：
- 模型: enterprise/app/models/captain/assistant.rb (第 1-80 行)
- CaptainInbox: enterprise/app/models/captain_inbox.rb (第 1-31 行)
- 监听: enterprise/app/listeners/captain_listener.rb (第 1-15 行)
- 任务: enterprise/app/jobs/captain/conversation/response_builder_job.rb (推理主循环)
- 工具: enterprise/lib/captain/tools/ (handoff, resolve, FAQs)
- API: enterprise/app/controllers/api/v1/accounts/captain/assistants_controller.rb (第 1-90 行)

**优点**：
- ✅ 深度集成：完整会话上下文、知识库、工具链
- ✅ 灵活配置：多 LLM 模型、温度、场景系统
- ✅ 自动化：自动收尾、FAQ 生成、联系人笔记
- ✅ 可视化：Copilot 辅助、Playground 测试

**缺点**：
- ❌ 仅 Enterprise：需要 license
- ❌ 依赖 Captain 服务：额外配置与成本
- ❌ 实时消息实现不完整：仍需配合 Webhook AgentBot 或 API Inbox

**选择指南**：

```
场景 1：简单自动回复（是/否）
  → AgentBot + 简单逻辑（可以是 AWS Lambda + 小型 LLM）

场景 2：多语言 Intent 识别
  → Dialogflow（如已使用 Google 生态）

场景 3：复杂上下文、知识库、工具链
  → Captain（enterprise 版本）

场景 4：混合方案（推荐生产）
  → AgentBot 作底层 → 根据消息类型路由到 Dialogflow/Captain/自定义LLM
```

---

## 完整事件流与消息转交机制

### Step-by-Step：用户消息 → Bot 处理 → 人工转交

```
T0: 用户发送消息
    ↓
    contact.send_message() → Channel 接收端处理
    ↓
T1: Message#create!
    account_id, inbox_id, sender=contact, content='Help!', incoming=true
    ↓
T2: Message#after_create_commit
    → dispatch_create_events()
    → Rails.configuration.dispatcher.dispatch(MESSAGE_CREATED, ...)
    ↓
T3: 事件分发（并发）

    ┌─ ActionCableListener (sync)
    │  → 实时推送到客户端 UI
    │
    ├─ AgentBotListener (sync)              ← 重点
    │  → agent_bots_for(inbox, conversation)
    │     ├ conversation.assignee_agent_bot ✓?
    │     └ inbox.agent_bot_inbox&.active? ✓?
    │  → AgentBots::WebhookJob.perform_later
    │     URL: agent_bot.outgoing_url
    │     Payload: message.webhook_data.merge(event: 'message_created')
    │
    ├─ WebhookListener (async)
    │  → deliver_webhook_payloads(payload, inbox)
    │     ├ account.webhooks.account_type (事件过滤)
    │     ├ inbox.channel.webhook_url (API Inbox only)
    │     → WebhookJob.perform_later
    │
    ├─ HookListener (async)
    │  → Dialogflow/Slack 等第三方集成
    │     → HookJob.perform_later
    │
    └─ AutomationRuleListener (async)
       → rule_present?('message_created', account)
       → ConditionsFilterService 条件匹配
       → ActionService 执行动作（send_message/assign_agent/等）

T4: AgentBots::WebhookJob 执行（高优先级）
    ↓
    Webhooks::Trigger.execute(url, payload, :agent_bot_webhook)
    ↓
    RestClient::Request.post(
      headers: {
        'X-Chatwoot-Delivery': uuid,
        'X-Chatwoot-Signature': "sha256=#{hmac}"  (if secret)
      }
    )
    ↓
    你的 Bot 服务收到 Webhook

T5: Bot 决策点

    Option A：继续对话
    ──────────────────
    Bot 侧逻辑：
      parsed_msg = parse(payload['message']['content'])
      response = invoke_your_llm(parsed_msg)
    
    返回给 Chatwoot：
      { "content": "I can help with that. Could you clarify..." }
    
    或调用 Chatwoot API 创建消息（需 Bot 的 AccessToken）：
      POST /api/v1/accounts/{account_id}/conversations/{conv_id}/messages
      { "message": { "content": "...", "private": false } }
    
    Chatwoot 内部：
      → 创建 outgoing message (sender = agent_bot)
      → 分发 MESSAGE_CREATED 事件

    Option B：转交给人工
    ────────────────────
    Bot 侧返回：
      { "action": "handoff", "handoff_message": "I need a human..." }
    
    Chatwoot 处理（如果实现了对应解析）：
      → conversation.bot_handoff!()
    
    或 Bot 调用 Chatwoot API：
      POST /api/v1/accounts/{account_id}/conversations/{conv_id}/toggle_status
      { "status": "open" }  # pending → open
    
    Chatwoot 内部（toggle_status）：
      → if pending_to_open_by_bot? (Current.user.is_a?(AgentBot))
      →   conversation.bot_handoff!()
      →   DispatchEvent: CONVERSATION_BOT_HANDOFF
    
    bot_handoff! 具体动作：
      1. update(waiting_since: Time.current) if waiting_since.blank?
      2. status = :open
      3. DispatchEvent: CONVERSATION_BOT_HANDOFF
      4. 触发下一轮事件监听

T6: Bot 转交后续事件分发

    CONVERSATION_BOT_HANDOFF 事件
    ↓
    ├─ WebhookListener.conversation_opened()
    │  → deliver_webhook_payloads({ event: 'conversation_opened', ... })
    │
    └─ AutoAssignmentHandler (after_save 钩子)
       → conversation_status_changed_to_open?() ✓
       → should_run_auto_assignment?() ✓
       → if inbox.auto_assignment_v2_enabled?
           AutoAssignment::AssignmentJob.perform_later(inbox_id:)
         else
           AutoAssignment::AgentAssignmentService(...).perform
       
       Assignment Logic：
       1. unassigned_conversations() 查询 open + assignee_id.nil?
       2. find_available_agent() 筛选：
          - inbox.available_agents (成员 + 能力)
          - 团队限制 (if team_id)
          - 分配容量限制 (rate limiter)
       3. assign_conversation(agent) 
          → conversation.update!(assignee_id: agent.id)
          → DispatchEvent: ASSIGNEE_CHANGED
          → 人工分配完成 ✓

T7: 人工回复

    Agent 创建回复消息
    → message_created 事件 (但 Current.user.is_a?(User) 不是 Bot)
    → AutomationRuleListener 不重复触发（performed_by: AutomationRule）
    → WebhookListener 投递 webhook
    
    流程同 T3（并发接收方）

T8: 人工解决会话

    agent.resolve_conversation()
    → conversation.update!(status: :resolved)
    ↓
    CONVERSATION_RESOLVED 事件分发
    ├─ WebhookListener → deliver_webhook_payloads
    ├─ CaptainListener (Enterprise)
    │  → if feature_memory: ContactNotesService.generate()
    │  → if feature_faq: ConversationFaqService.generate()
    └─ AutomationRuleListener
       → trigger('conversation_resolved', automation_actions)
    
    ↓
    会话对话流程完成 ✓
```

### 关键数据流：Payload 结构

**MESSAGE_CREATED 事件**：

```json
{
  "event": "message_created",
  "message": {
    "id": 12345,
    "content": "User message text",
    "message_type": "incoming",
    "content_type": "text",
    "private": false,
    "sender": {
      "id": 999,
      "name": "John Doe",
      "type": "contact",
      "email": "john@example.com"
    },
    "conversation": {
      "id": 567,
      "uuid": "uuid-string",
      "display_id": "#42",
      "status": "pending",
      "assignee_agent_bot_id": 42,
      "assignee_id": null,
      "waiting_since": null,
      "priority": 1,
      "team_id": null,
      "custom_attributes": {},
      "additional_attributes": {},
      "labels": ["urgent"],
      "created_at": 1704067200,
      "updated_at": 1704067260,
      "first_reply_created_at": null
    },
    "inbox": {
      "id": 10,
      "name": "Support"
    },
    "account": {
      "id": 1,
      "name": "MyAccount"
    },
    "attachments": []
  }
}
```

**CONVERSATION_STATUS_CHANGED 事件**（转交时）：

```json
{
  "event": "conversation_status_changed",
  "conversation": {},
  "changed_attributes": {
    "status": ["pending", "open"],
    "waiting_since": [null, 1704067260]
  }
}
```

---

## Enterprise Captain 助手系统

### Captain 工作流详解

Captain 是 Chatwoot 企业版内置的多 Agent 协作系统，基于 Ruby LLM (`ruby_llm` gem)。

#### 核心组件

```
Captain::Assistant
  ├ Inboxes (多对多通过 CaptainInbox)
  ├ Documents (知识库：URL 爬虫/PDF)
  ├ AssistantResponses (FAQ 条目，带向量 embedding)
  ├ Scenarios (专属 Agent，支持工具转接)
  ├ CustomTools (用户定义的 HTTP 工具)
  └ Messages (Assistant 发送的消息)

工作流：
1. 用户消息 → pending 会话
2. Captain::Conversation::ResponseBuilderJob 异步推理
3. 多 Agent 分流（通过 Scenarios）
4. 工具调用（知识库查询、HTTP API）
5. 生成响应或转交
6. 事件后处理（笔记生成、FAQ 去重）
```

#### LLM 配置

**默认模型** (config/llm.yml 第 1-100 行)：

```yaml
providers:
  openai:
    display_name: 'OpenAI'
  anthropic:
    display_name: 'Anthropic'
  gemini:
    display_name: 'Gemini'

models:
  gpt-5.1: { provider: openai, credit_multiplier: 3 }
  gpt-4.1: { provider: openai, credit_multiplier: 2 }
  claude-sonnet-4.5: { provider: anthropic, credit_multiplier: 3 }
  gemini-3-pro: { provider: gemini, credit_multiplier: 3 }
```

#### 推理引擎 V2（新）

**Agent Runner** (enterprise/app/services/captain/assistant/agent_runner_service.rb 第 1-360 行)：

```ruby
# 初始化
service = Captain::Assistant::AgentRunnerService.new(
  assistant: captain_assistant,
  conversation: conversation,
  source: 'response_builder'
)

# 生成响应
response = service.generate_response(
  message_history: [
    { role: 'user', content: '...' },
    { role: 'assistant', content: '...' }
  ]
)

# 返回
{
  'response': 'text' or 'conversation_handoff',
  'reasoning': 'why',
  'agent_name': 'scenario_name'
}
```

---

## Webhook 与自动化规则引擎

### Webhook 系统

**Webhook 模型** (app/models/webhook.rb 第 1-50 行)：

```ruby
class Webhook < ApplicationRecord
  # 字段
  url: String
  subscriptions: JSONB (array)     # 事件订阅
  secret: String (encrypted)
  webhook_type: Enum (account_type / inbox_type)
  
  ALLOWED_WEBHOOK_EVENTS = %w[
    conversation_created conversation_updated conversation_status_changed
    contact_created contact_updated
    message_created message_updated
    webwidget_triggered inbox_created inbox_updated
    conversation_typing_on conversation_typing_off
  ]
```

### 自动化规则引擎

**条件系统** (lib/filters/filter_keys.yml + app/services/automation_rules/condition_validation_service.rb)：

支持的条件属性：
- **Conversation**: status, assignee_id, inbox_id, team_id, priority, labels, created_at, last_activity_at
- **Contact**: name, email, phone_number, identifier, country_code, city
- **Custom Attributes**: 动态条件

支持的操作符：
- 文本：equal_to, not_equal_to, contains, starts_with
- 日期：is_greater_than, is_less_than
- 逻辑：attribute_changed

**动作系统** (app/services/automation_rules/action_service.rb 第 1-150 行)：

```ruby
# 支持动作
- send_message
- send_webhook_event
- assign_agent
- change_status
- add_label / remove_label
- assign_team
- change_priority
- resolve_conversation
- mute_conversation / snooze_conversation
- send_email_to_team
```

---

## 实现参考

### 快速开始：建立第一个 Webhook AgentBot

**1. 后端（Chatwoot 端）**

```bash
# 创建 Agent Bot
curl -X POST http://localhost:3000/api/v1/accounts/1/agent_bots \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Bot",
    "description": "Test bot",
    "outgoing_url": "https://mybot.example.com/webhook",
    "bot_type": "webhook",
    "bot_config": {}
  }'

# 激活到 Inbox
curl -X POST http://localhost:3000/api/v1/accounts/1/inboxes/10/set_agent_bot \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "agent_bot": 42 }'
```

**2. Bot 侧实现（Node.js 示例）**

```javascript
const express = require('express');
const crypto = require('crypto');
const app = express();

// 验证 webhook 签名
function verifySignature(req, secret) {
  const timestamp = req.headers['x-chatwoot-timestamp'];
  const signature = req.headers['x-chatwoot-signature'];
  const body = JSON.stringify(req.body);
  
  const expectedSignature = 
    'sha256=' + 
    crypto
      .createHmac('sha256', secret)
      .update(`${timestamp}.${body}`)
      .digest('hex');
  
  return signature === expectedSignature;
}

// 处理 webhook
app.post('/webhook', express.json(), async (req, res) => {
  const { event, message, conversation } = req.body;
  
  console.log(`收到事件: ${event}`);
  console.log(`消息: ${message.content}`);
  
  // 简单逻辑：检查关键词
  if (message.content.toLowerCase().includes('help')) {
    return res.json({
      content: '我可以提供帮助。',
    });
  }
  
  // 不匹配规则 → 转交
  return res.json({
    action: 'handoff',
    handoff_message: '连接人工客服...'
  });
});

app.listen(3000, () => {
  console.log('Bot 服务启动在 :3000');
});
```

**3. 业务流程**

```
用户: "我需要帮助"
  ↓ message_created 事件
Bot 服务收到 Webhook
  ↓ 检查关键词
返回: { "action": "handoff", "handoff_message": "连接人工..." }
  ↓
Chatwoot 处理 → conversation.bot_handoff!()
  ↓
自动分配给人工代理
```

### 切换到 Captain 助手

```bash
# 创建 Captain 助手
curl -X POST http://localhost:3000/api/v1/accounts/1/captain/assistants \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support Assistant",
    "description": "Handles customer support",
    "config": {
      "feature_faq": true,
      "feature_memory": true,
      "temperature": 0.7
    }
  }'

# 激活到 Inbox
curl -X POST http://localhost:3000/api/v1/accounts/1/captain/assistants/1/inboxes \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "inbox_id": 10 }'
```

---

## 总结与决策矩阵

| 特性 | Webhook AgentBot | Dialogflow | Captain |
|-----|-----------------|-----------|---------|
| **部署复杂度** | 中（需外部服务） | 低（仅凭证） | 高（企业版） |
| **模型灵活性** | 极高（任意 LLM） | 低（仅 Google） | 高（可配） |
| **知识库** | 手动集成 | 不内建 | 内建 RAG |
| **工具链** | 自己实现 | 不支持 | 内建 + 可扩展 |
| **成本** | 用户维护 | Google API 费用 | Chatwoot License |
| **生产就绪** | ✓（需规范部署） | ✓ | ✓（企业版） |
| **推荐场景** | MVP、通用集成 | 快速启动、多语言 | 复杂场景、规模化 |

---

**文档维护**：本文档与 Chatwoot 主分支同步更新。如有疑问，请参考源代码注释和 Git History。
