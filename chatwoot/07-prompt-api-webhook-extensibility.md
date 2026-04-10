# Prompt 7: Chatwoot API、Webhook 与扩展能力分析

## 任务
分析 Chatwoot 的对外 API 体系、Webhook 系统和整体扩展能力，评估外部系统集成的便利性。

## 输出要求
将分析结果写入 `/Users/wenkai/workspace/knowledges/chatwoot/output/07-api-webhook-extensibility.md`，使用中文撰写。

## 需要分析的文件和目录

### Swagger/OpenAPI 文档
- `swagger/` 目录 — 完整浏览结构
- `swagger/paths/` — 所有 API 路径定义文件
- `swagger/definitions/` — 数据模型定义
- `swagger/index.yml` — API 文档入口

### Account API（v1）— 核心业务 API
- `app/controllers/api/v1/accounts/` 目录 — 列出所有控制器
- 选读以下关键控制器的完整代码：
  - `conversations_controller.rb` — 会话操作
  - `messages_controller.rb` — 消息操作
  - `contacts_controller.rb` — 联系人操作
  - `inboxes_controller.rb` — 收件箱管理
  - `agent_bots_controller.rb` — AgentBot 管理（如果存在）
  - `webhooks_controller.rb` — Webhook 配置

### Platform API（跨账户 API）
- `app/controllers/platform/` 目录 — 所有 Platform API 控制器
- 理解 Platform API 的认证方式和使用场景

### Public API（无认证公开 API）
- `app/controllers/public/` 目录
- 理解公开 API 的设计（用于 Widget 等场景）

### Widget API
- `app/controllers/api/v1/widget/` 目录（如果存在）

### Webhook 系统
- `app/models/webhook.rb` — Webhook 配置模型
- 搜索代码中所有 `webhook` 事件触发点
- 搜索 `WEBHOOK_EVENTS` 或类似常量
- `app/dispatchers/` — 事件分发器

### 集成体系
- `app/models/integrations/` 目录（如果存在）
- `app/models/integrations.rb`
- `config/integration/` 目录
- `app/services/` 中的集成相关服务（slack, dialogflow 等）

### Dashboard App（自定义应用扩展）
- `app/models/dashboard_app.rb`
- 相关控制器

### 认证机制
- 搜索 `authenticate` 和 `access_token` 理解 API 认证方式
- `app/models/access_token.rb`
- `app/models/platform_app.rb`
- `app/models/platform_app_permissible.rb`

## 输出文档结构要求

```markdown
# Chatwoot API、Webhook 与扩展能力

## 1. API 体系总览
### 1.1 API 层次结构
（Account API / Platform API / Public API / Widget API 的关系图）
### 1.2 认证方式
（User Token / Bot Token / Platform App Token / HMAC 等）
### 1.3 API 基础约定
（基础 URL、分页、错误格式、Rate Limiting）

## 2. 核心业务 API
### 2.1 Conversation API
（CRUD、状态变更、分配、标签、备注等操作详情）
### 2.2 Message API
（发送消息、消息类型、附件、私有备注）
### 2.3 Contact API
（联系人管理、搜索、合并）
### 2.4 Inbox API
（创建/管理收件箱）
### 2.5 Agent Bot API
（Bot 管理、绑定 Inbox）

## 3. Platform API
### 3.1 适用场景
### 3.2 认证方式
### 3.3 主要操作（Account/User/AgentBot CRUD）

## 4. Webhook 系统
### 4.1 Webhook 配置
（如何创建和管理 webhook）
### 4.2 事件类型完整列表
（每个事件的触发时机和 payload 结构概要）
### 4.3 AgentBot Webhook vs Account Webhook
（两种 webhook 的区别和使用场景）

## 5. 集成体系
### 5.1 内置集成列表
（Slack, Dialogflow, Shopify, Linear 等）
### 5.2 集成实现模式
### 5.3 Dashboard App
（自定义嵌入应用扩展工作台）

## 6. 扩展能力评估
### 6.1 外部 AI Agent 接入的 API 使用指南
（需要用到哪些 API、完整的调用流程）
### 6.2 API 能力矩阵
（对于我们的需求，哪些已有 API 可以直接用）：
| 需求 | 对应 API | 是否可用 |
|------|---------|---------|
| 接收新消息通知 | Webhook | ? |
| AI 自动回复 | Message API | ? |
| 关闭会话 | Conversation API | ? |
| 转给人工客服 | Conversation Assignment API | ? |
| 获取会话历史 | Message API | ? |
| 获取联系人信息 | Contact API | ? |

### 6.3 限制与注意事项
```
