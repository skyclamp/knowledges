# Libredesk 后端核心模块详细设计（internal 深度分析）

## 0. 范围与方法

本文聚焦 `internal/` 下 23 个核心模块，按统一维度分析：

1. 职责
2. 数据模型（`models/`）
3. 核心接口/方法（主文件导出方法）
4. SQL 查询模式（`queries.sql`）
5. 模块依赖关系

并重点深挖：

- 消息处理流水线（从 `cmd/messages.go`、`cmd/conversation.go` 入手）
- Channel 抽象层（`internal/inbox/channel/`）
- Automation 规则引擎
- AI Provider 抽象与调用

---

## 1. 核心业务模块

### 1.1 internal/conversation/

**职责**
- 会话与消息生命周期管理中心，负责会话列表、消息收发、草稿、状态、优先级、标签、SLA 触发、自动化触发、WebSocket/Webhook 广播。

**数据模型（models）**
- `ConversationListItem`：会话列表投影，包含状态、优先级、收件箱、联系人、未读数、SLA 关键时间等。
- `Conversation`：完整会话聚合，包含 `AssignedUserID`、`AssignedTeamID`、`SLAPolicyID`、`AppliedSLAID`、`Subject`、`LastMessage` 等。
- `Message`：消息实体，包含 `Type`（incoming/outgoing/activity）、`Status`（pending/sent/failed/received）、`SenderType`、`Content`、`TextContent`、`Meta`、`SourceID`、附件与 mentions。
- `IncomingMessage`：入站消息封装（解析后的 message + contact + inbox 上下文）。
- `ConversationDraft`、`MentionInput`、`ConversationParticipant`、`ConversationCounts` 等。

**核心接口/方法**
- `New(...) (*Manager, error)`：组装 conversation manager（依赖 inbox/user/team/media/sla/template/webhook/ws/notification/automation）。
- `Run(ctx, incomingWorkers, outgoingWorkers, scanInterval)`：启动入站/出站 worker 和 pending 消息扫描器。
- `GetConversation(...)` / `GetAllConversationsList(...)` / `GetAssigned...` / `GetUnassigned...` / `GetView...`。
- `QueueReply(...)`：创建出站消息（`pending`）并入库，等待发送 worker。
- `SendPrivateNote(...)` / `CreateContactMessage(...)` / `InsertMessage(...)`。
- `EnqueueIncoming(...)` + `IncomingMessageWorker()` + `processIncomingMessage(...)`：入站处理核心链路。
- `UpdateConversationStatus(...)` / `UpdateConversationPriority(...)` / `UpdateConversationUserAssignee(...)` / `SetConversationTags(...)`。
- `ApplyAction(action, conv, user)`：Automation 动作执行入口。

**SQL 查询模式（queries.sql）**
- 会话：`insert-conversation`、`get-conversations`、`get-conversation`、`update-conversation-*`、`re-open-conversation`、`remove-conversation-assignee`。
- 列表/过滤：单一 `get-conversations` 模板动态承载多列表视图（全部、分配、未分配、团队、被提及、自定义 view）。
- 消息：`get-messages`、`get-message`、`insert-message`、`update-message-status`、`get-outgoing-pending-messages`。
- 幂等/关联：`message-exists-by-source-id`、`get-message-source-ids`、`get-conversation-by-message-id`。
- 草稿：`upsert-conversation-draft`、`get-all-user-drafts`、`delete-*draft*`。
- mentions：`insert-mention`。

**依赖关系**
- 依赖：`inbox`、`user`、`team`、`media`、`sla`、`template`、`automation`、`webhook`、`notification`、`ws`。
- 被依赖：几乎所有对话相关 API handler（`cmd/conversation.go`、`cmd/messages.go`）。


### 1.2 internal/inbox/

**职责**
- 管理收件箱配置与实例生命周期，统一抽象渠道（当前为 email），并负责接收/发送消息能力接入。

**数据模型（models）**
- `Inbox`：`ID`、`Channel`、`Config`、`Name`、`From`、`CSATEnabled`。
- `Config`：`SMTP[]`、`IMAP[]`、`OAuth`、`AuthType`、`EnablePlusAddressing`。
- `SMTPConfig` / `IMAPConfig` / `OAuthConfig`：承载发送/接收/鉴权细节。

