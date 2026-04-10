# Libredesk AI 能力、自动化引擎与 Agent 集成可行性分析

**文档日期**: 2026-04-10  
**分析项目**: libredesk v1.0  
**目标**: 评估接入自定义 AI Agent 的可行性和最佳实现方案

---

## 1. 当前 AI 能力分析

### 1.1 AI Provider 接口定义

```go
// Provider 接口（internal/ai/provider.go）
type ProviderClient interface {
    SendPrompt(payload PromptPayload) (string, error)
}

type PromptPayload struct {
    SystemPrompt string `json:"system_prompt"`
    UserPrompt   string `json:"user_prompt"`
}

type ProviderType string

const (
    ProviderOpenAI ProviderType = "openai"
    ProviderClaude ProviderType = "claude"  // 已定义但未实现
)
```

**核心方法**:
- `SendPrompt(payload)`: 发送 system + user prompt，返回单个字符串响应

### 1.2 当前支持的 AI 操作

| 操作 | 实现 | 位置 | 说明 |
|------|------|------|------|
| **Completion** | ✅ 已实现 | `Manager.Completion()` | 通用提示补全，支持自定义 prompt key |
| **Message Rewrite** | ❌ 未见 | - | 通过前端调用 AI 补全来实现 |
| **Summarize** | ❌ 未见 | - | 可通过自定义 prompt key 实现 |
| **Conversation Analysis** | ❌ 未见 | - | 不在当前功能范围内 |

### 1.3 AI 在前端的使用入口

**位置**: `frontend/src/features/conversation/ReplyBox.vue`

```javascript
// 获取可用的 AI prompts
await api.getAiPrompts()  // GET /api/v1/ai/prompts

// 请求 AI 补全
const resp = await api.aiCompletion({
    prompt_key: key,          // prompt 的唯一标识
    content: textContent.value // 用户输入的内容
})
```

**前端 UX 特点**:
- 在回复框中集成 AI prompt 下拉菜单
- 通过 prompt_key 选择预定义的 System Prompt
- 补全结果覆盖到编辑器中（替换文本）
- 若用户无 AI 管理权限且缺少 API Key，弹出 API Key 配置对话框

### 1.4 AI 配置方式

**API Key 存储**:
```
数据库表: ai_providers
配置结构 (JSON):
{
  "api_key": "encrypted_string"  // 加密存储
}
```

**配置端点**:
```
PUT /api/v1/ai/provider
{
  "provider": "openai",
  "api_key": "sk-..."
}
```

**权限要求**: `ai:manage` 权限

### 1.5 OpenAI 实现详解 (internal/ai/openai.go)

```go
func (o *OpenAIClient) SendPrompt(payload PromptPayload) (string, error) {
    // 使用 gpt-4o-mini 模型
    // max_tokens: 1024
    // temperature: 0.7
    // 超时: 10秒
}
```

**调用的上下文信息**:
- `SystemPrompt`: 预定义的指导文本（从 prompts 表读取）
- `UserPrompt`: 用户输入的实时内容

**限制**:
- 标准 OpenAI Chat Completions API (1 system + 1 user message)
- 无 conversation history（每次独立调用）
- 无工具调用 (function calling) 支持
- 无流式响应支持

### 1.6 Prompt 管理

**数据模型**:
```go
type Prompt struct {
    ID        int       `db:"id"`
    CreatedAt time.Time
    UpdatedAt time.Time
    Title     string    `db:"title"`      // 显示名称
    Key       string    `db:"key"`        // 程序标识符（唯一）
    Content   string    `db:"content"`    // System prompt 内容
}
```

**查询方式**:
```sql
SELECT * FROM ai_prompts WHERE key = ?
```

---

## 2. 自动化引擎详解

### 2.1 规则数据结构

**数据库表**: `automation_rules`

```go
type RuleRecord struct {
    ID            int             // 规则 ID
    Name          string          // 规则名称
    Description   string          // 描述
    Type          string          // 规则类型（见 § 2.2）
    Events        []string        // 触发事件列表
    Enabled       bool            // 是否启用
    Weight        int             // 执行顺序权重（升序）
    ExecutionMode string          // 执行模式："all" 或 "first_match"
    Rules         json.RawMessage // 条件组 (RuleGroup[])
}

// 内存结构
type Rule struct {
    Type          string       // e.g., "new_conversation"
    ExecutionMode string       // 执行模式
    Events        []string     // 触发事件
    GroupOperator string       // 组间操作："AND" 或 "OR"
    Groups        []RuleGroup  // 最多 2 个条件组
    Actions       []RuleAction // 要执行的动作列表
}

type RuleGroup struct {
    LogicalOp string       // "AND" 或 "OR"
    Rules     []RuleDetail // 条件列表
}

type RuleDetail struct {
    Field              string // 条件字段
    FieldType          string // 字段类型
    Operator           string // 比较操作符
    Value              string // 比较值
    CaseSensitiveMatch bool   // 大小写敏感
}

type RuleAction struct {
    Type         string   // 动作类型
    Value        []string // 动作参数
    DisplayValue []string // 显示用的参数
}
```

