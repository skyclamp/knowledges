# Chatwoot 后端架构

## 1. 总体架构

Chatwoot 后端是一个标准 Rails 7 单体应用，但在架构上做了明显的“业务边界分层 + 扩展注入”设计：

- 表现层：Controller 按 API 边界拆分（`api/v1/accounts`、`api/v2`、`platform`、`public`、`webhooks`、`super_admin`）。
- 领域层：Model + Concern 承载核心业务状态、回调、事件发布。
- 应用服务层：`app/services` 提供跨模型流程编排；Builder 负责创建对象/消息构造。
- 异步层：ActiveJob + Sidekiq，队列分级明确（critical/high/medium/default/...）。
- 事件层：`Dispatcher`（sync + async）+ Listener 形成事件驱动总线。
- 实时层：ActionCable + Redis，事件扇出给 Agent/Contact 客户端。
- 扩展层：通过 `include_mod_with / prepend_mod_with` 将 Enterprise 模块注入 OSS 类。

### 1.1 启动与配置关键点

- `config/application.rb`
  - 启用 Rails 7 默认配置，额外 eager load `lib`、`enterprise/lib`、`enterprise/app/**`。
  - 将 `enterprise/app/views` 放到视图路径前面，支持同名覆盖。
  - 在启动时加载 `enterprise/config/initializers`。
  - 根据环境变量动态加载 APM（Datadog/Elastic/Scout/NewRelic/Sentry/Judoscale）。
  - 配置 ActiveRecord Encryption（MFA/2FA 依赖）。
- `config/environments/production.rb`
  - `active_job.queue_adapter = :sidekiq`。
  - 静态资源、日志、Asset Host、SSL、ActiveStorage、ActionMailbox 入站均由 ENV 驱动。
- `config/puma.rb`
  - 线程池和多进程均由环境变量控制；支持 `preload_app!`。
- `config/database.yml`
  - PostgreSQL，连接池根据 Sidekiq/线程动态切换。
  - 设置 `statement_timeout` 控制长 SQL。
- `config/sidekiq.yml`
  - 多级队列：`critical > high > medium > default > ...`，并发可配。
- `config/cable.yml`
  - ActionCable 使用 Redis；`channel_prefix` 按环境区分。

### 1.2 请求处理主流程

典型 API 请求链路（以账号内接口为例）：

1. Router 命中命名空间路由。
2. 进入 `Api::BaseController`：
   - 若请求头包含 `api_access_token`，走 AccessToken 鉴权。
   - 否则走 Devise 会话/Token 鉴权（`authenticate_user!`）。
3. 进入 `Api::V1::Accounts::BaseController`：解析 `account_id`，校验账号可访问并设置 `Current.account`。
4. 控制器动作中调用 Builder/Service。
5. Model 持久化后通过回调触发事件分发。
6. 同步监听器触发 ActionCable 广播；异步监听器进入 Sidekiq 执行自动化、webhook、通知、报表聚合等。

---

## 2. API 设计

### 2.1 API 版本与分层

Chatwoot 的 API 分层非常清晰：

- v1 Account API（`/api/v1/accounts/:account_id/...`）
  - 主业务 API，覆盖会话、消息、联系人、收件箱、自动化、团队、标签、webhook、集成等。
- v2 API（`/api/v2/...`）
  - 重点在报表与分析能力（reports/summary/live reports），以及部分新流程（如 v2 注册入口）。
- Platform API（`/platform/api/v1/...`）
  - 跨账户、第三方平台应用场景，基于 Platform App 的资源许可模型。
- Public API（`/public/api/v1/...`）
  - 面向外部联系人的公开接口（API inbox contact/conversation/message、CSAT、Help Center）。
- Widget API（`/api/v1/widget/...`）
  - 网站嵌入组件专用，使用网站 token + 会话 token。
- Webhooks 接收器（`/webhooks/*`）
  - 各渠道入站事件入口（WhatsApp/Instagram/Line/Telegram/SMS/Twitter/TikTok/Shopify 等）。

### 2.2 认证机制

Chatwoot 在不同 API 边界使用不同认证方式：

- 用户鉴权（Dashboard/API）
  - Devise + devise_token_auth。
  - `ApplicationController` 集成 `DeviseTokenAuth::SetUserByToken`。
