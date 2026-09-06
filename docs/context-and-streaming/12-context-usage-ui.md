# 12 · 上下文用量可视化：让用户看见窗口被谁吃掉了

源码：`packages/app/src/components/session/session-context-metrics.ts`（65 行）、`session-context-breakdown.ts`（132 行）、`session-context-format.ts`（20 行）、`session-context-tab.tsx`

---

## 1. 解决什么问题

长会话产品最常收到的两个问题：

- 「为什么突然变慢了 / 变贵了？」
- 「为什么它把前面说的忘了？」

答案都在上下文用量里，但 provider 只回一个 `input_tokens` 总数，不告诉你这些 token 是被 system prompt、用户消息、模型回答还是工具输出吃掉的。

opencode 的做法：**用字符数估算各类占比，再按 provider 报告的真实 `input` 归一化。** 结果是一条堆叠条 + 一张统计表，用户能一眼看出「哦，78% 是工具输出」。

这个能力对 agent 类产品几乎是必需的——工具输出爆炸是最常见的上下文杀手，没有可视化用户根本不知道该收敛哪里。

---

## 2. 概念模型与不变量

1. **U1** 总用量取**最后一条有 token 记录的 assistant 消息**，不是累加所有消息。
2. **U2** 分类占比是**估算**（字符数 ÷ 4），但总和必须精确等于 provider 报告的 `input`。
3. **U3** 估算超过实际时**按比例缩放**；不足时差额归入 `other`。
4. **U4** 零值分类不显示。
5. **U5** 缺失数据显示 `—`，不显示 `0` 或 `NaN`。

---

## 3. 数据结构定义

### 3.1 原样摘录

```ts
type Context = {
  message: AssistantMessage
  provider?: Provider
  model?: Model
  providerLabel: string
  modelLabel: string
  limit: number | undefined
  input: number
  total: number
  usage: number | null      // 百分比，limit 缺失时为 null
}
```
> `packages/app/src/components/session/session-context-metrics.ts:16-26`

```ts
export type SessionContextBreakdownKey = "system" | "user" | "assistant" | "tool" | "other"

export type SessionContextBreakdownSegment = {
  key: SessionContextBreakdownKey
  tokens: number
  width: number      // 百分比（未取整，用于 CSS width）
  percent: number    // 百分比（保留一位小数，用于显示）
}
```
> `session-context-breakdown.ts:3-10`

### 3.2 等价纯 TS 版

可以原样复制，无框架依赖。

---

## 4. 核心算法

### 4.1 总用量（U1）

```ts
const tokenTotal = (msg: AssistantMessage) =>
  msg.tokens.input + msg.tokens.output + msg.tokens.reasoning + msg.tokens.cache.read + msg.tokens.cache.write

const lastAssistantWithTokens = (messages: Message[]) => {
  for (let i = messages.length - 1; i >= 0; i--) {
    const msg = messages[i]
    if (msg.role !== "assistant") continue
    if (tokenTotal(msg) <= 0) continue        // 跳过没有 usage 的（失败/中断的消息）
    return msg
  }
}

const build = (messages: Message[] = [], providers: Provider[] = []): Context | undefined => {
  const message = lastAssistantWithTokens(messages)
  if (!message) return undefined
  const provider = providers.find((item) => item.id === message.providerID)
  const model = provider?.models[message.modelID]
  const limit = model?.limit.context
  const total = tokenTotal(message)
  return {
    message, provider, model,
    providerLabel: provider?.name ?? message.providerID,
    modelLabel: model?.name ?? message.modelID,
    limit,
    input: message.tokens.input,
    total,
    usage: limit ? Math.round((total / limit) * 100) : null,
  }
}
```
> `session-context-metrics.ts:28-61`

**为什么取最后一条而不是累加**：每一轮的 `input` 已经包含了整个历史。累加会得到一个几倍于窗口的荒谬数字。**最后一轮的 `input` 就是"当前上下文有多满"**。