**核心接口/方法**
- `New(...)`、`InitInboxes(initFn)`、`Reload(ctx, initFn)`、`Start(ctx)`、`Close()`。
- `SetMessageStore(...)`、`SetUserStore(...)`：注入跨模块依赖。
- `Register(i Inbox)`、`Get(id)`：已初始化渠道实例注册与获取。
- `GetDBRecord` / `GetAll` / `Create` / `Update` / `Toggle` / `SoftDelete` / `UpdateConfig`。

**SQL 查询模式**
- `get-active-inboxes`、`get-all-inboxes`、`get-inbox`。
- `insert-inbox`、`update`、`toggle`、`soft-delete`、`update-config`。
- 设计特征：配置 JSON 入库，敏感字段按 manager 层加解密处理。

**依赖关系**
- 依赖：`crypto`（配置敏感字段加解密）、`conversation`（MessageStore 入站队列）、`user`（联系人查找）。
- 被依赖：`conversation` 的出站发送与消息接收入口。


### 1.3 internal/user/

**职责**
- 用户域管理（agent/contact），含密码认证、角色关联、团队关联、API Key、在线状态、联系人备注等。

**数据模型（models）**
- `User`：`Type`（agent/contact/system）、`Roles`、`Permissions`、`Teams`、`AvailabilityStatus`、`CustomAttributes`、`APIKey*`。
- `UserCompact`：列表场景轻量投影。
- `Note`：联系人备注（作者、内容、时间）。

**核心接口/方法**
- 认证相关：`VerifyPassword`、`SetResetPasswordToken`、`ResetPassword`。
- Agent：`GetAgent`、`GetAgentsCompact`、`CreateAgent`、`UpdateAgent`、`SoftDeleteAgent`。
- Contact：`CreateContact`、`UpdateContact`、`GetContact`、`GetContacts`。
- 状态：`UpdateAvailability`、`UpdateLastActive`、`MonitorAgentAvailability`。
- API Key：`GenerateAPIKey`、`ValidateAPIKey`、`RevokeAPIKey`。
- Notes：`GetNotes`、`CreateNote`、`DeleteNote`。

**SQL 查询模式**
- 用户查询与过滤：`get-user`、`get-users-compact`。
- 创建更新：`insert-agent`、`insert-contact`、`update-agent`、`update-contact`。
- 安全：`set-user-password`、`set-reset-password-token`、`get-user-by-api-key`、`set-api-key`、`revoke-api-key`。
- 活跃状态：`update-last-active-at`、`update-inactive-offline`。
- 备注：`get-notes`、`insert-note`、`delete-note`。

**依赖关系**
- 依赖：`role`、`team`（通过查询聚合角色与团队信息）。
- 被依赖：`auth`、`authz`、`conversation`、`notification` 等。


### 1.4 internal/envelope/

**职责**
- 统一 API 错误与分页响应封装。

**数据模型（结构）**
- `Error`：`Code`、`ErrorType`、`Message`、`Data`。
- `PageResults`：`Results`、`Total`、`PerPage`、`TotalPages`、`Page`。

**核心接口/方法**
- `NewError(etype, message, data)`：按错误类型映射 HTTP code。
- `NewErrorWithCode(...)`：允许指定 code。
- 错误类型常量：`GeneralError`、`PermissionError`、`InputError`、`DataError`、`NotFoundError` 等。

**SQL 查询模式**
- 无。

**依赖关系**
- 被所有模块依赖（统一错误语义与响应契约）。

---

## 2. 协作与组织模块

### 2.1 internal/team/

**职责**
- 团队实体与成员关系管理，提供会话分配策略参数（如自动分配上限）。

**数据模型**
- `Team`：`Name`、`Timezone`、`ConversationAssignmentType`、`MaxAutoAssignedConversations`、`BusinessHoursID`、`SLAPolicyID`。
- `TeamMember`：`ID`（user id）、`AvailabilityStatus`。
- `TeamCompact`：轻量展示。

**核心接口/方法**
- `GetAll`、`GetAllCompact`、`Get`。
- `Create`、`Update`、`Delete`。
- `GetUserTeams`、`UpsertUserTeams`、`UserBelongsToTeam`、`GetMembers`。

**SQL 查询模式**
- 读取：`get-teams`、`get-team-members`、`get-user-teams`。
- 写入：`insert-team`、`update-team`、`delete-team`。
- 关系：`upsert-user-teams`、`user-belongs-to-team`。

**依赖关系**
- 被 `conversation`、`autoassigner`、`authz`、`view` 等使用。


### 2.2 internal/role/

**职责**
- 管理角色与权限集合。

