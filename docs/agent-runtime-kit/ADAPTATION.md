# 栈适配矩阵

规范用语言无关的伪代码 + TypeScript 接口契约表达。本文件给出常见技术栈的概念对照，
让 AI（和你）在实施时不用每次重新推导等价写法。

**用法**：实施某个能力前，把本文件里**相关的那两三行**连同 SPEC 一起贴给 AI。不要整份贴。

---

## 1. 并发与生命周期原语

规范里出现的抽象概念 → 各栈的等价物。

| 规范概念 | 语义 | TypeScript / Node | Python | Go |
| --- | --- | --- | --- | --- |
| **并发遍历（保序）** | 并发执行、按输入顺序返回 | `Promise.all(xs.map(f))` | `asyncio.gather(*[f(x) for x in xs])` | `errgroup` + 预分配结果切片按下标写 |
| **有限并发** | 最多 N 个同时跑 | `p-limit(n)` / 手写信号量 | `asyncio.Semaphore(n)` | 带缓冲 channel 作信号量 |
| **串行化（全局）** | 一次只有一个在跑 | Promise 链：`chain = chain.then(f)` | `asyncio.Lock()` | `sync.Mutex` |
| **按 key 串行化** | 同 key 互斥，不同 key 并行 | `Map<K, Promise>` 链 | `defaultdict(asyncio.Lock)` | `sync.Map` + 每 key 一个 Mutex |
| **可外部完成的等待** | 挂起直到别处 resolve/reject | `new Promise((res, rej) => …)` 暴露 res/rej | `asyncio.Future()` | 无缓冲 channel |
| **作用域 + 自动回收** | 退出作用域时执行清理 | `try/finally`、`Symbol.dispose` | `contextlib` / `async with` | `defer` |
| **不可中断段** | 这段要么全做完要么不开始 | 不 `await` 外部信号；或标志位守卫 | `asyncio.shield()` | 不 select ctx.Done() |
| **取消信号** | 传播中断 | `AbortController` / `AbortSignal` | `asyncio.CancelledError` / `Task.cancel()` | `context.Context` |
| **发布订阅** | 一对多广播 | `EventEmitter` / `EventTarget` | 自建 `set[Callable]` | channel fan-out |
| **有界丢弃队列** | 满了就断开该订阅者 | 数组 + 长度检查 | `asyncio.Queue(maxsize)` + `put_nowait` | 带缓冲 channel + `select default` |
| **流** | 异步逐项产出 | `AsyncGenerator` / `ReadableStream` | `AsyncIterator` | channel |

> **opencode 用 Effect 实现以上全部。** 规范里不出现 Effect。凡是分析原文里看到
> `Effect.forEach(..., {concurrency: "unbounded"})`、`Deferred`、`Scope`、`acquireRelease`，
> 对照本表换成你栈里的等价物即可，语义完全一致。

### 一个高频陷阱

**「并发遍历（保序）」的"保序"不能丢。** 规范多处（SPEC-12 的来源观测、SPEC-15 的技能加载）
要求并发执行但结果顺序稳定，否则渲染文本会抖动、缓存会失效。

```ts
// ✅ 保序
const results = await Promise.all(sources.map((s) => load(s)))

// ❌ 按完成顺序收集 —— 破坏确定性
const results = []
sources.forEach((s) => load(s).then((r) => results.push(r)))
```

---

## 2. Schema / 校验

| 规范要求 | TypeScript | Python | Go |
| --- | --- | --- | --- |
| 类型定义 + 运行时校验 | `zod` / `valibot` / `@effect/schema` | `pydantic` | struct + `go-playground/validator` |
| 生成 JSON Schema（工具定义用） | `zod-to-json-schema` | `model_json_schema()` | `invopop/jsonschema` |
| 判别联合 | `z.discriminatedUnion("type", [...])` | `Annotated[Union[...], Field(discriminator="type")]` | interface + type 字段 + switch |
| 编解码 + 相等判定（SPEC-12 需要） | zod parse + 稳定深比较 | pydantic + `model_dump()` 比较 | json.Marshal 后比较 |

**SPEC-12 对 codec 有额外要求**：需要 `encode` / `decode` / `equals` 三件套，且 `equals`
**不能对对象键顺序敏感**。用 `JSON.stringify` 做深比较时要先排序键，或者自己写结构化比较。

---

## 3. 前端响应式原语

分析原文用 SolidJS。规范用框架无关描述。

| SolidJS（原文） | React | Vue 3 | Svelte 5 | 语义 |
| --- | --- | --- | --- | --- |
| `createSignal` | `useState` | `ref` | `$state` | 可变响应式值 |
| `createMemo` | `useMemo` | `computed` | `$derived` | 派生值（带缓存） |
| `createEffect` | `useEffect` | `watchEffect` | `$effect` | 副作用 |
| `createStore` | Zustand + Immer | `reactive` | `$state` 深层 | 细粒度可变 store |
| `produce(draft => …)` | Immer `produce` | 直接改 `reactive` | 直接改 `$state` | 不可变更新的可变写法 |
| `reconcile(next, {key})` | 手写 diff + `React.memo` | `v-for :key` | `{#each … (key)}` | 按 key 复用节点 |
| `batch(fn)` | React 18 自动批处理 / `flushSync` | `nextTick` | 自动 | 批量更新 |
| `<Show>` / `<For>` / `<Index>` | `{cond && …}` / `.map()` | `v-if` / `v-for` | `{#if}` / `{#each}` | 条件与列表 |
| `onCleanup` | `useEffect` 返回函数 | `onUnmounted` | `$effect` 返回函数 | 卸载清理 |

