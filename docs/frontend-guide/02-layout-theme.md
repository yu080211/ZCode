# 02 · 页面结构、排版布局与主题体系

> 前缀 `SRC = packages/ui/src`。Desktop 和 Web 共用同一棵 UI 树；Web 端只多了分享页。

## 一、页面/路由结构（没有路由库）

入口分流全靠"状态条件渲染 + 自制 tab store"，这是本仓库最特别的设计：

- 窗口级：`packages/desktop/src/renderer/src/main.tsx` 按查询参数 `windowKind` 分流——`update-status` 挂 `UpdateStatusWindowRoot`，否则走 `AppErrorBoundary → ZCodeIntlProvider → Root`。
- **`SRC/Root.tsx`（1089 行）就是"路由器"**，顶层分支（Root.tsx:967-1088）：
  1. 启动阻塞 → `SRC/root/RootStartupLoading.tsx`
  2. 未登录 → `SRC/WelcomeScreen.tsx`（登录/欢迎全屏页）
  3. 正常 → `SRC/onboarding/OccupationOnboarding.tsx` 包裹，内部分支：
     - 设置 tab 打开 → `SRC/SettingsPage.tsx`
     - 有 workspace → `SRC/root/RootWorkspaceContent.tsx`
     - 常驻 `OnboardingDialog`
- tab 状态：`SRC/store/tabStore.ts`，`WindowTabState = kind:"workspace" | "settings"`。**设置页是覆盖层不是替换视图**（RootWorkspaceContent.tsx:99-204：workspace 壳常驻，`opacity-0 + pointer-events-none + inert`，上面叠 `WorkspaceSettingsLayer`）——这个"保活覆盖层"手法值得抄。
- workspace 内主视图：`SRC/App.tsx:826` 的 `workspaceMainView: "chat" | "automations" | "plugin-store"`，由 `SRC/app-shell/useWorkspaceTaskNavigation.ts` 驱动（内含 back/forward 历史）。

## 二、对话主视图组装树（抄布局就看这棵树）

```
App.tsx（1280 行，shell 状态编排）
└─ WorkspaceShellLayout.tsx（1974 行，SRC/app-shell/）★布局核心
   ├─ DesktopWindowFrame.tsx                          ← 窗口 chrome/圆角/标题栏
   │  ├─ #sidebar（CSS 变量宽度 + 自绘 resize 分隔条）
   │  │  └─ WorkspaceSidebar.tsx（1700 行）
   │  │     ├─ WorkspacePinnedTasksSection / WorkspaceGroupedTasksSection
   │  │     │  / WorkspaceTimelineTasksSection（SRC/workspace-*/）
   │  │     │  └─ TaskList.tsx / TaskListItem.tsx / TaskListItemContextMenu.tsx
   │  │     ├─ WorkspaceSidebarFooter.tsx（登录/用量徽标）
   │  │     └─ WorkspaceFileTree（SRC/workspace-file-tree/）
   │  └─ #content → ResizablePanelGroup（SRC/components/ui/resizable.tsx）
   │     ├─ ResizablePanel "conversation-column"
   │     │  ├─ ResizablePanel "conversation"
   │     │  │  ├─ WorkspaceHeader.tsx（variant: draft | task）
   │     │  │  └─ 主视图三选一，默认 "chat" →
   │     │  │     V4WorkspaceChatArea.tsx（SRC/v4/，594 行）
   │     │  │        ├─ WorkbenchLeafPane / WorkbenchSplitDivider（分屏，SRC/v4/WorkbenchPane.tsx）
   │     │  │        └─ 每 pane → V4ChatPane.tsx
   │     │  │           └─ V4ConversationProvider → SessionPane.tsx（4834 行）★单 pane 全部对话逻辑
   │     │  │              ├─ ConversationHeader.tsx（pane 标题/分屏按钮）
   │     │  │              ├─ ConversationStatusPanel.tsx（目标/计划/后台任务汇总）
   │     │  │              ├─ ConversationTimeline.tsx（1963 行，消息流，虚拟滚动）
   │     │  │              │  └─ ConversationTurnGroup → ConversationTurnRow
   │     │  │              │     → ConversationRowView.tsx（2170 行，单条消息全量渲染）
   │     │  │              │        ├─ MessageResponse（ai-elements/message.tsx）
   │     │  │              │        ├─ Reasoning（ai-elements/reasoning.tsx）
   │     │  │              │        ├─ ToolCallBlock（ToolCallBlocks.tsx）
   │     │  │              │        └─ Attachments / AssistantPreviewCards / 行内编辑
   │     │  │              ├─ bottomDock：ConversationQuotaBanner / ConversationQueuePanel
   │     │  │              │   / V4InteractionDialogs（权限/问答弹窗）
   │     │  │              ├─ ConversationComposer.tsx（2387 行，输入区）
   │     │  │              │  ├─ ChatPromptEditor（SRC/prompt-editor/，Lexical）
   │     │  │              │  ├─ V4ComposerToolbar / ModeControls（SRC/v4/composer/）
   │     │  │              │  └─ chat-input-toolbar/（模型选择、上下文用量）
   │     │  │              └─ 草稿态：ConversationDraftEmptyState + ConversationDraftSuggestedPrompts
   │     │  └─ AnimatedTerminalPanel.tsx → Terminal.tsx（xterm）
   │     └─ sidePanePanel：AnimatedSidePanePanel.tsx（右侧共享侧板，多 tab）
   │        内含 PreviewPane.tsx（1806 行：md/pdf/office/xlsx/pptx/code 多态预览）
   └─ DesktopTopOverlay.tsx（桌面顶部悬浮工具条）
```

