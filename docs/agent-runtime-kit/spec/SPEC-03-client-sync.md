# SPEC-03 · 客户端同步层

> **优先级** P0 · **工作量** M · **依赖** SPEC-01, SPEC-02
> **分析原文** [ch08 客户端同步层](../../context-and-streaming/08-client-sync.md)
> **验收** [`acceptance/SPEC-03.yaml`](../acceptance/SPEC-03.yaml)（20 条）

**这是「流式不卡」的决定性一层。** 后端每秒推几十个事件，这一层负责让它们只触发每帧一次渲染。

---

## 1. 目标与非目标

### 目标

1. 把事件流 apply 到本地 store（有序、幂等、容忍乱序）
2. **不让每个 delta 触发一次渲染**
3. 断线能自愈且不丢事件
4. 长时间运行内存不无限增长

### 非目标

- 不规定状态管理库。
- 不做离线编辑/冲突合并（服务端是唯一真相源）。

---

## 2. 领域模型

```ts
interface SyncState {
  /** 有序数组，按 id 升序。见 R-03-02 */
  session: Session[]
  /** sessionId → 有序消息数组 */
  message: Record<string, Message[]>
  /** messageId → 有序片段数组 */
  part: Record<string, Part[]>
  /** partId → delta 累积串。见 R-03-03 */
  partTextAccumDelta: Record<string, string>
  sessionStatus: Record<string, SessionStatus>
  permission: Record<string, PermissionRequest[]>
  question: Record<string, QuestionRequest[]>
  todo: Record<string, Todo[]>
}
```

**三级索引**：session 是有序数组；message 按 sessionId 分组；part 按 messageId 分组。
这样单个会话的数据可以独立加载和驱逐，切会话时不用扫全表。

---

## 3. 规范条款

### R-03-01 必须实现两层事件合并 · **必须**

**要求**：

| 层 | 时机 | 合并规则 |
| --- | --- | --- |
| **入队层** | 读流时，事件进队列前 | 队尾是同一个 part 的 `part.updated` → **替换队尾**，不新增 |
| **刷帧层** | 每帧刷新时，apply 之前 | 相邻的、四元组 `(directory?, messageId, partId, field)` 相同的 delta → **拼接成一个** |

**理由**：一帧内收到 20 个字符的 delta，最终只走一次 reducer、一次渲染。不合并 = 每秒几十次渲染，直接掉帧。

**入队层单独存在的理由**：`part.updated` 携带完整值，中间态没有渲染价值；在入队时就丢弃能减少队列长度和刷帧层的工作量。

---

### R-03-02 所有有序集合必须用二分查找插入 · **必须**

**要求**：session / message / part / permission / question 等有序数组，增删改一律先二分定位。

**理由**：`push` 后 `sort` 是 O(n log n) × 每条消息。二分插入是 O(log n) 定位 + 一次 splice。

**前提**：依赖 SPEC-01 `R-01-02`（id 字典序 == 时间序）。

**参考实现**：

```ts
function binarySearch<T>(arr: T[], key: string, getKey: (x: T) => string) {
  let lo = 0, hi = arr.length
  while (lo < hi) {
    const mid = (lo + hi) >> 1
    if (getKey(arr[mid]) < key) lo = mid + 1
    else hi = mid
  }
  return { found: lo < arr.length && getKey(arr[lo]) === key, index: lo }
}
```

---

### R-03-03 delta 累积值与权威值必须分离 · **必须**

**要求**：

- 收到 `part.delta` → 同时写两处：`partTextAccumDelta[partId] += delta` **和** `part[field] += delta`
- 收到 `part.updated`（权威全量值）→ **先删除** `partTextAccumDelta[part.id]`，再写入 part

**理由**：delta 是乐观近似，`updated` 是权威真相（SPEC-02 的 durable / live-only 分野在前端的落点）。不删累积值会出现两份真相，UI 闪烁。

**渲染侧读取顺序**：优先读累积值，没有则读 part 上的权威值。

---

### R-03-04 part 未到达的 delta 必须丢弃 · **必须**

**要求**：`part.delta` 到达时若 `part` 尚不存在，**直接丢弃该 delta**，不创建占位对象。

**理由**：`Started`（durable）与 `Delta`（live-only）是两条路径，极端时序下可能反序。丢弃的代价是这段内容会在后续 `part.updated` 里补回来；硬造占位对象会产生类型不完整的脏数据。

---

### R-03-05 刷帧必须用双缓冲 · **必须**

**要求**：维护两个数组 `queue` / `buffer`，刷帧时交换角色，新事件写进新的 `queue`。

**理由**：刷帧期间到达的事件不能丢、也不能被并发修改。单数组 + `splice` 会漏事件。

---

### R-03-06 刷帧调度必须补齐剩余时间而非固定延时 · **必须**

**要求**：

```ts
timer = setTimeout(flush, Math.max(0, FLUSH_FRAME_MS - (Date.now() - lastFlushAt)))
```

**理由**：固定 `setTimeout(16)` 会让空闲后的第一个事件也白等一帧。补齐剩余时间保证「距上次刷帧够久就立刻刷」。

