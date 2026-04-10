# Chatwoot 部署与开发环境完全指南

## 目录
1. [系统架构](#1-系统架构)
2. [基础设施依赖](#2-基础设施依赖)
3. [本地开发环境搭建](#3-本地开发环境搭建)
4. [生产部署方案](#4-生产部署方案)
5. [进程架构](#5-进程架构)
6. [监控与运维](#6-监控与运维)

---

## 1. 系统架构

### 1.1 架构概览

Chatwoot 采取**完全分离的多进程架构**，支持水平扩展：

```
┌─────────────────────────────────────────────────────────┐
│                        Nginx 反向代理                     │
│            (SSL、负载均衡、静态资源缓存)                 │
└────────────────┬────────────────────────────────────────┘
                 │
    ┌────────────┴──────────────┐
    │                           │
┌───▼────────────┐    ┌────────▼────────┐
│   Rails App    │    │  Vite Dev Server │
│ (Puma Server)  │    │  (前端资源)      │
│  (Web 进程)    │    │  :3036         │
│   :3000        │    └──────────────────┘
└───┬────────────┘
    │ (communicates via Redis)
    │
    ├──────────────────────────┬─────────────────────────┐
    │                          │                         │
┌───▼────────────┐  ┌──────────▼──────────┐  ┌──────────▼──────┐
│   PostgreSQL   │  │      Redis         │  │  Sidekiq Worker  │
│   (DB, pgvector│  │  缓存/消息队列/    │  │   (后台任务)    │
│    扩展)       │  │   实时连接缓存      │  └──────────────────┘
└────────────────┘  └───────────────────┘
                           │
                           └─► ActionCable (WebSocket)
```

### 1.2 进程角色划分

| 进程 | 端口 | 职责 | 开发 | 生产 |
|------|------|------|------|------|
| Rails (Puma) | 3000 | 主应用服务器、API 端点、Web UI | ✓ | ✓ |
| Sidekiq Worker | - | 异步任务处理、邮件、消息 | ✓ | ✓ |
| Vite Dev Server | 3036 | 前端资源热更新（开发专用） | ✓ | - |
| Mailhog | 1025 | 邮件测试服务（开发专用） | ✓ | - |

### 1.3 部署形态

```
┌─────────────────────────────────────────┐
│       Chatwoot 部署形态选择             │
├─────────────────────────────────────────┤
│ 1. Docker Compose（推荐）               │
│    - 开发: docker-compose.yaml          │
│    - 生产: docker-compose.production.yml│
│    - 测试: docker-compose.test.yml      │
│                                         │
│ 2. 直接部署（Ubuntu/Debian）            │
│    - systemd 服务管理                   │
│    - Nginx 反向代理                     │
│                                         │
│ 3. Heroku / DigitalOcean                │
│    - 开箱即用（需要 Procfile）         │
│                                         │
│ 4. Kubernetes（企业级）                 │
│    - 需要自行编写 Helm Chart            │
└─────────────────────────────────────────┘
```

---

## 2. 基础设施依赖

### 2.1 PostgreSQL

**版本要求与功能**

| 项目 | 规格 |
|------|------|
| 版本 | PostgreSQL 16（含 pgvector 扩展） |
| 编码 | UTF-8 |
| 连接池 | 根据并发数动态计算 |
| 超时 | 14 秒或自定义 |

**数据库配置**

```yaml
# config/database.yml

default: &default
  adapter: postgresql
  encoding: unicode
  pool: 5  # 开发环境
         # Sidekiq Workers 时: 10
         # Rails Max Threads: $RAILS_MAX_THREADS
  reaping_frequency: 30  # 10 秒回收僵死连接

development:
  database: chatwoot_dev
  username: postgres
  password: ''

test:
  database: chatwoot_test
  username: postgres  
  password: ''

production:
  database: chatwoot_production
  username: chatwoot_prod
  password: $POSTGRES_PASSWORD  # 必须设置强密码
```

**环境变量配置**

```bash
# 数据库连接
POSTGRES_HOST=postgres              # 数据库服务器地址
POSTGRES_PORT=5432                 # 端口
POSTGRES_USERNAME=postgres         # 用户名
POSTGRES_PASSWORD=                 # 密码（生产必须）
POSTGRES_DATABASE=chatwoot_prod   # 数据库名（可选，默认）

# 性能优化
POSTGRES_STATEMENT_TIMEOUT=14s     # 查询超时
RAILS_MAX_THREADS=5                # 应用线程数
```

**pgvector 扩展**（用于 AI 功能）

```sql
-- 自动在首次迁移时创建
CREATE EXTENSION IF NOT EXISTS pgvector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;  -- 文本搜索
```

**多数据库配置（按account分库）**

Chatwoot 支持为不同 Account 配置不同的数据库（Enterprise 功能）。

### 2.2 Redis

**用途与指标**

| 用途 | 说明 |
|------|------|
| 缓存 | Session、业务数据缓存 |
| 消息队列 | Sidekiq 任务队列 |
| ActionCable | WebSocket 实时连接状态 |
| 计数器 | 速限、限流统计 |

**配置方式**

```bash
# 方式 1: URL 方式（推荐）
REDIS_URL=redis://:password@host:port/database

# 方式 2: 单个变量（旧格式，不推荐）
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_NAMESPACE=chatwoot_prod
REDIS_PASSWORD=                    # 可选

# 高可用：Redis Sentinel
REDIS_SENTINELS=sentinel1:26379,sentinel2:26379
REDIS_SENTINEL_MASTER_NAME=mymaster
REDIS_SENTINEL_PASSWORD=           # 哨兵密码（可选）
```

**ActionCable 配置**

```yaml
# config/cable.yml

production:
  adapter: redis
  url: <%= ENV.fetch('REDIS_URL') %>
  password: <%= ENV.fetch('REDIS_PASSWORD').presence %>
  channel_prefix: chatwoot_production_action_cable
```

**Docker Compose 中的 Redis**

```yaml
redis:
  image: redis:alpine
  restart: always
  command: ["sh", "-c", "redis-server --requirepass \"$REDIS_PASSWORD\""]
  ports:
    - '127.0.0.1:6379:6379'
  volumes:
    - redis_data:/data
```

### 2.3 文件存储

**本地存储**

```yaml
# config/storage.yml

# 开发/测试
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

# 生产本地存储
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
```

环境变量：
```bash
ACTIVE_STORAGE_SERVICE=local  # 或 amazon/google/microsoft/s3_compatible
```

**AWS S3 / S3 兼容服务**

```yaml
amazon:
  service: S3
  access_key_id: <%= ENV.fetch('AWS_ACCESS_KEY_ID') %>
  secret_access_key: <%= ENV.fetch('AWS_SECRET_ACCESS_KEY') %>
  region: <%= ENV.fetch('AWS_REGION') %>
  bucket: <%= ENV.fetch('S3_BUCKET_NAME') %>

# DigitalOcean Spaces / MinIO 等
s3_compatible:
  service: S3
  access_key_id: <%= ENV.fetch('STORAGE_ACCESS_KEY_ID') %>
  secret_access_key: <%= ENV.fetch('STORAGE_SECRET_ACCESS_KEY') %>
  region: <%= ENV.fetch('STORAGE_REGION') %>
  bucket: <%= ENV.fetch('STORAGE_BUCKET_NAME') %>
  endpoint: <%= ENV.fetch('STORAGE_ENDPOINT') %>
  force_path_style: true
```

环境变量：
```bash
# 选择存储服务
ACTIVE_STORAGE_SERVICE=amazon

# AWS S3
S3_BUCKET_NAME=chatwoot-prod
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=us-east-1

# 直接上传配置（允许浏览器上传文件到 S3）
DIRECT_UPLOADS_ENABLED=true
```

**Google Cloud Storage**

```yaml
google:
  service: GCS
  project: <%= ENV.fetch('GCS_PROJECT') %>
  credentials: <%= ENV.fetch('GCS_CREDENTIALS') %>  # JSON 凭证
  bucket: <%= ENV.fetch('GCS_BUCKET') %>
```

**Azure Blob Storage**

```yaml
microsoft:
  service: AzureStorage
  storage_account_name: <%= ENV.fetch('AZURE_STORAGE_ACCOUNT_NAME') %>
  storage_access_key: <%= ENV.fetch('AZURE_STORAGE_ACCESS_KEY') %>
  container: <%= ENV.fetch('AZURE_STORAGE_CONTAINER') %>
```

### 2.4 邮件服务

**SMTP 配置**

```bash
# SMTP 服务器
SMTP_ADDRESS=smtp.gmail.com           # SMTP 地址
SMTP_PORT=587                        # SMTP 端口
SMTP_DOMAIN=example.com              # HELO 域名
SMTP_USERNAME=notifications@example.com
SMTP_PASSWORD=your_app_password      # Gmail 应用密码

# 身份验证方式
SMTP_AUTHENTICATION=plain             # plain/login/cram_md5
SMTP_ENABLE_STARTTLS_AUTO=true       # TLS 自动升级
SMTP_OPENSSL_VERIFY_MODE=peer        # 证书验证

# 发件人地址
MAILER_SENDER_EMAIL=Chatwoot <notifications@example.com>
```

**邮件地址转发（Email Continuitiy）**

用于通过邮件回复保持会话连续性：

```bash
# 邮件转发配置
MAILER_INBOUND_EMAIL_DOMAIN=reply.example.com

# 选择邮件服务提供商
RAILS_INBOUND_EMAIL_SERVICE=ses  # ses/mailgun/postmark/sendgrid/mandrill

# 不同服务的凭证
MAILGUN_INGRESS_SIGNING_KEY=...
ACTION_MAILBOX_SES_SNS_TOPIC=arn:aws:sns:...
RAILS_INBOUND_EMAIL_PASSWORD=webhook_password
```

**Webhook URL 示例**

```
https://actionmailbox:${RAILS_INBOUND_EMAIL_PASSWORD}@chat.example.com/rails/action_mailbox/postmark/inbound_emails
```

**开发环境邮件**

```bash
# Docker Compose 中的 Mailhog（测试邮件服务）
SMTP_ADDRESS=mailhog
SMTP_PORT=1025

# 访问 http://localhost:1025 查看测试邮件
```

---

## 3. 本地开发环境搭建

### 3.1 前置条件

**系统要求**

| 组件 | 版本 | 安装方式 |
|------|------|---------|
| Ruby | 3.4.4 | `rbenv install 3.4.4` |
| Node | 24.13.0 | `nvm install 24.13.0` 或 `nodenv` |
| Git | 任意 | 系统包管理器 |
| Bundler | 2.5.16 | `gem install bundler -v 2.5.16` |
| pnpm | 10.2.0 | `npm install -g pnpm@10.2.0` |
| PostgreSQL | 16+ | Docker 或本地安装 |
| Redis | 最新 | Docker 或本地安装 |

**macOS 仓库检查**

```bash
cat .ruby-version    # 应输出 3.4.4
cat .nvmrc          # 应输出 24.13.0
```

### 3.2 方式一：直接安装（Native 开发）

**Step 1: 克隆仓库并进入目录**

```bash
git clone https://github.com/chatwoot/chatwoot.git
cd chatwoot

# 配置 rbenv（必须！）
eval "$(rbenv init -)"

# 验证 Ruby 版本
ruby --version  # 应为 3.4.4
```

**Step 2: 安装 Ruby 依赖**

```bash
# 安装 Bundler（如果未安装）
gem install bundler -v 2.5.16

# 安装 Gem 依赖
bundle install

# 验证：应该出现成功消息
bundle info chatwoot
```

**Step 3: 配置 Node.js**

```bash
# 使用 nvm（如果之前使用过）
nvm use  # 自动读取 .nvmrc

# 使用 nodenv
nodenv install 24.13.0

# 验证
node --version  # v24.13.0
pnpm --version   # 10.2.0
```

**Step 4: 安装前端依赖**

```bash
pnpm install           # 安装 npm 依赖
pnpm install -g pnpm  # 确保全局有 pnpm
```

**Step 5: 配置数据库**

```bash
# 创建 .env.local 或复制模板
cp .env.example .env.local

# 编辑 .env.local 中的数据库凭证
# 如果使用本地 PostgreSQL：
POSTGRES_HOST=localhost
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=your_password
```

```bash
# 创建数据库并运行迁移
bundle exec rails db:create
bundle exec rails db:migrate

# 添加种子数据（可选）
bundle exec rails db:seed
```

**Step 6: 启动开发服务器**

```bash
# 方式 1：使用 Procfile.dev（推荐，一次启动所有服务）
pnpm dev

# 或
overmind start -f Procfile.dev

# 方式 2：分别启动（需要多个终端）
# 终端 1: Rails 服务器
rails server -p 3000

# 终端 2: Vite Dev Server
bin/vite dev

# 终端 3: Sidekiq Worker
bundle exec sidekiq -C config/sidekiq.yml
```

**验证开发环境**

```bash
# Rails 应用
curl http://localhost:3000   # 应返回 HTML

# Sidekiq 仪表板
curl http://localhost:3000/admin/sidekiq

# WebSocket 连接
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  http://localhost:3000/cable
```

### 3.3 方式二：Docker Compose 开发环境（推荐）

**优点**

- 无需本地安装 PostgreSQL、Redis
- 完全隔离的开发环境
- 跨平台一致性

**快速启动**

```bash
# 创建 .env 文件（可复制 .env.example）
cp .env.example .env

# 启动所有服务
docker-compose up

# 后台运行
docker-compose up -d
```

**Docker Compose 架构**

```yaml
# docker-compose.yaml（开发环境）

services:
  rails:
    build: ./docker/Dockerfile
    ports:
      - '3000:3000'  # Rails
    depends_on:
      - postgres
      - redis
      - vite
      - sidekiq
    volumes:
      - ./:/app:delegated        # 代码热加载
      - bundle:/usr/local/bundle
    environment:
      - RAILS_ENV=development
      - NODE_ENV=development
      - VITE_DEV_SERVER_HOST=vite  # 关键：连接 Vite 容器

  sidekiq:
    build: ./docker/Dockerfile
    depends_on:
      - postgres
      - redis
    volumes:
      - ./:/app:delegated
    environment:
      - RAILS_ENV=development
    command: bundle exec sidekiq -C config/sidekiq.yml

  vite:
    build: ./docker/dockerfiles/vite.Dockerfile
    ports:
      - '3036:3036'  # Vite Dev Server
    volumes:
      - ./:/app:delegated
    environment:
      - VITE_DEV_SERVER_HOST=0.0.0.0  # 绑定所有接口

  postgres:
    image: pgvector/pgvector:pg16
    ports:
      - '5432:5432'
    volumes:
      - postgres:/data/postgres
    environment:
      POSTGRES_DB: chatwoot
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ''

  redis:
    image: redis:alpine
    ports:
      - '6379:6379'
    volumes:
      - redis:/data/redis

  mailhog:
    image: mailhog/mailhog
    ports:
      - '1025:1025'  # SMTP
      - '8025:8025'  # Web UI
```

**常用命令**

```bash
# 查看日志
docker-compose logs -f rails

# 在容器中运行命令
docker-compose exec rails bundle exec rails console

# 重启服务
docker-compose restart sidekiq

# 完全重建
docker-compose down -v
docker-compose up --build
```

**容器内数据库操作**

```bash
# 创建数据库
docker-compose exec rails bundle exec rails db:create

# 运行迁移
docker-compose exec rails bundle exec rails db:migrate

# 运行种子数据
docker-compose exec rails bundle exec rails db:seed

# 数据库控制台
docker-compose exec postgres psql -U postgres -d chatwoot
```

### 3.4 Dev Container 配置（VS Code）

```json
// .devcontainer/devcontainer.json

{
  "name": "Chatwoot Dev",
  "image": "mcr.microsoft.com/devcontainers/ruby:3.4.4",
  "features": {
    "ghcr.io/devcontainers/features/node:latest": {
      "version": "24.13.0"
    },
    "ghcr.io/devcontainers/features/postgres:latest": {}
  },
  "remoteEnv": {
    "RAILS_ENV": "development",
    "POSTGRES_HOST": "localhost"
  },
  "postCreateCommand": "bundle install && pnpm install",
  "forwardPorts": [3000, 3036, 5432, 6379]
}
```

在 VS Code 中：
1. 安装 "Dev Containers" 扩展
2. Cmd/Ctrl + Shift + P → "Dev Containers: Reopen in Container"

### 3.5 开发服务器启动说明

**Procfile.dev 详解**

```
backend: bin/rails s -p 3000
worker: dotenv bundle exec sidekiq -C config/sidekiq.yml
vite: bin/vite dev
```

| 进程 | 命令 | 端口 | 用途 |
|------|------|------|------|
| backend | `rails s -p 3000` | 3000 | Rails 应用服务器 |
| worker | `sidekiq` | - | 后台任务（邮件、消息推送等） |
| vite | `bin/vite dev` | 3036 | 前端热更新开发服务器 |

**启动方式对比**

```bash
# 使用 Procfile.dev 的进程编排工具
pnpm dev                    # 使用 foreman（简单）
overmind start -f Procfile.dev  # 使用 overmind（推荐，支持多个实例）

# 手动启动（演示用）
# 终端 1
bin/rails s -p 3000

# 终端 2
bin/vite dev

# 终端 3
bundle exec sidekiq -C config/sidekiq.yml
```

### 3.6 种子数据与测试数据

**基础种子数据**

```bash
# 快速填充：最小数据集（推荐开发调试）
bundle exec rails db:seed

# 搜索/性能测试数据集（大量数据）
bundle exec rails search:setup_test_data

# 账户富数据（通过 UI 或 CLI）
# UI 路径: Super Admin → Accounts → Seed
# CLI 路径:
bundle exec rails runner "Internal::SeedAccountJob.perform_now(Account.find(1))"

# 或直接调用:
bundle exec rails runner "Seeders::AccountSeeder.new(account: Account.find(1)).perform!"
```

**测试数据库初始化**

```bash
# 在 test 环境中初始化
RAILS_ENV=test bundle exec rails db:create db:migrate

# 运行测试（会自动创建测试数据）
bundle exec rspec
```

### 3.7 常用开发命令

```bash
# 代码格式和检查
pnpm eslint                 # 检查 JS/Vue 代码
pnpm eslint:fix            # 自动修复 ESLint 问题
bundle exec rubocop -a     # 自动修复 Ruby 风格

# 测试
pnpm test                  # 运行 Node.js 测试（Vitest）
pnpm test:watch           # 监视模式
bundle exec rspec spec/models/account_spec.rb  # 运行单个 Ruby 测试文件
bundle exec rspec spec/models/account_spec.rb:42  # 运行指定行号的测试

# 数据库操作
bundle exec rails db:reset          # 重置数据库
bundle exec rails g migration AddColumnToUsers  # 生成迁移
bundle exec rails db:rollback       # 回滚一个迁移

# 控制台
rails console               # Ruby 交互式 REPL
rails console --sandbox    # 测试用（不保存更改）

# 日志查看
tail -f log/development.log     # 查看 Rails 日志
tail -f log/sidekiq.log        # 查看 Worker 日志

# i18n 同步（本地翻译）
bin/sync_i18n_file_change

# 性能分析
QUERY_LOG=true bin/rails s     # 启用查询日志

# 清理缓存
bundle exec rails tmp:clear
bundle exec rails cache:clear
```

---

## 4. 生产部署方案

### 4.1 Docker Compose 生产部署

**使用 docker-compose.production.yaml**

```yaml
version: '3'

services:
  rails:
    image: chatwoot/chatwoot:latest
    depends_on:
      - postgres
      - redis
    ports:
      - '127.0.0.1:3000:3000'  # 仅本地访问，由 Nginx 反向代理
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    command: ['bundle', 'exec', 'rails', 's', '-p', '3000', '-b', '0.0.0.0']
    restart: always

  sidekiq:
    image: chatwoot/chatwoot:latest
    depends_on:
      - postgres
      - redis
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
    command: ['bundle', 'exec', 'sidekiq', '-C', 'config/sidekiq.yml']
    restart: always

  postgres:
    image: pgvector/pgvector:pg16
    restart: always
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=chatwoot
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}  # 环境变量注入
    ports:
      - '127.0.0.1:5432:5432'

  redis:
    image: redis:alpine
    restart: always
    volumes:
      - redis_data:/data
    command: ["sh", "-c", "redis-server --requirepass \"$REDIS_PASSWORD\""]
    ports:
      - '127.0.0.1:6379:6379'

volumes:
  postgres_data:
  redis_data:
  storage_data:
```

**部署步骤**

```bash
# 1. 克隆仓库
git clone https://github.com/chatwoot/chatwoot.git
cd chatwoot

# 2. 创建 .env 文件（生产配置）
cat > .env << EOF
# 核心配置
SECRET_KEY_BASE=$(rake secret)
FRONTEND_URL=https://chat.example.com
RAILS_ENV=production
NODE_ENV=production

# 数据库
POSTGRES_PASSWORD=strong_random_password_here
POSTGRES_HOST=postgres

# Redis
REDIS_PASSWORD=another_strong_password

# 邮件
SMTP_ADDRESS=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=notifications@example.com
SMTP_PASSWORD=app_password
MAILER_SENDER_EMAIL=Chatwoot <notifications@example.com>

# 文件存储
ACTIVE_STORAGE_SERVICE=amazon
S3_BUCKET_NAME=chatwoot-prod
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=us-east-1

# 其他
ENABLE_ACCOUNT_SIGNUP=false  # 关闭公开注册
FORCE_SSL=true               # 强制 HTTPS
EOF

# 3. 构建并启动容器
docker-compose -f docker-compose.production.yaml up -d

# 4. 初始化数据库（首次部署）
docker-compose -f docker-compose.production.yaml exec rails \
  bundle exec rails db:create db:migrate

# 5. 验证
docker-compose -f docker-compose.production.yaml ps
docker-compose -f docker-compose.production.yaml logs -f rails
```

### 4.2 Linux 服务器直接部署（Ubuntu 20.04+）

**前置条件检查**

```bash
# 运行自动化部署脚本
sudo bash deployment/setup_20.04.sh --install

# 或手动步骤：
sudo apt update
sudo apt install -y \
  build-essential \
  git \
  curl \
  postgresql postgresql-contrib \
  redis-server \
  nginx \
  ruby ruby-dev bundler \
  nodejs npm
```

**手动部署步骤**

**Step 1: 创建系统用户**

```bash
sudo useradd -m -s /bin/bash chatwoot
sudo -u chatwoot -i bash  # 切换到 chatwoot 用户
```

**Step 2: 克隆并配置应用**

```bash
# 在 chatwoot 用户下执行
cd ~
git clone https://github.com/chatwoot/chatwoot.git chatwoot
cd chatwoot

# Ruby 版本管理（使用 RVM）
curl -sSL https://get.rvm.io | bash -s stable
source ~/.rvm/scripts/rvm
rvm install 3.4.4
rvm use 3.4.4 --default

# Node.js
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
npm install -g pnpm

# 安装 Ruby & Node 依赖
bundle install
pnpm install
```

**Step 3: 配置环境变量**

```bash
# 复制模板并编辑
cp .env.example .env

# 编辑 .env（使用 nano 或 vim）
nano .env

# 关键变量：
# SECRET_KEY_BASE=...（生成: rake secret）
# RAILS_ENV=production
# FRONTEND_URL=https://chat.example.com
# POSTGRES_PASSWORD=...
# REDIS_PASSWORD=...
```

**Step 4: 初始化数据库**

```bash
# 创建 PostgreSQL 用户和数据库
sudo -u postgres psql << EOF
CREATE USER chatwoot_prod WITH PASSWORD 'strong_password';
CREATE DATABASE chatwoot_production OWNER chatwoot_prod;
ALTER DATABASE chatwoot_production SET timezone TO 'UTC';
EOF

# 运行迁移
RAILS_ENV=production bundle exec rails db:create db:migrate
```

**Step 5: 配置 Systemd 服务**

```bash
sudo cp deployment/chatwoot-web.1.service /etc/systemd/system/chatwoot-web.service
sudo cp deployment/chatwoot-worker.1.service /etc/systemd/system/chatwoot-worker.service

# 编辑服务文件，更新用户和路径
sudo nano /etc/systemd/system/chatwoot-web.service
sudo nano /etc/systemd/system/chatwoot-worker.service

# 启用并启动服务
sudo systemctl daemon-reload
sudo systemctl enable chatwoot-web chatwoot-worker
sudo systemctl start chatwoot-web chatwoot-worker

# 查看状态
sudo systemctl status chatwoot-web
sudo systemctl status chatwoot-worker
```

**Example Systemd Service 文件**

```ini
# /etc/systemd/system/chatwoot-web.service

[Unit]
Requires=network.target
PartOf=chatwoot.target

[Service]
Type=simple
User=chatwoot
WorkingDirectory=/home/chatwoot/chatwoot

ExecStart=/bin/bash -lc 'bin/rails server -p $PORT -e $RAILS_ENV'

Restart=always
RestartSec=1
TimeoutStopSec=30
KillMode=mixed
SyslogIdentifier=%p
LimitNOFILE=65536

Environment="PORT=3000"
Environment="RAILS_ENV=production"
Environment="NODE_ENV=production"
Environment="RAILS_LOG_TO_STDOUT=true"

[Install]
WantedBy=multi-user.target
```

**Step 6: 配置 Nginx 反向代理**

```bash
sudo cp deployment/nginx_chatwoot.conf /etc/nginx/sites-available/chatwoot

# 创建符号链接
sudo ln -s /etc/nginx/sites-available/chatwoot /etc/nginx/sites-enabled/

# 编辑配置，替换域名和 SSL 路径
sudo nano /etc/nginx/sites-available/chatwoot

# 测试配置
sudo nginx -t

# 重启 Nginx
sudo systemctl restart nginx
```

**Nginx 配置示例**

```nginx
upstream chatwoot_backend {
  server 127.0.0.1:3000;
  keepalive 32;
}

server {
  listen 80;
  server_name chat.example.com;
  return 301 https://$server_name$request_uri;  # HTTP → HTTPS 重定向
}

server {
  listen 443 ssl http2;
  server_name chat.example.com;

  # SSL 证书（Let's Encrypt）
  ssl_certificate /etc/letsencrypt/live/chat.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/chat.example.com/privkey.pem;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;

  access_log /var/log/nginx/chatwoot_access.log;
  error_log /var/log/nginx/chatwoot_error.log;

  client_max_body_size 100M;

  location / {
    proxy_pass http://chatwoot_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_read_timeout 36000s;
  }
}
```

**Step 7: SSL 证书配置（Let's Encrypt）**

```bash
sudo apt install -y certbot python3-certbot-nginx

# 自动配置 SSL
sudo certbot --nginx -d chat.example.com

# 自动续期
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

### 4.3 Heroku 一键部署

**前置条件**

```bash
# 安装 Heroku CLI
brew install heroku

# 登录
heroku login
```

**部署步骤**

```bash
# 创建 Heroku 应用
heroku create my-chatwoot-app

# 添加 PostgreSQL 和 Redis 插件
heroku addons:create heroku-postgresql:standard-0
heroku addons:create heroku-redis:premium-0

# 设置环境变量
heroku config:set SECRET_KEY_BASE=$(bundle exec rake secret)
heroku config:set FRONTEND_URL=https://my-chatwoot-app.herokuapp.com
heroku config:set RAILS_ENV=production
heroku config:set FORCE_SSL=true

# 部署
git push heroku main

# 初始化数据库
heroku run bundle exec rails db:migrate

# 查看日志
heroku logs -f
```

**关键：Procfile 配置**

```
release: bundle exec rails db:chatwoot_prepare && echo $SOURCE_VERSION > .git_sha
web: bundle exec rails server -p $PORT -e $RAILS_ENV
worker: bundle exec sidekiq -C config/sidekiq.yml
```

### 4.4 环境变量配置清单

**必需变量（生产)**

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `SECRET_KEY_BASE` | Django/Rails 加密密钥 | `$(rake secret)` |
| `RAILS_ENV` | 运行环境 | `production` |
| `FRONTEND_URL` | 应用访问地址 | `https://chat.example.com` |
| `POSTGRES_PASSWORD` | 数据库密码 | `secure_password_here` |
| `POSTGRES_HOST` | 数据库主机 | `postgres.example.com` 或 `localhost` |
| `REDIS_PASSWORD` | Redis 密码 | `redis_password_here` |

**邮件配置**

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `SMTP_ADDRESS` | SMTP 服务器 | `smtp.gmail.com` |
| `SMTP_PORT` | SMTP 端口 | `587` 或 `465` |
| `SMTP_USERNAME` | SMTP 用户名 | `notifications@example.com` |
| `SMTP_PASSWORD` | SMTP 密码 / App 密码 | `app_password_from_provider` |
| `MAILER_SENDER_EMAIL` | 发件人地址 | `Chatwoot <notifications@example.com>` |

**文件存储配置**

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `ACTIVE_STORAGE_SERVICE` | 存储服务 | `amazon` / `google` / `microsoft` / `local` |
| `S3_BUCKET_NAME` | S3 桶名称 | `chatwoot-production` |
| `AWS_ACCESS_KEY_ID` | AWS 访问密钥 | `AKIA...` |
| `AWS_SECRET_ACCESS_KEY` | AWS 秘密密钥 | `...` |
| `AWS_REGION` | AWS 地区 | `us-east-1` |

**安全性配置**

| 变量 | 说明 | 推荐值 |
|------|------|--------|
| `FORCE_SSL` | 强制 HTTPS | `true` |
| `ENABLE_ACCOUNT_SIGNUP` | 允许公开注册 | `false` |
| `ENABLE_RACK_ATTACK` | 限流保护 | `true` |
| `RACK_ATTACK_LIMIT` | 请求限制（/min） | `300` |

**监控和日志**

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `RAILS_LOG_TO_STDOUT` | 日志输出到 stdout | `true` |
| `LOG_LEVEL` | 日志级别 | `info` |
| `SENTRY_DSN` | Sentry 错误追踪 | `https://key@sentry.io/project` |
| `NEW_RELIC_LICENSE_KEY` | New Relic APM | `license_key` |

**频道/集成配置**

| 变量 | 用途 | 获取方式 |
|------|------|---------|
| `FB_APP_ID` / `FB_APP_SECRET` | Facebook Messenger | Facebook Apps 控制台 |
| `SLACK_CLIENT_ID` / `SLACK_CLIENT_SECRET` | Slack 集成 | Slack API 管理面板 |
| `GOOGLE_OAUTH_CLIENT_ID` | Google OAuth | Google Cloud Console |
| `TWITTER_CONSUMER_KEY` / `TWITTER_CONSUMER_SECRET` | Twitter 频道 | Twitter Developer Portal |

---

## 5. 进程架构

### 5.1 Web 进程（Puma）

**Puma 配置**

```ruby
# config/puma.rb

# 线程处理并发请求
max_threads_count = ENV.fetch('RAILS_MAX_THREADS', 5)
min_threads_count = ENV.fetch('RAILS_MIN_THREADS', max_threads_count)
threads min_threads_count, max_threads_count

# 端口
port ENV.fetch('PORT', 3000)

# 环境
environment ENV.fetch('RAILS_ENV', 'development')

# 进程数（多进程模式）
workers ENV.fetch('WEB_CONCURRENCY', 0)
preload_app!  # Copy-on-Write

# PID 文件
pidfile ENV.fetch('PIDFILE', 'tmp/pids/server.pid')
```

**性能参数**

| 参数 | 开发 | 生产 | 说明 |
|------|------|------|------|
| 最大线程数 | 5 | 10-20 | 每个进程的线程 |
| 最小线程数 | 等于最大 | 1-5 | 预热线程数 |
| 工作进程数 | 0 | CPU 核心数 × 2 | 多进程负载均衡 |

**容量规划**

```
单实例（4 CPU, 8GB RAM）:
  - 线程: 5 最大
  - 进程: 8 个（4 × 2）
  - 吞吐量: ~500 req/sec
  - 支持用户: 1000-2000 CCU

生产推荐（16 CPU, 32GB RAM）:
  - 多个 Rails 进程（Docker 容器）
  - Redis Sentinel 高可用
  - PostgreSQL 读写分离
  - CDN 加速静态资源
```

### 5.2 Worker 进程（Sidekiq）

**Sidekiq 配置**

```yaml
# config/sidekiq.yml

---
:verbose: false
:concurrency: <%= ENV.fetch("SIDEKIQ_CONCURRENCY", 10) %>
:timeout: 25
:max_retries: 3

# 优先级队列（高优先级任务优先处理）
:queues:
  - critical      # 立即处理（系统任务）
  - high         # 优先处理（重要用户操作）
  - medium       # 正常处理（常规操作）
  - default      # 默认队列
  - mailers      # 邮件发送
  - action_mailbox_routing  # 邮件路由
  - low          # 低优先级（批量操作）
  - scheduled_jobs  # 定时任务
  - deferred     # 延迟任务
  - purgable     # 可清理的（日志、清理）
  - housekeeping # 数据库清理
  - async_database_migration  # 异步数据库迁移
  - bulk_reindex_low  # 搜索索引（低优先级）
  - active_storage_analysis   # 文件分析
  - active_storage_purge      # 文件清理

production:
  :concurrency: 10  # 生产环境通常 10-20
```

**任务类型示例**

```ruby
# app/jobs/send_email_job.rb
class SendEmailJob < ApplicationJob
  queue_as :mailers
  sidekiq_options retry: 5

  def perform(user_id)
    user = User.find(user_id)
    UserMailer.notification_email(user).deliver_later
  end
end

# 调用
SendEmailJob.perform_later(user.id)  # 立即加入队列，异步执行
SendEmailJob.set(wait: 1.hour).perform_later(user.id)  # 延迟执行

# 定时任务
class HousekeepingJob < ApplicationJob
  queue_as :housekeeping
  sidekiq_options retry: 1

  def perform
    # 清理过期数据
  end
end
```

**监控 Sidekiq**

```bash
# 访问 Sidekiq 仪表板
http://localhost:3000/admin/sidekiq

# 或在容器中
docker-compose exec rails rails sidekiq:monitor

# CLI 查询队列状态
bundle exec sidekiq -C config/sidekiq.yml  # 启动 worker
# 在另一个终端
rails console
>> Sidekiq::Queue.new('default').size  # 队列中的任务数
>> Sidekiq::Workers.new.size            # 活跃工作进程
```

### 5.3 实时通信（ActionCable）

**WebSocket 架构**

```yaml
# config/cable.yml

production:
  adapter: redis
  url: <%= ENV.fetch('REDIS_URL') %>
  channel_prefix: chatwoot_production_action_cable
```

**关键功能**

```ruby
# app/channels/chat_channel.rb
module ChatChannel
  class ConversationChannel < ApplicationCable::Channel
    def subscribed
      stream_from "conversation_#{params[:conversation_id]}"
    end

    def receive(data)
      # 实时消息广播
      ActionCable.server.broadcast(
        "conversation_#{params[:conversation_id]}",
        data
      )
    end

    def unsubscribed
      # 清理连接
    end
  end
end
```

**性能考虑**

| 指标 | 单 Redis | Redis Cluster |
|------|----------|----------------|
| 最大连接数 | 10K | 100K+ |
| 消息延迟 | <100ms | <50ms |
| 网络吞吐 | 1M conn/sec | 10M conn/sec |

### 5.4 Sidekiq 队列设计

**根据优先级分层**

```
┌─────────────────────────────────────────┐
│ 优先级 1（Critical）                    │
│ 系统任务、支付、认证等                   │
│ 立即处理，最多 3 次重试                 │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 优先级 2（High）                        │
│ 用户操作、消息、通知                    │
│ 2-3 秒内处理，5 次重试                 │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 优先级 3（Default）                    │
│ 常规操作、报告                         │
│ <1 分钟内处理，3 次重试                │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 优先级 4（Low）                        │
│ 批量操作、索引、清理                    │
│ 可跳过高峰期，1 次重试                 │
└─────────────────────────────────────────┘
```

**并发配置**

```bash
# 根据服务器资源调整
# 4CPU 服务器
SIDEKIQ_CONCURRENCY=10

# 8CPU 服务器
SIDEKIQ_CONCURRENCY=20

# 16CPU 服务器（推荐 2+ 实例）
SIDEKIQ_CONCURRENCY=20  # 每个实例 20 * 2 = 40 并发
```

---

## 6. 监控与运维

### 6.1 健康检查端点

```bash
# Rails 应用健康检查
curl -i http://localhost:3000/health

# Sidekiq 健康检查
curl -i http://localhost:3000/admin/sidekiq/live

# 数据库连接检查
curl -i http://localhost:3000/health/db

# 缓存检查
curl -i http://localhost:3000/health/cache
```

### 6.2 日志与调试

**Rails 日志**

```bash
# 开发日志
tail -f log/development.log

# 生产日志（到 stdout）
docker-compose logs -f rails

# 根据级别过滤
grep ERROR log/production.log
grep -i "exception\|error" log/production.log | tail -100
```

**Sidekiq 日志**

```bash
# Worker 日志
tail -f log/sidekiq.log

# 查看进行中的任务
bundle exec sidekiq -C config/sidekiq.yml -v

# 失败的任务
docker-compose exec rails rails console
>> Sidekiq::DeadSet.new.each { |job| p job }
```

**数据库查询日志**

```bash
# 启用查询日志
QUERY_LOG=true bin/rails s

# 或在 console 中
>> ActiveRecord::Base.logger = Logger.new(STDOUT)
```

### 6.3 性能监控工具

**内置 Sidekiq 仪表板**

访问 `http://localhost:3000/admin/sidekiq`

- 活跃任务、失败任务统计
- 队列深度监控
- 重试管理
- 慢任务检测

**New Relic APM（可选）**

```bash
# 配置
NEW_RELIC_LICENSE_KEY=your_license_key
NEW_RELIC_APP_NAME="Chatwoot Production"

# 访问仪表板监控：
# - 响应时间分布
# - 数据库查询时间
# - 错误率
```

**Scout APM（可选）**

```bash
# 配置
SCOUT_KEY=your_scout_key
SCOUT_NAME="Chatwoot Prod"
SCOUT_MONITOR=true

# 轻量级 APM，性能开销小
```

**Datadog / 监控集成**

```bash
# Datadog APM
DD_TRACE_AGENT_URL=http://localhost:8126
DD_SERVICE=chatwoot
DD_ENV=production

# Elastic APM
ELASTIC_APM_SERVER_URL=http://localhost:8200
ELASTIC_APM_SERVICE_NAME=chatwoot
```

### 6.4 告警和SLA 指标

**关键指标**

| 指标 | 告警阈值 | 处理方式 |
|------|---------|---------|
| Rails 响应时间 P95 | >1000ms | 扩容或优化查询 |
| Sidekiq 队列深度 | >10000 | 增加 Worker 并发 |
| 数据库连接池饱和 | >90% | 增加 pool 大小或扩容 |
| 错误率 | >0.5% | 查看日志、回滚变更 |
| 磁盘使用率 | >85% | 清理日志、迁移数据 |
| Redis 内存使用 | >80% | 检查泄漏、调整驱逐策略 |

**自动恢复脚本**

```bash
#!/bin/bash
# auto-recovery.sh

# 检查 Rails 进程健康
if ! curl -s http://localhost:3000/health > /dev/null; then
  echo "Rails health check failed, restarting..."
  systemctl restart chatwoot-web
fi

# 检查 Sidekiq
if ! curl -s http://localhost:3000/admin/sidekiq/live > /dev/null; then
  echo "Sidekiq check failed, restarting..."
  systemctl restart chatwoot-worker
fi

# 检查 Redis
if ! redis-cli ping > /dev/null; then
  echo "Redis is down, restarting..."
  systemctl restart redis-server
fi

# 检查磁盘空间
DISK_USAGE=$(df / | awk 'NR==2 {print $5}' | cut -d'%' -f1)
if [ $DISK_USAGE -gt 85 ]; then
  echo "Disk usage is critical: $DISK_USAGE%, rotating logs..."
  logrotate -f /etc/logrotate.conf
fi
```

### 6.5 备份与灾难恢复

**数据库备份**

```bash
# PostgreSQL 全量备份
pg_dump -h localhost -U postgres chatwoot_production > backup_$(date +%Y%m%d_%H%M%S).sql

# Docker 中备份
docker-compose exec postgres pg_dump -U postgres chatwoot > backup.sql

# 恢复备份
psql -h localhost -U postgres chatwoot_production < backup.sql

# 自动化备份脚本
#!/bin/bash
BACKUP_DIR="/backups/chatwoot"
RETENTION_DAYS=30

mkdir -p $BACKUP_DIR

# 备份
pg_dump -h localhost -U postgres chatwoot_production | \
  gzip > $BACKUP_DIR/chatwoot_prod_$(date +%Y%m%d_%H%M%S).sql.gz

# 清理旧备份
find $BACKUP_DIR -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

# 上传到 S3（可选）
aws s3 sync $BACKUP_DIR s3://my-backups/chatwoot/ --delete
```

**Redis 快照备份**

```bash
# RDB 快照备份
BGSAVE  # 后台保存

# 恢复
# 将 dump.rdb 复制到 Redis 数据目录，重启服务
```

**定期恢复测试**

```bash
# 建议每个月实行一次恢复演练
# 1. 在测试环境恢复备份
# 2. 验证数据完整性
# 3. 记录恢复时间（RTO）
```

---

## 附录：快速参考

### 本地开发快速启动

```bash
# 方式 1：Docker（推荐新手）
docker-compose up

# 方式 2：Native（推荐开发者）
eval "$(rbenv init -)"       # 初始化 rbenv
bundle install
pnpm install
bundle exec rails db:setup
pnpm dev
```

### 生产部署快速启动

```bash
# Docker Compose
cp .env.example .env
# 编辑 .env 中的生产变量
docker-compose -f docker-compose.production.yaml up -d
docker-compose -f docker-compose.production.yaml exec rails bundle exec rails db:migrate

# Linux 直接部署
sudo bash deployment/setup_20.04.sh --install
```

### 常见问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| 503 Service Unavailable | Rails 未启动 | `pnpm dev` 或检查日志 |
| WebSocket 连接失败 | Redis 未连接 | 检查 REDIS_URL 配置 |
| 邮件未发送 | SMTP 配置错误 | 检查 SMTP_ADDRESS、端口、认证 |
| 文件上传失败 | S3 权限或凭证错误 | 检查 AWS 密钥、bucket 策略 |
| 内存泄漏 | Sidekiq Worker 内存增长 | 增加 worker 池、减少超时 |

---

**文档版本**: Chatwoot v25.01（2026年4月）
**最后更新**: 2026-04-10
**维护者**: Chatwoot 社区