**数据模型**
- `Role`：`Name`、`Description`、`Permissions []string`。
- 默认角色常量：`admin`、`agent`。

**核心接口/方法**
- `GetAll`、`Get`、`Create`、`Update`、`Delete`。
- `ComparePermissions(old,new)`：权限差异比较。

**SQL 查询模式**
- `get-all`、`get-role`、`insert-role`、`update-role`、`delete-role`。

**依赖关系**
- 被 `user` 与管理端角色配置调用。


### 2.3 internal/authz/

**职责**
- 基于 Casbin 的授权判断与权限缓存。

**数据模型**
- 模型层主要是权限常量（`internal/authz/models/models.go`）。
- 运行时结构：`Enforcer`（`SyncedEnforcer` + `permsCache`）。

**核心接口/方法**
- `NewEnforcer`。
- `LoadPermissions(user)`：将用户权限拆为 `obj:act` 并加载到 casbin policy。
- `Enforce(user,obj,act)`。
- `EnforceConversationAccess(user, conversation)`：按 read/read_all/read_assigned/read_team_* 等进行会话级判定。
- `EnforceMediaAccess(user, model)`。

**SQL 查询模式**
- 无（授权决策在内存 enforcer + 用户权限数据）。

**依赖关系**
- 依赖：`user`（用户权限）、`conversation`（资源上下文）。
- 被依赖：`cmd/*` handler 权限中间流程。


### 2.4 internal/auth/

**职责**
- OIDC 登录、会话管理、CSRF cookie 管理。

**数据模型**
- `models.User`（session 载荷）。
- `Provider`、`OIDCclaim`、`Config`（在 `auth.go` 内部结构/配置使用）。

**核心接口/方法**
- `New(cfg, i18n, redis, logger)`。
- `LoginURL(providerID, state)`。
- `ExchangeOIDCToken(ctx, providerID, code)`。
- `SaveSession`、`ValidateSession`、`DestroySession`。
- `SetSessionValues`、`GetSessionValue`、`SetCSRFCookie`。

**SQL 查询模式**
- 无（session 在 Redis，OIDC 通过 provider 配置）。

**依赖关系**
- 依赖：`redis`、OIDC provider。
- 被依赖：登录相关 cmd handler。


### 2.5 internal/tag/

**职责**
- 会话标签定义管理。

**数据模型**
- `Tag`：`ID`、`Name`、`CreatedAt`、`UpdatedAt`。

**核心接口/方法**
- `GetAll`、`Create`、`Update`、`Delete`。

**SQL 查询模式**
- `get-all-tags`、`insert-tag`、`update-tag`、`delete-tag`。

**依赖关系**
- 被 `conversation` 标签动作与管理接口使用。

---

## 3. 智能化模块

### 3.1 internal/automation/

**职责**
- 自动化规则执行引擎：监听会话事件，评估条件组并执行动作。

**数据模型**
- 常量：动作类型、比较操作符、规则类型、事件类型、执行模式。
- `RuleRecord`：数据库记录（`Name`、`Type`、`Events`、`Enabled`、`Weight`、`ExecutionMode`、`Rules JSON`）。
- `Rule`：运行态规则（`Groups` + `Actions`）。
- `RuleGroup`：`LogicalOp` + 多个 `RuleDetail`。
- `RuleDetail`：`Field`、`FieldType`、`Operator`、`Value`、`CaseSensitiveMatch`。
- `RuleAction`：`Type`、`Value []string`。

**核心接口/方法**
- `New`、`SetConversationStore`、`ReloadRules`。
- `Run(ctx, workerCount)`：worker pool + 每小时 time trigger。
- CRUD：`GetAllRules`、`GetRule`、`CreateRule`、`UpdateRule`、`DeleteRule`、`ToggleRule`、`UpdateRuleWeights`、`UpdateRuleExecutionMode`。
- 触发入口：`EvaluateNewConversationRules`、`EvaluateConversationUpdateRules`、`EvaluateConversationUpdateRulesByID`。
- 评估细节：`evalConversationRules`、`evaluateGroup`、`evaluateRule`、`evaluateFinalResult`。

**SQL 查询模式**
- 规则读取：`get-enabled-rules`、`get-all`、`get-rule`。
- 规则变更：`insert-rule`、`update-rule`、`delete-rule`、`toggle-rule`。
- 排序与策略：`update-rule-weight`、`update-rule-execution-mode`。

