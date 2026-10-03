# 03 · 前端状态所有权与数据处理逻辑

> 前缀 `SRC = packages/ui/src`。这是"前端处理逻辑"的核心文档：谁拥有什么状态、消息怎么进 store、发送一条消息要经过什么。

## 一、状态所有权总表（先背下来）

| 状态 | 所有者 | 文件 | 说明 |
|---|---|---|---|
| **消息列表（会话事实）** | `ConversationProjectionStore`（**不是 Zustand**） | `SRC/v4/conversationProjectionStore.ts` | plain class + `useSyncExternalStore`；唯一写入方是订阅推送 |
| 会话连接生命周期 | `SessionDataLayer` | `SRC/v4/sessionDataLayer.ts` | `Map<topic, store>` + 引用计数 + 30s keep-warm（关 pane ≠ 停 session） |
| UI 运行态（按 workspace 分桶） | `zcodeSessionStore`（Zustand） | `SRC/store/zcodeSessionStore.ts` + `*Types/WorkspaceSlice/TaskSlice/Selectors/Navigation` | 任务状态、权限弹窗队列、草稿、模型选择、乐观任务项 |
| 任务列表缓存 | `taskQueryCacheStore` | `SRC/store/taskQueryCacheStore.ts` | 分片查询缓存 + unread overlay |
| 会话索引 | `sessionsIndexStore` | `SRC/v4/sessionsIndexStore.ts` | sessions-index topic 订阅（与 conversation 同款 snapshot/delta 策略） |
| 输入框草稿 | `composerDraftStore` | `SRC/v4/composer/composerDraftStore.ts` | |
| 命令账本（跨刷新） | `pendingCommandRegistry` | `SRC/v4/pendingCommandRegistry.ts` | localStorage 持久化（key `zcode-v4-pending-commands:v1`，TTL 24h） |
| tab / 分屏 / 远端连接 | `tabStore` / `paneLayoutStore` / `remoteWorkspaceSessionStore` | `SRC/store/`、`SRC/v4/` | |

**关键设计**：store 层对 services **零直接依赖**（只有 `import type`），连接只发生在 hooks/context 层——这保证了对话内核可以脱离 Node/Electron 运行。

## 二、ConversationProjectionStore（最值得精读的一个文件）

位置：`SRC/v4/conversationProjectionStore.ts`（约 60KB 单文件原子状态机）。订阅 hook：`SRC/v4/useConversationProjection.ts`。

**三条客户端铁律**（头注释明说，抄架构先抄这三条）：

1. snapshot 帧 → **整体替换，绝不 merge**；
2. delta 帧仅在区间衔接（`frame.fromSeq === snapshot.seq`）时 apply，**断档不猜**——携当前水位重订阅，由服务端裁决 resume 还是重发 snapshot；
3. 除 optimistic overlay 外，store 不产生任何会话事实。

**核心 state**（`ConversationStoreState`）：`status("connecting"|"live"|"error"|"closed")`、`snapshot: ConversationSnapshot`（含 `rows.window` 尾窗 60 行、`queue`、`pendingInteractions`、`pendingCommands`、`usage`、`inputRouting`…）、`subscriptionId`、`optimisticCommands`、`loadingOlder`、`sessionPlans`。

**帧应用**：`applyFrame()`（约 648-758 行）——snapshot 整体替换；delta 迟到帧（`toSeq <= current.seq`）静默丢弃；断档 → `requestRecovery()`；衔接 → `applyConversationDeltas()`（来自 `@zcode/shared/zcode-protocol-v4`），流式文本就是 delta op `row.delta`（`path` + `append` 字符串）。

**恢复状态机**：`requestRecovery / issueRecovery / settleRecovery`——resume 优先、超时升级 forceSnapshot、`fault.subscription.notOwned` 特判自动重订阅；runtime 换代走 `handleRuntimeRestart`（250ms/1s/3s 有界退避）。

## 三、乐观更新的三层机制（抄"发送不卡顿"的实现)

1. **命令级 optimistic overlay**（`conversationProjectionStore.ts`）：
   发送前 `markCommandPending({commandId,...})` 进 `optimisticCommands`（输入框清空、气泡 pending）→ 权威投影出现同 `commandId` 时 `reconcileOptimistic()` 让 overlay 退场 → ACK 非 accepted 或传输错误 → `settleCommand` 撤销。`sendText` accepted 后 `expectAcceptedInputProjection` 启动 2s watchdog，投影静默则复用订阅恢复。
2. **renderer 命令账本**（`SRC/v4/pendingCommandRegistry.ts` + `pendingCommandReplay.ts`）：
   localStorage 持久化，`record → applyAck → applyQuery → reconcileSnapshot → reconcileSession`；renderer 刷新/ACK 丢失后用 `v4/commands/query` 与 CLI 权威事实对账；**丢弃的命令绝不自动重放**，经 `usePendingCommandRecovery` + `PendingCommandRecoveryBanner.tsx` 让用户手动重发。信封工厂 `SRC/v4/commandFactory.ts`：`commandId = uuidv7`（重试不变）、`clientId` 持久化（`zcode-v4-client-id:v1`）、CAS 命令缺 `baseRevision` 就地抛错。
