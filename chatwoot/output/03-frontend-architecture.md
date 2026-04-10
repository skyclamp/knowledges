# Chatwoot 前端架构

## 1. 技术栈与工具链

Chatwoot 前端是典型的“多应用单仓库”架构，核心技术栈如下：

- 框架层：Vue 3（`vue@^3.5.x`）
- 路由层：Vue Router 4（Dashboard/V3 使用 `createWebHistory`，Widget 使用 `createWebHashHistory`）
- 状态层：Vuex 4 为主，Pinia 3 渐进引入（目前 Dashboard 中有独立 Pinia store）
- 样式层：Tailwind CSS 3 + PostCSS + SCSS（并存）
- 构建层：Vite 5 + `vite-plugin-ruby`（与 Rails 资产管线融合）
- 测试层：Vitest + jsdom
- 实时通信：Rails ActionCable（`@rails/actioncable`）
- 国际化：`vue-i18n@9`，每个子应用维护独立 locale 集合

### 1.1 app/javascript 目录（第一层与第二层）

`app/javascript` 第一层是按“应用边界”拆分的：

- `dashboard/`：客服工作台主应用
- `widget/`：嵌入式访客聊天窗口
- `v3/`：新版本登录/引导应用
- `portal/`：帮助中心 Portal 前端
- `survey/`：满意度调查页面
- `sdk/`：嵌入脚本与 iframe 控制层
- `shared/`：跨应用共享组件/工具/常量/store
- `design-system/`：设计系统资源（与 Histoire 配套）
- `entrypoints/`：多入口启动文件
- `superadmin_pages/`：超管页面（独立挂载）

第二层体现分层模式，大致统一为：

- `api/`：请求封装
- `components/`：UI 组件
- `composables/`：组合式逻辑
- `helper(s)/`：工具与桥接层
- `store/`：Vuex 模块
- `routes/` 或 `views/`：路由和页面容器
- `i18n/`：语言资源

### 1.2 package.json 关键信号

- 同时存在 `vuex` 和 `pinia`，说明状态管理处于渐进迁移阶段。
- `build:sdk` 使用 `BUILD_MODE=library vite build`，说明 SDK 走单独库模式构建。
- `story:*` 脚本存在，配合 Histoire 做组件文档/预览。
- `size-limit` 对 `widget` 与 `sdk` 包体积设置阈值，体现嵌入场景下的性能约束。

### 1.3 Vite 构建配置（vite.config.ts）

核心设计是“双模式构建”：

- 常规模式：`vite-plugin-ruby` 自动收集多 entrypoints（dashboard/widget/...）
- `BUILD_MODE=library`：只打 SDK 单文件 IIFE，输出 `public/packs/js/sdk.js`

原因是 SDK 需要 `inlineDynamicImports=true`（单文件），而该选项与多入口天然冲突，因此拆成模式开关。

其他重点：

- Alias 明确跨应用 import 规范：`dashboard`、`widget`、`shared`、`v3` 等
- 测试配置直接在 Vite 中维护（Vitest 环境、coverage、inline deps）

### 1.4 Tailwind 与 PostCSS

- `tailwind.config.js`
  - 扫描范围覆盖 Dashboard/Widget/V3/Portal/Survey/Shared 与 Enterprise 视图
  - 深度扩展颜色、字体、动画、Typography（尤其消息气泡富文本）
  - 配置图标插件（多 icon collections）
- `postcss.config.js`
  - `postcss-preset-env` + `postcss-import` + `tailwindcss` + `autoprefixer`

这意味着 Chatwoot 采用“Tailwind utility-first + 少量 SCSS 补充”的组合模型。

## 2. 多应用架构

Chatwoot 前端不是单一 SPA，而是多个独立前端子应用通过多 entrypoints 并行交付。

### 2.1 Dashboard — Agent 工作台

入口：`app/javascript/entrypoints/dashboard.js`

职责：

- 挂载主工作台 `dashboard/App.vue`
- 初始化 Vue Router + Vuex + Pinia + i18n
- 注入 UI 插件（FormKit、FloatingVue、Highlight、DOMPurify）
- 初始化全局 Axios（带鉴权头）
- 启动路由守卫、埋点、Chatwoot 事件

### 2.2 Widget — 用户聊天窗口

入口：`app/javascript/entrypoints/widget.js`

职责：

- 挂载 iframe 内聊天应用
- 初始化 Vuex + Hash Router + i18n
- 建立 ActionCable 连接（`window.actionCable`）
- 与宿主页 SDK 做 postMessage 双向通信

### 2.3 V3 — 新版应用

入口：`app/javascript/entrypoints/v3app.js`

职责：

- 承载登录/SSO/注册/密码重置/引导等新版流程
- 复用 Dashboard i18n、埋点与 Sentry 初始化链路
- 使用独立路由与轻量 store（当前仅 `globalConfig`）

### 2.4 Portal — 帮助中心

入口：`app/javascript/entrypoints/portal.js`

职责：

- 以 Rails UJS + Turbolinks 为主，不是完整 SPA
- 按需挂载 Vue 组件（搜索、目录）
- 处理主题、语言切换、外链策略、锚点滚动

