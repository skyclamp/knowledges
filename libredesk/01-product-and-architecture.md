# Libredesk 产品概览与架构设计分析

## 1. 产品定位

### 1.1 是什么产品，解决什么问题
Libredesk 是一个现代化、开源、自托管（self-hosted）的客户支持平台（customer support desk），定位为“高性能、全渠道支持”的客服协同系统。

它主要解决的问题是：
- 多团队、多坐席协同处理客户对话的统一收敛与分发。
- 从“收件箱 + 会话 + 回复”扩展到“自动化 + SLA + 报表 + 审计 + 集成”的完整客服运营闭环。
- 企业可自部署并掌控数据，适用于对隐私、合规、可控性有要求的团队。

### 1.2 核心功能列表（README + 代码归纳）
从 README、路由与内部模块可提取的核心能力如下：
- 多共享收件箱（multi shared inbox）
- 会话管理：分配/改派、状态、优先级、标签、草稿、提及、未读、参与者
- 消息系统：来信/回信/活动消息、重试发送、媒体附件
- 客户与坐席管理：联系人、坐席、团队、角色权限
- 细粒度 RBAC 鉴权（角色 + 权限点）
- 自动化规则引擎（规则、权重、执行模式）
- 自动分配（auto assignment）
- SLA 策略、事件、预警与 breach 通知
- CSAT 满意度调查与反馈
- 宏（Macros）与模板（邮件模板/通知模板）
- 自定义属性（联系人/会话）
- 视图系统（个人视图/共享视图）
- 全文检索（会话、消息、联系人）
- 活动日志与用户通知中心
- AI 辅助改写（AI provider + prompts）
- Webhook 事件订阅与测试投递
- OAuth/OIDC 登录与邮箱接入授权
- WebSocket 实时推送（前端实时更新）

### 1.3 当前版本状态和路线图
- 当前仓库构建版本来自 Makefile 注入：优先使用 Git tag（形如 vX.Y.Z）与 commit hash 组合成构建字符串。
- VERSION 文件在源码中是占位格式（Git archive 占位符），因此“精确版本号”由构建环境决定。
- ROADMAP 状态：
  - Near Term 中多项已完成（Done）：Microsoft OAuth inbox、草稿保存、mentions、批量导入坐席。
  - Mid Term：全功能在线聊天组件（WIP）。
  - Long Term：WhatsApp 渠道（TODO）。

---

## 2. 技术栈

### 2.1 后端语言、框架、关键依赖库
- 语言：Go（go.mod 指定 go 1.25.0）
- HTTP 框架：fasthttp + fastglue
- 数据访问：sqlx + PostgreSQL（lib/pq）
- 配置管理：koanf（file + env + flag + confmap/rawbytes）
- 鉴权与授权：
  - 会话：simplesessions（Redis store）
  - OIDC/OAuth2：coreos/go-oidc + golang.org/x/oauth2
  - 权限：casbin
- 实时通信：fasthttp/websocket
- 邮件协议：IMAP/SMTP 相关库（go-imap, smtppool, enmime 等）
- 对象存储：S3（simples3）或本地文件系统
- 其他：
  - 国际化：go-i18n
  - 结构化日志：logf
  - SSRF 防护：ssrfguard（用于 webhook 场景）
  - 二进制静态资源打包：stuffbin

### 2.2 前端语言、框架、UI 库
- 框架：Vue 3
- 构建工具：Vite
- 状态管理：Pinia
- 路由：Vue Router
- 国际化：vue-i18n
- UI 与交互生态：
  - Shadcn 风格组件体系（项目 README 明确）
  - Radix Vue / Reka UI
  - Tailwind CSS + tailwindcss-animate
- 编辑器与富文本：TipTap、CodeMirror
- 图表与可视化：Unovis
- 请求与工具：Axios、Zod、Vee Validate

### 2.3 数据库、缓存、消息队列
- 主数据库：PostgreSQL（业务主存储）
- 缓存/会话存储：Redis（会话、认证相关状态）
- 消息处理模型：
  - 应用内 worker queue（消息发送/接收、通知、自动化、webhook、SLA 等）
  - 未看到独立外部 MQ（如 Kafka/RabbitMQ）

### 2.4 构建工具链
- 后端：Go build / go run
- 前端：pnpm + vite
- 资源封装：stuffbin 将 frontend/dist、i18n、static、schema.sql 打入单二进制
- 自动化：Makefile（build/test/demo-build/run-backend/run-frontend）
- 容器化：Dockerfile + docker-compose

