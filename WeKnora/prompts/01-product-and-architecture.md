# Prompt 01: 产品设计与系统架构分析

## 任务

分析 WeKnora 项目的产品定位、核心功能、系统架构，输出一份全面的产品与架构设计文档。

## 需要阅读的文件

按以下顺序阅读：

1. **项目 README**（了解产品定位和功能概述）:
   - `/Users/wenkai/oss/WeKnora/README.md`
   - `/Users/wenkai/oss/WeKnora/README_CN.md`

2. **已有文档**（了解官方设计说明）:
   - `/Users/wenkai/oss/WeKnora/docs/WeKnora.md`
   - `/Users/wenkai/oss/WeKnora/docs/ROADMAP.md`
   - `/Users/wenkai/oss/WeKnora/docs/开发指南.md`

3. **版本和变更记录**:
   - `/Users/wenkai/oss/WeKnora/VERSION`
   - `/Users/wenkai/oss/WeKnora/CHANGELOG.md`

4. **项目入口和配置**（了解系统组成）:
   - `/Users/wenkai/oss/WeKnora/cmd/server/` — 主服务入口
   - `/Users/wenkai/oss/WeKnora/config/config.yaml` — 默认配置
   - `/Users/wenkai/oss/WeKnora/internal/config/` — 配置结构定义
   - `/Users/wenkai/oss/WeKnora/docker-compose.yml` — 部署架构
   - `/Users/wenkai/oss/WeKnora/docker-compose.dev.yml`

5. **依赖声明**（了解技术选型）:
   - `/Users/wenkai/oss/WeKnora/go.mod` — Go 后端依赖
   - `/Users/wenkai/oss/WeKnora/frontend/package.json` — 前端依赖
   - `/Users/wenkai/oss/WeKnora/docreader/pyproject.toml` — 文档解析服务依赖
   - `/Users/wenkai/oss/WeKnora/mcp-server/pyproject.toml` — MCP 服务依赖

6. **依赖注入容器**（了解模块组装）:
   - `/Users/wenkai/oss/WeKnora/internal/container/container.go`
   - `/Users/wenkai/oss/WeKnora/internal/runtime/container.go`

7. **路由总览**（了解 API 全貌）:
   - `/Users/wenkai/oss/WeKnora/internal/router/router.go`

## 输出要求

写一份中文文档，保存到 `/Users/wenkai/workspace/knowledges/WeKnora/docs/01-产品设计与系统架构.md`，包含以下章节：

### 1. 产品定位
- WeKnora 是什么产品，解决什么问题
- 目标用户群体
- 核心价值主张

### 2. 核心功能清单
- 列出所有主要功能模块，每个简要说明用途
- 区分已实现 vs 规划中的功能（参考 ROADMAP）

### 3. 系统架构
- 整体架构图（用 Mermaid 图表）
- 服务组成：Go 后端、Vue 前端、Python docreader、Python MCP server、依赖服务（数据库、向量库、Redis 等）
- 各服务之间的交互关系

### 4. 技术选型
- 后端框架和关键库
- 前端框架和关键库
- 数据库和向量数据库选型（支持哪些）
- LLM 提供商（支持哪些）
- 存储方案
- 消息队列 / 任务队列

### 5. 代码结构总览
- 顶层目录结构说明
- `internal/` 包的分层架构（handler → service → repository 等）
- 前端 `src/` 目录结构说明
- 配置管理方式

### 6. 部署架构
- Docker Compose 部署方案
- Helm Chart 部署方案（如有）
- 必需的外部依赖服务

### 7. 关键设计决策
- 为什么选择这种架构分层
- 多租户设计
- 可插拔的 LLM / 向量库 / 存储设计
