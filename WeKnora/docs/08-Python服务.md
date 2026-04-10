# WeKnora Python 服务设计文档

本文基于当前仓库中的 Python 代码与配套文档，对 WeKnora 的两个 Python 子服务进行设计分析：

- docreader：文档解析服务
- mcp-server：MCP 协议接入服务

需要特别说明的是，这两个目录中都存在一部分“历史实现 / 兼容保留代码 / 文档先行描述”。本文优先按当前代码主链路描述，并在必要处明确指出与 README 或预留能力之间的差异。

## Part A: docreader 文档解析服务

### A1. 服务概述

#### 1. 服务职责和定位

docreader 是 WeKnora 的文档读取与格式转换子服务，核心职责是把输入文档或 URL 转换为统一的 Markdown 表示，并返回解析过程中发现的图片引用与元数据。

当前版本的主职责可以概括为三点：

1. 按文件类型或 URL 路由到对应解析器。
2. 将原始内容转换为 Markdown 文本。
3. 将图片以内联字节或引用形式返回给 Go 主服务，由 Go 侧负责后续持久化和索引流程。

当前代码中的注释已经明确表明，docreader 经过一次“轻量化重构”后，以下能力不再由 Python 主链路承担，而是移交给 Go 应用层：

- 文本切块
- 图片存储
- OCR 主流程编排
- VLM 图像描述

也就是说，当前 docreader 更接近“统一文档转 Markdown 引擎”，而不是一个完整的文档 ETL 编排中心。

#### 2. 与 Go 后端的通信方式（gRPC/HTTP）

当前运行时的主通信方式是 gRPC。

- Python 侧在 `main.py` 中启动 `grpc.server(...)`，默认监听 `50051`。
- Go 侧通过 `internal/infrastructure/docparser/grpc_parser.go` 建立 `DocReaderClient`，调用 `Read` 和 `ListEngines` 两个 RPC。
- Docker Compose 中主应用通过 `DOCREADER_ADDR=docreader:50051` 与 docreader 通信，并显式设置 `DOCREADER_TRANSPORT=grpc`。

仓库中同时保留了 Go 侧的 `HTTPDocumentReader` 适配器，说明系统设计上兼容 HTTP/JSON 形式的远程 docreader。但当前 Python 服务并没有提供 `/read`、`/list-engines` 之类的 HTTP 端点实现，因此在现状下 HTTP 更像是预留接口而不是正在运行的主链路。

结论：

- 当前有效路径：Go -> gRPC -> Python docreader
- 兼容预留路径：Go -> HTTP/JSON -> 远程解析服务（当前 Python 目录未实现）

#### 3. 部署方式

docreader 主要以独立容器部署：

- 镜像名：`wechatopenai/weknora-docreader`
- 默认端口：`50051`
- 健康检查：`grpc_health_probe -addr=:50051`
- 挂载卷：`/tmp/docreader`

在 `docker-compose.yml` 和 `docker-compose.dev.yml` 中，docreader 都被定义为独立服务，由主应用依赖其健康状态后再启动。

部署上有两个明显特征：

1. 它被当作独立进程/容器运行，而不是嵌入 Go 进程。
2. 它保留了若干环境变量接口，例如 `MINERU_ENDPOINT`、`MAX_FILE_SIZE_MB`、代理参数和存储参数，但这些配置并不都已经接入当前主代码路径。

此外，`Makefile` 中的 `run` 目标仍指向旧路径 `src/server/server.py`，与当前根目录入口 `main.py` 不一致，说明 Makefile 存在一定历史残留。

### A2. 文档解析架构

#### 1. 解析器接口设计

docreader 的解析架构由三层组成：

1. `Parser` 门面类：统一暴露 `parse_file` 和 `parse_url`。
2. `ParserEngineRegistry`：根据“解析引擎 + 文件类型”选择具体解析器。
3. `BaseParser` 及其子类：每种格式各自实现 `parse_into_text(content: bytes) -> Document`。

统一数据模型是 `models/document.py` 中的 `Document`：

