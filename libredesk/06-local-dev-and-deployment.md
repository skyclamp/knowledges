# Libredesk 本地开发环境搭建与部署指南

**最后更新**：2026年4月

---

## 目录
- [系统要求](#系统要求)
- [本地开发环境搭建](#本地开发环境搭建)
- [开发工作流](#开发工作流)
- [构建流程](#构建流程)
- [部署方案](#部署方案)
- [关键配置项说明](#关键配置项说明)
- [常见问题](#常见问题)

---

## 系统要求

### 开发环境

| 工具/依赖 | 版本 | 说明 |
|----------|------|------|
| **Go** | 1.25.0+ | 后端编译 |
| **Node.js** | 18.0.0+（推荐 20.x）| 前端工具链 |
| **pnpm** | 最新版本 | 包管理器（推荐 v9.x+） |
| **PostgreSQL** | 17.x | 主数据库 |
| **Redis** | 7.x+ | 缓存和队列存储 |
| **stuffbin** | $GOPATH/bin/stuffbin | 资源嵌入工具，由 make 自动安装 |
| **Docker** & **Docker Compose** | 最新版本 | 本地开发快速启动（可选）|

### 系统要求

- **操作系统**：Linux、macOS、Windows WSL2
- **磁盘空间**：最少 5GB
- **网络**：需要互联网连接以下载依赖

### 本地环境

- **PostgreSQL**：需要本地运行或通过 Docker 运行
- **Redis**：需要本地运行或通过 Docker 运行
- **Go 工作目录**：`$HOME/go` (或自定义 `$GOPATH`)

---

## 本地开发环境搭建

### 第1步：克隆仓库

```bash
git clone https://github.com/abhinavxd/libredesk.git
cd libredesk
```

### 第2步：检查系统依赖

```bash
# 检查 Go 版本
go version   # 应显示 go1.25.0 或更高

# 检查 Node.js 版本
node --version
npm install -g pnpm  # 全局安装 pnpm
pnpm --version
```

### 第3步：准备数据库和 Redis

#### 选项 A：使用 Docker Compose（推荐快速开发）

```bash
# 启动 PostgreSQL 和 Redis（不启动应用容器）
docker-compose up -d db redis

# 验证服务启动
docker-compose ps
```

此时 PostgreSQL 和 Redis 会在以下地址运行：
- **PostgreSQL**：`localhost:5432`
- **Redis**：`localhost:6379`

#### 选项 B：本地安装

如使用已有的本地 PostgreSQL 和 Redis，可跳过 Docker 步骤。

### 第4步：配置应用程序

复制配置示例文件：

```bash
cp config.sample.toml config.toml
```

编辑 `config.toml`，修改以下关键配置：

```toml
[app]
log_level = "debug"
env = "dev"
# 生成加密密钥：openssl rand -hex 16
encryption_key = "生成的32字符密钥"

[app.server]
address = "0.0.0.0:9000"
disable_secure_cookies = true  # 开发环境可设为 true

[db]
host = "localhost"      # 如使用 Docker，使用 "db"
port = 5432
user = "libredesk"
password = "libredesk"
database = "libredesk"
ssl_mode = "disable"

[redis]
address = "localhost:6379"  # 如使用 Docker，使用 "redis:6379"
```

### 第5步：初始化数据库

首次运行需要初始化数据库架构和默认数据：

```bash
# 安装数据库schema并创建系统用户
# 该命令是幂等的，可安全重复运行
go run cmd/*.go --install --idempotent-install --yes

# 如指定系统用户密码
LIBREDESK_SYSTEM_USER_PASSWORD="your-secure-password" \
  go run cmd/*.go --install --idempotent-install --yes
```

**默认系统用户**：
- **用户名**：`system`
- **密码**：命令行中指定或交互式输入

### 第6步：安装工程依赖

```bash
# 后端依赖已在 go.mod 中定义，go run 时自动下载

# 前端依赖
cd frontend
pnpm install
cd ..
```

### 第7步：运行开发服务器

#### 运行后端服务器

在一个终端运行：

```bash
make run-backend
```

**输出示例**：
```
→ Running backend...
[2026-04-10 10:30:45.123] api|info Server started on 0.0.0.0:9000
```

后端将监听 `http://0.0.0.0:9000`

#### 运行前端开发服务器

在另一个终端运行：

```bash
make run-frontend
```

**输出示例**：
```
→ Running frontend...
  ➜  Local:   http://localhost:8000/
```

前端开发服务器将在 `http://localhost:8000` 运行。

### 第8步：访问应用程序

打开浏览器访问：

```
http://localhost:8000
```

登录凭证：
- **用户名**：`system`
- **密码**：安装时设定的密码

---

## 开发工作流

### 前后端通信

前端开发服务器通过 Vite 代理与后端通信：

**Vite 配置** (`frontend/vite.config.js`)：
```javascript
server: {
  port: 8000,
  proxy: {
    '/api': {
      target: 'http://127.0.0.1:9000',
    },
    '/logout': {
      target: 'http://127.0.0.1:9000',
    },
    '/uploads': {
      target: 'http://127.0.0.1:9000',
    },
    '/ws': {
      target: 'ws://127.0.0.1:9000',
      ws: true,  // WebSocket 代理
    },
  },
},
```

前端请求 `/api/*` 会被自动转发到后端的 `http://127.0.0.1:9000/api/*`。

### 代码修改和热重载

#### 后端热重载

后端使用 `go run` 执行（不使用编译后的二进制），修改后自动重新加载：

```bash
# 修改任何 cmd/*.go 或 internal/*.go 文件后，
# 后端会自动重启（如配置了 file watcher）
# 或手动重启 make run-backend
```

#### 前端热重载

前端使用 Vite 提供热重载支持，修改 Vue 组件、样式或脚本后自动更新：

- 修改 `.vue` 文件：页面自动更新
- 修改 CSS/Tailwind：样式立即生效
- 修改 JavaScript：热替换模块（HMR）

### 数据库迁移

Libredesk 的数据库迁移系统位于 `internal/migrations/`：

**迁移文件结构**：
```
internal/migrations/
├── v0.3.0.go
├── v0.4.0.go
├── ...
├── v1.0.1.go
└── migrations.go
```

**迁移执行流程**：

1. **自动升级**：升级应用版本时自动运行
   ```bash
   go run cmd/*.go --upgrade --yes
   ```

2. **手动迁移**：开发过程中可运行特定版本迁移
   ```bash
   # 在 upgrade 函数中指定版本执行
   ```

**如需创建新迁移**：

1. 在 `internal/migrations/` 下创建 `v{version}.go` 文件
2. 实现迁移函数（必须非常幂等）
3. 在 `cmd/upgrade.go` 的 `migList` 中添加新迁移

```go
var migList = []migFunc{
    // ... 现有迁移 ...
    {"v1.1.0", migrations.V1_1_0},  // 新迁移
}
```

### 运行测试

```bash
# 运行后端单元测试
make test

# 或单独运行
go test -count=1 ./...

# 前端测试（如有）
cd frontend
pnpm test:run
```

---

## 构建流程

### 构建命令

#### 完整构建（前后端 + 资源嵌入）

```bash
make build
```

**构建步骤**：
1. **frontend-build**：编译 Vue.js 应用到 `frontend/dist/`
2. **build-backend**：编译 Go 二进制（带完整链接信息）
3. **stuff**：使用 stuffbin 将静态资源嵌入二进制

**输出**：`./libredesk` 二进制文件

#### 分步构建

```bash
# 仅构建前端
make frontend-build

# 仅构建后端
make build-backend

# 仅嵌入资源（requires build-backend first）
make stuff
```

### stuffbin 机制解析

**stuffbin** 是一个资源嵌入工具，将以下文件打包进二进制：

```
STATIC := ${FRONTEND_DIST} i18n schema.sql static
```

- `frontend/dist/`：编译后的前端文件
- `i18n/`：国际化语言文件
- `schema.sql`：数据库 schema
- `static/`：静态文件（邮件模板等）

**Makefile 中的配置**：

```makefile
stuff: $(STUFFBIN)
	@echo "→ Stuffing static assets into binary..."
	@$(STUFFBIN) -a stuff -in ${BIN} -out ${BIN} ${STATIC}
```

### 前端构建产物集成

前端构建过程：

```bash
# 前端开发
cd frontend
pnpm install
pnpm build  # 输出到 frontend/dist/
```

**构建产物结构**：
```
frontend/dist/
├── index.html      # 主入口
├── assets/
│   ├── index-*.js
│   ├── index-*.css
│   └── ...
└── locales/        # 翻译文件（如有）
```

构建完成后，整个 `frontend/dist/` 被 stuffbin 嵌入到二进制中。

**运行时**：应用启动时从 Go 的嵌入式文件系统提供这些文件。

### 版本和构建字符串

在构建时，以下版本信息被注入：

```bash
# Makefile 中定义
BUILDSTR := ${VERSION} (\#${LAST_COMMIT} $(shell date -u +"%Y-%m-%dT%H:%M:%S%z"))
VERSION := $(git describe --tags) || 版本文件值 || v0.0.0

# 链接选项
-X 'main.buildString=${BUILDSTR}'
-X 'main.versionString=${VERSION}'
```

从 `/version` API 端点或应用界面可查看构建信息。

---

## 部署方案

### Docker 部署

#### 快速启动（使用 docker-compose）

```bash
# 复制配置文件
cp config.sample.toml config.toml
# 编辑 config.toml 中的必要项

# 启动所有服务
docker-compose up -d

# 查看日志
docker-compose logs -f app
```

#### docker-compose.yml 解析

```yaml
services:
  app:
    image: libredesk/libredesk:latest
    ports:
      - "9000:9000"
    environment:
      # 首次启动时自动设置系统用户密码
      LIBREDESK_SYSTEM_USER_PASSWORD: ""
    depends_on:
      - db
      - redis
    volumes:
      - ./uploads:/libredesk/uploads:rw   # 持久化上传文件
      - ./config.toml:/libredesk/config.toml  # 挂载配置
    command:
      - sh
      - -c
      - |
        ./libredesk --install --idempotent-install --yes --config /libredesk/config.toml
        ./libredesk --upgrade --yes --config /libredesk/config.toml
        ./libredesk --config /libredesk/config.toml

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: libredesk
      POSTGRES_PASSWORD: libredesk
      POSTGRES_DB: libredesk
    ports:
      - "127.0.0.1:5432:5432"  # 仅本机访问
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "127.0.0.1:6379:6379"  # 仅本机访问
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

#### 数据持久化

**重要**：确保 `uploads` 目录权限正确

```bash
# 创建上传目录
mkdir -p uploads
chmod 755 uploads

# 重要：确保容器内能写入此目录
docker-compose up -d
```

#### 首次安装

首次启动时，应用会：

1. **检查数据库**：确认 schema 是否存在
2. **创建 schema**：如不存在则运行 `schema.sql`
3. **创建系统用户**：使用 `LIBREDESK_SYSTEM_USER_PASSWORD` 环境变量

```bash
# 带密码启动
LIBREDESK_SYSTEM_USER_PASSWORD="YourStrongPassword@123" \
  docker-compose up -d
```

#### 升级现有实例

```bash
# 拉取新镜像
docker-compose pull

# 重启应用（自动运行迁移）
docker-compose up -d

# 查看日志确认升级成功
docker-compose logs app | head -20
```

### 二进制部署

#### 下载和配置

1. **获取二进制**：从 GitHub Releases 下载或本地编译

```bash
# 从 releases 下载
# 或本地编译
make build
```

2. **创建部署目录**

```bash
mkdir -p /opt/libredesk
cd /opt/libredesk
cp /path/to/libredesk .
cp /path/to/config.toml .
mkdir -p uploads
```

3. **准备数据库**

```bash
# 确保 PostgreSQL 已运行
# 创建用户和数据库（如需要）
createuser libredesk
createdb -O libredesk libredesk
```

4. **编辑配置文件**

```toml
[db]
host = "localhost"
user = "libredesk"
password = "your-password"
database = "libredesk"

[redis]
address = "localhost:6379"

[app.server]
address = "0.0.0.0:9000"
```

#### 命令行参数说明

| 参数 | 说明 | 示例 |
|-----|------|------|
| `--config` | 指定配置文件路径 | `--config /etc/libredesk/config.toml` |
| `--install` | 初始化数据库 schema | `--install` |
| `--idempotent-install` | 安全的幂等安装（已存在则跳过） | `--idempotent-install` |
| `--upgrade` | 运行数据库迁移 | `--upgrade` |
| `--yes` | 跳过确认提示 | 配合 `--install` 使用 |
| `--set-system-user-password` | 交互式设置系统用户密码 | `--set-system-user-password` |

#### 首次部署流程

```bash
# 1. 初始化数据库
./libredesk --install --idempotent-install --yes --config config.toml

# 输出示例：
# installing database schema...
# database schema installed successfully
# created system user

# 2. 运行应用
./libredesk --config config.toml
```

#### 升级现有部署

```bash
# 1. 停止应用
systemctl stop libredesk  # 如使用 systemd

# 2. 备份数据库
pg_dump -U libredesk libredesk > backup-$(date +%s).sql

# 3. 替换二进制
cp ./libredesk-new /opt/libredesk/libredesk
chmod +x /opt/libredesk/libredesk

# 4. 运行迁移
cd /opt/libredesk
./libredesk --upgrade --yes --config config.toml

# 5. 重启应用
systemctl start libredesk
```

#### 系统用户密码管理

设置系统用户密码：

```bash
# 交互式设置
./libredesk --set-system-user-password --config config.toml

# 或在 Docker 中
docker exec -it libredesk_app ./libredesk --set-system-user-password
```

#### systemd 服务配置

创建 `/etc/systemd/system/libredesk.service`：

```ini
[Unit]
Description=Libredesk Customer Support Desk
After=network.target postgresql.service redis.service

[Service]
Type=simple
User=libredesk
Group=libredesk
WorkingDirectory=/opt/libredesk
ExecStart=/opt/libredesk/libredesk --config /etc/libredesk/config.toml
Restart=always
RestartSec=5s

# 日志
StandardOutput=journal
StandardError=journal
SyslogIdentifier=libredesk

[Install]
WantedBy=multi-user.target
```

启用并启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable libredesk
sudo systemctl start libredesk

# 查看状态
sudo systemctl status libredesk

# 查看日志
sudo journalctl -u libredesk -f
```

---

## 关键配置项说明

### 必须修改的配置

#### 1. 加密密钥（encryption_key）

**用途**：用于加密敏感数据（如 API 密钥、OAuth token）

**必须修改**：是

**生成方式**：
```bash
openssl rand -hex 16
# 输出示例：a7f3e9c2d1b4f8a6e5c3d2b1a9f7e4c6
```

**配置**：
```toml
[app]
encryption_key = "生成的32字符密钥"  # 必须正好32个字符（16字节的16进制表示）
```

**危险**：修改后无法解密已有数据！应在生产环境前生成并固定不变。

#### 2. 数据库连接（[db]）

```toml
[db]
host = "localhost"     # 必改：参考实际 DB 地址
port = 5432
user = "libredesk"     # 必改：参考实际用户名
password = "libredesk" # 必改：参考实际密码
database = "libredesk"
ssl_mode = "disable"   # 生产环境建议改为 "require"

# 连接池配置
max_open = 30          # 最大开放连接数
max_idle = 30          # 空闲连接数
max_lifetime = "300s"  # 连接最大生命周期
```

#### 3. Redis 连接（[redis]）

```toml
[redis]
address = "localhost:6379"  # 必改：参考实际地址
# 或使用 URL 格式（覆盖其他配置）
url = "redis://user:password@host:port/db"

user = ""       # 如需认证
password = ""   # 如需认证
db = 0          # 数据库编号
```

### 安全相关配置

#### 1. 安全 Cookie（disable_secure_cookies）

```toml
[app.server]
disable_secure_cookies = false  # 必须为 false（生产环境）
```

- `false`（默认）：仅 HTTPS 传输 Cookie，防止中间人攻击
- `true`：开发环境或测试用（不安全）

#### 2. CORS 和跨域请求

确保前端和后端的 CORS 配置一致。检查 `cmd/middlewares.go` 中的 CORS 处理。

**常见设置**：

```go
// 允许的源（需在代码中配置）
allowedOrigins := []string{"http://localhost:8000"}  // 开发环境
```

#### 3. SSL/TLS（生产环境）

配置 Nginx 或其他反向代理处理 SSL/TLS：

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    location / {
        proxy_pass http://localhost:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 性能调优配置

#### 消息队列（[message]）

```toml
[message]
# 优化为根据服务器资源调整
outgoing_queue_workers = 10           # 出站消息处理线程数
incoming_queue_workers = 10           # 入站消息处理线程数
message_outgoing_scan_interval = "50ms"
incoming_queue_size = 5000            # 最大入站队列
outgoing_queue_size = 5000            # 最大出站队列
```

**调优建议**：
- 高消息量：增加 `outgoing_queue_workers`（如 20-30）
- Redis 内存压力：减少 `*_queue_size`
- 低延迟需求：减少 `message_outgoing_scan_interval`（如 10ms）

#### 通知（[notification]）

```toml
[notification]
concurrency = 2          # 并发处理数
queue_size = 2000        # 最大通知队列
```

#### 自动化和 Webhook

```toml
[automation]
worker_count = 10        # 自动化规则执行线程

[webhook]
workers = 5              # 并发 webhook 投递数
queue_size = 10000       # webhook 队列大小
timeout = "15s"          # webhook 请求超时
```

### 文件存储配置

#### 本地文件系统存储（默认）

```toml
[upload]
provider = "fs"

[upload.fs]
upload_path = 'uploads'  # 相对或绝对路径
```

**必须确保**：
- 目录存在且可写
- 足够磁盘空间（建议定期清理）
- 定期备份

```bash
# 备份上传文件
tar -czf uploads-backup.tar.gz uploads/
```

#### S3 存储（云存储）

```toml
[upload]
provider = "s3"

[upload.s3]
url = ""                 # 仅 MinIO 等非 AWS 需要
access_key = "AWS_KEY"
secret_key = "AWS_SECRET"
region = "ap-south-1"
bucket = "my-bucket"
bucket_path = "uploads"  # 可选前缀
expiry = "30m"          # 预签名 URL 有效期
```

**优势**：
- 无限存储空间
- 自动备份
- CDN 加速

**缺点**：
- 需要 AWS 账户和配置
- API 调用成本

### 电子邮件和通知

**确认配置的文件**：`config.sample.toml` 中的邮件相关部分（如有）

```toml
# 示例：如有邮件配置
[service.smtp]
host = "smtp.gmail.com"
port = 587
username = "your-email@gmail.com"
password = "app-password"
```

### 日志和调试

```toml
[app]
log_level = "debug"      # 开发环境：debug, 生产环境：info
env = "dev"              # dev, prod（影响日志格式和输出）
check_updates = true     # 是否检查更新
```

---

## 常见问题

### Q1：如何更换加密密钥？

**A**：加密密钥通常不能更换，因为已加密的数据无法解密。建议：

1. 在初始化前生成和设置
2. 如必须更换：需要手动迁移（导出-解密-重加密-导入）

### Q2：如何批量导入现有数据？

**A**：查看 `internal/importer/` 模块（如有实现）。支持从其他系统迁移。

### Q3：前端 build 时出现内存不足？

**A**：
```bash
# 增加 Node 内存限制
NODE_OPTIONS="--max-old-space-size=4096" pnpm build
```

### Q4：后端编译出现 CGO 错误？

**A**：Libredesk 已配置 `CGO_ENABLED=0`，不需要 C 编译器。如仍出错：

```bash
# 显式禁用 CGO
CGO_ENABLED=0 go build -o libredesk cmd/*.go
```

### Q5：如何启用 HTTPS 开发？

**A**：使用 mkcert 生成本地证书：

```bash
mkcert localhost 127.0.0.1
# 然后配置 Nginx 或其他反向代理
```

### Q6：Docker 容器无法连接到主机 PostgreSQL？

**A**：修改 `config.toml` 中的 DB 地址：

```toml
[db]
# 不能使用 localhost，改用
host = "host.docker.internal"  # macOS/Windows WSL2
# 或
host = "172.17.0.1"             # Linux（容器网关）
```

### Q7：如何查看应用版本和构建信息？

**A**：
```bash
# 从二进制
./libredesk --version

# 或从 API
curl http://localhost:9000/api/version
```

### Q8：如何启用调试日志？

**A**：
```toml
[app]
log_level = "debug"
```

或运行时环境变量：
```bash
LOG_LEVEL=debug ./libredesk
```

---

## 总结

Libredesk 提供了灵活的开发和部署选项：

- **快速开发**：使用 Docker Compose + make 命令
- **本地开发**：准备 PostgreSQL/Redis + 修改配置 + make 命令
- **生产部署**：Docker Compose 或二进制 + systemd
- **可扩展性**：模块化设计，支持自定义功能和集成

遵循本指南，即可快速搭建本地开发环境或部署到生产环境。

