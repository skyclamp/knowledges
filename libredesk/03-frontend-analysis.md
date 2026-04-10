# Libredesk 前端架构与页面设计文档

## 1. 技术栈与工具链

### 1.1 核心框架与构建
- 前端框架: Vue 3 (`vue@^3.4.37`)
- 路由: `vue-router@^4.2.5`
- 构建工具: Vite 5 (`vite@^5.4.21`, `@vitejs/plugin-vue`)
- 样式体系:
  - Tailwind CSS 3 + PostCSS + Autoprefixer
  - Sass (`main.scss`)
  - `tailwindcss-animate`
- 国际化: `vue-i18n@9`

### 1.2 UI 组件体系
- 基础 UI: 以 `radix-vue` + `reka-ui` 为底层能力，封装在 `src/components/ui/*`
- 设计辅助:
  - `class-variance-authority` (`cva`) 变体样式
  - `clsx` + `tailwind-merge` class 组合
- 图标: `lucide-vue-next` + `@radix-icons/vue`
- 富文本编辑: TipTap (`@tiptap/vue-3` 及相关扩展)
- 表格: `@tanstack/vue-table`
- 图表: `@unovis/ts` + `@unovis/vue`

### 1.3 状态管理方案
- 全局状态: Pinia (`pinia@^2.1.7`)
- 主要 store:
  - `user`, `appSettings`, `conversation`, `notification`
  - `inbox`, `users`, `team`, `sla`, `tag`, `customAttributes`, `macro`, `sharedView`

### 1.4 HTTP 客户端
- 客户端: Axios (`axios@^1.13.5`)
- 参数序列化: `qs`
- 请求基线:
  - timeout = 10s
  - 自动附加 CSRF Token (`csrf_token` cookie -> `X-CSRFTOKEN` header)
  - POST/PUT 默认 `application/json`
  - 表单场景使用 `application/x-www-form-urlencoded` 时自动 `qs.stringify`

### 1.5 测试与工程化
- 单元测试: Vitest
- E2E/组件测试: Cypress
- 代码质量: ESLint + Prettier
- 包管理: pnpm 9

---

## 2. 应用结构

### 2.1 启动与初始化流程
`src/main.js` 启动流程:
1. 调用 `api.getConfig()` 获取公共配置
2. 根据配置加载语言包 `api.getLanguage(lang)` 并创建 i18n
3. 创建 app + Pinia + Router
4. 初始化 `appSettings` store（写入 public config，并拉取 general settings）
5. 注入全局事件总线（mitt）并 mount

### 2.2 布局层级
运行时布局链路:
- `Root.vue`: 全局 `TooltipProvider` + `Toaster` + 顶层 `RouterView`
- `OuterApp.vue`: 登录/重置密码等匿名页面容器
- `App.vue`: 已登录主壳（左侧图标栏 + 主侧边栏 + 顶部 Header + 内容区），并在这里初始化 WebSocket 和全局数据
- 业务 Layout:
  - `InboxLayout.vue`: 会话列表/详情双栏布局
  - `AccountLayout.vue`: 账户中心布局
  - `AdminLayout.vue`: 管理后台布局
- 最终页面: `views/*`

可概括为:
`Root -> (OuterApp | App) -> (Layout) -> View`

### 2.3 路由设计（完整路径与组件）

#### 认证与公共
- `/` -> `views/auth/UserLoginView.vue`
- `/reset-password` -> `views/auth/ResetPasswordView.vue`
- `/set-password` -> `views/auth/SetPasswordView.vue`

#### 主业务
- `/contacts` -> `views/contact/ContactsView.vue`
- `/contacts/:id` -> `views/contact/ContactDetailView.vue`
- `/reports` -> redirect `/reports/overview`
- `/reports/overview` -> `views/reports/OverviewView.vue`
- `/inboxes/search` -> `views/search/SearchView.vue`