- `content`：最终 Markdown 或纯文本内容
- `images`：`{ref_path: base64}` 形式的图片映射
- `chunks`：遗留字段，当前主链路很少依赖
- `metadata`：标题等附加信息

`BaseParser` 的设计重点是“只做文本与图片抽取”，不负责切块和存储。这一点与当前服务定位完全一致。

#### 2. 支持的文档格式列表

当前注册到解析器注册表中的文件类型分两类。

内置 `builtin` 引擎支持：

- `docx`
- `doc`
- `pdf`
- `md`
- `markdown`
- `xlsx`
- `xls`
- `jpg`
- `jpeg`
- `png`
- `gif`
- `bmp`
- `tiff`
- `webp`

可选 `markitdown` 引擎支持：

- `md`
- `markdown`
- `pdf`
- `docx`
- `doc`
- `pptx`
- `ppt`
- `xlsx`
- `xls`
- `csv`

URL 模式不走文件扩展名注册表，而是由 `Parser.parse_url()` 直接调用 `WebParser`。

需要注意的几个事实：

- 当前没有为独立 `.html` 文件注册 parser；HTML 处理主要是 URL 抓取场景。
- 注释里提到的纯文本、更多格式支持，并不都体现在当前注册表中。
- `ppt/pptx/csv` 仅在显式选择 `markitdown` 引擎时可用，不属于 builtin 默认能力。

#### 3. 各格式的解析策略

##### PDF 解析（文本提取、表格识别、图片处理）

当前 `PDFParser` 继承自 `FirstParser`，但实际只配置了一个后端：`MarkitdownParser`。

因此，当前 PDF 主策略是：

1. 使用 MarkItDown 将 PDF 转为 Markdown。
2. 使用 `MarkdownParser` 做二次处理：
   - 规范 Markdown 表格格式
   - 提取 data URI/base64 图片
   - 把图片替换为 `images/...` 引用，并把图片字节写入 `Document.images`

这意味着：

- 文本提取：主要依赖 MarkItDown 内部的 PDF 转 Markdown 能力。
- 表格识别：没有自建表格识别流程，依赖 MarkItDown 输出的表格结构，再由 `MarkdownTableFormatter` 统一格式。
- 图片处理：主要是抽取 MarkItDown 输出中的内嵌图片数据，再交给上层返回给 Go 侧。

虽然 `pyproject.toml` 中声明了 `pdfplumber`、`pypdf`、`pypdf2` 等依赖，README 里也提到了 MinerU，但当前 Python 注册表并没有暴露 MinerU PDF 解析器，`PDFParser` 也没有接入这些库形成主解析链。因此，现阶段的 PDF 解析是“MarkItDown 主导”，而不是“多后端智能切换”。

##### Word/DOCX 解析

DOCX 的主入口是 `Docx2Parser`，它按顺序尝试：

1. `MarkitdownParser`
2. `DocxParser`

也就是“先用通用转换器，失败后退回专用解析器”。

`DocxParser` 的能力比 MarkItDown 版本更细：

- 用 `python-docx` 读取段落、表格和图片
- 对损坏关系文件做兼容 patch，避免部分 Word 文件加载失败
- 表格转换为 HTML `<table>` 结构保留到文本中
- 提取嵌入图片，转为 base64 放入 `Document.images`
- 失败时回退到简化模式，只抽取段落和表格文本

因此 DOCX 的策略可以理解为：

- 优先快速转换为 Markdown
- 失败时用专用 Word 解析器兜底
- 图片和表格尽量保留

##### Word/DOC 解析

DOC 由 `DocParser` 处理，策略是三级回退：

1. 先尝试调用 `soffice` 把 `.doc` 转成 `.docx`
2. 转换成功后复用 DOCX 解析流程
3. 如果失败，再调用 `antiword` 提取纯文本

代码中保留了 `textract` 路径，但注明因 SSRF 风险而禁用，不属于当前执行链。

这说明 DOC 支持是明显的兼容型能力：

- 优先保留结构和图片
- 实在不行退回纯文本提取

##### Excel 解析

Excel 由 `ExcelParser` 负责，处理方式非常直接：