`tokenTotal` 把 input + output + reasoning + cache.read + cache.write 全加起来，反映的是「这一轮总共经手了多少 token」，用于算窗口占用率。而堆叠条用的是 `input`（纯输入侧）。

**`providerLabel` / `modelLabel` 的回退**：目录里查不到就显示 id。**不要显示 "Unknown"**——id 至少是可查的信息。

### 4.2 字符数估算

```ts
const estimateTokens = (chars: number) => Math.ceil(chars / 4)

const charsFromUserPart = (part: Part) => {
  if (part.type === "text") return part.text.length
  if (part.type === "file") return part.source?.text.value.length ?? 0
  if (part.type === "agent") return part.source?.value.length ?? 0
  return 0
}

const charsFromAssistantPart = (part: Part) => {
  if (part.type === "text") return { assistant: part.text.length, tool: 0 }
  if (part.type === "reasoning") return { assistant: part.text.length, tool: 0 }
  if (part.type !== "tool") return { assistant: 0, tool: 0 }

  const input = Object.keys(part.state.input).length * 16          // 参数按键数 × 16 粗估
  if (part.state.status === "pending")   return { assistant: 0, tool: input + part.state.raw.length }
  if (part.state.status === "completed") return { assistant: 0, tool: input + part.state.output.length }
  if (part.state.status === "error")     return { assistant: 0, tool: input + part.state.error.length }
  return { assistant: 0, tool: input }                              // running
}
```
> `session-context-breakdown.ts:12-33`

三个约定：
- **`chars / 4`** 是英文文本的经验值。中文会低估（一个汉字通常 ≈ 1 token 但占 1 字符，实际比率接近 1:1）——但因为最后要按 U3 归一化，系统性偏差会被吸收掉。
- **工具参数按 `键数 × 16` 估**，不按 JSON 字符串长度。理由是参数的 key 名和结构本身有固定开销，而值的长度差异极大；这个粗估在参数不大时够用。
- **reasoning 算在 assistant 里**。可以拆出来单独一类，看你的产品要不要强调思考成本。

### 4.3 归一化（U2 / U3）

```ts
export function estimateSessionContextBreakdown(args: {
  messages: Message[]
  parts: Record<string, Part[] | undefined>
  input: number
  systemPrompt?: string
}) {
  if (!args.input) return []

  const counts = args.messages.reduce(
    (acc, msg) => {
      const parts = args.parts[msg.id] ?? []
      if (msg.role === "user") {
        const user = parts.reduce((sum, part) => sum + charsFromUserPart(part), 0)
        return { ...acc, user: acc.user + user }
      }
      if (msg.role !== "assistant") return acc
      const assistant = parts.reduce(
        (sum, part) => {
          const next = charsFromAssistantPart(part)
          return { assistant: sum.assistant + next.assistant, tool: sum.tool + next.tool }
        },
        { assistant: 0, tool: 0 },
      )
      return { ...acc, assistant: acc.assistant + assistant.assistant, tool: acc.tool + assistant.tool }
    },
    { system: args.systemPrompt?.length ?? 0, user: 0, assistant: 0, tool: 0 },
  )

  const tokens = {
    system: estimateTokens(counts.system),
    user: estimateTokens(counts.user),
    assistant: estimateTokens(counts.assistant),
    tool: estimateTokens(counts.tool),
  }
  const estimated = tokens.system + tokens.user + tokens.assistant + tokens.tool

  if (estimated <= args.input) {
    return build({ ...tokens, other: args.input - estimated }, args.input)     // 差额归 other
  }

  const scale = args.input / estimated                                          // 按比例缩放
  const scaled = {
    system: Math.floor(tokens.system * scale),
    user: Math.floor(tokens.user * scale),
    assistant: Math.floor(tokens.assistant * scale),
    tool: Math.floor(tokens.tool * scale),
  }
  const total = scaled.system + scaled.user + scaled.assistant + scaled.tool
  return build({ ...scaled, other: Math.max(0, args.input - total) }, args.input)   // 取整余数归 other
}
```
> `session-context-breakdown.ts:70-132`

