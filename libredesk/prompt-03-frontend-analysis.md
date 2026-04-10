# Prompt 03: 前端架构与页面分析

## 任务

请分析 libredesk 前端代码（`frontend/` 目录），输出一份**前端架构与页面设计文档**。

## 需要分析的文件和目录

1. `frontend/package.json` — 依赖和脚本
2. `frontend/vite.config.js` — 构建配置
3. `frontend/src/main.js` — 入口
4. `frontend/src/App.vue` / `OuterApp.vue` / `Root.vue` — 顶层组件
5. `frontend/src/router/` — 路由定义
6. `frontend/src/stores/` — 状态管理（Pinia）
7. `frontend/src/api/` — API 调用层
8. `frontend/src/views/` — 页面视图（递归查看各子目录）
9. `frontend/src/features/` — 功能模块
10. `frontend/src/components/` — 通用组件
11. `frontend/src/composables/` — 组合式函数
12. `frontend/src/websocket.js` — WebSocket 客户端
13. `frontend/src/constants/` — 常量定义

## 输出要求

写入文件 `/Users/wenkai/workspace/knowledges/libredesk/03-frontend-analysis.md`，包含以下章节：

### 1. 技术栈与工具链
- Vue 3 版本、构建工具、UI 组件库
- 状态管理方案
- HTTP 客户端
- 主要第三方依赖

### 2. 应用结构
- 路由设计：列出所有路由路径和对应组件
- 布局层级：Root → App → Layout → View
- 认证流程：登录、token 管理

### 3. 页面功能清单
对 `views/` 和 `features/` 下的每个模块：
- 页面名称和 URL 路径
- 核心功能描述
- 使用的 store 和 API

### 4. 状态管理设计
- 每个 store 的职责和关键 state/actions
- store 之间的关系

### 5. API 调用层
- API 基础配置
- 错误处理模式
- 请求/响应拦截器

### 6. 实时通信
- WebSocket 连接管理
- 事件类型和处理方式
- 哪些页面/功能使用了 WebSocket

### 7. 组件设计模式
- 通用组件库的组织方式
- composables 的使用模式
