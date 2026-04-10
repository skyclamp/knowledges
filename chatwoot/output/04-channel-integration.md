# Chatwoot 全渠道集成设计

## 1. 渠道架构总览

### 1.1 Inbox-Channel 多态模型

Chatwoot 的全渠道核心是 `Inbox` 与 `Channel::*` 的多态一对一关系：

- `Inbox` 侧：`belongs_to :channel, polymorphic: true, dependent: :destroy`
- `Channel` 侧（通过 `Channelable` concern）：`has_one :inbox, as: :channel`
- `inboxes` 表通过 `channel_type + channel_id` 绑定具体渠道实现。

这带来几个关键效果：

- 统一工作台语义：无论来源是 Telegram、WhatsApp、Email 还是 API，自上层都落到 `Conversation / Message / ContactInbox`。
- 渠道能力分层：
  - `Inbox` 管通用会话运营能力（分配、工作时间、满意度、事件）
  - `Channel::*` 管外部平台认证、Webhook、发送协议、媒体能力。
- 发送侧统一调度：`SendReplyJob` 依据 `inbox.channel.class` 路由到具体 `SendOn*Service`。

此外，`Inbox` 上存在渠道识别方法（如 `telegram? / whatsapp? / api?`），用于在控制器和服务层做能力分支。

### 1.2 支持的渠道列表与状态

| 渠道 | 模型类型 | 接入方式 | 消息能力概要 |
|---|---|---|---|
| Web Widget | `Channel::WebWidget` | 前端 SDK + 网站 Token | 文本、附件、预聊天表单、Widget 功能旗标 |
| Facebook | `Channel::FacebookPage` | Meta 授权 + Page 订阅 | Messenger 入站/出站，自动订阅/退订 |
| Instagram | `Channel::Instagram` | Instagram OAuth + webhook | DM 入站/出站，token 刷新 |
| Twitter | `Channel::TwitterProfile` | Twitter token | 私信（及可选 tweets 标志） |
| Telegram | `Channel::Telegram` | Bot Token + Telegram Webhook | 文本、媒体、位置、联系人、回调按钮（重点） |
| WhatsApp | `Channel::Whatsapp` | 360dialog / WhatsApp Cloud（Meta） | 文本、媒体、交互消息、模板消息、状态回执（重点） |
| Twilio SMS/WhatsApp | `Channel::TwilioSms` | Twilio SID/Token/服务号 | SMS 或 WhatsApp（`medium` 区分） |
| SMS（Bandwidth） | `Channel::Sms` | Bandwidth API 凭据 | 文本、附件 |
| LINE | `Channel::Line` | LINE channel id/secret/token | 通过 Line SDK 客户端通信 |
| Email | `Channel::Email` | SMTP/IMAP 或 OAuth Provider | 邮件收发、转发地址 |
| API Channel | `Channel::Api` | Public API + 可选回调 Webhook | 自定义渠道入站、事件外发（重点） |
| TikTok | `Channel::Tiktok` | TikTok OAuth token | DM 处理，token 刷新 |

说明：

- “状态”在代码层主要体现为：鉴权状态、Webhook 是否可用、账号是否 active、是否需要 reauthorization。
- WhatsApp 还存在 `reauthorization_required?` 的治理路径（来自 `Reauthorizable` 流程）。

### 1.3 渠道消息流通用流程

入站（Inbound）统一模式：

1. 外部平台向 `webhooks/*` 回调。
2. `Webhooks::*Controller` 只做轻逻辑（验证、过滤）并异步投递 `Webhooks::*EventsJob`。
3. Job 根据渠道配置查找 `Channel`，再调用 `IncomingMessageService`。
4. Service 完成联系人映射（`ContactInboxWithContactBuilder`）、会话定位/创建、消息/附件落库。

出站（Outbound）统一模式：

1. 坐席在会话中回复，消息创建后由 `SendReplyJob` 处理。
2. Job 按 `channel_type` 派发 `SendOn*Service`。
3. 具体渠道 Provider 调用外部 API，成功后回填 `message.source_id`，失败写入 `external_error` 并标记失败。

