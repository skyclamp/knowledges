# Chatwoot 产品概述

## 1. 产品定位

Chatwoot 是一个现代化、开源、可自托管的客户支持平台，定位上可以理解为 Intercom、Zendesk、Salesforce Service Cloud 一类客服系统的开源替代方案。它的核心目标不是单一聊天工具，而是把企业和客户之间分散在不同渠道的沟通统一收敛到一个工作台里，由客服、运营、自动化规则和机器人共同处理。

它主要解决三类问题：

- 多渠道消息分散，客服需要在不同后台来回切换。
- 团队协作低效，分配、跟进、备注、升级、统计缺少统一闭环。
- 企业希望保留客户数据控制权，同时又需要自动化、AI 和集成能力。

从源码可以看出，Chatwoot 的目标用户覆盖：

- 需要自建客服系统的 SaaS、出海、电商、互联网产品团队。
- 以客服/支持中心为核心的中小企业到中大型团队。
- 对数据主权、可定制性、可扩展性有要求的技术型组织。

其产品核心可以概括为一句话：以 Account 为租户边界，以 Inbox 为渠道入口，以 Conversation 为工作单元，以 Message 为事件载体，围绕联系人、分配、自动化、知识库和 AI 构建统一客户服务平台。

## 2. 核心功能全景

- 全渠道消息：把网站在线聊天、Email、Facebook、Instagram、WhatsApp、Telegram、Line、SMS、TikTok、API 等渠道统一进入同一收件箱体系。
- 网站聊天组件：提供 Web Widget、预聊天表单、问候语、营业时间、离线提示和文件上传能力。
- 团队协作：支持私有备注、@提及、参与者、团队分组、收件箱成员、标签和快捷回复。
- 会话分配：支持手动分配、自动分配、轮询分配、按可用性分配，以及更高级的分配策略扩展。
- 联系人与 CRM 能力：维护联系人档案、联系方式、活动历史、自定义属性、分组标签，并在 Enterprise 中扩展到 Company。
- 自动化：通过 Automation Rule 按事件、条件和动作驱动自动打标签、改状态、分配、发消息、发 webhook、加私有备注等。
- Agent Bot：支持将 Inbox 绑定机器人，通过 webhook 型 Agent Bot 或更高级的 AI Assistant 参与对话。
- AI 能力：README 和安装配置表明产品内置 Captain AI 方向，包括 FAQ 检索、自动回复、Copilot、标签建议、语音转写、帮助中心检索等能力。
- 知识库/帮助中心：支持 Help Center、Portal、文章和 FAQ 自助服务，Enterprise 进一步提供文章向量化与 AI 检索相关能力。
- 运营活动：Campaign 可用于持续触达或一次性触达，在 Website、SMS、WhatsApp 等场景中主动触发消息。
- 通知体系：对会话创建、分配、新消息、提及、SLA 超时等事件产生通知，并支持 Push/Email 分发。
- 报表分析：支持会话、客服、Inbox、标签、团队、CSAT 等维度的报表和汇总能力。
- 集成能力：支持 Webhook、Slack、Dialogflow、Shopify、Linear、Google Translate 等外部集成。
- 安全与身份：基础版支持常规认证与双因素认证，Enterprise 进一步扩展到 SAML。
- 品牌与安装配置：安装级配置支持品牌名、Logo、域名、上传限制、Webhook 超时、第三方应用凭据等系统级管理。

从 feature flag 可以看到，产品的功能模块大体围绕以下分类组织：

- 全渠道消息
- 团队协作
- 自动化
- AI 能力
- 知识库/帮助中心
- 报表分析
- 集成

## 3. 核心领域概念

可以用下面这组关系理解 Chatwoot 的主数据模型：

```text
Account
  ├─ AccountUser ── User
  ├─ Inbox ── Channel
  │    ├─ InboxMember ── User
  │    ├─ AgentBotInbox ── AgentBot
  │    └─ ContactInbox ── Contact
  │              └─ Conversation
  │                    ├─ Message ── Attachment
  │                    ├─ assignee -> User
  │                    ├─ assignee_agent_bot -> AgentBot
  │                    ├─ team -> Team
  │                    ├─ campaign -> Campaign
  │                    └─ notifications / reporting / labels / automations
  ├─ Team
  ├─ Label
  ├─ AutomationRule
  ├─ Webhook
  ├─ Campaign
  ├─ CannedResponse
  ├─ Notification
  ├─ CustomAttributeDefinition
  └─ AssignmentPolicy
```

### 3.1 多租户模型

