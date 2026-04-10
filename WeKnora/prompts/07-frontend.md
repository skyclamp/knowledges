# Prompt 07: 前端架构分析

## 任务

分析 WeKnora 的 Vue.js 前端架构、页面组织、状态管理、API 通信、组件设计，输出前端架构设计文档。

## 需要阅读的文件

1. **项目配置**:
   - `/Users/wenkai/oss/WeKnora/frontend/package.json` — 依赖和脚本
   - `/Users/wenkai/oss/WeKnora/frontend/vite.config.ts` — 构建配置
   - `/Users/wenkai/oss/WeKnora/frontend/tsconfig.json` — TypeScript 配置
   - `/Users/wenkai/oss/WeKnora/frontend/index.html` — 入口 HTML

2. **应用入口**:
   - `/Users/wenkai/oss/WeKnora/frontend/src/main.ts` — 应用入口
   - `/Users/wenkai/oss/WeKnora/frontend/src/App.vue` — 根组件

3. **路由**:
   - `/Users/wenkai/oss/WeKnora/frontend/src/router/` — 路由配置

4. **页面视图**:
   - `/Users/wenkai/oss/WeKnora/frontend/src/views/` — 整个目录（递归查看子目录结构）
   - 子目录包括：`auth/`, `chat/`, `creatChat/`, `knowledge/`, `organization/`, `platform/`, `settings/`, `agent/`

5. **状态管理**:
   - `/Users/wenkai/oss/WeKnora/frontend/src/stores/` — Pinia stores

6. **API 通信**:
   - `/Users/wenkai/oss/WeKnora/frontend/src/api/` — API 请求封装

7. **组件**:
   - `/Users/wenkai/oss/WeKnora/frontend/src/components/` — 公共组件

8. **Composables/Hooks**:
   - `/Users/wenkai/oss/WeKnora/frontend/src/composables/` — 组合式函数
   - `/Users/wenkai/oss/WeKnora/frontend/src/hooks/` — 自定义 hooks

9. **类型定义**:
   - `/Users/wenkai/oss/WeKnora/frontend/src/types/` — TypeScript 类型

10. **国际化**:
    - `/Users/wenkai/oss/WeKnora/frontend/src/i18n/` — 多语言支持

11. **工具函数**:
    - `/Users/wenkai/oss/WeKnora/frontend/src/utils/` — 前端工具

12. **前端子包**（如有）:
    - `/Users/wenkai/oss/WeKnora/frontend/packages/` — monorepo 子包

## 输出要求

写一份中文文档，保存到 `/Users/wenkai/workspace/knowledges/WeKnora/docs/07-前端架构.md`，包含以下章节：

### 1. 技术栈
- Vue 3 + TypeScript + Vite 的版本
- UI 组件库
- 状态管理方案
- 其他关键前端依赖

### 2. 应用架构
- 目录结构说明
- 模块划分方式
- 路由结构（用树形结构展示所有路由）

### 3. 页面功能清单
对每个页面/视图模块：
- 功能说明
- 主要组件组成
- 关键交互

### 4. 状态管理
- Store 列表及各自管理的状态
- Store 之间的关系
- 持久化策略

### 5. API 通信层
- HTTP 客户端配置（拦截器、基础 URL、错误处理）
- API 模块划分
- SSE 流式通信的处理方式
- 认证 Token 管理

### 6. 聊天界面设计
- 对话界面的组件组成
- 消息渲染（Markdown、代码高亮、图片等）
- 流式消息的实时显示
- Agent 推理过程的展示

### 7. 知识库管理界面
- 知识库列表和详情
- 文档上传和管理
- 分块预览
- 搜索测试

### 8. 公共组件
- 关键公共组件列表及用途
- 组件设计模式

### 9. 国际化
- 支持的语言
- 国际化实现方式
- 翻译文件组织

### 10. 主题/样式
- 主题方案（亮/暗模式）
- CSS 方案（Tailwind/SCSS/其他）
- 设计规范
