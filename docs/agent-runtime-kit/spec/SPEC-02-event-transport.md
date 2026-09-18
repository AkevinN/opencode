# SPEC-02 · 事件模型与流式传输

> **优先级** P0 · **工作量** M · **依赖** SPEC-01
> **分析原文** [ch07 事件模型与 SSE](../../context-and-streaming/07-events-and-sse.md)
> **验收** [`acceptance/SPEC-02.yaml`](../acceptance/SPEC-02.yaml)（23 条）

---

## 1. 目标与非目标

### 目标

把「模型正在干什么」变成一条**既能实时看、又能回放**的事件流，同时不让数据库每秒写几十次。

核心设计判断：

> **「模型看到什么」和「用户看到什么」是同一份事件流的两次不同投影。**
> 后端不为前端单独造数据，前端也不需要理解模型协议。

### 非目标

- 不规定传输协议（SSE / WebSocket / 长轮询都可以，SSE 是推荐实现）。
- 不规定存储引擎（任何支持事务的都行）。
- 不做跨进程/集群的事件分发（单进程即可满足全部条款）。

---

## 2. 领域模型

```ts
/** 事件定义。durable 字段的有无决定它落不落库——这是本规范最核心的一个区分 */
export interface EventDefinition<D> {
  type: string
  /** 有 = 持久化 + 广播；无 = 仅广播（live-only） */
  durable?: {
    /** data 里哪个字段是聚合 id（通常是 sessionId） */
    aggregate: keyof D & string
    version: number
  }
}

export interface EventPayload<D = unknown> {
  id: string
  type: string
  data: D
  /** 仅 durable 事件有 */
  durable?: { aggregateId: string; seq: number; version: number }
}

export interface PublishOptions {
  id?: string
  /** 与事件同事务提交的本地投影。只允许配 durable 事件。见 R-02-05 */
  commit?: (seq: number) => Promise<void>
}
```

存储：

```sql
CREATE TABLE event_sequence (
  aggregate_id TEXT PRIMARY KEY,
  seq          INTEGER NOT NULL
);
CREATE TABLE event (
  id           TEXT PRIMARY KEY,
  aggregate_id TEXT    NOT NULL REFERENCES event_sequence(aggregate_id) ON DELETE CASCADE,
  seq          INTEGER NOT NULL,
  type         TEXT    NOT NULL,        -- 建议带版本后缀
  data         TEXT    NOT NULL         -- JSON
);
CREATE UNIQUE INDEX event_aggregate_seq_idx ON event(aggregate_id, seq);
```

---

## 3. 规范条款

### R-02-01 片段增量事件必须是 live-only · **必须**

**要求**：把事件分成两类：

| 类别 | 例子 | 落库 | 广播 |
| --- | --- | --- | --- |
| **durable** | `*.Started`、`*.Ended`、`Tool.Called`、`Tool.Success`、`Step.Ended`、`Compaction.*` | ✅ | ✅ |
| **live-only** | `Text.Delta`、`Reasoning.Delta`、`ToolInput.Delta`、`Compaction.Delta`、`session.status` | ❌ | ✅ |

每个 live-only 的 `*.Delta` **必须**有一个对应的 `*.Ended` 承载完整值。

**理由**：一段 500 字的回答会产生几百个 delta。全落库 = 每秒几十次磁盘写；全不落库 = 刷新就丢。分野之后：连着的客户端收 delta 逐字渲染，刷新/后来的客户端收 `Ended` 里的完整值，数据库一段文本只写一行。

**常见错误**：为了"简单"把 delta 也落库；或者只有 delta 没有 `Ended`（刷新后内容全无）。

---

### R-02-02 durable 事件的序号必须严格连续无空洞 · **必须**

**要求**：按 `aggregateId` 分配 `seq = 0, 1, 2, …`，不得跳号。

**理由**：增量拉取（`after=N`）依赖连续性判断有没有漏事件。空洞会让客户端永远等一个不存在的序号。

---

### R-02-03 一次发布必须在单个事务内完成 · **必须**

**要求**：事务内按此顺序：

