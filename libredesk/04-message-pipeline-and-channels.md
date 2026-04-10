# 04 — 消息流水线与渠道系统详细分析

> 基于 libredesk 源码深度分析，版本快照：2026-04-10

---

## 1. 消息生命周期图

### 1.1 入站 Email 完整流程

```
[IMAP 服务器]
      │
      │  (轮询，默认 5 分钟，可配置 read_interval)
      ▼
email.ReadIncomingMessages(ctx, cfg)          ← internal/inbox/channel/email/imap.go
  └─ email.processMailbox()
       ├─ 连接 IMAP（支持 TLS/STARTTLS/none）
       ├─ 认证（Password 或 OAuth2/XOAuth2）
       ├─ SEARCH messages since N hours ago（支持 ESEARCH 优化）
       ├─ 批量 FETCH envelope + 关键 headers
       │     检测: Auto-Submitted / X-Autoreply → 跳过自动回复
       │     检测: X-Libredesk-Loop-Prevention  → 跳过环路消息
       └─ 对每封邮件调用 processEnvelope()
              │
              ├─ Drop: 无 From 或无 MessageID
              ├─ 查询 messageStore.MessageExists(messageID)
              │    └─ 如已存在 → 跳过（防重入）
              ├─ 查询 userStore.GetContact(email) 检查是否封禁
              ├─ 构建 contact(umodels.User)
              ├─ 构建 incomingMsg(models.IncomingMessage)
              │     包含: Channel, SenderType, Type, InboxID, Status,
              │           Subject, SourceID(MessageID), Meta(from/to/cc/bcc)
              └─ FETCH 完整邮件 body → processFullMessage()
                     ├─ enmime 解析 RFC5322 邮件
                     ├─ 提取 HTML/Text 内容（优先 HTML）
                     ├─ 提取 In-Reply-To / References headers
                     ├─ 提取 plus-addressing UUID（Delivered-To/X-Original-To/To）
                     ├─ 处理附件 (Attachments) 和内联图片 (Inlines)
                     └─ messageStore.EnqueueIncoming(incomingMsg)
                              │
                              ▼
                    内存 Channel: incomingMessageQueue
                              │
                              ▼ (IncomingMessageWorker goroutine，10 个并发)
                    processIncomingMessage(in)             ← internal/conversation/message.go
                              │
                              ├─ userStore.CreateContact(&in.Contact)  → upsert 联系人
                              ├─ 去重: messageExistsBySourceID(sourceID)
                              │
                              ├─ 会话匹配 (优先级顺序):
                              │   1. plus-addressing Reply-To UUID 匹配
                              │      └─ 验证联系人邮箱一致 → 复用会话
                              │   2. Subject 中的引用号匹配 (#392)
                              │      └─ 验证联系人邮箱一致 → 复用会话
                              │   3. In-Reply-To / References 中的 MessageID 匹配
                              │      └─ 查找包含这些 SourceID 的旧消息所在会话
                              │   4. 以上均未匹配 → CreateConversation() 新建会话
                              │
                              ├─ uploadMessageAttachments() → 上传到 S3/LocalFS
                              │    失败时若是新会话则删除该会话（原子性保证）
                              │
                              ├─ InsertMessage(&in.Message)
                              │     ├─ 写入 DB
                              │     ├─ 附加 Media 到消息
                              │     ├─ 更新会话 last_message
                              │     ├─ BroadcastNewMessage()  → WebSocket 推送
                              │     └─ webhookStore.TriggerEvent(message.created)
                              │
                              ├─ [新会话]:
                              │     ├─ webhookStore.TriggerEvent(conversation.created)
                              │     └─ automation.EvaluateNewConversationRules()
                              │
                              └─ [已有会话]:
                                    ├─ ReOpenConversation()（若非 Open 状态）
                                    ├─ UpdateConversationWaitingSince()
                                    ├─ automation.EvaluateConversationUpdateRules(
                                    │     EventConversationMessageIncoming)
                                    └─ slaStore.CreateNextResponseSLAEvent()
                                         └─ BroadcastConversationUpdate(next_response_deadline_at)
```