### 2.2 支持的条件类型完整列表

**字段类型**: `FieldTypeConversationField` 或 `FieldTypeContactCustomAttribute`

#### A. 会话字段条件 (FieldTypeConversationField)

| 字段 | 类型 | 描述 | 示例 |
|------|------|------|------|
| `contact_email` | string | 联系人邮箱 | `user@example.com` |
| `subject` | string | 会话主题 | 包含特定关键词 |
| `content` | string | 最后一条消息内容 | 包含特定问题 |
| `status` | int | 会话状态 ID | `1`=未处理, `2`=已处理等 |
| `priority` | int | 优先级 ID | `1`=低, `2`=中, `3`=高 |
| `assigned_user` | int | 分配代理 ID | 必须非空 |
| `assigned_team` | int | 分配团队 ID | 必须非空 |
| `inbox` | int | Inbox ID | 属于特定 inbox |
| `hours_since_created` | float | 创建后小时数 | 超过 24 小时 |
| `hours_since_first_reply` | float | 首次回复后小时数 | 超过 12 小时 |
| `hours_since_last_reply` | float | 最后回复后小时数 | 超过 8 小时 |
| `hours_since_resolved` | float | 解决后小时数 | 超过 168 小时（1周） |

#### B. 自定义属性条件 (FieldTypeContactCustomAttribute)

- 字段名来自 `users.custom_attributes` JSONB
- 存储为任意类型（string, int, bool, float）
- 自动类型转换进行比较

### 2.3 支持的操作符完整列表

| 操作符 | 类型 | 说明 | 支持字段 |
|--------|------|------|---------|
| `equals` | string | 完全匹配 | 所有 |
| `not equals` | string | 不等于 | 所有 |
| `contains` | string | 包含（支持多值逗号分隔） | string 字段 |
| `not contains` | string | 不包含 | string 字段 |
| `set` | 状态 | 字段有值 | 所有 |
| `not set` | 状态 | 字段为空 | 所有 |
| `greater than` (>) | 数值 | 大于 | 数值字段 |
| `less than` (<) | 数值 | 小于 | 数值字段 |

### 2.4 支持的动作类型完整列表

| 动作 | 常量 | 参数 | 说明 |
|------|------|------|------|
| 分配团队 | `ActionAssignTeam` | `[team_id]` | 将会话分配给团队 |
| 分配代理 | `ActionAssignUser` | `[agent_id]` | 将会话分配给特定代理 |
| 设置状态 | `ActionSetStatus` | `[status_id]` | 更改会话状态 |
| 设置优先级 | `ActionSetPriority` | `[priority_id]` | 更改优先级 |
| 发送私人备注 | `ActionSendPrivateNote` | `[note_text]` | 添加不可见的内部备注 |
| 自动回复 | `ActionReply` | `[message_text]` | 自动发送回复给客户 |
| 应用 SLA | `ActionSetSLA` | `[sla_policy_id]` | 应用 SLA 策略 |
| 添加标签 | `ActionAddTags` | `[tag1, tag2, ...]` | 附加标签（保留现有） |
| 设置标签 | `ActionSetTags` | `[tag1, tag2, ...]` | 替换所有标签 |
| 移除标签 | `ActionRemoveTags` | `[tag1, tag2, ...]` | 删除特定标签 |
| 发送 CSAT | `ActionSendCSAT` | `[]` | 发送满意度调查链接 |

### 2.5 规则触发时机

**三种触发模式**:

1. **New Conversation** (`RuleTypeNewConversation`)
   - 触发点: 新会话创建时
   - 条件: 基于初始消息/元数据评估

2. **Conversation Update** (`RuleTypeConversationUpdate`)
   - 触发点: 会话状态、优先级、分配、标签改变时
   - 参数: 事件类型标识改动类型

3. **Time Trigger** (`RuleTypeTimeTrigger`)
   - 触发点: 每小时检查一次
   - 条件: 主要用于基于时间的条件（hours_since_* 字段）

### 2.6 执行模式

```go
const (
    ExecutionModeAll        = "all"         // 所有条件组通过则执行所有动作
    ExecutionModeFirstMatch = "first_match" // 第一个通过的规则执行后停止
)
```

**执行模式是按规则类型设置的**，不是每个规则单独设置。

### 2.7 评估器工作流程

```
1. 加载所有启用的规则 (sorted by weight ASC)
2. 对每个规则:
   a. 验证规则有 ≤2 个条件组
   b. 对每个条件组:
      - 按 LogicalOp (AND/OR) 评估所有条件
      - 返回布尔结果
   c. 按 GroupOperator (AND/OR) 合并组结果
   d. 若最终结果为真:
      - 执行所有动作
      - 若 ExecutionMode = first_match: 跳出
3. 返回
```

**关键代码**:
```go
// internal/automation/evaluator.go
func (e *Engine) evalConversationRules(rules []Rule, conv Conversation)
func (e *Engine) evaluateGroup(rules []RuleDetail, operator, conv) bool
func (e *Engine) evaluateRule(rule RuleDetail, conv) bool
```

---

## 3. Webhook 系统

### 3.1 支持的事件类型