```
1. SELECT 当前 seq                    → latest（无行则 -1）
2. 编码 data
3. 检查事件 id 是否已存在 → 存在则失败
4. seq = latest + 1
5. 运行该事件类型的全部投影器
6. 运行 commit 钩子（若有）
7. UPSERT 序号表
8. INSERT 事件行
```

**理由**：投影器（第 5 步）和 commit 钩子（第 6 步）跑在写事件行**之前**，看起来反直觉，但保证了：投影失败 → 整个事务回滚 → 事件也不存在。不会出现「事件在库里但投影没跑」的裂开状态。

**常见错误**：先写事件、事务外跑投影。

---

### R-02-04 广播必须发生在事务提交之后 · **必须**

**要求**：订阅者收到 durable 事件时，该事件已经落库。

**理由**：客户端收到事件后可能立刻发起一个读请求（比如拉全量），如果事务还没提交，它会读到不一致的状态。

**验证**：订阅者收到事件的回调里，用另一个连接查库应该能查到（A-02-05）。

---

### R-02-05 本地投影钩子只允许配 durable 事件 · **必须**

**要求**：给 live-only 事件传 `commit` 回调时直接报错。

**理由**：`commit` 的语义是「与事件原子提交的本地状态推进」。live-only 事件没有事务，配了它会让本地状态与事件流不同步。SPEC-13 的快照原子推进依赖这个钩子。

---

### R-02-06 事件 id 必须全局唯一且重复插入失败 · **必须**

**理由**：重放场景（崩溃恢复、多节点同步）下，重复 id 会产生歧义——同一个事件在两个位置。

---

### R-02-07 重放必须校验序号连续性与内容一致性 · **必须**

**要求**：重放一个带 `seq` 的事件时：

| 情况 | 行为 |
| --- | --- |
| `seq == latest + 1` | 正常写入 |
| `seq <= latest` 且内容与已存的完全一致 | **幂等成功**（不重复插入） |
| `seq <= latest` 但内容不同 | 失败（"重放分叉"） |
| `seq > latest + 1` | 失败（"序号不连续"） |

**理由**：崩溃恢复时必然出现重复重放，不能报错；但内容不同的同序号意味着真正的数据分叉，必须炸出来。

---

### R-02-08 每个订阅者必须有独立的有界队列，溢出时断开该订阅者 · **必须**

**要求**：每个订阅者一个固定容量的队列。满了就**关闭这个订阅者**（让它重连），**绝不阻塞发布方**。

**理由**：一个卡住的浏览器标签页不能拖慢 agent 的执行。无界队列会 OOM；阻塞发布方会让所有人一起卡。

**常见错误**：用无界数组；或者 `await` 每个订阅者的处理。

---

### R-02-09 流式片段必须有累积器且在流结束时强制收尾 · **必须**

**要求**：为 text / reasoning / toolInput 各维护一个 `Map<id, string[]>` 累积器：

- `start(id)`：建条目；已存在则报错
- `append(id, chunk)`：追加；未 start 则报错
- `end(id)`：发 `*.Ended`（内容 = 累积串），删条目
- `flush()`：把所有未 end 的条目强制 end

**要求**：provider 流结束时（无论成功、失败还是中断）**必须**调用 `flush()`。

**理由**：provider 流中途断开时，那些只有 start 和 delta 没有 end 的片段，内容会全部丢失。

**实现提示**：用 `try/finally` 或等价机制保证 flush 一定执行。

---

### R-02-10 协议违约必须立刻失败并带上下文 · **必须**

**要求**：以下情况**不得**静默忽略，必须抛出带 id/名字的错误：

- delta 出现在 start 之前，或 end 之后
- 同一 id 被 start 两次 / end 两次
- 同一工具调用的 `name` 在 input-start / input-delta / call / result / error 之间不一致
- tool-result / tool-error 出现在 tool-call 之前
- `step-finish` 出现两次

**理由**：这些都是 provider 协议违约。静默忽略会产生一条语义损坏的消息，然后在下一轮回放时让 provider 报 400，而那时你已经无从排查。**早失败，带上下文。**

---

### R-02-11 provider 错误之后的事件必须全部丢弃 · **必须**

