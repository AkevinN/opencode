# 10 · Part 渲染与过程视觉：打字节奏、工具卡片、状态指示

源码：`packages/session-ui/src/components/message-part.tsx`（2642 行）、`basic-tool.tsx`、`tool-status-title.tsx`、`tool-count-summary.tsx`、`part-default-open.ts`、`packages/session-ui/src/v2/components/session-progress-indicator-v2.{tsx,css}`

---

## 1. 解决什么问题

「过程可见」不是把文本打出来就行。真实需求是：

1. **文本要以人类可读的速度出现**，但不能落后太多。provider 有时一次吐 2000 字符（重试后的补发、缓存命中），逐字打要打半分钟。
2. **工具调用要一眼看懂在干什么**，且状态变化要平滑（`Reading` → `Read` 不能突兀跳字）。
3. **默认展开策略要合理**：几十个 `read` 全展开等于刷屏，一个 `edit` 全折叠等于看不到改了什么。
4. **忙碌状态要有明确指示**，且要尊重 `prefers-reduced-motion`。

---

## 2. 概念模型与不变量

1. **V1** 打字是**追赶式**（catch-up），不是固定速率：落后越多，每帧吐得越多；落后不足阈值直接全量。
2. **V2** 打字步进落在**词/标点边界**上，不切断单词。
3. **V3** 工具卡片的可交互性由状态决定：pending/running 默认不可展开。
4. **V4** 标题在 active/done 之间切换时，提取**共同前缀**只动后缀（`Read` + `ing` ↔ `Read` + ``）。
5. **V5** 默认展开策略按工具类型分级，且「只删文件」的编辑不自动展开。
6. **V6** 重内容延迟挂载，且**从底部优先**（视口在底部）。
7. **V7** 所有动画尊重 `prefers-reduced-motion`。

---

## 3. Part → 组件的分派

### 3.1 注册表

```ts
export const PART_MAPPING: Record<string, PartComponent | undefined> = {}
```
> `message-part.tsx:250`

已注册的 part 渲染器（`message-part.tsx:1534, 1649, 1654, 1759`）：

| part.type | 组件 | 渲染成 |
| --- | --- | --- |
| `text` | `TextPartDisplay` | `PacedMarkdown` + 复制按钮 + 元信息（agent · 模型 · 耗时） |
| `reasoning` | `ReasoningPartDisplay` | 仅 `PacedMarkdown`，无操作区 |
| `tool` | `ToolPartDisplay` | 分派到工具渲染器或错误卡片 |
| `compaction` | `CompactionPartDisplay` | 一条带文案的分隔线 |

其余 part 类型（`patch` / `step-start` / `step-finish`）在客户端 store 层就被 `SKIP_PARTS` 过滤掉了（第 08 章 §4.1），不会到达这里。

### 3.2 text part 的元信息

```ts
const meta = createMemo(() => {
  if (props.message.role !== "assistant") return ""
  const agent = (props.message as AssistantMessage).agent
  const items = [
    agent ? agent[0]?.toUpperCase() + agent.slice(1) : "",
    model(),            // provider 目录里的模型显示名，回退到 modelID
    duration(),         // < 60s 显示秒；否则 "N 分 M 秒"
    interrupted() ? i18n.t("ui.message.interrupted") : "",
  ]
  return items.filter((x) => !!x).join(" · ")     // 用 · 连接
})
```
> `message-part.tsx:1692-1702`

显示成 `Build · Claude Opus 4 · 42s`。**只在最后一个文本 part 上显示**（`isLastTextPart`），避免每段文本后面都跟一行元信息。

复制按钮的 2 秒反馈：

```ts
const handleCopy = async () => {
  const content = text()
  if (!content) return
  if (await writeClipboard(content)) {
    setCopied(true)
    setTimeout(() => setCopied(false), 2000)
  }
}
```
> `message-part.tsx:1720-1728`

### 3.3 文本从哪读

```ts
const text = () => readPartText(data.store.part_text_accum_delta, part())
```
> `message-part.tsx:1707`

**优先读 delta 累积值，没有则读 part 上的权威值**——这就是第 08 章 §4.3 双写机制在渲染侧的落点。

---

## 4. 工具卡片

### 4.1 完整的工具 → 图标/标题/副标题映射表

`getToolInfo(tool, input, metadata)`（`message-part.tsx:469-566`）是唯一的真相来源，被卡片、时间线摘要、TUI 共用。英文文案取自 `packages/ui/src/i18n/en.ts:151-168`。

| tool | icon | title | subtitle |
| --- | --- | --- | --- |
| `read` | `glasses` | `Read` | `getFilename(input.filePath)` |
| `list` | `bullet-list` | `List` | `getFilename(input.path)` |
| `glob` | `magnifying-glass-menu` | `Glob` | `input.pattern` |
| `grep` | `magnifying-glass-menu` | `Grep` | `input.pattern` |
| `webfetch` | `window-cursor` | `Webfetch` | `input.url` |
| `websearch` | `window-cursor` | `{provider} Web Search`（provider 为 `parallel`→`Parallel`、`exa`→`Exa`，否则 `Web Search`） | `input.query` |
| `task` | `task` | `{Type} Agent`（`input.subagent_type` 首字母大写），无则 `Agent` | `input.description` |
| `bash` / `shell` | `console` | `Shell` | `input.command` |
| `edit` | `code-lines` | `Edit` | `getFilename(input.filePath)` |
| `write` | `code-lines` | `Write` | `getFilename(input.filePath)` |
| `patch` / `apply_patch` | `code-lines` | `Patch` | `N file` / `N files` |
| `todowrite` | `checklist` | `To-dos` | — |
| `question` | `bubble-5` | `Questions` | — |
| `skill` | `brain` | `input.name` 或 `Skill` | — |
| **其它（含 MCP）** | `mcp` | 工具名原样 | — |