1. 用 `pandas.ExcelFile` 打开工作簿
2. 遍历所有 sheet
3. 删除全空行
4. 将每行转成 `列名: 值` 的逗号串
5. 每行生成一个 chunk，同时拼接成全文

这种策略的优点是简单稳定、利于后续检索；缺点是：

- 不保留复杂样式
- 不处理公式语义和图表
- 更适合作为结构化表格转文本，而非高保真复原

##### HTML 解析

当前没有独立 `.html` 文件 parser 的 builtin 注册项。

HTML 解析的主场景是“网页 URL 解析”，由 `WebParser` 完成：

1. `StdWebParser` 用 Playwright WebKit 抓取完整页面 HTML
2. 用 Trafilatura 提取正文并直接输出 Markdown
3. 打开 `with_metadata`、`include_images`、`include_tables`、`include_links`
4. 再交给 `MarkdownParser` 标准化表格和抽取 base64 图片

因此，这里的 HTML 解析本质上是“网页抓取 + 正文抽取”，而不是“上传 HTML 文件后逐标签解析”。

##### 图片解析（OCR）

当前注册到主链路中的图片文件解析器是 `ImageParser`，但它并不做 OCR。

它的行为非常简单：

- 为图片生成一个 Markdown 图片链接 `![file](images/...)`
- 把原始图片内容以 base64 写入 `Document.images`

这说明当前主路径对图片文件的处理是“保留图片引用”，不是“把图片内容识别成文字”。

OCR 能力虽然存在于 `ocr/` 目录，但没有接入现行主解析流程，详见 A3。

##### 其他格式

当前值得单独说明的其他格式包括：

- Markdown：`MarkdownParser`，负责表格规范化与 base64 图片抽取
- PPT/PPTX：仅在 `markitdown` 引擎下可用
- CSV：仅在 `markitdown` 引擎下可用
- Web URL：`WebParser`

总体上，docreader 当前格式支持是“核心常见格式内置 + 其他格式依赖 MarkItDown 扩展”。

### A3. OCR 处理

#### 1. OCR 引擎选择

`ocr/__init__.py` 中定义了 `OCREngine` 工厂，可选择三类后端：

- `paddle`：`PaddleOCRBackend`
- `vlm`：`VLMOCRBackend`
- 其他默认值：`DummyOCRBackend`

从代码设计看，OCR 希望支持两种模式：

1. 传统 OCR：PaddleOCR
2. 视觉语言模型 OCR：OpenAI 兼容接口

但要强调的是，这套 OCR 工厂目前没有接入 `main.py -> Parser -> registry parser` 这条主执行链，因此它更像是保留的扩展能力，而不是默认启用能力。

#### 2. 图像预处理

PaddleOCR 后端的预处理较轻量，主要包括：

- 强制 CPU 模式，禁用 GPU
- Linux 下探测 AVX 指令集兼容性，必要时降低指令集要求
- 将输入统一为 PIL Image
- 非 RGB 图像先转 RGB
- 再转为 `numpy` 数组输入 PaddleOCR
- 开启文档方向分类和文本行方向检测

当前代码没有看到更复杂的前处理，例如：

- 自适应二值化
- 去噪
- 倾斜矫正
- 版面切块

因此它更偏向直接 OCR，而不是完整图像增强流水线。

#### 3. OCR 结果后处理

PaddleOCR 的后处理非常简单：

- 从识别结果中取每一行文字
- 去掉空串
- 用空格拼接成单一文本字符串

VLM OCR 后端则走另一条路线：

- 把图片编码为 base64 data URI
- 调用 OpenAI 兼容的 chat completion 接口
- 提示词要求输出 Markdown，表格用 HTML，公式用 LaTeX，忽略页眉页脚

这说明两类 OCR 的定位不同：

- PaddleOCR：偏纯文本识别
- VLM OCR：偏文档理解和结构化转写

#### 4. 当前落地状态

从当前代码状态看，OCR 有两个现实限制：

1. 主链路未接入。
2. `config.py` 中并没有提供 `VLMOCRBackend` 所访问的 `ocr_model`、`ocr_api_key`、`ocr_api_base_url` 等字段。