**依赖关系**
- 依赖：`conversation` store（读取会话 + 执行动作）。
- 被依赖：`conversation` 模块在新会话/消息进出/状态变化时触发。


### 3.2 internal/autoassigner/

**职责**
- 周期性自动分配未分配会话给团队成员（round-robin + 上限控制）。

**数据模型**
- 无独立 `models/`；核心是内存平衡器映射（每 team 一个 balancer）。

**核心接口/方法**
- `New(teamStore, conversationStore, systemUser, lo)`。
- `Run(ctx, autoAssignInterval)`。
- 内部流程：重建 balancer、取未分配会话、按团队策略分配、调用 conversation 更新。

**SQL 查询模式**
- 无独立 SQL，依赖 `team` 与 `conversation` 查询。

**依赖关系**
- 依赖：`team`、`conversation`、`user(system)`。


### 3.3 internal/ai/

**职责**
- 提供 LLM 能力接入层（当前默认 OpenAI），管理提示词模板与 provider 密钥。

**数据模型**
- `Provider`：`Name`、`Provider`、`Config`、`IsDefault`。
- `Prompt`：`Key`、`Title`、`Content`。
- 抽象：`ProviderClient` 接口 + `PromptPayload`。

**核心接口/方法**
- `New(opts)`。
- `Completion(key, prompt)`：读取系统提示词，取默认 provider client，调用 `SendPrompt`。
- `GetPrompts()`。
- `UpdateProvider(provider, apiKey)`（当前实现 OpenAI）。
- provider 内部：`getDefaultProviderClient()` + `OpenAIClient.SendPrompt()`。

**SQL 查询模式**
- `get-default-provider`、`get-prompt`、`get-prompts`、`set-openai-key`。

**依赖关系**
- 依赖：`crypto`（API key 加解密）、外部 OpenAI API。
- 被依赖：AI 相关 handler（文案建议、自动回复等场景）。


### 3.4 internal/sla/

**职责**
- SLA 策略管理与事件追踪（首响、解决、下一次响应），计算 deadline，触发提醒通知与 breach 标记。

**数据模型**
- `SLAPolicy`：各指标时长与通知配置。
- `SlaNotification`、`ScheduledSLANotification`。
- `AppliedSLA`：会话级应用记录。
- `SLAEvent`：指标事件（deadline/met/breached）。

**核心接口/方法**
- 策略 CRUD：`Get`、`GetAll`、`Create`、`Update`、`Delete`。
- 执行：`ApplySLA`、`CreateNextResponseSLAEvent`、`SetLatestSLAEventMetAt`。
- 计算：`GetDeadlines`、`CalculateDeadline`（考虑 business hours/timezone）。
- 调度：`Run(ctx, interval)`、`SendNotifications()`、`SendNotification(...)`。

**SQL 查询模式**
- 策略：`get-sla-policy`、`get-all-sla-policies`、`insert-sla-policy`、`update-sla-policy`、`delete-sla-policy`。
- 会话应用：`apply-sla`、`get-applied-sla`、`update-applied-sla-*`。
- 事件：`insert-next-response-sla-event`、`set-latest-sla-event-met-at`、`get-pending-sla-events`、`update-sla-event-as-*`。
- 通知：`insert-scheduled-sla-notification`、`get-scheduled-sla-notifications`、`update-notification-processed`。

**依赖关系**
- 依赖：`team`、`business_hours`、`template`、`notification`、`user`。
- 被依赖：`conversation`（入站/出站消息驱动 SLA 指标推进）。

---

## 4. 集成模块

### 4.1 internal/webhook/

**职责**
- 管理 webhook 配置，并异步投递事件（签名、安全校验、开关控制）。

**数据模型**
- `Webhook`：`Name`、`URL`、`Events[]`、`Secret`、`Enabled` 等。
- 事件常量：会话创建/变更、消息创建/更新等。

**核心接口/方法**
- `New`、`Run(ctx)`、`Close()`。
- `GetAll`、`Get`、`Create`、`Update`、`Delete`、`Toggle`、`SendTestWebhook`。
- `TriggerEvent(event, data)`：事件入队异步发送。

**SQL 查询模式**
- `get-all-webhooks`、`get-webhook`、`get-webhook-secret`。
- `get-active-webhooks`、`get-webhooks-by-event`。
- `insert-webhook`、`update-webhook`、`delete-webhook`、`toggle-webhook`。