兜底渲染器：

```ts
export function GenericTool(props: { tool: string; status?: string; hideDetails?: boolean; input?: Record<string, unknown> }) {
  const i18n = useI18n()
  return (
    <BasicTool
      icon="mcp"
      status={props.status}
      trigger={{ title: i18n.t("ui.basicTool.called", { tool: props.tool }),   // "Called `{{tool}}`"
                 subtitle: label(props.input), args: args(props.input) }}
      hideDetails={props.hideDetails}
    />
  )
}
```
> `basic-tool.tsx:323-342`

别名解析：

```ts
export function getTool(name: string) {
  return state[name === "apply_patch" ? "patch" : name === "bash" ? "shell" : name]?.render
}
```
> `message-part.tsx:1490-1492`

**副标题的选择原则值得单独说**：每个工具都选了「一眼能认出这次调用干了什么」的那个参数——read 是文件名（不是全路径）、grep 是 pattern（不是路径）、shell 是完整命令、task 是任务描述。**不要显示整个 input JSON**，那等于没显示。

### 4.2 卡片骨架 `BasicTool`

```ts
export type TriggerTitle = {
  title: string
  titleClass?: string
  subtitle?: string
  subtitleClass?: string
  args?: string[]
  argsClass?: string
  action?: JSX.Element
}

export interface BasicToolProps {
  icon: IconProps["name"]
  trigger: TriggerTitle | JSX.Element | ((open: Accessor<boolean>) => JSX.Element)
  children?: JSX.Element
  status?: string
  hideDetails?: boolean
  defaultOpen?: boolean
  open?: boolean                     // 受控
  onOpenChange?: (open: boolean) => void
  forceOpen?: boolean                // 强制展开且不可关
  allowOpenWhilePending?: boolean
  defer?: boolean                    // 延迟挂载内容
  locked?: boolean                   // 不允许关闭
  animated?: boolean                 // 高度弹簧动画
  onSubtitleClick?: () => void
  onTriggerClick?: JSX.EventHandlerUnion<HTMLElement, MouseEvent>
  onTriggerKeyDown?: JSX.EventHandlerUnion<HTMLElement, KeyboardEvent>
  triggerHref?: string
  triggerAsLink?: boolean
  clickable?: boolean
}
```
> `basic-tool.tsx:9-45`

**V3 的实现**：

```ts
const pending = () => props.status === "pending" || props.status === "running"

const handleOpenChange = (value: boolean) => {
  if (pending() && !props.allowOpenWhilePending) return       // 跑着的时候不给展开
  if (props.locked && !value) return                           // locked 时不给关
  setOpen(value)
}
```
> `basic-tool.tsx:92, 175-179`

工具还在跑的时候内容是不完整的，展开只会看到半截然后跳变。等它 completed 再允许展开。

**高度动画**用 motion 的弹簧：

```ts
const SPRING = { type: "spring" as const, visualDuration: 0.35, bounce: 0 }
```
> `basic-tool.tsx:47`

```ts
if (isOpen) {
  contentRef.style.overflow = "hidden"
  heightAnim = animate(contentRef, { height: "auto" }, SPRING)
  void heightAnim.finished.then(() => {
    if (!contentRef || !open()) return
    contentRef.style.overflow = "visible"      // ← 动画结束后放开，否则内部的浮层被裁
    contentRef.style.height = "auto"
  })
} else {
  contentRef.style.overflow = "hidden"
  heightAnim = animate(contentRef, { height: "0px" }, SPRING)
}
```
> `basic-tool.tsx:148-170`

`bounce: 0` + `visualDuration: 0.35`：无回弹，350ms 观感。**动画结束把 `overflow` 放回 `visible`** 很关键——不放的话内容里的下拉菜单、tooltip 会被裁掉。

### 4.3 延迟挂载（V6）

```ts
const deferredMounts: Array<{ active: boolean; fn: () => void }> = []
let deferredFrame: number | undefined

function flushDeferredMounts() {
  while (deferredMounts.length > 0) {
    // Timeline tools are mounted top-to-bottom, but the viewport starts at the latest turn.
    // Pop from the end so heavy default-open bodies near the bottom become interactive first.
    const item = deferredMounts.pop()!                      // ← LIFO
    if (item.active) {
      deferredFrame = deferredMounts.length > 0 ? requestAnimationFrame(flushDeferredMounts) : undefined
      item.fn()                                              // 每帧只挂一个
      return
    }
  }
  deferredFrame = undefined
}
```
> `basic-tool.tsx:48-79`

三个设计点：
- **LIFO**：时间线自上而下挂载，但视口在底部。从后往前挂，用户看得见的先可交互。
- **每帧一个**：一次挂 20 个大 diff 会冻结一帧。
- **`active` 标志**：组件卸载时置 false，队列里的任务变成空转（不用从数组里删）。