**WebSocket 推送路径**（在 `BroadcastNewMessage` 中触发）：
```
conversation.BroadcastNewMessage(message)
  └─ broadcastToUsers([], wsmodels.Message{type: "new_message", data: {...}})
       └─ wsHub.BroadcastMessage(BroadcastMessage{Data: bytes, Users: []})
            └─ 遍历所有在线用户的所有 Client.SendMessage()
                 └─ Client.Send channel → Client.Serve() goroutine → WebSocket 推送
```

**通知路径**（由自动化或 SLA 规则触发）：
```
automation.ApplyAction() / sla 规则
  └─ dispatcher.Send(Notification{...})
       ├─ inApp.Create()       → 写入 DB (user_notifications 表)
       ├─ broadcastNotification() → wsHub → WebSocket 推送 (new_notification)
       └─ outbound.Send(email) → notification.Service channel → 发送通知邮件
```

---

### 1.2 出站消息（Agent 发送）完整流程

```
[Agent 前端 - 点击发送]
      │  POST /api/conversations/{uuid}/messages
      ▼
handleSendMessage(r)                          ← cmd/messages.go
      ├─ 鉴权 & 权限检查 (enforceConversationAccess)
      ├─ 解析 req: {message, to, cc, bcc, attachments, private, sender_type}
      │
      ├─ [私信/内部备注 private=true]:
      │     conversation.SendPrivateNote()
      │           ├─ InsertMessage(status=sent)  → 立即标记为已发送
      │           ├─ BroadcastNewMessage()       → WebSocket 通知
      │           └─ NotifyMention()             → 如有 @提及则通知
      │
      ├─ [以联系人身份发送]:
      │     conversation.CreateContactMessage()
      │           └─ InsertMessage(type=incoming, status=received)
      │
      └─ [普通 Agent 回复]:
            conversation.QueueReply(media, inboxID, senderID, uuid, content, to, cc, bcc, meta)
                  ├─ 生成 sourceID = GenerateEmailMessageID(convUUID, inbox.From)
                  │     格式: <convUUID.timestamp@inbox-domain>
                  ├─ InsertMessage(status=pending)  → 写入 DB，状态=pending
                  │     ├─ BroadcastNewMessage()    → WebSocket 实时推送给前端
                  │     └─ webhookStore.TriggerEvent(message.created)
                  └─ 返回消息给前端（此时状态为 pending）

[后台 DB 扫描 goroutine - 每 50ms]
      ├─ GetOutgoingPendingMessages (跳过正在处理中的)
      └─ 将消息 ID 放入 outgoingProcessingMessages map
           └─ 推送到 outgoingMessageQueue channel

[MessageSenderWorker goroutine × 10]
      ├─ 从 outgoingMessageQueue 取出消息
      └─ sendOutgoingMessage(message)
              ├─ inboxStore.Get(inboxID)  → 获取 Inbox 实例
              ├─ RenderMessageInTemplate()
              │     └─ template.RenderEmailWithTemplate(data, content)
              │          注入: Conversation, Contact, Author 变量
              ├─ attachAttachmentsToMessage()  → 从 mediaStore 读取附件内容
              ├─ message.From = inbox.FromAddress()
              ├─ GetMessageSourceIDs(convID, 20) → 构建 References header
              │     最近 20 条消息的 SourceID，倒序排列
              ├─ message.InReplyTo = references[last]
              │
              ├─ inbox.Send(message)            ← email.Send() via SMTP Pool
              │     ├─ refreshOAuthIfNeeded()   → 自动刷新 OAuth token
              │     ├─ 随机选择一个 SMTP Server（多 SMTP 负载均衡）
              │     ├─ 设置邮件头:
              │     │     X-Libredesk-Loop-Prevention: inbox-email
              │     │     Reply-To: inbox+conv-{uuid}@domain (plus-addressing)
              │     │     In-Reply-To: <last-message-id>
              │     │     References: <msg1> <msg2> ...
              │     │     Message-ID: <sourceID>
              │     │     X-Libredesk-Conversation-UUID: {uuid}
              │     └─ smtppool.Send(email)
              │
              ├─ [成功]: UpdateMessageStatus(sent)
              │           └─ BroadcastMessageUpdate(prop="status", value="sent")
              │                → WebSocket 通知前端更新消息状态
              │
              ├─ [失败]: UpdateMessageStatus(failed)
              │           └─ BroadcastMessageUpdate(prop="status", value="failed")
              │
              └─ [非系统用户]:
                    ├─ UpdateConversationFirstReplyAt / LastReplyAt
                    ├─ UpdateConversationWaitingSince(nil)  → 清除等待时间
                    ├─ slaStore.SetLatestSLAEventMetAt(next_response) → SLA 标记满足
                    └─ automation.EvaluateConversationUpdateRules(
                            EventConversationMessageOutgoing)
```

