# 13 · 移植清单：从零到全套，分阶段落地

这一章是把前 12 章串成一条可执行路径。每个阶段都能独立跑起来、独立验收、独立上线。

---

## 1. 阶段总览

| 阶段 | 目标 | 依赖章节 | 大致工作量 | 上线后用户能感知的 |
| --- | --- | --- | --- | --- |
| **P0** 统一模型 | 前后端共用一套 Message/Part | 04 | 1–2 天 | 无（地基） |
| **P1** 事件与传输 | 事件表 + SSE + 断线重连 | 07 | 2–3 天 | 刷新不丢过程 |
| **P2** 流式渲染 | store reducer + 合帧 + 打字 + markdown | 08、10、11 | 3–5 天 | 逐字出现、不卡 |
| **P3** 输入编排 | 收件箱 + drain + 排队/打断 | 05 | 2–3 天 | 跑着的时候能插话 |
| **P4** 工具展示 | 四态 + 卡片 + 折叠 + 输出治理 | 06、10 | 2–3 天 | 看得懂 agent 在干什么 |
| **P5** 时间线 | 行投影 + 虚拟化 + 贴底跟随 | 09 | 3–5 天 | 千条消息不卡、不跳 |
| **P6** 上下文管理 | SystemContext + Epoch | 01、02 | 3–4 天 | 改配置后模型立刻知道 |
| **P7** 压缩与可视化 | 压缩 + 用量条 | 03、12 | 2–3 天 | 长会话不再报超窗 |

**P0–P2 是最小闭环**（能上线的流式对话）。**P3 是体验分水岭**。**P6–P7 是长会话产品的分水岭**。

---

## 2. P0 · 统一模型

### 做什么

在共享包里定义一份 `Message` / `Part` 判别联合，前后端同时 import。

### 关键决策

| 决策 | 推荐 | 理由 |
| --- | --- | --- |
| 消息 id | ULID（单调递增） | 字典序 = 时间序，前端二分插入的前提（第 08 章 §4.4） |
| assistant 内容 | 有序 `content[]` 数组 | 文本/思考/工具必须交错有序（第 04 章 M2） |
| 工具状态 | 四态判别联合 | `pending.input` 是 string，其余是对象——这个差异不能抹平 |
| 事件粒度 | part 级 | 消息级会让每个 delta 推整条消息 |

### 骨架

```ts
// shared/types.ts —— 前后端唯一定义处
export type MessageID = string          // "msg_" + ULID

export type ToolState =
  | { status: "pending";   input: string }                       // 还在拼 JSON
  | { status: "running";   input: Record<string, unknown>; structured: Record<string, unknown>; content: ToolContent[] }
  | { status: "completed"; input: Record<string, unknown>; structured: Record<string, unknown>
      content: ToolContent[]; outputPaths?: string[]; result?: unknown }
  | { status: "error";     input: Record<string, unknown>; structured: Record<string, unknown>
      content: ToolContent[]; error: { type: "unknown"; message: string }; result?: unknown }

export type AssistantContent =
  | { type: "text"; id: string; text: string }
  | { type: "reasoning"; id: string; text: string; providerMetadata?: Record<string, unknown> }
  | { type: "tool"; id: string; name: string; state: ToolState
      provider?: { executed: boolean; metadata?: Record<string, unknown>; resultMetadata?: Record<string, unknown> }
      time: { created: number; ran?: number; completed?: number } }

export type Message =
  | { id: MessageID; type: "user"; text: string; files?: FileAttachment[]; time: { created: number } }
  | { id: MessageID; type: "assistant"; content: AssistantContent[]; model: ModelRef; agent: string
      tokens?: { input: number; output: number; reasoning: number; cache: { read: number; write: number } }
      error?: { type: "unknown"; message: string }
      time: { created: number; completed?: number } }
  | { id: MessageID; type: "system"; text: string; time: { created: number } }
  | { id: MessageID; type: "compaction"; reason: "auto" | "manual"; summary: string; recent: string
      time: { created: number } }
```

### 验收

- [ ] 构造含全部消息类型 × 全部工具态的会话 → 序列化再反序列化后深相等
- [ ] 前后端 import 的是同一个文件（不是两份复制）
- [ ] 加一个新 `type` 时，所有 `switch` 编译报错（穷尽性检查）

---

## 3. P1 · 事件与传输

