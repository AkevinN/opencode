# SPEC-12 · 系统上下文来源

> **优先级** P2 · **工作量** L · **依赖** 无（但被 13/15/16 依赖）
> **分析原文** [ch01 SystemContext 抽象](../../context-and-streaming/01-system-context.md)
> **验收** [`acceptance/SPEC-12.yaml`](../acceptance/SPEC-12.yaml)（15 条）

**零依赖但被多方依赖**，可以独立于会话系统先做出来并单测。与 SPEC-13 **必须成对实现**。

---

## 1. 目标与非目标

### 目标

一个 agent 的系统提示词里混着两类东西：

- **真正静态的**：角色设定、工具规范、输出格式
- **反映当前世界状态的**：工作目录、是否 git 仓库、今天几号、项目规则文件说了什么、当前可用哪些技能

第二类会在会话进行中变化。两条朴素路都是死路：

| 做法 | 后果 |
| --- | --- |
| 每轮重拼 system prompt | provider 缓存每轮全失效（前缀变了），成本和延迟暴涨；模型困惑于「同一件事上轮看到的是另一个值」 |
| 第一轮拼好就不动 | 模型拿着过期事实干活：在旧目录找文件、无视刚加的项目规范 |

**第三条路**：system prompt 在一个「纪元」内字节不变，世界状态的变化以**会话中系统消息**追加进历史。

要做到这一点，必须能精确回答：「这一轮，相比上次告诉模型的，到底有什么变了？」——**这就是本规范存在的唯一理由**。它不负责发消息、不负责存储，只负责**观测、比较、渲染**。

### 非目标

- 不负责持久化（SPEC-13 负责）。
- 不负责发消息（SPEC-13 负责）。
- 不规定有哪些来源（SPEC-16 给出参考来源集）。

---

## 2. 领域模型

```ts
export type Json = null | boolean | number | string | Json[] | { [k: string]: Json }

/** 稳定命名空间 key，形如 "core/environment"、"plugin/my-thing" */
export type SourceKey = string
const KEY_PATTERN = /^[a-z0-9][a-z0-9._-]*\/[a-z0-9][a-z0-9._/-]*$/

/** 表示「暂时观测不到」，区别于「观测到不存在」。见 R-12-05 */
export const UNAVAILABLE = Symbol.for("system-context/unavailable")
export type Unavailable = typeof UNAVAILABLE

export interface Source<A> {
  key: SourceKey
  codec: {
    encode: (value: A) => Json
    decode: (json: Json) => { ok: true; value: A } | { ok: false }
    /** 不得对对象键顺序敏感 */
    equals: (a: A, b: A) => boolean
  }
  /** 不会 reject。读不到时返回 UNAVAILABLE */
  load: () => Promise<A | Unavailable>
  /** 纪元开始时的完整渲染 */
  baseline: (current: A) => string
  /** 值变化时的「现在生效值是」渲染 */
  update: (previous: A, current: A) => string
  /** 来源消失时的渲染。不提供则该来源消失会触发整体重建 */
  removed?: (previous: A) => string
}

export interface SourceSnapshot {
  value: Json
  /** 预渲染的移除文本，仅当 Source 提供了 removed 时存在。见 R-12-08 */
  removed?: string
}
export type Snapshot = Readonly<Record<SourceKey, SourceSnapshot>>

export interface Generation { baseline: string; snapshot: Snapshot }

export type ReconcileResult =
  | { tag: "Unchanged" }
  | { tag: "Updated"; text: string; snapshot: Snapshot }
  | { tag: "ReplacementReady"; generation: Generation }
  | { tag: "ReplacementBlocked" }
```

**`SourceSnapshot.removed` 是最容易漏掉的一个字段**：移除文本必须在来源**还活着**的时候预先渲染并存进快照。等来源真的消失时你已经拿不到它的值，渲染不出「XX 不再适用」这句话。

---

## 3. 规范条款

### R-12-01 key 必须唯一，重复即失败 · **必须**

**要求**：组合多个来源时出现重复 key → 立即抛错，**不做静默去重或覆盖**。

**理由**：静默覆盖会造成极难排查的「我的指令为什么没生效」。

---

### R-12-02 渲染必须确定性 · **必须**

**要求**：同一组来源、同样的值，组合出的文本**逐字节相同**。具体要求两条：

1. 并发读取必须**保序返回**（用 `Promise.all` 而非按完成顺序收集）
2. 注册表组合前必须**按 key 排序**

**理由**：文本抖动会让 SPEC-13 的 baseline 变化，provider 缓存全失效。

---

### R-12-03 渲染结果不得为空 · **必须**