- AccessToken 鉴权（机器人/集成/API 客户端）
  - `Api::BaseController` 通过 `AccessToken`（多态 owner：User/AgentBot/PlatformApp）识别请求主体。
  - `AccessTokenAuthHelper` 对 AgentBot 可访问接口做白名单控制。
- 账号上下文隔离
  - `EnsureCurrentAccountHelper` 校验当前用户/机器人是否属于目标 account。
- Platform API 鉴权
  - `PlatformController` 强制 access token，并校验 `platform_app_permissibles`。
- Widget/Public 鉴权
  - Widget：`WebsiteTokenHelper` 解码 `X-Auth-Token`，定位 contact_inbox。
  - Public API：按 `Channel::Api` 标识符与 contact source_id 访问。
- Webhook 验证
  - Meta 系列通过 `MetaTokenVerifyConcern` 验证 challenge token。

### 2.3 路由结构

`config/routes.rb` 是系统边界定义核心。主干可概括为：

```text
/
├─ auth/* (devise_token_auth)
├─ /api
├─ /api/v1
│  ├─ /accounts
│  │  └─ /:account_id
│  │     ├─ conversations
│  │     │  └─ /:conversation_id/messages
│  │     ├─ contacts
│  │     ├─ inboxes
│  │     ├─ search
│  │     ├─ automation_rules / macros / sla_policies
│  │     ├─ webhooks / integrations/*
│  │     └─ captain/* (AI assistant/coplilot 相关)
│  ├─ /widget/*
│  ├─ /profile
│  └─ /integrations/webhooks
├─ /api/v2
│  └─ /accounts/:account_id
│     ├─ reports
│     ├─ summary_reports
│     ├─ live_reports
│     └─ year_in_review
├─ /platform/api/v1
│  ├─ users
│  ├─ accounts
│  └─ agent_bots
├─ /public/api/v1
│  ├─ inboxes/:inbox_id/contacts/:contact_id/conversations/:conversation_id/messages
│  └─ csat_survey
├─ /webhooks/* (channel inbound)
├─ /super_admin/*
└─ /enterprise/* (仅 Enterprise)
```

额外观察：

- Enterprise 条件路由大量使用 `if ChatwootApp.enterprise?`。
- 帮助中心（`/hc/:slug/...`）在公共路由层直接暴露。
- Super Admin 与 Sidekiq Web UI 按身份隔离。

### 2.4 API 设计模式

- RESTful 为主，辅以动作型 member/collection endpoint。
  - 例如 conversation 的 `toggle_status`、`toggle_priority`、`mute/unmute`。
- 分页统一：大量接口使用 `page(...).per(...)`。
- 过滤统一：`FilterService` + 各实体子类（`Conversations::FilterService`、`Contacts::FilterService`）。
- 搜索统一：`SearchController -> SearchService`，可切换 SQL LIKE / GIN / OpenSearch(Searchkick)。
- 错误处理统一：`RequestExceptionHandler` 捕获并映射 404/401/422。
- 授权统一：Pundit `authorize / policy_scope`。

---

## 3. 数据层

### 3.1 数据库设计概要

#### 核心关系

- 租户核心：`accounts`
- 用户归属：`account_users(account_id, user_id, role)`
- 渠道入口：`inboxes(account_id, channel_type, channel_id)`
- 客户：`contacts(account_id, ...)`
- 客户渠道映射：`contact_inboxes(contact_id, inbox_id, source_id)`
- 会话：`conversations(account_id, inbox_id, contact_id, assignee_id, status, display_id, ...)`
- 消息：`messages(account_id, inbox_id, conversation_id, sender_type/sender_id, message_type, content_type, ...)`
- 令牌：`access_tokens(owner_type, owner_id, token)`
- 报表事件：`reporting_events(account_id, inbox_id, conversation_id, user_id, name, value, ...)`
- 外发 webhook：`webhooks(account_id, inbox_id, url, subscriptions)`

#### 多租户隔离策略

- 几乎所有业务表都有 `account_id`。
- 关键唯一性在租户内保证：
  - `conversations(account_id, display_id)` 唯一。
  - `account_users(account_id, user_id)` 唯一。
  - `contacts` 的 email/identifier 结合 account 唯一。
