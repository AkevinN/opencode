# SPEC-04 · 时间线投影与滚动

> **优先级** P1 · **工作量** L · **依赖** SPEC-01, SPEC-03
> **分析原文** [ch09 时间线投影与滚动](../../context-and-streaming/09-timeline-and-scroll.md)
> **验收** [`acceptance/SPEC-04.yaml`](../acceptance/SPEC-04.yaml)（27 条）

---

## 1. 目标与非目标

### 目标

把消息流变成**不抖动的虚拟列表**，同时解决四件互相纠缠的事：

1. 消息 ≠ 渲染行（一次提问会产生多条消息、几十个片段，外加分隔线、思考指示、diff 摘要、错误块）
2. 流式时每帧都在变，不能整表重建
3. 行高未知且会变（一个 markdown 块渲染完高度可能翻三倍）
4. 贴底跟随是个状态机，不是 `scrollTop = scrollHeight`

### 非目标

- 不规定虚拟化库（任何支持动态测量 + 自定义 key 的都行）。
- 不规定视觉设计。

---

## 2. 领域模型

### 2.1 渲染行

```ts
/** 每一行都带 turnId（归属于哪一轮），key 因此天然按轮次分段 */
export type TimelineRow =
  | { tag: "TurnGap";       turnId: string }                                   // 轮次间距
  | { tag: "UserMessage";   turnId: string; anchor: boolean }
  | { tag: "TurnDivider";   turnId: string; label: "compaction" | "interrupted" }
  | { tag: "AssistantPart"; turnId: string; group: PartGroup; hasPrevious: boolean }
  | { tag: "Thinking";      turnId: string; heading?: string }
  | { tag: "DiffSummary";   turnId: string; diffs: SummaryDiff[] }
  | { tag: "Error";         turnId: string; text: string }
  | { tag: "Retry";         turnId: string }

/** 连续的只读工具折叠成一个 context 组 */
export type PartGroup =
  | { key: string; type: "part";    ref: { messageId: string; partId: string } }
  | { key: string; type: "context"; refs: { messageId: string; partId: string }[] }

export function rowKey(row: TimelineRow): string {
  switch (row.tag) {
    case "TurnGap":       return `turn-gap:${row.turnId}`
    case "UserMessage":   return `user-message:${row.turnId}`
    case "TurnDivider":   return `turn-divider:${row.turnId}:${row.label}`
    case "AssistantPart": return `assistant-part:${row.turnId}:${row.group.key}`
    case "Thinking":      return `thinking:${row.turnId}`
    case "DiffSummary":   return `diff-summary:${row.turnId}`
    case "Error":         return `error:${row.turnId}`
    case "Retry":         return `retry:${row.turnId}`
  }
}
```

### 2.2 Turn（轮次）

一条 user 消息 + 它引出的所有 assistant 消息。UI 的分组单位。
依赖 assistant 消息带 `parentId` 指向它所属的 user 消息。

---

## 3. 规范条款

### R-04-01 行必须有稳定 key，且不得用数组下标 · **必须**

**要求**：虚拟化列表的 `getItemKey` 必须返回 `rowKey(row)`，不是 index。

**理由**：用 index 做 key 时，在列表头部插入一条会让所有 item 身份错位，整表重建。

**要求**：行已从投影中消失但测量回调仍在触发时，返回一个安全的占位 key（如 `removed:${index}`），不得直接访问 `rows[index]`。

---

### R-04-02 重投影必须复用未变化的行对象 · **必须**

**要求**：每次重投影后跑一次「行复用」：

- **行级**：新行与旧行结构相等 → 返回**旧对象引用**
- **数组级**：所有行都未变 → 返回**旧数组引用**

**理由**：下游用引用相等做 memo 判断。不复用 = 每帧几百个组件重建 = 掉帧 + 折叠状态丢失 + 滚动位置乱跳。

---

### R-04-03 context 组的 key 必须在流式追加时保持稳定 · **必须**

**要求**：把连续的只读工具折叠成 context 组时，组 key 取**组内第一个成员的 id**；并且重投影时通过成员 id 反查旧组，沿用旧 key。

**场景**：模型先调 `grep`（成组，key = `context:p1`），中间插了一个非只读工具，又回到只读工具。重新分组后第一个成员变了，key 就变了 → 整行重建 → 折叠状态丢失、高度重测、滚动跳变。

**要求**：反查时用两个集合防止多个新组抢同一个旧 key（一个旧 key 只能被认领一次）。

---

### R-04-04 贴底跟随必须是显式状态 · **必须**

**要求**：维护一个布尔 `userScrolled`，由**用户主动滚开**这个事件驱动，不由滚动位置直接判断。

