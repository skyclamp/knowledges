# Chatwoot 源码分析 Prompt 索引

## 目标
通过分步分析 Chatwoot 源代码，输出完整的技术文档，涵盖产品设计、架构设计和实现细节。

## Prompt 清单

| # | Prompt 文件 | 输出文档 | 分析范围 |
|---|------------|---------|---------|
| 1 | `01-prompt-product-overview.md` | `output/01-product-overview.md` | 产品定位、功能全景、核心概念 |
| 2 | `02-prompt-backend-architecture.md` | `output/02-backend-architecture.md` | Rails 后端架构、数据模型、路由设计 |
| 3 | `03-prompt-frontend-architecture.md` | `output/03-frontend-architecture.md` | Vue 前端架构、模块划分、状态管理 |
| 4 | `04-prompt-channel-integration.md` | `output/04-channel-integration.md` | 全渠道集成设计（含 Telegram, WhatsApp） |
| 5 | `05-prompt-deployment-devops.md` | `output/05-deployment-devops.md` | 部署架构、Docker、本地开发环境搭建 |
| 6 | `06-prompt-ai-agent-captain.md` | `output/06-ai-agent-captain.md` | Captain AI、AgentBot、自动化规则 |
| 7 | `07-prompt-api-webhook-extensibility.md` | `output/07-api-webhook-extensibility.md` | REST API、Webhook、Platform API、扩展能力 |

## 执行方式
1. 按顺序在 VS Code Copilot Chat 中执行每个 Prompt
2. 每个 Prompt 的输出保存到 `output/` 目录下对应文件
3. 完成后由 Opus 进行 Review