### 2.5 Survey — 客户满意度调查

入口：`app/javascript/entrypoints/survey.js`

职责：

- 挂载轻量 Vue 应用（`survey/App.vue`）
- 使用 Vuex + i18n
- 单页流程，面向访客回访评分场景

### 2.6 SDK — 嵌入代码

入口：`app/javascript/entrypoints/sdk.js`

职责：

- 暴露 `window.chatwootSDK.run()`
- 创建 Widget iframe 与气泡按钮
- 管理宿主页与 iframe 的通信协议
- 兼容 Turbo/Turbolinks/Astro 的 DOM 替换场景

## 3. Dashboard 应用详细架构

### 3.1 目录结构与模块划分

`dashboard/` 是最重应用，目录可分为：

- `routes/`：按业务域拆分路由（conversation/settings/contacts/...）
- `store/`：Vuex 模块化状态树（数十个模块）
- `stores/`：Pinia 试点模块（如 `calls`、`companies`）
- `api/`：按资源拆分的 REST 客户端
- `components/`：历史组件库
- `components-next/`：新一代组件体系（覆盖 Conversation/Settings/Inbox 等）
- `composables/`：组合式复用逻辑（权限、配置、快捷键、上传、翻译）
- `helper/`：基础工具、桥接层、实时连接、埋点、缓存

整体体现“业务域纵向切片 + 横向基础层复用”的组织方式。

### 3.2 路由设计

路由入口：`dashboard/routes/index.js`

关键机制：

- 顶层 `beforeEach` 先执行 `setUser`，再做鉴权与跳转
- 未登录跳转 `/app/login`
- 无 account 用户跳转 `no-accounts`
- 每次路由变化打点（Analytics）

业务路由聚合：`dashboard/routes/dashboard/dashboard.routes.js`

- 以 `accounts/:accountId` 为根容器
- 子路由来自多个域：inbox、conversation、settings、contacts、companies、captain、helpcenter、campaigns、notifications、search
- 各子路由通过 `meta.permissions`（以及部分 `featureFlag`）做访问控制

典型案例：

- 会话路由 `conversation.routes.js` 使用同一 `ConversationView`，通过 props 组合 inbox/team/label/custom_view/mention/unattended 等视图语义
- 设置路由 `settings.routes.js` 再次聚合 account/agents/integrations/reports/sla/roles/security/captain 等二级域

### 3.3 状态管理

Dashboard 主状态管理仍是 Vuex：

- 核心入口：`dashboard/store/index.js`
- 共享配置模块：`shared/store/globalConfig`

Store 模块职责可归纳为：

- 账号与身份：`auth`、`accounts`、`agents`、`teamMembers`
- 会话核心：`conversations`、`conversationPage`、`conversationSearch`、`conversationStats`、`conversationTypingStatus`
- 联系人与组织：`contacts`、`contactConversations`、`companies`（Pinia 已有）
- 收件箱与分配：`inboxes`、`inboxMembers`、`inboxAssignableAgents`、`assignmentPolicies`
- 运营配置：`automations`、`macros`、`cannedResponse`、`labels`、`customViews`
- 报表与 SLA：`reports`、`summaryReports`、`sla`、`slaReports`、`csat`
- 通知与集成：`notifications`、`integrations`、`dashboardApps`
- Help Center：`helpCenterPortals`、`helpCenterCategories`、`helpCenterArticles`
- Copilot/Captain：`captain/*`、`copilotThreads`、`copilotMessages`

并行存在的 `dashboard/stores/`（Pinia）说明团队在局部能力上已开始 Composition + Pinia 化。

### 3.4 API 层设计

Dashboard API 分层清晰：

- 基类 `ApiClient`
  - 统一 CRUD 方法（get/show/create/update/delete）
  - 支持 `accountScoped` 与 `enterprise` 路径策略
- 增强类 `CacheEnabledApiClient`
  - 通过 IndexedDB `DataManager` 做本地缓存
  - 使用 `/cache_keys` 做缓存一致性校验
  - 失效后自动回源并回写本地

资源 API 按域拆分：

- 常规业务：`conversations.js`、`contacts.js`、`reports.js`、`inboxes.js` 等
- 嵌套域：`channel/*`、`helpCenter/*`、`integrations/*`、`captain/*`

请求实例由 `dashboard/helper/APIHelper.js` 创建，会注入当前登录态 token headers。

### 3.5 组件组织

Dashboard 组件层存在两套体系：

- `components/`：传统组件集合（历史积累）
- `components-next/`：新体系，按设计原语与业务块双向组织
  - 设计原语：`button`、`input`、`dialog`、`table`、`dropdown-menu` 等
  - 业务块：`Conversation`、`Inbox`、`Settings`、`HelpCenter`、`captain` 等

这是一种“渐进重构”结构：保留旧组件可运行，同时在新功能中优先使用 `components-next`。

### 3.6 实时通信

Dashboard 实时层基于 `BaseActionCableConnector`：