- 控制器层会再次验证 account 访问权限，形成“数据层 + 应用层”双保险。

#### 检索与性能

- 大量组合索引围绕 `account_id + 状态/时间/关联键`。
- 消息内容、联系人名称等启用 GIN/trigram 索引。
- reporting_events 有汇总表 `reporting_events_rollups`。

### 3.2 关键 Model 设计模式

- Concern 组合行为
  - `Conversation` 混入 `AssignmentHandler`、`AutoAssignmentHandler`、`ActivityMessageHandler`、`PushDataHelper`。
  - `Message` 混入 `MessageFilterHelpers`、`Liquidable`。
- 回调驱动业务状态
  - `Message after_create_commit` 触发：会话重开、事件派发、异步发送、模板 hook、联系人活跃度更新。
  - `Conversation after_update_commit` 触发：状态事件、分配事件、活动消息。
- 枚举状态机（轻量）
  - `Conversation.status`: open/resolved/pending/snoozed
  - `Message.message_type/content_type/status`
- 数据完整性与约束
  - JSON schema 验证（settings/template_params）。
  - 防泛洪校验（单位时间消息上限）。

---

## 4. 服务层

### 4.1 服务组织方式

`app/services` 按业务域分目录，呈现“领域服务 + 渠道适配器 + 基础服务”三层：

- 领域服务
  - `conversations/*`, `messages/*`, `contacts/*`, `auto_assignment/*`, `automation_rules/*`
- 基础能力
  - `filter_service.rb`, `search_service.rb`, `notification/*`
- 渠道适配
  - `twilio/*`, `whatsapp/*`, `instagram/*`, `telegram/*`, `line/*`, `sms/*`, `email/*`, `facebook/*`

Builder 与 Service 分工清楚：

- Builder 负责“构造/创建聚合”：`ConversationBuilder`, `Messages::MessageBuilder`
- Service 负责“流程决策/查询/副作用”：分配、过滤、搜索、通知、发送

### 4.2 典型服务调用链

以“Agent 在会话内发送消息”为例：

1. Controller
   - `Api::V1::Accounts::Conversations::MessagesController#create`
2. Builder
   - `Messages::MessageBuilder.new(user, conversation, params).perform`
   - 负责 content_attributes、附件、Email 扩展字段、sender 决策
3. Model 持久化
   - `Message` 保存成功后进入 `after_create_commit`
4. 事件派发
   - `dispatch_create_events` 发布 `MESSAGE_CREATED`（必要时 `FIRST_REPLY_CREATED`）
5. 实时推送
   - `ActionCableListener` 消费事件，触发 `ActionCableBroadcastJob`
6. 渠道下发
   - `send_reply` 入队 `SendReplyJob`
   - `SendReplyJob` 根据 channel_type 选择 `*::SendOn*Service`
7. 异步联动
   - async listeners 触发 webhook、自动化规则、通知、报表事件等

这个链路体现了 Chatwoot 的核心思想：

- Controller 薄，Builder/Service 承载编排，Model 回调只负责状态变化与事件发布。
- “同一事件”可同时驱动实时 UI、一致性任务和外部系统通知。

---

## 5. 异步任务与事件系统

### 5.1 Sidekiq 任务体系

Job 类型大致分为：

- 核心业务任务
  - `SendReplyJob`, `WebhookJob`, `HookJob`, `EventDispatcherJob`
- 自动分配与会话维护
  - `AutoAssignment::AssignmentJob`, `Inboxes::BulkAutoAssignmentJob`, 重新打开/关闭类任务
- 渠道 webhook 入站处理
  - `Webhooks::*EventsJob`（instagram/whatsapp/line/telegram/sms...）
- 通知与后台维护
  - notification jobs、migration jobs、internal housekeeping jobs

队列优先级与职责匹配：

- `critical`：事件广播等低延迟链路
- `high/medium/default`：消息发送、webhook、常规业务
- `housekeeping/deferred/scheduled`：低优先级维护任务

### 5.2 事件监听与分发机制

事件总线实现：

- `Dispatcher` = `SyncDispatcher + AsyncDispatcher`
- Sync listener（即时）
  - `ActionCableListener`, `AgentBotListener`
