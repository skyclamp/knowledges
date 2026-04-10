# Prompt 06: 本地开发与部署指南

## 任务

请分析 libredesk 项目的开发和部署相关配置，输出一份**本地开发环境搭建与部署指南**。

## 需要分析的文件

1. `Makefile` — 构建和开发命令
2. `config.sample.toml` — 配置项说明
3. `docker-compose.yml` — Docker 部署
4. `Dockerfile` — 镜像构建
5. `frontend/package.json` — 前端依赖和脚本
6. `frontend/vite.config.js` — Vite 配置（特别是 proxy 设置）
7. `frontend/.env` — 前端环境变量
8. `go.mod` — Go 依赖
9. `CONTRIBUTING.md` — 贡献指南
10. `cmd/main.go` — 命令行参数
11. `cmd/install.go` — 安装逻辑
12. `cmd/upgrade.go` — 升级逻辑
13. `schema.sql` — 数据库初始化
14. `internal/migrations/` — 数据库迁移
15. `.goreleaser.yaml` — 发布配置

## 输出要求

写入文件 `/Users/wenkai/workspace/knowledges/libredesk/06-local-dev-and-deployment.md`，包含：

### 1. 系统要求
- Go 版本
- Node.js / pnpm 版本
- PostgreSQL 版本
- Redis 版本
- 其他工具（stuffbin 等）

### 2. 本地开发环境搭建步骤
完整的 step-by-step 指南：
1. 克隆仓库
2. 安装依赖
3. 准备 PostgreSQL 和 Redis（可用 docker-compose 部分服务）
4. 配置 config.toml
5. 初始化数据库
6. 运行后端开发服务器
7. 运行前端开发服务器
8. 访问应用

### 3. 开发工作流
- 前后端分离开发模式（前端 dev server proxy 到后端）
- 代码修改后如何生效（热重载情况）
- 运行测试
- 数据库迁移方式

### 4. 构建流程
- `make build` 的完整流程
- stuffbin 打包机制说明
- 前端构建产物如何嵌入 Go 二进制

### 5. 部署方案

#### Docker 部署
- docker-compose 完整配置解析
- 环境变量配置
- 数据持久化
- 首次安装 vs 升级

#### 二进制部署
- 下载和配置
- 数据库准备
- 命令行参数说明（--install, --upgrade, --set-system-user-password 等）
- systemd 服务配置建议

### 6. 关键配置项说明
- 必须修改的配置（encryption_key, db, redis）
- 安全相关配置（secure_cookies, CORS）
- 性能调优配置（worker 数量、队列大小）
- 文件存储配置（fs vs s3）