#### 收件箱（含嵌套路由）
- `/inboxes/:type(assigned|unassigned|all|mentioned)?` -> `layouts/inbox/InboxLayout.vue` -> `views/inbox/InboxView.vue`
- `/inboxes/:type/conversation/:uuid` -> `views/conversation/ConversationDetailView.vue`
- `/inboxes/teams/:teamID` -> `layouts/inbox/InboxLayout.vue` -> `views/inbox/InboxView.vue`
- `/inboxes/teams/:teamID/conversation/:uuid` -> `views/conversation/ConversationDetailView.vue`
- `/inboxes/views/:viewID` -> `layouts/inbox/InboxLayout.vue` -> `views/inbox/InboxView.vue`
- `/inboxes/views/:viewID/conversation/:uuid` -> `views/conversation/ConversationDetailView.vue`

#### 账户
- `/account/:page?` -> redirect `/account/profile`
- `/account/profile` -> `views/account/profile/ProfileEditView.vue`

#### 管理台
- `/admin` -> `layouts/admin/AdminLayout.vue`
- `/admin/custom-attributes` -> `views/admin/custom-attributes/CustomAttributes.vue`
- `/admin/general` -> `views/admin/general/General.vue`
- `/admin/business-hours` -> `views/admin/business-hours/BusinessHours.vue` +
  - `/admin/business-hours` -> `BusinessHoursList.vue`
  - `/admin/business-hours/new` -> `CreateOrEditBusinessHours.vue`
  - `/admin/business-hours/:id/edit` -> `CreateOrEditBusinessHours.vue`
- `/admin/sla` -> `views/admin/sla/SLA.vue` +
  - `/admin/sla` -> `SLAList.vue`
  - `/admin/sla/new` -> `CreateEditSLA.vue`
  - `/admin/sla/:id/edit` -> `CreateEditSLA.vue`
- `/admin/inboxes` -> `views/admin/inbox/InboxView.vue` +
  - `/admin/inboxes` -> `InboxList.vue`
  - `/admin/inboxes/new` -> `NewInbox.vue`
  - `/admin/inboxes/:id/edit` -> `EditInbox.vue`
- `/admin/notification` -> `features/admin/notification/NotificationSetting.vue`
- `/admin/teams/agents` -> `views/admin/agents/Agents.vue` +
  - `/admin/teams/agents` -> `AgentList.vue`
  - `/admin/teams/agents/new` -> `CreateAgent.vue`
  - `/admin/teams/agents/:id/edit` -> `EditAgent.vue`
- `/admin/teams/teams` -> `views/admin/teams/Teams.vue` +
  - `/admin/teams/teams` -> `TeamList.vue`
  - `/admin/teams/teams/new` -> `CreateTeamForm.vue`
  - `/admin/teams/teams/:id/edit` -> `EditTeamForm.vue`
- `/admin/teams/roles` -> `views/admin/roles/Roles.vue` +
  - `/admin/teams/roles` -> `RoleList.vue`
  - `/admin/teams/roles/new` -> `NewRole.vue`
  - `/admin/teams/roles/:id/edit` -> `EditRole.vue`
- `/admin/teams/activity-log` -> `views/admin/activity-log/ActivityLog.vue`
- `/admin/automations` -> `views/admin/automations/Automation.vue`
  - `/admin/automations/new` -> `CreateOrEditRule.vue`
  - `/admin/automations/:id/edit` -> `CreateOrEditRule.vue`
- `/admin/templates` -> `views/admin/templates/Templates.vue`
  - `/admin/templates/new` -> `CreateEditTemplate.vue`
  - `/admin/templates/:id/edit` -> `CreateEditTemplate.vue`
- `/admin/sso` -> `views/admin/oidc/OIDC.vue`
  - `/admin/sso` -> `OIDCList.vue`
  - `/admin/sso/new` -> `CreateEditOIDC.vue`
  - `/admin/sso/:id/edit` -> `CreateEditOIDC.vue`
- `/admin/webhooks` -> `views/admin/webhooks/Webhooks.vue`
  - `/admin/webhooks` -> `WebhookList.vue`
  - `/admin/webhooks/new` -> `CreateEditWebhook.vue`
  - `/admin/webhooks/:id/edit` -> `CreateEditWebhook.vue`
- `/admin/conversations/tags` -> `views/admin/tags/TagsView.vue`
- `/admin/conversations/statuses` -> `views/admin/status/StatusView.vue`
- `/admin/conversations/macros` -> `views/admin/macros/Macros.vue` +
  - `/admin/conversations/macros` -> `MacroList.vue`
  - `/admin/conversations/macros/new` -> `CreateMacro.vue`
  - `/admin/conversations/macros/:id/edit` -> `EditMacro.vue`