**这是整个模块的精髓，两条路径：**

| 情况 | 处理 | `other` 的语义 |
| --- | --- | --- |
| 估算 ≤ 实际 | 各项保持估算值 | 没算到的部分：工具定义 schema、消息元数据、provider 自己加的包装、tokenizer 的实际开销 |
| 估算 > 实际 | 各项按 `input / estimated` 缩放 | 只是 `Math.floor` 的取整余数（很小） |

**这个设计的好处**：条形图的宽度加起来永远是 100%，且各段比例反映真实结构。用户不会看到「加起来 137%」这种破绽。

`build` 过滤零值（U4）并算出两种百分比：

```ts
const toPercent = (tokens: number, input: number) => (tokens / input) * 100
const toPercentLabel = (tokens: number, input: number) => Math.round(toPercent(tokens, input) * 10) / 10

const build = (tokens, input) =>
  [
    { key: "system", tokens: tokens.system },
    { key: "user", tokens: tokens.user },
    { key: "assistant", tokens: tokens.assistant },
    { key: "tool", tokens: tokens.tool },
    { key: "other", tokens: tokens.other },
  ]
    .filter((x) => x.tokens > 0)                                 // U4
    .map((x) => ({ key: x.key, tokens: x.tokens,
                   width: toPercent(x.tokens, input),            // CSS 用，不取整
                   percent: toPercentLabel(x.tokens, input) }))  // 显示用，保留一位小数
```
> `session-context-breakdown.ts:35-68`

**`width` 不取整、`percent` 取一位小数** —— 取整的 width 累加会有误差导致条形图右侧留白。

### 4.4 显示格式化（U5）

```ts
export function createSessionContextFormatter(locale: string) {
  return {
    number(value: number | null | undefined) {
      if (value === undefined) return "—"
      if (value === null) return "—"
      return value.toLocaleString(locale)
    },
    percent(value: number | null | undefined) {
      if (value === undefined) return "—"
      if (value === null) return "—"
      return value.toLocaleString(locale) + "%"
    },
    time(value: number | undefined) {
      if (!value) return "—"
      return DateTime.fromMillis(value).setLocale(locale).toLocaleString(DateTime.DATETIME_MED)
    },
  }
}
```
> `session-context-format.ts:3-19`

统一用 `—`（em dash）表示"无数据"，且所有数字走 `toLocaleString(locale)`（千分位按语言环境）。

---

## 5. 视觉规格

### 5.1 配色

```ts
const BREAKDOWN_COLOR: Record<SessionContextBreakdownKey, string> = {
  system:    "var(--syntax-info)",
  user:      "var(--syntax-success)",
  assistant: "var(--syntax-property)",
  tool:      "var(--syntax-warning)",
  other:     "var(--syntax-comment)",
}
```
> `packages/app/src/components/session/session-context-tab.tsx:25-31`

**复用语法高亮的调色板**而不是新定义一套。好处：自动跟随主题切换、与代码块视觉一致、深浅色模式都已验证过对比度。`tool` 用 warning 色也是个信号——工具输出是最该警惕的那一类。

### 5.2 结构

```tsx
{/* 堆叠条 */}
<div class="...">
  <For each={breakdown()}>
    {(segment) => (
      <div style={{ width: `${segment.width}%`, "background-color": BREAKDOWN_COLOR[segment.key] }} />
    )}
  </For>
</div>

{/* 图例 */}
<div class="flex flex-wrap gap-x-3 gap-y-1">
  <For each={breakdown()}>
    {(segment) => (
      <div class="...">
        <div class="size-2 rounded-sm" style={{ "background-color": BREAKDOWN_COLOR[segment.key] }} />
        <div>{breakdownLabel(segment.key)}</div>
        <div class="text-text-weaker">{segment.percent.toLocaleString(language.intl())}%</div>
      </div>
    )}
  </For>
</div>
```
> `session-context-tab.tsx:322-341`