**从 schema.sql 定义**:

```sql
CREATE TYPE webhook_event AS ENUM (
    'conversation.created',        -- 新会话创建
    'conversation.status_changed', -- 会话状态改变
    'conversation.tags_changed',   -- 会话标签改变
    'conversation.assigned',       -- 会话被分配
    'conversation.unassigned',     -- 会话取消分配
    'message.created',             -- 新消息到达
    'message.updated'              -- 消息被编辑
);

-- 内部标准事件（非用户可配置）
EventWebhookTest = "webhook.test"
```

### 3.2 Webhook 投递机制

**架构**:
- **异步队列**: `DeliveryTask` channel，默认大小可配
- **Worker Pool**: 可配置数量的并发投递工作线程
- **SSRF 防护**: 使用 `ssrfguard` 库限制可投递的 IP 范围
- **HMAC 签名**: 所有 webhook 使用 SHA256 HMAC 签名

**流程**:
```
1. 事件发生 -> TriggerEvent(event, payload)
2. 投递任务入队（若队列满则丢弃并日志记录警告）
3. Worker 获取任务
4. 计算 HMAC 签名
5. HTTP POST 到 webhook URL（3秒超时）
6. 记录投递结果（成功/失败）
```

### 3.3 Payload 格式

**接口约定**:
```javascript
{
    "event": "message.created",
    "timestamp": "2026-04-10T12:00:00Z",  // 推测
    "data": {
        // 事件特定的数据
        // 见 TriggerEvent 调用处
    }
}
```

**实际负载示例** (从代码推断):

#### message.created
```javascript
{
    "id": 12345,
    "uuid": "msg-uuid",
    "conversation_uuid": "conv-uuid",
    "sender_id": 42,
    "sender_type": "agent",  // or "contact"
    "message": "Hello...",
    "status": "sent",
    "created_at": "2026-04-10T12:00:00Z"
}
```

#### conversation.created
```javascript
{
    "id": 789,
    "uuid": "conv-uuid",
    "subject": "...",
    "inbox_id": 1,
    "contact": {...},
    "assigned_user_id": null,
    "assigned_team_id": null,
    "status_id": 1
}
```

#### conversation.assigned
```javascript
{
    "conversation_uuid": "conv-uuid",
    "actor_id": 42,
    "conversation": {...full_object...}
}
```

### 3.4 SSRF 防护

**实现** (internal/webhook/webhook.go):
```go
// 使用 ssrfguard.Guard 过滤请求
guard := ssrfguard.New(allowed...)  // allowed = 管理员配置的 CIDR

// DNS rebind 防护通过 net.Dialer.Control hook
Transport: &http.Transport{
    DialContext: guard.Control,
}
```

**绕过条件**:
- 在 `Opts.AllowedHosts` 中配置的 CIDR 前缀
- 用于允许内网 webhook 使用

### 3.5 Webhook 数据库结构

```go
type Webhook struct {
    ID        int
    CreatedAt time.Time
    UpdatedAt time.Time
    Name      string
    URL       string                // 投递 URL
    Events    pq.StringArray        // ['conversation.created', 'message.created']
    Secret    string                // 加密存储的签名密钥
    IsActive  bool                  // 是否启用
}
```

---

## 4. 当前 API 端点梳理

### 4.1 消息 API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/conversations/{uuid}/messages` | POST | 发送消息 |
| `/api/v1/conversations/{uuid}/messages` | GET | 获取消息列表 |
| `/api/v1/conversations/{uuid}/messages/{msg_uuid}` | GET | 获取单条消息 |
| `/api/v1/conversations/{uuid}/messages/{msg_uuid}/retry` | PUT | 重试失败的消息 |

**发送消息结构** (从 cmd/messages.go 推断):
```go
type messageReq struct {
    Attachments []int                  // 附件 ID
    Message     string                 // 消息内容
    Private     bool                   // 是否为私人备注
    To          []string               // 收件人邮箱
    CC          []string
    BCC         []string
    SenderType  string                 // "agent" or "contact"
    Mentions    []cmodels.MentionInput
}
```

### 4.2 会话 API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/conversations` | POST | 创建会话 |
| `/api/v1/conversations/{uuid}` | GET | 获取会话 |
| `/api/v1/conversations/{uuid}/status` | PUT | 更新状态 |
| `/api/v1/conversations/{uuid}/priority` | PUT | 更新优先级 |
| `/api/v1/conversations/{uuid}/assignee/{type}` | PUT | 分配（type=user/team） |
| `/api/v1/conversations/{uuid}/assignee/{type}/remove` | PUT | 取消分配 |
| `/api/v1/conversations/{uuid}/tags` | POST | 管理标签 |

**状态更新结构**:
```go
type statusUpdateReq struct {
    Status       string // 状态 ID
    SnoozedUntil string // 延期时间
}
```