### 做什么

durable / live-only 分野 + 事件表 + SSE 端点 + 客户端重连。

### 关键决策

**哪些事件 durable，哪些 live-only** —— 抄这张表：

| 事件 | durable? |
| --- | --- |
| `Prompted` / `PromptAdmitted` | ✅ |
| `Step.Started` / `Step.Ended` / `Step.Failed` | ✅ |
| `Text.Started` / `Text.Ended` | ✅ |
| `Reasoning.Started` / `Reasoning.Ended` | ✅ |
| `Tool.Input.Started` / `Tool.Input.Ended` / `Tool.Called` / `Tool.Success` / `Tool.Failed` | ✅ |
| `ContextUpdated` / `Compaction.Started` / `Compaction.Ended` | ✅ |
| **`*.Delta`（Text / Reasoning / Tool.Input / Compaction）** | ❌ **live-only** |
| `session.status` | ❌ live-only |

### 骨架

```ts
// server/event-bus.ts
type EventDef<D> = { type: string; durable?: { aggregate: keyof D & string } }

export class EventBus {
  private listeners = new Set<(e: AnyEvent) => void>()
  private projectors = new Map<string, ((e: AnyEvent) => Promise<void>)[]>()

  project(type: string, fn: (e: AnyEvent) => Promise<void>) {
    const list = this.projectors.get(type) ?? []
    list.push(fn); this.projectors.set(type, list)
  }

  async publish<D>(def: EventDef<D>, data: D, opts?: { commit?: (seq: number) => Promise<void> }) {
    if (!def.durable && opts?.commit) throw new Error("commit hooks require a durable event")
    const event = { id: ulid(), type: def.type, data }
    if (!def.durable) { this.broadcast(event); return event }         // live-only：直接广播

    const aggregateId = (data as any)[def.durable.aggregate] as string
    const committed = await db.transaction(async (tx) => {
      const row = await tx.get<{ seq: number }>(
        "SELECT seq FROM event_sequence WHERE aggregate_id = ?", aggregateId)
      const seq = (row?.seq ?? -1) + 1
      const dup = await tx.get("SELECT 1 FROM event WHERE id = ?", event.id)
      if (dup) throw new Error(`Event ${event.id} already exists`)
      const full = { ...event, durable: { aggregateId, seq } }
      for (const p of this.projectors.get(def.type) ?? []) await p(full)   // 投影器在事务内
      if (opts?.commit) await opts.commit(seq)                              // 本地投影也在事务内
      await tx.run(`INSERT INTO event_sequence(aggregate_id, seq) VALUES(?,?)
                    ON CONFLICT(aggregate_id) DO UPDATE SET seq = ?`, aggregateId, seq, seq)
      await tx.run("INSERT INTO event(id,aggregate_id,seq,type,data) VALUES(?,?,?,?,?)",
                   event.id, aggregateId, seq, def.type, JSON.stringify(data))
      return full
    })
    this.broadcast(committed)                                              // 事务提交后才广播
    return committed
  }

  private broadcast(e: AnyEvent) { for (const l of this.listeners) l(e) }

  /** 有界订阅：慢消费者被断开，不阻塞发布方 */
  subscribe(fn: (e: AnyEvent) => void, capacity = 256) {
    let queue: AnyEvent[] = []
    let closed = false
    const listener = (e: AnyEvent) => {
      if (closed) return
      if (queue.length >= capacity) { closed = true; this.listeners.delete(listener); onOverflow?.(); return }
      queue.push(e); drain()
    }
    let draining = false
    const drain = () => {
      if (draining) return
      draining = true
      queueMicrotask(() => { draining = false; const b = queue; queue = []; b.forEach(fn) })
    }
    let onOverflow: (() => void) | undefined
    this.listeners.add(listener)
    return { stop: () => { closed = true; this.listeners.delete(listener) },
             onOverflow: (fn: () => void) => { onOverflow = fn } }
  }
}
```

```ts
// server/sse.ts
app.get("/event", (req, res) => {
  res.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache, no-transform",
    "X-Accel-Buffering": "no",              // ← 不加，nginx 后面会整段延迟
    "X-Content-Type-Options": "nosniff",
    Connection: "keep-alive",
  })
  res.write(`data: ${JSON.stringify({ type: "server.connected" })}\n\n`)    // 首帧
  const sub = bus.subscribe((e) => res.write(`data: ${JSON.stringify(e)}\n\n`))
  sub.onOverflow(() => res.end())                                           // 慢消费者断开
  const hb = setInterval(() => res.write(": heartbeat\n\n"), 15_000)
  req.on("close", () => { sub.stop(); clearInterval(hb) })
})
```

