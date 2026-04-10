# Prompt 08: Python 服务分析

## 任务

分析 WeKnora 的两个 Python 子服务：docreader（文档解析服务）和 mcp-server（MCP 服务），输出 Python 服务设计文档。

## 需要阅读的文件

### docreader 文档解析服务

1. **项目配置**:
   - `/Users/wenkai/oss/WeKnora/docreader/pyproject.toml` — 项目配置和依赖
   - `/Users/wenkai/oss/WeKnora/docreader/Makefile` — 构建命令
   - `/Users/wenkai/oss/WeKnora/docreader/README.md` — 服务说明

2. **服务入口**:
   - `/Users/wenkai/oss/WeKnora/docreader/main.py` — 主入口
   - `/Users/wenkai/oss/WeKnora/docreader/config.py` — 配置

3. **解析器**:
   - `/Users/wenkai/oss/WeKnora/docreader/parser/` — 整个目录，各文档格式的解析实现

4. **文本分割**:
   - `/Users/wenkai/oss/WeKnora/docreader/splitter/` — 文本分割器

5. **OCR**:
   - `/Users/wenkai/oss/WeKnora/docreader/ocr/` — OCR 处理

6. **模型**:
   - `/Users/wenkai/oss/WeKnora/docreader/models/` — 数据模型

7. **gRPC/Proto**:
   - `/Users/wenkai/oss/WeKnora/docreader/proto/` — gRPC 服务定义

8. **工具函数**:
   - `/Users/wenkai/oss/WeKnora/docreader/utils/` — 工具
   - `/Users/wenkai/oss/WeKnora/docreader/client/` — 客户端

### mcp-server MCP 服务

1. **项目配置和文档**:
   - `/Users/wenkai/oss/WeKnora/mcp-server/pyproject.toml`
   - `/Users/wenkai/oss/WeKnora/mcp-server/README.md`
   - `/Users/wenkai/oss/WeKnora/mcp-server/INSTALL.md`
   - `/Users/wenkai/oss/WeKnora/mcp-server/EXAMPLES.md`
   - `/Users/wenkai/oss/WeKnora/mcp-server/MCP_CONFIG.md`
   - `/Users/wenkai/oss/WeKnora/mcp-server/PROJECT_SUMMARY.md`

2. **服务实现**:
   - `/Users/wenkai/oss/WeKnora/mcp-server/main.py`
   - `/Users/wenkai/oss/WeKnora/mcp-server/run.py`
   - `/Users/wenkai/oss/WeKnora/mcp-server/run_server.py`
   - `/Users/wenkai/oss/WeKnora/mcp-server/weknora_mcp_server.py` — MCP 服务主实现
   - `/Users/wenkai/oss/WeKnora/mcp-server/__init__.py`

3. **测试**:
   - `/Users/wenkai/oss/WeKnora/mcp-server/test_imports.py`
   - `/Users/wenkai/oss/WeKnora/mcp-server/test_module.py`

## 输出要求

写一份中文文档，保存到 `/Users/wenkai/workspace/knowledges/WeKnora/docs/08-Python服务.md`，包含以下章节：

### Part A: docreader 文档解析服务

#### A1. 服务概述
- 服务的职责和定位
- 与 Go 后端的通信方式（gRPC/HTTP）
- 部署方式

#### A2. 文档解析架构
- 解析器接口设计
- 支持的文档格式列表
- 各格式的解析策略：
  - PDF 解析（文本提取、表格识别、图片处理）
  - Word/DOCX 解析
  - Excel 解析
  - HTML 解析
  - 图片解析（OCR）
  - 其他格式

#### A3. OCR 处理
- OCR 引擎选择
- 图像预处理
- OCR 结果后处理

#### A4. 文本分割
- 分割策略
- 分割参数配置
- 中文/英文分割的差异处理

#### A5. gRPC 接口
- Proto 定义
- 服务端点列表
- 请求/响应格式

#### A6. 依赖和技术选型
- Python 版本要求
- 关键依赖库（如 pymupdf、python-docx、openpyxl、paddleocr 等）

### Part B: mcp-server MCP 服务

#### B1. 服务概述
- MCP Server 的用途
- 作为独立服务提供的能力
- 部署和配置方式

#### B2. MCP 协议实现
- 暴露的 MCP 工具列表
- 每个工具的功能描述和参数
- 与 WeKnora 后端 API 的交互方式

#### B3. 使用场景
- 在 Claude Desktop、Cursor 等 MCP 客户端中使用
- 配置示例

#### B4. 技术实现
- 服务框架
- 认证机制
- 错误处理