**要求**：任何渲染函数返回空串 → 抛错（带来源 key 和渲染种类）。

**理由**：空串会让连接符产生连续空行，污染 prompt。这是编程错误，应该早失败。

---

### R-12-04 值类型必须被隐藏 · **必须**

**要求**：不同来源的值类型不同（字符串、数组、对象），但组合后必须能放进同一个数组统一处理。

**实现方式**：用存在类型编码——构造函数闭包捕获类型 `A` 和它的 codec，对外只暴露「已经闭合了 A 的操作」：

```ts
interface PackedSource {
  key: SourceKey
  load: () => Promise<Loaded | Unavailable>
}
interface Loaded {
  baseline: () => { text: string; snapshot: SourceSnapshot }
  compare: (previousJson: Json) => Compared
}
type Compared =
  | { tag: "Incompatible" }
  | { tag: "Unchanged" }
  | { tag: "Updated"; render: () => { text: string; snapshot: SourceSnapshot } }
```

**外部代码永远不需要知道 `A` 是什么。** 这是本规范最值得抄的一个设计。

---

### R-12-05 「暂时读不到」与「读到不存在」必须严格区分 · **必须**

| 情况 | 返回 | 处理 |
| --- | --- | --- |
| 成功读到，值为空/不存在 | 正常值（如空数组） | 走正常比较；可能触发 `removed` 渲染 |
| **观测失败**（文件被锁、网络超时） | `UNAVAILABLE` | **保留上次已告知模型的快照，不发任何更新** |

**理由**：这是整个规范里最容易做错的一条。把「读失败」当成「配置被删了」，模型会突然不守规矩。

**要求**：从未成功读过的来源返回 `UNAVAILABLE` 时，**不写快照条目**（不要写 `{value: null}` 占位——那会让下次真读到时误判成"变化"）。

---

### R-12-06 快照与通知必须原子推进 · **必须**

**要求**：「告诉模型变化」这件事持久化成功后，快照才能前进；否则下次必须重新报告同一个变化。

**实现**：由 SPEC-13 通过事务钩子保证。本规范只要求 `reconcile` 返回的 `snapshot` 必须与 `text` 成对交付，调用方不得只用其一。

---

### R-12-07 新注册的来源必须补发一次 baseline · **必须**

**要求**：出现在当前来源集合但不在旧快照里的 key → 渲染它的 `baseline()` 并作为更新发出。

---

### R-12-08 来源消失的处理必须二分 · **必须**

| 情况 | 处理 |
| --- | --- |
| 快照里有预渲染的 `removed` 文本 | 发出该文本，从快照删掉该 key |
| 没有 `removed` 文本 | **整体重建 baseline**（无法用增量表达这个变化） |

**理由**：不做后一条会静默丢弃，模型还记着那条已失效的指令。

---

### R-12-09 快照解码失败必须触发重建 · **必须**

**要求**：快照里存的 JSON 用当前 codec 解不出来（代码升级改了值结构）→ 整体重建。

**理由**：不能猜。当作「无变化」会永久卡住（新值永远发不出去）。

---

### R-12-10 初始化时任一来源不可用必须整体失败 · **必须**

**要求**：`initialize` 时有任何来源返回 `UNAVAILABLE` → 抛出「初始化被阻塞」错误（带 key 列表），**不生成任何 baseline**。

**理由**：宁可让这一轮跑不起来（用户输入保持可重试），也不要用缺了半截的 system prompt 去问模型——那会污染整个纪元的 provider 缓存。

---

### R-12-11 重建必须有守门条件 · **必须**

**要求**：`replace` 时，**只有「曾经成功告知过模型、现在却读不到」的来源**才阻塞重建，返回 `ReplacementBlocked`。

**要求**：从来没成功读过的来源不阻塞（它本来就不在 baseline 里，缺了不算退化）。

---

### R-12-12 一次边界上的多个变化必须合并成一条 · **必须**

**要求**：多个来源同时变化时，全部合并进**一个** `Updated.text`（用统一连接符，如空行）。

**理由**：发多条会在历史里留下一串碎片。

---

### R-12-13 注册必须作用域化 · **应该**

**要求**：注册返回注销句柄；作用域结束自动注销。注销后该来源在下一个边界上通过 `removed` 或整体重建反映出去。

---

### R-12-14 规则类来源的 update 必须全量重发 · **应该**

**要求**：值语义是「规则/枚举」的来源（项目指令、可用技能名录、可用资源名录），`update` 渲染**全量重发**，并明说「取代之前全部」。

**理由**：指令是命令式的，发 diff 会让模型不知道最终生效的是什么。这条在 SPEC-15/16 里会反复出现。