### 验收

见第 07 章 §11 全表。最关键的三条：
- [ ] 投影器抛错 → 事务回滚，事件不存在
- [ ] 订阅者收到事件时，事件已在库里
- [ ] `*.Delta` 事件不写库、不推进 seq

---

## 4. P2 · 流式渲染

### 做什么

事件泵（合帧 + delta 合并）→ store reducer → 打字节奏 → markdown 块投影。

### 关键数值（照抄）

```ts
const FLUSH_FRAME_MS = 16
const STREAM_YIELD_MS = 8
const RECONNECT_DELAY_MS = 250
const TEXT_RENDER_PACE_MS = 24
const TEXT_RENDER_IMMEDIATE = 512
```

### 骨架

事件泵见第 08 章 §10.1，SSE 消费循环见第 08 章 §10.2，打字 hook 见第 10 章 §9.1，markdown 块投影见第 11 章 §7.1。全部可直接复制。

store（Zustand + Immer）：

```ts
export const useSync = create<SyncState>()(immer((set) => ({
  message: {}, part: {}, partTextAccumDelta: {}, sessionStatus: {},
  applyBatch: (events) => set((d) => { for (const e of events) applyEvent(d, e) }),
})))

function applyEvent(d: Draft<SyncState>, e: ServerEvent) {
  switch (e.type) {
    case "message.updated": {
      const info = e.properties.info
      const list = (d.message[info.sessionID] ??= [])
      const r = binarySearch(list, info.id, (m) => m.id)
      if (r.found) list[r.index] = info
      else list.splice(r.index, 0, info)
      break
    }
    case "message.part.updated": {
      const part = e.properties.part
      if (SKIP_PARTS.has(part.type)) break
      delete d.partTextAccumDelta[part.id]              // 权威值覆盖累积值
      const list = (d.part[part.messageID] ??= [])
      const r = binarySearch(list, part.id, (p) => p.id)
      if (r.found) list[r.index] = part
      else list.splice(r.index, 0, part)
      break
    }
    case "message.part.delta": {
      const { messageID, partID, field, delta } = e.properties
      const list = d.part[messageID]
      if (!list) break
      const r = binarySearch(list, partID, (p) => p.id)
      if (!r.found) break                                // part 还没到 → 丢弃
      const cur = (list[r.index] as any)[field]
      d.partTextAccumDelta[partID] = (d.partTextAccumDelta[partID] ?? (typeof cur === "string" ? cur : "")) + delta
      ;(list[r.index] as any)[field] = ((list[r.index] as any)[field] ?? "") + delta
      break
    }
    case "session.status":
      d.sessionStatus[e.properties.sessionID] = e.properties.status
      break
    case "server.connected":
      queueMicrotask(() => refetchAll())                 // 重连后全量补齐
      break
  }
}
```

### React 性能红线

opencode 靠 SolidJS 的细粒度响应式白拿了「只有变的 part 重渲染」。**React 里必须自己做，否则前面所有优化都白费**：

```tsx
// ✅ 订阅最小切片
const parts = useSync((s) => s.part[messageId])
const text  = useSync((s) => s.partTextAccumDelta[partId] ?? partText)

// ✅ 列表项 memo，按 part 引用比较
const PartView = React.memo(
  ({ part }: { part: Part }) => { /* ... */ },
  (a, b) => a.part === b.part,
)

// ❌ 千万别这样：store 一变整棵树重渲染
const store = useSync()
```

### 验收

见第 08 章 §11 和第 10 章 §10。最关键的：
- [ ] 一帧内 20 个 delta → reducer 只调 1 次
- [ ] 一个 part 流式更新时，其它 part 组件不重渲染（Profiler 验证）
- [ ] 主线程无 > 50ms 长任务

---

## 5. P3 · 输入编排

### 做什么

`session_input` 收件箱表 + `promoted_seq` CAS + drain 双层循环 + 会话级串行。

### 最容易抄错的三处