### 4.4 状态标题的共同前缀交叉淡入（V4）

```ts
function common(active: string, done: string) {
  const a = Array.from(active)
  const b = Array.from(done)
  let i = 0
  while (i < a.length && i < b.length && a[i] === b[i]) i++
  return { prefix: a.slice(0, i).join(""), active: a.slice(i).join(""), done: b.slice(i).join("") }
}
```
> `tool-status-title.tsx:5-15`

`common("Reading", "Read")` → `{ prefix: "Read", active: "ing", done: "" }`。

启用 suffix 模式的条件：

```ts
const suffix = createMemo(
  () => (props.split ?? true) && split().prefix.length >= 2 && split().active.length > 0 && split().done.length > 0,
)
```
> `tool-status-title.tsx:30-32`

**要求 `done` 也非空**——所以 `Reading`/`Read` 这对（done 后缀为空）实际上走的是 `swap` 模式（整体交叉淡入），而不是 suffix 模式。suffix 模式适用于像 `Searching`/`Searched` 这种前后缀都非空的组合。

宽度过渡：

```ts
const animate = () => {
  const first = contentWidth(widthRef)          // 变化前的宽度
  const next = props.active
  finish()
  setState("active", next)
  if (!first) return
  setState("animating", true)
  setState("width", first)                       // 先锁死旧宽度
  frame = requestAnimationFrame(() => {
    frame = undefined
    const last = contentWidth(next ? activeRef : doneRef)   // 新内容的宽度
    if (!last) { finish(); return }
    if (first !== last) setState("width", last)             // 触发 CSS width 过渡
    finishTimer = setTimeout(finish, 600)                    // 600ms 后解除锁定
  })
}
```
> `tool-status-title.tsx:60-79`

**先锁旧宽度 → 下一帧设新宽度 → 600ms 后解锁回 `auto`。** 这样文字长度变化时旁边的内容平滑推移而不是瞬移。

DOM 上暴露的状态属性（CSS 钩子）：

```tsx
<span
  data-component="tool-status-title"
  data-active={active() ? "true" : "false"}
  data-ready={animating() ? "true" : "false"}
  data-mode={suffix() ? "suffix" : "swap"}
  aria-label={active() ? props.activeText : props.doneText}
>
```
> `tool-status-title.tsx:87-95`

**`aria-label` 始终给出当前语义文本**——屏幕阅读器不会读到被拆开的 prefix/suffix 碎片。

### 4.5 上下文组的计数摘要

```ts
function contextToolSummary(parts: ToolPart[]) {
  const read = parts.filter((part) => part.tool === "read").length
  const search = parts.filter((part) => part.tool === "glob" || part.tool === "grep").length
  const list = parts.filter((part) => part.tool === "list").length
  return { read, search, list }
}
```
> `message-part.tsx:899-904`

渲染成 `Exploring   3 reads, 2 searches`：

```tsx
<ToolStatusTitle
  active={pending()}
  activeText={i18n.t("ui.sessionTurn.status.gatheringContext")}   // "Exploring"
  doneText={i18n.t("ui.sessionTurn.status.gatheredContext")}      // "Explored"
  split={false}
/>
<AnimatedCountList
  items={[
    { key: "ui.messagePart.context.read",   count: summary().read },
    { key: "ui.messagePart.context.search", count: summary().search },
    { key: "ui.messagePart.context.list",   count: summary().list },
  ]}
  fallback=""
/>
```
> `message-part.tsx:1078-1108`

复数文案（`packages/ui/src/i18n/en.ts:105-110`）：

```
"ui.messagePart.context.read.one":   "{{count}} read"
"ui.messagePart.context.read.other": "{{count}} reads"
"ui.messagePart.context.search.one": "{{count}} search"
"ui.messagePart.context.search.other": "{{count}} searches"
"ui.messagePart.context.list.one":   "{{count}} list"
"ui.messagePart.context.list.other": "{{count}} lists"
```

`AnimatedCountList` 的关键设计：**所有 item 始终在 DOM 里**，靠 `data-active` 控制显隐：

```tsx
<Index each={props.items}>
  {(item, index) => {
    const active = createMemo(() => item().count > 0)
    const hasPrev = createMemo(() => {
      for (let i = index - 1; i >= 0; i--) if (props.items[i].count > 0) return true
      return false
    })
    return (
      <>
        <span data-slot="tool-count-summary-prefix" data-active={active() && hasPrev() ? "true" : "false"}>,</span>
        <span data-slot="tool-count-summary-item" data-active={active() ? "true" : "false"}>
          <span data-slot="tool-count-summary-item-inner">
            <AnimatedCountLabel plural={item().key} count={Math.max(0, Math.round(item().count))} />
          </span>
        </span>
      </>
    )
  }}
</Index>
```
> `tool-count-summary.tsx:21-44`

**逗号也是独立的 `data-active` 元素**，只有「自己有值且前面有值」时才显示。这样从 `3 reads` 变成 `3 reads, 1 search` 时逗号是淡入的，而不是突然出现。`Index` 而非 `For`（React 里就是按 index 渲染固定长度列表）保证 DOM 节点不重建，CSS 过渡才有效。