**依赖关系**
- 被 `conversation`（消息/会话事件）广泛触发。
- 依赖 `crypto`（secret 处理）与 HTTP client。


### 4.2 internal/notification/

**职责**
- 通知中枢：站内通知持久化 + 实时推送 + 邮件等外部通知分发。

**数据模型**
- `UserNotification`：通知主体（类型、标题、正文、会话/消息/actor 关联、已读状态）。
- `NotificationStats`：未读/总量统计。

**核心接口/方法**
- 服务层：`NewService`、`Send(message)`、`Run(ctx)`、`Close()`。
- 分发层：`NewDispatcher`、`Send`、`SendWithEmails`。
- 用户通知管理：`GetAll`、`GetStats`、`Create`、`MarkAsRead`、`MarkAllAsRead`、`Delete`、`DeleteAll`、`DeleteOldNotifications`。

**SQL 查询模式**
- `get-notifications`、`get-notification-stats`。
- `insert-notification`、`mark-as-read`、`mark-all-as-read`。
- `delete-notification`、`delete-all-notifications`、`delete-old-notifications`。

**依赖关系**
- 被 `conversation`（@mention/分配）、`sla`（告警）调用。
- 与 `ws` 协同实现实时投递。


### 4.3 internal/ws/

**职责**
- 管理 WebSocket 客户端连接并按用户/广播投递实时事件。

**数据模型**
- `WSMessage`、`Message`、`BroadcastMessage`（type + data + users）。

**核心接口/方法**
- `NewHub(userStore)`、`AddClient`、`RemoveClient`、`BroadcastMessage`。

**SQL 查询模式**
- 无。

**依赖关系**
- 被 `conversation`、`notification` 调用进行实时更新推送。
- 依赖 `userStore.UpdateLastActive`（client 层心跳/活跃上报）。


### 4.4 internal/media/

**职责**
- 文件上传存储与元数据管理，支持关联模型、签名 URL、垃圾回收。

**数据模型**
- `Media`：`UUID`、`Name`、`ContentType`、`Size`、`Model`、`ModelID`、`Disposition`、`ContentID`、`Meta` 等。

**核心接口/方法**
- `New`。
- `Upload`、`UploadAndInsert`、`Insert`。
- `Get`、`GetURL`、`SignedURLValidator`、`GetBlob`。
- `Attach`、`GetByModel`、`Delete`、`DeleteUnlinkedMedia`。

**SQL 查询模式**
- `insert-media`、`get-media`、`get-media-by-uuid`、`delete-media`。
- 关联：`attach-to-model`、`get-model-media`。
- 清理/去重：`get-unlinked-message-media`、`content-id-exists`。

**依赖关系**
- 被 `conversation`（消息附件/内联图）、`user`（头像等）调用。

---

## 5. 辅助模块

### 5.1 internal/search/

**职责**
- 会话/消息/联系人检索。

**数据模型**
- `ConversationResult`、`MessageResult`、`ContactResult`。

**核心接口/方法**
- `Conversations(query)`、`Messages(query)`、`Contacts(query)`。

**SQL 查询模式**
- `search-conversations-by-reference-number`。
- `search-conversations-by-contact-email`。
- `search-messages`、`search-contacts`（ILIKE + LIMIT）。

**依赖关系**
- 被搜索 API handler 调用。


### 5.2 internal/template/

**职责**
- 模板管理与渲染（邮件模板、通知模板、页面模板）。

**数据模型**
- `Template`：`Name`、`Type`、`Subject`、`Body`、`IsDefault`、`IsBuiltIn`。

**核心接口/方法**
- 管理：`Create`、`Update`、`Get`、`GetAll`、`Delete`、`Reload`。
- 渲染：`RenderEmailWithTemplate`、`RenderStoredEmailTemplate`、`RenderInMemoryTemplate`、`RenderWebPage`。

**SQL 查询模式**
- `insert`、`update`、`get-default`、`get-all`、`get-template`、`delete`、`get-by-name`、`is-builtin`。

**依赖关系**
- 被 `conversation`（出站邮件渲染）、`sla`、`notification` 使用。


### 5.3 internal/csat/

**职责**
- CSAT 链接生成、评分反馈记录。

**数据模型**
- `CSATResponse`：`UUID`、`ConversationID`、`Score/Rating`、`Feedback`、`RespondedAt`。

**核心接口/方法**
- `Create(conversationID)`、`Get(uuid)`、`UpdateResponse(uuid, score, feedback)`、`MakePublicURL(...)`。