**参考措辞**：`These instructions replace all previously loaded ambient instructions.` / `This list supersedes the previous list.`

---

## 4. 算法规范

### 4.1 make（类型隐藏）

```
makeSource(source):
    return {
      key: source.key,
      load: async () => {
        value = await source.load()
        if value is UNAVAILABLE: return UNAVAILABLE

        snapshot = () => ({
          value: source.codec.encode(value),
          ...(source.removed ? { removed: requireText(source.removed(value)) } : {}),   # R-12-08 预渲染
        })

        return {
          baseline: () => ({ text: requireText(source.baseline(value)), snapshot: snapshot() }),
          compare: (previousJson) => {
            d = source.codec.decode(previousJson)
            if not d.ok: return { tag: "Incompatible" }                                 # R-12-09
            if source.codec.equals(d.value, value): return { tag: "Unchanged" }
            return { tag: "Updated",
                     render: () => ({ text: requireText(source.update(d.value, value)),
                                      snapshot: snapshot() }) }
          },
        }
      },
    }

requireText(t): if t.length == 0: throw "rendered an empty text"; return t              # R-12-03
```

### 4.2 initialize

```
initialize(context):
    entries = await observe(context)                        # 并发保序
    unavailable = entries 中 UNAVAILABLE 的 key 列表
    if unavailable 非空: throw InitializationBlocked(unavailable)                        # R-12-10
    rendered = entries.map(e => [e.key, e.baseline()])
    return {
      baseline: rendered.map(([, r]) => r.text).join(SEPARATOR),
      snapshot: Object.fromEntries(rendered.map(([k, r]) => [k, r.snapshot])),
    }
```

### 4.3 reconcile（本规范最复杂的函数）

**必须严格两趟**：第一趟只判定不产生副作用，这样 `Replace` 能在不浪费渲染的情况下早退出。

```
reconcile(context, previous):
    entries = await observe(context)
    r = reconcileObservation(entries, previous)
    if r.tag in ("Unchanged", "Updated"): return r
    return replaceObservation(entries, previous)             # r.tag == "Replace"

reconcileObservation(entries, previous):
    keys = entries 的 key 集合

    # ---- 第一趟：只判定 ----
    comparisons = {}
    for e in entries:
        if e 是 UNAVAILABLE: continue                        # R-12-05 不参与比较
        stored = previous[e.key];  if not stored: continue   # 新增，第二趟处理
        c = e.compare(stored.value)
        if c.tag == "Incompatible": return { tag: "Replace" }                            # R-12-09
        comparisons[e.key] = c

    for key in Object.keys(previous).sort():                 # 排序 → 确定性
        if keys 含 key: continue
        if previous[key].removed 未定义: return { tag: "Replace" }                        # R-12-08

    # ---- 第二趟：产出 ----
    snapshot = {};  updates = []
    for e in entries:
        stored = previous[e.key]
        if e 是 UNAVAILABLE:
            if stored: snapshot[e.key] = stored              # R-12-05 保留旧快照
            continue                                          # 从未成功读过 → 不写条目
        if not stored:
            r = e.baseline();  updates.push(r.text);  snapshot[e.key] = r.snapshot       # R-12-07
            continue
        c = comparisons[e.key]
        if c.tag == "Unchanged": snapshot[e.key] = stored     # 原样搬运，不重新编码
        else:
            r = c.render();  updates.push(r.text);  snapshot[e.key] = r.snapshot

    for key in Object.keys(previous).sort():
        if keys 含 key: continue
        updates.push(previous[key].removed)                   # 用预渲染的移除文本
        # 注意：不把该 key 写进新快照 —— 它就此消失

    if updates 为空: return { tag: "Unchanged" }
    return { tag: "Updated", text: updates.join(SEPARATOR), snapshot }                   # R-12-12
```

### 4.4 replace（重建守门人）

```
replaceObservation(entries, previous):
    if entries 中存在「UNAVAILABLE 且 previous 里有该 key」的:                            # R-12-11
        return { tag: "ReplacementBlocked" }
    return { tag: "ReplacementReady", generation: initializeObservation(entries) }
```

### 4.5 注册表

```
registry.load():
    current = 全部注册项，按 key 字典序排序                    # R-12-02
    parts   = await Promise.all(current.map(e => e.load()))    # 并发但保序
    return combine(parts)                                      # combine 内部校验 key 唯一（R-12-01）
```

### 4.6 四种结果的调用方动作