### 4.6 默认展开策略（V5）

```ts
function deletionOnly(part: ToolPart) {
  if (!("metadata" in part.state)) return false
  const metadata = part.state.metadata
  if (!metadata) return false
  const files = metadata.files
  if (Array.isArray(files) && files.length > 0) {
    return files.every((file) => !!file && typeof file === "object" && "type" in file && file.type === "delete")
  }
  const filediff = metadata.filediff
  if (!filediff || typeof filediff !== "object") return false
  if (!("additions" in filediff) || !("deletions" in filediff)) return false
  return filediff.additions === 0 && typeof filediff.deletions === "number" && filediff.deletions > 0
}

export function partDefaultOpen(part: PartType, shell = false, edit = false) {
  if (part.type !== "tool") return                                  // undefined = 用组件自己的默认
  if (part.tool === "bash" || part.tool === "shell") return shell
  if (part.tool === "edit" || part.tool === "write" || part.tool === "patch" || part.tool === "apply_patch") {
    if (!edit) return false
    return !deletionOnly(part)                                      // 纯删除不自动展开
  }
}
```
> `part-default-open.ts:3-26`

三档：

| 工具 | 默认 | 依据 |
| --- | --- | --- |
| `bash` / `shell` | 用户设置 `shellToolDefaultOpen` | 输出可能很长 |
| `edit` / `write` / `patch` / `apply_patch` | 用户设置 `editToolDefaultOpen`，但**纯删除除外** | 改动是用户最关心的；但「删了 50 行」展开出来是 50 行红色，没有信息量 |
| 其余（read/grep/list/...） | 返回 `undefined`（组件自己决定，实际是折叠） | 这类调用量最大，全展开等于刷屏 |

`deletionOnly` 的双路径判定：多文件 patch 看 `metadata.files[].type` 是否全是 `delete`；单文件看 `metadata.filediff.additions === 0 && deletions > 0`。

### 4.7 部分工具不渲染成卡片

```ts
PART_MAPPING["tool"] = function ToolPartDisplay(props) {
  const part = () => props.part as ToolPart
  if (part().tool === "todowrite") return null                     // 由 todo dock 呈现

  const hideQuestion = createMemo(
    () => part().tool === "question" && (part().state.status === "pending" || part().state.status === "running"),
  )                                                                 // 由 question dock 呈现
  ...
}
```
> `message-part.tsx:1534-1541`

`question` 只在**已回答之后**才在时间线里显示成卡片；进行中时由底部的提问 dock 承载交互。

错误分支单独走 `ToolErrorCard`，并且对「用户主动忽略提问」做了特判：

```tsx
<Match when={part().state.status === "error" && (part().state as any).error}>
  {(error) => {
    const cleaned = error().replace("Error: ", "")
    if (part().tool === "question" && cleaned.includes("dismissed this question")) {
      return (
        <div style="width: 100%; display: flex; justify-content: flex-end;">
          <span class="text-13-regular text-text-weak cursor-default">
            {i18n.t("ui.messagePart.questions.dismissed")}
          </span>
        </div>
      )
    }
    return <ToolErrorCard tool={part().tool} error={error()} ... />
  }}
</Match>
```
> `message-part.tsx:1574-1607`

**用户忽略提问不是「错误」**，渲染成一行右对齐的灰字比一张红色错误卡片合理得多。

---

## 5. 追赶式打字（V1 / V2）

这是本章最值得抄的一段代码。

```ts
const TEXT_RENDER_PACE_MS = 24
const TEXT_RENDER_IMMEDIATE = 512
const TEXT_RENDER_SNAP = /[\s.,!?;:)\]]/

function step(size: number) {
  if (size <= 12) return 2
  if (size <= 48) return 4
  if (size <= 96) return 8
  return Math.min(256, Math.ceil(size / 4))
}

function next(text: string, start: number) {
  const end = Math.min(text.length, start + step(text.length - start))
  const max = Math.min(text.length, end + 8)
  for (let i = end; i < max; i++) {
    if (TEXT_RENDER_SNAP.test(text[i] ?? "")) return i + 1     // 向后最多找 8 个字符对齐到边界
  }
  return end
}
```
> `message-part.tsx:252-270`

**`step()` 是分段的追赶曲线**：

| 落后字符数 | 每帧吐出 |
| --- | --- |
| ≤ 12 | 2 |
| ≤ 48 | 4 |
| ≤ 96 | 8 |
| > 96 | `min(256, ceil(size / 4))` |

落后 400 字符时每帧吐 100，8 帧（约 200ms）追平。**不会出现「打字机打了半分钟」的荒谬体验。**

`next()` 在算出的位置**向后最多 8 个字符**找空白或标点，对齐到词边界（V2）。不然会看到单词被切成 `implemen` → `implementa`。

**完整的节奏控制器**：

