# 05 · Skills 机制与提示词 / 记忆工程（Agent 侧）

> 位置：`apps/zcode-cli/packages/`（下文 `CLI = apps/zcode-cli/packages`）。虽然你是前端为主，但这套"SKILL.md 两段式注入 + 提示词分层缓存 + 记忆四件套"的设计模式，在前端做 AI 产品时直接可用。

## 一、CLI 包结构速览

| 子包 | 职责 |
|---|---|
| `CLI/contracts` | 全部类型与端口接口（SkillPort、SessionEvent、Compact、Tools）；事件定义 `src/events/session.events.ts` |
| `CLI/adapters` | 端口的 Node 实现：config、context、exec、fs、model、mcp、plugins、**skills**、storage |
| `CLI/core` | Agent 核心：`agent`（turn 状态机）、`compact`（压缩）、`context`（**system prompt 组装**）、`memory`、`tool/handlers`、`runtime`、`subagent`、`system-reminder` |
| `CLI/bootstrap` | 装配层：`skills.ts`、`custom-commands.ts`、`slash-command-surface.ts`；`zcode-protocol-v4/` 负责事件归一化与产品投影 |
| `CLI/cli` | 命令行入口：`main.ts`、`run.ts`、`skills-command.ts`（`zcode skills list`） |
| `CLI/tui` | Ink 终端 UI（`app-session-event-handler.ts` 消费 SessionEvent） |
| `CLI/bundled-skills` | 内置技能（`skills/dynamic-workflows/`） |
| `CLI/dynamic-workflow(-runtime)` | 动态工作流编译器/引擎 |

## 二、Skills 机制：两段式注入

**设计精髓**：技能**列表**常驻上下文（让模型知道有什么），技能**正文**按需经工具加载（省 token）。

### 发现与解析

| 文件 | 内容 |
|---|---|
| `CLI/adapters/src/skills/index.ts` | `NodeSkillAdapter implements SkillPort`：`discoverSkills / loadSkill / parseSkill / createNodeSkillAdapter`。frontmatter 是**自写扁平 YAML 解析**（`extractFrontmatter / parseFlatYaml`，专门支持 `description: >` 折叠块标量）；合法键白名单 `SAFE_FRONTMATTER_KEYS`（name/description/when_to_use/license/metadata）；正文上限 100KB 截断 |
| `CLI/adapters/src/skills/scan.ts` | `scanSkillFilesUnderRoot()`：根目录自身有 SKILL.md 先算一个，再扫一层子目录；不跟随符号链接（安全注释细） |
| `CLI/adapters/src/skills/roots.ts` | `resolveDefaultSkillRoots()`：技能根 = 项目 skills 配置根 + 用户级 `~/.zcode/skills`、`~/.agents/skills` + 项目级逐层 `.zcode/skills`、`.agents/skills` + 插件技能根 + bundled |
| `CLI/contracts/src/skills/index.ts` | SkillPort / SkillMetadata / SkillContent / SkillDiagnostic 契约 |
| `CLI/bootstrap/src/skills.ts` | `listZCodeSkills() / inspectZCodeSkill()`：CLI 命令与 runtime 共用同一 discovery |

### 两条注入路径（精读这两段）

1. **列表注入**：`CLI/core/src/context/sections/skills.ts` 的 `buildSkillsSection()`——`injectionTarget: "meta_user"`（以 meta user attachment / system-reminder 附在对话最前，source 为 `skills_listing`），**预算 20K token，超了降级为"仅名字+路径"列表**。总装配在 `CLI/core/src/context/builder.ts`（只有工具表里有 Skill 工具时才注入）。
2. **正文注入**：`CLI/core/src/tool/handlers/skill.ts` 的 `skillToolEntry`（工具名 `"Skill"`）——handler 调 `skillPort.loadSkill()`，正文包成 `<skill_content name="...">...</skill_content>` **作为 tool result 注入当前对话**（不是 system prompt）；支持 `${ZCODE_SKILL_DIR}` 变量展开。**工具 description 本身就是一段"调用纪律"提示词范本**："BLOCKING REQUIREMENT：先调 Skill 工具再回答"、"看到 `<command-name>` 标签说明已加载过"。
3. 用户输入 `/<skill-name>`：`CLI/core/src/context/dynamic-sections.ts` 只给一行提示"invoke it via Skill"，模型自己调工具——没有独立的命令展开器。
4. 加载门：`CLI/core/src/agent/loaded-skills.ts` 的 `sessionHasLoadedSkill()`：以 provider 可见历史里有无成功的 Skill 调用为准（compaction 挤掉后门自动关上——细节设计）。