补充：

- 全局事件监听器 `WebhookListener` 会把消息/会话事件投递到账号级 webhook。
- 若 Inbox 为 API Channel，且配置了 `channel.webhook_url`，还会额外触发 API Inbox 级回调 webhook（实现“出站回调”）。

---

## 2. Telegram 渠道详细设计

### 2.1 接入配置（Bot Token）

`Channel::Telegram` 关键字段：

- `bot_token`（唯一）
- `bot_name`

创建时逻辑：

- `before_validation` 调用 Telegram `getMe` 校验 token 合法性。
- 校验通过后写入 `bot_name`。

安全性：

- 启用加密配置时，对 `bot_token` 做加密存储（deterministic）。

### 2.2 Webhook 注册机制

`Channel::Telegram` 在 `before_save` 中执行 webhook 设置：

1. 先 `deleteWebhook`
2. 再 `setWebhook` 到
   `FRONTEND_URL/webhooks/telegram/:bot_token`

路由入口：

- `POST /webhooks/telegram/:bot_token`
- 控制器仅入队：`Webhooks::TelegramEventsJob`

### 2.3 入站消息处理流程（收到用户消息）

主链路：

1. `Webhooks::TelegramController#process_payload` 入队。
2. `Webhooks::TelegramEventsJob`：
   - 校验 `bot_token` 能找到 channel
   - 校验账号 active
   - `edited_message` 走 `Telegram::UpdateMessageService`
   - 其余走 `Telegram::IncomingMessageService`
3. `IncomingMessageService`：
   - 仅处理私聊（群聊过滤）
   - 兼容 business message payload 结构
   - 建立/复用 `ContactInbox` 与 `Conversation`
   - 创建 `Message`（包含 reply 关联的 `in_reply_to_external_id`）
   - 处理附件：图片/视频/音频/文件、位置、联系人

会话策略：

- 若 Inbox 开启 `lock_to_single_conversation`，复用最后会话；
- 否则复用最后未 resolved 会话，找不到则新建。

### 2.4 出站消息处理流程（Agent 回复）

1. `SendReplyJob` 路由到 `Telegram::SendOnTelegramService`。
2. `Channel::Telegram#send_message_on_telegram`：
   - 文本走 `sendMessage`
   - 附件走 `Telegram::SendAttachmentsService`
3. 成功后回填 `message.source_id`。
4. 失败时记录 `external_error` 并置 `status = failed`。

附件发送策略：

- 图片/视频可合并 `sendMediaGroup`
- 音频可分组
- 文档通常单独走 multipart `sendDocument`
- 任一请求失败会中断后续发送

### 2.5 支持的消息类型

入站支持：

- 文本、caption
- 图片、视频、音频、语音、文档、贴纸缩略图
- 位置（含 venue）
- 联系人卡片
- 回调按钮（callback_query）
- 编辑消息（edited_message / edited_business_message）

出站支持：

- 文本（HTML parse mode）
- input_select（inline keyboard）
- 多媒体与文档附件

### 2.6 限制与注意事项

- 当前仅支持 private chat，不支持群组会话。
- business 消息场景下存在“已读同步”待完善注释（服务内明确标注 TODO）。
- webhook URL 将 bot_token 暴露在路径中（依赖 token 随机性与 HTTPS，运维上要注意日志脱敏）。
- 文件下载依赖 Telegram 文件 URL，失败时仅记录日志并跳过附件。

---

## 3. WhatsApp 渠道详细设计

### 3.1 接入方式（Cloud API vs BSP vs Twilio）

Chatwoot 现有三条 WhatsApp 路径：

1. `Channel::Whatsapp` + `provider=whatsapp_cloud`
   - 直连 Meta WhatsApp Cloud API
2. `Channel::Whatsapp` + `provider=default`
   - 360dialog BSP 路径
3. `Channel::TwilioSms` + `medium=whatsapp`
   - Twilio WhatsApp 通道