```ts
function createPacedValue(getValue: () => string, live?: () => boolean) {
  const [value, setValue] = createSignal(getValue())
  let shown = getValue()
  let timeout: ReturnType<typeof setTimeout> | undefined

  const clear = () => { if (!timeout) return; clearTimeout(timeout); timeout = undefined }
  const sync = (text: string) => { shown = text; setValue(text) }

  const run = () => {
    timeout = undefined
    const text = getValue()
    if (!live?.()) { sync(text); return }                                 // 已结束 → 全量
    if (!text.startsWith(shown) || text.length <= shown.length) { sync(text); return }  // 非追加 → 全量
    if (text.length - shown.length <= TEXT_RENDER_IMMEDIATE) { sync(text); return }     // 落后 < 512 → 全量
    const end = next(text, shown.length)
    sync(text.slice(0, end))
    if (end < text.length) timeout = setTimeout(run, TEXT_RENDER_PACE_MS)
  }

  createEffect(() => {
    const text = getValue()
    if (!live?.()) { clear(); sync(text); return }
    if (!text.startsWith(shown) || text.length < shown.length) { clear(); sync(text); return }
    if (text.length - shown.length <= TEXT_RENDER_IMMEDIATE) { clear(); sync(text); return }
    if (text.length === shown.length || timeout) return
    timeout = setTimeout(run, TEXT_RENDER_PACE_MS)
  })

  onCleanup(clear)
  return value
}
```
> `message-part.tsx:272-331`

**这里最反直觉、也最重要的一点：落后不足 512 字符时直接全量显示，不做任何节奏控制。**

也就是说，**正常流式过程中根本不走打字机**——provider 的 delta 本身就是逐字来的，前端 16ms 合帧后天然是逐字效果。节奏控制只在「一次性收到一大坨」时才启动：重连后的补发、缓存命中的瞬时返回、切换会话时的历史加载。

`!text.startsWith(shown)` 的分支处理内容被替换（不是追加）的情况——比如权威值覆盖了累积值且内容不同，此时立刻全量同步，不能继续增量。

---

## 6. 忙碌指示

### 6.1 5×5 点阵

```ts
const grid = 5
const dot = 2
const gap = 1
const origin = 1.5
const dots = Array.from({ length: grid * grid }, (_, index) => ({
  index,
  x: origin + (index % grid) * (dot + gap),
  y: origin + Math.floor(index / grid) * (dot + gap),
}))

export function SessionProgressIndicatorV2(props: ComponentProps<"svg">) {
  const [local, rest] = splitProps(props, ["class", "classList", "width", "height"])
  return (
    <svg {...rest} width={local.width ?? 16} height={local.height ?? 16} viewBox="0 0 16 16"
         fill="none" data-component="session-progress-indicator-v2" aria-hidden={rest["aria-hidden"] ?? "true"}>
      <For each={dots}>{(cell) => <rect data-dot={cell.index} x={cell.x} y={cell.y} width={dot} height={dot} />}</For>
    </svg>
  )
}
```
> `session-progress-indicator-v2.tsx:4-31`

25 个 2×2 的方块，间距 1，起点 1.5，装在 16×16 的 viewBox 里。默认 16px。`aria-hidden="true"`（纯装饰，语义由旁边的文字承载）。

### 6.2 动画规格

```css
[data-component="session-progress-indicator-v2"] {
  --_duration: 1200ms;
  display: block;
  flex-shrink: 0;
  color: var(--v2-icon-icon-muted, #808080);
}

[data-component="session-progress-indicator-v2"] [data-dot] {
  fill: currentColor;
  opacity: 0.2;                    /* 基线透明度 */
  animation-duration: var(--_duration);
  animation-timing-function: ease-out;
  animation-iteration-count: infinite;
  animation-fill-mode: both;
}
```
> `session-progress-indicator-v2.css:1-15`

每个点有**独立的 keyframes**（`session-progress-indicator-v2-dot-{0..24}`），关键帧固定在 8 等分（0 / 12.5% / 25% / 37.5% / 50% / 62.5% / 75% / 87.5% / 100%），透明度在 `0.2 / 0.5 / 0.75 / 1` 四档之间取值。相位随位置递进，形成**对角波纹**。

例：`dot-12`（正中心）全程 `opacity: 1`；`dot-20`（左下角）在 12.5% 达到峰值 1，`dot-0`（左上角）在 37.5% 才到峰值。

**V7 的实现**：

```css
@media (prefers-reduced-motion: reduce) {
  [data-component="session-progress-indicator-v2"] [data-dot] { animation: none; }
  [data-component="session-progress-indicator-v2"] [data-dot="12"] { opacity: 1; }
}
```
> `session-progress-indicator-v2.css:867-875`

关掉动画，但保留**中心点高亮**——静态的"有东西在运行"提示，不是一片死灰。

### 6.3 Thinking 行的文案

```tsx
<div data-slot="session-turn-thinking">
  <TextShimmer text={i18n.t("ui.sessionTurn.status.thinking")} />     // "Thinking"
  ...
</div>
```
> `packages/session-ui/src/components/session-turn.tsx:423-424`

时间线的 `Thinking` 行可以带一个从 reasoning 内容里提取的标题（第 09 章 §4.2 的 `reasoningHeading`），提取规则按优先级尝试四种形式：