图例用 `flex-wrap`——窄面板下自动换行，不做水平滚动。

### 5.3 统计表（16 行）

```ts
const stats = [
  { label: "context.stats.session",           value: () => info()?.title ?? params.id ?? "—" },
  { label: "context.stats.messages",          value: () => counts().all.toLocaleString(language.intl()) },
  { label: "context.stats.provider",          value: providerLabel },
  { label: "context.stats.model",             value: modelLabel },
  { label: "context.stats.limit",             value: () => formatter().number(ctx()?.limit) },
  { label: "context.stats.totalTokens",       value: () => formatter().number(ctx()?.total) },
  { label: "context.stats.usage",             value: () => formatter().percent(ctx()?.usage) },
  { label: "context.stats.inputTokens",       value: () => formatter().number(ctx()?.input) },
  { label: "context.stats.outputTokens",      value: () => formatter().number(ctx()?.message.tokens.output) },
  { label: "context.stats.reasoningTokens",   value: () => formatter().number(ctx()?.message.tokens.reasoning) },
  { label: "context.stats.cacheTokens",
    value: () => `${formatter().number(ctx()?.message.tokens.cache.read)} / ${formatter().number(ctx()?.message.tokens.cache.write)}` },
  { label: "context.stats.userMessages",      value: () => counts().user.toLocaleString(language.intl()) },
  { label: "context.stats.assistantMessages", value: () => counts().assistant.toLocaleString(language.intl()) },
  { label: "context.stats.totalCost",         value: cost },
  { label: "context.stats.sessionCreated",    value: () => formatter().time(info()?.time.created) },
  { label: "context.stats.lastActivity",      value: () => formatter().time(ctx()?.message.time.created) },
]
```
> `session-context-tab.tsx:204-226`

**缓存读/写显示成 `1,234 / 567`**（读在前）——一眼能看出缓存命中率。

### 5.4 重算时机

```ts
const breakdown = createMemo(
  on(
    () => [ctx()?.message.id, ctx()?.input, messages().length, systemPrompt()],
    () => { /* 计算 */ },
  ),
)
```
> `session-context-tab.tsx:180-194`

**只依赖四个值**：最新有 usage 的消息 id、它的 input、消息总数、system prompt。**不依赖 parts 内容本身**——否则流式期间每帧都要遍历全部消息做字符统计，那是 O(全部内容) 的操作。

React 等价：

```ts
const breakdown = useMemo(
  () => estimateSessionContextBreakdown({ messages, parts, input: ctx?.input ?? 0, systemPrompt }),
  [ctx?.message.id, ctx?.input, messages.length, systemPrompt],   // 故意不依赖 parts
)
```

> ⚠️ React 的 `exhaustive-deps` lint 会警告缺少 `messages` / `parts`。**这里要显式禁用它并写注释说明**——这是刻意的性能取舍，不是疏忽。

---

## 6. 边界情况

| 场景 | opencode 怎么处理 | 你自己实现时必须处理 |
| --- | --- | --- |
| 会话还没有 assistant 消息 | `build` 返回 `undefined`，整个面板显示空态 | 别渲染出一堆 `NaN%` |
| 最后一条 assistant 失败（无 usage） | `tokenTotal <= 0` 跳过，往前找 | 会显示 0 token |
| 模型不在 provider 目录里 | `limit` 为 `undefined` → `usage` 为 `null` → 显示 `—` | 会算出 `Infinity%` |
| `input` 为 0 | `if (!args.input) return []` | 除零得到 `NaN` |
| 估算超过实际（中文场景常见） | 按比例缩放 | 条形图超过 100% |
| 缩放取整后有余数 | 归入 `other`，`Math.max(0, ...)` 兜底 | 条形图右侧留白 |
| 某分类为 0 | `filter(x => x.tokens > 0)` | 图例里出现 `0%` 的项 |
| 流式期间频繁重算 | `createMemo` 只依赖 4 个标量 | 每帧遍历全部消息，长会话直接卡死 |
| 工具还在 pending（input 未解析） | 用 `state.raw.length` | 访问 `state.output` 会 undefined |
| 数字本地化 | 全部走 `toLocaleString(locale)` | 中文环境下没有千分位 |