**布局要点（抄的时候注意）**

- **三帧布局**：会话帧 / 终端帧 / SidePane 帧之间是 4px 可拖拽缝隙（DESIGN.md 明文规定），实现就是 `ResizablePanelGroup` + `AnimatedTerminalPanel`/`AnimatedSidePanePanel` 的展开收起动画。
- 侧栏宽度不用 react-resizable-panels，是 CSS 变量 + 自绘分隔条（WorkspaceShellLayout 里）；主区三帧才用 `components/ui/resizable.tsx`（react-resizable-panels）。
- 分屏（workbench）是 v4 的 `WorkbenchPane.tsx` + `paneLayoutStore.ts` + `paneLayoutTree.ts` 一套，树状布局持久化在 `paneLayoutPersistence.ts`。
- 吸底滚动锚定看 `SRC/v4/timelineScrollAnchor.ts` + `timelineRowHeightCache.ts` + `conversationTimelineLiveTail.ts`，配合 `@tanstack/react-virtual`。

## 三、窗口清单（Electron 侧）

主进程 `packages/desktop/src/main/`：

| 窗口 | 创建处 | 加载 |
|---|---|---|
| 主窗口（多开） | `index.ts:1672` | renderer `index.html` → Root |
| 更新状态窗 | `index.ts:1528` | 同一 index.html + windowKind 参数 → `SRC/UpdateStatusWindowRoot.tsx` |
| 资源管理器窗 | `resourceManagerWindow.ts:261` | `resource-manager.html` → 独立入口 |
| CUA 权限浮窗 / About / BrowserView | 各自文件 | 独立入口或原生 |

`DeveloperToolsPane.tsx` 不是独立窗口，是右侧 SidePane 的一个 tab。vite 多入口见 `packages/desktop/vite.config.ts:226-233`。

## 四、样式与主题体系（Tailwind v4 CSS-first）

**唯一样式入口：`SRC/styles.css`（2053 行），没有 tailwind.config 文件。**

