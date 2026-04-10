# Prompt 05: API 与集成层分析

## 任务

分析 WeKnora 的 HTTP API 路由、Handler 层、中间件、IM 即时通讯集成，输出 API 与集成层设计文档。

## 需要阅读的文件

1. **路由定义**:
   - `/Users/wenkai/oss/WeKnora/internal/router/router.go` — 主路由
   - `/Users/wenkai/oss/WeKnora/internal/router/task.go` — 后台任务路由
   - `/Users/wenkai/oss/WeKnora/internal/router/sync_task.go` — 同步任务路由

2. **Handler 层**（API 请求处理）:
   - `/Users/wenkai/oss/WeKnora/internal/handler/` — 整个目录
   - 重点：`auth.go`, `session/`（会话和聊天）, `knowledgebase.go`, `knowledge.go`, `chunk.go`, `model.go`, `custom_agent.go`, `datasource.go`, `mcp_service.go`, `tenant.go`, `organization.go`, `im.go`, `web_search.go`, `evaluation.go`, `faq.go`, `tag.go`, `skill_handler.go`, `system.go`, `initialization.go`, `message.go`

3. **中间件**:
   - `/Users/wenkai/oss/WeKnora/internal/middleware/` — 整个目录
   - `auth.go` — 认证中间件
   - `error_handler.go` — 错误处理
   - `logger.go` — 日志
   - `recovery.go` — panic 恢复
   - `trace.go` — 链路追踪
   - `language.go` — 多语言

4. **认证系统**:
   - `/Users/wenkai/oss/WeKnora/internal/handler/auth.go`
   - `/Users/wenkai/oss/WeKnora/docs/OIDC认证调用流程.md`

5. **IM 集成**（即时通讯）:
   - `/Users/wenkai/oss/WeKnora/internal/im/` — 整个目录
   - 重点：`service.go`（IM 服务核心）, `adapter.go`（适配器接口）, `types.go`, `command.go`, `command_registry.go`
   - IM 子命令：`cmd_help.go`, `cmd_info.go`, `cmd_search.go`, `cmd_stop.go`, `cmd_clear.go`
   - 各 IM 平台适配器：`wecom/`, `feishu/`, `slack/`, `telegram/`, `dingtalk/`, `mattermost/`
   - `qaqueue.go` — 问答队列
   - `ratelimit.go` — 限速
   - `think.go` — IM 思考指示
   - `/Users/wenkai/oss/WeKnora/docs/IM集成开发文档.md`

6. **流式传输**:
   - `/Users/wenkai/oss/WeKnora/internal/stream/` — 整个目录

7. **事件系统**:
   - `/Users/wenkai/oss/WeKnora/internal/event/` — 整个目录

8. **Swagger API 文档**:
   - `/Users/wenkai/oss/WeKnora/docs/swagger.yaml` 或 `swagger.json`
   - `/Users/wenkai/oss/WeKnora/docs/docs.go`

9. **SDK 客户端**（了解 API 的调用方式）:
   - `/Users/wenkai/oss/WeKnora/client/` — Go SDK

## 输出要求

写一份中文文档，保存到 `/Users/wenkai/workspace/knowledges/WeKnora/docs/05-API与集成层.md`，包含以下章节：

### 1. API 设计总览
- API 分组和版本管理
- RESTful 设计风格
- 完整的 API 路由表（按模块分组列出所有端点、HTTP 方法、Handler 函数）
- 认证方式（JWT、API Key、OIDC）

### 2. Handler 层设计
- Handler 的职责（参数校验 → 调用 Service → 返回响应）
- 请求/响应格式约定
- 错误码规范
- 分页约定

### 3. 聊天/对话 API
- 知识问答 API 的请求流程
- 流式响应（SSE）的实现方式
- Agent 对话 API
- 消息管理 API

### 4. 中间件链
- 中间件的执行顺序
- 认证中间件的工作流程
- OIDC 认证流程
- 错误处理和 panic 恢复机制
- 请求日志和链路追踪
- 多语言支持

### 5. IM 即时通讯集成
- IM 集成架构（用 Mermaid 图）
- Adapter 适配器模式设计
- 各平台（企业微信、飞书、Slack、Telegram、钉钉、Mattermost）的集成方式
- 斜杠命令系统（/help, /info, /search, /stop, /clear）
- 问答队列和限速机制
- IM 消息引用/回复上下文处理
- 线程（Thread）会话模式

### 6. 流式传输
- SSE 流式传输的实现
- 流管理器设计
- 流的生命周期

### 7. 事件系统
- 事件总线设计
- 事件类型和订阅者

### 8. Go SDK
- SDK 的功能覆盖范围
- 使用示例
