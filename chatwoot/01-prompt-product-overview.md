# Prompt 1: Chatwoot 产品概述与核心概念分析

## 任务
分析 Chatwoot 源代码，输出一份产品概述文档，涵盖产品定位、核心功能全景、核心领域概念。

## 输出要求
将分析结果写入 `/Users/wenkai/workspace/knowledges/chatwoot/output/01-product-overview.md`，使用中文撰写。

## 需要分析的文件和目录

### 产品定位与功能概述
- `README.md` — 官方产品介绍
- `app.json` — Heroku 部署清单（含功能插件列表）
- `config/features.yml` — 功能特性标志定义
- `config/installation_config.yml` — 安装配置项

### 核心领域模型（重点阅读）
以下模型文件定义了 Chatwoot 的核心业务概念，阅读每个文件理解其属性和关联关系：
- `app/models/account.rb` — 租户/账户（多租户根节点）
- `app/models/user.rb` — 用户（客服Agent）
- `app/models/account_user.rb` — 账户-用户关联（角色、可用性）
- `app/models/contact.rb` — 联系人（终端客户）
- `app/models/contact_inbox.rb` — 联系人-收件箱关联
- `app/models/inbox.rb` — 收件箱（渠道入口）
- `app/models/inbox_member.rb` — 收件箱成员
- `app/models/conversation.rb` — 会话（核心业务对象）
- `app/models/message.rb` — 消息
- `app/models/attachment.rb` — 附件
- `app/models/team.rb` — 团队
- `app/models/label.rb` — 标签
- `app/models/agent_bot.rb` — Agent 机器人
- `app/models/agent_bot_inbox.rb` — 机器人-收件箱关联
- `app/models/webhook.rb` — Webhook 配置
- `app/models/automation_rule.rb` — 自动化规则
- `app/models/campaign.rb` — 运营活动
- `app/models/canned_response.rb` — 快捷回复模板
- `app/models/notification.rb` — 通知
- `app/models/custom_attribute_definition.rb` — 自定义属性定义
- `app/models/assignment_policy.rb` — 分配策略

### 渠道模型（概览即可，不用深入）
- `app/models/channel/` 目录下所有文件 — 理解支持哪些渠道

### 数据库 Schema
- `db/schema.rb` — 浏览前200行，理解表结构的整体规模和命名风格

### Enterprise 扩展
- `enterprise/app/models/` 目录 — 快速浏览，了解 Enterprise 额外提供了哪些模型

## 输出文档结构要求

```markdown
# Chatwoot 产品概述

## 1. 产品定位
（Chatwoot 是什么、解决什么问题、目标用户）

## 2. 核心功能全景
（按类别列出所有主要功能，每个功能一句话描述）
- 全渠道消息
- 团队协作
- 自动化
- AI 能力
- 知识库/帮助中心
- 报表分析
- 集成

## 3. 核心领域概念
（画出概念关系，每个概念说明其职责和关键属性）

### 3.1 多租户模型
### 3.2 用户与角色体系
### 3.3 Inbox 与 Channel 模型
### 3.4 Conversation 生命周期
### 3.5 Message 模型
### 3.6 Contact 模型
### 3.7 自动化与规则引擎
### 3.8 Agent Bot 模型

## 4. OSS vs Enterprise 功能边界
（哪些功能是开源版有的，哪些是 Enterprise 专属的）

## 5. 技术栈总结
（后端、前端、数据库、缓存、消息队列、搜索等）
```
