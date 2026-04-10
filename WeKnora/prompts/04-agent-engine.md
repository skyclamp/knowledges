# Prompt 04: Agent 引擎与 AI 能力分析

## 任务

分析 WeKnora 的 Agent 引擎、工具系统、技能系统、MCP 集成、LLM 模型适配层，输出 AI 能力设计文档。

## 需要阅读的文件

1. **Agent 引擎核心**:
   - `/Users/wenkai/oss/WeKnora/internal/agent/engine.go` — Agent 引擎主逻辑
   - `/Users/wenkai/oss/WeKnora/internal/agent/think.go` — 思考/推理
   - `/Users/wenkai/oss/WeKnora/internal/agent/act.go` — 行动/执行
   - `/Users/wenkai/oss/WeKnora/internal/agent/observe.go` — 观察/反馈
   - `/Users/wenkai/oss/WeKnora/internal/agent/finalize.go` — 最终化
   - `/Users/wenkai/oss/WeKnora/internal/agent/prompts.go` — Prompt 模板
   - `/Users/wenkai/oss/WeKnora/internal/agent/const.go` — 常量

2. **Agent Token 管理**:
   - `/Users/wenkai/oss/WeKnora/internal/agent/token/` — 整个目录

3. **Agent Memory**:
   - `/Users/wenkai/oss/WeKnora/internal/agent/memory/` — 整个目录

4. **Agent 工具系统**:
   - `/Users/wenkai/oss/WeKnora/internal/agent/tools/` — 整个目录，所有工具实现
   - 重点关注：`registry.go`（工具注册）, `tool.go`（工具接口）, `definitions.go`（工具定义）
   - 每个工具文件：knowledge_search, web_search, web_fetch, database_query, data_analysis, mcp_tool, skill_execute, final_answer, todo_write, sequentialthinking, query_knowledge_graph, grep_chunks, list_knowledge_chunks, get_document_info 等

5. **Agent 技能系统**:
   - `/Users/wenkai/oss/WeKnora/internal/agent/skills/` — 整个目录
   - `/Users/wenkai/oss/WeKnora/skills/preloaded/` — 预加载技能
   - `/Users/wenkai/oss/WeKnora/config/builtin_agents.yaml` — 内置 Agent 配置
   - `/Users/wenkai/oss/WeKnora/docs/agent-skills.md` — 技能文档

6. **MCP（Model Context Protocol）集成**:
   - `/Users/wenkai/oss/WeKnora/internal/mcp/` — MCP 客户端
   - `/Users/wenkai/oss/WeKnora/docs/MCP功能使用说明.md`
   - `/Users/wenkai/oss/WeKnora/docs/BUILTIN_MCP_SERVICES.md`

7. **LLM 模型层**:
   - `/Users/wenkai/oss/WeKnora/internal/models/` — 整个目录
   - 子目录：`chat/`（聊天模型）, `embedding/`（向量模型）, `rerank/`（重排模型）, `vlm/`（视觉语言模型）, `asr/`（语音识别模型）, `provider/`（提供商抽象）, `utils/`
   - `/Users/wenkai/oss/WeKnora/docs/BUILTIN_MODELS.md`

8. **Prompt 模板**:
   - `/Users/wenkai/oss/WeKnora/config/prompt_templates/` — 整个目录

9. **沙箱执行**:
   - `/Users/wenkai/oss/WeKnora/internal/sandbox/` — 代码沙箱

## 输出要求

写一份中文文档，保存到 `/Users/wenkai/workspace/knowledges/WeKnora/docs/04-Agent引擎与AI能力.md`，包含以下章节：

### 1. Agent 引擎架构
- ReACT（Reasoning + Acting）循环的实现方式
- Think → Act → Observe 的完整流程（用 Mermaid 流程图）
- Agent 的生命周期（从接收问题到产出最终答案）
- 渐进式策略（progressive strategy）的具体实现
- 最大迭代轮数和终止条件

### 2. 工具系统
- 工具注册和发现机制
- 所有内置工具列表，每个工具的：
  - 功能描述
  - 输入/输出参数
  - 使用场景
- 工具调用的执行流程（包括并行工具调用）
- JSON 参数解析和修复机制

### 3. 技能系统
- 技能与工具的区别
- 技能的加载和管理机制
- 预加载技能列表和用途
- 自定义技能的扩展方式

### 4. MCP 集成
- MCP 协议在 WeKnora 中的角色
- MCP 客户端实现
- 内置 MCP 服务
- MCP 工具调用流程
- 自动重连机制
- MCP 工具图像处理（VLM 自动描述）

### 5. LLM 模型适配层
- 模型提供商抽象设计
- 支持的 LLM 提供商完整列表及各自的特点
- Chat 模型、Embedding 模型、Rerank 模型、VLM 模型、ASR 模型的适配
- 流式响应处理
- Token 计量

### 6. Prompt 工程
- System Prompt 的构建方式
- 工具描述 Prompt 的生成
- 上下文注入策略
- Prompt 模板管理

### 7. 内置 Agent 配置
- 内置 Agent 的种类和用途
- Agent 配置项说明
- 自定义 Agent 的扩展机制

### 8. 代码沙箱
- 沙箱的用途和场景
- Docker 沙箱 vs 本地沙箱
- 安全机制和限制