- Async listener（延后）
  - `AutomationRuleListener`, `WebhookListener`, `HookListener`, `NotificationListener`, `ReportingEventListener` 等

关键点：

- 业务模型不直接依赖具体消费者，只发布领域事件。
- `AsyncDispatcher` 通过 `EventDispatcherJob` 异步化，避免阻塞主请求。
- Listener 分工清晰，便于扩展新事件消费者。

### 5.3 WebSocket/ActionCable 实时通信

- ActionCable Redis 适配，支持特殊 Redis 环境（禁用 client 命令场景）。
- `ActionCableListener` 根据事件类型计算广播目标 token：
  - Agent/Admin token（用户维度）
  - Contact token（contact_inbox 维度）
- `ActionCableBroadcastJob` 对会话更新类事件会回源读取最新数据，减少乱序导致的前端状态回退。

---

## 6. Enterprise 扩展机制

Chatwoot 的 Enterprise 采用“模块注入 + 同名覆盖”双机制，而非硬分叉：

- 注入机制
  - 通过 initializer `01_inject_enterprise_edition_module.rb` 给 Module 注入：
    - `prepend_mod_with`
    - `include_mod_with`
    - `extend_mod_with`
  - 业务类中常见：
    - `Conversation.prepend_mod_with('Conversation')`
    - `Api::V1::Accounts::ConversationsController.prepend_mod_with(...)`
- 加载机制
  - `ChatwootApp.extensions` 决定扩展集合（enterprise/custom）。
  - `config/application.rb` eager load enterprise 目录并加载 enterprise initializers。
- 路由机制
  - `routes.rb` 中大量 `if ChatwootApp.enterprise?` 条件路由。

典型扩展示例：

- `Enterprise::Api::V1::Accounts::ConversationsController`
  - 增加 `reporting_events`、`inbox_assistant`，并扩展 permitted params（如 `sla_policy_id`）。
- `Enterprise::AutoAssignment::AssignmentService`
  - 在 OSS 自动分配上增加 capacity policy、balanced selector、排除规则。
- `Enterprise::SearchService`
  - 接管 advanced search，走 Searchkick/OpenSearch 并附加筛选条件。

结论：Enterprise 不是独立后端，而是对 OSS 核心类进行可控覆写与增强。

---

## 7. 关键技术决策

### 7.1 多租户方案

- 采用共享库（shared DB）+ `account_id` 逻辑隔离。
- 在 Controller 层强制校验租户归属，模型和索引层做租户内唯一性约束。
- 优点：实现简单、查询灵活、运维成本低。
- 代价：必须严格遵守 account scope 约束，防止越权查询。

### 7.2 搜索方案

- 分层回退策略：
  - 基础：ILIKE
  - 增强：Postgres GIN/tsquery（`search_with_gin`）
  - 高级：Searchkick/OpenSearch（Enterprise + `OPENSEARCH_URL`）
- 好处：在无外部搜索服务时可用，有服务时可平滑升级。

### 7.3 文件与媒体存储

- ActiveStorage 统一附件抽象。
- 生产环境存储服务由 `ACTIVE_STORAGE_SERVICE` 决定（本地/S3 等）。
- 与消息发送链路结合：附件消息发送采用延迟入队，避免事务后文件尚未可读。

### 7.4 认证与安全

- Devise + TokenAuth 处理用户认证。
- AccessToken 支持机器与平台应用。
- Rack::Attack 做多层限流（登录、MFA、报表、搜索、widget 滥用等）。
- CORS 按场景可控开放（API-only/server 配置）。

### 7.5 实时与异步解耦

- 通过事件总线将“写请求成功”与“副作用执行”解耦。
- 同步事件保证前端即时反馈，异步事件保障吞吐与稳定性。
- Sidekiq 队列优先级和任务分类使系统在高并发下具备可调度性。

---

## 总结

Chatwoot 后端的核心架构特征可以概括为：

- 以 account 为边界的多租户 Rails 单体。
- 以 Controller 分层 + Service/Builder 编排 + Model 事件回调为主线。
- 以 Dispatcher/Listener 为中枢的事件驱动体系，将实时推送、自动化、webhook、统计任务统一起来。
- 通过可组合的 Enterprise 模块注入机制，在保持 OSS 主干稳定的前提下扩展高级能力。