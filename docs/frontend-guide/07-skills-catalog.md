# 07 · Skills 全集（8 个项目技能，含中文翻译）

> 位置：`.agents/skills/`。每个技能 = 一个目录 + `SKILL.md`（frontmatter 写触发条件，正文写操作步骤）。AI 平时只在 system-reminder 里看到一行描述，判断相关才通过 Skill 工具加载正文——**技能自己内部的附属文件也是按需读取的**（见下表）。

## 一、skill 目录约定（渐进披露三件套）

| 子目录 | 作用 | 本仓库实例 |
|---|---|---|
| `references/` | 深度文档，SKILL.md 里明确写"何时去读哪个" | `agent-browser` 7 个（commands/authentication/snapshot-refs…）；`ai-elements` **48 个组件文档**；`architecture-governance` 5 个（policy-schema/module-contract/rule-catalog/ai-guidance/troubleshooting + golden-module 样例目录）；`feature-boundary-planner` 4 个（feature-graph.yaml/source-discovery/case-planning-template/impact-brief-template）；`dogfood` 1 个（issue-taxonomy） |
| `templates/` | 可直接复制使用的模板文件 | `dogfood` 的报告模板；`agent-browser` 3 个 shell 脚本模板 |
| `scripts/` | 可执行脚本 | `architecture-governance/scripts/context-package.mjs`（生成受控阅读包）；`ai-elements/scripts/` **约 100 个 .tsx 组件示例** |
| `agents/` | 子代理配置（`openai.yaml`） | architecture-governance / feature-boundary-planner / dep-refs 各有 |
| `rules/` | 规则文件库，SKILL.md 只放索引 | `react-best-practices/rules/` **72 个 .md**（70 条规则 + 索引），每条含"为什么/错误示例/正确示例" |

**关键观察**：`SKILL.md` 只放核心工作流；大块内容全部拆到附属目录，SKILL.md 里带"何时读哪个"的表格。比如 react-best-practices 的 SKILL.md 只列 70 条规则的名字，需要哪条才去读 `rules/async-parallel.md`。

---

## 二、architecture-governance（架构治理）

```yaml
name: architecture-governance
description: 把仓库的架构策略应用到代码改动上——生成受控上下文包、检查模块与层级边界、报告基于基线的违规。任何代码改动都适用；纯文档工作跳过。
```

**在写代码之前：**

1. 用 `pnpm architecture:check --changed` 识别改动文件及其所属模块。
2. 运行 `pnpm architecture:context <module-id>`（或可复用的包装脚本 `node .agents/skills/architecture-governance/scripts/context-package.mjs <module-id>`）。在打开大范围实现文件之前，先读目标 contract、直接引用的 contract、以及现有的相关 spec 和测试。**不要假设文档或测试路径存在；以当前检出为准逐项核实。**
3. 在实现之前编写或更新 spec；目录不存在就创建。在 spec 里写明：行为、所有权、不变量、失败语义、迁移边界。
4. 编码前做一个简短的设计决策：
   - **单一所有者（One owner）**：为每一份可变状态指名唯一的属主组件。其他层通过它的 contract 读取、向它发送命令；不得另建第二份被认可的队列、缓存或派生真相。
   - **单一路径（One path）**：当已有的命令、服务、hook、适配器或 contract 已经能表达该行为时复用它。不要为同一职责创建平行 helper。
   - **显式边界（Explicit boundaries）**：为每个新文件选定所属层，为每条跨模块边选定公共 contract。使用模块声明中的 `layers` 和 `layerOrder`；一个文件只能导入自己所在的层或更低的层（经其公共接口）。`domain` 层是纯的（无 IO、不 await 外部世界）；`app` 层通过端口决策副作用；`adapters` 层执行副作用；`ui` 只依赖本模块的 `contract.ts`。快速判断：需要 `await` 吗？那就不是 domain。知道自己是 sqlite / MessagePort / 定时器吗？那是 adapters。
   - **显式时序（Explicit time）**：对异步或远端行为，写清事件顺序、owner/lease、幂等键、过期结果规则、重放/恢复边界、桌面与手机的投递类型——先于实现。
   - **有界上下文（Bounded context）**：优先使用生成的阅读包，而不是把整套实现拷进 prompt。只有在某个 contract 或测试证明必要时才扩大阅读范围。
5. 如果改动跨模块或改变了状态所有权，把决策写进 spec，并在实现前新增或更新模块 contract。

规划有状态改动时，用这张紧凑的设计草图：

```text
输入 → 单一所有者 → 命令准入 → 状态迁移 → contract/事件
                    └── 持久化 / 重放 / 投影均从所有者派生
```

对远端或流式改动，把投递边界写显式：

```text
桌面: continuous ── 直接实时流 ──┐
                                ├─ 同一个所有者与序列
手机: replayable ─ 快照 + 断档修复 ┘
```

**编辑期间与之后：**

6. 把改动保持在声明的模块及其允许的层方向之内。引入跨模块边之前先补模块依赖或公共 contract。
7. 编辑完成后再次运行 `pnpm architecture:check --changed`。**把新增违规与基线存量违规分开报告**，并附上：改动的模块、测试、状态所有者、事件顺序假设、净行数变化。

可执行策略是 `architecture-policy.yaml`；不要把它的规则复制进本文件或 AGENTS.md。只有当一次经过评审的改动有意改变既存基线时，才使用 `pnpm architecture:baseline:update`。**CI 永远不会自动刷新基线。**

