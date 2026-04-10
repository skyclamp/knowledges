# Prompt 6: Chatwoot AI Agent / Captain / AgentBot 分析

## 任务
深入分析 Chatwoot 的 AI 和自动化能力，包括 Captain（Enterprise AI 功能）、AgentBot（OSS 机器人框架）和自动化规则系统。这是我们最关心的部分——我们希望接入自定义 AI Agent 来自动回复、关闭会话或转给人工客服。

## 输出要求
将分析结果写入 `/Users/wenkai/workspace/knowledges/chatwoot/output/06-ai-agent-captain.md`，使用中文撰写。

## 需要分析的文件和目录

### AgentBot（OSS 核心机器人框架）— 重点
- `app/models/agent_bot.rb` — 机器人模型
- `app/models/agent_bot_inbox.rb` — 机器人与收件箱绑定
- 搜索 `agent_bot` 相关的所有控制器（api/v1 和 platform 下）
- 搜索 `agent_bot` 相关的所有服务
- 搜索 `agent_bot` 相关的所有 listener 和 dispatcher
- `app/dispatchers/` 目录 — 事件分发机制
- `app/listeners/` 目录 — 所有 listener，特别关注与 bot/webhook 相关的

### Webhook 系统（Bot 通信机制）
- `app/models/webhook.rb`
- `app/services/webhooks/` 或搜索 webhook trigger 相关代码
- `app/controllers/api/v1/accounts/` 中 webhook 相关控制器
- 搜索 `webhook_data` 或 `deliver_webhook` 理解 webhook 触发点

### Automation Rules（自动化规则）
- `app/models/automation_rule.rb`
- `app/services/automation_rules/` 目录
- `app/listeners/automation_rule_listener.rb`（如果存在）

### Captain（Enterprise AI 功能）— 重点
- `enterprise/app/models/captain/` 目录 — 所有 Captain 相关模型
- `enterprise/app/models/captain_inbox.rb`
- `enterprise/app/services/captain/` 目录 — Captain 服务
- `enterprise/app/controllers/api/v1/accounts/captain/` 目录
- `enterprise/app/listeners/captain_listener.rb`
- `enterprise/app/jobs/captain/` 目录
- `enterprise/lib/captain/` 目录
- `lib/captain/` 目录 — OSS 中的 Captain 基础代码

### LLM 集成
- `config/llm.yml` — LLM 配置
- `lib/llm/` 目录 — LLM 相关代码
- `lib/llm_constants.rb`
- `app/services/llm_formatter/` — LLM 格式化

### Dialogflow 集成
- 搜索 `dialogflow` 相关文件 — 了解已有的 NLU 集成方式

### 会话分配与转交
- `app/models/conversation.rb` 中的 assignee 相关字段和方法
- `app/services/auto_assignment/` 目录
- 搜索 `handoff` 或 `transfer` 或 `assign` 相关逻辑

### 关键字段（在 db/schema.rb 中搜索）
- `agent_bot_id` 字段在哪些表中出现
- `assignee_agent_bot_id` — conversations 表中的机器人分配字段

## 输出文档结构要求

```markdown
# Chatwoot AI Agent 与自动化能力

## 1. AI/自动化能力全景
（Chatwoot 提供了哪些 AI 和自动化能力，各自的定位）

## 2. AgentBot 框架（OSS）
### 2.1 AgentBot 模型设计
（属性、类型、访问令牌）
### 2.2 AgentBot 与 Inbox 绑定
### 2.3 AgentBot Webhook 通信机制
（当会话/消息事件发生时，如何通知外部 Bot）
### 2.4 AgentBot 可以执行的操作
（通过 API 回复消息、更改会话状态、分配给人工等）
### 2.5 AgentBot 生命周期
（创建 Bot → 绑定 Inbox → 接收事件 → 调用 API 操作）

## 3. Webhook 事件系统
### 3.1 支持的 Webhook 事件列表
### 3.2 Webhook Payload 结构
### 3.3 事件触发时机与流程

## 4. 自动化规则引擎
### 4.1 规则结构（条件 + 动作）
### 4.2 支持的条件类型
### 4.3 支持的动作类型
### 4.4 执行时机

## 5. Captain（Enterprise AI）
### 5.1 Captain 架构概览
### 5.2 Captain Inbox 配置
### 5.3 AI 回复生成流程
### 5.4 AI 能力列表（摘要、建议回复、标签建议等）
### 5.5 LLM 配置与模型支持
### 5.6 知识库/RAG 集成方式

## 6. Dialogflow 集成
（已有的 NLU/Bot 平台集成方式）

## 7. 自定义 AI Agent 接入方案评估
### 7.1 方案一：通过 AgentBot + Webhook
（利用 OSS 的 AgentBot 框架，外部服务接收 webhook，调用 API 操作）
### 7.2 方案二：通过 API Channel + 外部路由
### 7.3 方案三：扩展 Captain（如果是 Enterprise）
### 7.4 推荐方案与可行性分析
（对于我们的需求——AI 自动回复、关闭会话、转人工——哪个方案最合适）

## 8. 会话转交机制
### 8.1 Agent 分配模型
### 8.2 如何从 Bot 转给人工
### 8.3 如何从人工转回 Bot
```