---

## 3. 系统架构图（文字描述）

### 3.1 整体架构
典型部署形态为：
- 一个 Go 单体应用进程（HTTP API + WebSocket + 业务 worker + inbox 连接器）
- PostgreSQL（事务数据）
- Redis（会话与认证相关缓存）

即：Monolith App + Postgres + Redis。

### 3.2 进程内组件划分（基于 init/main 初始化顺序）
启动时，核心装配流程可概括为：
1. 解析命令行参数与配置（文件、环境变量、flag）。
2. 初始化文件系统（stuffbin embedded FS，失败则回退本地文件系统）。
3. 初始化 Postgres 与 Redis。
4. 执行安装/升级/设置系统用户密码等运维命令分支。
5. 从 DB 加载 settings，覆盖运行时配置。
6. 构建基础管理器：i18n、OIDC、Auth、Media、Inbox、User、Team、Role、Status、Priority、Tag、Template、View、Macro、AI、Search、Report、ActivityLog、CustomAttribute、Webhook、Notification、SLA、CSAT、Automation、Conversation 等。
7. 初始化 WebSocket Hub，并将其注入会话、通知分发链路。
8. 启动后台协程：
   - automation engine
   - autoassigner
   - conversation message processing + unsnoozer + draft cleaner
   - webhook delivery workers
   - notifier workers
   - SLA evaluator + notifications
   - media unlinked cleaner
   - user availability monitor
   - notification cleaner
9. 注册 HTTP/WS 路由并启动 fasthttp server。

### 3.3 组件间通信方式
- HTTP/JSON：前后端主 API 通信。
- WebSocket：服务端向坐席端推送实时事件（消息、通知、会话变化等）。
- 内部方法调用：大部分业务模块通过 manager/engine 直接调用。
- 进程内队列与 worker：消息处理、通知、自动化、webhook、SLA。
- Redis：主要用于 session store 与认证上下文，不是核心业务事件总线。

### 3.4 外部集成点
- 邮件服务：
  - SMTP（发送）
  - IMAP（拉取）
  - inbox channel = email
- AI 服务：当前 provider enum 为 openai，通过 ai_providers/ai_prompts 驱动。
- Webhook：对外发送 conversation/message 事件。
- OAuth/OIDC：
  - OIDC 用于 SSO 登录
  - OAuth 用于邮箱接入授权（inbox oauth authorize/callback）

---

## 4. 数据模型概览

### 4.1 核心实体关系
重点实体与关系可归纳为：
- user：区分 agent/contact（user_type）
- role 与 user_roles：RBAC 角色与权限
- team 与 team_members：团队与成员
- inbox：渠道入口（当前 channels enum 仅 email）
- conversation：会话主实体，关联 contact、inbox、assigned_user、assigned_team、status、priority、sla_policy
- conversation_messages：会话消息，关联 sender（agent/contact）
- tags 与 conversation_tags：标签体系
- contact_channels：联系人在某 inbox 的渠道标识（邮箱地址等）
- applied_slas / sla_events / scheduled_sla_notifications：SLA 执行与通知

### 4.2 关键 ENUM 类型及含义
schema.sql 中关键枚举包括：
- channels: email
- message_type: incoming / outgoing / activity
- message_sender_type: agent / contact
- message_status: received / sent / failed / pending
- content_type: text / html
- user_type: agent / contact
- conversation_assignment_type: Round robin / Manual
- automation_execution_mode: all / first_match
- macro_visibility: all / team / user
- view_visibility: all / team / user
- media_store: s3 / fs
- user_availability_status: online / away / away_manual / offline / away_and_reassigning
- applied_sla_status: pending / breached / met / partially_met
- sla_event_status: pending / breached / met
- sla_metric: first_response / resolution / next_response
- sla_notification_type: warning / breach
- webhook_event: conversation.created 等会话/消息事件
- user_notification_type: mention / assignment / sla_warning / sla_breach
- ai_provider: openai

### 4.3 重要关联关系
- conversations.contact_id -> users(id)（contact）
- conversations.assigned_user_id -> users(id)（agent，可空）
- conversations.assigned_team_id -> teams(id)（可空）
- conversations.inbox_id -> inboxes(id)
- conversation_messages.conversation_id -> conversations(id)
- conversation_messages.sender_id -> users(id)
- conversation_tags(conversation_id, tag_id)
- team_members(team_id, user_id)
- user_roles(user_id, role_id)
- contact_channels(contact_id, inbox_id)
- applied_slas.conversation_id + sla_events.applied_sla_id 构成 SLA 生命周期