### 4.3 自动化 API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/automations/rules` | GET | 获取规则列表 |
| `/api/v1/automations/rules` | POST | 创建规则 |
| `/api/v1/automations/rules/{id}` | GET | 获取单个规则 |
| `/api/v1/automations/rules/{id}` | PUT | 更新规则 |
| `/api/v1/automations/rules/{id}` | DELETE | 删除规则 |
| `/api/v1/automations/rules/{id}/toggle` | PUT | 启用/禁用 |
| `/api/v1/automations/rules/weights` | PUT | 更新执行顺序 |
| `/api/v1/automations/rules/execution-mode` | PUT | 更新执行模式 |

### 4.4 AI API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/ai/prompts` | GET | 获取 prompts 列表 |
| `/api/v1/ai/completion` | POST | 请求 AI 补全 |
| `/api/v1/ai/provider` | PUT | 更新 Provider 设置 |

---

## 5. AI Agent 集成方案评估

### 5.1 方案 A：通过 Webhook + 外部 Agent 服务

**架构**:
```
Customer Message → libredesk → Webhook Trigger (message.created)
                                    ↓
                            External AI Agent Service
                                    ↓
                    调用 libredesk API 回复/更新状态
```

#### 优点
✅ **最少侵入**: 无需修改 libredesk 核心代码  
✅ **最大灵活性**: Agent 完全独立，可用任何技术栈  
✅ **可扩展**: 多个 Webhook 接收端轻松并行  
✅ **解耦**: Agent 故障不影响 libredesk  

#### 缺点
❌ **网络延迟**: 跨系统调用增加响应时间  
❌ **错误处理复杂**: 需要实现重试、超时、幂等性  
❌ **知识库集成成本**: Agent 需要自行实现 RAG 和上下文管理  
❌ **权限管理**: 需要生成 API 凭证管理外部服务访问  

#### 需要的 API 端点（已有）

| 功能 | 端点 | 状态 |
|------|------|------|
| 获取会话详情 | `GET /api/v1/conversations/{uuid}` | ✅ |
| 获取消息历史 | `GET /api/v1/conversations/{uuid}/messages` | ✅ |
| 发送回复 | `POST /api/v1/conversations/{uuid}/messages` | ✅ |
| 更新状态 | `PUT /api/v1/conversations/{uuid}/status` | ✅ |
| 分配给代理 | `PUT /api/v1/conversations/{uuid}/assignee/user` | ✅ |
| 添加标签 | `POST /api/v1/conversations/{uuid}/tags` | ✅ |
| 添加私人备注 | `POST /api/v1/conversations/{uuid}/messages` (private=true) | ✅ |

**实现示例**:
```python
# External Agent Service
@app.route('/webhook', methods=['POST'])
def handle_webhook():
    event = request.json
    if event['event'] == 'message.created':
        message = event['data']
        conversation_uuid = message['conversation_uuid']
        
        # 获取会话上下文
        conv = api.get(f'/conversations/{conversation_uuid}')
        history = api.get(f'/conversations/{conversation_uuid}/messages')
        
        # 调用 AI Agent (with RAG)
        response = ai_agent.answer(
            query=message['message'],
            context=conv,
            history=history
        )
        
        if response['needs_human']:
            # 转交给人工
            api.put(f'/conversations/{conversation_uuid}/assignee/user',
                   {'assignee_id': <agent_id>})
            api.post(f'/conversations/{conversation_uuid}/messages', {
                'message': '已转交给人工服务',
                'private': False
            })
        else:
            # AI 直接回复
            api.post(f'/conversations/{conversation_uuid}/messages', {
                'message': response['reply'],
                'private': False
            })
            
            # 自动关闭
            if response.get('close_conversation'):
                api.put(f'/conversations/{conversation_uuid}/status',
                       {'status': '<closed_status_id>'})
```

#### ⚠️ 问题和解决方案

**Q1: 如何验证 webhook 签名？**  
A: libredesk webhook 使用 HMAC-SHA256，External Agent 应验证请求头中的签名。

**Q2: 外部 Agent 如何处理 libredesk API 缺少的功能？**  
A: 需要 libredesk 扩展 API（见下文）。

**Q3: 如何避免无限回环？**  
A: 在消息中标记来源（e.g., `meta: {source: 'ai_agent'}`），Agent 收到时跳过。

---

### 5.2 方案 B：扩展 Automation 引擎

**架构**:
```
新的 Action 类型: ActionCallAIAgent

Rule Conditions → Evaluator → Actions
                                  ↓
                          ActionCallAIAgent
                                  ↓
                    Call Custom Handler (in-process)
```

#### 优点
✅ **最小延迟**: 在 libredesk 进程内执行  
✅ **原子性**: 条件匹配和 Agent 调用为单一事务  
✅ **无网络开销**: 进程内调用  
✅ **集成紧密**: 可直接访问会话和消息对象  

#### 缺点
❌ **需要修改核心代码**: 需要改 automation 引擎  
❌ **性能风险**: Agent 阻塞 automation worker  
❌ **扩展性有限**: Agent 必须是 Go 实现或 gRPC 调用  
❌ **知识库集成**: 需要在 libredesk 内部维护  

#### 代码改造

**1. 定义新动作**:
```go
// internal/automation/models/models.go
const (
    ActionCallAIAgent = "call_ai_agent"
)

// 新增权限
ActionPermissions[ActionCallAIAgent] = "ai:execute"
```