- `/admin/conversations/shared-views` -> `views/admin/shared-views/SharedViews.vue` +
  - `/admin/conversations/shared-views` -> `SharedViewList.vue`
  - `/admin/conversations/shared-views/new` -> `CreateSharedView.vue`
  - `/admin/conversations/shared-views/:id/edit` -> `EditSharedView.vue`

#### 兜底
- `/:pathMatch(.*)*` -> redirect `/inboxes/assigned`

### 2.4 认证流程（登录与 token）
- 登录页面调用 `api.login(email, password)`，成功后写入 `userStore` 并跳转 inbox。
- 支持 OIDC 登录: 前端跳转 `/api/v1/oidc/{provider.id}/login`（可带 `next` 参数）。
- 密码重置: `/reset-password` 发起重置；`/set-password?token=...` 提交新密码。
- token 管理模型:
  - 前端未使用 localStorage 保存 Bearer Token。
  - 以 Cookie 会话为主，CSRF Token 从 cookie 提取并在请求头携带 `X-CSRFTOKEN`。

---

## 3. 页面功能清单（views + features）

说明:
- URL 路径来自 router；features 通常是嵌入式功能组件，无独立 URL。
- Store/API 为代码中直接引用的主要依赖。

### 3.1 views 模块

| 模块 | 页面/路径 | 核心功能 | 主要 Store | 主要 API |
|---|---|---|---|---|
| `views/auth` | `/`, `/reset-password`, `/set-password` | 账号登录、OIDC 跳转、重置/设置密码 | `useUserStore`, `useAppSettingsStore` | `login`, `resetPassword`, `setPassword` |
| `views/inbox` | `/inboxes/*`, `/inboxes/teams/:teamID`, `/inboxes/views/:viewID` | 收件箱列表容器（会话列表 + 会话详情路由出口） | `useConversationStore` | 主要通过 store 间接调用 |
| `views/conversation` | `/inboxes/*/conversation/:uuid` | 会话详情页，承载消息流与侧边信息 | `useConversationStore` | 主要通过 store 间接调用 |
| `views/search` | `/inboxes/search` | 全局搜索（会话/消息） | - | `searchConversations`, `searchMessages` |
| `views/contact` | `/contacts`, `/contacts/:id` | 联系人列表、详情、封禁、编辑资料 | `useUserStore` | `getContact`, `updateContact`, `blockContact` |
| `views/reports` | `/reports/overview` | 运营报表总览图表 | - | `getOverviewCounts`, `getOverviewSLA`, `getOverviewCharts`, `getOverviewCSAT`, `getOverviewMessageVolume`, `getOverviewTagDistribution` |
| `views/account` | `/account/profile` | 当前用户资料修改、头像管理 | `useUserStore` | `updateCurrentUser`, `deleteUserAvatar` |
| `views/admin/general` | `/admin/general` | 工作区通用设置 | `useAppSettingsStore` | `updateSettings` |
| `views/admin/custom-attributes` | `/admin/custom-attributes` | 自定义字段管理 | - | `getCustomAttributes`, `createCustomAttribute`, `updateCustomAttribute` |
| `views/admin/business-hours` | `/admin/business-hours/*` | 营业时间策略管理 | - | `getAllBusinessHours`, `getBusinessHours`, `createBusinessHours`, `updateBusinessHours` |
| `views/admin/sla` | `/admin/sla/*` | SLA 策略 CRUD | - | `getAllSLAs`, `getSLA`, `createSLA`, `updateSLA` |
| `views/admin/inbox` | `/admin/inboxes/*` | Inbox 渠道配置、启停、OAuth 回调处理 | `useInboxStore` | `getInbox`, `createInbox`, `updateInbox`, `deleteInbox`, `toggleInbox` |
| `views/admin/agents` | `/admin/teams/agents/*` | Agent 列表、创建编辑、导入 | `useUsersStore` | `getUser`, `createUser`, `updateUser`, `importAgents`, `getAgentImportStatus` |
| `views/admin/teams` | `/admin/teams/teams/*` | Team 管理 | - | `getTeams`, `getTeam`, `createTeam`, `updateTeam` |
| `views/admin/roles` | `/admin/teams/roles/*` | 角色与权限管理 | - | `getRoles`, `getRole`, `createRole`, `updateRole` |
| `views/admin/activity-log` | `/admin/teams/activity-log` | 操作日志页（展示层） | - | 由 feature 拉取 |
| `views/admin/automations` | `/admin/automations/*` | 自动化规则创建编辑 | - | `getAutomationRule`, `createAutomationRule`, `updateAutomationRule` |
| `views/admin/templates` | `/admin/templates/*` | 模板管理 | - | `getTemplates`, `getTemplate`, `createTemplate`, `updateTemplate` |
| `views/admin/oidc` | `/admin/sso/*` | SSO/OIDC Provider 管理 | - | `getAllOIDC`, `getOIDC`, `createOIDC`, `updateOIDC` |
| `views/admin/webhooks` | `/admin/webhooks/*` | Webhook 管理、测试 | - | `getWebhooks`, `getWebhook`, `createWebhook`, `updateWebhook`, `testWebhook` |
| `views/admin/status` | `/admin/conversations/statuses` | 会话状态字典管理 | - | `getStatuses`, `createStatus` |
| `views/admin/tags` | `/admin/conversations/tags` | 标签管理 | - | `getTags`, `createTag` |
| `views/admin/macros` | `/admin/conversations/macros/*` | 宏管理 | `useMacroStore` | `getAllMacros`, `getMacro`, `createMacro`, `updateMacro` |
| `views/admin/shared-views` | `/admin/conversations/shared-views/*` | 共享视图管理 | `useSharedViewStore` | `getAllSharedViews`, `getSharedView`, `createSharedView`, `updateSharedView` |