**出站消息失败重试**：
```
Agent 前端 → PUT /api/conversations/{cuuid}/messages/{uuid}/retry
  └─ handleRetryMessage()
       └─ conversation.MarkMessageAsPending(uuid)
            └─ UpdateMessageStatus(pending)
                 └─ DB 扫描 goroutine 下次扫描时重新入队
```

---

## 2. Envelope 设计

### 2.1 澄清：libredesk 中有两个"Envelope"概念

**注意**：`internal/envelope/envelope.go` 中的 `Envelope` **并非**消息信封（Message Envelope），而是 **API 错误封装包**。真正的消息信封概念体现在 `models.IncomingMessage` 中。

#### 2.1.1 `internal/envelope/envelope.go` — API 错误包装器

```go
// API 错误类型常量
const (
    GeneralError      = "GeneralException"
    PermissionError   = "PermissionException"
    InputError        = "InputException"
    DataError         = "DataException"
    NetworkError      = "NetworkException"
    NotFoundError     = "NotFoundException"
    ConflictError     = "ConflictException"
    UnauthorizedError = "UnauthorizedException"
)

// Error — 统一的 API 错误类型
type Error struct {
    Code      int         // HTTP 状态码
    ErrorType string      // 错误分类
    Message   string      // 错误消息
    Data      interface{} // 附加数据
}

// PageResults — 分页结果包装
type PageResults struct {
    Results    interface{} `json:"results"`
    Total      int         `json:"total"`
    PerPage    int         `json:"per_page"`
    TotalPages int         `json:"total_pages"`
    Page       int         `json:"page"`
}
```

- **作用**：全系统统一错误响应格式，所有 handler 通过 `sendErrorEnvelope(r, err)` 返回标准化错误
- **HTTP 状态码映射**：`GeneralError→500, PermissionError→403, InputError→400, NotFoundError→404` 等

#### 2.1.2 `models.IncomingMessage` — 真正的消息信封

```go
// 位于 internal/conversation/models/models.go
// （IncomingMessage 定义通过 grep 确认位于 conversation/models）

type IncomingMessage struct {
    Message                    models.Message  // 消息主体
    Contact                    umodels.User    // 联系人信息（含 InboxID）
    InboxID                    int             // 所属收件箱 ID
    ConversationUUIDFromReplyTo string         // plus-addressing 提取的会话 UUID
}
```

**字段含义**：
| 字段 | 含义 |
|------|------|
| `Message` | 消息主体，包含 Channel、Content、SourceID(MessageID)、Attachments、InReplyTo、References 等 |
| `Contact` | 发件人联系人信息，包含 Email、Name、SourceChannel、SourceChannelID，用于 upsert 联系人 |
| `InboxID` | 接收该消息的收件箱 ID（收件箱与渠道绑定） |
| `ConversationUUIDFromReplyTo` | 从 plus-addressed Reply-To 中解析出的会话 UUID，用于会话匹配 |