**2. 扩展 ApplyAction**:
```go
// internal/conversation/conversation.go
case amodels.ActionCallAIAgent:
    agentID, err := strconv.Atoi(action.Value[0])
    if err != nil {
        return fmt.Errorf("invalid agent config ID: %w", err)
    }
    return m.InvokeAIAgent(conv, agentID, user)
```

**3. 实现 InvokeAIAgent**:
```go
func (m *Manager) InvokeAIAgent(conv Conversation, configID int, actor umodels.User) error {
    // 1. 加载 AI 配置
    config, err := m.aiStore.GetAgentConfig(configID)
    
    // 2. 收集上下文
    context := map[string]any{
        "conversation": conv,
        "messages": m.GetConversationMessages(conv.UUID, ...),
    }
    
    // 3. 调用 Agent（HTTP RPC 或本地）
    result := m.aiAgent.Execute(config.Endpoint, context)
    
    // 4. 应用结果
    if result.Reply != "" {
        m.QueueReply(..., result.Reply, ...)
    }
    if result.SetStatus {
        m.UpdateConversationStatus(conv.UUID, result.StatusID, ...)
    }
    if result.AssignToAgent {
        m.UpdateConversationUserAssignee(conv.UUID, result.AgentID, actor)
    }
    
    return nil
}
```

**代码变更范围估算**:
- 新增文件: ~2-3 个 (ai_agent_config, executor)
- 修改文件: ~4-5 个 (automation/models, conversation/conversation, cmd/automation)
- 行数: ~300-500 行

---

### 5.3 方案 C：扩展 AI Provider 为对话 Agent

**架构**:
```
ProviderClient interface 扩展:
  - SendPrompt(payload) → string          [当前]
  - ExecuteAgent(context) → AgentResult   [新增]
```

#### 优点
✅ **完整的会话管理**: Provider 可维护完整对话历史  
✅ **自然的类型系统**: AgentResult 明确定义了 Agent 能做什么  
✅ **多 Provider 支持**: 易于支持 Claude、GPT-4 等不同 Agent 后端  
✅ **灵活的集成**: 可在 Automation 或 Message Handler 中调用  

#### 缺点
❌ **语义混乱**: "Provider" 从工具变成 Agent  
❌ **状态管理复杂**: 多个 Agent 并发时的会话一致性  
❌ **API 设计挑战**: 需要定义 AgentResult 的完整规范  
❌ **向后兼容**: 现有代码以为 Provider 只做文本转换  

#### 代码改造

```go
// internal/ai/provider.go
type ProviderClient interface {
    SendPrompt(payload PromptPayload) (string, error)
    ExecuteAgent(context AgentContext) (AgentResult, error)  // 新增
}

type AgentContext struct {
    Conversation models.Conversation
    Messages     []models.Message
    Contact      models.Contact
    Metadata     map[string]any
}

type AgentResult struct {
    Reply           string
    Action          string          // "reply", "escalate", "close"
    AssignToAgentID *int
    StatusID        *int
    Tags            []string
    Confidence      float32         // Agent 信心度
    Reason          string          // 决策文本
}
```

**实现示例**:
```go
// internal/ai/openai_agent.go
func (o *OpenAIClient) ExecuteAgent(ctx AgentContext) (AgentResult, error) {
    // 使用 function_tools 调用
    // - send_reply(text)
    // - escalate_to_agent(agent_id)
    // - close_conversation()
    // - add_tags(tags[])
    
    systemPrompt := buildSystemPrompt(ctx)
    userPrompt := fmt.Sprintf(
        "Customer question: %s\n\nConversation history: %s\nDecision:",
        ctx.Messages[0].Content,
        formatHistory(ctx.Messages),
    )
    
    // 调用 gpt-4o with function_tools
    response, err := o.client.CreateChatCompletion(...)
    
    // 解析响应和 function_calls
    return parseAgentResult(response), nil
}
```

**代码变更范围估算**:
- 新增文件: ~2-3 个 (ai_agent, openai_agent_executor)
- 修改文件: ~3-4 个 (ai/provider, ai/openai, cmd/messages)
- 行数: ~400-600 行

---

### 5.4 方案 D：新增 Channel 类型作为 Agent

**架构**:
```
Inbox Channel Type 扩展: "email" → ["email", "ai_agent"]

AI Agent 表现为虚拟 Channel:
- 每个 message.created → AI 自动处理
- 可参与 automation 规则
```

#### 优点
✅ **最小改动**: 重用 Inbox/Contact/Channel 现有机制  
✅ **操作统一**: 与其他 Channel 相同的消息流  
✅ **扩展性最好**: 多个 AI Agent 只需多个 Inbox  
✅ **隔离性**: Agent 与消息系统深度解耦  

#### 缺点
❌ **语义不符**: "AI Agent" 不是真实的 Channel  
❌ **概念混淆**: 用户会困惑  
❌ **工作量最大**: 需要伪造完整的 Inbox/Contact 生命周期  
❌ **性能浪费**: 完整的消息持久化  

#### 代码改造思路