| 区段 | 位置 | 内容 |
|---|---|---|
| 导入 | 头部 | `@import "tailwindcss" source(".")` + tw-animate-css + shadcn/tailwind.css + xterm/katex css |
| 扫描 | styles.css:42 | `@source "./**/*.{ts,tsx}"` 显式扫 ui 包源码（workspace 消费场景的坑，注释有解释） |
| 基础 | `:root` | `--ui-font-size: 14px` |
| **token** | `@theme`（137 行起） | `text-ui-*` 字号刻度 + 语义色（这是 DESIGN.md 的最高优先级约束） |
| 暗色 | `.dark`（308 行起） | 变量覆盖 |
| 品牌 | `.theme-zai-light`（461 行起） / `.theme-zai-dark`（606 行起） | 两套品牌主题变量 |
| 平台 | `@custom-variant` | `platform-mac-desktop` / `platform-windows-desktop` / `platform-linux-desktop` |

**DESIGN.md 硬约束（抄排版必须遵守）**：

1. 字号只用 `text-ui-*` 刻度，禁用 Tailwind 内建 `text-base/sm/xs` 和任意 px 字号。
2. 语义色 token 体系（background/surface/card/popover/menu + foreground 三级 + border/diff/interaction），禁止一次性色值。
3. 圆角按容器嵌套层级 `xl→lg→md→sm`，仅主聊天输入壳/状态浮板/Toast 允许 `rounded-2xl`。
4. 间距 4px 基、克制阴影。

**主题切换**：`SRC/useTheme.ts`（`Theme = light|dark|zai-light|zai-dark|system`，写 `localStorage["zcode-theme"]`，toggle `<html>` 的 class）；App 层主题真值在 App.tsx:356 的 zustand store；main.tsx:77-100 在 React 接管前先做初始化（防闪白）。

## 五、i18n

- 自研轻量 intl（非 react-intl）：`SRC/i18n/IntlProvider.tsx` 维护 `Locale = "zh-CN" | "en-US"`，`formatMessage({id}, values)` 查 `SRC/i18n/locales/zh-CN.ts` / `en-US.ts`（各约 425KB 扁平 key→string 表）。
- 偏好存 `localStorage["zcode-locale-preference"]`，可经 `ISettingService` 持久化并跨窗口同步（broadcast 频道 `state:locale`，注意防回环）。
- 导出：`ZCodeIntlProvider / useZCodeIntl / LocaleSwitcher`。
- 用法是组件内 `useZCodeIntl()` + 内联 `intl.formatMessage({ id: "chat.send" })`，key 就在 locales 表里 grep。
- **抄库时**：要么自建只含 chat key 的迷你表，要么改 props 传文案。相关 key 的样例：`chat.send / chat.stop / chat.message.copy|edit|fork / chat.attachments.* / chat.toolCall.* / chat.history.* / chat.queue.* / codeBlock.*`。

## 六、Web 端差异（复用示范）

- `packages/web/src/main.tsx`（479 行）手工分流：OAuth 回调页 → `WebCallbackPage`；分享页 `/share/*` → `ConversationShareLandingPage`（复用 `@zcode/ui/conversation-share-readonly`，即 `SRC/v4/ConversationShareReadonlyTimeline.tsx`——**只读时间线的现成范本**）；其余 → `connectViaWebSocket` 后渲染与桌面相同的 `Root`。
- 平台差异用 no-op 桩 `createWebPlatform()`（main.tsx:190-353）实现 `IPlatformService`——**想看"哪些平台能力要注入"，读这个桩最快**。
- `packages/server` 是 Node Hono 服务：`/ws`（RPC over WebSocket）、`/ws/host`（受信 host）、`/api/connect-remote` + `/ws/remote/:id`（手机远控 relay）、静态托管 + SPA fallback（http.ts:449-453）。

## 七、消费方接线（独立出去要知道的）

- `packages/ui/package.json`：exports 全指向 `./src/*.ts(x)`（零构建源码模式），子路径有 `./useTheme`、`./message`、`./conversation-share-readonly`、`./styles.css` 等。
- 消费方 vite 必须补别名：`packages/desktop/vite.config.ts:180` 与 `packages/web/vite.config.ts:49` 都有 `"@": resolve(__dirname, "../ui/src")`。
- 依赖方向：ui → services/rpc/shared 单向；desktop/web → ui。