**总开关**：`shouldAnchorBottom = !urlHasHash && !targetMessageId && !pendingMessage && !userScrolled`

**理由**：直接用「当前距底距离」判断会在内容变高的那一瞬间误判成「用户滚开了」。

---

### R-04-05 程序化滚动必须可识别 · **必须**

**要求**：每次程序化滚动前记录 `{targetTop, timestamp}`；滚动事件到达时，若在时间窗内（参考 1500ms）且 `|scrollTop - targetTop| < 2px`，判定为「自己滚的」，**不置 `userScrolled`**。

**理由**：浏览器异步派发滚动事件。如果在我们调 `scrollTo()` 和滚动事件触发之间有新内容到达，处理器会看到非零的距底距离并错误地认为用户滚动了。**不做这个识别，流式一开始跟随就失效。**

---

### R-04-06 只有向上滚才停止跟随 · **必须**

**要求**：滚轮处理里 `deltaY >= 0`（向下）直接返回，不停止跟随。

---

### R-04-07 嵌套可滚区域内的滚动不得中断外层跟随 · **必须**

**要求**：约定一个属性（如 `data-scrollable`）标记内部可滚区域（工具输出块、代码块）。滚轮事件的 target 在这类区域内且不是外层容器时，直接返回。

**理由**：用户在工具输出里滚动查看，不代表他想脱离对话底部。

---

### R-04-08 内容尺寸变化时必须同帧贴底 · **必须**

**要求**：在 `ResizeObserver` 回调里**直接**调贴底，不要包一层 `requestAnimationFrame`。

**理由**：`ResizeObserver` 在 layout 之后、paint 之前触发。在这里贴底用户看不到抖动；用 rAF 就晚了一帧，会看到「先跳上去再追下来」。

---

### R-04-09 生成结束后必须有跟随尾巴 · **必须**

**要求**：`working` 由 true 变 false 后，继续跟随一小段时间（参考 300ms）。

**理由**：生成结束时最后的布局（代码高亮完成、图片加载）还会改变高度。没有尾巴会停在离底几百 px 的位置。

---

### R-04-10 行高变化的补偿必须分两种情况 · **必须**

| 状态 | 行为 |
| --- | --- |
| 贴底中 | **不做位置补偿**，直接重新贴底 |
| 非贴底 | **只补偿视口之上**的行的高度变化 |

**理由**：视口内和视口下的变化不补偿——补偿了反而会让用户正在看的内容乱动。

---

### R-04-11 向上加载历史必须做位置锚定 · **必须**

**要求**：
1. 加载前：记录第一个可见行的 key 和它距视口顶的偏移
2. 加载后：在下一帧找回该行，把 `scrollTop` 补回差值

**要求**：每个渲染行必须带一个可查询的 key 属性（如 `data-timeline-key`）供锚定使用。

**理由**：不做锚定，插入历史后视口内容会整体跳走。

---

### R-04-12 首屏渲染 overscan 必须渐进提升 · **应该**

**要求**：首屏用较小的渲染 overscan（参考 6），挂载后两帧内提升到常规值（参考 20）。

**理由**：一上来渲染 50 个会显著拖慢首次可交互时间；但稳定后 overscan 太小会在滚动时看到空白。

---

### R-04-13 当前活跃轮次的最后一行必须始终渲染 · **应该**

**要求**：range 计算时强制并入当前活跃轮次的最后一行索引，即使它滚出视口。

**理由**：保证流式内容始终在测量，高度不会在用户滚回去时突变。

---

### R-04-14 巨幅高度变化必须钉住可见项 · **应该**

**要求**：某行高度变化超过一整个视口高度时，把当前可见的行索引强制并入 range 若干帧。

**理由**：重算 range 时这些 item 可能瞬间被移出，产生白屏闪烁（典型场景：展开一个大 diff）。

---

### R-04-15 滚动状态派生必须合帧 · **应该**

**要求**：滚动事件驱动的派生状态（是否在底部、是否显示「跳到最新」）用 rAF 合帧，且值未变化时不写 store。

**理由**：滚动事件每秒几十次，不合帧会疯狂触发响应式更新。

**参考判据**：
- `atBottom = !overflow || distanceFromBottom <= 2`
- `showJumpButton = overflow && distanceFromBottom > max(400, clientHeight)`

---

## 4. 算法规范

### 4.1 投影：消息 → 行