```go
// 1. schema.sql 扩展
ALTER TYPE channels ADD VALUE 'ai_agent';

// 2. 创建虚拟 Inbox
INSERT INTO inboxes (name, channel, enabled, config)
VALUES (
    'AI Agent - Triage',
    'ai_agent',
    true,
    '{"agent_type": "openai", "config_id": 123}'
);

// 3. CreateConversation 时检查 Channel 类型
// 若为 ai_agent，自动调用 agent handler

func (c *Manager) CreateConversation(...) (int, string, error) {
    inbox, err := c.inboxStore.GetDBRecord(inboxID)
    if inbox.Channel == "ai_agent" {
        config := parseAIConfig(inbox.Config)
        result := c.aiAgent.Execute(config, conversation)
        // 处理 result
        if result.Reply {
            c.QueueAutoReply(...)
        }
        if result.CloseConversation {
            c.UpdateConversationStatus(..., closedStatusID)
        }
    }
}
```

**代码变更范围估算**:
- 新增文件: ~4-5 个 (ai_channel_handler, virtual_inbox)
- 修改文件: ~6-8 个 (schema, inbox, conversation, message, handlers)
- 行数: ~600-800 行

---

## 6. 推荐方案与实施路径

### 6.1 综合评估矩阵

| 方案 | 延迟 | 修改量 | 灵活性 | 可维护性 | 知识库集成 | 推荐指数 |
|------|------|--------|---------|----------|-----------|---------|
| A (Webhook) | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 需自建 | ⭐⭐⭐⭐ |
| B (扩展 Automation) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | 可集成 | ⭐⭐⭐⭐⭐ |
| C (扩展 Provider) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | 可集成 | ⭐⭐⭐ |
| D (Channel 类型) | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐ | 可集成 | ⭐⭐ |

### 6.2 推荐方案：双层策略

**第一阶段（MVP）**：**方案 A - Webhook + 外部 Agent**

```
立即可用，零改动 libredesk
- 创建外部 Python/Node.js 服务
- 配置 message.created webhook
- 通过现有 API 回复和更改状态
```

**优势**:
- 立即上线，无需等待 libredesk 版本发布
- 可独立测试 AI 逻辑
- 发现问题易于回滚

**实现清单**:
- [ ] 注册 Webhook: `message.created`
- [ ] 外部服务接收 message.created 事件
- [ ] RAG 知识库集成（向量数据库 + embedding）
- [ ] AI 调用逻辑（LLM + function calling）
- [ ] API 调用错误处理和重试
- [ ] 日志和监控

**预期工作量**: 2-3 周

---

**第二阶段（生产优化）**：**方案 B - 扩展 Automation 引擎**

```
性能优化，降低延迟
- 在 automation rule 中添加 ActionCallAIAgent
- 直接在 libredesk 进程内调用 Agent
```

**优势**:
- 响应更快（无网络往返）
- 条件和 Agent 调用原子化
- 与现有 automation 规则集成

**实现步骤**:
1. 定义 `ActionCallAIAgent` 常量和权限
2. 在 `ApplyAction` 中添加 case 分支
3. 实现 `Manager.InvokeAIAgent()` 方法
4. 创建 AI Agent gRPC/HTTP 客户端
5. 添加 UI 用于配置 Agent 参数

**预期工作量**: 3-4 周

**迁移路径**: 
- 保持 Webhook 仍然工作（可做 A/B 测试）
- 逐步迁移规则到新 action
- 验证结果一致性

---

### 6.3 关键设计决策

#### 决策 1：如何处理 RAG 知识库？

**推荐**: 在外部 Agent 服务中维护
```
┌─────────────┐         ┌──────────────────┐
│   客户消息  │────────▶ │ 外部 AI 服务      │
└─────────────┘         │ ┌──────────────┐  │
                        │ │ 向量数据库    │  │ ◀─ 知识库管理
                        │ │ (Weaviate)   │  │
                        │ └──────────────┘  │
                        │ └──────────────┐  │
                        │ │ LLM + Prompt│  │
                        │ └──────────────┘  │
                        └──────────────────┘
                              │
                              ▼
                        ┌─────────────┐
                        │  libredesk  │
                        │  回复/状态  │
                        └─────────────┘
```

**理由**:
- libredesk 不需要知道 RAG 的内部工作
- 知识库可独立版本管理和更新
- AI 服务可灵活选择向量库实现

#### 决策 2：Agent 的可配置性

**推荐等级**:

| 等级 | 功能 | 实现时间 |
|------|------|---------|
| L1 (MVP) | API Key 配置 → ✅ 已有 | - |
| L2 | 选择 Prompt Key | 需扩展 |
| L3 | 行为参数（温度、max_tokens） | 需改 OpenAI 客户端 |
| L4 | 多 Model 支持（GPT-4 vs o1） | 需改 provider 接口 |
| L5 | 自定义 System Prompt 每规则 | 需改 automation 规则 schema |

**推荐**: 从 L1+L2 开始，L3-L5 作为未来工作

---

#### 决策 3：Agent 错误处理

