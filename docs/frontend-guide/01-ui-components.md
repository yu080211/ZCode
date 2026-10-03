# 01 · 对话组件库清单（抄 UI 用）

> 前缀 `SRC = packages/ui/src`。核心结论：这个仓库的"AI 对话组件库"就是 **`SRC/components/ai-elements/`（47 个文件）+ `SRC/components/ui/`（44 个 shadcn 原语）**，其中约 40 个文件零耦合可直接复制；重耦合的是 v4 编排层（SessionPane/Composer/Timeline），建议只复用视觉规格。
> 配套官方教程：`.agents/skills/ai-elements/SKILL.md`。

## 一、三档归类总表

### A 档：纯展示，可直接复制（只依赖 cn + ui 原语 + 三方库）

**ai-elements（36 个，`SRC/components/ai-elements/`）**

| 分组 | 文件 | 说明 |
|---|---|---|
| 会话容器 | `conversation.tsx` | 滚动容器：use-stick-to-bottom 自动吸底 + 滚到底按钮 + `messagesToMarkdown` 导出 |
| | `chat-loading.tsx` / `checkpoint.tsx` / `persona.tsx` | 加载动画 / 检查点行 / Rive 数字人（B 档其实更准，有 i18n） |
| 输入框家族 | `prompt-input.tsx` | 表单容器 + 附件/文本 Context（nanoid 生成 id），尾部 `export *` 另 4 个文件 |
| | `prompt-input-textarea.tsx` | 自动高度 textarea |
| | `prompt-input-buttons.tsx` | 工具栏按钮 + 提交/停止（ChatStatus 驱动） |
| | `prompt-input-actions.tsx` | 附件/截图/菜单动作 |
| | `prompt-input-primitives.tsx` | Select/HoverCard/Tabs/Command 包装件（19 个导出） |
| 工具/结果卡 | `tool.tsx` | 通用工具调用折叠卡（ToolUIPart） |
| | `task.tsx` | 待办列表折叠块 |
| | `plan.tsx` | 计划卡（流式 Shimmer） |
| | `agent.tsx` | 子 Agent 卡（Accordion 工具列表） |
| | `confirmation.tsx` | 工具审批确认卡（createContext 分发） |
| | `test-results.tsx` | 测试报告块（Suite/Test/Status/ErrorStack） |
| | `sandbox.tsx` | 沙箱执行 Tabs |
| | `terminal.tsx` | 终端输出块（ansi-to-react） |
| | `queue.tsx` | 排队消息队列展示 |
| | `snippet.tsx` / `commit.tsx` | 命令片段 / git 提交卡（20+ 导出） |
| Markdown 定制 | `markdown-table.tsx`（~1450 行） | 表格 + 虚拟横向滚动条 + 复制 CSV/TSV + 全屏预览 |
| | `markdown-list.tsx` / `markdown-blockquote.tsx` / `markdown-image.tsx*` | Streamdown 节点定制（*带 services 耦合，见 C 档） |
| | `code-block.tsx` | 代码块（shiki、换行开关、语言选择、mermaid 预览入口） |
| | `mermaid-block.tsx` | mermaid 图（渲染队列、主题色解析） |
| | `diagram-preview-dialog.tsx` / `image-preview-dialog.tsx` | 图/图片全屏预览（缩放平移） |
| | `image.tsx` / `image-thumbnail-gallery.tsx` | 生成图展示 / 缩略图横排 |
| 流程图（@xyflow/react） | `canvas/node/edge/connection/controls/panel/toolbar.tsx` | ReactFlow 容器与节点件 |
| 其他 | `sources.tsx` / `context.tsx` / `artifact.tsx` / `shimmer.tsx` / `suggestion.tsx` / `streamdown-controls.ts` | 引用来源 / token 用量 HoverCard(tokenlens) / 生成物容器 / 流光 / 建议项 / streamdown 类名约定 |

**ui 原语（`SRC/components/ui/`，抄聊天需要的子集）**

```
button button-group input input-group textarea badge card avatar alert alert-dialog
dialog dropdown-menu context-menu popover hover-card tooltip command（cmdk，mention/slash 面板底层）
collapsible（Tool/Reasoning/Task 全在用） scroll-area scroll-fade-viewport resizable
select separator switch checkbox checkbox progress tabs kbd spinner toast accordion
```
另有按需：`diff-viewer` / `code-viewer` / `lightweight-diff-preview`（@pierre/diffs，edit 工具卡的 diff）、`pdf-viewer` + `usePdfZoomOverlay.ts`（react-pdf）、`chart`（recharts）。

