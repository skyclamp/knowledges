# Prompt 03: 业务逻辑层分析

## 任务

分析 WeKnora 的 Service 层（业务逻辑层），理解核心业务流程和领域逻辑，输出业务逻辑层设计文档。

## 需要阅读的文件

1. **Service 层**（核心业务逻辑，这是重中之重）:
   - `/Users/wenkai/oss/WeKnora/internal/application/service/` — 整个目录
   - 重点文件：
     - `knowledgebase.go`, `knowledgebase_search.go`, `knowledgebase_search_faq.go`, `knowledgebase_search_fusion.go`, `knowledgebase_search_results.go`, `knowledgebase_search_shared.go` — 知识库与检索
     - `knowledge.go` — 知识管理
     - `chunk.go` — 分块管理
     - `session.go`, `session_knowledge_qa.go`, `session_agent_qa.go`, `session_qa_helpers.go` — 会话和问答
     - `agent_service.go`, `custom_agent.go` — Agent 管理
     - `model.go` — 模型管理
     - `datasource_service.go` — 数据源管理
     - `mcp_service.go` — MCP 服务管理
     - `web_search.go`, `web_search_provider.go`, `web_search_state.go` — 网络搜索
     - `evaluation.go` — 评测
     - `graph.go` — 知识图谱
     - `organization.go`, `tenant.go`, `user.go` — 组织管理
     - `tag.go` — 标签
     - `message.go` — 消息
     - `skill_service.go` — 技能
     - `extract.go` — 信息提取
     - `image_multimodal.go` — 多模态图像
     - `ocr_sanitizer.go` — OCR 清洗
     - `metric.go`, `metric_hook.go` — 指标

2. **Chat Pipeline**（问答管道）:
   - `/Users/wenkai/oss/WeKnora/internal/application/service/chat_pipeline/` — 整个目录

3. **Retriever**（检索器）:
   - `/Users/wenkai/oss/WeKnora/internal/application/service/retriever/` — 整个目录

4. **Memory**（会话记忆）:
   - `/Users/wenkai/oss/WeKnora/internal/application/service/memory/` — 整个目录

5. **LLM Context**（LLM 上下文管理）:
   - `/Users/wenkai/oss/WeKnora/internal/application/service/llmcontext/` — 整个目录

## 输出要求

写一份中文文档，保存到 `/Users/wenkai/workspace/knowledges/WeKnora/docs/03-业务逻辑层.md`，包含以下章节：

### 1. Service 层概览
- 所有 Service 的职责列表
- Service 之间的依赖关系（用 Mermaid 图表示）

### 2. 知识库管理
- 知识库的创建、更新、删除流程
- 知识库的配置项（检索策略、分块策略等）
- 共享知识库机制

### 3. 知识处理流水线
- 知识文档从上传到可检索的完整流程
- 文档解析 → 分块 → 向量化 → 入库 的每个步骤
- 支持的文档格式和各自的处理方式
- OCR 和多模态图像处理

### 4. 检索与问答
- **快速问答（RAG）流程**：Query → 检索 → 重排 → 生成答案
- **智能推理（Agent）流程**：Query → Agent 规划 → 工具调用 → 迭代推理 → 最终答案
- 混合检索策略（向量检索 + 关键词检索 + FAQ）
- Rerank 重排机制
- Chat Pipeline 的设计和各阶段

### 5. 会话管理
- 会话创建和生命周期
- 消息历史管理
- 会话记忆（Memory）机制
- 流式响应处理

### 6. 数据源自动同步
- 数据源类型（飞书等）
- 自动同步机制和调度
- 增量同步 vs 全量同步

### 7. 网络搜索
- 支持的搜索引擎
- 搜索提供商抽象层
- 搜索结果处理流程

### 8. 评测系统
- 评测的设计和流程
- 评测指标

### 9. 知识图谱
- 知识图谱的构建方式
- 图谱查询和使用