1. **`POST /prompt` 必须立刻返回**，不等模型。前端自己生成 id 做乐观插入。
2. **steer 提升要带 cutoff**（本轮开始时的最新 seq），否则高频输入让本轮永不收敛。
3. **drain 内层循环末尾要再查一次 steer**：模型说完了但用户刚插话，要继续跑而不是 idle。

### 骨架

```ts
async function drain(sessionId: string, force: boolean) {
  const hasSteer = await hasPending(sessionId, "steer")
  const hasQueue = hasSteer ? false : await hasPending(sessionId, "queue")
  if (!force && !hasSteer && !hasQueue) return
  await failInterruptedTools(sessionId)                    // 崩溃恢复

  let promotion: Delivery | undefined = hasSteer ? "steer" : hasQueue ? "queue" : undefined
  let shouldRun = force || hasSteer || hasQueue

  while (shouldRun) {                                      // 外层：queue
    let needsContinuation = true
    let step = 1
    while (needsContinuation) {                            // 内层：step
      const result = await runTurn(sessionId, promotion, step)
      needsContinuation = result.needsContinuation
      step = result.step + 1
      promotion = "steer"                                  // 第二轮起固定 steer
      if (!needsContinuation) needsContinuation = await hasPending(sessionId, "steer")
    }
    shouldRun = await hasPending(sessionId, "queue")
    promotion = shouldRun ? "queue" : undefined
  }
}

// runTurn 内部
const cutoff = await latestSequence(sessionId)
let promoted = 0
if (promotion === "steer") promoted = await promoteSteers(sessionId, cutoff)
if (promotion === "queue") {
  promoted += Number(await promoteNextQueued(sessionId))
  promoted += await promoteSteers(sessionId, cutoff)
}
if (promoted > 0) step = 1                                 // 新输入重置步数配额
```

会话级串行（最简版）：

```ts
const active = new Map<string, { promise: Promise<void>; pendingWake: boolean }>()

export function wake(sessionId: string) {
  const entry = active.get(sessionId)
  if (entry) { entry.pendingWake = true; return entry.promise }
  return start(sessionId, false)
}
export function run(sessionId: string) {
  const entry = active.get(sessionId)
  if (entry) return entry.promise                          // 等当前，不排新的
  return start(sessionId, true)
}
function start(sessionId: string, force: boolean): Promise<void> {
  const entry = { pendingWake: false } as any
  entry.promise = (async () => {
    try { await drain(sessionId, force) }
    finally {
      const again = entry.pendingWake
      active.delete(sessionId)
      if (again) await start(sessionId, false)             // 合并的后继
    }
  })()
  active.set(sessionId, entry)
  return entry.promise
}
```

### 验收

见第 05 章 §8。最关键的：
- [ ] 模型跑工具期间发 steer → 下一轮就被提升
- [ ] queue 里有 3 条 → 分 3 个外层循环，每次一条
- [ ] 并发 `run(sessionId)` → 只有一个 drain

---

## 6. P4 · 工具展示

### 做什么

工具输出治理（落盘）+ 卡片骨架 + 图标标题映射 + 默认展开策略 + 上下文组折叠。

### 骨架

工具输出治理见第 06 章 §7.1，`getToolInfo` 见第 10 章 §9.2，`partDefaultOpen` 见第 10 章 §4.6。

上下文组折叠（第 09 章 §4.3 的 `groupParts`）是**投入产出比最高的一个功能**——一次任务里 read/grep/glob/list 能占 70% 的工具调用，折叠后时间线的噪音立刻降下来。

### 验收

- [ ] 2001 行的工具输出 → 落盘 + 头尾采样 + 中间 marker
- [ ] UTF-8 截断不产生乱码
- [ ] 工具 running 时不可展开
- [ ] 连续 8 个 read → 折叠成一行 `Explored  8 reads`
- [ ] 未注册的工具 → 兜底渲染，不崩

---

## 7. P5 · 时间线

### 做什么

9 种行 + turn 分组 + 增量行复用 + 虚拟化 + 贴底跟随 + prepend 锚定。

### 关键数值（照抄）

```ts
overscan: 50, renderOverscan: 6 → 20, scrollEndThreshold: 80, paddingEnd: 64
bottomThreshold: 10, autoScrollWindow: 1500ms / 2px, settling: 300ms
jumpThreshold: max(400, clientHeight), bottomDistance: <= 2
```

### 最容易抄错的三处