```
场景 1: Agent 超时或返回错误
→ 转交给人工（标记为 escalation_reason）

场景 2: Agent 不确定答案（confidence < 0.5）
→ 添加私人备注，并转交

场景 3: 知识库查询无结果
→ 使用 fallback 回复，并标记为二级问题

场景 4: API 故障（webhook 不可达）
→ 自动禁用 rule，告知管理员
```

---

### 6.4 实施路线图

```
┌─────────────────────────┐
│ 第一阶段：外部 Agent MVP │ (Week 1-3)
├─────────────────────────┤
│ ✓ 搭建 Python/Node 服务  │
│ ✓ 集成向量数据库         │
│ ✓ OpenAI 集成           │
│ ✓ Function calling 集成  │
│ ✓ Webhook 配置           │
│ ✓ 测试和验证            │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────────────┐
│ 第二阶段：性能优化               │ (Week 4-6)
├─────────────────────────────────┤
│ ✓ libredesk 修改（Automation）  │
│ ✓ 迁移评估规则                  │
│ ✓ A/B 测试对比                  │
│ ✓ 性能基准测试                  │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│ 第三阶段：功能完善               │ (Week 7+)
├─────────────────────────────────┤
│ ○ 多 Model 支持                 │
│ ○ 高级 Prompt 模版              │
│ ○ Agent 性能仪表盘              │
│ ○ 手动覆盖干预 UI               │
└─────────────────────────────────┘
```

---

## 7. 关键代码位置速查表

### API 层 (cmd/)

| 功能 | 文件 | 函数 |
|------|------|------|
| 发送消息 | cmd/messages.go | handleSendMessage() |
| 会话更新 | cmd/conversation.go | handleUpdateConversationStatus() |
| 自动化规则 | cmd/automation.go | handleCreateAutomationRule() |
| Webhook 管理 | cmd/webhooks.go | handleCreateWebhook() |
| AI 补全 | cmd/ai.go | handleAICompletion() |

### 业务逻辑层 (internal/)

| 模块 | 文件 | 核心类/方法 |
|------|------|-----------|
| 会话管理 | conversation/conversation.go | Manager.ApplyAction() |
| 自动化 | automation/automation.go | Engine.EvaluateNewConversationRules() |
| 自动化评估 | automation/evaluator.go | Engine.evaluateRule() |
| Webhook | webhook/webhook.go | Manager.TriggerEvent() |
| AI | ai/ai.go | Manager.Completion() |
| AI OpenAI | ai/openai.go | OpenAIClient.SendPrompt() |

### 数据模型 (internal/*/models/)

| 模块 | 结构体 | 用途 |
|------|--------|------|
| Automation | RuleRecord, Rule, RuleAction | 规则定义 |
| Webhook | Webhook, WebhookEvent | Webhook 配置 |
| AI | Prompt, Provider | AI 配置 |
| Conversation | Conversation, Message | 会话和消息 |

---

## 8. 注意事项和已知限制

### 8.1 当前系统的限制

1. **AI 功能受限**: 当前 AI 模块只支持单轮 Completion，无对话历史维护
2. **Provider 接口简单**: 仅支持 systemPrompt + userPrompt，无函数调用
3. **自动化无 Agent**: 无法在规则中调用外部 AI 服务
4. **Webhook 投递无重试**: 单次失败则丢弃（需手动配置重试）
5. **唯一的 LLM 实现**: 仅有 OpenAI，无其他 Provider
6. **API 认证**: 外部服务调用 API 需要 API Key（非 OAuth）

### 8.2 集成建议

**认证方案**:
```
推荐: API Key + HMAC 签名
- 为 AI Agent 服务创建专用 Agent 账户
- 生成 api_key 和 api_secret
- 所有 API 请求涂上 HMAC-SHA256 签名
```

**错误处理**:
```
不要: catch-all 异常处理
要: 分类处理
  - 429 (速率限制) → 指数退避重试
  - 401/403 (认证) → 告知管理员
  - 5xx → 自动转到人工
  - Timeout → 降级回复
```

**监控指标**:
```
关键 KPI:
  - Agent 响应时间 (P50/P95/P99)
  - 成功率 (Agent 能处理的 % )
  - 误差率 (需要人工干预的 % )
  - API 调用成本
```

---

## 9. 总结与决策

### 9.1 最终建议

**采用方案 A (Webhook + 外部 Agent)** 作为第一步：

| 优点 | 缺点 | 缓解方式 |
|------|------|---------|
| 零侵入 | 网络延迟 | 优化 webhook 处理逻辑 |
| 快速上线 | 错误处理复杂 | 完整的监控和告警 |
| 独立演进 | 知识库自建 | 使用托管 RAG 服务 |

**后续优化 (第二步)**：同时实施方案 B (扩展 Automation)，以寻求低延迟和高集成。

### 9.2 成功标准

- ✅ AI Agent 回复率 > 70%
- ✅ Agent 处理响应时间 < 3 秒
- ✅ 客户满意度提升 (基于 CSAT 分数)
- ✅ 人工工作量减少 >= 40%
- ✅ 系统稳定性不受影响 (99.9% uptime)

### 9.3 后续研究方向