**纯逻辑模块（随用随抄）**

- mention 纯逻辑 10 件：`SRC/mentions/` 下 `mentionSearch / mentionMarkdown / mentionHelpers / mentionTypes / mentionChip / mentionPanelRouting / activePromptInputToken / promptSerialization / promptMentionLabel / components/scrollMask.ts`
- `SRC/prompt-editor/useComposerToolbarFit.ts`、`usePromptEditorDragState.ts`
- `SRC/v4/toolCallRowAdapter.ts`（v4 工具状态 → 展示状态映射，纯函数）
- `SRC/v4/messageTimeLabel.ts`、`conversationWorkDuration.ts`、`conversationTurnRenderUnits.ts`
- `SRC/lib/utils.js`（`cn`）——一切的基础，先抄这个

### B 档：轻耦合（仅 i18n / 主题 / type-only，小改即独立）

| 文件 | 耦合点 | 改造方式 |
|---|---|---|
| `SRC/components/ai-elements/reasoning.tsx` | `useZCodeIntl` + @zcode/shared TID 常量 + QueuedSummaryContent | i18n 注入；TID 内联 |
| `SRC/components/ai-elements/code-block.tsx` | useZCodeIntl + `@/lib/fileDisplay/mermaidLanguage/mermaidRenderBudget` + logger | 同上 |
| `SRC/components/ai-elements/markdown-table.tsx` | useZCodeIntl + ui Button/Dialog/toast | i18n 注入 |
| `SRC/components/ai-elements/mermaid-block.tsx` / `diagram-preview-dialog.tsx` | useZCodeIntl + useTheme 的 resolveTheme / `@/lib/diagramViewport` | i18n/主题注入 |
| `SRC/components/ai-elements/chat-loading.tsx` / `context.tsx` | useZCodeIntl / `@/lib/tokenNumberFormat` | i18n 注入 |
| `SRC/ToolCallBlocks/ToolLayout.tsx`、`ToolSummaryRow.tsx`、`QueuedSummaryContent.tsx` | useZCodeIntl + memoryDiagnostics | i18n 注入 |
| `SRC/prompt-editor/ChatPromptEditor.tsx`（454 行） | 自述"纯 props 组件无 store 依赖"；useZCodeIntl + workspaceFileDrag lib | i18n 注入；壳可整抄 |
| `SRC/components/ui/lightweight-diff-preview.tsx` 等 | `CodePreviewSettings` 仅 type import | 类型内联 |

### C 档：重耦合（需注入改造，或连状态层一起搬）

| 文件 | 耦合点 | 价值与建议 |
|---|---|---|
| `SRC/components/ai-elements/message.tsx`（~1700 行） | useZCodeStore、usePlatform、useFileContextActions、useWorkspaceOpenInEditorTarget、useOptionalServices、文件引用 remark 插件 | **`MessageResponse` 是全仓消息 Markdown 渲染核心**（Streamdown + cjk/code/math/mermaid 插件 + shiki + 自定义 renderer）。把"打开文件/编辑器"收敛为回调 props 后整套抄 |
| `SRC/components/ai-elements/markdown-image.tsx` / `image-preview-dialog.tsx` | useOptionalServices / useOptionalPlatform | 同上改回调 |
| `SRC/LexicalChatInput.tsx`（1531 行） | activeTaskProvider、store 类型、shortcuts 体系 | 输入编辑器本体：IME 兼容、中文"、"→"/"、历史、mention node。值得精读后按需抄 |
| `SRC/SlashCommandPlugin.tsx`（461 行） | useSlashCommands/useSubagents/useSkills | `/` 命令面板，provider 注入化 |
| `SRC/mentions/MentionPlugin.tsx`（722 行）+ `providers/*` | 6 类候选源全走 useServices | **providers 是重耦合集中地**；面板 UI（MentionPanel，@tanstack/react-virtual）和 PromptMentionNode 可拆 |
| `SRC/prompt-editor/ChatPromptActionMenu.tsx` | useFileMentionProvider 等五个 provider | 同上 |
| `SRC/chat-input-toolbar/`（13 文件） | TabStore + CodingPlan 业务 hooks | 工具栏 UI 可抄，CodingPlan 用量部分按需 |
| `SRC/ToolCallBlocks.tsx` + `SRC/ToolCallBlocks/`（40+ 工具卡） | RenderContext 间接依赖 services | 工具卡注册表模式值得抄（`resolveRenderer.ts`）；卡片本身多为展示 |
| `SRC/v4/ConversationTimeline.tsx`（1963 行） | 投影 store + i18n + telemetry | 虚拟滚动 + 吸底锚定 + 行高缓存 + 查找高亮的完整范本 |
| `SRC/v4/ConversationRowView.tsx`（2170 行） | 多个 store/hooks | 单条消息全量渲染，抄"排版"看它 |
| `SRC/v4/ConversationComposer.tsx`（2387 行） | zcodeSessionStore、attachments、TTFT | 输入区编排，C 档最重 |

