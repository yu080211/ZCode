# 04 · 对话运行机制（端到端）

> UI 之下发生了什么：协议、传输、Host、CLI 权威侧、投递档位、断线恢复。读懂这层才能解释 03 文档里每一步"为什么"。

## 一、总架构

**命令上行 + 快照/增量下行的"单一权威投影"**。消息事实的唯一权威在 CLI 进程（`apps/zcode-cli`）；任何客户端（桌面 renderer、手机 Web、分享页）都是同一份权威的投影。这带来一个关键性质：**冷启动、刷新、断线重连是同一件事**——携水位重订阅，由权威侧裁决。

```
Renderer(React) ──MessagePort/RPC──> Host 进程(Electron utilityProcess) ──ndjson stdio──> CLI(Agent)
      ▲                                                                                        │
      └────────────── Web/手机: WebSocket relay（packages/server）<──────────────────────────────┘
```

- **Host = 每窗口一个 Electron utilityProcess**（`packages/desktop/src/host/index.ts` 头注释）。同一窗口的 Renderer 和手机都 attachment 到它：Renderer/Mobile ←MessagePort→ Window Host ├─ local services └─ remote connection registry。main 侧创建在 `packages/desktop/src/main/desktopHostProcess.ts`（`utilityProcess.fork()`）。
- Main 进程只管窗口、原生操作、进程调度、消息转发，**不承载 task/session 业务状态**。
- 服务装配：`packages/services/src/descriptors.ts`（channel 名）+ `collection.ts`（`ServiceCollection.register` + `exposeOnChannelServer`）+ `packages/rpc/src/proxy-channel.ts`（`ProxyChannel.fromService` 服务→channel；`toService` channel→类型安全代理）。

## 二、协议层（`packages/shared/src/zcode-protocol-v4/`，全 zod，两侧共用）

| 文件 | 内容 |
|---|---|
| `command.ts` | 命令全集 `commandPayloadSchemas`：`createSession`（可携 firstInput 一步建会话+首发）、`sendText`（`requestedDelivery: startNow|queue|guide`）、`stop`（带 expectedForegroundExecutionId 防误杀）、`compact`、`editUserQuery`、`retryTurn`、`resolveInteraction`（权限/问答回执）、队列系列、`forkAssistant`、`applyFileRewind` 等。信封 `CommandEnvelope{commandId(uuidv7), clientId, sessionId, baseRevision(CAS), baseLogEpoch, type, payload, issuedAt}`；回执 `CommandAck{status: accepted|duplicate|rejected|failed, reasonCode, revisionAtDecision}` |
| `rows.ts` | 行类型：`turnHeader / userInput / assistantText / reasoning / toolCall / artifact / subagent / hookInvocation / timelineMarker` |
| `delta.ts` | **封闭 7 op**：`row.appended / row.upserted / row.removed(fromRowId 截断) / row.delta(流式追加) / state.updated / workflowRun.updated / workflowRun.removed`。没有中间插入、没有字段级 JSON patch——表达不了就发 snapshot resync，刻意压缩客户端错误面 |
| `snapshot.ts` | `ConversationSnapshot{revision, logEpoch, seq, rows.window(尾窗), queue, pendingInteractions, pendingCommands, backgroundWorks, subagents, usage, inputRouting, ...}` |
| `core.ts` | `DELIVERY_PROFILES`：`continuous`（桌面：30ms flush、desktopOnlyRows、text/inputText/output.text/summaryText 流式路径、256KB cap）vs `replayable`（远程：150ms、仅 text 流式、toolProgress=true） |
| `wire.ts` / `wire-assembler.ts` / `wire-codec.ts` / `wire-fault.ts` | 物理帧 `complete|fragment`（1MB 上限）→ 逻辑帧组装（16MB/1024 fragment 上限、30s 组装超时）；assembly fault → 客户端 single-flight resync |
| `input-intent.ts` | `ConversationInputIntent`：`delivery{requested,admitted,fallbackReasonCode}` + `order{admissionSeq,queuePosition}` + `steer` + `dispatch{admitted|queued|...}` |
| `handshake.ts` | 握手能力位（如 `independentPlanState` 不支持时客户端直接拒发对应命令） |

运行时校验：renderer 逐帧 zod parse（`SRC/v4/topicWireDecoder.ts`），CLI `parseCommandEnvelope`，host 各 result schema。

## 三、传输层

- `packages/rpc/src/channelServer.ts` / `channelClient.ts`：`call/listen` 复用同一 handlers 表；`foundation.ts`：`Emitter/Event/CancellationToken`；`ipc.ts`：Electron IPC。
- `packages/client/src/index.ts`：`connectViaMessagePort`（desktop renderer→host）、`connectViaWebSocket`（web→relay）、`RemoteServiceAccess`（`remoteServiceAccess.ts`，channel→整个 `IServiceAccessor` 代理）。
- renderer 侧传输实现：`SRC/v4/agentConversationTransport.ts`——`ensureHandshake`（hello + 能力位 + TTFT 时钟校准）、`ackActivationBarrier`（**ACK 前缓冲帧、activate 后按序放行**）、帧解码校验、runtime 换代原子失效。

## 四、Host 层（`packages/services/src/zcode-agent/`）