**`models.Message` 的完整字段**：
```go
type Message struct {
    // DB 字段
    ID               int             // 自增 ID
    UUID             string          // 公开标识符
    CreatedAt        time.Time
    UpdatedAt        time.Time
    Type             string          // incoming / outgoing / activity
    Status           string          // received / sent / failed / pending
    ConversationID   int             // 关联会话 ID (DB join key)
    ConversationUUID string          // 关联会话 UUID (路由标识)
    Content          string          // 消息 HTML/plaintext 内容
    TextContent      string          // 纯文本版（用于搜索、last_message 展示）
    ContentType      string          // text / html
    Private          bool            // 是否为内部备注
    SourceID         null.String     // 渠道原始 ID（邮件为 MessageID）
    SenderID         int             // 发送人用户 ID
    SenderType       string          // agent / contact
    InboxID          int             // 所属收件箱
    Meta             json.RawMessage // 扩展元数据（to/from/cc/bcc/subject 等）
    Attachments      attachment.Attachments // 附件列表

    // 邮件发送专用字段（仅内存使用，不存 DB）
    From             string          // 发件人地址
    Subject          string          // 邮件主题
    Channel          string          // 渠道类型
    To               pq.StringArray  // 收件人列表
    CC               pq.StringArray
    BCC              pq.StringArray
    References       []string        // 邮件线程引用链
    InReplyTo        string          // 回复的消息 ID
    Headers          textproto.MIMEHeader // 自定义邮件头
    AltContent       string          // 纯文本备用内容（HTML 邮件的纯文本版）
    Media            []mmodels.Media // 上传的媒体文件
    IsCSAT           bool            // 是否为 CSAT 调查消息
}
```

### 2.2 传递路径

```
email.imap.go
  processFullMessage()
    → 构建 models.IncomingMessage（包含 models.Message）
    → messageStore.EnqueueIncoming(incomingMsg)
         │ (interface: inbox.MessageStore)
         ▼
conversation.Manager.incomingMessageQueue（内存 chan）
         │
         ▼
processIncomingMessage(in models.IncomingMessage)
  → 消息持久化前的所有字段都在 IncomingMessage 中流转
  → InsertMessage(&in.Message)  → 写入 DB
```

### 2.3 为什么需要 IncomingMessage 中间抽象

1. **解耦渠道与业务逻辑**：IMAP goroutine 构建 `IncomingMessage`，不直接调用 `conversation.Manager`，通过 `inbox.MessageStore` 接口（`EnqueueIncoming`）解耦
2. **异步处理**：IMAP 是密集型 I/O 操作，通过 channel 异步化，不阻塞 IMAP 轮询
3. **携带上下文**：`IncomingMessage` 同时携带消息体 + 联系人 + 邮件特有信息（plus-address UUID），避免多次跨层查询
4. **背压机制**：channel 有 size 限制（默认 5000），EnqueueIncoming 在 channel 满时立即返回错误，防止内存爆炸

---

## 3. Channel 接口分析

### 3.1 `inbox.Inbox` 接口定义

位于 `internal/inbox/inbox.go`：

```go
// Inbox 是所有渠道必须实现的完整接口
type Inbox interface {
    Closer        // Close() error
    Identifier    // Identifier() int  — 返回 DB id
    MessageHandler // Receive(ctx) error + Send(models.Message) error
    FromAddress() string  // 发件人地址
    Channel() string      // 渠道标识符 ("email")
}

// 接口分解
type Closer interface {
    Close() error
}
type Identifier interface {
    Identifier() int
}
type MessageHandler interface {
    Receive(context.Context) error
    Send(models.Message) error
}
```

### 3.2 Email 渠道的实现

`internal/inbox/channel/email/email.go` 中 `*Email` 实现所有接口方法：

| 方法 | 实现逻辑 |
|------|----------|
| `Identifier() int` | 返回 DB inbox.id |
| `Channel() string` | 返回常量 `"email"` |
| `FromAddress() string` | 返回配置的 from 地址 |
| `Receive(ctx) error` | 为每个 IMAPConfig 启动 goroutine 调用 `ReadIncomingMessages()` |
| `Send(models.Message) error` | 通过 SMTP Pool 发送邮件，支持 OAuth token 自动刷新 |
| `Close() error` | 关闭所有 SMTP Pool 连接 |

### 3.3 渠道注册与初始化机制