---

## 5. API 设计概览

### 5.1 路由分组与命名模式（handlers）
整体命名风格：
- 统一前缀：/api/v1
- 资源化路径 + 动词语义后缀（如 /toggle, /retry, /apply）
- 按领域分组：
  - auth / oidc / settings
  - conversations / messages / drafts / search
  - views / shared-views
  - statuses / priorities / tags / macros
  - agents / contacts / teams / roles
  - automations / inboxes / oauth
  - templates / business-hours / sla
  - ai / custom-attributes / activity-logs
  - notifications / webhooks / reports
- 非 API 页面路由：SPA 页面入口与静态资源路由
- 公共端点：/health、/csat/{uuid}、/uploads/{uuid}

### 5.2 认证/鉴权机制（middlewares 推断）
- 认证（auth）：
  - 支持 API Key（Authorization header）
  - 支持 Session（Cookie）
- CSRF：
  - 对 POST/PUT/DELETE 校验 cookie 与 X-CSRFTOKEN 头一致
- 鉴权（perm）：
  - 权限字符串模式 object:action
  - 通过 Casbin Enforcer 执行授权判断
- 页面保护：
  - authPage / notAuthPage 控制已登录与未登录页面访问
- 媒体访问：
  - authOrSignedURL 支持“登录态”或“签名 URL”二选一

### 5.3 WebSocket 端点用途
- 端点：/ws（需认证）
- 作用：建立坐席端实时连接，服务端 Hub 按用户维度广播事件。
- 场景：消息更新、会话状态变化、站内通知等实时同步。

---

## 6. 配置体系

### 6.1 配置加载机制（文件 + 环境变量）
配置汇聚链路：
- 默认与命令行：--config 支持多文件合并
- 文件格式：TOML
- 环境变量：LIBREDESK_ 前缀，双下划线映射为层级点号
  - 例：LIBREDESK_APP__SERVER__ADDRESS -> app.server.address
- 启动后还会从 settings 表加载配置并覆盖到运行态（例如 app.logo_url 等）

### 6.2 关键配置分组及用途
- app：日志级别、环境、root url、站点品牌、加密密钥、更新检查
- app.server：监听地址/Unix socket、超时、请求体大小、cookie 安全策略
- db：PostgreSQL 连接与连接池
- redis：Redis 地址/认证/库
- upload：fs 或 s3 存储配置
- message：收发消息 worker 与队列容量
- notification：通知并发与队列
- automation：规则执行 worker 数
- autoassigner：自动分配周期
- conversation：unsnooze 周期、草稿保留策略
- webhook：并发、队列、超时、allowed_hosts（SSRF 防护白名单）
- sla：评估周期

---

## 7. 构建与部署模型

### 7.1 单二进制打包机制（stuffbin）
构建流程：
1. 前端 pnpm build 生成 frontend/dist
2. Go 编译输出 libredesk 二进制
3. stuffbin 将静态资源打包进二进制
   - frontend/dist
   - i18n
   - static
   - schema.sql

运行时优先从二进制内 FS 读取；开发或未打包时可回退本地文件系统。

### 7.2 Docker 部署模式
- Dockerfile：基于 alpine，复制 libredesk 二进制与 sample 配置，默认启动 ./libredesk。
- docker-compose：三容器
  - app
  - postgres:17-alpine
  - redis:7-alpine
- app 启动命令会自动串行执行：
  - --install --idempotent-install
  - --upgrade
  - 正式启动
- 挂载：
  - uploads 持久化
  - config.toml 外置

### 7.3 开发模式运行方式（前后端分离）
Makefile 提供清晰分离：
- run-backend：go run cmd/*.go（可注入 build/version ldflags）
- run-frontend：pnpm dev（Vite 开发服务器）

即本地开发通常采用“Go API 服务 + Vite 前端热更新”双进程模式；生产则是单二进制承载前后端静态资源。

---

## 补充结论
Libredesk 当前是一个边界清晰的 Go 单体：
- 在部署形态上保持简单（单应用 + PG + Redis）
- 在业务能力上已接近完整客服平台（会话、自动化、SLA、报表、审计、集成）
- 在扩展性上通过 internal 模块拆分、配置化、Webhook/OIDC/AI/provider 化设计，为后续渠道扩展（如 roadmap 的 live chat、WhatsApp）留出了明确演进路径。