### 3.2 features 模块

| 模块 | URL 路径 | 核心功能 | 主要 Store | 主要 API |
|---|---|---|---|---|
| `features/conversation` | 嵌入 `/inboxes/*` & 会话详情页 | 回复框、消息渲染、侧边栏、创建会话、宏应用、AI 回复、附件 | `useConversationStore`, `useUserStore`, `useUsersStore`, `useTeamStore`, `useInboxStore`, `useMacroStore`, `useTagStore`, `useCustomAttributeStore`, `useAppSettingsStore` | `sendMessage`, `retryMessage`, `createConversation`, `applyMacro`, `searchContacts`, `getAiPrompts`, `aiCompletion`, `updateAIProvider`, `updateContactCustomAttribute`, `updateConversationCustomAttribute` |
| `features/contact` | 嵌入 `/contacts*` | 联系人列表、联系人备注增删、联系人表单 | `useUserStore` | `getContacts`, `getContactNotes`, `createContactNote`, `deleteContactNote` |
| `features/search` | 嵌入 `/inboxes/search` | 搜索头与结果展示 | - | 由 view 触发搜索 API |
| `features/reports` | 嵌入 `/reports/overview` | 统计卡片、柱状图、折线图组件 | - | 由 view 拉取数据 |
| `features/command` | 全局（`App.vue` 内） | 命令面板，快捷操作会话与宏 | `useConversationStore`, `useMacroStore` | - |
| `features/view` | 全局（用户视图弹窗） | 新建/编辑个人视图过滤器 | - | `createView`, `updateView` |
| `features/sla` | 多处复用 | SLA 徽章/状态展示 | - | - |
| `features/admin/activity-log` | `/admin/teams/activity-log` | 日志筛选与列表 | - | `getActivityLogs` |
| `features/admin/agents` | `/admin/teams/agents/*` | Agent 表单、删除、API Key 管理 | - | `deleteUser`, `generateAPIKey`, `revokeAPIKey`, `getTeams`, `getRoles` |
| `features/admin/automation` | `/admin/automations*` | 自动化规则列表、拖拽排序、开关、动作配置 | `useTagStore` | `getAutomationRules`, `toggleAutomationRule`, `deleteAutomationRule`, `updateAutomationRuleWeights`, `updateAutomationRulesExecutionMode` |
| `features/admin/business-hours` | `/admin/business-hours*` | 营业时间表单与表格动作 | - | `deleteBusinessHours` |
| `features/admin/custom-attributes` | `/admin/custom-attributes` | 自定义字段表单与表格动作 | - | `deleteCustomAttribute` |
| `features/admin/general` | `/admin/general` | 通用设置表单 | - | `getAllBusinessHours` |
| `features/admin/inbox` | `/admin/inboxes*` | 邮箱 inbox 表单，支持 OAuth 发起与重连 | `useAppSettingsStore` | `initiateOAuthFlow` |
| `features/admin/macros` | `/admin/conversations/macros*` | 宏表单、动作编辑器、表格动作 | `useMacroStore`, `useUsersStore`, `useTeamStore`, `useTagStore` | `deleteMacro` |
| `features/admin/notification` | `/admin/notification` | 邮件通知设置表单 | `useAppSettingsStore` | `getEmailNotificationSettings`, `updateEmailNotificationSettings` |
| `features/admin/oidc` | `/admin/sso*` | OIDC 表单与表格动作 | - | `deleteOIDC` |
| `features/admin/roles` | `/admin/teams/roles*` | 角色表单与删除动作 | - | `deleteRole` |
| `features/admin/shared-views` | `/admin/conversations/shared-views*` | 共享视图表单与删除动作 | `useSharedViewStore`, `useTeamStore` | `deleteSharedView` |
| `features/admin/sla` | `/admin/sla*` | SLA 表单与删除动作 | `useUsersStore` | `deleteSLA` |
| `features/admin/status` | `/admin/conversations/statuses` | 状态表单、编辑、删除 | - | `updateStatus`, `deleteStatus` |
| `features/admin/tags` | `/admin/conversations/tags` | 标签编辑与删除 | - | `updateTag`, `deleteTag` |
| `features/admin/teams` | `/admin/teams/teams*` | Team 表单、删除、关联 SLA/Business Hours | `useSlaStore` | `getAllBusinessHours`, `deleteTeam` |
| `features/admin/templates` | `/admin/templates*` | 模板表单与删除 | - | `deleteTemplate` |
| `features/admin/webhooks` | `/admin/webhooks*` | Webhook 列表动作（开关/测试/删除）与表单 | - | `toggleWebhook`, `testWebhook`, `deleteWebhook` |