```ts
function reasoningHeading(text: string) {
  const markdown = text.replace(/\r\n?/g, "\n")
  const html = markdown.match(/<h[1-6][^>]*>([\s\S]*?)<\/h[1-6]>/i)          // ① HTML 标题
  if (html?.[1]) { const v = cleanHeading(html[1].replace(/<[^>]+>/g, " ")); if (v) return v }
  const atx = markdown.match(/^\s{0,3}#{1,6}[ \t]+(.+?)(?:[ \t]+#+[ \t]*)?$/m)  // ② ATX 标题 (# xxx)
  if (atx?.[1]) { const v = cleanHeading(atx[1]); if (v) return v }
  const setext = markdown.match(/^([^\n]+)\n(?:=+|-+)\s*$/m)                 // ③ Setext 标题
  if (setext?.[1]) { const v = cleanHeading(setext[1]); if (v) return v }
  const strong = markdown.match(/^\s*(?:\*\*|__)(.+?)(?:\*\*|__)\s*$/m)      // ④ 独占一行的加粗
  if (strong?.[1]) { const v = cleanHeading(strong[1]); if (v) return v }
}

function cleanHeading(value: string) {
  return value
    .replace(/`([^`]+)`/g, "$1")                    // 去行内代码标记
    .replace(/\[([^\]]+)\]\([^)]+\)/g, "$1")        // 链接只留文字
    .replace(/[*_~]+/g, "")                          // 去强调标记
    .trim()
}
```
> `packages/app/src/pages/session/timeline/rows.ts:234-267`

**这是一个很好用的技巧**：模型的 reasoning 通常自带小标题（`## Checking the auth flow`），把它提出来显示成 `Thinking — Checking the auth flow`，用户就知道模型在想什么，而不用展开整段思考。

> i18n 里还定义了 `delegating` / `planning` / `searchingCodebase` / `searchingWeb` / `makingEdits` / `runningCommands` / `gatheringThoughts` / `consideringNextSteps` 等状态文案（`packages/ui/src/i18n/en.ts:85-96`），但当前分支代码中**只有 `thinking` / `gatheringContext` / `gatheredContext` 被实际使用**。其余是预留。

---

## 7. 状态 → 视觉对照表

| 状态 | 视觉 | 数据来源 |
| --- | --- | --- |
| 文本流式中 | markdown 逐块出现；落后 > 512 字符时按 24ms 节奏追赶 | `message.time.completed === undefined` |
| 文本完成 | 全量渲染 + 复制按钮 + `Agent · Model · 42s` | `time.completed` 有值 |
| 思考中（无内容） | `Thinking` + shimmer + 5×5 点阵 | `status === "busy"` 且无可渲染 part |
| 思考中（有内容） | reasoning 内容 + 从中提取的标题 | reasoning part 有文本 |
| 工具 pending/running | 卡片显示但**不可展开**；标题走 active 文案 | `state.status` |
| 工具 completed | 可展开；标题交叉淡入到 done 文案 | `state.status === "completed"` |
| 工具 error | `ToolErrorCard`（`Failed` + 可复制错误） | `state.status === "error"` |
| 连续只读工具 | 折叠成 `Exploring / Explored` + `3 reads, 2 searches` | `groupParts` 的 context 组 |
| 重试中 | `Retry` 行，带 attempt / message / 倒计时 | `SessionStatus.type === "retry"` |
| 被中断 | `TurnDivider(interrupted)` 分隔线 | `error.name === "MessageAbortedError"` |
| 压缩发生 | `TurnDivider(compaction)` 分隔线 | user part 中有 `type === "compaction"` |
| 提问待回答 | 时间线**不显示**，由 question dock 承载 | `tool === "question"` 且 pending/running |
| 提问被忽略 | 右对齐灰字 | error 含 `"dismissed this question"` |
| 用户已滚开且距底 > max(400, 视口高) | 显示「跳到最新」按钮 | `ui.scroll.jump` |

关键动画数值：

| 元素 | 数值 | 位置 |
| --- | --- | --- |
| 打字节奏间隔 | `24ms` | `message-part.tsx:252` |
| 打字全量阈值 | `512` 字符 | `message-part.tsx:253` |
| 卡片展开/收起 | spring `visualDuration: 0.35, bounce: 0` | `basic-tool.tsx:47` |
| 标题宽度过渡 | `600ms` | `tool-status-title.tsx:77` |
| 进度点阵一轮 | `1200ms` ease-out infinite | `session-progress-indicator-v2.css:2` |
| 复制反馈 | `2000ms` | `message-part.tsx:1726` |
| 延迟挂载 | 每 `requestAnimationFrame` 一个 | `basic-tool.tsx:54-62` |

---

## 8. 边界情况与陷阱

