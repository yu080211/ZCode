# ZCode 前端学习指南（总指引）

> 目标：① 抄 UI（对话组件、排版布局）；② 学前端状态与数据处理逻辑；③ 学对话运行机制；④ 学 Skills 机制与提示词/记忆工程。
> 基于当前检出源码 `zcode 3.14.3`。所有路径均相对仓库根目录。

## 分文档索引

| 文档 | 内容 | 适合什么时候读 |
|---|---|---|
| [01-ui-components.md](./01-ui-components.md) | 对话组件库完整清单（按可抄程度分三档）+ 依赖清单 + 抄写步骤 | 开始抄 UI 前 |
| [02-layout-theme.md](./02-layout-theme.md) | 页面结构、组装树、三帧布局、样式/主题/i18n 体系、DESIGN.md 硬约束 | 抄排版布局前 |
| [03-state-dataflow.md](./03-state-dataflow.md) | 状态所有权、消息投影 store、乐观更新、hooks 层、发送链路 | 抄完静态 UI，开始接数据时 |
| [04-runtime-mechanism.md](./04-runtime-mechanism.md) | 协议 v4、传输层、Host、CLI 权威侧、端到端时序、断线恢复 | 想理解"为什么这么设计"时 |
| [05-skills-prompts-memory.md](./05-skills-prompts-memory.md) | Skills 机制、system prompt 分层、压缩提示词、记忆四件套 | 学习 Agent 侧设计时 |
| [06-token-design-prompts.md](./06-token-design-prompts.md) | CLI 省 token 设计清单 + 内置提示词全文中文翻译 | 学提示词工程 / 省 token 设计时 |
| [07-skills-catalog.md](./07-skills-catalog.md) | 8 个项目 skill 的目录约定 + 全文中文翻译（含附属资源清单） | 学 skills 怎么写、怎么组织时 |

## 项目全景速览

pnpm monorepo，React 19.2 + TypeScript(NodeNext) + Tailwind v4 + Zustand 5 + Lexical：

```
packages/
  ui        @zcode/ui —— 全部对话组件都在这（约1500个源文件，无构建、源码直接被消费）
  shared    协议与类型（src/zcode-protocol-v4/ 是新版对话协议，全 zod schema）
  services  服务实现（Host 进程内运行，含与 CLI 子进程的 stdio 通信）
  rpc       RPC 原语（ChannelServer/Client、ProxyChannel）
  client    连接 SDK（MessagePort / WebSocket 两种接入）
  desktop   Electron（main / host / renderer）
  web       Web 客户端（极少代码，复用 ui 的 Root）
  server    Node 服务（WS RPC + 远控 relay + 静态托管）
apps/
  zcode-cli Agent 运行时（对话事实的唯一权威，skills/记忆/压缩都在这）
```

**一句话架构**：命令上行 + 快照/增量下行的"单一权威投影"。消息事实的唯一权威在 CLI 进程；UI 端消息列表的唯一所有者是 `packages/ui/src/v4/conversationProjectionStore.ts`（plain class + useSyncExternalStore，不是 Zustand）。

## 推荐学习路线

```
第一周  抄 UI
  01 文档 → 先抄 A 档纯展示组件（ai-elements + ui 原语）
  02 文档 → 抄布局：三帧结构、resizable、styles.css 的 @theme token
第二周  接数据
  03 文档 → 读 conversationProjectionStore + 一次发送的完整链路
第三周  懂机制
  04 文档 → 对照源码读一遍端到端时序（重点：断档自愈、乐观转正）
第四周  Agent 侧
  05 文档 → skills 两段式注入、压缩 9 节模板、记忆抽取 agent
```

## 仓库自带的参考文档

| 文件 | 内容 |
|---|---|
| `AGENTS.md` | 仓库工作约定（架构边界、UI 平台边界、日志规范）——改代码前必读 |
| `DESIGN.md` | UI 设计规范（537 行）：字号刻度、语义色、圆角、间距、三帧布局 |
| `CONTEXT.md` | 插件商店领域词汇表（改相关 UI 前读） |
| `README.md` / `README.en.md` | 项目介绍 |
| `.agents/skills/ai-elements/SKILL.md` | **教你用 ai-elements 组件搭 AI 聊天界面的官方指南**——抄 UI 的最佳入门 |
| `.agents/skills/react-best-practices/SKILL.md` | Vercel React 性能优化指南 |
| `architecture-policy.yaml` | 架构检查规则（单文件 400 行上限、禁循环依赖、禁深导入） |
| `packages/ui/components.json` | shadcn 配置（style: radix-mira，registry 挂了 @ai-elements） |

## 常用命令（根目录执行）

| 用途 | 命令 |
|---|---|
| 桌面开发 | `pnpm dev:desktop` |
| Web 开发 | `pnpm dev:web` |
| 类型检查 | `pnpm typecheck` |
| Lint | `pnpm lint` / `pnpm lint:fix` |
| 架构检查 | `pnpm architecture:check --changed` |
| 读某模块的受控上下文 | `pnpm architecture:context <module-id>` |
| 未使用依赖/导出 | `pnpm knip` |
| 导出引用查询 | `pnpm dep:refs --list-exports <file>` |

## 抄代码的通用注意事项

1. `packages/ui` 包内用 `@/*` 别名指 `packages/ui/src/*`，且 NodeNext 要求**相对导入带 `.js` 后缀**。独立出去时 `@` 别名会断（消费方 vite 各自配了 `@` 指向 ui/src），新包内要改成包内别名或相对路径。
2. `conversation.tsx` 头部注释明确：ai-elements 组件在 `@zcode/ui` 包内**不自引用** `@zcode/ui/...`，互相引用走相对路径——这对抄出去是好事。
3. UI 访问服务只经 `packages/ui/src/hooks/`；平台操作只经 `IPlatformService`（`packages/shared/src/platform.ts`）注入，不直接碰 `window.zcode`。
4. 日志用 `packages/ui/src/logger.ts`，不用 console.log。
5. 单文件超 400 行会被架构检查标记（存量豁免文件靠 `/* eslint-disable max-lines */` 锁定），抄的时候顺手拆小。
