# Prompt 5: Chatwoot 部署架构与本地开发环境

## 任务
分析 Chatwoot 的部署架构和本地开发环境搭建，输出部署与开发环境文档。

## 输出要求
将分析结果写入 `/Users/wenkai/workspace/knowledges/chatwoot/output/05-deployment-devops.md`，使用中文撰写。

## 需要分析的文件和目录

### Docker 部署
- `docker-compose.yaml` — 开发环境 Docker Compose
- `docker-compose.production.yaml` — 生产环境 Docker Compose
- `docker-compose.test.yaml` — 测试环境
- `docker/` 目录 — Dockerfile 和 entrypoint 脚本
- `docker/dockerfiles/` — 如果存在，各个 Dockerfile

### 本地开发
- `Procfile.dev` — 开发进程管理
- `Procfile` — 生产进程
- `Procfile.test` — 测试进程
- `.env.example` — **完整阅读**，环境变量配置参考
- `.ruby-version` — Ruby 版本
- `.nvmrc` — Node 版本
- `Gemfile` — 前 50 行和与数据库/Redis/Sidekiq 相关的部分
- `package.json` — scripts 部分
- `bin/` 目录 — 启动脚本
- `.devcontainer/` — Dev Container 配置（如果存在）

### 系统部署
- `deployment/` 目录 — systemd 服务文件、Nginx 配置
- `Capfile` — Capistrano 部署配置（如果有实际内容）

### 基础设施依赖
- `config/database.yml` — PostgreSQL 配置
- `config/sidekiq.yml` — Sidekiq 队列定义
- `config/cable.yml` — ActionCable（WebSocket）配置
- `config/puma.rb` — Puma 服务器配置
- `config/storage.yml` — 文件存储配置（本地/S3）
- `config/schedule.yml` — 定时任务（如果有）

### 外部依赖与集成
- `.env.example` 中的各服务配置段：
  - 数据库 (PostgreSQL)
  - 缓存/消息队列 (Redis)
  - 邮件 (SMTP)
  - 文件存储 (S3/GCS)
  - 推送通知 (Firebase)
  - 各渠道 API Key 配置

## 输出文档结构要求

```markdown
# Chatwoot 部署与开发环境

## 1. 系统架构图
（组件关系：Rails App / Sidekiq Worker / PostgreSQL / Redis / Vite Dev Server / Nginx）

## 2. 基础设施依赖
### 2.1 PostgreSQL
（版本要求、扩展依赖如 pgvector/pg_trgm、多数据库配置）
### 2.2 Redis
（用途：缓存/Sidekiq/ActionCable、配置方式）
### 2.3 文件存储
（本地存储 vs S3/GCS，配置方式）
### 2.4 邮件服务
（SMTP 配置，Mailhog 开发环境）

## 3. 本地开发环境搭建
### 3.1 前置条件
（Ruby 版本、Node 版本、系统依赖）
### 3.2 方式一：直接安装（Native）
（step-by-step 指南）
### 3.3 方式二：Docker Compose 开发环境
（step-by-step 指南）
### 3.4 方式三：Dev Container
### 3.5 开发服务器启动
（Procfile.dev 各进程说明）
### 3.6 种子数据与测试数据
### 3.7 常用开发命令

## 4. 生产部署方案
### 4.1 Docker Compose 部署
（基于 docker-compose.production.yaml 的完整部署流程）
### 4.2 Linux 服务器直接部署
（systemd 服务配置、Nginx 反向代理）
### 4.3 Heroku / DigitalOcean 一键部署
### 4.4 环境变量配置清单
（按类别列出所有关键环境变量及说明）

## 5. 进程架构
### 5.1 Web 进程（Puma）
### 5.2 Worker 进程（Sidekiq）
### 5.3 实时通信（ActionCable）
### 5.4 Sidekiq 队列设计
（列出所有队列及其优先级）

## 6. 监控与运维
（健康检查端点、日志、APM 集成）
```