---

## 7. 移植到你自己的项目

### 7.1 完整可复制实现

两个文件加起来不到 200 行，且**零框架依赖**，可以逐字复制：

```ts
// context-breakdown.ts
export type BreakdownKey = "system" | "user" | "assistant" | "tool" | "other"
export type BreakdownSegment = { key: BreakdownKey; tokens: number; width: number; percent: number }

const estimateTokens = (chars: number) => Math.ceil(chars / 4)
const toPercent = (t: number, input: number) => (t / input) * 100
const toPercentLabel = (t: number, input: number) => Math.round(toPercent(t, input) * 10) / 10

const charsFromUserPart = (part: Part) => {
  if (part.type === "text") return part.text.length
  if (part.type === "file") return part.source?.text.value.length ?? 0
  return 0
}

const charsFromAssistantPart = (part: Part) => {
  if (part.type === "text" || part.type === "reasoning") return { assistant: part.text.length, tool: 0 }
  if (part.type !== "tool") return { assistant: 0, tool: 0 }
  const input = Object.keys(part.state.input ?? {}).length * 16
  if (part.state.status === "pending")   return { assistant: 0, tool: input + (part.state.raw?.length ?? 0) }
  if (part.state.status === "completed") return { assistant: 0, tool: input + (part.state.output?.length ?? 0) }
  if (part.state.status === "error")     return { assistant: 0, tool: input + (part.state.error?.length ?? 0) }
  return { assistant: 0, tool: input }
}

const build = (t: Record<BreakdownKey, number>, input: number): BreakdownSegment[] =>
  (["system", "user", "assistant", "tool", "other"] as BreakdownKey[])
    .map((key) => ({ key, tokens: t[key] }))
    .filter((x) => x.tokens > 0)
    .map((x) => ({ key: x.key, tokens: x.tokens,
                   width: toPercent(x.tokens, input), percent: toPercentLabel(x.tokens, input) }))

export function estimateContextBreakdown(args: {
  messages: Message[]
  parts: Record<string, Part[] | undefined>
  input: number
  systemPrompt?: string
}): BreakdownSegment[] {
  if (!args.input) return []
  const counts = args.messages.reduce(
    (acc, msg) => {
      const parts = args.parts[msg.id] ?? []
      if (msg.role === "user")
        return { ...acc, user: acc.user + parts.reduce((s, p) => s + charsFromUserPart(p), 0) }
      if (msg.role !== "assistant") return acc
      const a = parts.reduce((s, p) => {
        const n = charsFromAssistantPart(p)
        return { assistant: s.assistant + n.assistant, tool: s.tool + n.tool }
      }, { assistant: 0, tool: 0 })
      return { ...acc, assistant: acc.assistant + a.assistant, tool: acc.tool + a.tool }
    },
    { system: args.systemPrompt?.length ?? 0, user: 0, assistant: 0, tool: 0 },
  )
  const tokens = {
    system: estimateTokens(counts.system), user: estimateTokens(counts.user),
    assistant: estimateTokens(counts.assistant), tool: estimateTokens(counts.tool),
  }
  const estimated = tokens.system + tokens.user + tokens.assistant + tokens.tool
  if (estimated <= args.input) return build({ ...tokens, other: args.input - estimated }, args.input)
  const scale = args.input / estimated
  const scaled = {
    system: Math.floor(tokens.system * scale), user: Math.floor(tokens.user * scale),
    assistant: Math.floor(tokens.assistant * scale), tool: Math.floor(tokens.tool * scale),
  }
  const total = scaled.system + scaled.user + scaled.assistant + scaled.tool
  return build({ ...scaled, other: Math.max(0, args.input - total) }, args.input)
}
```

React 组件：

