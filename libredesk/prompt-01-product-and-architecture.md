# Prompt 01: 产品概览与架构设计分析

## 任务

请分析 libredesk 项目的源代码，输出一份**产品概览与架构设计文档**。

## 需要分析的文件

按以下顺序阅读：

1. `README.md` — 产品简介
2. `ROADMAP.md` — 路线图
3. `config.sample.toml` — 配置结构（反映系统组件）
4. `docker-compose.yml` / `Dockerfile` — 部署架构
5. `Makefile` — 构建流程
6. `schema.sql` — 数据模型（重点关注 ENUM 类型定义和表结构）
7. `cmd/init.go` — 应用初始化，所有模块如何组装
8. `cmd/main.go` — 入口和启动流程
9. `cmd/handlers.go` — HTTP 路由注册，反映 API 设计
10. `cmd/middlewares.go` — 中间件设计
11. `internal/` 目录结构 — 内部模块划分

## 输出要求

写入文件 `/Users/wenkai/workspace/knowledges/libredesk/01-product-and-architecture.md`，包含以下章节：

### 1. 产品定位
- 是什么产品，解决什么问题
- 核心功能列表（从 README 和代码中提取）
- 当前版本状态和路线图

### 2. 技术栈
- 后端语言、框架、关键依赖库
- 前端语言、框架、UI 库
- 数据库、缓存、消息队列
- 构建工具链

### 3. 系统架构图（文字描述）
- 整体架构：单体 Go 应用 + PostgreSQL + Redis
- 进程内组件划分（从 `cmd/init.go` 的初始化顺序推断）
- 组件间通信方式（HTTP, WebSocket, Redis pub/sub, 内部方法调用等）
- 外部集成点（邮件服务、AI 服务、Webhook、OAuth/OIDC）

### 4. 数据模型概览
- 核心实体关系：conversation, message, user(agent/contact), inbox, team, role
- 关键 ENUM 类型及其含义
- 重要的关联关系

### 5. API 设计概览
- 从 `cmd/handlers.go` 提取 API 路由分组和命名模式
- 认证/鉴权机制（从 middlewares 推断）
- WebSocket 端点用途

### 6. 配置体系
- 配置加载机制（文件 + 环境变量）
- 关键配置分组及用途

### 7. 构建与部署模型
- 单二进制打包机制（stuffbin）
- Docker 部署模式
- 开发模式运行方式（前后端分离开发）
