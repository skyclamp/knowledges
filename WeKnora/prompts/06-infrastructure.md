# Prompt 06: 基础设施层分析

## 任务

分析 WeKnora 的基础设施组件：文档解析、文本分块、搜索工具、向量数据库集成、配置管理、日志、链路追踪、错误处理等，输出基础设施层设计文档。

## 需要阅读的文件

1. **文档解析**:
   - `/Users/wenkai/oss/WeKnora/internal/infrastructure/docparser/` — 文档解析器
   - `/Users/wenkai/oss/WeKnora/internal/infrastructure/web_fetch/` — 网页抓取

2. **文本分块**:
   - `/Users/wenkai/oss/WeKnora/internal/infrastructure/chunker/` — 分块器

3. **搜索工具**:
   - `/Users/wenkai/oss/WeKnora/internal/searchutil/` — 搜索工具集
   - `/Users/wenkai/oss/WeKnora/internal/infrastructure/web_search/` — 网络搜索引擎集成

4. **数据源连接器**:
   - `/Users/wenkai/oss/WeKnora/internal/datasource/` — 整个目录
   - `/Users/wenkai/oss/WeKnora/internal/datasource/connector/` — 各数据源连接器实现
   - `/Users/wenkai/oss/WeKnora/internal/datasource/CONNECTOR_IMPLEMENTATION_GUIDE.md`
   - `/Users/wenkai/oss/WeKnora/docs/数据源导入开发文档.md`

5. **配置管理**:
   - `/Users/wenkai/oss/WeKnora/internal/config/` — 配置结构和加载
   - `/Users/wenkai/oss/WeKnora/config/config.yaml` — 默认配置
   - `/Users/wenkai/oss/WeKnora/.env.example` — 环境变量

6. **日志**:
   - `/Users/wenkai/oss/WeKnora/internal/logger/` — 日志系统

7. **链路追踪**:
   - `/Users/wenkai/oss/WeKnora/internal/tracing/` — 链路追踪

8. **错误处理**:
   - `/Users/wenkai/oss/WeKnora/internal/errors/` — 错误定义

9. **工具函数**:
   - `/Users/wenkai/oss/WeKnora/internal/utils/` — 通用工具

10. **文件存储**:
    - `/Users/wenkai/oss/WeKnora/internal/assets/` — 文件资产管理
    - `/Users/wenkai/oss/WeKnora/internal/application/service/file/` — 文件服务

11. **数据库迁移**:
    - `/Users/wenkai/oss/WeKnora/migrations/` — 迁移文件
    - `/Users/wenkai/oss/WeKnora/internal/database/migration.go`

12. **构建和脚本**:
    - `/Users/wenkai/oss/WeKnora/Makefile` — 构建命令
    - `/Users/wenkai/oss/WeKnora/scripts/` — 脚本

## 输出要求

写一份中文文档，保存到 `/Users/wenkai/workspace/knowledges/WeKnora/docs/06-基础设施层.md`，包含以下章节：

### 1. 文档解析
- 支持的文档格式（PDF、Word、Excel、图片、HTML 等）
- 解析器架构和接口设计
- 各格式的解析实现方式
- 与 docreader Python 服务的协作方式
- OCR 处理

### 2. 文本分块策略
- 支持的分块策略（固定大小、语义、父子分块等）
- 分块参数（大小、重叠等）
- 父子分块（Parent-Child Chunking）机制

### 3. 搜索与检索工具
- 全文搜索实现
- 向量搜索实现
- 混合搜索（Hybrid Search）策略
- 搜索结果融合算法
- 繁简转换（OpenCC）支持

### 4. 向量数据库集成
- 向量数据库抽象接口
- 各向量数据库适配器（SQLite-Vec、Milvus、Weaviate、Elasticsearch 等）
- 向量索引管理
- `/Users/wenkai/oss/WeKnora/docs/使用其他向量数据库.md` 中的使用指南

### 5. 网络搜索引擎集成
- 搜索引擎提供商接口
- 已集成的搜索引擎（Bing、Google、Tavily 等）
- 搜索结果处理
- `/Users/wenkai/oss/WeKnora/docs/添加新的网络搜索引擎.md` 中的扩展指南

### 6. 数据源连接器
- 连接器架构和接口
- 已实现的连接器（飞书等）
- 调度器设计（全量同步、增量同步）
- 扩展新连接器的方式

### 7. 配置管理
- 配置文件结构
- 配置层级（默认值 → 配置文件 → 环境变量）
- 关键配置项说明

### 8. 日志与可观测性
- 日志框架和级别
- 链路追踪（Tracing）实现
- 指标收集

### 9. 错误处理
- 错误类型定义
- 错误码规范
- 错误传播策略

### 10. 构建与部署
- Makefile 中的构建命令
- Docker 构建流程
- CI/CD 配置（如有）