```tsx
const COLOR: Record<BreakdownKey, string> = {
  system: "var(--syntax-info)",
  user: "var(--syntax-success)",
  assistant: "var(--syntax-property)",
  tool: "var(--syntax-warning)",
  other: "var(--syntax-comment)",
}

export function ContextBreakdownBar({ segments }: { segments: BreakdownSegment[] }) {
  return (
    <>
      <div style={{ display: "flex", height: 8, borderRadius: 4, overflow: "hidden" }}
           role="img"
           aria-label={segments.map((s) => `${s.key} ${s.percent}%`).join(", ")}>
        {segments.map((s) => (
          <div key={s.key} style={{ width: `${s.width}%`, backgroundColor: COLOR[s.key] }} />
        ))}
      </div>
      <div style={{ display: "flex", flexWrap: "wrap", gap: "4px 12px", marginTop: 8 }}>
        {segments.map((s) => (
          <span key={s.key} style={{ display: "inline-flex", alignItems: "center", gap: 6 }}>
            <span style={{ width: 8, height: 8, borderRadius: 2, backgroundColor: COLOR[s.key] }} />
            <span>{LABEL[s.key]}</span>
            <span style={{ opacity: 0.6 }}>{s.percent.toLocaleString()}%</span>
          </span>
        ))}
      </div>
    </>
  )
}
```

**`role="img"` + `aria-label`** 让屏幕阅读器能读出整条的构成，而不是读到一串空 div。

### 7.2 落地步骤

1. 复制 `estimateContextBreakdown`（§7.1）。
2. 实现 `getSessionContext`：从后往前找最后一条有 token 的 assistant 消息（§4.1）。
3. 实现格式化器：`—` 兜底 + `toLocaleString(locale)`（§4.4）。
4. 渲染堆叠条 + 图例 + 统计表。配色复用你的语法高亮调色板。
5. **`useMemo` 只依赖 4 个标量**，加 lint 禁用注释说明这是刻意的。
6. 可选：`usage > 80%` 时给条形图加警示态，提示用户可以手动压缩。

### 7.3 你需要自己提供的依赖

| 依赖 | 用途 | 备注 |
| --- | --- | --- |
| 模型目录（context limit） | 算占用百分比 | 缺失时降级为「不显示百分比」，不要猜 |
| system prompt 文本 | 估算 system 段 | 拿不到就传 `undefined`，差额自动进 `other` |
| 语法高亮调色板 | 配色 | 或自定义 5 个色，注意深浅色对比度 |

---

## 8. 验收清单

- [ ] **U1** 会话有 5 条 assistant 消息 → 用量取最后一条的，不是累加
- [ ] **U1** 最后一条 assistant 失败（token 全 0）→ 往前取上一条有效的
- [ ] **U1** 没有任何有效 assistant 消息 → 返回 `undefined`，面板显示空态
- [ ] **U2** 各段 `width` 之和 === 100（浮点误差内）
- [ ] **U2** 各段 `tokens` 之和 === provider 报告的 `input`
- [ ] **U3** 估算 < 实际 → `other` = 差额，其余项保持估算值
- [ ] **U3** 估算 > 实际（灌入大量中文）→ 各项按比例缩小，总和仍等于 `input`
- [ ] **U4** 某分类为 0 → 不出现在条形图和图例里
- [ ] **U5** `limit` 缺失 → 占用率显示 `—`，不是 `NaN%` 或 `Infinity%`
- [ ] **U5** 数字带千分位，且随语言环境变化
- [ ] **零输入** `input === 0` → 返回空数组，不抛错
- [ ] **pending 工具** 工具还在 pending → 用 `state.raw.length`，不访问 `state.output`
- [ ] **性能** 500 条消息的会话流式期间 → breakdown 不在每帧重算（打点验证）
- [ ] **无障碍** 堆叠条有 `role="img"` 和描述性 `aria-label`
- [ ] **主题** 切换深浅色 → 配色跟随，对比度仍然可读
