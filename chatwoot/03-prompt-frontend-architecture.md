# Prompt 3: Chatwoot 前端架构分析

## 任务
分析 Chatwoot 的 Vue.js 前端架构设计，输出一份详尽的前端架构文档。

## 输出要求
将分析结果写入 `/Users/wenkai/workspace/knowledges/chatwoot/output/03-frontend-architecture.md`，使用中文撰写。

## 需要分析的文件和目录

### 整体结构
- `app/javascript/` 目录结构（第一层和第二层）
- `package.json` — 依赖列表和脚本
- `vite.config.ts` — 构建配置
- `tailwind.config.js` — 样式框架配置
- `postcss.config.js`

### 入口文件
- `app/javascript/entrypoints/` 目录 — 所有入口点，逐个阅读理解各入口的职责
  - dashboard.js/ts — 客服工作台
  - widget.js/ts — 嵌入式聊天 Widget
  - v3app.js/ts — V3 新版应用（如果存在）
  - 其他入口

### Dashboard 应用（核心 Agent 工作台）
- `app/javascript/dashboard/` 目录结构
- `app/javascript/dashboard/routes/` — 路由定义
- `app/javascript/dashboard/store/` — Vuex/Pinia 状态管理
- `app/javascript/dashboard/api/` — API 调用层
- `app/javascript/dashboard/components/` — 组件目录结构（不用深入每个组件，理解组织方式）
- `app/javascript/dashboard/composables/` — 组合式函数
- `app/javascript/dashboard/helper/` — 工具函数

### Widget 应用（终端用户聊天窗口）
- `app/javascript/widget/` 目录结构
- `app/javascript/widget/store/` — Widget 状态管理
- `app/javascript/widget/api/` — Widget API 调用

### V3 应用（如果存在）
- `app/javascript/v3/` 目录结构

### 共享模块
- `app/javascript/shared/` 目录结构 — 跨应用共享的组件、工具、composables

### SDK
- `app/javascript/sdk/` — 嵌入式 SDK（网站安装代码）

### 设计系统
- `app/javascript/design-system/` — 如果存在，了解其结构

## 输出文档结构要求

```markdown
# Chatwoot 前端架构

## 1. 技术栈与工具链
（Vue 版本、构建工具、CSS 方案、状态管理方案、路由方案）

## 2. 多应用架构
（Chatwoot 前端包含多个独立应用，各自的定位和入口）
### 2.1 Dashboard — Agent 工作台
### 2.2 Widget — 用户聊天窗口
### 2.3 V3 — 新版应用
### 2.4 Portal — 帮助中心
### 2.5 Survey — 客户满意度调查
### 2.6 SDK — 嵌入代码

## 3. Dashboard 应用详细架构
### 3.1 目录结构与模块划分
### 3.2 路由设计
### 3.3 状态管理
（Store 模块列表与职责）
### 3.4 API 层设计
### 3.5 组件组织
### 3.6 实时通信
（WebSocket 连接与消息推送）

## 4. Widget 应用架构
### 4.1 嵌入方式
### 4.2 通信机制（SDK ↔ Widget iframe）
### 4.3 状态管理

## 5. 共享层设计
### 5.1 shared 模块内容
### 5.2 Design System
### 5.3 I18n 国际化方案

## 6. 前端开发模式
（Composition API vs Options API 使用情况，组件编写约定）
```
