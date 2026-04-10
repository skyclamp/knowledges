# WeKnora 源代码分析 — Prompt 执行指南

## 工作方式

按编号顺序执行每个 prompt 文件（01 → 08），每个 prompt 会指示你：
- 阅读哪些源代码目录/文件
- 输出什么格式的文档
- 文档保存到哪里

## 输出目录

所有分析文档输出到：`/Users/wenkai/workspace/knowledges/WeKnora/docs/`

## Prompt 列表

| # | 文件 | 分析范围 | 输出文档 |
|---|------|---------|---------|
| 01 | `01-product-and-architecture.md` | 产品设计 + 系统架构总览 | `01-产品设计与系统架构.md` |
| 02 | `02-data-model-and-storage.md` | 数据模型、存储层、Repository | `02-数据模型与存储层.md` |
| 03 | `03-business-logic.md` | Service 层业务逻辑 | `03-业务逻辑层.md` |
| 04 | `04-agent-engine.md` | Agent 引擎、工具、技能、MCP | `04-Agent引擎与AI能力.md` |
| 05 | `05-api-and-integration.md` | API 路由、Handler、中间件、IM 集成 | `05-API与集成层.md` |
| 06 | `06-infrastructure.md` | 文档解析、分块、检索、向量库、配置 | `06-基础设施层.md` |
| 07 | `07-frontend.md` | Vue.js 前端架构 | `07-前端架构.md` |
| 08 | `08-python-services.md` | docreader + mcp-server Python 服务 | `08-Python服务.md` |

## 注意事项

- 每个 prompt 都是独立的，可以单独执行
- 分析时要基于源代码实际内容，不要臆测
- 关注设计决策和 why，不只是 what
- 中文输出