### SKILL.md 怎么写（学习样本）

- 仓库自带 8 个（`.agents/skills/`）：`agent-browser`（浏览器自动化）、`ai-elements`（**搭聊天界面的教程，抄 UI 前先读**）、`architecture-governance`、`dep-refs`、`dogfood`（探索式 QA）、`electron`、`feature-boundary-planner`、`react-best-practices`。
- **渐进披露范本**：`CLI/bundled-skills/skills/dynamic-workflows/`——SKILL.md 只放核心，`examples.md` / `patterns.md` 作附加资源按需读。这是"一个 skill 多文件"的标准结构。

## 三、System prompt 分层组装（缓存友好）

**全部内联在 TS 字符串里，没有 .md 模板文件。** 总装入口：`CLI/core/src/context/builder.ts` 的 `ContextBuilder.build()`——产出 **3 条 system message**（cli_prefix / stable body / dynamic body，各带 `cacheControl: ephemeral`）+ meta_user attachments。这个"稳定段/动态段分层 + 逐条 ephemeral 缓存"是为了命中 provider 的 prompt cache，值得照抄。

段落清单（`CLI/core/src/context/sections/` 与 `dynamic-sections.ts`）：

| 段 | 文件 | 学什么 |
|---|---|---|
| 前缀 | `cli-prefix.ts` | 一句身份（稳定缓存） |
| 身份+Harness | `identity.ts` | 身份行 + `SECURITY_NOTICE` + `buildHarnessBlock()`（markdown 输出、权限模式、并行工具调用、`file_path:line_number` 可点击引用）；被子代理身份 `CLI/core/src/subagent/workflow-actor.ts` 逐字复用 |
| **沟通风格** | `dynamic-sections.ts` 的 `buildDynamicBehaviorSection` | **COMMUNICATION_PROMPTS：先说结论、可读性优先于简洁、不要箭头链、注释只写代码说不出的约束**——可直接复用的输出风格提示词 |
| 会话引导 | 同上 `buildSessionGuidanceSection` | skill slash 提示 |
| **上下文管理** | 同上 `buildContextManagementSection` | **CONTEXT_MANAGEMENT_PROMPTS**："对话变长会自动摘要" + 自主执行 + turn 结束前检查等行为规范 |
| 环境/日期/桌面 | `env-info.ts` / `current-date.ts` / `desktop.ts` | |
| 记忆 | `memory.ts` | 见第五节 |
| 用户指令 | `request-user-context.ts` | AGENTS.md + MEMORY.md 索引注入（`# agentsMd`，附 "OVERRIDE any default behavior"） |

其他大块提示词：`CLI/core/src/tool/handlers/plan-mode-prompts.ts`（**"工具描述即提示词"的典范**，EnterPlanMode 的超长 when-to-use）、`CLI/core/src/subagent/system-prompt.ts`（子代理）、`CLI/core/src/system-reminder/incoming-message.ts`（steer/peer/task-notification 的包装话术）。

## 四、压缩（Compaction）提示词工程

| 文件 | 内容 |
|---|---|
| `CLI/core/src/compact/prompt.ts` | `buildCompactPrompt()` = `NO_TOOLS_PREAMBLE`（禁调工具）+ `BASE_COMPACT_PROMPT`：`<analysis>`/`<summary>` 两段结构 + **9 节模板**（Primary Request and Intent / Key Technical Concepts / Files and Code Sections / Errors and fixes / Problem Solving / **All user messages** / Pending Tasks / Current Work / Optional Next Step；要求逐字保留安全相关指令与用户反馈）；支持用户自定义 compact instructions。`buildCompactSummaryMessage()` 压缩后续接消息 |
| `CLI/core/src/compact/policy.ts` | `shouldAutoCompact`：默认 200K 窗口、输出预留 32K/21K、buffer 13K、连续失败熔断 3 次 |
| `CLI/core/src/compact/microcompact.ts` | **本地微压缩**：不调模型，直接清旧 tool result |
| `CLI/core/src/compact/rounds.ts` / `CLI/core/src/agent/compact-session.ts` | 按 assistant 轮分组 / 会话侧入口 |

## 五、记忆四件套