**要求**：收到 `provider-error` 后置一个标志，后续所有流事件直接忽略。

**理由**：错误后的事件会污染一条已被标记为失败的消息。

---

### R-02-12 SSE 响应头与握手 · **必须**（用 SSE 时）

**要求**：

```
Content-Type: text/event-stream
Cache-Control: no-cache, no-transform
X-Accel-Buffering: no
X-Content-Type-Options: nosniff
```

**要求**：连接建立后**第一帧**必须是一个约定的「已连接」事件（如 `server.connected`）。

**要求**：必须有周期性心跳（SSE 注释行 `: heartbeat\n\n`）。

**理由**：
- `X-Accel-Buffering: no` 不加，nginx 后面的 SSE 会整段延迟数秒才出现。
- `no-transform` 防止中间代理压缩/改写流。
- 「已连接」首帧是客户端「先订阅后拉取」的信号（见 R-02-13）。
- 无输出的连接会被中间代理按空闲超时切断。

---

### R-02-13 必须先订阅后拉取全量 · **必须**

**要求**：客户端的顺序是：建立订阅 → 收到「已连接」首帧 → 拉全量快照 → apply 期间到达的事件。

**理由**：反过来（先拉全量再订阅）会丢掉两个动作之间窗口期产生的事件。

**服务端配合**：监听器必须在「已连接」帧发出**之前**就装好。

---

### R-02-14 并发发布必须串行化 · **必须**

**要求**：所有 `publish` 经过一个容量 1 的串行通道。

**理由**：工具是并发执行的，它们的结果事件可能同时到达。没有串行，两个事件会抢同一个 `seq`。

---

### R-02-15 token 统计口径必须区分缓存 · **应该**

**要求**：`tokens` 拆成 `{ input, output, reasoning, cache: { read, write } }`，其中 `input` **不含**缓存命中部分。所有值经非负有限数兜底（provider 可能回 `null` / `NaN`）。

**理由**：UI 才能显示「本轮实际付费的 input 是多少、缓存省了多少」。SPEC-17 依赖这个拆分。

---

## 4. 算法规范

### 4.1 发布

```
publish(definition, data, options):
  if not definition.durable and options.commit:
      throw "commit hooks require a durable event"             # R-02-05

  event = { id: options.id ?? newId(), type: definition.type, data }

  if not definition.durable:
      broadcast(event)                                          # live-only 直接广播
      return event

  aggregateId = data[definition.durable.aggregate]
  committed = transaction:                                      # R-02-03
      row     = SELECT seq FROM event_sequence WHERE aggregate_id = aggregateId
      latest  = row?.seq ?? -1
      encoded = encode(definition.schema, data)
      if EXISTS(SELECT 1 FROM event WHERE id = event.id): fail  # R-02-06
      seq = latest + 1
      full = { ...event, durable: { aggregateId, seq } }
      for projector in projectors[definition.type]: projector(full)
      if options.commit: options.commit(seq)
      UPSERT event_sequence SET seq
      INSERT INTO event(...)
      return full

  broadcast(committed)                                          # R-02-04 事务后
  return committed
```

### 4.2 有界订阅

```
subscribe(handler, capacity = 256):
  queue  = []
  closed = false

  listener(event):
    if closed: return
    if queue.length >= capacity:                                # R-02-08
        closed = true
        removeListener(listener)
        onOverflow()            # 调用方据此断开连接
        return
    queue.push(event)
    scheduleDrain()

  scheduleDrain():  # 微任务里批量取出交给 handler
    ...

  addListener(listener)
  return { stop, onOverflow }
```

### 4.3 provider 流 → 领域事件

这张映射表是本规范最实用的部分。左列是通用的 provider 流事件形状，右列是你要发的领域事件。