`Account` 是整个平台的租户根节点，也是绝大多数业务对象的归属边界。源码中 `Account` 直接关联用户、联系人、会话、消息、收件箱、团队、标签、自动化、活动、知识库文章、Webhook、自定义属性等几乎所有核心对象。

`Account` 的关键职责：

- 作为数据隔离边界，保证不同客户组织之间的数据互不混淆。
- 承载功能开关 `feature_flags`、系统限制 `limits`、配置 `settings`。
- 提供租户级默认行为，例如自动关闭会话、AI 功能开关、报表时区等。

`Account` 的关键属性：

- `name`、`domain`、`support_email`：租户标识与联系信息。
- `feature_flags`：功能是否启用的压缩位字段。
- `limits`：容量和套餐限制。
- `settings`：自动关闭、AI 模型、AI 子功能、报表时区等租户级配置。
- `status`：租户状态，至少有 active 和 suspended。

### 3.2 用户与角色体系

`User` 是平台级用户实体，负责认证、账号信息、2FA、SSO/OAuth 登录等通用身份能力；真正落到某个租户里的身份和权限，是通过 `AccountUser` 完成的。

`AccountUser` 是用户加入某个 Account 的关系对象，承担三件事：

- 声明该用户在租户内的角色。
- 维护租户内在线状态和可用性。
- 连接 Enterprise 中的自定义角色和容量策略。

角色体系分两层：

- OSS 基础层：`agent` 与 `administrator`。
- Enterprise 扩展层：`custom_role_id` 指向 `CustomRole`，可以给更细粒度的权限集合。

另一个重要层次是 `InboxMember`。它表示某个用户是否能处理某个 Inbox 中的会话，因此用户权限是租户级的，处理范围则可以被 Inbox 级别进一步收缩。

### 3.3 Inbox 与 Channel 模型

`Inbox` 是渠道入口的统一抽象。业务上它代表一个可接待客户消息的“服务入口”，而底层真实渠道则由多态关联 `channel` 承载。

为什么 Inbox 很关键：

- 会话一定属于某个 Inbox。
- 联系人与渠道标识的绑定通过 `ContactInbox` 落在 Inbox 上。
- 成员、自动分配、机器人、Webhook、活动都围绕 Inbox 配置。

`Inbox` 的关键属性包括：

- `channel_type`、`channel_id`：指向具体渠道对象。
- `name`、`business_name`、`email_address`：对外展示信息。
- `enable_auto_assignment`、`auto_assignment_config`：分配行为。
- `greeting_message`、`out_of_office_message`、`working_hours_enabled`：接待体验配置。
- `portal_id`：将 Inbox 与知识库 Portal 关联。

OSS 渠道模型从 `app/models/channel/` 可以看到主要包括：

- `Channel::WebWidget`：网站聊天组件。
- `Channel::Api`：API 渠道，适合系统集成或自定义客户端。
- `Channel::Email`：邮箱接入。
- `Channel::FacebookPage`：Facebook Messenger。
- `Channel::Instagram`：Instagram 私信。
- `Channel::Telegram`：Telegram。
- `Channel::Line`：Line。
- `Channel::Sms`：通用 SMS。
- `Channel::TwilioSms`：Twilio 承载的 SMS/WhatsApp，`medium` 区分短信和 WhatsApp。
- `Channel::Whatsapp`：Meta WhatsApp Business。
- `Channel::TwitterProfile`：Twitter/X。
- `Channel::Tiktok`：TikTok。

Enterprise 还额外增加：

- `Channel::Voice`：语音渠道，目前从实现上明显依赖 Twilio Voice。

### 3.4 Conversation 生命周期

`Conversation` 是 Chatwoot 最核心的业务对象，等价于“一个联系人在某个渠道入口上的一次服务会话”。它既像工单，又保留实时聊天的连续上下文。

Conversation 的关系定位：

- 属于一个 `Account`。
- 属于一个 `Inbox`。
- 属于一个 `Contact`。
- 通过 `ContactInbox` 与“联系人在该渠道下的身份”关联。
- 可分配给 `User` 或 `AgentBot`。
- 可关联 `Team`、`Campaign`。

Conversation 的关键属性：

- `status`：`open`、`resolved`、`pending`、`snoozed`。
- `priority`：`low`、`medium`、`high`、`urgent`。
- `display_id`：租户内可读编号，通过数据库序列生成。
- `uuid`：全局唯一外部标识。
- `waiting_since`：客户等待开始时间，是 SLA 和排队体验的关键字段。
- `first_reply_created_at`：首响时间。
- `cached_label_list`、`custom_attributes`、`additional_attributes`：分类与上下文扩展。

生命周期大致如下：