---

### R-03-07 读流循环必须周期性让出主线程 · **必须**

**要求**：读流的 `for await` 循环里，每隔约 8ms 强制 `await` 一次宏任务（`setTimeout(resolve, 0)`）。

**理由**：不让出的话，事件密集时这个循环会变成一个几百毫秒的长任务，刷帧的 `setTimeout` 排在它后面永远轮不上——**用户看到的是「卡住然后一次性全出来」**。

---

### R-03-08 断线必须自动重连并在重连后拉全量 · **必须**

**要求**：

1. 固定退避重连（参考 250ms）
2. 重连后收到「已连接」首帧 → **触发全量重新拉取**
3. 用 `generation` 计数 + `started` 标志防止 stop/start 快速切换时两个读流循环并存
4. 新循环开始前 `await` 上一个循环彻底结束

**理由**：只重连不拉全量会导致永久性状态漂移（断线期间的事件永远补不回来）。两个循环并存会让事件重复 apply。

**要求**：错误日志必须去重（断网时每 250ms 一次失败会刷屏），收到第一个事件时重置去重标志。

---

### R-03-09 渲染必须订阅最小切片 · **必须**

**要求**（细粒度响应式框架之外的栈，如 React）：

- 组件订阅 store 的**最小必要切片**，不订阅整个 store
- 列表项用引用相等做 memo 比较

**理由**：SolidJS 等细粒度响应式框架天然做到「只有变的那个 part 重渲染」。React 必须手动达成，否则 R-03-01 到 R-03-07 的所有优化都会被「store 一变整棵树重渲染」抵消。

**这不是可选优化，是本规范的硬条款。**

```tsx
// ✅
const parts = useSync((s) => s.part[messageId])
const PartView = React.memo(Inner, (a, b) => a.part === b.part)
// ❌
const store = useSync()
```

---

### R-03-10 必须使用可控的流式读取而非 EventSource · **应该**

**理由**：`EventSource` 不支持自定义 header（认证）、不支持 POST、重连策略不可控。用 `fetch` + `ReadableStream` 手动解析 SSE 帧。

---

### R-03-11 必须正确处理页面生命周期 · **应该**

**要求**：用 `pagehide` / `pageshow` 管理连接，不是 `beforeunload` / `load`。

**理由**：只有前者能正确处理浏览器的前进后退缓存（bfcache）。从 bfcache 恢复时连接早已断开但页面状态还在。

---

### R-03-12 必须有内存治理 · **应该**

**要求**：

1. **会话列表裁剪**：保留最近 N 个根会话 + 额外保留时间窗内活跃的若干个
2. **子会话保留规则**（三选一即保留）：父会话还在 / **有待处理的权限请求** / 时间窗内有更新
3. **缓存清理**：会话被裁剪时，其 message / part / todo / permission / question / status / diff **全部**缓存字典都要清

**理由**：
- 第 2 条的「有待处理权限请求」最关键：**裁掉一个正在等用户批准的子会话，用户就永远批不了了，agent 永久卡死。**
- 第 3 条漏掉任何一个字典就是内存泄漏。part 字典尤其绕（按 messageId 索引，要反查 sessionId）。

---

### R-03-13 不需要的片段类型应在 store 层过滤 · **应该**

**要求**：维护一个跳过集合（如记账用的 `step-start` / `step-finish` / `patch`），这些 part 不进前端 store。

**理由**：UI 从别处（消息级的 tokens、summary）取同样的信息，进 store 只是浪费内存和 diff 成本。

---

## 4. 算法规范

### 4.1 事件泵

```
createEventPump(apply):
  queue = [];  buffer = [];  timer = undefined;  lastFlushAt = 0

  enqueue(event):
    prev = queue[last]
    if event 和 prev 都是 part.updated 且 partId 相同:      # R-03-01 入队层
        queue[last] = event
        return
    queue.push(event)
    schedule()

  schedule():
    if timer: return
    timer = setTimeout(flush, max(0, FLUSH_FRAME_MS - (now - lastFlushAt)))   # R-03-06

  flush():
    timer = undefined
    if queue 为空: return
    events = queue;  queue = buffer;  buffer = events;  queue.length = 0      # R-03-05 双缓冲
    lastFlushAt = now
    apply(coalesce(events))                                                   # 一次性 set store
    buffer.length = 0

  coalesce(events):                                                           # R-03-01 刷帧层
    out = []
    for e in events:
      p = out[last]
      if e 是 delta 且 p 是 delta 且 (messageId, partId, field) 相同:
          out[last] = { ...e, delta: p.delta + e.delta }
          continue
      out.push(浅拷贝 e)        # 浅拷贝：后面会就地改 delta，不拷贝会污染原事件
    return out
```

### 4.2 读流循环

