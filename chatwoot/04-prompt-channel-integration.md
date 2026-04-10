# Prompt 4: Chatwoot 全渠道集成设计分析

## 任务
深入分析 Chatwoot 的全渠道（Omnichannel）集成设计，重点关注 Telegram 和 WhatsApp 渠道的实现细节，输出渠道集成架构文档。

## 输出要求
将分析结果写入 `/Users/wenkai/workspace/knowledges/chatwoot/output/04-channel-integration.md`，使用中文撰写。

## 需要分析的文件和目录

### Channel 模型层（全部阅读）
- `app/models/channel/web_widget.rb` — 网页聊天
- `app/models/channel/facebook_page.rb` — Facebook
- `app/models/channel/instagram.rb` — Instagram
- `app/models/channel/twitter_profile.rb` — Twitter
- `app/models/channel/telegram.rb` — **重点** Telegram
- `app/models/channel/whatsapp.rb` — **重点** WhatsApp
- `app/models/channel/twilio_sms.rb` — Twilio SMS
- `app/models/channel/sms.rb` — SMS
- `app/models/channel/line.rb` — LINE
- `app/models/channel/email.rb` — Email
- `app/models/channel/api.rb` — API Channel（自定义渠道）
- `app/models/channel/tiktok.rb` — TikTok

### Inbox 模型
- `app/models/inbox.rb` — **完整阅读**，理解 Inbox 如何与 Channel 多态关联

### Telegram 渠道（完整分析）
- `app/models/channel/telegram.rb`
- `app/services/telegram/` 目录下所有文件
- `app/controllers/webhooks/telegram_controller.rb`
- 搜索 `telegram` 相关的 job 和 listener

### WhatsApp 渠道（完整分析）
- `app/models/channel/whatsapp.rb`
- `app/services/whatsapp/` 目录下所有文件
- `app/controllers/webhooks/whatsapp_controller.rb`
- 搜索 `whatsapp` 相关的 job 和 listener

### API Channel（重要 — 自定义集成通道）
- `app/models/channel/api.rb`
- 搜索 API channel 相关的控制器和服务

### Webhook 接收
- `app/controllers/webhooks/` 目录下所有文件

### 渠道创建流程
- `app/controllers/api/v1/accounts/inboxes_controller.rb` — 创建 inbox 的 API
- `app/builders/` 目录 — 相关的 Builder

## 输出文档结构要求

```markdown
# Chatwoot 全渠道集成设计

## 1. 渠道架构总览
### 1.1 Inbox-Channel 多态模型
（Inbox 与 Channel 的关系，Channel 如何实现多态）
### 1.2 支持的渠道列表与状态
（每个渠道一行：名称、类型、接入方式、消息能力概要）
### 1.3 渠道消息流通用流程
（入站消息: 外部平台 → Webhook → Service → Message 创建）
（出站消息: Agent 回复 → Service → 外部平台 API）

## 2. Telegram 渠道详细设计
### 2.1 接入配置（Bot Token）
### 2.2 Webhook 注册机制
### 2.3 入站消息处理流程（收到用户消息）
### 2.4 出站消息处理流程（Agent 回复）
### 2.5 支持的消息类型
### 2.6 限制与注意事项

## 3. WhatsApp 渠道详细设计
### 3.1 接入方式（Cloud API vs BSP vs Twilio）
### 3.2 Webhook 注册与验证
### 3.3 入站消息处理流程
### 3.4 出站消息处理流程
### 3.5 消息模板（HSM）支持
### 3.6 支持的消息类型
### 3.7 限制与注意事项

## 4. API Channel（自定义渠道）
### 4.1 设计理念
### 4.2 入站消息 API
### 4.3 出站消息回调
### 4.4 使用场景（可以用来接入自定义 IM）

## 5. 添加新渠道的扩展模式
（如果要添加一个新渠道，需要实现哪些模块）
```
