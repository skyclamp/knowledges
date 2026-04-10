# Prompt 05: AI 能力、自动化引擎与 Agent 集成可行性分析

## 任务

请深入分析 libredesk 的 AI 模块、自动化引擎、Webhook 系统，评估接入自定义 AI Agent 的可行性。

## 背景需求

我们希望实现：
- 客户消息到达后，AI Agent 自动回复（基于 RAG 知识库）
- AI Agent 可以选择 close 会话，或转交给人工客服
- 至少能定制 Agent 的部分行为

## 需要分析的文件

### AI 模块
1. `internal/ai/ai.go` — AI 模块主文件
2. `internal/ai/openai.go` — OpenAI provider 实现
3. `internal/ai/provider.go` — Provider 接口定义
4. `internal/ai/models/` — AI 数据模型
5. `internal/ai/queries.sql` — AI 相关数据查询
6. `cmd/ai.go` — AI 相关 API handler

### 自动化引擎
7. `internal/automation/automation.go` — 自动化主逻辑
8. `internal/automation/evaluator.go` — 规则评估器
9. `internal/automation/evaluator_test.go` — 测试用例（反映支持的条件/动作）
10. `internal/automation/models/` — 规则数据模型
11. `internal/automation/queries.sql` — 规则查询
12. `cmd/automation.go` — 自动化 API handler

### Webhook 系统
13. `internal/webhook/` — 所有文件
14. `cmd/webhooks.go` — Webhook handler

### 宏系统
15. `internal/macro/` — 宏定义和执行

## 输出要求

写入文件 `/Users/wenkai/workspace/knowledges/libredesk/05-ai-automation-extensibility.md`，包含：

### 1. 当前 AI 能力分析
- AI Provider 接口定义（完整签名）
- 当前支持的 AI 操作（rewrite? summarize? 等）
- AI 在前端的使用入口
- AI 配置方式（如何配置 API key 等）
- AI 调用的上下文信息（传了什么给 AI）

### 2. 自动化引擎详解
- 规则数据结构：Rule → Conditions → Actions
- 支持的条件类型完整列表（附描述）
- 支持的动作类型完整列表（附描述）
- 规则触发时机（何时评估规则）
- 执行模式：all vs first_match
- 评估器的工作流程

### 3. Webhook 系统
- 支持的事件类型（从 schema.sql 的 ENUM 和代码中提取）
- Webhook 投递机制（队列、重试）
- Payload 格式
- SSRF 防护

### 4. AI Agent 集成方案评估

#### 方案 A：通过 Webhook + 外部 Agent 服务
- 使用 `message.created` webhook 触发外部 AI Agent
- Agent 通过 API 回复消息、更改状态
- 分析：libredesk 是否有 API 允许外部系统发送消息和修改会话状态？
- 需要的 API 端点列表

#### 方案 B：扩展 Automation 引擎
- 在 automation action 中添加 "call AI agent" 动作
- 分析 automation 的 action 扩展机制
- 需要修改哪些代码

#### 方案 C：扩展 AI Provider
- 将 AI Provider 从 "rewrite 工具" 扩展为 "对话 agent"
- 分析 provider 接口是否支持这种扩展

#### 方案 D：新增 Channel 类型作为 Agent
- 将 AI Agent 实现为一个虚拟 "inbox channel"
- 分析可行性

### 5. 推荐方案与实施路径
- 综合评估各方案的优劣
- 推荐最佳方案
- 所需代码变更范围估算