| 场景 | opencode 怎么处理 | 你自己实现时必须处理 |
| --- | --- | --- |
| 一次收到 5000 字符 | `step()` 追赶曲线，8 帧内追平 | 固定速率打字机会打几十秒 |
| 正常流式（每次几字符） | 落后 < 512 → 直接全量，不启动节奏 | 无条件走打字机会让本来就慢的输出更慢 |
| 内容被替换而非追加 | `!text.startsWith(shown)` → 立刻全量 | 继续增量会拼出乱码 |
| 单词被切断 | `next()` 向后最多 8 字符对齐到 `[\s.,!?;:)\]]` | 会看到 `implemen` → `implementa` |
| 工具跑着时用户点展开 | `handleOpenChange` 直接 return | 展开看到半截内容，完成后跳变 |
| 展开动画结束后内部浮层被裁 | 动画完成回调里把 `overflow` 设回 `visible` | 下拉菜单/tooltip 会被裁掉 |
| 同时挂载 20 个默认展开的大 diff | 延迟队列每帧一个，LIFO | 一次挂载冻结数百毫秒 |
| 组件在延迟队列里时被卸载 | `active = false`，任务变空转 | 直接调用会操作已卸载的组件 |
| `Reading`/`Read` 这类 done 后缀为空 | `suffix()` 要求 done 非空 → 走 swap 模式 | suffix 模式下 done 为空会让宽度算成 0 |
| 屏幕阅读器读到拆分的前后缀 | `aria-label` 给完整文本 | 会读出 "Read ing" 这种碎片 |
| 计数从 `3 reads` 变 `3 reads, 1 search` | 逗号是独立的 `data-active` 元素 | 逗号突然出现，无过渡 |
| 计数列表节点重建 | 用 `Index`（React 里按固定 index 渲染） | 节点重建 = CSS 过渡失效 |
| 只删文件的 patch 自动展开 | `deletionOnly` 检测后不展开 | 展开出来是一片红，没有信息量 |
| `todowrite` / 进行中的 `question` | 返回 `null`，由 dock 承载 | 时间线里出现一张没法交互的卡片 |
| 用户忽略提问 | 特判成右对齐灰字 | 渲染成红色错误卡片，用户以为出错了 |
| `prefers-reduced-motion` | 关动画，中心点保持 `opacity: 1` | 全关 = 一片死灰，看不出在运行 |
| MCP 等未注册工具 | `GenericTool` + `Called \`{tool}\`` + `mcp` 图标 | 崩溃或显示空白卡片 |

---

## 9. 移植到你自己的项目（React 版）

### 9.1 追赶式打字 hook（可直接用）

```ts
const TEXT_RENDER_PACE_MS = 24
const TEXT_RENDER_IMMEDIATE = 512
const TEXT_RENDER_SNAP = /[\s.,!?;:)\]]/

function step(size: number) {
  if (size <= 12) return 2
  if (size <= 48) return 4
  if (size <= 96) return 8
  return Math.min(256, Math.ceil(size / 4))
}

function nextIndex(text: string, start: number) {
  const end = Math.min(text.length, start + step(text.length - start))
  const max = Math.min(text.length, end + 8)
  for (let i = end; i < max; i++) if (TEXT_RENDER_SNAP.test(text[i] ?? "")) return i + 1
  return end
}

export function usePacedText(text: string, live: boolean) {
  const [shown, setShown] = useState(text)
  const shownRef = useRef(text)
  const timer = useRef<ReturnType<typeof setTimeout> | undefined>(undefined)
  const textRef = useRef(text)
  textRef.current = text

  useEffect(() => {
    const clear = () => { if (timer.current) { clearTimeout(timer.current); timer.current = undefined } }
    const sync = (v: string) => { shownRef.current = v; setShown(v) }

    const run = () => {
      timer.current = undefined
      const t = textRef.current
      if (!live) return sync(t)
      if (!t.startsWith(shownRef.current) || t.length <= shownRef.current.length) return sync(t)
      if (t.length - shownRef.current.length <= TEXT_RENDER_IMMEDIATE) return sync(t)
      const end = nextIndex(t, shownRef.current.length)
      sync(t.slice(0, end))
      if (end < t.length) timer.current = setTimeout(run, TEXT_RENDER_PACE_MS)
    }

    if (!live) { clear(); sync(text); return }
    if (!text.startsWith(shownRef.current) || text.length < shownRef.current.length) { clear(); sync(text); return }
    if (text.length - shownRef.current.length <= TEXT_RENDER_IMMEDIATE) { clear(); sync(text); return }
    if (text.length === shownRef.current.length || timer.current) return
    timer.current = setTimeout(run, TEXT_RENDER_PACE_MS)
    return clear
  }, [text, live])

  return shown
}
```

### 9.2 工具信息映射（可直接用）

```ts
type ToolInfo = { icon: string; title: string; subtitle?: string }

const basename = (p?: string) => (p ? p.split(/[/\\]/).pop() ?? p : undefined)

export function getToolInfo(tool: string, input: any = {}, metadata: Record<string, unknown> = {}): ToolInfo {
  switch (tool) {
    case "read":     return { icon: "glasses", title: "Read", subtitle: basename(input.filePath) }
    case "list":     return { icon: "bullet-list", title: "List", subtitle: basename(input.path) }
    case "glob":     return { icon: "magnifying-glass-menu", title: "Glob", subtitle: input.pattern }
    case "grep":     return { icon: "magnifying-glass-menu", title: "Grep", subtitle: input.pattern }
    case "webfetch": return { icon: "window-cursor", title: "Webfetch", subtitle: input.url }
    case "websearch": {
      const p = metadata.provider
      const name = p === "parallel" ? "Parallel" : p === "exa" ? "Exa" : undefined
      return { icon: "window-cursor", title: name ? `${name} Web Search` : "Web Search", subtitle: input.query }
    }
    case "task": {
      const t = typeof input.subagent_type === "string" && input.subagent_type
        ? input.subagent_type[0].toUpperCase() + input.subagent_type.slice(1) : undefined
      return { icon: "task", title: t ? `${t} Agent` : "Agent", subtitle: input.description }
    }
    case "bash":
    case "shell":    return { icon: "console", title: "Shell", subtitle: input.command }
    case "edit":     return { icon: "code-lines", title: "Edit", subtitle: basename(input.filePath) }
    case "write":    return { icon: "code-lines", title: "Write", subtitle: basename(input.filePath) }
    case "patch":
    case "apply_patch":
      return { icon: "code-lines", title: "Patch",
               subtitle: input.files?.length ? `${input.files.length} file${input.files.length > 1 ? "s" : ""}` : undefined }
    case "todowrite": return { icon: "checklist", title: "To-dos" }
    case "question":  return { icon: "bubble-5", title: "Questions" }
    case "skill":     return { icon: "brain", title: input.name || "Skill" }
    default:          return { icon: "mcp", title: tool }
  }
}
```