1. **`getItemKey` 必须用行 key，不能用 index**。
2. **程序化滚动识别**（`markAuto` / `isAuto`）——不做这个，流式一开始跟随就失效。
3. **在 `ResizeObserver` 回调里同帧贴底**，不要包 `requestAnimationFrame`。

### 骨架

见第 09 章 §9.1（虚拟化）和 §9.2（`useAutoScroll` 完整 hook）。

### 验收

见第 09 章 §10。最关键的：
- [ ] 1000 条消息流式追加 → 稳定 60fps
- [ ] 用户上滑 → 停止跟随；回到底部 10px 内 → 恢复
- [ ] 向上加载历史 → 视口内容位置不变

---

## 8. P6 · 上下文管理

### 做什么

`SystemContext` 抽象 + 注册表 + `ContextEpoch` 表 + 会话中系统消息 + 双 cutoff 历史投影。

### 做这一步之前要确认

- 你的 system prompt 里**确实有会变的部分**（工作目录、项目规则、时间、可用工具集）。如果全是静态的，跳过 P6。
- 你**在意 provider prompt cache**。如果不在意（比如自建模型无缓存），可以简化成「每轮重拼 system prompt」。

### 最小实现路径

1. 先只做**一个** Context Source（比如「当前工作目录」），跑通全链路。
2. 再加第二个（`AGENTS.md` 聚合），验证多来源组合与排序确定性。
3. 最后加 `Unavailable` 和 `removed` 语义。

### 骨架

`Source<A>` 与 `reconcile` 见第 01 章 §3.2 / §4.5，纪元表与 `prepare` 见第 02 章 §3.2 / §4.3。

**一个务必记住的顺序**：

```
initializeEpoch()        ← 必须在提升输入之前（基线不可用时输入可重试）
  ↓
promoteInputs()
  ↓
prepareEpoch()           ← 必须在提升输入之后（user 消息 seq 早于 system 消息）
  ↓
loadHistory(baselineSeq)
  ↓
buildRequest(baseline, history)
```

### 验收

见第 01 章 §9 和第 02 章 §9。最关键的：
- [ ] 同一纪元内连续 3 轮，system 段**逐字节相同**
- [ ] 改 `AGENTS.md` → 下一轮出现一条 system 消息，baseline 不变
- [ ] 系统消息与快照推进同事务

---

## 9. P7 · 压缩与可视化

### 做什么

两级压缩 + 摘要 prompt + 用量条。

### 最容易抄错的三处

1. **`estimate` 要包含工具定义**——coding agent 里工具 schema 常占几千 token。
2. **`select` 从尾往前累加**，保证最近的对话完整保留。
3. **压缩重跑时 `promotion = undefined`**，不重复提升输入。

### 骨架

判据与切分见第 03 章 §4.1 / §4.2，prompt 原文见第 03 章 §5（直接抄），用量条见第 12 章 §7.1。

### 验收

见第 03 章 §9 和第 12 章 §8。最关键的：
- [ ] 摘要失败 → 无任何痕迹，下轮重试
- [ ] 压缩后 `baseline_seq === compaction 消息 seq`
- [ ] 条形图各段之和 === provider 报告的 `input`

---

## 10. 全局验收：一次真实会话的端到端检查

做完全部阶段后，跑这一遍：

1. **开一个新会话**，发一条需要用工具的提问。
   - [ ] `POST` 立刻返回，UI 立刻出现用户气泡（乐观插入）
   - [ ] 几百毫秒内出现 `Thinking` + 点阵
   - [ ] 文本逐字出现，代码块实时高亮
   - [ ] 工具卡片从 pending 走到 completed，标题从进行时变过去时

2. **模型跑工具时插一句话**。
   - [ ] 输入框可用，发送后立刻显示
   - [ ] 下一轮模型的回答体现了这句话

3. **刷新页面**。
   - [ ] 全部历史完整恢复，包括工具卡片的展开状态之外的一切
   - [ ] 正在进行的生成继续流式（如果还在跑）

4. **上滑到中间，再让模型说话**。
   - [ ] 不自动跳到底部
   - [ ] 出现「跳到最新」按钮
   - [ ] 点击后回到底部并恢复跟随

5. **在工具输出块内滚动**。
   - [ ] 外层继续贴底跟随

6. **断网 10 秒再恢复**。
   - [ ] 自动重连
   - [ ] 断网期间产生的消息全部补齐，无重复无缺失