```go
// 1. 启动时调用 initFn 初始化所有 active 收件箱
inbox.Manager.InitInboxes(initFn initFn)

// 2. initFn 在 cmd/main.go / cmd/init.go 中定义:
initFn = func(record imodels.Inbox, msgStore, userStore) (inbox.Inbox, error) {
    switch record.Channel {
    case "email":
        var cfg imodels.Config
        json.Unmarshal(record.Config, &cfg)
        return email.New(msgStore, userStore, email.Opts{
            ID:                   record.ID,
            Config:               cfg,
            Lo:                   lo,
            TokenRefreshCallback: ...  // OAuth token 持久化回调
        })
    // 未来可扩展: case "telegram": ...
    default:
        return nil, fmt.Errorf("unknown channel: %s", record.Channel)
    }
}

// 3. 启动接收 goroutine
inbox.Manager.StartReceivers(ctx) 或在 Reload() 中启动:
for _, inb := range m.inboxes {
    go inb.Receive(ctx)  // 每个收件箱独立 goroutine
}
```

### 3.4 消息收发的抽象程度评估

**收消息（Receive）**：
- 接口层面只定义 `Receive(ctx) error`，任何实现（IMAP 轮询、HTTP Webhook、长连接）均可
- 消息最终必须调用 `messageStore.EnqueueIncoming(models.IncomingMessage)`
- 问题：`IncomingMessage.Contact` 和字段设计目前**高度依赖 email 语义**（has Email, InboxID 等）

**发消息（Send）**：
- 接口层面只接受 `models.Message`
- `Message` struct 包含大量 email 专有字段：`To`、`CC`、`BCC`、`References`、`InReplyTo`、`Headers`
- `RenderMessageInTemplate()` 方法有 `switch channel { case "email": ... default: return error }`，**硬编码只支持 email**

---

## 4. Email 渠道实现细节

### 4.1 支持的协议与认证方式

| 维度 | 支持情况 |
|------|----------|
| **接收协议** | IMAP（v2 协议，通过 `go-imap/v2` 库）|
| **发送协议** | SMTP（通过 `knadh/smtppool` 连接池）|
| **IMAP TLS** | `none` / `starttls` / `tls`（含 InsecureSkipVerify 选项）|
| **SMTP TLS** | `none` / `starttls` / `tls`（含 SSL 直连模式）|
| **认证方式** | Password 模式：plain / cram-md5 / login / none |
| | OAuth2 模式：XOAuth2 SASL（Google/Microsoft）|
| **多 SMTP** | 支持配置多个 SMTP 服务器，随机负载均衡 |
| **OAuth 刷新** | 自动检测 token 过期，使用 refresh_token 续期，并通过 callback 持久化 |

### 4.2 邮件解析逻辑（enmime 库）

```
FETCH 分两次:
  第一次: Fetch envelope + 关键 headers（轻量，用于去重和防环检测）
  第二次: Fetch 完整 body（仅新邮件执行）

解析步骤:
  1. enmime.ReadEnvelope() 解析 RFC5322 邮件
  2. HTML 提取: 遍历 MIME 树收集所有 text/html part，拼接为 <div>...</div>
     优先级: 树遍历 HTML > envelope.HTML > envelope.Text
  3. InReplyTo 清洗: 去除 <> 括号
  4. References 清洗: 按空格分割，逐个去除 <> 和空白
  5. plus-addressing: 从 Delivered-To / X-Original-To / To header 提取 conv UUID
```

### 4.3 附件处理

| 类型 | 处理方式 |
|------|----------|
| **普通附件** | `envelope.Attachments` → `attachment.Attachment{Disposition: attachment}` |
| **内联附件（有 ContentID）** | `envelope.Inlines` → `attachment.Attachment{Disposition: inline}` |
| **内联附件（无 ContentID）** | 降级为普通附件 (`Disposition: attachment`) |
| **上传**（processIncomingMessage） | `mediaStore.UploadAndInsert()` → 替换 HTML 中的 `cid:xxx` 为 `/uploads/{uuid}` |
| **重复 ContentID 检测** | content_id 加上 conversation_uuid 前缀后查重，已存在则直接复用 URL |
| **出站附件** | 从 `mediaStore.GetBlob(name)` 读取内容 → 转换为 `smtppool.Attachment` |

### 4.4 邮件线程关联算法

系统使用三层关联策略（优先级从高到低）：