```
runStream(url, pump, signal):
  generation += 1;  active = generation
  while not signal.aborted and generation == active:                          # R-03-08
      attempt = new AbortController()
      link(signal → attempt)
      try:
          res    = fetch(url, { signal: attempt.signal, headers: {Accept: "text/event-stream"} })
          reader = res.body.pipeThrough(TextDecoderStream).getReader()
          buf = "";  yielded = now
          loop:
              { done, value } = reader.read()
              if done: break
              buf += value
              while buf 含 "\n\n":
                  frame = 取出一帧
                  if frame 以 ":" 开头: continue                              # 心跳注释行
                  line = frame 中以 "data:" 开头的行
                  if line: pump.enqueue(JSON.parse(line.slice(5)))
              if now - yielded >= STREAM_YIELD_MS:                            # R-03-07
                  yielded = now
                  await macrotask()
      catch: 去重记日志
      finally: unlink
      if signal.aborted or generation != active: return
      await sleep(RECONNECT_DELAY_MS)
```

### 4.3 事件 reducer（关键分支）

```
applyEvent(draft, event):
  case "message.updated":
    二分定位 → 命中则替换，否则 splice 插入                                    # R-03-02

  case "message.part.updated":
    if part.type in SKIP_PARTS: break                                         # R-03-13
    delete draft.partTextAccumDelta[part.id]                                  # R-03-03 权威覆盖
    二分定位 → 替换或插入

  case "message.part.delta":
    list = draft.part[messageId];  if not list: break
    r = 二分定位(partId);  if not r.found: break                              # R-03-04 丢弃
    cur = list[r.index][field]
    draft.partTextAccumDelta[partId] = (已有 ?? (cur 是字符串 ? cur : "")) + delta
    list[r.index][field] = (已有 ?? "") + delta                               # 双写

  case "message.removed":
    删消息 + 删该消息全部 part + 删这些 part 的 delta 累积                     # 三处都要清

  case "session.created" / "session.updated":
    二分插入 → 跑裁剪 → 清理被裁掉会话的全部缓存                               # R-03-12

  case "server.connected":
    触发全量重新拉取                                                          # R-03-08

  case "permission.asked" / "question.asked":
    二分插入
  case "permission.replied" / "question.replied" / "question.rejected":
    按 requestId 二分定位后移除
```

---

## 5. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| `FLUSH_FRAME_MS` | `16` | 8 – 33 | 调大：渲染次数更少更省 CPU，但文字出现有顿挫感；调小：更顺滑但 CPU 占用上升 |
| `STREAM_YIELD_MS` | `8` | 4 – 16 | 调大：读流吞吐略高但可能出现长任务；调小：让出更频繁，吞吐略降 |
| `RECONNECT_DELAY_MS` | `250` | 200 – 2000 | 调大：断网时重试更少更省；调小：恢复更快但断网时日志更吵 |
| 会话保留上限 | `30` 目录 store / 按产品定会话数 | — | 影响内存占用 |
| 空闲驱逐 TTL | `20 min` | 5 – 60 min | 调小：内存更省但切回旧会话要重新加载 |
| 最近活跃时间窗 | `4 h` | 1 – 24 h | 决定「额外保留」的范围 |

---

## 6. 接口契约

```ts
interface ClientSync {
  /** 由事件泵批量调用，一次 set store */
  applyBatch(events: ServerEvent[]): void
  /** 收到「已连接」时调用 */
  refetchAll(): Promise<void>
  /** 渲染侧读文本：优先累积值 */
  readPartText(partId: string, fallback: string): string
}
```

---

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| 每个事件直接 setState | 每秒几十次渲染，掉帧 |
| 固定 `setTimeout(16)` | 空闲后第一个事件白等一帧 |
| 单数组队列 + splice | 刷帧期间的事件丢失 |
| 读流不让出主线程 | 「卡住然后一次性全出来」 |
| `push` 后 `sort` | O(n log n) × 每条消息 |
| delta 先到就造占位 part | 类型不完整的脏数据 |
| `updated` 到达不清累积值 | 两份真相，UI 闪烁 |
| 只重连不拉全量 | 永久性状态漂移 |
| 不做 generation 防重入 | 两个读流循环并存，事件重复 apply |
| React 里订阅整个 store | 抵消掉全部性能优化 |
| 用 `beforeunload` / `load` | bfcache 场景下连接不恢复 |
| 会话裁剪漏清缓存 | 内存泄漏 |
| 裁掉在等权限的子会话 | agent 永久卡死 |

---

## 8. 分级实现路径

### 最小可用版（1 天）

事件泵（两层合并 + 双缓冲 + 补齐剩余时间）+ 读流循环（generation + 退避 + 让出）+ reducer 的三个关键分支
（`message.updated` / `part.updated` / `part.delta`）。

**这一步做完，流式渲染就从「掉帧」变成「丝滑」，约 150 行代码。**

### 完整版

补齐全部事件分支 + 内存治理 + 页面生命周期 + 错误日志去重。

---

## 9. 验收

见 [`acceptance/SPEC-03.yaml`](../acceptance/SPEC-03.yaml)（20 条）。

性能类断言（`kind: manual`）需要用浏览器 Performance 面板确认：
连续投递 10000 个事件时主线程无 > 50ms 长任务（A-03-09）；
一个 part 流式更新时其它 part 组件不重渲染（A-03-20）。