1. **多 LLM 支持**: Claude, Llama, 本地模型
2. **上文含量管理**: 完整的会话历史压缩
3. **Agentic workflow**: 支持多步骤任务和子 Agent 调度
4. **微调模型**: 针对 libredesk 的特定领域模型
5. **离线 Agent**: 低网络环境下的本地部署

---

## 附录 A：数据库查询参考

### 获取启用的自动化规则

```sql
SELECT type, events, rules, execution_mode 
FROM automation_rules 
WHERE enabled = TRUE 
ORDER BY weight ASC;
```

### 获取活动的 Webhook

```sql
SELECT id, name, url, events, secret, is_active 
FROM webhooks 
WHERE is_active = TRUE 
  AND events @> ARRAY['message.created'];
```

### 获取会话消息历史

```sql
SELECT * FROM messages 
WHERE conversation_id = ? 
ORDER BY created_at ASC 
LIMIT 20;
```

---

## 附录 B：外部 AI Agent 实现框架

**参考架构** (Python):

```python
# ai_agent_service/main.py

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import httpx
import logging
from datetime import datetime

app = FastAPI()

class WebhookPayload(BaseModel):
    event: str
    data: dict

class AIAgentResponse(BaseModel):
    reply: str
    action: str  # "reply", "escalate", "close"
    assign_to_agent_id: int = None
    confidence: float

@app.post("/webhook")
async def handle_webhook(payload: WebhookPayload):
    """处理来自 libredesk 的 webhook"""
    
    if payload.event != "message.created":
        return {"status": "ignored"}
    
    message = payload.data
    conversation_uuid = message["conversation_uuid"]
    
    # 1. 获取会话上下文
    conv = await get_conversation(conversation_uuid)
    history = await get_messages(conversation_uuid)
    
    # 2. 调用 AI 决策
    decision = await decide_with_ai(
        customer_message=message["message"],
        conversation=conv,
        history=history,
        knowledge_base=kb  # RAG
    )
    
    # 3. 执行决策
    if decision.action == "escalate":
        await libredesk_api.put(
            f"/conversations/{conversation_uuid}/assignee/user",
            {"assignee_id": 1}  # 转给 agent 1
        )
    elif decision.action == "reply":
        await libredesk_api.post(
            f"/conversations/{conversation_uuid}/messages",
            {"message": decision.reply, "private": False}
        )
        
        if decision.confidence < 0.5:
            # 低信心，标记为私人备注
            await libredesk_api.post(
                f"/conversations/{conversation_uuid}/messages",
                {
                    "message": f"[AI] 低信心回复，建议人工审核 (confidence={decision.confidence:.2f})",
                    "private": True
                }
            )
    elif decision.action == "close":
        await libredesk_api.put(
            f"/conversations/{conversation_uuid}/status",
            {"status": 2}  # 已解决
        )
    
    return {"status": "processed", "action": decision.action}

async def decide_with_ai(customer_message, conversation, history, knowledge_base):
    """AI 决策逻辑"""
    
    # 从知识库检索相关文档
    relevant_docs = await knowledge_base.search(customer_message, top_k=3)
    
    # 构建 prompt
    context = "\n".join([
        f"- {doc['title']}: {doc['content']}"
        for doc in relevant_docs
    ])
    
    system_prompt = f"""
    You are a helpful customer service AI assistant for libredesk.
    Use the following knowledge base to answer questions:
    
    {context}
    
    If you cannot answer with high confidence, escalate to a human agent.
    """
    
    user_prompt = f"""
    Customer: {customer_message}
    
    Previous context:
    {format_history(history)}
    
    Please decide:
    1. Can you answer this question? (reply)
    2. Or should it go to a human? (escalate)
    3. Or is the issue resolved? (close)
    
    Respond with JSON:
    {{
        "action": "reply" | "escalate" | "close",
        "reply": "...",
        "confidence": 0.95
    }}
    """
    
    response = await call_openai(system_prompt, user_prompt)
    decision_json = parse_json_from_response(response)
    
    return AIAgentResponse(**decision_json)

async def call_openai(system, user):
    """调用 OpenAI API"""
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            "https://api.openai.com/v1/chat/completions",
            headers={
                "Authorization": f"Bearer {OPENAI_API_KEY}",
                "Content-Type": "application/json"
            },
            json={
                "model": "gpt-4o-mini",
                "messages": [
                    {"role": "system", "content": system},
                    {"role": "user", "content": user}
                ],
                "temperature": 0.7,
                "max_tokens": 1024
            }
        )
        return resp.json()["choices"][0]["message"]["content"]

async def get_conversation(uuid: str):
    """从 libredesk 获取会话"""
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"http://libredesk/api/v1/conversations/{uuid}",
            headers={"Authorization": f"Bearer {LIBREDESK_API_KEY}"}
        )
        return resp.json()["data"]

async def get_messages(uuid: str):
    """从 libredesk 获取消息历史"""
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"http://libredesk/api/v1/conversations/{uuid}/messages?per_page=10",
            headers={"Authorization": f"Bearer {LIBREDESK_API_KEY}"}
        )
        return resp.json()["data"]["results"]

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

**文档完成于**: 2026-04-10  
**版本**: 1.0  
**审核状态**: 初稿完成