```
策略 1: Plus-Addressing Reply-To
  条件: 收件箱开启 enable_plus_addressing，
        上次回复时 Reply-To 设置为 support+conv-{uuid}@domain
  匹配: 从 Delivered-To / X-Original-To / To 提取 UUID
  验证: 联系人邮箱与会话联系人邮箱一致

策略 2: Subject 中的引用号
  格式: "RE: 原主题 - #392"
  匹配: stringutil.ExtractReferenceNumber(subject)
  验证: 联系人邮箱匹配

策略 3: In-Reply-To / References header 链
  匹配: messageExistsBySourceID([inReplyTo, ...references])
        → 查找包含这些 MessageID 的已有消息所在会话
  无需邮箱验证（MessageID 全局唯一性保证）

策略 4: 新建会话
  所有策略均未匹配时，CreateConversation() 创建新会话
```

**出站增强**：
```go
// 发送回复时:
email.Headers.Set("Reply-To", "support+conv-{uuid}@domain")  // plus-addressing
email.Headers.Set("References", "<id1> <id2> ...")           // 线程追踪
email.Headers.Set("In-Reply-To", "<last-msg-id>")            // 直接回复引用
email.Headers.Set("X-Libredesk-Conversation-UUID", "{uuid}") // 自定义头（辅助追踪）
email.Headers.Set("X-Libredesk-Loop-Prevention", "inbox@domain") // 防环
```

---

## 5. 消息队列机制

### 5.1 入站/出站队列实现

libredesk **没有使用 Redis 或外部消息队列**，所有队列均为 **Go 内存 channel**：

```go
// 位于 conversation.Manager（internal/conversation/conversation.go）
incomingMessageQueue chan models.IncomingMessage  // 容量: incoming_queue_size
outgoingMessageQueue chan models.Message          // 容量: outgoing_queue_size
outgoingProcessingMessages sync.Map              // 正在处理的出站消息 ID 集合（防重复）
```

**通知队列**：
```go
// 位于 notification.Service（internal/notification/notification.go）
messageChannel chan Message  // 容量: notification.queue_size
```

**自动化规则队列**：
```go
// 位于 automation.Engine（internal/automation/automation.go）
taskQueue chan ConversationTask  // 容量: MaxQueueSize = 10000
```

**Webhook 队列**：
```go
// 位于 webhook.Manager（internal/webhook/）
// 独立队列，容量: webhook.queue_size = 10000
```

### 5.2 Worker 数量和配置（config.sample.toml）

| 队列 | Worker 数 | 队列容量 | 配置项 |
|------|-----------|---------|--------|
| 入站消息处理 | `incoming_queue_workers = 10` | `incoming_queue_size = 5000` | `[message]` |
| 出站消息发送 | `outgoing_queue_workers = 10` | `outgoing_queue_size = 5000` | `[message]` |
| 通知发送 | `concurrency = 2` | `queue_size = 2000` | `[notification]` |
| 自动化规则 | `worker_count = 10` | `MaxQueueSize = 10000` | `[automation]` |
| Webhook 投递 | `workers = 5` | `queue_size = 10000` | `[webhook]` |
| 自动分配 | 1（ticker goroutine） | — | `autoassign_interval = "5m"` |

**出站消息 DB 扫描**：
```
scan_interval = "50ms"  每 50ms 扫描一次 pending 状态的出站消息
```

### 5.3 错误重试机制

**出站消息失败重试**（手动触发）：
```
1. 消息发送失败 → 状态更新为 "failed"
2. Agent 在前端点击"重试" → PUT /messages/{uuid}/retry
3. handleRetryMessage() 验证:
   - 消息必须是 agent 消息
   - 状态必须为 "failed"
   - 只允许发送者重试
4. MarkMessageAsPending(uuid) → 状态改为 "pending"
5. DB 扫描 goroutine 在下次扫描时重新入队发送
```

**SMTP 内置重试**：
```
smtppool 配置: max_msg_retries (SMTPConfig.MaxMessageRetries)
每个 SMTP pool 内部支持消息级别重试
```

**入站消息：无自动重试**
- 如果 `EnqueueIncoming` 失败（队列满），该消息会被丢弃并记录日志
- IMAP 轮询是幂等的（基于 MessageID 去重），下次轮询仍会尝试同一封邮件