新增源码时，先确定它的归属模块。需要新受管模块时，在 `architecture-policy.yaml` 注册它的 roots、依赖、层和公共入口，并保持本地 manifest 与策略一致。以现有受管模块和 golden-module 样例为范本：新受管模块需提供 `module.ts`、`contract.ts`、`contract.example.ts` 和简短的 `CONTRACT.md`。运行时与持久化细节藏在 contract 之后。一对一交互优先用类型化服务调用；状态变更用命令；广播事实用类型化事件。

需要更详细指导时看 `references/policy-schema.md`、`references/module-contract.md`、`references/rule-catalog.md`。`references/golden-module` 是最小合规样例。`references/ai-guidance.md` 列出了本工作流要防的反模式、以及 agent 在提出代码前必须回答的问题。

---

## 三、feature-boundary-planner（功能边界规划）

```yaml
name: feature-boundary-planner
description: 把一个 ZCode 行为变更映射到当前 UI 面、状态所有者、协议命令、持久化与验证。用于影响面分析、产品边界规划、或基于当前检出的实现交接。
```

把请求的行为在**当前检出**里追溯一遍。只有当现有源码或一条明确的新需求支撑时，一个功能点才进入结论；使用任何引用的路径和符号前先核实；**不要从历史目录里重建已删除的功能**。

**选择范围：**
- **impact-only（仅影响面）**：只检查、只报告，不改代码和产品 spec。
- **planning（规划）**：实现之前先确立行为与验收用例。
- **implementation-handoff（交接）**：把已确认的决策变成有边界的实现与验证计划。

用请求隐含的模式；只澄清会实质改变范围且未知的决策；不要请用户重新确认本任务中已确立的事实。

**找到当前证据：**
1. 在 `references/zcode-feature-graph.yaml` 里按用户术语搜别名和节点 ID。只读命中的节点及其一跳关系，然后对照当前检出核实声明的文件、符号与语义。该图是**人工维护的种子索引，不是完整功能清单**。没有命中时用 `references/source-discovery.md` 补位或起步。读相关既有 contract 与包脚本；UI 工作读 `DESIGN.md`，插件商店工作读 `CONTEXT.md`。
2. 用 `rg --files` 和聚焦的 `rg -n` 定位入口。符号是 TypeScript 导出时，用 `pnpm dep:refs <file>:<symbol>` 追溯直接调用方。有索引化 codegraph 工具就作为补充证据，并对照检出核实其路径。
3. 给改动分类：展示、选项来源、草稿/默认值、校验、提交效果、持久化、恢复。
4. **逐个用户面独立追溯**到校验和提交命令。共享 UI 组件不构成共享状态或副作用。
5. 识别权威所有者、派生视图、协议边界、持久化与失败行为。为每条上下游依赖说明语义理由；仅有 import 不能证明产品关系。
6. 先检查一个有意义的语义跳，只有未解决的所有者或调用方需要时才扩展。关系分级：must-inspect / should-inspect / conditional / invariant-only / evidence-only。

有状态或异步行为，给一张简图：

```text
用户操作 → 界面草稿 → 校验 → 所有者命令 → 事件 / 持久化
                                └── 派生的 UI 投影
```

**维护种子图**：未变的语义边界保留原节点 ID，新术语加别名。缺种子或关系变化记为 `graph-drift-candidate`；**静态可达性本身不构成产品依赖**。impact-only 模式只报告建议、不改文件；规划/实现中只更新任务范围内已验证的条目。遵循 graph contract（`docs/skills/feature-boundary-graph.md`）：检查 YAML 解析、ID 唯一、关系端点与分级、受跟踪的源码路径与符号。不要恢复缺失的历史文档，不要凭图里的条目声称测试覆盖。

**规划与剪枝**：规划模式下实现前先写/更新功能 spec；目录不存在就创建；**不要假设已存在用例目录、测试运行器或覆盖率工作流**。只选能改变行为的维度：状态、事件、目标、客户端模式、投递类型、workspace identity、运行时可用性、持久化来源。用代表性、高风险组合，不要全局笛卡尔积。用例分类：accepted / undefined / pruned / ignored / bug-candidate。每个被剪枝的用例必须给出不变量或守卫；未定义行为留作明确决策。已接受用例按模板记录设置、动作、断言与所需证据（`references/case-planning-template.md`）。交接验证前，核实目标包里真正存在哪些测试、fixture 和命令；区分计划中、已执行、已纳入回归的覆盖。**测试路径缺失是缺口，不是覆盖。**

**输出**（用 `references/impact-brief-template.md` 的相关部分）：行为摘要与范围；UI 面矩阵（共享实现与分歧行为）；带当前文件/符号证据的依赖分级排序；状态所有者、校验点、提交命令与持久化；不变量与代表性验证用例；未决决策、缺失证据、图的漂移或更新。

**边界**：保留 `workspaceIdentity?.trim() || workspacePath` 做隔离、`workspacePath` 做执行/展示；桌面 `desktop-continuous` 与手机 `web-remote-replayable` 分开追溯；已接受命令追溯到其权威所有者，**不得把客户端草稿或乐观覆盖变成第二个被认可的队列**；运行时状态、快照、索引、设置、缓存，在被证明同步之前视为互不相同；不要从构建产物遗留的目录、旧文档或历史分支推断当前功能；不要为了迁就实现差异而悄悄改变产品语义——报告证据与需要做的决策。

---

## 四、dep-refs（导出引用追踪）

```yaml
name: dep-refs
description: 需要检查 z-code 工作区的 TypeScript 导出引用、列出文件导出、删除前确认导出是否未使用、重构时调查谁在导入某个符号、或把 pnpm knip 的未使用导出结果与 pnpm dep:refs 的符号级引用追踪结合使用时使用。
disable-model-invocation: true
```

