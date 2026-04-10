# Prompt 02: 数据模型与存储层分析

## 任务

分析 WeKnora 的数据模型定义、数据库 schema、Repository 层实现，输出数据模型与存储层设计文档。

## 需要阅读的文件

1. **类型定义**（核心数据模型，这是最重要的部分）:
   - `/Users/wenkai/oss/WeKnora/internal/types/` — 整个目录，包含所有业务实体定义
   - 重点关注：`tenant.go`, `organization.go`, `user.go`, `knowledgebase.go`, `knowledge.go`, `chunk.go`, `session.go`, `message.go`, `agent.go`, `custom_agent.go`, `model.go`, `tag.go`, `faq.go`, `mcp.go`, `datasource.go`, `web_search.go`, `web_search_provider.go`, `evaluation.go`, `memory.go`, `retriever.go`, `graph.go`
   - 同时关注 `interfaces/` 子目录 — 接口定义

2. **Repository 层**（数据访问层实现）:
   - `/Users/wenkai/oss/WeKnora/internal/application/repository/` — 整个目录

3. **数据库迁移**（了解 schema 演变）:
   - `/Users/wenkai/oss/WeKnora/migrations/` — 数据库迁移文件
   - `/Users/wenkai/oss/WeKnora/internal/database/migration.go`

4. **向量数据库集成**:
   - 在 go.mod 中搜索向量数据库相关的依赖（sqlite-vec, milvus, weaviate, elasticsearch 等）
   - 在 `internal/` 中搜索 vector / embedding 相关的实现

5. **存储层**:
   - `/Users/wenkai/oss/WeKnora/internal/assets/` — 文件存储相关
   - 搜索 S3、MinIO、本地存储相关实现

## 输出要求

写一份中文文档，保存到 `/Users/wenkai/workspace/knowledges/WeKnora/docs/02-数据模型与存储层.md`，包含以下章节：

### 1. 实体关系模型
- 用 Mermaid ER 图展示核心实体之间的关系
- 重点实体：Tenant、Organization、User、KnowledgeBase、Knowledge、Chunk、Session、Message、Agent、Model、Tag、FAQ、MCP Service
- 标注一对多 / 多对多关系

### 2. 核心实体详解
对每个核心实体：
- 字段说明（名称、类型、用途）
- 关键业务规则和约束
- 与其他实体的关联关系

### 3. 多租户模型
- 租户（Tenant）的数据隔离方式
- 组织（Organization）和用户的关系
- 权限模型概述

### 4. Repository 层设计
- Repository 接口设计模式
- 各 Repository 的职责
- 数据访问层的通用模式（分页、过滤、软删除等）

### 5. 数据库 Schema
- 主要数据表列表及用途
- 索引设计
- 迁移管理机制

### 6. 向量数据库集成
- 支持的向量数据库列表
- 向量存储抽象层设计
- Embedding 存储和检索方式

### 7. 文件存储
- 支持的存储后端（本地、S3、MinIO 等）
- 存储抽象层设计
- 文件上传和管理流程