也就是说，README 中关于 `OCR_BACKEND`、`OCR_API_BASE_URL`、`OCR_MODEL` 的配置说明，比当前主代码的实际接入程度更超前。文档设计上应将 OCR 视作“保留/演进中的能力”，而不是“当前默认启用的处理步骤”。

### A4. 文本分割

#### 1. 分割策略

`splitter/splitter.py` 中实现了一个较完整的 `TextSplitter`，支持：

- chunk size / overlap 控制
- 按多级分隔符递归拆分
- 对公式、图片、链接、表格、代码块做“保护分裂”
- 通过 `HeaderTracker` 追踪表头，在跨 chunk 时保留上下文

默认流程是：

1. 先按分隔符递归切分
2. 识别受保护片段，避免被打碎
3. 重新合并成最终 chunk
4. 在 chunk 之间保留 overlap
5. 对表格等结构尝试附加活动表头上下文

从算法设计上看，这套 splitter 更偏面向 Markdown/知识检索，而不是简单按字数截断。

#### 2. 分割参数配置

默认参数如下：

- `chunk_size = 512`
- `chunk_overlap = 100`
- `separators = ["\n", "。", " "]`

另外还可以自定义：

- `protected_regex`
- `length_function`
- `header_hook`

保护规则默认覆盖：

- LaTeX 数学块
- Markdown 图片
- Markdown 链接
- Markdown 表格头与表格体
- 代码块头

#### 3. 中文/英文分割的差异处理

当前 splitter 没有显式做中英文分词，但通过分隔符差异处理了两类语言：

- 中文：默认把 `。` 作为分割点，适合句级切分
- 英文：主要依赖空格与换行

同时，它的长度函数默认按字符数而不是 token 数计算，所以这是“字符近似切分”，并不是基于 LLM tokenizer 的精确 token 切块。

#### 4. 当前运行路径中的地位

虽然 splitter 设计完整，但当前主链路实际上并不依赖它。

代码和注释都明确说明：切块已经迁移到 Go 侧完成，Python docreader 现在主要返回整篇 Markdown 和图片引用。`models/read_config.py` 中的 chunking 配置类也被标注为“仅用于兼容旧构造函数”。

因此可以这样理解：

- `splitter/` 代表 docreader 的历史或备用切块实现
- 当前生产主流程中的切块责任已经上移到 Go 后端

### A5. gRPC 接口

#### 1. Proto 定义

docreader 的 gRPC 接口定义在 `proto/docreader.proto`，服务名为 `DocReader`，包含两个 RPC：

- `Read(ReadRequest) returns (ReadResponse)`
- `ListEngines(ListEnginesRequest) returns (ListEnginesResponse)`

这是一个很小但边界清晰的接口：

- `Read` 负责统一读取文件或 URL
- `ListEngines` 负责告诉 Go 端当前 Python 服务支持哪些解析引擎与文件类型

#### 2. 服务端点列表

从 RPC 角度，当前有效端点只有两个：

1. `docreader.DocReader/Read`
2. `docreader.DocReader/ListEngines`

此外还挂载了 gRPC Health 服务，供容器健康检查使用。

#### 3. 请求格式

`ReadRequest` 支持两种模式：

- 文件模式：设置 `file_content`、`file_name`、`file_type`
- URL 模式：设置 `url`、`title`

附加字段包括：

- `config.parser_engine`：指定解析引擎，例如 `builtin` 或 `markitdown`
- `config.parser_engine_overrides`：引擎级覆盖配置
- `request_id`：请求追踪标识

`ListEnginesRequest` 只包含：

- `config_overrides`：用于检查某些引擎在当前配置下是否可用

#### 4. 响应格式

`ReadResponse` 的主要字段是：

- `markdown_content`：解析结果正文
- `image_refs`：图片引用列表
- `image_dir_path`：图片目录路径
- `metadata`：元数据键值对
- `error`：错误信息

`ImageRef` 中包括：

- `filename`
- `original_ref`
- `mime_type`
- `storage_key`
- `image_data`