| 文件 | 职责 |
|---|---|
| `zcodeAgentService.ts` | 对话服务实现（`createZCodeAgentService`）。`subscribeConversationV4`（约 4939 行）懒启动/复用 CLI 并发起订阅；`sendConversationCommandV4` → `v4/command`；subscription ownership 管理（同 `(workspace, topic, connectionId)` 重订阅=替换）；帧 fan-out：`conversationFrameEmitters`（约 1134 行）收 CLI stdio 通知，经 RPC Event `onDynamicConversationFrame`（约 5473 行）推给 renderer |
| `zcodeAgentProcessManager.ts` | per workspace spawn CLI 子进程，泳道（chat/plugin/mcp-status）、idle 回收、`onRuntimeRestarted` |
| 同目录 ZCodeProtocolClient / ZCodeStdioTransport | 与 CLI 的 ndjson-stdio JSON 协议客户端（3 分钟超时） |
| `zcodeAgentConnectionScope.ts` | **每个 RPC attachment 独立持有 subscription ownership**；`ZCodeAgentV4ConnectionContext{connectionId, clientMode, ...}` 经 `__zcodeTrustedV4Connection` 私有字段由 host 注入（renderer 不能伪造） |

CLI 侧协议服务在 `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/`（server.ts）。

## 五、CLI 权威层（`apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/`）

| 文件 | 职责 |
|---|---|
| `command-inbox.ts` | **CommandInbox**：zod 校验 → 幂等（inFlight pinned / 持久化事实按优先级 / settled 512/session LRU）→ **固定锁序：key gate（同 commandId）→ per-session admission gate（AsyncGateRegistry FIFO，持有直到 settle）**——这就是 busy 时输入进队列而非并发执行的机制 → CAS 校验（先 logEpoch 后 revision）→ 分配 admissionSeq/queueItemId |
| `conversation-topic-publisher.ts` | ConversationTopicPublisher：内存有界 delta 日志（logEpoch + 保留窗 2000 条）+ `subscribe(base)` 裁决（resume/snapshot）+ 每订阅者 flush 管线（filter(profile) → coalesce → 打帧）；慢订阅超限 → `resyncRequired` |
| `v4-gateway.ts` | CLI 侧网关：per-session Publisher 注册表 + `flushWindowMs` 定时调度（唯一时间源）+ `v4/command → CommandInbox → host.executeCommand` 收口；不做网络 IO（`host.emitWireFrame` 交宿主） |
| `event-normalizer.ts` | agent core `SessionEvent` → canonical facts（userIntent/assistantText/visibility/origin/lane） |
| `product-projection.ts` + `projection-rows.ts` / `projection-state.ts` | 事实 → `ConversationRow` / StatePatch |
| `queue-auto-drain.ts` | `shouldAutoDrainV4QueueHead`（autoDrain && queued && !busy && goal 完成） |

## 六、端到端时序（完整版）

```
[UI]        1 ConversationComposer 提交 → SessionPane.dispatchCommand
               （commandFactory 信封 → pendingCommandRegistry.record → markCommandPending）
[传输]      2 agentConversationTransport.sendCommand（握手/barrier）→ IZCodeAgentService RPC 代理
            3 packages/rpc ProxyChannel → MessagePort → host ChannelServer
[host]      4 zcodeAgentService.sendConversationCommandV4 → ZCodeProtocolClient → ndjson stdio
[CLI]       5 v4-gateway → CommandInbox.handle：
               校验→幂等→key gate→per-session FIFO admission→CAS→admissionSeq
               → ConversationInputIntent → executor 开始 turn；ACK 原路返回
[UI]        6 applyAck；accepted → expectAcceptedInputProjection（2s watchdog）
[事件回流]  7 agent core 流式输出 → SessionEvent → event-normalizer → product-projection
               → TopicPublisher 记账（logEpoch/seq + 有界 delta 日志）
               → flush 窗口（continuous 30ms）→ filter(profile) → coalesce → 打帧(>1MB 分片)
               → host.emitWireFrame → stdio 通知
[host/UI]   8 host: conversationFrameEmitters → RPC Event onDynamicConversationFrame → MessagePort
            9 renderer: barrier.accept → 组帧 → zod 校验 → SessionDataLayer 扇入
           10 ConversationProjectionStore.applyFrame（snapshot 替换 / delta 衔接 / 断档自愈）
               → reconcileOptimistic（pending 气泡转正）→ useSyncExternalStore → Timeline
[旁路]      任务列表：CLI SessionsIndexPublisher → sessionsIndexStore → useWorkspaceSessionsIndexItems
               → buildTaskListResultFromSessions → taskQueryCacheStore → 侧栏
```

## 七、两种投递档位（必须区分）

- **desktop-continuous**：本机实时流，`DELIVERY_PROFILES.continuous`（30ms、desktopOnlyRows——某些行只给桌面）。renderer→host 用 MessagePort 长连接；跨 host 事件广播见 `packages/desktop/src/host/taskRealtimeBridge.ts`。
- **web-remote-replayable**：Web/手机，`replayable` 档（150ms、仅 text 流式）。**断线重连不依赖本地状态**：携 `base{logEpoch, seq}` 重订阅，publisher 裁决 resume（保留窗内 delta 续传）或 snapshot 全量；更早历史走 `rowsRange`（`loadOlder/loadAllOlder`，游标=窗口首行 rowId，`atLogEpoch` 陈旧读防护）。改 stream/snapshot/queue/重连逻辑时两种语义都要验证（AGENTS.md 明文要求）。

## 八、建议精读顺序

1. `packages/shared/src/zcode-protocol-v4/delta.ts` + `snapshot.ts` —— 数据形状，一切的基础
2. `SRC/v4/transport.ts` —— 客户端接缝（换后端只动这里）
3. `apps/zcode-cli/.../conversation-topic-publisher.ts` —— 权威侧的订阅裁决，最精彩的一段
4. `apps/zcode-cli/.../command-inbox.ts` —— 幂等 + 串行 admission 的工程化实现
5. `SRC/v4/agentConversationTransport.ts` —— barrier/组帧/校验
6. `packages/services/src/zcode-agent/zcodeAgentService.ts` —— 只看 subscribeConversationV4 与 fan-out，别通读（5000+ 行）