---

## 4. 状态管理设计

### 4.1 Store 职责与关键 state/actions

- `user`
  - state: 当前用户信息（id、权限、团队、角色、availability）
  - actions: `getCurrentUser`, `setCurrentUser`, `updateUserAvailability`, `setAvatar`
  - 特点: 使用 `useStorage` 同步 availability（跨标签页）

- `appSettings`
  - state: `settings`, `public_config`
  - actions: `fetchSettings`, `fetchPublicConfig`, `setSettings`, `setPublicConfig`

- `conversation`
  - state:
    - 会话列表（分页、筛选、排序、总数）
    - 当前会话与参与者
    - 消息缓存 `MessageCache`
    - 状态/优先级选项、宏上下文、草稿 Map
  - actions:
    - 列表与详情: `fetchConversationsList`, `fetchConversation`, `fetchMessages`
    - 会话操作: `updateStatus`, `updatePriority`, `updateAssignee`, `upsertTags`, `markAsUnread`
    - 增量更新: `updateConversationList`, `updateConversationMessage`, `updateMessageProp`, `updateConversationProp`
    - 草稿: `fetchAllDrafts`, `getDraft`, `setDraft`, `removeDraft`

- `notification`
  - state: 列表、未读数、总数、加载状态、分页是否还有更多
  - actions: `fetchNotifications`, `fetchStats`, `markAsRead`, `markAllAsRead`, `deleteNotification`, `deleteAll`, `addNotification`

- `inbox` / `users` / `team` / `sla` / `tag` / `customAttributes`
  - 职责: 拉取并缓存下拉选项数据，供表单和过滤器复用

- `macro`
  - state: `macroList`, `currentView`
  - actions: `loadMacros`, `setCurrentView`
  - 特点: 按用户权限和可见范围过滤可执行宏

- `sharedView`
  - state: `sharedViewList`, `isLoaded`
  - actions: `loadSharedViews`, `refresh`, `reset`

### 4.2 Store 之间关系
- `App.vue` 启动后并行初始化多个 store（用户、会话元数据、teams/inboxes/users、SLA、宏、标签、自定义字段、shared views）。
- `conversation` 是业务核心 store，依赖 API 最多，并被以下模块消费:
  - Inbox 列表
  - 会话详情
  - 回复框/消息流/命令面板