需要注意，当前 `main.py` 中的 `_resolve_images()` 会把图片解码成内联字节塞到 `image_data`，并返回空的 `image_dir_path`。这再次说明：图片持久化的最终责任已经交给 Go 侧。

`ListEnginesResponse` 返回若干 `ParserEngineInfo`：

- `name`
- `description`
- `file_types`
- `available`
- `unavailable_reason`

#### 5. 与 Go 后端的协作方式

Go 侧 `GRPCDocumentReader` 会把应用层的 `types.ReadRequest` 映射为 proto 请求，再把 `ReadResponse` 转回 `types.ReadResult`。

因此整体协作链路是：

1. Go 应用收到文件/URL 导入请求
2. 调用 docreader gRPC `Read`
3. Python 返回 Markdown、图片字节和元数据
4. Go 侧继续做图片存储、切块、索引、入库

### A6. 依赖和技术选型

#### 1. Python 版本要求

`docreader/pyproject.toml` 要求：

- Python `>= 3.10.18`

这是一个相对明确的生产要求，而不是泛泛的“3.10+”。

#### 2. 关键依赖库

当前依赖可以分为几组。

gRPC 与协议层：

- `grpcio`
- `grpcio-health-checking`
- `grpcio-tools`
- `protobuf`

文档解析主依赖：

- `markitdown[docx,pdf,xls,xlsx]`
- `python-docx`
- `antiword`
- `textract`（代码中保留，但某些路径已禁用）
- `pypdf` / `pypdf2` / `pdfplumber`（已声明，但当前 PDF 主链路并未直接使用）

网页抓取与 HTML 正文抽取：

- `playwright`
- `trafilatura`
- `goose3`
- `beautifulsoup4`
- `lxml`
- `markdownify`

图像与 OCR：

- `pillow`
- `paddleocr`
- `paddlepaddle`
- `openai`
- `ollama`

存储与对象访问：

- `minio`
- `cos-python-sdk-v5`
- `requests`

建模与配置：

- `pydantic`

#### 3. 关于 Excel 依赖的现实情况

代码中的 `ExcelParser` 明确使用了 `pandas.ExcelFile`，但 `pyproject.toml` 并没有直接声明 `pandas`、`openpyxl` 或 `xlrd`。

这意味着当前 Excel 解析存在一个现实特征：

- 代码层面依赖 pandas 生态
- 依赖声明层面可能依靠转译依赖、镜像环境预装，或与当前代码不同步

如果从工程治理角度看，这部分依赖声明并不完全收敛，后续值得补齐。

#### 4. 技术选型总结

docreader 的技术路线非常明确：

- 协议层：gRPC
- 格式解析：专用 parser + MarkItDown 混合
- 网页解析：Playwright + Trafilatura
- Word 兼容：python-docx + soffice + antiword
- OCR：PaddleOCR / OpenAI 兼容 VLM（当前未完全接入主链）
- 服务形态：独立 Python 容器，作为 Go 主服务的外部解析器

---

## Part B: mcp-server MCP 服务

### B1. 服务概述

#### 1. MCP Server 的用途

`mcp-server` 的用途是把 WeKnora 的 REST API 封装为 MCP 工具集合，供支持 MCP 的 AI 客户端直接调用。

换句话说，它不是 WeKnora 内部主业务服务的一部分，而是一个“协议适配层”：

- 上游：Claude Desktop、Cursor、KiloCode 等 MCP 客户端
- 下游：WeKnora 的 HTTP API

它的本质工作是：

1. 向 MCP 客户端声明可用工具列表。
2. 接收工具调用参数。
3. 转换为对 WeKnora API 的 HTTP 请求。
4. 把 JSON 结果再包装成 MCP 文本响应返回。

#### 2. 作为独立服务提供的能力

当前代码实际暴露了 22 个工具，按领域分组如下：

- 租户管理：`create_tenant`、`list_tenants`
- 知识库管理：`create_knowledge_base`、`list_knowledge_bases`、`get_knowledge_base`、`delete_knowledge_base`、`hybrid_search`
- 知识管理：`create_knowledge_from_file`、`create_knowledge_from_url`、`list_knowledge`、`get_knowledge`、`delete_knowledge`
- 模型管理：`create_model`、`list_models`、`get_model`
- 会话管理：`create_session`、`get_session`、`list_sessions`、`delete_session`
- 聊天：`chat`
- Chunk 管理：`list_chunks`、`delete_chunk`