**出站消息：无自动重试**
- 网络故障/SMTP 错误后，消息状态变为 `failed`，需 Agent 手动点击重试
- `smtppool` 的 `MaxMessageRetries` 是连接层重试，不是业务层重试

---

## 6. 渠道扩展性评估

### 6.1 当前扩展障碍清单

添加一个新渠道（以 Telegram 为例）需要面对的问题：

#### 6.1.1 必须实现的接口

```go
// 必须实现 internal/inbox/inbox.go 的 Inbox 接口
type TelegramInbox struct { /* ... */ }

func (t *TelegramInbox) Identifier() int           { return t.id }
func (t *TelegramInbox) Channel() string           { return "telegram" }
func (t *TelegramInbox) FromAddress() string       { return "@botname" }
func (t *TelegramInbox) Close() error              { /* 关闭 webhook/polling */ }
func (t *TelegramInbox) Receive(ctx context.Context) error {
    // 方案A: 长轮询 Telegram Bot API getUpdates
    // 方案B: 接收 Telegram Webhook HTTP 请求
}
func (t *TelegramInbox) Send(msg models.Message) error {
    // 调用 Telegram Bot API sendMessage
    // 需从 models.Message 中提取 content，忽略邮件专有字段
}
```

**挑战**：`Receive()` 接口设计为阻塞式长运行 goroutine（适合 IMAP polling），对于 Telegram webhook 模式，`Receive()` 需要启动 HTTP server 或监听某种推送事件，与接口语义略有不符。

#### 6.1.2 需要修改的文件

| 文件 | 修改内容 |
|------|----------|
| `schema.sql` | **必须**：`ALTER TYPE channels ADD VALUE 'telegram';` |
| `cmd/init.go` | 在 `initFn` 的 switch 中添加 `case "telegram": return telegram.New(...)` |
| `internal/conversation/message.go` | `RenderMessageInTemplate()` 的 switch 中添加 `case "telegram"` 或重构为通用逻辑 |
| `internal/inbox/inbox.go` | 添加常量 `ChannelTelegram = "telegram"` |
| `internal/inbox/channel/telegram/` | 新建渠道实现包 |
| `cmd/inboxes.go` | 可能需要添加 Telegram 专有的 inbox 创建/配置 handler |
| `frontend/src/` | 添加 Telegram Inbox 的配置 UI 界面 |
| `i18n/*.json` | 添加翻译字符串 |

#### 6.1.3 数据库变更

```sql
-- schema.sql 中当前定义：
CREATE TYPE "channels" AS ENUM ('email');

-- 扩展需要：
ALTER TYPE channels ADD VALUE 'telegram';
-- 同时需要考虑 inbox.config JSONB 的结构变化（telegram 有独立配置字段）
```

#### 6.1.4 前端需要的配套变更

1. **Inbox 创建 UI**：添加 Telegram 收件箱配置表单（Bot Token、Webhook URL 等）
2. **`inbox.Config` 类型区分**：前端按 channel 类型展示不同配置字段
3. **消息发送界面**：Telegram 无 To/CC/BCC 字段，需隐藏这些输入框
4. **消息格式**：Telegram 支持 Markdown，需适配 editor

### 6.2 当前抽象层对非邮件渠道的支持评估

#### 设计较好的部分（✅）

| 设计点 | 评价 |
|--------|------|
| `Inbox` 接口定义 | 简洁，5 个方法，非邮件渠道均可实现 |
| 渠道配置存储 | `inbox.config` 为 JSONB，不同渠道可存不同结构 |
| `initFn` 回调模式 | 类工厂模式，新渠道只需扩展 switch case |
| 热重载 `Reload()` | 渠道实例可运行时重新初始化 |
| WebSocket 广播 | 与渠道无关，所有渠道共用同一 WS 推送机制 |
| 自动化规则 | 以 Conversation 为单位执行，也与渠道无关 |

#### 设计需改进的部分（⚠️）