- `macro` 依赖 `user` 的权限判断来裁剪可用宏动作。
- `notification` 通过 WebSocket 事件增量更新（`addNotification`）。

---

## 5. API 调用层

### 5.1 基础配置
- 文件: `src/api/index.js`
- 实例: `axios.create({ timeout: 10000, responseType: 'json' })`
- 路径约定: 全部 `/api/v1/*`
- 覆盖域: 认证、会话、消息、联系人、团队、角色、宏、模板、自动化、报表、通知、视图、Webhook、OIDC 等

### 5.2 错误处理模式
- API 层本身不做统一业务错误弹窗。
- 组件/store 侧统一通过 `handleHTTPError` 将 Axios 错误标准化，再配合 emitter -> toaster 展示。
- 常见模式:
  - `try/catch + emitter.emit(SHOW_TOAST, { variant: 'destructive', description })`
  - 或局部 `errorMessage` 绑定表单错误区域

### 5.3 拦截器
- 已实现请求拦截器:
  - 自动读取 cookie 中 `csrf_token`
  - 设置 `X-CSRFTOKEN`
  - 补全请求 `Content-Type`
  - 对表单 URL Encoded 自动序列化
- 未看到响应拦截器（401/403 等无统一拦截，按页面/store 分散处理）

---

## 6. 实时通信

### 6.1 WebSocket 连接管理
- 文件: `src/websocket.js`
- 连接地址: `/ws`（Vite 开发代理到 `ws://127.0.0.1:9000`）
- 生命周期策略:
  - 自动重连（指数退避，上限间隔 30s，最多 50 次）
  - `online` / `window focus` 触发补连
  - 心跳机制: 每 5s `ping`，60s 无 `pong` 主动断开重连
  - 手动关闭标记（避免误重连）

### 6.2 事件类型与处理
- 事件常量:
  - `new_message`
  - `message_prop_update`
  - `conversation_prop_update`
  - `new_notification`
- 处理方式:
  - `new_message`: 更新会话列表 + 当前会话消息缓存
  - `message_prop_update`: 更新消息属性
  - `conversation_prop_update`: 更新会话属性
  - `new_notification`: 写入通知 store

### 6.3 WebSocket 使用场景
- 初始化入口: `App.vue` 调用 `initWS()`
- 直接受益页面/功能:
  - Inbox/Conversation 页面（消息与会话状态实时刷新）
  - 顶栏通知（`NotificationBell`, `NotificationPanel`）

---

## 7. 组件设计模式

### 7.1 通用组件库组织方式
- 结构分层:
  - `components/ui/*`: 原子/基础组件（button、dialog、table、sidebar、form、dropdown、tabs...）
  - `components/layout/*`: 页面框架级组件（PageHeader 等）
  - `components/sidebar/*`: 导航与通知侧栏
  - `features/*`: 业务复合组件（跨页面可复用的功能块）
- 典型模式:
  - 每个 UI 组件目录提供 `index.js` 统一导出
  - 数据表格采用 `DataTable + columns schema + dropdown actions` 模式
  - 表单采用 `zod schema + vee-validate`，在 admin 子模块大量复用

### 7.2 composables 使用模式
- `useEmitter`: 全局事件总线接入（toast、刷新事件）
- `useConversationFilters` / `useActivityLogFilters`: 将筛选字段定义与 store options 解耦
- `useDraftManager`: 草稿本地缓存 + 后端同步（切换会话、visibilitychange 等触发）
- `useFileUpload`: 上传流程标准化（上传中状态、回调、失败兜底）
- `useIdleDetection`: 空闲检测驱动用户在线状态变更
- `useSla`: SLA 倒计时/状态周期更新

### 7.3 当前架构特点总结
- 优点:
  - 路由清晰，admin 模块化较好
  - store 切分合理，核心业务在 `conversation` 聚合
  - UI 基础层与业务层分离明确
  - WebSocket 与草稿、通知等关键体验点处理完善
- 可优化点:
  - API 层缺少统一响应拦截与错误规范化出口
  - `api/index.js` 体量较大，可按领域拆分模块（auth/conversation/admin/report 等）
  - 路由未见显式前端权限守卫，当前主要依赖页面内权限渲染与后端鉴权