其中本次重点在 `Channel::Whatsapp`。

### 3.2 Webhook 注册与验证

Cloud 路径：

- model create 后（满足条件）自动 `setup_webhooks`
- embedded signup 场景由 `EmbeddedSignupService` 显式调用 setup
- `WebhookSetupService`：
  - 必要时先注册手机号（register）
  - 再订阅 WABA webhook 并覆盖 callback URL
  - 订阅字段含 `messages` 与 `smb_message_echoes`

验证入口：

- `GET /webhooks/whatsapp/:phone_number` 走 Meta challenge
- `MetaTokenVerifyConcern` 调 `valid_token?`
- token 来自 channel `provider_config['webhook_verify_token']`

入站入口：

- `POST /webhooks/whatsapp/:phone_number`
- 控制器会先过滤 `INACTIVE_WHATSAPP_NUMBERS`
- 再入队 `Webhooks::WhatsappEventsJob`

### 3.3 入站消息处理流程

1. `Webhooks::WhatsappEventsJob` 先从 payload 或 URL phone_number 找 channel。
2. 校验：channel 存在、账号 active、未 reauthorization_required。
3. 若为 `smb_message_echoes`，按 outgoing echo 流程处理。
4. 普通消息按 provider 分发：
   - cloud -> `IncomingMessageWhatsappCloudService`
   - default -> `IncomingMessageService`
5. `IncomingMessageBaseService` 执行：
   - 状态回执更新（delivered/read/failed 等）
   - 消息类型过滤（reaction/ephemeral/unsupported）
   - Redis 去重锁（source_id）防重入
   - 联系人/会话定位与消息创建
   - 附件与位置、联系人卡片落库

### 3.4 出站消息处理流程

1. `SendReplyJob` -> `Whatsapp::SendOnWhatsappService`。
2. 根据回复窗口与模板参数：
   - 会话窗口内：发送 session message
   - 窗口外或显式模板：发送 template message
3. Provider (`WhatsappCloudService` / `Whatsapp360DialogService`) 调外部 API。
4. 成功回填 `source_id`，失败更新 `external_error` + `failed`。

补充机制：

- 支持 input_select，按条目数量自动选择 button 或 list interactive payload。
- 支持 reply context（引用外部 message_id）。

### 3.5 消息模板（HSM）支持

模板主链路：

- `TemplateProcessorService` 负责按模板定义和参数生成组件结构
- `TemplateParameterConverterService` 将 legacy 参数格式转换为 enhanced 组件格式
- `PopulateTemplateParametersService` 构建 body/header/footer/button 参数，支持媒体 header、URL/coupon code 按钮等

模板同步：

- `Channels::Whatsapp::TemplatesSyncSchedulerJob` 定时挑选需要同步的 channel
- `TemplatesSyncJob` 调 `channel.sync_templates`

CSAT 模板：

- Cloud 路径有专门 `CsatTemplateService` 创建/删除/查询状态。

### 3.6 支持的消息类型

入站：

- text
- image/audio/video/document/sticker/voice
- interactive(button_reply/list_reply)
- button
- location
- contacts
- statuses（回执）
- smb_message_echoes（共存模式业务端回声消息）

出站：

- text
- media/document
- interactive（button/list）
- template（含增强组件参数）

### 3.7 限制与注意事项

- Cloud 与 360dialog API 版本、字段格式有差异，Provider 分层是必要的。
- webhook 查找 channel 时会校验 `phone_number_id` 以防多号码场景串号。
- 存在 reauthorization 状态；该状态下 webhook job 会直接丢弃事件。
- echo 消息（来自 WhatsApp Business App）会入库为 outgoing，状态置 delivered，避免被再次发送。
- 电话号格式存在国家差异，内置巴西/阿根廷归一化逻辑用于联系人匹配。

---

## 4. API Channel（自定义渠道）

### 4.1 设计理念

`Channel::Api` 的定位是“协议无关接入层”：