1. 联系人在某个 Inbox 下产生或复用一个 `ContactInbox`。
2. 系统创建 `Conversation`，默认通常是 open；若联系人被屏蔽则直接 resolved；若 Inbox 存在活跃 Bot，则初始可能是 pending。
3. 客户发送消息后，Conversation 进入等待状态，`waiting_since` 被设置。
4. 人工客服或机器人回复后，等待状态被清空，必要时记录首次回复时间。
5. 会话可以在 open、pending、snoozed、resolved 之间切换，也可以被重新打开。

这说明 Chatwoot 不是单纯消息流水，而是显式围绕“待处理中的会话”建模。

### 3.5 Message 模型

`Message` 是会话中的原子事件，也是绝大多数实时通知、Webhook、搜索和自动化的触发载体。

其设计有三层重要维度：

- `message_type`：区分 `incoming`、`outgoing`、`activity`、`template`。
- `content_type`：区分文本、表单、卡片、文章、邮件、CSAT、语音通话等不同内容形态。
- `sender`：多态发送者，可来自 `Contact`、`User`、`AgentBot`，Enterprise 中还会出现 `Captain::Assistant`。

Message 的关键职责：

- 保存消息正文、结构化内容属性、外部源 ID。
- 关联附件。
- 推动会话状态演进，例如更新 `waiting_since`、记录首次回复。
- 作为通知、Webhook、索引、搜索和自动化的核心事件源。

其关键属性包括：

- `content`、`processed_message_content`。
- `message_type`、`content_type`、`status`。
- `private`：是否是内部私有消息。
- `content_attributes`：承载富结构元数据，例如表单提交、邮件上下文、回复引用、翻译内容、语音数据等。
- `external_source_ids`：对接外部系统消息 ID。

这套模型说明 Chatwoot 的消息层并非只支持“文本聊天”，而是支持多种结构化交互与渠道回执。

### 3.6 Contact 模型

`Contact` 表示终端客户，是所有对话的客户主体。它是租户级实体，而不是某个单一渠道下的临时身份。

`Contact` 的关键属性：

- 基础身份：`name`、`email`、`phone_number`、`identifier`。
- 分类：`contact_type`，包括 `visitor`、`lead`、`customer`。
- 扩展字段：`additional_attributes`、`custom_attributes`。
- 行为字段：`last_activity_at`、`blocked`。
- Enterprise 扩展：`company_id` 指向 `Company`。

真正把 Contact 映射到渠道身份的是 `ContactInbox`：

- 一个 Contact 可以在多个 Inbox 中出现。
- `source_id` 保存该联系人在该渠道下的唯一外部标识，例如手机号、WhatsApp 标识等。
- `ContactInbox` 是 Conversation 创建的直接前置条件之一。

因此 Contact 是“统一客户档案”，而 ContactInbox 是“客户在某渠道中的地址”。

### 3.7 自动化与规则引擎

自动化核心由 `AutomationRule` 承担。它本质上是一个事件驱动规则对象：

- `event_name` 定义在哪类事件上触发。
- `conditions` 定义命中条件。
- `actions` 定义执行动作。
- `active` 表示规则是否启用。

从源码可见的条件字段覆盖：

- 会话状态、优先级、Inbox、Team、Assignee、标签。
- 联系人邮箱、手机号、国家、城市、语言、来源页面等。
- 消息类型和内容相关条件。
- 租户自定义属性。

动作覆盖：

- 发消息、加/删标签、发邮件、分配团队、分配客服。
- 发送 Webhook、静音会话。
- 改状态、改优先级、挂起、解决、打开。
- 添加私有备注、发附件、发送邮件转录等。

`AssignmentPolicy` 则是自动化分配的另一条主线。OSS 中它至少支持：

- 会话挑选优先级：最早创建、等待最久。
- 分配顺序：基础实现是 round robin。

Enterprise 通过 `enterprise/app/models/enterprise/concerns/assignment_policy.rb` 和相关模型，继续扩展更高级的分配能力。

### 3.8 Agent Bot 模型

Chatwoot 里有两类“机器人”概念：

- OSS 的 `AgentBot`。
- Enterprise 的 `Captain::Assistant`。

`AgentBot` 是基础机器人抽象：

- 通过 `AgentBotInbox` 绑定到一个或多个 Inbox。
- 通过 `outgoing_url` 与外部 bot 服务通信。
- 可以直接成为消息发送者，也可以作为会话当前分配对象。

它更像“Webhook 驱动机器人接入层”。

`Captain::Assistant` 则是更完整的 AI Agent 模型：

