# Prompt 04: 消息流水线与渠道系统详细分析

## 任务

这是最重要的分析之一。请深入追踪 libredesk 的消息处理全链路，以及渠道(channel)抽象层的设计，输出一份**消息流水线与渠道扩展性分析文档**。

## 需要分析的关键文件

按消息流转顺序追踪：

### 入站消息路径（外部 → 系统）
1. `internal/inbox/channel/email/` — 所有文件，email 渠道实现
2. `internal/inbox/inbox.go` — 收件箱管理
3. `internal/inbox/models/` — 收件箱数据模型
4. `internal/envelope/envelope.go` — 消息信封定义
5. `cmd/messages.go` — 消息处理 handler
6. `cmd/inboxes.go` — 收件箱 handler
7. `internal/conversation/message.go` — 消息持久化
8. `internal/conversation/conversation.go` — 会话创建/更新

### 出站消息路径（系统 → 外部）
1. `cmd/messages.go` — agent 发送消息的 handler
2. `internal/conversation/message.go` — 出站消息处理
3. `internal/inbox/channel/email/` — email 发送实现
4. `internal/notification/` — 通知发送

### 实时推送路径
1. `internal/ws/` — WebSocket hub 和 client
2. `internal/conversation/ws.go` — 会话 WebSocket 事件
3. `cmd/websocket.go` — WebSocket handler

### 自动化处理路径
1. `internal/automation/` — 消息到达后触发的自动化
2. `internal/autoassigner/` — 自动分配

## 输出要求

写入文件 `/Users/wenkai/workspace/knowledges/libredesk/04-message-pipeline-and-channels.md`，包含：

### 1. 消息生命周期图（文字版）
- 一条入站 email 从「IMAP/webhook 接收」→「解析」→「创建 Envelope」→「匹配/创建 Conversation」→「存储 Message」→「触发 Automation」→「WebSocket 推送」→「通知」的完整流程
- 一条出站消息从「Agent 点击发送」→「API 调用」→「消息入队」→「渠道发送」→「状态更新」→「WebSocket 通知」的完整流程

### 2. Envelope 设计
- Envelope struct 的所有字段及含义
- Envelope 在系统中的传递路径
- 为什么需要 Envelope 这个中间抽象

### 3. Channel 接口分析
- Channel 接口定义（如果有的话）或 email 实现暴露的方法签名
- 渠道注册和初始化机制
- 消息收发的抽象程度

### 4. Email 渠道实现细节
- 支持的协议（IMAP/SMTP/OAuth）
- 邮件解析逻辑
- 附件处理
- 邮件线程关联（如何关联到已有 Conversation）

### 5. 消息队列机制
- 入站/出站队列的实现方式（内存队列？Redis？）
- Worker 数量和配置
- 错误重试机制

### 6. 渠道扩展性评估
基于当前代码，评估：
- 添加一个新渠道（如 Telegram/WhatsApp）需要：
  - 实现哪些接口/方法
  - 修改哪些文件
  - 数据库需要哪些变更（schema.sql 中的 ENUM）
  - 前端需要哪些配套变更
- 当前抽象层是否足够支持非 email 渠道（实时聊天类）
- 需要做哪些架构调整
