# Prompt 2: Chatwoot 后端架构分析

## 任务
深入分析 Chatwoot 的 Rails 后端架构设计，输出一份详尽的后端架构文档。

## 输出要求
将分析结果写入 `/Users/wenkai/workspace/knowledges/chatwoot/output/02-backend-architecture.md`，使用中文撰写。

## 需要分析的文件和目录

### 应用配置与启动
- `config/application.rb` — Rails 应用配置
- `config/routes.rb` — **完整阅读**，这是理解 API 设计的核心文件
- `config/database.yml` — 数据库配置
- `config/sidekiq.yml` — 后台任务配置
- `config/cable.yml` — WebSocket/ActionCable 配置
- `config/puma.rb` — Web 服务器配置
- `config/environments/production.rb` — 生产环境配置

### 控制器层（重点分析结构和模式）
- `app/controllers/api/v1/accounts/` 目录 — 浏览所有文件名，选读 3-5 个典型控制器（如 conversations_controller, messages_controller, inboxes_controller）理解 API 设计模式
- `app/controllers/api/v2/` 目录 — 了解 v2 API 的范围
- `app/controllers/platform/` 目录 — Platform API（跨账户操作）
- `app/controllers/public/` 目录 — 公开 API（Widget 等）
- `app/controllers/webhooks/` 目录 — Webhook 接收器
- `app/controllers/concerns/` 目录 — 共享 Controller Concern

### 服务层（重点分析模式）
- `app/services/` 目录结构 — 理解服务的组织方式
- 选读以下关键服务：
  - `app/services/conversations/` — 会话相关操作
  - `app/services/messages/` — 消息处理
  - `app/services/auto_assignment/` — 自动分配逻辑
  - `app/services/filter_service.rb` — 过滤/查询
  - `app/services/search_service.rb` — 搜索

### 后台任务
- `app/jobs/` 目录 — 浏览所有 Job 文件，理解异步任务类型

### 事件系统
- `app/listeners/` 目录 — 事件监听器
- `app/dispatchers/` 目录 — 事件分发器
- `app/builders/` 目录 — Builder 模式

### 中间件与认证
- `app/controllers/application_controller.rb`
- `app/controllers/api_controller.rb`
- `config/initializers/` 目录 — 快速浏览所有初始化器文件名，选读关键的（devise, rack_attack, cors 等）

### 关键 Model Concern
- `app/models/concerns/` 目录 — 浏览文件名，选读与核心逻辑相关的 concern（如消息处理、自动化相关的）

### Enterprise 后端扩展
- `enterprise/app/controllers/` — 了解 Enterprise 如何扩展 API
- `enterprise/app/services/` — Enterprise 专有服务
- `enterprise/lib/` — 了解 Enterprise 模块加载机制

## 输出文档结构要求

```markdown
# Chatwoot 后端架构

## 1. 总体架构
（Rails 应用的整体结构，各层职责，请求处理流程）

## 2. API 设计
### 2.1 API 版本与分层
（v1 Account API, v2 API, Platform API, Public API, Widget API 的定位和区别）
### 2.2 认证机制
（Token 认证、API Key、用户会话等）
### 2.3 路由结构
（用树形结构展示主要路由命名空间和资源）
### 2.4 API 设计模式
（RESTful 约定、分页、过滤、错误处理）

## 3. 数据层
### 3.1 数据库设计概要
（核心表关系，多租户隔离策略）
### 3.2 关键 Model 设计模式
（Concern 使用、回调模式、验证策略）

## 4. 服务层
### 4.1 服务组织方式
### 4.2 典型服务调用链
（以一次消息发送为例，从 Controller 到 Service 到 Model 的完整链路）

## 5. 异步任务与事件系统
### 5.1 Sidekiq 任务体系
### 5.2 事件监听与分发机制
### 5.3 WebSocket/ActionCable 实时通信

## 6. Enterprise 扩展机制
（Enterprise 如何通过 prepend/include 扩展 OSS 功能）

## 7. 关键技术决策
（多租户方案、搜索方案、文件存储方案等）
```