3. **任务列表乐观项**：`SRC/store/zcodeSessionStoreTaskSlice.ts` 的 `upsertOptimisticTaskListItem` + hook `SRC/hooks/workspaceTaskListOptimisticOverlay.ts`（`mergeWorkspaceTaskListItemsWithOptimistic`：只合并已可见任务和 active task，防服务端旧 meta 压回）。

## 四、hooks / context 层（组件怎么拿到数据）

| Hook/Provider | 文件 | 职责 |
|---|---|---|
| `useServices` / `ServiceProvider` | `SRC/hooks/useServices.tsx` | React Context 提供 `IServiceAccessor`（约 40 个服务的聚合接口，定义在 `packages/services/src/accessor.ts`） |
| `useWorkspaceServices` | `SRC/hooks/useWorkspaceServices.tsx` | **本地/远端服务路由核心**：远端 tab → `remoteWorkspaceSessionStore.sessionsById[id].services`；无服务远端 → 断连代理（所有 call 拒绝） |
| `V4ConversationProvider` / `useV4Conversation` | `SRC/v4/V4ConversationContext.tsx` | 每 workspace 一条 host 连接 + 一个 SessionDataLayer；分屏 pane 经 `SRC/v4/workspaceConnectionRegistry.ts` 租用连接（refCount + keep-warm）。暴露 `{layer, sendCommand, attachmentPut, ...}`。transport 由 `createAgentConversationTransport(services.zcodeAgentService, {workspacePath, workspaceIdentity})` 构造 |
| `useWorkspaceTaskLists` | `SRC/hooks/useWorkspaceTaskLists.ts` | 任务列表：分片查询 + 缓存 + 乐观合并 ← `SRC/v4/useWorkspaceSessionsIndexItems.ts` ← `buildTaskListResultFromSessions.ts` |

传输接缝：**`SRC/v4/transport.ts` 定义 `ConversationTransport` 接口**——desktop 走 host（实现 `SRC/v4/agentConversationTransport.ts`：握手 hello + 能力位 + TTFT 时钟校准 + ACK activation barrier + `TopicWireFrameAssembler` 组帧 + zod 校验），web 走 WebSocket relay，静态回放用 `SRC/v4/replaceableConversationTransport.ts`。**换后端 = 实现这个接口**。

## 五、发送一条消息的完整前端链路（SessionPane.tsx 为中心）

```
ConversationComposer.tsx 提交
  → composerSubmissionConfig.ts：createComposerSubmissionConfig
      点击瞬间把 composer 意图冻结为 {modelSelection, mode, planEnabled}（防 await 后用户切模）
  → SessionPane.tsx dispatchCommand("sendText" | "createSession")
      ① commandFactory.createCommandEnvelope()   → uuidv7 commandId + 持久 clientId
      ② pendingCommandRegistry.record(envelope)  → localStorage 账本（先于上行）
      ③ lease.store.markCommandPending()         → optimisticCommands（输入框清空、气泡 pending）
      ④ useV4Conversation().sendCommand(envelope)
  → agentConversationTransport：ensureAgentV4ConnectionHandshake → agentService.sendConversationCommandV4
  → RPC（packages/rpc ProxyChannel）→ MessagePort → host 进程 → CLI
  ... ACK 回来：
      pendingCommandRegistry.applyAck；accepted → 2s projection watchdog；非 accepted → settleCommand
  ... 流式帧回来：
      barrier 放行 → TopicWireFrameAssembler 组逻辑帧 → conversationTopicFrameSchema 校验
      → SessionDataLayer.onFrame 按 topic 扇入 → ConversationProjectionStore.applyFrame
      → reconcileOptimistic（pending 气泡转正）
      → useConversationProjection（useSyncExternalStore）→ SessionPane 重渲染
```

配套细节：
- **草稿预热**：`SRC/v4/composer/useDraftSessionPrewarm.ts`——pane 无会话时后台建 phase=draft 会话，首发直接 `sendText` 复用；transport error 不自动重发（防跑两遍）。
- **权限/问答**：`SRC/v4/V4InteractionDialogs.tsx` 读 `snapshot.pendingInteractions`，`resolveInteraction` 命令回执；Esc → `dispatchCommand("stop")`。
- **队列**：snapshot.queue 的增删改排序对应 `sendQueuedNow / editQueueItem / reorderQueueItem / deleteQueueItem / setAutoDrain` 命令，UI 在 `SRC/v4/ConversationQueuePanel.tsx`。

## 六、建议精读顺序

1. `SRC/v4/conversationProjectionStore.ts` —— 只读头注释 + applyFrame + 恢复状态机，建立"权威投影"心智模型
2. `SRC/v4/transport.ts` + `agentConversationTransport.ts` —— 接缝与传输
3. `SRC/v4/SessionPane.tsx` —— 只看 `dispatchCommand` 与提交流程（文件 4834 行，别通读）
4. `SRC/v4/pendingCommandRegistry.ts` + `commandFactory.ts` —— 可靠发送
5. `SRC/store/zcodeSessionStoreTypes.ts` —— UI 运行态有哪些
6. `SRC/v4/useWorkspaceSessionsIndexItems.ts` —— 列表订阅怎么复用同一套 snapshot/delta 策略
7. `packages/shared/src/zcode-protocol-v4/snapshot.ts` + `delta.ts` —— 数据形状