**SQL 查询模式**
- `insert`、`get`、`update`。

**依赖关系**
- 被 `conversation` 的 `ActionSendCSAT` 与会话关闭后满意度流程调用。


### 5.4 internal/macro/

**职责**
- 宏（快捷回复/动作模板）管理与使用统计。

**数据模型**
- `Macro`：`Name`、`MessageContent`、`Actions`、`Visibility`、`VisibleWhen`、`UserID/TeamID`、`UsageCount`。

**核心接口/方法**
- `Get`、`GetAll`、`Create`、`Update`、`Delete`、`IncrementUsageCount`。

**SQL 查询模式**
- `get`、`get-all`、`create`、`update`、`delete`、`increment-usage-count`。

**依赖关系**
- 被会话操作 UI/API 使用，动作最终落到 `conversation.ApplyAction`。


### 5.5 internal/view/

**职责**
- 自定义视图（过滤器）管理，支持个人/团队/全局可见性。

**数据模型**
- `View`：`Name`、`Filters`、`Visibility`、`UserID`、`TeamID`。

**核心接口/方法**
- `Get`、`GetUsersViews`、`GetSharedViewsForUser`、`GetAllSharedViews`。
- `Create`、`CreateSharedView`、`Update`、`UpdateSharedView`、`Delete`。

**SQL 查询模式**
- `get-view`、`get-user-views`、`get-shared-views-for-user`、`get-all-shared-views`。
- `insert-view`、`update-view`、`delete-view`。

**依赖关系**
- 与 `conversation` 列表过滤能力组合使用。


### 5.6 internal/setting/

**职责**
- 系统设置（通用 + 邮件通知等）读写管理，并对敏感字段进行加密。

**数据模型**
- `General`：站点、语言、上传限制、Logo/Favicon、RootURL、Timezone 等。
- `EmailNotification`：SMTP/认证/启用状态等。
- `Settings`：聚合配置结构。

**核心接口/方法**
- `GetAll`、`GetAllJSON`、`GetByPrefix`、`Get`、`GetAppRootURL`。
- `Update(s any)`：结构化更新并加密敏感字段。

**SQL 查询模式**
- `get-all`、`get-by-prefix`、`get`、`update`（键值配置聚合更新）。

**依赖关系**
- 被多个模块读取（如 `conversation` 生成 CSAT 公网 URL、通知发件配置等）。

---

## 6. 重点深挖

## 6.1 消息处理流水线（入站/出站）

### 6.1.1 入口层（cmd）

- `cmd/messages.go` 中 `handleSendMessage` 是发送入口。
- `cmd/conversation.go` 负责会话查询、分配、状态优先级变更等，与消息侧联动。

`handleSendMessage` 分三条分支：

1. Contact 发消息：`CreateContactMessage(...)`
2. Agent 发私有备注：`SendPrivateNote(...)`
3. Agent 正常回复：`QueueReply(...)`（核心异步发送链路）

### 6.1.2 入站消息路径（Receive -> Process -> Store -> Trigger）

1. 渠道接收
- `internal/inbox/channel/email/` 中 `Email.Receive(ctx)` 启动 IMAP 读取。
- 解析后构造 `models.IncomingMessage`，调用 `messageStore.EnqueueIncoming(...)`。

2. 队列消费
- `conversation.Manager.IncomingMessageWorker()` 从 `incomingMessageQueue` 消费。
- 调用 `processIncomingMessage(in)`。

3. 归并与会话匹配
- 先 `userStore.CreateContact(...)` upsert 联系人。
- 幂等检查：`messageExistsBySourceID(...)`。
- 会话匹配优先级：
  - Reply-To plus-addressing 中的 `conv-uuid`
  - Subject 中 reference number
  - References / In-Reply-To 链路
  - 都失败则 `findOrCreateConversation(...)` 新建会话

4. 附件与消息入库
- `uploadMessageAttachments(...)`（含 Content-ID 去重与 inline 替换）。
- `InsertMessage(...)`（会更新会话 last_message，广播新消息，触发 message webhook）。

5. 业务副作用
- 新会话：触发 `EventConversationCreated` + `automation.EvaluateNewConversationRules(...)`。
- 已有会话：
  - `ReOpenConversation(...)`
  - 设置 `waiting_since`
  - `automation.EvaluateConversationUpdateRules(..., EventConversationMessageIncoming)`
  - SLA：`CreateNextResponseSLAEvent(...)`