其中有两个细节值得注意：

1. README 和项目总结里通常写“21 个工具”，但当前代码实际是 22 个，因为 `create_knowledge_from_file` 已实现并已注册。
2. `WeKnoraClient` 内部还存在 `get_tenant`、`update_knowledge_base` 等方法，但没有暴露成 MCP 工具。

#### 3. 部署和配置方式

mcp-server 的部署方式与 docreader 不同，它不是 HTTP/gRPC 常驻网络服务，而是标准 MCP 的 stdio 子进程。

常见启动方式：

- `python main.py`
- `python run_server.py`
- `python run.py`
- `python weknora_mcp_server.py`
- `python -m weknora_mcp_server`
- 安装后执行 `weknora-mcp-server`
- 安装后执行 `weknora-server`

核心环境变量只有两个：

- `WEKNORA_BASE_URL`：WeKnora API 基地址，默认 `http://localhost:8080/api/v1`
- `WEKNORA_API_KEY`：调用 WeKnora 的 API Key

在 MCP 客户端中，通常通过 `uv --directory ... run run_server.py` 方式拉起该进程，并将上述环境变量一并注入。

### B2. MCP 协议实现

#### 1. 服务实现方式

`weknora_mcp_server.py` 使用官方 MCP Python SDK 的 stdio 模式：

- `Server("weknora-server")` 创建服务器实例
- `@app.list_tools()` 返回工具元数据
- `@app.call_tool()` 负责处理实际调用
- `mcp.server.stdio.stdio_server()` 建立输入输出流

因此协议实现非常典型：

- 工具元数据是静态注册的
- 每个工具入参都通过 JSON Schema 描述
- 每次调用的结果都被序列化为 JSON 文本返回给 MCP 客户端

#### 2. 暴露的 MCP 工具列表

下面按功能分组说明每个工具的用途和主要参数。

##### 租户管理

`create_tenant`

- 功能：创建租户
- 主要参数：`name`、`description`、`business`、`retriever_engines`
- 后端交互：`POST /tenants`

`list_tenants`

- 功能：列出租户
- 主要参数：无
- 后端交互：`GET /tenants`

##### 知识库管理

`create_knowledge_base`

- 功能：创建知识库
- 主要参数：`name`、`description`、`embedding_model_id`、`summary_model_id`
- 服务器侧会补入默认 `chunking_config`
- 后端交互：`POST /knowledge-bases`

`list_knowledge_bases`

- 功能：列出知识库
- 主要参数：无
- 后端交互：`GET /knowledge-bases`

`get_knowledge_base`

- 功能：获取知识库详情
- 主要参数：`kb_id`
- 后端交互：`GET /knowledge-bases/{kb_id}`

`delete_knowledge_base`

- 功能：删除知识库
- 主要参数：`kb_id`
- 后端交互：`DELETE /knowledge-bases/{kb_id}`

`hybrid_search`

- 功能：在知识库中执行混合检索
- 主要参数：`kb_id`、`query`、`vector_threshold`、`keyword_threshold`、`match_count`
- 后端交互：`GET /knowledge-bases/{kb_id}/hybrid-search`

这里有一个实现细节：代码用 `GET` 请求同时传 JSON body，这种写法在某些代理或框架下兼容性一般，但当前客户端就是这样封装的。

##### 知识管理

`create_knowledge_from_file`

- 功能：从 MCP 服务器所在机器的本地文件创建知识
- 主要参数：`kb_id`、`file_path`、`enable_multimodel`
- 后端交互：`POST /knowledge-bases/{kb_id}/knowledge/file`
- 请求方式：multipart/form-data

`create_knowledge_from_url`

- 功能：从 URL 创建知识
- 主要参数：`kb_id`、`url`、`enable_multimodel`
- 后端交互：`POST /knowledge-bases/{kb_id}/knowledge/url`

`list_knowledge`