## 二、npm 依赖清单（抄 UI 要装的包）

### 必装（A 档就用到）

```
tailwindcss ^4（CSS-first，无 config 文件，token 全在 css 里）
clsx tailwind-merge class-variance-authority      # cn 与 cva
radix-ui（统一包 ^1.4.3）+ lucide-react + cmdk    # ui 原语
streamdown ^2.5 + @streamdown/cjk + @streamdown/code + @streamdown/math + @streamdown/mermaid
shiki ^4                                          # message/code-block 高亮
use-stick-to-bottom                               # conversation.tsx
motion                                            # shimmer 等
nanoid                                            # prompt-input
ai ^6                                             # ToolUIPart/ChatStatus 等类型
```

### 按需

```
@tanstack/react-virtual ^3   # MentionPanel、ConversationTimeline 虚拟滚动（peerDep）
katex ^0.16                  # streamdown math 插件间接用
@xyflow/react ^12            # 流程图 7 件
ansi-to-react                # terminal 卡
@pierre/diffs                # diff-viewer
react-pdf + pdfjs-dist       # pdf-viewer
mermaid                      # 经 @streamdown/mermaid
tokenlens                    # context.tsx 用量
@rive-app/react-webgl2       # persona 数字人
lexical + @lexical/react ^0.42  # 输入编辑器
```

### peerDependencies（照抄 ui 的做法）

`react ^19.2`、`react-dom ^19.2`、`@tanstack/react-virtual ^3`。根 `package.json` 用 pnpm overrides 统一 react 19.2.7。

## 三、抄写步骤（建议顺序）

1. **基建**：`SRC/lib/utils.js`（cn）→ `SRC/components/ui/` 需要的子集 → `SRC/styles.css` 里的 `@theme` token 段 + 三段主题变量（见 02 文档）。
2. **A 档整抄**：ai-elements 的 A 档文件按"conversation → prompt-input 家族 → tool/task/plan/agent/confirmation → markdown-* → code-block/mermaid"顺序抄，每抄一个跑通一个。
3. **B 档改造**：统一把 `useZCodeIntl().formatMessage({id})` 换成注入式 i18n（自建轻量 Provider 或 props 传文案）；把 `@zcode/shared` 的 TID 常量内联。
4. **C 档选择性抄**：`message.tsx` 的 MessageResponse 值得做注入改造后整抄（Markdown 渲染这一套自己写要很久）；输入编辑器（LexicalChatInput + MentionPlugin + SlashCommandPlugin）建议精读后按"编辑器内核 / 面板 UI / 候选源 provider"三层拆开抄。
5. **验证**：在目标项目里起一个静态 demo 页，装假数据跑通 message + conversation + prompt-input 三件套，再验证暗色主题。

## 四、抄写时的坑

- `prompt-input.tsx` 尾部 `export *` 自另外 4 个 prompt-input 文件——整个家族一起搬。
- `attachments.tsx`、`tool.tsx`、`confirmation.tsx` 等依赖 `ai` 包的类型（`FileUIPart`/`ToolUIPart`/`ChatStatus`），要么带上 `ai` 依赖，要么自己定义等价类型。
- `message.tsx` 里的文件引用链接（`openMessageFileLinkInEditor`）是编辑器场景特有的，抄通用聊天时直接删掉这条链路，保留 `MessageFileLinkTarget` 类型占位。
- `reasoning.tsx` 引了 `SRC/ToolCallBlocks/QueuedSummaryContent` 和 `SRC/mentions/components/scrollMask`——这两个在 B/A 档里，一起带走。
- `components.json`（shadcn 配置）一起带走，保留 `shadcn` CLI 后续添加组件的能力。