| 结果 | 含义 | 调用方（SPEC-13）该做什么 |
| --- | --- | --- |
| `Unchanged` | 世界没变 | 沿用现有 baseline，什么都不发 |
| `Updated{text, snapshot}` | 有增量变化，可用一条消息表达 | 发一条会话中系统消息，**同事务**推进快照 |
| `ReplacementReady{generation}` | 无法增量表达，但能安全重建 | 换 baseline，重置快照，开启新纪元 |
| `ReplacementBlocked` | 需要重建，但所需来源读不到 | **什么都不做**，沿用旧 baseline，下次边界再试 |

---

## 5. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 连接符 | 空行（`"\n\n"`） | — | 影响 prompt 可读性 |
| key 命名规范 | `namespace/name` | — | 建议保留命名空间以便区分核心与插件来源 |
| 并发度 | 无限制 | 有限/无限 | 来源很多且都是 I/O 时可加限流 |

---

## 6. 接口契约

```ts
interface SystemContext {
  make<A>(source: Source<A>): PackedContext
  combine(contexts: PackedContext[]): PackedContext              // 重复 key 抛错
  initialize(ctx: PackedContext): Promise<Generation>            // 阻塞则抛错
  reconcile(ctx: PackedContext, previous: Snapshot): Promise<ReconcileResult>
  replace(ctx: PackedContext, previous: Snapshot): Promise<ReconcileResult>
}

interface SourceRegistry {
  register(entry: { key: SourceKey; load: () => Promise<PackedContext> }): { unregister(): void }
  load(): Promise<PackedContext>                                 // 按 key 排序后组合
}
```

---

## 7. 参考来源集

实现时先做**一个**来源跑通全链路，再加第二个验证组合与排序确定性。

| key | 值类型 | baseline 措辞 | update 措辞 |
| --- | --- | --- | --- |
| `core/environment` | 字符串 | `Here is some useful information about the environment you are running in:` + 结构化块 | `The environment you are running in is now:` + 结构化块 |
| `core/date` | 字符串 | `Today's date: X` | `Today's date is now: X` |
| `core/instructions` | 文件数组 | 逐个 `Instructions from: <path>\n<content>` | `These instructions replace all previously loaded ambient instructions.` + 全量 |
| `core/skill-guidance` | 摘要数组 | 技能名录（见 SPEC-15） | `The available skills have changed. This list supersedes the previous available skills list.` |
| `core/reference-guidance` | 摘要数组 | 资源名录（见 SPEC-16） | `The available project references have changed. This list supersedes the previous reference list.` |

**注意 baseline 与 update 的措辞差异**——这是提示工程，不是随手写：baseline 陈述初始事实，update 用 `now` / `supersedes` 明确表达「取代之前」。

---

## 8. 反模式

| 反模式 | 后果 |
| --- | --- |
| 重复 key 静默覆盖 | 极难排查的「指令不生效」 |
| 按完成顺序收集并发结果 | 文本抖动 → 缓存全失效 |
| 组合前不排序 | 同上 |
| 渲染空串不报错 | 连续空行污染 prompt |
| 不隐藏值类型 | 异构来源无法统一处理，每加一种类型改一处调度 |
| 读失败当成「被删了」 | 模型突然不守规矩 |
| 从未读过的来源写 `{value:null}` 占位 | 首次读到时误判成变化 |
| 移除文本在来源消失后才渲染 | 渲染不出来（值已经没了） |
| 来源消失且无 removed 时静默丢弃 | 模型还记着失效的指令 |
| 快照解码失败当作无变化 | 永久卡住 |
| 初始化不完整仍生成 baseline | 污染整个纪元的缓存 |
| 多个变化发多条消息 | 历史里一串碎片 |
| 规则类来源发 diff | 模型不确定最终生效什么 |
| `reconcile` 不分两趟 | `Replace` 时浪费渲染 |

---

## 9. 分级实现路径

### 最小可用版（1 天）

可以砍掉：`removed` 渲染（所有来源进程启动时固定注册）、`UNAVAILABLE`（来源全是内存值）、
`Incompatible` 重建（不在乎跨版本会话）、并发（来源少于 10 个且都是内存值）。

**绝对不能砍**：`baseline`/`update` 双渲染函数、快照按 key 存、快照与消息原子推进。
砍掉任何一个，这套设计就退化成「每轮重拼 prompt」，缓存全丢。

### 完整版

补齐 `UNAVAILABLE` 语义、`removed` 渲染、重建守门、作用域注册。

---

## 10. 验收

见 [`acceptance/SPEC-12.yaml`](../acceptance/SPEC-12.yaml)（15 条）。

**必须优先验证**：打乱来源注册顺序后 baseline 逐字节相同（A-12-02）；
某来源第二次返回不可用时 `reconcile` 返回 `Unchanged` 且快照保持不变（A-12-06）。