在改动或删除 TypeScript 导出之前，用仓库的 `pnpm dep:refs` CLI 回答符号级问题。命令在仓库根目录运行。

**工作流**——删代码时先宽后窄：

```bash
pnpm knip    # 先找出疑似未使用的导出
pnpm dep:refs packages/shared/src/remoteTarget.ts:stripRemoteTargetSecrets   # 再用 dep:refs 检查有风险/不明确的
pnpm dep:refs --list-exports packages/shared/src/remoteTarget.ts             # 不知道符号名时列出文件全部导出
pnpm dep:refs --scope packages/services packages/shared/src/remoteTarget.ts:stripRemoteTargetSecrets  # 范围扫描——仅在有意限定范围时用
```

**声称某个导出可安全删除之前，优先不带 scope 跑一遍**，以免漏掉跨包调用方。

**JSON 模式**（机读输出要用 pnpm 静默模式，避免 banner 行）：

```bash
pnpm -s dep:refs packages/shared/src/remoteTarget.ts:stripRemoteTargetSecrets --json | jq .
```

汇总多个符号、喂给 `jq`、程序化比较 `references` 与 `reExports` 计数时用 JSON。

**解读结果**：`References (0)` 和 `Re-exports (0)` 的含义是"未发现静态引用"，**不是**"不存在动态使用"的证明——脚本检测不到 `dynamic import()` 路径和字符串形式的引用。符号只出现在 `Re-exports` 下时，删除前先追一遍对外的 barrel 路径；即使没有直接内部导入，re-export 也可能属于公共面。重构场景：从 CLI 输出里报出具体调用方（文件+行号），再决定更新调用方、保留导出还是删除。

---

## 五、ai-elements（AI 聊天组件教程）

```yaml
name: ai-elements
description: 用 ai-elements 组件构建 AI 聊天界面——会话、消息、工具展示、输入框等。用户想搭聊天机器人、AI 助手 UI 或任何 AI 驱动的聊天界面时使用。
```

> 本技能派生自 vercel/ai-elements（Apache-2.0），ZCode 做了本地集成与适配，许可与出处见仓库根 `THIRD-PARTY-NOTICES.md`。

# AI Elements