7. **改 `AGENTS.md`，再发一条消息**。
   - [ ] 历史里出现一条系统消息说明新规则
   - [ ] 模型的回答体现了新规则
   - [ ] 抓包确认 system 段没变（缓存仍命中）

8. **持续对话直到接近上下文上限**。
   - [ ] 用量条的 `tool` 段随工具调用增长
   - [ ] 到达阈值时自动压缩，出现压缩分隔线
   - [ ] 压缩后模型仍记得最初的目标（摘要生效）
   - [ ] 压缩后 system 段被重新渲染（新纪元）

9. **打开 1000 条消息的老会话**。
   - [ ] 500ms 内可交互
   - [ ] 滚动流畅，无白屏
   - [ ] 向上加载更多历史时视口不跳

10. **开启系统「减少动态效果」**。
    - [ ] 点阵停止动画但仍可见
    - [ ] 无其它动画残留

---

## 11. 常见踩坑清单（按被踩频率排序）

| # | 坑 | 症状 | 出处 |
| --- | --- | --- | --- |
| 1 | React store 一变整棵树重渲染 | 做完所有优化还是卡 | 第 08 章 §3.3 |
| 2 | 用 index 做虚拟列表 key | 头部插入后全表错位 | 第 09 章 §5 |
| 3 | 程序化滚动被误判成用户滚动 | 流式一开始跟随就失效 | 第 09 章 §6.1 |
| 4 | delta 不合帧 | 每秒几十次渲染 | 第 08 章 §5 |
| 5 | 读流不让出主线程 | 「卡住然后一次性全出来」 | 第 08 章 §5.4 |
| 6 | 事务外跑投影器 | 「有事件无投影」的裂开状态 | 第 07 章 §6 |
| 7 | 先拉全量后订阅 | 窗口期事件丢失 | 第 07 章 §7 |
| 8 | 不发 SSE 心跳 / 不设 `X-Accel-Buffering` | 代理切连接 / 整段延迟 | 第 07 章 §7 |
| 9 | 提升输入不做 CAS | 并发下同一条消息进历史两次 | 第 05 章 §4.2 |
| 10 | 崩溃后不清理 pending 工具 | 下轮 provider 报 400 | 第 05 章 §6 |
| 11 | 跨模型不降级 reasoning | provider 直接 400 | 第 04 章 §5.1 |
| 12 | 工具输出按 `slice` 截断 | 中文/emoji 乱码 | 第 06 章 §4.2 |
| 13 | 压缩估算漏算工具定义 | 预检失效，直接溢出 | 第 03 章 §4.1 |
| 14 | 压缩摘要用 `system` role | 模型把「Next Move」当新指令执行 | 第 03 章 §4.8 |
| 15 | 每轮重拼 system prompt | prompt cache 全失效，成本翻几倍 | 第 02 章 §1 |
| 16 | 历史投影漏 baseline_seq cutoff | 模型收到互相矛盾的环境描述 | 第 04 章 §4 |
| 17 | markdown 每次全量 parse | 长回答后期每帧几十毫秒 | 第 11 章 §4.1 |
| 18 | 流式中不修补未闭合语法 | 看到 `**` 字面量，闭合时整块重排 | 第 11 章 §4.4 |
| 19 | 会话裁剪漏清缓存 | 内存持续增长 | 第 08 章 §7.2 |
| 20 | 裁掉了在等权限的子会话 | agent 永久卡死 | 第 08 章 §7.2 |

---

## 12. 如果只能做三件事

假设你时间极其有限，只能从这套方案里挑三个改动，选这三个：

1. **事件泵：16ms 合帧 + 相邻 delta 合并**（第 08 章 §5）。
   ~80 行代码，直接把流式渲染从「掉帧」变成「丝滑」。收益最高、风险最低。

2. **markdown 块投影 + 语法修补**（第 11 章 §7.1）。
   ~60 行代码，长回答后期从每帧几十毫秒降到几毫秒，同时消除 `**` 字面量闪烁。

3. **贴底跟随状态机**（第 09 章 §9.2）。
   ~80 行代码，解决「流式时页面乱跳」「上滑后被强制拉回底部」这两个最招骂的问题。

这三个都是**纯前端、无后端依赖、可独立上线**的。做完再考虑 P3 和 P6/P7。