### 6.1.3 出站消息路径（Queue -> Scan -> Send -> Update）

1. 入队
- `QueueReply(...)` 创建消息，状态 `pending`，生成 `SourceID`（message-id），写库。

2. 扫描与发送 worker
- `Manager.Run(...)` 定时扫描 `get-outgoing-pending-messages`。
- 防重：`outgoingProcessingMessages` map。
- `MessageSenderWorker()` 调 `sendOutgoingMessage(...)`。

3. 发送前处理
- `RenderMessageInTemplate(...)`（当前 email 模板渲染）。
- `attachAttachmentsToMessage(...)`。
- 补齐 References / In-Reply-To。
- 调用渠道 `inbox.Send(message)`。

4. 渠道发送（email）
- `Email.Send(...)`：OAuth 刷新、SMTP pool 选择、headers 组装、附件发送。

5. 发送后状态推进
- 成功：`UpdateMessageStatus(...sent)`；失败：`...failed`。
- 非系统用户回复时：
  - 更新 `first_reply_at` / `last_reply_at`
  - 清空 `waiting_since`
  - SLA `SetLatestSLAEventMetAt(...next_response)`
  - 触发 `automation.EvaluateConversationUpdateRulesByID(..., EventConversationMessageOutgoing)`

### 6.1.4 入站 vs 出站关键差异

- 触发源：入站来自渠道回调；出站来自 API 请求。
- 初始状态：入站 `received`，出站 `pending`。
- 会话决策：入站需要“匹配或新建”；出站默认已有会话。
- SLA 语义：入站创建 next response 事件；出站将该事件标记 met。
- 自动化事件：入站触发 `message.incoming`/new_conversation；出站触发 `message.outgoing`。
- 容错形态：出站可重试（`MarkMessageAsPending`）；入站更偏幂等去重与失败回滚（附件失败可删除新建会话）。


## 6.2 Channel 抽象层（internal/inbox/channel/）

### 6.2.1 抽象接口设计

`internal/inbox/inbox.go` 定义：

- `MessageHandler`：`Receive(context.Context) error`、`Send(models.Message) error`
- `Identifier`：`Identifier() int`
- `Closer`：`Close() error`
- `Inbox`：聚合上述接口 + `FromAddress() string` + `Channel() string`

这使得渠道实现对上层 conversation 来说是统一的“收 + 发 + 生命周期”能力。

### 6.2.2 当前实现：email

- 目录：`internal/inbox/channel/email/`
- 核心能力：
  - IMAP 收件读取
  - SMTP 发件（多服务器池）
  - OAuth2 token refresh
  - plus-addressing（`support+conv-uuid@...`）
  - loop-prevention headers

### 6.2.3 抽象程度评估

优点：
- conversation 不感知 email 协议细节，仅依赖 `Inbox.Send/Receive`。
- 生命周期统一（Register/Start/Reload/Close）。
- 可插拔渠道具备最小必要接口。

局限：
- `models.Message` 当前结构明显偏 email（headers、references、mime-like 语义）。
- Receive 侧输入 `IncomingMessage` 也带有邮件语义字段（Subject/SourceID/References）。
- 新渠道若非邮件（如 WhatsApp/Telegram）会有字段映射损耗。

### 6.2.4 新增渠道需要实现什么

至少需要：

1. 实现 `Inbox` 接口全部方法：
- `Receive(ctx)`
- `Send(message)`
- `Identifier()`
- `FromAddress()`
- `Channel()`
- `Close()`

2. 在收件侧把渠道事件转换为 `conversation/models.IncomingMessage`，并调用 `EnqueueIncoming(...)`。

3. 在发件侧把 `conversation/models.Message` 映射到渠道 API（并处理状态回写失败逻辑）。

4. 在 inbox manager 的 init 注册流程接入该渠道构造函数。

5. 若有鉴权 token，需要实现 refresh + config 回写（可参考 email 的 callback 方案）。


## 6.3 Automation 引擎

### 6.3.1 规则定义结构

规则核心是 `RuleRecord`（数据库）+ `Rule`（运行态 JSON）：

- `Type`：`new_conversation` / `conversation_update` / `time_trigger`
- `Events`：如 `conversation.message.incoming`、`conversation.status.change`
- `ExecutionMode`：`all` 或 `first_match`
- `GroupOperator`：组间 `AND/OR`
- `Groups[]`：每组 `LogicalOp + RuleDetail[]`
- `Actions[]`：匹配成功后的动作列表