1. **AGENTS.md / 用户指令注入**：`CLI/adapters/src/context/index.ts` 的 `resolveUserInstructions()`（`DEFAULT_PRIORITY_FILES = ["AGENTS.md"]`，收集 workspace 与用户级 `~/.zcode/AGENTS.md`）→ `request-user-context.ts` 注入 meta_user。
2. **文件式自动记忆**：
   - 协议提示词：`CLI/core/src/context/sections/memory.ts` 的 `buildMemorySection()`——memory 目录下**每个事实一个 md 文件**（frontmatter `type: user|feedback|project|reference`），`[[name]]` 互链，`MEMORY.md` 是每次会话注入的索引。
   - 索引注入：`CLI/core/src/runtime/methods/context.ts` → `CLI/core/src/memory/index-content.ts`（格式化/截断 200 行 / 25KB）→ `request-user-context.ts`（"user's auto-memory, persists across conversations"）。
   - **后台记忆抽取 agent**：`CLI/core/src/memory/extraction.ts`（`buildMemoryExtractionPrompt()`：扮演 memory extraction subagent、限制工具与轮次、"Nothing to save." 协议、已有记忆清单防重复）+ `memory-agent-loop.ts`（**复用主工具目录但权限收窄：只许 Read/Grep/Glob/只读 Bash，Write/Edit 仅限 memory 目录**）+ `CLI/core/src/runtime/helpers/project-memory-extraction.ts`（每轮后调度、5 turn 上限）。清单扫描 `CLI/core/src/memory/recall/manifest.ts`。
   - **完整版记忆提示词（最值得摘抄）**：`CLI/core/src/subagent/persistent-memory-prompt.ts` 的 `PERSISTENT_AGENT_MEMORY_PROMPT_TEMPLATE`——每种 type 都有 `<when_to_save>/<how_to_use>/<examples>` 三段式。
3. **上下文压缩**：即第四节，system prompt 的 Context Management 段就是告诉模型这件事。
4. **记忆开关**：`CLI/bootstrap/src/app/runtime-config.ts` 汇总（enabled/extractionEnabled/use）。

## 六、Agent 核心事件流（简要）

- 事件定义：`CLI/contracts/src/events/session.events.ts`——`SessionEvent{sessionId, turnId, type, payload}` + 几十种 `SessionEventType`（TurnStarted / ModelStreaming / ToolCallStarted / ToolCallResult / SessionCompacted / PermissionRequested…）。
- 产生：`CLI/core/src/runtime/agent-runtime.ts`（方法在 `runtime/methods/`）；turn 状态机 `CLI/core/src/agent/turn-machine.ts`（start → startModelRequest → receiveModelResponse → scheduleTools → completeTool → aggregateResults → complete）。
- 归约：`CLI/contracts/src/events/event-reducer.ts`（`EventReducer/reduce/apply` → `SessionProjection`）→ 桌面协议侧 `CLI/bootstrap/src/zcode-protocol-v4/event-normalizer.ts` → UI（见 04 文档）。
- Todo 工具：`CLI/core/src/tool/handlers/todo.ts`；Plan 工具：`plan-mode.ts`（提示词同目录）。

## 七、建议精读顺序

1. 一个真实 SKILL.md：`.agents/skills/ai-elements/SKILL.md`（和你抄 UI 直接相关）+ `CLI/bundled-skills/skills/dynamic-workflows/SKILL.md`（多文件结构）
2. `CLI/adapters/src/skills/index.ts` —— frontmatter 解析（`parseFlatYaml` 的块标量处理是自研解析的坑点示范）
3. `CLI/core/src/tool/handlers/skill.ts` —— tool result 注入 + 调用纪律写法
4. `CLI/core/src/context/builder.ts` + `sections/skills.ts` —— 分层缓存 + 20K 预算降级
5. `CLI/core/src/context/dynamic-sections.ts` —— COMMUNICATION_PROMPTS / CONTEXT_MANAGEMENT_PROMPTS 直接摘抄
6. `CLI/core/src/compact/prompt.ts` —— 9 节摘要模板
7. `CLI/core/src/context/sections/memory.ts` ↔ `CLI/core/src/subagent/persistent-memory-prompt.ts` —— 精简版 vs 完整版记忆协议对照
8. `CLI/core/src/memory/extraction.ts` + `memory-agent-loop.ts` —— 后台抽取 agent 的权限收窄思路