```
buildRows(messages, parts, status, showReasoning):
  # ① 聚成 turn
  turns = [];  byUserId = {}
  for m in messages:
      if m 是 user:
          if byUserId 已有 m.id: continue
          turn = {user: m, assistants: []};  turns.push(turn);  byUserId[m.id] = turn
      else if m 是 assistant:
          t = byUserId[m.parentId]
          if t: t.assistants.push(m)
          else:                                    # assistant 先于 user 到达（乱序）
              u = 查 m.parentId
              if u 是 user:
                  turn = {user: u, assistants: [m]};  turns.push(turn);  byUserId[u.id] = turn
  # 本地乐观插入但服务端还没确认的 user 消息，按 id 序插入
  activeTurnId = turns.last?.user.id

  # ② 每个 turn 展开成行（顺序即视觉顺序）
  rows = []
  for (turn, i) in turns:
      isActive = turn.user.id == activeTurnId
      userParts = parts[turn.user.id]
      hasCompaction = userParts 含 compaction
      interruptedIndex = turn.assistants 中第一个「中断」错误的下标
      latestError = turn.assistants.last?.error
      error = (latestError 是中断错误) ? undefined : latestError

      partRefs = turn.assistants.flatMap((m, mi) =>
          parts[m.id].filter(p => renderable(p, showReasoning)).map(p => ({m, mi, p})))

      items = 中断存在且无压缩
              ? groupParts(中断点之前) + [{type:"interrupted"}] + groupParts(中断点之后)
              : groupParts(partRefs)

      if i > 0:            rows.push(TurnGap)
      rows.push(UserMessage)
      if hasCompaction:    rows.push(TurnDivider("compaction"))
      for it in items:
          if it 是 interrupted: rows.push(TurnDivider("interrupted"))
          else:                 rows.push(AssistantPart(it.group, hasPrevious))
      if isActive and status == "busy" and not error
         and (showReasoning ? partRefs 为空 : true):
          rows.push(Thinking(从 reasoning 提取的标题))
      if isActive and status == "retry":            rows.push(Retry)
      if diffs 非空 and (status == "idle" or not isActive): rows.push(DiffSummary)
      if error:                                     rows.push(Error)
  return { rows, activeTurnId }
```

**三个容易抄错的条件**：

| 条件 | 规则 | 为什么 |
| --- | --- | --- |
| Thinking 显示 | `showReasoning` 为真时**只有还没有任何 part** 才显示 | 有 reasoning 内容了就显示内容本身 |
| DiffSummary 显示 | 当前轮 busy 时**不显示** | 文件还在改，显示了会不停跳变 |
| 中断 vs 错误 | 「中断」走 TurnDivider，其它错误走 Error 行 | 中断是用户主动的，不是错误 |

### 4.2 只读工具折叠

```
groupParts(refs):
  result = [];  start = -1
  flush(end):
      if start < 0: return
      result.push({ key: `context:${refs[start].p.id}`,      # ← 组内第一个成员的 id
                    type: "context",
                    refs: refs[start..end].map(toRef) })
      start = -1
  for (r, i) in refs:
      if isReadOnlyTool(r.p):                                 # read / glob / grep / list 之类
          if start < 0: start = i
          continue
      flush(i - 1)
      result.push({ key: `part:${r.m.id}:${r.p.id}`, type: "part", ref: toRef(r) })
  flush(refs.length - 1)
  return result
```

**这是 coding agent UI 里最有效的降噪手段**——一次任务里只读工具能占 70% 的调用，折叠后时间线噪音立刻下来。

### 4.3 行复用

```
reuseRows(previous, next):
  if previous 为空: return next
  byKey = Map(previous.map(r => [rowKey(r), r]))

  # 建立「旧 context 组包含哪些 partId」的反向索引
  contextByPart = {}
  for (r, i) in previous:
      if r 是 AssistantPart 且 group.type == "context":
          for ref in r.group.refs: contextByPart[`${r.turnId}:${ref.partId}`] = {i, r}

  reserved = {}   # 已被同 key 新行占用的位置
  claimed  = {}   # 已被认领的旧 key

  out = next.map((row, index) => {
      row = stabilizeContextKey(row, index, contextByPart, reserved, claimed)   # R-04-03
      old = byKey[rowKey(row)]
      if not old: return row
      return structurallyEqual(old, row) ? old : row                            # R-04-02 行级
  })

  if previous.length == out.length and 全部引用相同: return previous             # R-04-02 数组级
  return out
```

### 4.4 贴底跟随状态机