| 问题 | 影响 | 改进建议 |
|------|------|---------|
| `models.Message.To/CC/BCC` 为 email 语义 | Telegram 等渠道无收件人概念，字段冗余 | 抽取 email 专有字段到 `EmailMeta` 子结构 |
| `RenderMessageInTemplate()` 硬编码 email | 添加新渠道会触发 `"unknown message channel"` 错误 | 重构为接口 or 让 Inbox 实现 `RenderContent(msg) string` |
| `IncomingMessage.Contact.Email` 必填假设 | 部分代码用 email 匹配联系人，Telegram 用 user_id | 统一用 `SourceChannel + SourceChannelID` 作为联系人标识 |
| 会话匹配逻辑全为 email 专有 | plusaddressing/References/InReplyTo 对实时聊天无意义 | 允许 `ConversationUUIDFromReplyTo` 为可选，非 email 渠道直接按 SourceChannelID 匹配 |
| IMAP polling 是拉取模型 | 实时聊天（Telegram/WhatsApp）是推送模型 | `Receive()` 接口语义已经足够抽象，Telegram 实现可以做成内置 webhook |
| `GenerateEmailMessageID()` 在 QueueReply 中调用 | 对非邮件渠道生成了无意义的 email message-id | 抽象为 `GenerateSourceID(channel, ...)` 由渠道决定格式 |

### 6.3 添加 Telegram 渠道所需架构调整总结

**最小可行改动（MVP 路径）**：

```
Step 1: 数据库
  ALTER TYPE channels ADD VALUE 'telegram';

Step 2: 新建渠道包
  internal/inbox/channel/telegram/
    telegram.go    — 实现 inbox.Inbox 接口
    webhook.go     — 接收 Telegram webhook 推送（或长轮询）
    send.go        — 调用 Telegram Bot API 发送消息

Step 3: 注册渠道
  cmd/init.go → initFn switch 添加 case "telegram"

Step 4: 修复 RenderMessageInTemplate
  添加 case "telegram": 直接使用 message.Content，不套 email 模板

Step 5: 前端
  添加 Telegram Inbox 配置界面
  消息发送界面隐藏 To/CC/BCC（对非 email 渠道）
```

**深层架构重构（推荐长期改进）**：
1. 将 `models.Message` 中的 email 专有字段移到 `Meta` JSONB 中
2. 在 `Inbox` 接口加入 `RenderOutgoing(msg) (models.Message, error)` 方法，由各渠道自行处理模板渲染
3. 会话匹配策略抽象为 `ConversationMatcher` 接口，email 和 messaging 渠道各自实现
4. 联系人标识统一用 `SourceChannel` + `SourceChannelID`，而非依赖 email 字段
5. 考虑是否需要将实时聊天渠道（Telegram/WhatsApp）的 `Receive()` 模式改为 WebSocket/Webhook 注册接口，而非 goroutine 轮询

---

## 附录：关键代码路径速查

| 功能 | 文件 | 核心函数 |
|------|------|---------|
| IMAP 接收邮件 | `internal/inbox/channel/email/imap.go` | `ReadIncomingMessages()` → `processEnvelope()` → `processFullMessage()` |
| 消息入队 | `internal/conversation/message.go` | `EnqueueIncoming()` |
| 入站消息处理 | `internal/conversation/message.go` | `processIncomingMessage()` |
| 会话匹配/创建 | `internal/conversation/message.go` | `findOrCreateConversation()` |
| 消息写 DB | `internal/conversation/message.go` | `InsertMessage()` |
| Agent 发送接口 | `cmd/messages.go` | `handleSendMessage()` |
| 消息入出站队列 | `internal/conversation/message.go` | `QueueReply()` |
| 出站消息发送 | `internal/conversation/message.go` | `sendOutgoingMessage()` |
| SMTP 发送 | `internal/inbox/channel/email/smtp.go` | `Email.Send()` |
| WS 广播 | `internal/conversation/ws.go` | `BroadcastNewMessage()` / `BroadcastMessageUpdate()` |
| WS Hub | `internal/ws/ws.go` | `Hub.BroadcastMessage()` |
| WS 客户端连接 | `cmd/websocket.go` | `handleWS()` |
| 自动化触发 | `internal/automation/automation.go` | `EvaluateNewConversationRules()` / `EvaluateConversationUpdateRules()` |
| 自动分配 | `internal/autoassigner/autoassigner.go` | `Engine.assignConversations()` |
| 通知发送 | `internal/notification/dispatcher.go` | `Dispatcher.Send()` |