- 连接 `RoomChannel`，携带 `pubsub_token`、`account_id`、`user_id`
- 定时 presence（默认 20s）
- 断线重连检测（1s 轮询）

在 `dashboard/helper/actionCable.js` 中扩展业务事件：

- 消息事件：`message.created` / `message.updated`
- 会话事件：`conversation.created` / `status_changed` / `updated` / `read`
- 协作事件：`typing_on/off`、`assignee.changed`、`mention`
- 系统事件：`page:reload`、`user:logout`、`account.cache_invalidated`
- 通知事件：`notification.created/updated/deleted`
- Copilot 事件：`copilot.message.created`

事件处理统一落到 Vuex dispatch，形成“WebSocket 事件 -> Store 变更 -> UI 更新”闭环。

## 4. Widget 应用架构

### 4.1 嵌入方式

Widget 由 SDK 在宿主页动态注入：

- 注入 iframe（`/widget?website_token=...`）
- 注入气泡按钮与样式（SDK 内置 CSS）
- 挂载位置支持 left/right
- 支持标准/flat 样式和展开型 launcher

### 4.2 通信机制（SDK ↔ Widget iframe）

通信协议核心：

- 消息前缀：`chatwoot-widget:`
- SDK -> Widget：`IFrameHelper.sendMessage(...)` 通过 `contentWindow.postMessage`
- Widget -> SDK：`widget/helpers/utils.js` 的 `window.parent.postMessage`

双向事件包含：

- 生命周期：`chatwoot:ready`
- 消息：`chatwoot:on-message`
- 错误：`chatwoot:error`
- 业务回调：`chatwoot:postback`
- UI 控制：如 `sdk-set-bubble-visibility`

SDK 同时提供 JS API：

- 会话控制：`toggle`、`reset`、`popoutChatWindow`
- 用户态：`setUser`
- 属性同步：`setCustomAttributes`、`setConversationCustomAttributes`
- 展示控制：`toggleBubbleVisibility`、`setLocale`、`setColorScheme`

### 4.3 状态管理

Widget 采用 Vuex 模块化：

- `conversation`、`message`：会话与消息主链路
- `appConfig`：UI 配置与路由过渡状态
- `agent`、`contacts`：客服在线态与联系人数据
- `campaign`、`articles`：营销与帮助内容
- `conversationAttributes`、`conversationLabels`、`events`
- 共享：`globalConfig`

路由层采用 Hash History，且全局 before/after guard 将“路由切换中”状态写入 store，防止慢网下重复提交。

## 5. 共享层设计

### 5.1 shared 模块内容

`shared/` 是跨应用公共层，包含：

- `components/`：通用组件（如 FluentIcon、PhoneInput、EmojiInput、Chart）
- `composables/`：品牌替换、格式化、筛选、多语言工具
- `helpers/`：日期、文件、校验、事件总线、sanitize、ActionCable 基类
- `constants/`：事件常量、国家语言、消息类型、SDK frame 事件
- `store/`：`globalConfig` 跨应用共享状态

这是 Chatwoot 多应用复用的关键基础设施。

### 5.2 Design System

`app/javascript/design-system/` 目前以资源文件为主（logo、histoire.scss），设计系统运行依赖 Histoire：

- `histoire.config.ts` 配置了 `@histoire/plugin-vue`
- UI 品牌标题为 `@chatwoot/design`
- 组件树分组、默认预览布局、主题资源都已配置

实践上，真正的大规模组件沉淀在 `dashboard/components-next/`，可视为“设计系统实现层”。

### 5.3 I18n 国际化方案

多应用 i18n 采用统一模式：

- 各应用维护 `i18n/index.js`，静态导入 locale 文件并导出 messages 对象
- 入口文件创建 `createI18n({ locale: 'en', messages })`
- 运行时可基于配置或 SDK 指令切换 locale

Dashboard locale 文件为 JS 模块，Widget/Survey 多为 JSON；两者并存但使用方式统一。

## 6. 前端开发模式

### 6.1 Composition API vs Options API

从代码统计看：

- `<script setup>` 约 787 处
- `export default`（Vue SFC 内）约 229 处

说明当前主流已是 Composition API，但仍保留不少 Options API 页面（尤其历史模块与 V3/Survey 部分组件）。

### 6.2 组件与代码组织约定

- 路由、Store、API 均按业务域拆分，避免大一统文件
- 组合逻辑优先沉淀到 `composables/`
- 跨应用复用逻辑放入 `shared/`
- Dashboard 正在推进 `components-next` + Tailwind 的新 UI 体系
- 状态层保持“Vuex 稳定主干 + Pinia 渐进迁移”策略

### 6.3 架构结论

Chatwoot 前端是一个成熟的多应用生态：

- 通过多 entrypoints 与 Rails 深度耦合，适配不同产品面（工作台、访客 Widget、Portal、Survey、V3）
- 通过 shared 层和统一工具链保持横向一致性
- 通过 Vuex->Pinia、components->components-next 的双轨机制实现低风险演进
- 通过 SDK + iframe + postMessage + ActionCable 构建了“嵌入式实时客服”完整闭环