- 属于某个 Account。
- 通过 `CaptainInbox` 绑定 Inbox。
- 有 `documents`、`responses`、`scenarios`、`guardrails`、`response_guidelines` 等专属结构。
- 内置工具链至少包括 FAQ 检索与 handoff，并支持自定义工具。

从源码上看，Captain 不只是聊天补全，而是具备知识检索、场景切换、转人工、Copilot、嵌入搜索等更完整的 AI 工作流。

## 4. OSS vs Enterprise 功能边界

从代码结构、`config/features.yml` 中的 `premium` 标记以及 `enterprise/app/models` 目录可以比较清晰地看出产品边界。

### OSS 已具备的核心能力

- 多租户 Account / User / Contact / Inbox / Conversation / Message 基础模型。
- 多数主流文字渠道接入：Web Widget、API、Email、Facebook、Instagram、Telegram、Line、SMS、WhatsApp、TikTok、Twitter。
- 团队协作：Team、Inbox 成员、标签、备注、提及、快捷回复。
- 自动化与基础自动分配。
- Agent Bot 接入。
- Webhook 与多种外部集成。
- Campaign、Help Center、报表、CSAT 等基础产品模块。

### Enterprise 或付费能力的明显边界

- `disable_branding`：去除品牌露出。
- `audit_logs`：审计日志。
- `sla`：SLA 策略、Applied SLA、SLA 事件。
- `custom_roles`：自定义角色权限模型。
- `saml`：企业级单点登录。
- `companies`：联系人公司实体。
- `channel_voice`：语音渠道。
- `captain_integration` / `captain_integration_v2`：Captain AI 能力。
- `advanced_search` / `advanced_search_indexing`：高级搜索与索引。
- `advanced_assignment`：高级分配能力。
- `conversation_required_attributes`：会话必填属性。
- `csat_review_notes`：CSAT 复核说明。
- `help_center_embedding_search`：帮助中心嵌入搜索。

### 更准确的边界判断

需要注意，Chatwoot 的边界不完全等价于“开源代码 vs 闭源代码”。它更接近三层：

- OSS 通用能力：直接在主应用中可见。
- Enterprise 叠加能力：通过 `enterprise/` 覆盖或扩展核心模型。
- Cloud/Internal 能力：部分 flag 虽在代码中存在，但带有 `chatwoot_internal` 标识，更多用于官方云或内部功能开关。

因此在产品分析时，更适合表述为“源码可见的能力边界”和“商业版/云版开启的能力边界”，而不是简单二分。

## 5. 技术栈总结

### 后端

- Ruby 3.4.4。
- Ruby on Rails 7.1。
- Devise + devise_token_auth + devise-two-factor 负责认证与多因素认证。
- Pundit 负责授权。
- Wisper/dispatcher 风格事件分发贯穿大量领域事件。

### 前端

- Vue 3 为核心前端框架。
- Vite 负责构建。
- Tailwind CSS 负责样式体系。
- 状态管理同时存在 Pinia 与 Vuex，说明代码库处于演进兼容阶段。
- Vitest 用于前端测试，ESLint 用于代码质量。

### 数据与存储

- PostgreSQL 是主数据库。
- `pg_trgm`、`pgcrypto`、`vector` 扩展已启用。
- Active Storage 负责附件存储，支持 S3、Azure Blob、Google Cloud Storage。

### 缓存、异步与实时

- Redis 用于缓存/状态支撑。
- Sidekiq + sidekiq-cron 负责后台任务与定时任务。
- Action Cable 负责实时事件推送。

### 搜索与 AI

- `pg_search` 用于数据库内全文搜索。
- `searchkick` + `opensearch-ruby` 表明支持更高级的外部搜索方案。
- `pgvector`、`neighbor`、`ruby-openai`、`ai-agents`、`ruby_llm` 说明产品已经把向量检索和 LLM Agent 纳入主干架构。

### 渠道与第三方集成

- Facebook、Instagram、Line、Twilio、Telegram、Twitter、Shopify、Slack、Dialogflow、Linear、Google Translate 等均有直接依赖或模型支撑。
- Email 渠道支持 SMTP/OAuth 及 inbound mailbox 处理。

### 基础设施与可观测性

- Puma 作为 Web Server。
- Overmind/Foreman 用于本地多进程开发。
- Sentry、New Relic、Datadog、Elastic APM、Scout APM 均有接入位。
- OpenTelemetry 已用于 LLM 观测。

整体上，Chatwoot 的技术方案非常典型地服务于“实时客服平台”这一场景：Rails 承载强业务模型，Vue 承载操作台，PostgreSQL 作为核心事实存储，Redis/Sidekiq 负责异步和实时协作，外加搜索、向量检索和第三方渠道集成构成平台化能力。