### React 特别提醒（规范 SPEC-03 明确要求）

SolidJS 的细粒度响应式**天然做到「只有变的那个 part 重渲染」**。React 必须手动达成同等效果，
否则 SPEC-03/04/05/06 的所有性能优化都会被「store 一变整棵树重渲染」抵消：

```tsx
// ✅ 订阅最小切片
const parts = useSync((s) => s.part[messageId])

// ✅ 列表项按引用比较 memo
const PartView = React.memo(Inner, (a, b) => a.part === b.part)

// ❌ 整个 store 订阅
const store = useSync()
```

**这一条是 SPEC-03 的 `R-03-09` 条款**，不是可选优化。

---

## 4. 存储

| 规范概念 | 说明 | 选型无关要求 |
| --- | --- | --- |
| 事件日志表 | `(id, aggregate_id, seq, type, data)` | `UNIQUE(aggregate_id, seq)` 必须有 |
| 序号表 | `(aggregate_id, seq)` | 与事件写入同事务 |
| 消息表 | 一行一条完整消息 | `UNIQUE(session_id, seq)` |
| 片段表（可选） | 一行一个 part | 只有需要 part 级事件时才拆 |
| 事务 | 投影 + 事件写入原子 | **必须支持嵌套调用内的事务** |

SQLite / Postgres / MySQL 都可以。**唯一的硬要求是事务**——SPEC-02 的 `R-02-04`（投影与事件同事务）
和 SPEC-13 的 `R-13-04`（快照与系统消息同事务）没有事务就无法满足。

无服务端存储（纯前端 + localStorage）**无法满足 SPEC-02**，此时应明确记录为「不适用」而不是假装实现。

---

## 5. 流式传输

| 方案 | 适用 | 规范满足度 | 备注 |
| --- | --- | --- | --- |
| **SSE（`fetch` + ReadableStream）** | 推荐 | 完全满足 | 规范的参考实现方式 |
| SSE（`EventSource`） | 不推荐 | 部分 | 不支持自定义 header（认证）、不支持 POST、重连策略不可控 |
| WebSocket | 可以 | 完全满足 | 需自己实现心跳与重连退避；双向能力用不上 |
| 长轮询 | 降级方案 | 勉强 | 延迟高，但 `server.connected` + 全量补齐的语义仍可保留 |

用 SSE 时这几个响应头**不是可选的**（SPEC-02 `R-02-12`）：

```
Content-Type: text/event-stream
Cache-Control: no-cache, no-transform
X-Accel-Buffering: no          ← 不加，nginx 后面会整段延迟数秒
X-Content-Type-Options: nosniff
```

---

## 6. LLM Provider 适配

规范不绑定 provider。需要从 provider SDK 拿到的能力：

| 规范依赖 | 说明 | 拿不到时的降级 |
| --- | --- | --- |
| 流式事件（text / tool-call / usage） | SPEC-02 的事件映射源头 | 无法降级，必须有 |
| 工具定义（JSON Schema） | SPEC-08 | 无法降级 |
| 上下文窗口大小 | SPEC-14 的压缩判据 | 拿不到则**不压缩**（规范要求），不要猜 |
| 输出 token 上限 | SPEC-14 | 用保守默认值 |
| usage 分项（input / output / cache 读写） | SPEC-17 用量拆分 | 只显示总量，`cache` 段标为不可用 |
| reasoning 签名 / 续接凭据 | SPEC-01 跨模型降级 | 换模型时把 reasoning 降级成普通文本 |
| prompt cache key | SPEC-13 的收益体现处 | 纪元机制仍然正确，只是省不到钱 |

---

## 7. 常见「我没有这个」的处理

| 缺什么 | 影响的能力 | 建议 |
| --- | --- | --- |
| 没有后端（纯前端应用） | SPEC-02、07、13、14 | 只做前端能力（01/03/04/05/06/17）；事件层用内存版，明确记录刷新会丢 |
| 没有工具调用 | SPEC-08~11、15 | 全部标为不适用 |
| 没有文件系统访问 | SPEC-10、15（目录来源）、16（L1/L2） | 技能可退化为「数据库里的一张表」，规范其余部分不变 |
| 没有多会话 | SPEC-07 的会话级串行 | 用全局串行替代，其余不变 |
| 单用户本地应用 | SPEC-09 权限 | 仍建议实现：它同时是「操作确认」机制，不只是安全机制 |
| 模型不支持 reasoning | SPEC-01 的 reasoning part | 该 part 类型不出现，其余不变 |

**重要**：不适用的能力要在审计报告里**显式标注 `不适用` + 理由**，不要标成「已实现」或留空。
`01-audit.md` 会强制要求这一点。
