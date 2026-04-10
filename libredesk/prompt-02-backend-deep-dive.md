# Prompt 02: 后端核心模块深度分析

## 任务

请深入分析 libredesk 后端 `internal/` 下的核心模块，输出一份**后端模块详细设计文档**。

## 需要分析的文件和目录

重点分析以下模块（每个模块关注：models 子目录的数据结构、queries.sql 的 SQL 操作、主文件的业务逻辑）：

### 核心业务模块
1. `internal/conversation/` — 会话管理（含 message, draft, priority, status 子模块）
2. `internal/inbox/` — 收件箱和渠道管理（含 channel/ 子目录）
3. `internal/user/` — 用户管理（agent 和 contact 两种类型）
4. `internal/envelope/` — 消息信封（消息传递的核心抽象）

### 协作与组织模块
5. `internal/team/` — 团队管理
6. `internal/role/` — 角色和权限
7. `internal/authz/` — 授权逻辑
8. `internal/auth/` — 认证
9. `internal/tag/` — 标签系统

### 智能化模块
10. `internal/automation/` — 自动化规则引擎
11. `internal/autoassigner/` — 自动分配
12. `internal/ai/` — AI 集成
13. `internal/sla/` — SLA 管理

### 集成模块
14. `internal/webhook/` — Webhook 推送
15. `internal/notification/` — 通知系统
16. `internal/ws/` — WebSocket 实时通信
17. `internal/media/` — 文件上传存储

### 辅助模块
18. `internal/search/` — 搜索
19. `internal/template/` — 模板
20. `internal/csat/` — 客户满意度
21. `internal/macro/` — 宏（快捷操作）
22. `internal/view/` — 自定义视图
23. `internal/setting/` — 系统设置

## 输出要求

写入文件 `/Users/wenkai/workspace/knowledges/libredesk/02-backend-deep-dive.md`，包含以下内容：

### 对每个模块，分析：

1. **职责** — 一句话说明做什么
2. **数据模型** — models/ 下定义的 struct 和关键字段
3. **核心接口/方法** — 公开的关键函数签名和作用
4. **SQL 查询模式** — queries.sql 中的关键操作
5. **与其他模块的依赖关系**

### 重点深入分析：

#### 消息处理流水线
- 从 `cmd/messages.go` 和 `cmd/conversation.go` 入手
- 跟踪一条消息从「接收」到「处理」到「发送」的完整路径
- 入站消息和出站消息的处理差异
- `internal/envelope/` 的 Envelope 结构设计

#### Channel 抽象层
- `internal/inbox/channel/` 的接口设计
- 当前只有 email 实现，分析接口抽象程度
- 添加新渠道需要实现哪些接口

#### Automation 引擎
- 规则定义结构（conditions + actions）
- 评估器(evaluator)的工作方式
- 支持的条件类型和动作类型

#### AI 模块
- 当前的 AI 能力范围
- Provider 抽象层设计
- 调用方式和使用场景