### 6.3.2 Evaluator 工作方式

1. 按规则类型/事件过滤规则。
2. 对每条规则：
- 最多 2 个分组
- 每组按 `AND/OR` 评估多个条件
- 组间再按 `GroupOperator` 合并
3. 命中后逐个执行动作（委托 `conversation.ApplyAction`）。
4. 若 `ExecutionMode=first_match`，执行完首条命中规则即停止。

### 6.3.3 支持的条件类型

字段类型：
- 会话字段（`conversation`）
- 联系人自定义属性（`contact_custom_attribute`）

典型字段：
- `subject`、`content`、`status`、`priority`
- `assigned_user`、`assigned_team`、`inbox`
- `contact_email`
- `hours_since_created/first_reply/last_reply/resolved`

运算符：
- `contains`、`not contains`
- `equals`、`not equals`
- `set`、`not set`
- `greater than`、`less than`

### 6.3.4 支持的动作类型

- `assign_team`
- `assign_user`
- `set_status`
- `set_priority`
- `send_private_note`
- `send_reply`
- `set_sla`
- `add_tags`
- `set_tags`
- `remove_tags`
- `send_csat`

动作权限映射由 `ActionPermissions` 声明（如 assign 需 conversations 更新权限，reply 需 messages.write）。


## 6.4 AI 模块

### 6.4.1 当前 AI 能力范围

- 读取 prompt 模板（系统提示词）+ 用户输入，调用默认 provider 获取补全。
- 当前生产可用 provider 为 OpenAI。
- 管理端支持读取 prompts、更新 provider API key。

### 6.4.2 Provider 抽象层

- `ProviderClient` 接口：`SendPrompt(payload PromptPayload) (string, error)`
- `ProviderType`：当前有 `openai`，并预留 `claude` 常量。
- `PromptPayload`：`SystemPrompt` + `UserPrompt`。

`Manager.Completion` 流程：
1. 读 prompt（按 key）
2. 取默认 provider（`get-default-provider`）
3. 解密 API key
4. 构造 client 并调用 `SendPrompt`

### 6.4.3 OpenAI 实现要点

- endpoint：`/v1/chat/completions`
- model：`gpt-4o-mini`
- 参数：`max_tokens=1024`、`temperature=0.7`
- 错误语义：
  - 401 -> `ErrInvalidAPIKey`
  - 空 key -> `ErrApiKeyNotSet`

### 6.4.4 使用场景

- 智能回复建议
- 文案润色/改写
- 自动化动作中可拓展 AI 生成内容（当前动作引擎未直接内嵌 AI action，但 architecture 上可扩展）

---

## 7. 跨模块依赖拓扑（简化）

- `conversation` 是业务核心枢纽：
  - 上游：`cmd/messages.go`、`cmd/conversation.go`
  - 下游：`inbox`、`automation`、`sla`、`webhook`、`notification`、`ws`、`media`、`template`、`user`、`team`

- `inbox` 是渠道适配层：
  - 对 conversation 提供“收发统一抽象”
  - 当前 email 实现最完整

- `automation` 是规则驱动器：
  - 被 conversation 事件驱动
  - 通过 `conversation.ApplyAction` 回写领域状态

- `sla` 是时效状态机：
  - 被消息进出站推动指标状态
  - 与通知系统协作告警

- `auth` + `authz` + `role` + `user` 组成权限与身份域

---

## 8. 设计结论与演进建议

1. 现状优势
- 模块边界清晰，conversation 作为 orchestration 中枢。
- 事件驱动路径完整（automation/webhook/ws/notification 均有触发点）。
- inbox channel 抽象已经具备扩展基础。

2. 关键技术债
- `models.Message` 偏 email 语义，跨渠道泛化空间有限。
- conversation 责任较重（业务编排、状态推进、副作用触发高度集中）。

3. 优先演进方向
- 抽象统一“消息协议层”以降低对 email header 的耦合。
- 将 conversation 的副作用触发进一步事件化（domain event + async consumers）。
- AI/Automation 联动可引入受控的 AI action（带审计与回滚策略）。

---

## 9. 覆盖清单校验

已覆盖模块（23/23）：

1. conversation
2. inbox
3. user
4. envelope
5. team
6. role
7. authz
8. auth
9. tag
10. automation
11. autoassigner
12. ai
13. sla
14. webhook
15. notification
16. ws
17. media
18. search
19. template
20. csat
21. macro
22. view
23. setting