[AI Elements](https://www.npmjs.com/package/ai-elements) 是构建在 [shadcn/ui](https://ui.shadcn.com/) 之上的组件库与自定义 registry，帮你更快构建 AI 原生应用。它提供预构建组件，如会话、消息等。

安装很简单，两种方式：专用 CLI 命令（最快），或通过标准 shadcn/ui CLI（如果你已经采用 shadcn 工作流）。

> **重要**：所有 CLI 命令都要用项目对应的包运行器：`npx ai-elements@latest`、`pnpm dlx ai-elements@latest` 或 `bunx --bun ai-elements@latest`——取决于项目的 `packageManager`。下文示例用 `npx`，实际替换为正确运行器。

**前置条件**：Node.js 18+；装有 AI SDK 的 Next.js 项目；已安装 shadcn/ui（没装的话跑任何安装命令会自动装上）；另外强烈推荐用 AI Gateway 并在 `env.local` 加 `AI_GATEWAY_API_KEY`，这样不必每家 provider 都配 API key（每月还送 $5 用量）。

**安装组件**：AI Elements CLI 或 shadcn/ui CLI 都行，效果一样——把所选组件的代码和依赖加进你的项目。默认加到 `@/components/ai-elements/` 目录（或你在 shadcn 配置里设的目录）。跑完命令终端会确认文件已添加，然后就能用了。

**用法**：安装后像普通 React 组件一样 import 使用。**组件的代码就放在你的代码库里（不是藏在依赖包里）**，用起来非常自然，你甚至可以直接打开组件文件看它怎么实现、或做自定义修改。

**示例**：

```tsx title="conversation.tsx"
"use client";

import { Message, MessageContent, MessageResponse } from "@/components/ai-elements/message";
import { useChat } from "@ai-sdk/react";

const Example = () => {
  const { messages } = useChat();

  return (
    <>
      {messages.map(({ role, parts }, index) => (
        <Message from={role} key={index}>
          <MessageContent>
            {parts.map((part, i) => {
              switch (part.type) {
                case "text":
                  return <MessageResponse key={`${role}-${i}`}>{part.text}</MessageResponse>;
              }
            })}
          </MessageContent>
        </Message>
      ))}
    </>
  );
};

export default Example;
```

上面从 AI Elements 目录 import `Message` 组件，与 `MessageContent`、`MessageResponse` 子组件组合。你可以像自己写的组件一样定制它——代码就在你的项目里。

**可扩展性**：所有 AI Elements 组件尽量透传原生属性。例如 `Message` 扩展了 `HTMLAttributes<HTMLDivElement>`，`div` 支持的 props 都能传——很容易用自己的样式或功能扩展。

**定制**：安装后无需额外配置，样式（Tailwind CSS 类）已集成。例如想去掉 `Message` 的圆角，进 `components/ai-elements/message.tsx` 删掉 `rounded-lg` 即可。

**故障排查**：
- **组件没样式？** 确认项目按 Tailwind 4 + shadcn/ui 正确配置：`globals.css` 要 import Tailwind 并包含 shadcn 基础样式。
- **CLI 跑了但什么都没加？** 检查：当前工作目录是项目根（`package.json` 所在处）；`components.json` 配置正确；CLI 是最新版。
- **主题切换不生效（一直亮色）？** 确认应用使用 shadcn/ui 和 AI Elements 期望的 data-theme 系统（默认在 `<html>` 上切 data-theme 属性），tailwind.config 用 class 或 data- 选择器。
- **报 "module not found"？** 确认文件存在；确认 `tsconfig.json` 有 `@/` 路径别名（`"paths": { "@/*": ["./*"] }`）。
- **AI 编程助手访问不到组件？** ① 校验配置文件是合法 JSON ② 检查 AI 工具的文件路径 ③ 改完重启助手 ④ 确认网络稳定。
- 都不行就去 [GitHub issue](https://github.com/vercel/ai-elements/issues)。

**可用组件**：见 `references/` 文件夹里每个组件的详细文档（本仓库有 48 个：agent/artifact/attachments/canvas/chain-of-thought/checkpoint/code-block/commit/confirmation/connection/context/conversation/file-tree/image/inline-citation/jsx-preview/message/model-selector/plan/prompt-input/queue/reasoning/sandbox/schema-display/shimmer/snippet/sources/speech-input/stack-trace/suggestion/task/terminal/test-results/tool/toolbar/transcription/voice-selector/web-preview 等）；`scripts/` 下是对应组件示例（约 100 个 .tsx）。**references 里以 `scripts/` 开头的路径是相对本技能目录的示例，不是仓库根的 scripts；它们是组件示例，不是已安装的应用功能。**

---

## 六、react-best-practices（React 性能最佳实践）

```yaml
name: vercel-react-best-practices
description: Vercel 工程团队的 React 与 Next.js 性能优化指南。编写、评审或重构 React/Next.js 代码时使用，确保采用最优性能模式。任务涉及 React 组件、Next.js 页面、数据获取、包体积优化或性能改进时触发。
license: MIT
metadata: { author: vercel, version: "1.0.0" }
```

# Vercel React 最佳实践

Vercel 维护的 React/Next.js 综合性能优化指南。**含 8 大类共 70 条规则，按影响面排序**，用于指导自动重构和代码生成。

**何时适用**：编写新 React 组件或 Next.js 页面；实现数据获取（客户端或服务端）；评审代码找性能问题；重构既有 React/Next.js 代码；优化包体积或加载时间。

**规则分类（按优先级）**：

| 优先级 | 类别 | 影响 | 前缀 |
|---|---|---|---|
| 1 | 消除瀑布流 | 关键 | `async-` |
| 2 | 包体积优化 | 关键 | `bundle-` |
| 3 | 服务端性能 | 高 | `server-` |
| 4 | 客户端数据获取 | 中高 | `client-` |
| 5 | 重渲染优化 | 中 | `rerender-` |
| 6 | 渲染性能 | 中 | `rendering-` |
| 7 | JavaScript 性能 | 低-中 | `js-` |
| 8 | 高级模式 | 低 | `advanced-` |

**快速索引（70 条，规则 ID 保持英文以便对应原文）**：

1. **消除瀑布流（关键）**：`async-cheap-condition-before-await`（await 标志位或远端值之前先检查廉价的同步条件）、`async-defer-await`（把 await 移进真正用到它的分支）、`async-parallel`（独立操作用 Promise.all）、`async-dependencies`（部分依赖用 better-all）、`async-api-routes`（API 路由里 Promise 尽早启动、晚 await）、`async-suspense-boundaries`（用 Suspense 流式输出内容）
2. **包体积（关键）**：`bundle-barrel-imports`（直接导入，别走 barrel 文件）、`bundle-analyzable-paths`（用可静态分析的导入与文件系统路径，避免过宽打包与跟踪）、`bundle-dynamic-imports`（重组件用 next/dynamic）、`bundle-defer-third-party`（分析/日志在水合后加载）、`bundle-conditional`（功能激活时才加载模块）、`bundle-preload`（hover/focus 时预加载提升体感速度）
3. **服务端性能（高）**：`server-auth-actions`（server action 要像 API 路由一样鉴权）、`server-cache-react`（React.cache() 做单请求去重）、`server-cache-lru`（跨请求用 LRU 缓存）、`server-dedup-props`（避免 RSC props 重复序列化）、`server-hoist-static-io`（字体/logo 等静态 IO 提升到模块级）、`server-no-shared-module-state`（RSC/SSR 里避免模块级可变请求状态）、`server-serialization`（最小化传给客户端组件的数据）、`server-parallel-fetching`（重构组件以并行取数）、`server-parallel-nested-fetching`（嵌套取数按条目并入 Promise.all）、`server-after-nonblocking`（非阻塞操作用 after()）
4. **客户端数据获取（中高）**：`client-swr-dedup`（SWR 自动请求去重）、`client-event-listeners`（去重全局事件监听）、`client-passive-event-listeners`（滚动用 passive 监听器）、`client-localstorage-schema`（localStorage 数据要版本化并最小化）
5. **重渲染优化（中）**：`rerender-defer-reads`（别订阅只在回调里用的状态）、`rerender-memo`（昂贵计算抽进 memo 组件）、`rerender-memo-with-default-value`（默认非原始类型 props 提升出去）、`rerender-dependencies`（effect 依赖用原始值）、`rerender-derived-state`（订阅派生布尔值而非原始值）、`rerender-derived-state-no-effect`（渲染期派生状态，别放 effect）、`rerender-functional-setstate`（函数式 setState 稳定回调）、`rerender-lazy-state-init`（useState 传函数做昂贵初始化）、`rerender-simple-expression-in-memo`（简单原始值别包 memo）、`rerender-split-combined-hooks`（按独立依赖拆分 hooks）、`rerender-move-effect-to-event`（交互逻辑放事件处理器）、`rerender-transitions`（非紧急更新用 startTransition）、`rerender-use-deferred-value`（推迟昂贵渲染保持输入响应）、`rerender-use-ref-transient-values`（高频瞬时值用 ref）、`rerender-no-inline-components`（别在组件内定义组件）
6. **渲染性能（中）**：`rendering-animate-svg-wrapper`（动画加在 div 包装上而非 SVG 元素）、`rendering-content-visibility`（长列表用 content-visibility）、`rendering-hoist-jsx`（静态 JSX 提出组件外）、`rendering-svg-precision`（降低 SVG 坐标精度）、`rendering-hydration-no-flicker`（仅客户端数据用内联 script）、`rendering-hydration-suppress-warning`（抑制预期内的水合不匹配警告）、`rendering-activity`（显隐用 Activity 组件）、`rendering-conditional-render`（条件渲染用三元不用 &&）、`rendering-usetransition-loading`（加载态优先 useTransition）、`rendering-resource-hints`（用 React DOM 资源提示预加载）、`rendering-script-defer-async`（script 标签用 defer 或 async）
7. **JavaScript 性能（低-中）**：`js-batch-dom-css`（CSS 改动按类或 cssText 分组）、`js-index-maps`（重复查找建 Map）、`js-cache-property-access`（循环里缓存对象属性）、`js-cache-function-results`（函数结果存模块级 Map）、`js-cache-storage`（缓存 localStorage/sessionStorage 读取）、`js-combine-iterations`（多次 filter/map 合成一趟循环）、`js-length-check-first`（昂贵比较前先查数组长度）、`js-early-exit`（函数尽早 return）、`js-hoist-regexp`（RegExp 创建提出循环）、`js-min-max-loop`（求最值用循环别用排序）、`js-set-map-lookups`（O(1) 查找用 Set/Map）、`js-tosorted-immutable`（不可变用 toSorted()）、`js-flatmap-filter`（一趟 flatMap 完成 map+filter）、`js-request-idle-callback`（非关键工作推迟到浏览器空闲）
8. **高级模式（低）**：`advanced-effect-event-deps`（别把 useEffectEvent 结果放进 effect 依赖）、`advanced-event-handler-refs`（事件处理器存 ref）、`advanced-init-once`（应用加载只初始化一次）、`advanced-use-latest`（useLatest 做稳定回调 ref）

**怎么用**：读单条规则文件看详细解释与代码示例（`rules/async-parallel.md`、`rules/bundle-barrel-imports.md`…）。每个规则文件含：为什么重要、错误示例+说明、正确示例+说明、补充上下文与参考。完整合订版见本技能目录的 `AGENTS.md`。

---

## 七、electron（Electron 应用自动化）

```yaml
name: electron
description: 用 agent-browser 经 Chrome DevTools Protocol 自动化 Electron 桌面应用（VS Code、Slack、Discord、Figma、Notion、Spotify 等）。需要与 Electron 应用交互、自动化桌面应用、连接运行中的应用、控制原生应用或测试 Electron 应用时使用。触发词包括"自动化 Slack 应用""控制 VS Code""和 Discord 应用交互""测试这个 Electron 应用""连接桌面应用"，或任何需要自动化原生 Electron 应用的任务。
allowed-tools: Bash(agent-browser:*), Bash(npx agent-browser:*)
```

> 派生自 vercel-labs/agent-browser（Apache-2.0），ZCode 适配。出处见 `THIRD-PARTY-NOTICES.md`。

# Electron 应用自动化

用 agent-browser 自动化任何 Electron 桌面应用。Electron 应用基于 Chromium，暴露一个 CDP 端口供 agent-browser 连接，从而复用与网页相同的"快照-交互"工作流。

**核心工作流**：① 带远程调试标志启动 Electron 应用 → ② agent-browser 连接 CDP 端口 → ③ 快照发现可交互元素 → ④ 用元素 ref 交互 → ⑤ 导航或状态变化后重新快照。

```bash
open -a "Slack" --args --remote-debugging-port=9222   # 启动并开启远程调试
agent-browser connect 9222                            # 连接
agent-browser snapshot -i                             # 快照
agent-browser click @e5                               # 交互
agent-browser screenshot slack-desktop.png            # 截图
```

**启动**：所有 Electron 应用都支持 `--remote-debugging-port`（Chromium 内置）。macOS 用 `open -a "应用名" --args --remote-debugging-port=9222`（VS Code 9223、Discord 9224、Figma 9225、Notion 9226、Spotify 9227）；Linux/Windows 直接给可执行文件加同样标志。**重要：应用已在运行时先退出，再带标志重启——该标志必须在启动时存在。**

**连接**：`agent-browser connect 9222`（连接后后续命令都指向该应用）；或每条命令带 `--cdp 9222`；或 `--auto-connect` 自动发现运行中的 Chromium 系应用。

**标签页管理**：Electron 常有多个窗口/ webview——`agent-browser tab` 列出全部目标，`tab 2` 按序号切换，`tab --url "*settings*"` 按 URL 模式切换。

**Webview 支持**：`<webview>` 元素自动发现，在 tab 列表里以 `type: "webview"` 出现，可像普通页面一样控制和交互（经原始 CDP 连接实现）。

**常用模式**：检查并导航应用（启动 → sleep 3 等待 → connect → snapshot → click → 重新 snapshot）；截图（普通/`--full`/`--annotate`）；提取数据（`get text @e5`、`snapshot --json > app-state.json`）；填表单（fill → press Enter → wait → snapshot）；多应用并行（`--session slack connect 9222`、`--session vscode connect 9223` 各自独立操作）。

**配色**：CDP 连接默认可能是亮色；`--color-scheme dark` 或 `AGENT_BROWSER_COLOR_SCHEME=dark` 保留暗色模式。

**故障排查**：连接被拒——确认启动时带了调试端口、应用先退出再重启、端口没被占用（`lsof -i :9222`）；连接失败——启动后等几秒（webview 初始化慢）；元素在快照里不出现——应用可能用多个 webview，用 `tab` 列目标切换；无法在输入框打字——试 `keyboard type`（当前焦点直接输入）或 `keyboard inserttext`（绕过按键事件，适配自定义输入组件）。

**支持的应​​用**：任何 Electron 应用——通讯（Slack、Discord、Teams、Signal、Telegram Desktop）、开发（VS Code、GitHub Desktop、Postman、Insomnia）、设计（Figma、Notion、Obsidian）、媒体（Spotify、Tidal）、效率（Todoist、Linear、1Password）。

---

## 八、dogfood（探索式测试 / 找 bug）

```yaml
name: dogfood
description: 系统地探索并测试 Web 应用，找出 bug、UX 问题和其他问题。被要求"dogfood""QA""探索式测试""找问题""找 bug""测试这个应用/网站/平台"或评审 Web 应用质量时使用。产出带完整复现证据的结构化报告——每个问题都有分步截图、复现视频和详细复现步骤——可直接交给负责团队。
allowed-tools: Bash(agent-browser:*), Bash(npx agent-browser:*)
disable-model-invocation: true
```

> 派生自 vercel-labs/agent-browser（Apache-2.0），ZCode 适配。

# Dogfood（吃自己的狗粮）

系统性地探索 Web 应用、发现问题，并**为每个发现产出带完整复现证据的报告**。

**准备**：只有**目标 URL** 是必填的，其余全有合理默认值，用户没明确覆盖就用默认。参数表：目标 URL（必填）、会话名（默认域名 slug 化，如 `vercel.com` → `vercel-com`）、输出目录（默认 `./dogfood-output/`）、范围（默认全应用）、认证（默认无）。用户说"dogfood vercel.com"就直接用默认值开工，**不要问澄清问题**（除非提到认证但没给凭据）。**始终直接用 `agent-browser`，绝不用 `npx agent-browser`**——直接二进制是快的 Rust 客户端，npx 走 Node.js 明显更慢。

**工作流**：① 初始化（建目录、复制报告模板、开命名会话）→ ② 认证（需要就登录并保存状态；OTP 码问用户）→ ③ 定位（初始标注截图与快照，摸清应用结构）→ ④ 探索（系统性地走每个页面测每个功能）→ ⑤ 记录（发现即记录，截图+录屏）→ ⑥ 收尾（更新统计、关会话）。

**初始化**：

```bash
mkdir -p {OUTPUT_DIR}/screenshots {OUTPUT_DIR}/videos
cp {SKILL_DIR}/templates/dogfood-report-template.md {OUTPUT_DIR}/report.md
agent-browser --session {SESSION} open {TARGET_URL}
agent-browser --session {SESSION} wait --load networkidle
```

**认证**：snapshot 找登录表单 ref → fill 账号密码 → click 提交 → wait networkidle；OTP/邮箱验证码问用户后输入；登录成功后 `state save` 保存状态复用。

**探索**：先读 `references/issue-taxonomy.md`（完整检查清单与严重度分级）。策略：从主导航开始逐节访问；每节测交互元素（点击、填表、开下拉/弹窗）；测边界情况（空状态、错误处理、边界输入）；走真实端到端流程（创建/编辑/删除）；定期查浏览器控制台错误。每个页面执行 snapshot、标注截图、`errors`、`console`。深度自行判断：核心功能多花时间，边缘页面少花；某区域问题扎堆就深挖。

**记录问题（复现优先）**——第 4、5 步同时进行，**发现即停下记录，不许先逛完整站再补记录**。每个问题必须可复现，不能只记一笔——要用证据证明，目标是读者能看清发生了什么并一步步重放：

- **交互/行为类问题**（需要用户操作复现）：复现**之前**先开录屏（`record start .../issue-{NNN}-repro.webm`）→ 以人类节奏走步骤（动作之间暂停 1-2 秒，每步截图）→ 捕获坏掉的状态（暂停 2 秒再标注截图）→ 停录屏 → 在报告里写编号复现步骤，每步引用对应截图。
- **静态/加载即可见的问题**（错别字、占位文案、文字被裁、错位、加载时的控制台错误）：无需交互，一张标注截图足够，不录屏，报告里 Repro Video 填 `N/A`。

对所有问题：**立即追加到报告**（不要攒到最后，会话中断就全丢）；问题编号递增（ISSUE-001…）。

**收尾**：目标是 **5-10 个记录充分的问题**就收——证据深度比总数重要，5 个带完整复现的胜过 20 个含糊描述。然后：重读报告更新严重度统计（每个 `### ISSUE-` 块都必须反映在总数里）→ `close` 关会话 → 告诉用户报告就绪，总结发现（总数、严重度分布、最关键项）。

**指导原则**：复现就是一切，但证据要和问题匹配（交互 bug 要视频+分步截图；静态 bug 一张标注截图即可）；**收集证据前先验证可复现**（至少重试一次还能重现才算有效问题）；静态问题不录屏；交互问题每步截图（前、动作、后）；复现步骤要能对上截图（读者不动浏览器也能看图走完）；用对快照命令（`snapshot -i` 找可点/可填元素，`snapshot` 无参数读页面内容）；彻底但有判断力（你不是在跑测试脚本，是在像真实用户一样探索）；**增量写发现**；**绝不删除输出文件**（不 rm 截图/视频/报告，不关会话重来，只能向前不能倒退）；**绝不读被测应用的源码**（以用户身份测试，不是审计代码——所有发现必须来自浏览器里观察到的现象）；**查控制台**（很多问题 UI 看不见但表现为 JS 错误或请求失败）；像用户一样测（走常见端到端流程、点真人会点的东西、填真实数据）；**像人一样打字**（录屏时填表用 `type` 逐字输入而不是 `fill`，只有录屏之外追求速度才用 `fill`）；**复现视频要有人的节奏**（动作间 `sleep 1`、最终截图前 `sleep 2`，1 倍速可看，不是瞬间状态切换的模糊一片）；命令讲效率（独立命令一条 shell 里批量执行；滚动用 `scroll down 300`，不要用 `key` 或 `evaluate`）。

**参考**：`references/issue-taxonomy.md`（会话开始时读：校准找什么、严重度分级、探索清单）。**模板**：`templates/dogfood-report-template.md`（复制到输出目录当报告文件）。

---

## 九、agent-browser（浏览器自动化 CLI）

```yaml
name: agent-browser
description: 面向 AI 代理的浏览器自动化 CLI。需要与网站交互时使用——导航页面、填表单、点击按钮、截图、提取数据、测试 Web 应用或任何浏览器任务。触发词包括"打开网站""填表""点按钮""截图""抓取页面数据""测试这个 Web 应用""登录某个网站""自动化浏览器操作"或任何需要程序化网页交互的任务。
allowed-tools: Bash(npx agent-browser:*), Bash(agent-browser:*)
```

> 派生自 vercel-labs/agent-browser（Apache-2.0），ZCode 适配。

# 用 agent-browser 做浏览器自动化

CLI 直接经 CDP 驱动 Chrome/Chromium。安装：`npm i -g agent-browser` / `brew install agent-browser` / `cargo install agent-browser`；`agent-browser install` 下载 Chrome，`agent-browser upgrade` 升级。

**核心工作流**（每个浏览器自动化都遵循此模式）：导航 `open <url>` → 快照 `snapshot -i`（拿到 `@e1`、`@e2` 等元素 ref）→ 用 ref 交互（click/fill/select）→ 导航或 DOM 变化后重新快照拿新 ref。

```bash
agent-browser open https://example.com/form
agent-browser snapshot -i
# 输出：@e1 [input type="email"], @e2 [input type="password"], @e3 [button] "Submit"
agent-browser fill @e1 "user@example.com"
agent-browser fill @e2 "password123"
agent-browser click @e3
agent-browser wait --load networkidle
agent-browser snapshot -i  # 检查结果
```

**命令链**：浏览器在命令之间由后台守护进程保持，所以 `&&` 串联安全且比分开调用更高效。不需要读中间输出的步骤可链式（open + wait + screenshot）；需要先解析输出再决定下一步的（先 snapshot 拿 ref 再交互）就分开跑。

**处理认证（5 种方案，按场景选）**：
1. **从用户浏览器导入认证**（一次性任务最快）：`--auto-connect state save ./auth.json`（连用户已登录的 Chrome）→ 用该状态 `--state ./auth.json open ...`。状态文件含明文 session token——加进 `.gitignore`、不用了就删；可设 `AGENT_BROWSER_ENCRYPTION_KEY` 加密存储。
2. **持久化 profile**（周期性任务最省事）：`--profile ~/.myapp` 首次手动登录，之后每次启动都已是登录态。
3. **命名会话**（自动保存/恢复 cookie + localStorage）：`--session-name myapp` 登录一次，close 时自动保存，下次自动恢复。
4. **认证保险库**（凭据加密存储，按名登录，**LLM 全程看不到密码**）：`auth save myapp --url ... --username user --password-stdin` → `auth login myapp`；`auth login` 会先等登录表单选择器出现再填写，对慢加载的 SPA 登录页更可靠。
5. **状态文件**（手动存取）：`state save ./auth.json` / `state load ./auth.json`。

OAuth、2FA、cookie 认证、token 刷新等模式见 `references/authentication.md`。

**常用命令**（按组，完整清单见 `references/commands.md`）：

```bash
# 导航：open <url>（别名 goto、navigate）/ close
# 快照：snapshot -i（带 ref 的可交互元素，推荐）/ snapshot -s "#selector"（限定 CSS 选择器）
# 交互：click @e1 / click @e1 --new-tab / fill @e2 "text"（清空再输入）/ type（追加输入）/ select 下拉
#       check 勾选 / press Enter / keyboard type/inserttext / scroll down 500 [--selector ...]
# 取信息：get text @e1 / get url / get title / get cdp-url
# 等待：wait @e1 / wait --load networkidle / wait --url "**/page" / wait 2000（毫秒）
#       wait --text "Welcome"（文本出现）/ wait --fn "条件"（JS 条件，如等 Loading 消失）/ wait "#spinner" --state hidden
# 下载：download @e1 ./file.pdf / wait --download ./output.zip / --download-path 设默认目录
# 网络：network requests [--type xhr,fetch] [--method POST] [--status 2xx] / network request <id>
#       network route "**/api/*" --abort（拦截）/ network har start|stop ./capture.har
# 视口：set viewport 1920 1080 [2]（2=2x retina）/ set device "iPhone 14"
# 捕获：screenshot [--full] [--annotate] [--screenshot-dir] [--screenshot-format jpeg --screenshot-quality 80] / pdf output.pdf
# 剪贴板：clipboard read|write|copy|paste
# 对比：diff snapshot [--baseline before.txt] / diff screenshot --baseline before.png（像素差异）/ diff url <url1> <url2> [--selector]
```

**批量执行**：把 JSON 字符串数组管道给 `batch`（多步流程避免逐条启动进程的开销；`--bail` 首个错误即停）。适合"命令序列已知、不依赖中间输出"时；需要解析中间输出就用分步或 `&&`。

**常用模式**：表单提交（open→snapshot→fill→select→check→click→wait）；认证保险库登录（推荐，密码经 stdin 管道传入避免 shell 历史泄漏）；状态持久化复用；会话持久化（自动存 cookie/localStorage 到 `~/.agent-browser/sessions/`，`state list|show|clear|clean --older-than 7` 管理）；iframe（内容自动内联到快照，ref 自带 frame 上下文直接交互；`frame @e2` 限定快照范围，`frame main` 回主框架）；数据提取（`get text`、`snapshot --json` 便于解析）；并行会话（`--session site1/site2` 各开各的）；连接已有 Chrome（`--auto-connect`，经 DevToolsActivePort/常用调试端口发现）；暗色模式（`--color-scheme dark` 或 `set media dark`）；响应式测试（改 viewport 尺寸截图、`set device` 模拟设备、2x retina）；可视化调试（`--headed`、`highlight @e1`、`inspect` 开 DevTools、`record start demo.webm` 录屏、`profiler start|stop` 性能分析）；本地文件（`--allow-file-access open file:///...`）；iOS 模拟器（`-p ios --device "iPhone 16 Pro"`，需要 macOS + Xcode + Appium）。

**安全（全部可选，默认无限制）**：
- **内容边界**（推荐给 AI 代理）：`--content-boundaries` 把页面输出包在带 nonce 的标记里，帮 LLM 区分工具输出与不可信页面内容（防提示注入）。
- **域名白名单**：`AGENT_BROWSER_ALLOWED_DOMAINS="example.com,*.example.com"` 限制导航，子资源/WebSocket/EventSource 也拦；记得包含目标页依赖的 CDN 域名。
- **动作策略**：`AGENT_BROWSER_ACTION_POLICY=./policy.json`，如 `{ "default": "deny", "allow": ["navigate","snapshot","click",...] }`。
- **输出上限**：`AGENT_BROWSER_MAX_OUTPUT=50000` 防止大页面淹没上下文。

**Diff（验证改动生效）**：动作后 `diff snapshot` 对比当前无障碍树与上一次快照（`+` 新增 `-` 移除，类 git diff）；视觉回归用 `diff screenshot --baseline baseline.png`（变更像素标红 + 差异百分比）；`diff url` 对比两个页面（如 staging vs 生产）。

**超时与慢页面**：默认 25 秒（`AGENT_BROWSER_DEFAULT_TIMEOUT` 可改）。慢站用显式等待代替默认超时：`wait --load networkidle`（等网络活动平息，最适合慢页）、`wait <selector>`/`wait @ref`（等特定元素）、`wait --url`（等跳转）、`wait --fn`（等 JS 条件）、`wait 5000`（固定时长，最后手段）。

**会话管理与清理**：并发多代理一律用命名会话避免冲突；结束务必 `close`（防进程泄漏）；上回没关干净先 close 清理；`AGENT_BROWSER_IDLE_TIMEOUT_MS=60000` 可在闲置后自动关闭守护进程（适合 CI/临时环境）。

**ref 生命周期（重要）**：页面变化时 ref 会失效——点击导航链接/按钮、表单提交、动态内容加载（下拉/弹窗）之后**必须重新快照**再交互。

**标注截图（视觉模式）**：`screenshot --annotate` 在可交互元素上叠加编号标签 `[N]` 对应 ref `@eN`，同时缓存 ref 可直接交互。适用：无标签的图标按钮、需核实视觉布局/样式、Canvas/图表（文本快照看不见）、需要元素位置的空间推理。

**语义定位器（ref 的替代）**：ref 不可用/不可靠时——`find text "Sign In" click`、`find label "Email" fill "..."`、`find role button click --name "Submit"`、`find placeholder "Search" type "..."`、`find testid "submit-btn" click`。

**JS 求值（eval）**：shell 引号会破坏复杂表达式——复杂 JS 用 `eval --stdin <<'EVALEOF' ... EVALEOF`（推荐），程序化生成用 `eval -b`（base64，绕开一切 shell 转义）。规则：单行无嵌套引号用普通 `eval '表达式'`；嵌套引号/箭头函数/模板字符串/多行用 `--stdin`；生成脚本用 base64。

**配置文件**：项目根 `agent-browser.json` 持久化设置（headed/proxy/profile…）。优先级（低到高）：`~/.agent-browser/config.json` < `./agent-browser.json` < 环境变量 < CLI 标志。所有 CLI 选项映射为 camelCase 键；布尔标志接受 true/false 值覆盖配置；用户与项目配置的扩展会合并而非替换。

**引擎选择**：`--engine chrome`（默认，经 CDP 的 Chrome/Chromium）或 `--engine lightpanda`（无头浏览器，速度与内存均为 Chrome 的 1/10；不支持 `--extension`/`--profile`/`--state`/`--allow-file-access`）。

**深度文档（按需读）**：`references/commands.md`（完整命令参考）、`references/snapshot-refs.md`（ref 生命周期与排障）、`references/session-management.md`（并行会话/状态持久化/并发抓取）、`references/authentication.md`（登录/OAuth/2FA/状态复用）、`references/video-recording.md`（录屏工作流）、`references/profiling.md`（性能分析）、`references/proxy-support.md`（代理配置/地域测试）。

**模板**：`templates/form-automation.sh`（带校验的表单填写）、`templates/authenticated-session.sh`（登录一次复用状态）、`templates/capture-workflow.sh`（内容提取+截图）。