- 让外部自研 IM / App / Bot 可以通过 Chatwoot Public API 写入会话。
- Chatwoot 自身只关心标准对象（Contact/Conversation/Message），不限定上游协议。

核心字段：

- `identifier`（公共 API 访问标识）
- `hmac_token` + `hmac_mandatory`
- `webhook_url`（用于 Chatwoot -> 外部系统回调）

### 4.2 入站消息 API

Public API 路由：

- `POST /public/api/v1/inboxes/:inbox_id/contacts`
- `POST /public/api/v1/inboxes/:inbox_id/contacts/:contact_id/conversations`
- `POST /public/api/v1/inboxes/:inbox_id/contacts/:contact_id/conversations/:conversation_id/messages`

流程要点：

- Inbox 通过 `Channel::Api.identifier` 查找。
- Contact 创建用 `ContactInboxWithContactBuilder`。
- 可选 HMAC 校验：`identifier_hash = HMAC_SHA256(hmac_token, identifier)`。
- API Inbox 允许创建 incoming message（`Messages::MessageBuilder` 明确限制：非 API Inbox 不允许 incoming）。

### 4.3 出站消息回调

当会话/消息事件产生时：

- `WebhookListener` 会统一分发账号 webhook。
- 若 `inbox.channel_type == Channel::Api` 且 `webhook_url` 非空，则额外触发 `WebhookJob` 回调该 URL。

因此 API Channel 天然具备：

- 入站 API 接收
- 出站事件推送

可以组成双向桥接。

### 4.4 使用场景（可以用来接入自定义 IM）

典型场景：

- 企业自研 App IM 接入客服工作台
- 游戏内聊天系统接入工单/客服
- IoT 设备消息转客服
- 第三方渠道聚合器与 Chatwoot 解耦集成

建议实践：

- 开启 `hmac_mandatory`
- 对 `webhook_url` 做签名校验与重试幂等
- 以 `source_id / echo_id` 做外部侧去重

---

## 5. 添加新渠道的扩展模式

如果要新增一个渠道，按 Chatwoot 现有设计至少实现以下模块：

1. 渠道模型
- 新建 `app/models/channel/<new_channel>.rb`
- `include Channelable`
- 定义 `EDITABLE_ATTRS`、凭据校验、必要的 webhook setup/teardown

2. Inbox 创建入口接入
- 在 `Api::V1::Accounts::InboxesController` 中加入：
  - `allowed_channel_types`
  - `channel_type_from_params`
- 保证 `channel.create!` 所需参数可透传

3. Webhook 接收层
- 新建 `app/controllers/webhooks/<new_channel>_controller.rb`
- 仅做验证/过滤后，入队 `Webhooks::<NewChannel>EventsJob`
- 在 `config/routes.rb` 添加 webhook 路由

4. 事件 Job 与入站 Service
- `app/jobs/webhooks/<new_channel>_events_job.rb`
- `app/services/<new_channel>/incoming_message_service.rb`
- 处理：联系人映射、会话复用策略、消息与附件落库、状态回执

5. 出站发送 Service
- 新建 `app/services/<new_channel>/send_on_<new_channel>_service.rb`
- 接入 `SendReplyJob::CHANNEL_SERVICES` 映射
- 处理发送成功回填 `source_id` 与失败回填 `external_error`

6. ContactInbox source_id 规则
- 若渠道需要特定 source_id（如手机号格式），在 `ContactInboxBuilder` 扩展生成逻辑
- 同时考虑号码归一化与重复联系人问题

7. 模板/能力扩展（可选）
- 若渠道有模板消息、交互按钮、状态回执、健康检查，需要独立 Provider/Service 子层

8. 运维与治理
- 处理账号 inactive、channel reauthorization_required、去重锁、webhook 验签
- 必要时加入定时任务（如模板同步、token 刷新）

总体模式可以概括为：

- `Controller (thin) -> Job -> Incoming/Outgoing Service -> Model`
- `Inbox 统一业务语义 + Channel 封装平台差异`