```
状态：userScrolled: boolean

distanceFromBottom(el) = el.scrollHeight - el.clientHeight - el.scrollTop
canScroll(el)          = el.scrollHeight - el.clientHeight > 1

markAuto(el):                                   # 程序化滚动前调用
    auto = { top: max(0, el.scrollHeight - el.clientHeight), time: now }
    清除旧定时器；AUTO_WINDOW_MS 后清空 auto

isAuto(el) = auto 存在 and now - auto.time <= AUTO_WINDOW_MS
             and |el.scrollTop - auto.top| < AUTO_TOLERANCE_PX

scrollToBottom(force):
    if not force and (userScrolled or (not working and not settling)): return
    if distanceFromBottom < 2: markAuto(el); return
    markAuto(el)
    el.scrollTop = el.scrollHeight              # 直接赋值绕过 CSS smooth

onScroll():                                      # R-04-04/05
    if not canScroll(el):                    userScrolled = false; return
    if distanceFromBottom < BOTTOM_PX:       userScrolled = false; return   # 回到底部 → 恢复
    if not userScrolled and isAuto(el):      scrollToBottom(false); return  # 自己滚的
    userScrolled = true                                                     # 其余 → 用户滚开

onWheel(e):                                      # R-04-06/07
    if e.deltaY >= 0: return
    if e.target 在 [data-scrollable] 内 且 不是外层容器: return
    userScrolled = true

ResizeObserver(content):                         # R-04-08
    if not canScroll: userScrolled = false; return
    if userScrolled: return
    scrollToBottom(false)                        # 直接做，不包 rAF

working 变 false:                                # R-04-09
    settling = true;  SETTLE_MS 后 settling = false
```

---

## 5. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 测量 overscan | `50` | 20 – 100 | 调大：高度提前算好、滚动更稳，测量成本上升 |
| 渲染 overscan（首屏 → 稳定） | `6` → `20` | 3–10 → 10–40 | 首屏值调大：首次可交互变慢；稳定值调小：快速滚动见空白 |
| 列表底部留白 | `64px` | 0 – 120 | 纯视觉 |
| `BOTTOM_PX`（算在底部的阈值） | `10` | 2 – 40 | 调大：更容易恢复跟随，但用户微调也会被当成回到底部 |
| 严格贴底判据 | `<= 2px` | 0 – 4 | 用于隐藏「回到底部」按钮 |
| `AUTO_WINDOW_MS` | `1500` | 800 – 3000 | 调小：慢设备上程序化滚动会被误判成用户滚动 |
| `AUTO_TOLERANCE_PX` | `2` | 1 – 5 | 同上 |
| `SETTLE_MS` | `300` | 150 – 800 | 调小：高亮/图片加载完后停在半空 |
| 跳转按钮阈值 | `max(400, clientHeight)` | — | 距底多远才显示「跳到最新」 |

---

## 6. 接口契约

```ts
interface Timeline {
  /** 消息 → 行（含 turn 分组、折叠、条件行） */
  buildRows(input: { messages; parts; status; showReasoning }): { rows: TimelineRow[]; activeTurnId?: string }
  /** 增量复用，必须在 buildRows 之后调用 */
  reuseRows(previous: TimelineRow[] | undefined, next: TimelineRow[]): TimelineRow[]
}

interface AutoScroll {
  userScrolled: boolean
  onScroll(): void
  onWheel(e: WheelEvent): void
  /** 用户点「跳到最新」 */
  resume(): void
}
```

---

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| 用 index 做虚拟列表 key | 头部插入后全表错位 |
| 不做行复用 | 每帧几百组件重建，折叠状态丢失 |
| context 组 key 随成员变化 | 整行重建 + 高度重测 + 滚动跳变 |
| 不识别程序化滚动 | 流式一开始跟随就失效 |
| 向下滚也停止跟随 | 用户轻微下滑就脱离底部 |
| 嵌套区域滚动算作离开 | 看工具输出就失去跟随 |
| 在 rAF 里贴底 | 看得见「先跳上去再追下来」 |
| 生成结束立刻停止跟随 | 停在离底几百 px |
| 贴底时还做位置补偿 | 与 scrollToEnd 打架 |
| 补偿视口内/下的高度变化 | 用户正看的内容乱动 |
| 不做 prepend 锚定 | 加载历史后视口跳走 |
| 滚动状态不合帧 | 每秒几十次响应式更新 |

---

## 8. 分级实现路径

### 最小可用版（1–2 天）

**只做贴底跟随状态机**（§4.4），不做虚拟化。约 80 行。
这一步就能解决「流式时页面乱跳」和「上滑后被强制拉回底部」两个最招骂的问题。

### 中间版（+2 天）

加上行投影（§4.1）+ 行复用（§4.3）+ `React.memo` 引用比较。
这一步解决「每帧整表重建」。

### 完整版（+2–3 天）

接虚拟化库 + prepend 锚定 + 高度变化补偿 + overscan 渐进提升。
1000 条消息稳定 60fps。

---

## 9. 验收

见 [`acceptance/SPEC-04.yaml`](../acceptance/SPEC-04.yaml)（27 条）。

性能与视觉类断言需人工确认：1000 条消息流式追加稳定 60fps（A-04-26）；
向上加载 20 条历史后视口内容视觉位置不变（A-04-21，建议截图对比）。