| provider 事件 | 发出的领域事件 | 备注 |
| --- | --- | --- |
| `step-start` | （无） | |
| `text-start` | `Text.Started{textId}` | 首次会先触发 `startAssistant()` → `Step.Started` |
| `text-delta` | `Text.Delta{textId, delta}` | **live-only**，同时在累积器里 append |
| `text-end` | `Text.Ended{textId, text}` | 累积串一次性落库 |
| `reasoning-start` | `Reasoning.Started{reasoningId, providerMetadata}` | |
| `reasoning-delta` | `Reasoning.Delta{…}` | **live-only** |
| `reasoning-end` | `Reasoning.Ended{reasoningId, text, providerMetadata}` | providerMetadata = 思考签名 |
| `tool-input-start` | `Tool.Input.Started{callId, name}` | 建工具条目 |
| `tool-input-delta` | `Tool.Input.Delta{callId, delta}` | **live-only**，工具参数也是流式的 |
| `tool-input-end` | `Tool.Input.Ended{callId, text}` | |
| `tool-call` | `Tool.Called{callId, tool, input, provider}` | 若前面没 input-start 需补发 |
| `tool-result` | `Tool.Success{structured, content, outputPaths}` 或 `Tool.Failed` | 结果类型为 error 时走 Failed |
| `tool-error` | `Tool.Failed{error}` | |
| `step-finish` | （仅记录结算信息） | `Step.Ended` 由执行循环在工具全部结算后发 |
| `finish` | （无） | |
| `provider-error` | `Step.Failed{error}` | 置错误标志，后续事件全丢（R-02-11） |

---

## 5. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 订阅者队列容量 | `256` | 64 – 1024 | 调大：慢客户端更不容易被断开，内存占用上升；调小：更早断开重连，内存更省 |
| 心跳间隔 | `15s` | 10 – 45s | 调大：可能被中间代理按空闲超时切断；调小：无谓流量 |
| 事件类型版本 | `1` | — | 改 data 形状时递增，配合重放校验 |

---

## 6. 接口契约

```ts
interface EventBus {
  publish<D>(def: EventDefinition<D>, data: D, opts?: PublishOptions): Promise<EventPayload<D>>
  /** 注册在事务内运行的投影器 */
  project(type: string, fn: (e: EventPayload) => Promise<void>): void
  /** 有界订阅；溢出时通过 onOverflow 通知调用方断开 */
  subscribe(fn: (e: EventPayload) => void, capacity?: number): { stop(): void; onOverflow(cb: () => void): void }
  /** 增量拉取 durable 事件 */
  readAggregate(aggregateId: string, after?: number, limit?: number): Promise<EventPayload[]>
}
```

---

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| delta 也落库 | 每秒几十次磁盘写，长会话事件表爆炸 |
| 只有 delta 没有 `Ended` | 刷新后内容全无 |
| 事务外跑投影 | 「有事件无投影」裂开状态 |
| 事务提交前广播 | 客户端读到不一致状态 |
| 无界订阅队列 | 慢客户端导致 OOM |
| `await` 每个订阅者 | 一个卡住的标签页拖慢整个 agent |
| 协议违约静默忽略 | 下一轮 provider 400 且无从排查 |
| 不 flush 累积器 | provider 中断时已生成内容丢失 |
| 先拉全量再订阅 | 窗口期事件丢失 |
| 不发心跳 / 不设 `X-Accel-Buffering` | 代理切连接 / 整段延迟 |
| 并发 publish 不串行 | 抢同一个 seq |

---

## 8. 分级实现路径

### 最小可用版（1–2 天）

- 事件总线：`Map<type, projector[]>` + `Set<listener>`
- 两张表 + 一个事务
- SSE 端点（首帧 + 心跳 + 四个响应头）
- durable / live-only 分野**必须做**

### 完整版

- 有界订阅队列 + 溢出断开
- 重放校验（幂等 + 分叉检测）
- 事件类型版本化
- 增量拉取 `after=N`

### 无后端时

事件总线可以只在内存里（`EventTarget`），此时 R-02-02/03/04/06/07 **标记为不适用**，
并在文档里写明「刷新会丢失过程」。不要假装实现。

---

## 9. 验收

见 [`acceptance/SPEC-02.yaml`](../acceptance/SPEC-02.yaml)（23 条）。

最关键的三条：投影器抛错 → 事务回滚且事件不存在（A-02-03）；订阅者收到事件时事件已在库里（A-02-05）；
`*.Delta` 不写库不推进 seq（A-02-10）。