- 功能：列出知识条目
- 主要参数：`kb_id`、`page`、`page_size`
- 后端交互：`GET /knowledge-bases/{kb_id}/knowledge`

`get_knowledge`

- 功能：获取知识详情
- 主要参数：`knowledge_id`
- 后端交互：`GET /knowledge/{knowledge_id}`

`delete_knowledge`

- 功能：删除知识
- 主要参数：`knowledge_id`
- 后端交互：`DELETE /knowledge/{knowledge_id}`

##### 模型管理

`create_model`

- 功能：创建模型配置
- 主要参数：`name`、`type`、`source`、`description`、`base_url`、`api_key`、`is_default`
- 后端交互：`POST /models`

`list_models`

- 功能：列出模型
- 主要参数：无
- 后端交互：`GET /models`

`get_model`

- 功能：获取模型详情
- 主要参数：`model_id`
- 后端交互：`GET /models/{model_id}`

##### 会话管理

`create_session`

- 功能：创建问答会话
- 主要参数：`kb_id`、`max_rounds`、`enable_rewrite`、`fallback_response`、`summary_model_id`
- 服务端会自动拼装 `session_strategy`
- 后端交互：`POST /sessions`

`get_session`

- 功能：获取会话详情
- 主要参数：`session_id`
- 后端交互：`GET /sessions/{session_id}`

`list_sessions`

- 功能：列出会话
- 主要参数：`page`、`page_size`
- 后端交互：`GET /sessions`

`delete_session`

- 功能：删除会话
- 主要参数：`session_id`
- 后端交互：`DELETE /sessions/{session_id}`

##### 聊天

`chat`

- 功能：向指定会话发送问题
- 主要参数：`session_id`、`query`
- 后端交互：`POST /knowledge-chat/{session_id}`

代码注释指出，真实后端更像返回 SSE 流，但这里被简化为一次性 HTTP 调用并返回完整结果。

##### Chunk 管理

`list_chunks`

- 功能：列出知识对应的 chunk
- 主要参数：`knowledge_id`、`page`、`page_size`
- 后端交互：`GET /chunks/{knowledge_id}`

`delete_chunk`

- 功能：删除 chunk
- 主要参数：`knowledge_id`、`chunk_id`
- 后端交互：`DELETE /chunks/{knowledge_id}/{chunk_id}`

#### 3. 与 WeKnora 后端 API 的交互方式

底层由 `WeKnoraClient` 负责统一 HTTP 请求，特点如下：

- 使用 `requests.Session` 做连接复用
- 默认请求头包含 `X-API-Key` 和 `Content-Type: application/json`
- `_request()` 统一调用 `response.raise_for_status()`
- 正常返回时统一按 JSON 解析

特殊处理只有一个：

- `create_knowledge_from_file()` 由于走文件上传，会临时移除 `Content-Type`，交给 `requests` 自动生成 multipart boundary

因此 mcp-server 不是业务逻辑实现者，而是“参数整形 + API 转发 + JSON 文本化返回”的薄适配层。

### B3. 使用场景

#### 1. 在 Claude Desktop、Cursor 等 MCP 客户端中使用

mcp-server 的典型使用场景是让本地 AI 客户端直接操作 WeKnora。

适合的场景包括：

- 在 Claude Desktop 中直接创建知识库、导入 URL、发起检索
- 在 Cursor 中把 WeKnora 当作外部知识工具使用
- 在支持 MCP 的编辑器或智能体框架中，把 WeKnora 作为知识管理后端

由于它使用 stdio 传输，所以很适合被桌面客户端按需拉起，而不需要额外暴露网络端口。

#### 2. 配置示例

当前文档推荐通过 `uv` 运行。例如 Claude Desktop / Cursor 的配置都可以写成：

```json
{
  "mcpServers": {
    "weknora": {
      "command": "uv",
      "args": [
        "--directory",
        "/path/WeKnora/mcp-server",
        "run",
        "run_server.py"
      ],
      "env": {
        "WEKNORA_API_KEY": "your_api_key_here",
        "WEKNORA_BASE_URL": "http://localhost:8080/api/v1"
      }
    }
  }
}
```