### 9.3 落地步骤

1. 定义 `PART_MAPPING`：`text` / `reasoning` / `tool` / `compaction` 四个渲染器，其余类型不渲染。
2. 抄 `usePacedText`（§9.1），套在 markdown 渲染外面：`<Markdown text={usePacedText(text, streaming)} />`。
3. 抄 `getToolInfo`（§9.2）。**副标题选那个"一眼认出干了什么"的参数**，不要 dump 整个 input。
4. 实现 `BasicTool`：折叠容器 + `pending → 禁止展开` + 弹簧高度动画（`framer-motion` 的 `AnimatePresence` + `height: auto`），动画结束把 `overflow` 设回 `visible`。
5. 实现延迟挂载队列：全局数组 + LIFO + 每 rAF 一个 + `active` 标志。
6. 实现 `ToolStatusTitle`：`common()` 提共同前缀 → 锁旧宽度 → 下一帧设新宽度 → 600ms 解锁。`aria-label` 给完整文本。
7. 实现 `partDefaultOpen`（§4.6 可直接抄），接两个用户设置开关。
8. 实现 context 组折叠：`contextToolSummary` 数三类 + 复数文案 + 逗号独立元素 + 按 index 渲染。
9. 实现进度点阵：25 个 `<rect>`，1200ms，8 等分关键帧，加 `prefers-reduced-motion` 分支保留中心点。
10. 实现 `reasoningHeading`（§6.3）提取思考标题。

### 9.4 你需要自己提供的依赖

| 依赖 | 用途 | 建议 |
| --- | --- | --- |
| 折叠容器 | 工具卡片 | Radix Collapsible，或自己写 |
| 弹簧动画 | 高度过渡 | `motion` / `framer-motion` |
| 图标集 | 工具图标 | 任意；映射表里的名字自己换 |
| i18n + 复数 | 计数文案 | `i18next` 的 `_one` / `_other` |

---

## 10. 验收清单

- [ ] **V1** 一次性注入 5000 字符 → 8 帧内追平，不是逐字打半分钟
- [ ] **V1** 每次追加 3 字符（正常流式）→ 立刻全量显示，无额外延迟
- [ ] **V1** 落后正好 512 字符 → 全量；513 → 走节奏
- [ ] **V1** 内容被替换（非前缀追加）→ 立刻全量同步
- [ ] **V1** `live = false`（生成结束）→ 立刻全量，无残留 timer
- [ ] **V2** 步进位置落在空白或 `.,!?;:)]` 之后，不切断单词
- [ ] **V2** 向后 8 字符内没有边界字符 → 就按原位置切（不无限找）
- [ ] **V3** 工具 running 时点击标题 → 不展开
- [ ] **V3** `allowOpenWhilePending` 为 true 时 → 可展开
- [ ] **V3** `locked` 为 true 时 → 可展开不可收起
- [ ] **V4** `Searching` ↔ `Searched` → 前缀 `Search` 不动，只有后缀交叉淡入
- [ ] **V4** `Reading` ↔ `Read`（done 后缀为空）→ 走 swap 模式，整体淡入
- [ ] **V4** 切换期间宽度平滑过渡，600ms 后恢复 `auto`
- [ ] **V4** 屏幕阅读器读到完整文案，不是拆分碎片
- [ ] **V5** `read` 工具 → 默认折叠
- [ ] **V5** `edit` 且用户开启 editToolDefaultOpen → 展开
- [ ] **V5** 纯删除的 patch（`additions === 0 && deletions > 0`）→ 即使开启也不展开
- [ ] **V5** 多文件 patch 且全部 `type === "delete"` → 同上
- [ ] **V6** 同时挂载 20 个默认展开的工具 → 每帧一个，无长任务
- [ ] **V6** 挂载顺序从底部开始（视口内的先可交互）
- [ ] **V6** 组件卸载后其挂载任务不执行
- [ ] **V7** 开启系统"减少动态效果" → 点阵停止动画，中心点保持可见
- [ ] **计数** `3 reads, 2 searches` → 逗号是淡入的，不是突然出现
- [ ] **计数** 计数为 0 的项不显示，但 DOM 节点存在
- [ ] **兜底** 未注册的 MCP 工具 → `Called \`xxx\`` + mcp 图标，不崩溃
- [ ] **特判** 进行中的 `question` → 时间线不显示卡片
- [ ] **特判** 被忽略的 question → 右对齐灰字，不是红色错误卡
- [ ] **元信息** 只有最后一个文本 part 显示 `Agent · Model · 耗时`
- [ ] **展开动画** 展开后内部的下拉菜单不被 `overflow: hidden` 裁掉