这类配置的含义是：

1. MCP 客户端启动一个本地 Python 进程。
2. 该进程通过 stdio 与客户端通信。
3. 进程内部再调用远端或本地部署的 WeKnora HTTP API。

#### 3. 典型工作流

一个典型使用链路是：

1. 通过 MCP 创建知识库
2. 从 URL 或本地文件导入知识
3. 创建聊天会话
4. 直接在 AI 客户端中执行 `hybrid_search` 或 `chat`

这种模式把知识管理操作前移到了 AI 客户端内，降低了手工切换 Web 控制台的成本。

### B4. 技术实现

#### 1. 服务框架

mcp-server 的技术实现非常轻：

- 协议框架：`mcp` Python SDK
- 运行模型：`asyncio`
- 传输层：stdio
- HTTP 客户端：`requests`
- 包装方式：既支持源码运行，也支持 pip 安装为 console script

需要注意的是，`handle_call_tool()` 虽然是异步函数，但内部调用的是同步 `requests`。这意味着它并没有真正做到异步 HTTP I/O，更像是“异步壳 + 同步转发”。

#### 2. 认证机制

认证机制非常直接：

- 从环境变量读取 `WEKNORA_API_KEY`
- 在每次请求头中写入 `X-API-Key`

此外读取：

- `WEKNORA_BASE_URL`

没有看到更复杂的认证机制，例如：

- OAuth
- Token 自动刷新
- 多租户动态凭证切换

因此它更适合受控环境下的 API Key 接入，而不是复杂企业身份集成。

#### 3. 错误处理

错误处理分为三层：

1. HTTP 请求层
   - `requests` 抛出 `RequestException`
   - `_request()` 中记录日志并继续抛出

2. 工具执行层
   - `handle_call_tool()` 捕获所有异常
   - 返回一个 `types.TextContent(type="text", text="Error executing ...")`

3. 启动层
   - `main.py` / `run_server.py` 捕获导入失败、环境检查失败和运行异常
   - `--verbose` 模式下可打印 traceback

这种错误处理方式的优点是简单；缺点是：

- 错误没有结构化分类
- 没有重试、熔断、退避
- MCP 客户端拿到的基本只是文本错误

#### 4. 工程特征与局限

从工程设计看，mcp-server 的特点是“足够薄、足够直接”，但也有一些边界：

- 它不实现 WeKnora 业务逻辑，只是协议转接层。
- 工具清单与 README 文档存在轻微不同步。
- `create_knowledge_from_file` 依赖 MCP 服务器本机可访问目标文件路径，不适合直接读取客户端本地文件。
- `chat` 把潜在的流式接口简化成普通请求，功能上可用，但不是最完整的 MCP streaming 体验。
- `requests` 同步调用放在异步 handler 中，在高并发下并不理想，但对桌面 MCP 场景通常已经够用。

#### 5. 技术实现总结

mcp-server 的设计可以概括为：

- 一个运行在本地 stdio 通道中的 Python 适配器
- 向 MCP 客户端提供静态定义的 WeKnora 工具集
- 通过 API Key 调用 WeKnora REST API
- 将 JSON 响应原样转换为 MCP 文本结果

它的价值不在于复杂算法，而在于把 WeKnora 能力以 MCP 标准接入到主流 AI 客户端生态中。

---

## 总结

这两个 Python 子服务在 WeKnora 中承担的角色不同：

- docreader 负责“内容解析”，是 Go 主服务的外部文档转换引擎。
- mcp-server 负责“协议接入”，是 AI 客户端访问 WeKnora 的 MCP 适配器。

从当前代码状态看：

- docreader 已经向“轻量解析器”收敛，很多历史上的 OCR、切块、图片存储职责被移到 Go 层或保留为扩展能力。
- mcp-server 则是一层非常薄的 MCP-to-REST 桥接，实现简单、上手快、便于桌面端集成。

如果后续继续演进，这两个方向大概率会分别聚焦于：

- docreader：更强的结构化解析与多引擎编排
- mcp-server：更完整的工具覆盖、流式输出和更强的错误语义
