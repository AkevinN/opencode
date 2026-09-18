# SPEC-06 · 流式 Markdown 管线

> **优先级** P1 · **工作量** M · **依赖** SPEC-05
> **分析原文** [ch11 流式 Markdown 管线](../../context-and-streaming/11-streaming-markdown.md)
> **验收** [`acceptance/SPEC-06.yaml`](../acceptance/SPEC-06.yaml)（22 条）

**单项收益最高的能力之一**：约 60 行核心代码，把长回答后期的每帧耗时从几十毫秒降到几毫秒。

---

## 1. 目标与非目标

### 目标

流式渲染 markdown 时同时解决四个问题：

1. **每次 delta 全量重新解析是 O(n²)**（3000 字符的回答，最后一次解析 3000 字符，前面还有 300 次）
2. **markdown 中途语法不完整**（`**bold` 未闭合、代码围栏未闭合），直接渲染会看到字面量，闭合瞬间整块重排
3. **语法高亮很慢**（加载语法定义 + tokenize，同步做会阻塞主线程几十到几百毫秒）
4. **代码块在流式中途**：高亮需要完整上下文，但内容还在增长

### 非目标

- 不规定 markdown 解析器/高亮库。
- 不要求必须用 Worker（见 §8 分级）。

---

## 2. 领域模型

```ts
export type Block = {
  /** 原始 markdown 片段，用于比较和复用判定 */
  raw: string
  /** 实际交给渲染器的内容（live 块会被语法修补过） */
  src: string
  mode: "full" | "live" | "code"
  language?: string
  complete?: boolean
}

export type Projection = { text: string; blocks: Block[] }

/** 高亮结果是扁平二元组，不是 HTML 字符串。见 R-06-07 */
export type Token = [content: string, style: string]

export type WorkerRequest =
  | { type: "project";   id: number; key: string; text: string; live: boolean }
  | { type: "highlight"; id: number; key: string; text: string; language: string; complete?: boolean }
  | { type: "dispose";   key: string }

export type WorkerResponse =
  | { type: "project";    id: number; key: string; projection: Projection }
  | { type: "highlight";  id: number; key: string; language: string
      reset: boolean; stable: Token[]; unstable: Token[] }
  | { type: "error";      id: number; key?: string; message: string }
  | { type: "superseded"; id: number; key: string }
```

---

## 3. 规范条款

### R-06-01 必须把增长中的文本切成「稳定块 + 一个活动块」 · **必须**

**要求**：用块级词法分析把文本分词，**最后一个非空白 token 之前**的全部视为稳定块，只有尾部那个是活动块（`live`）。

**理由**：markdown 的块级语法保证——一旦一个段落后面出现了新的块，前面那个段落不可能再变。稳定块的渲染结果可以无限期缓存。

---

### R-06-02 含链接引用定义时必须放弃增量 · **必须**

**要求**：检测到文本含链接引用定义（形如 `[foo]: http://…`）时，整体当作一个 `live` 块。

**要求**：先做便宜的子串检查再做正则（多数文本一次字符串搜索就短路）。

**理由**：引用定义可以出现在文档任何位置并影响**前面**的引用，破坏了 R-06-01 的前提。

---

### R-06-03 活动块必须做语法修补 · **必须**

**要求**：活动块交给渲染器前，补全未闭合的 markdown 标记（`**`、`*`、行内代码等）。

**要求**：**未闭合的链接必须补成纯文本**，不能补成一个坏链接。

**理由**：不修补，用户会看到 `**` 字面量，闭合的瞬间整块重排。链接补成 `[text](htt)` 会渲染出一个可点击的坏链接——等 URL 收完了下一帧自然变成真链接。

---

### R-06-04 未闭合代码围栏必须走 O(1) 增量快路径 · **必须**

**要求**：满足以下全部条件时，**只把新增后缀追加到尾块**，其余块对象**原样复用**：

- 新文本是旧文本的前缀扩展
- 尾块 `mode === "code"` 且 `complete` 为假
- 新增后缀**不会**闭合围栏

**理由**：代码块是最常见的大块内容，也是词法分析最贵的输入。一个 200 行代码块流式会收到几百次 delta——快路径是几百次 O(1) 字符串拼接，而不是几百次全量分词。

---

### R-06-05 围栏闭合判定必须容忍跨批次切分 · **必须**

**要求**：判断新增后缀是否闭合围栏时，**必须拼上尾块末尾的 `markLength - 1` 个字符**再判断。

**理由**：围栏标记可能跨 delta 边界被切开（前一批收到反引号的前两个，这一批收到第三个）。不拼会漏判，代码块永远不闭合。

**要求**：闭合围栏的标记数必须 **≥** 开启围栏的标记数（4 个开启、3 个出现不算闭合）。

**要求**：支持 0–3 空格缩进的围栏（4 空格是代码缩进，不是围栏）。

---

### R-06-06 流结束时必须收尾 · **必须**

**要求**：`live` 变 false 时：`live` 块 → `full`，未完成的 `code` 块 → `complete: true`。

**要求**：优先在上次的流式投影基础上"收尾"，而不是整体重算。

---

### R-06-07 高亮结果必须以 token 二元组传输 · **应该**

**要求**：高亮产物是 `[content, style][]`，不是 HTML 字符串。

**理由**：一段 500 行代码会产生几千个 token。跨 Worker 边界传对象数组的序列化开销远高于二元组；传 HTML 字符串则要么 `innerHTML`（XSS 风险 + 整块重建），要么再解析一遍。

---

### R-06-08 流式高亮必须分 stable / unstable 两段 · **应该**

**要求**：流式 tokenize 每次只喂增量，返回两段：

- `stable`：确定不会再变的 token → 渲染时**只追加**
- `unstable`：可能被后续输入改变的 token（未结束的字符串、可能是关键字前缀的标识符）→ 每次**整体替换**

**要求**：合并响应时按 `id` 单调过滤（丢弃乱序/过期响应）；`reset` 为真时替换 stable，为假时追加。

**理由**：一个 500 行代码块流式渲染时，只有末尾几个 token 在反复变。

---

### R-06-09 必须有两级 latest-wins 调度 · **必须**

**要求**：

| 层 | 位置 | 规则 |
| --- | --- | --- |
| 传输层 | 主线程 | 每个 key 最多一个**在途**请求；第二个进排队位并取代前一个排队项 |
| 执行层 | Worker 内 | 每个 key 一个**槽位**，槽位在队列里位置固定（FIFO 公平），但**槽位里的请求可被覆盖** |

**理由**：
- 传输层防止消息洪水（结构化克隆本身有开销）
- 执行层防止执行洪水
- 槽位机制同时保证：顺序公平（不会有块饿死）+ 不积压（一个块最多一个待执行请求）

**要求**：`dispose` 也必须进队列（保证在同 key 的执行之后），不能直接删状态。

---

### R-06-10 块 key 必须包含 mode · **必须**

**要求**：块 key 形如 `${owner}:${cacheKey}:${index}:${mode}`。

**理由**：同一个 index 的块从 `live` 变成 `full` 时 key 必须变，强制丢弃旧的流式状态、重新走完整渲染路径。不带 mode 会复用错误的流式状态。

---

### R-06-11 代码块内容非前缀增长时必须整体重置 · **必须**

**要求**：满足任一条件即重置该块的 token 状态：语言变了 / 上游发生过 reset / stable 数量回退 / 新内容不是旧内容的前缀扩展。

**理由**：不重置会导致 token 追加错位，高亮全乱。

---

### R-06-12 稳定块必须可被引用相等短路 · **必须**

**要求**：稳定块的组件用 `raw` 做记忆化比较（`full` 块要求完全相等；`code`/`live` 块要求前缀扩展）。

**理由**：这是 R-06-01 产生收益的地方。不做记忆化，切块了也还是每帧全部重渲染。

---

### R-06-13 未知语言必须降级 · **必须**

**要求**：高亮库不认识的语言退化为纯文本，不得抛错。语法定义按需加载。

---

### R-06-14 全量 tokenize 时必须补回行间换行 · **必须**

**要求**：按行 tokenize 的结果拼接时，行之间要补 `\n` token（最后一行除外）。

**理由**：不补，10 行代码会渲染成 1 行。

---

## 4. 算法规范

### 4.1 块切分

```
splitBlocks(text, live):
    if not live: return [{ raw: text, src: text, mode: "full" }]
    if 含链接引用定义: return [{ raw: text, src: heal(text), mode: "live" }]        # R-06-02

    tokens = lex(text)
    tail   = 最后一个非空白 token 的下标
    if tail < 0: return [{ raw: text, src: heal(text), mode: "live" }]

    out = []
    for i in [0, tail):                                                            # R-06-01 稳定块
        t = tokens[i];  if t 是空白: continue
        raw = t.raw
        while tokens[i+1] 是空白 and i+1 < tail: raw += tokens[++i].raw            # 吞掉后续空白
        if t 是代码块: out.push({ raw, src: t.text, mode: "code",
                                 language: 解析语言(t.lang), complete: true })
        else:          out.push({ raw, src: raw, mode: "full" })

    raw = tokens[tail..] 的 raw 拼接
    last = tokens[tail]
    if last 不是代码块: return out + [{ raw, src: heal(raw), mode: "live" }]
    if 围栏已闭合(last.raw):
        return out + [{ raw, src: last.text, mode: "code", language, complete: true }]
    return out + [{ raw, src: 去掉首行(last.raw), mode: "code", language }]        # 未闭合
```

### 4.2 增量投影

```
project(previous, text, live):
    if not live:                                                                   # R-06-06 收尾
        current = previous?.text == text ? previous
                : previous and text.startsWith(previous.text) ? project(previous, text, true)
                : undefined
        if not current: return { text, blocks: [{ raw: text, src: text, mode: "full" }] }
        return { text, blocks: current.blocks.map(b =>
            b.mode == "live"                  ? { raw: b.raw, src: b.raw, mode: "full" }
          : b.mode == "code" and not b.complete ? { ...b, complete: true }
          : b) }

    if not previous or not text.startsWith(previous.text):                         # 慢路径
        return { text, blocks: splitBlocks(text, true) }

    tail   = previous.blocks.last
    suffix = text.slice(previous.text.length)
    if suffix 为空 or tail?.mode != "code" or tail.complete or 闭合围栏(tail.raw, suffix):
        return { text, blocks: splitBlocks(text, true) }

    return { text, blocks: previous.blocks[0..-1] +                                # R-06-04 快路径
             [{ ...tail, raw: tail.raw + suffix, src: tail.src + suffix }] }
```

**慢路径也不算全量重建**：重新分词后产出的稳定块 `raw` 与上次相同，下游的记忆化比较会判定可复用（R-06-12）。

### 4.3 围栏判定

```
围栏已闭合(raw):
    mark = raw 开头匹配 /^[ \t]{0,3}(`{3,}|~{3,})/ 的标记
    if not mark: return true                       # 不是围栏块
    last = raw.trimEnd() 的最后一行.trim()
    return 正则(`^[\t ]{0,3}${mark[0]}{${mark.length},}[\t ]*$`).test(last)         # R-06-05 数量 >=

闭合围栏(raw, suffix):                                                              # R-06-05 跨批次
    mark = raw 开头的围栏标记
    if not mark: return suffix 含 "```" or suffix 含 "~~~"
    return (raw.slice(-(mark.length - 1)) + suffix).includes(mark)
```

### 4.4 两级调度

```
# 主线程：传输层
createTransport(post, supersede):
    active = {};  queued = {}
    send(req):
        if not active[req.key]: active[req.key] = req; post(req); return
        if queued[req.key]: supersede(queued[req.key])
        queued[req.key] = req
    complete(key, id):
        if active[key]?.id != id: return            # 不是当前在途的响应 → 忽略
        delete active[key]
        next = queued[key]
        if next: delete queued[key]; active[key] = next; post(next)

# Worker 内：执行层（槽位机制）
createLatestQueue(run, supersede, dispose):
    jobs = [];  slots = {};  running = false
    submit(req):
        slot = slots[req.key]
        if slot:                                    # 槽位已在队列里
            if slot.request: supersede(slot.request)
            slot.request = req                      # 覆盖，不新增排队项
            return
        slot = { key: req.key, request: req }
        slots[req.key] = slot;  jobs.push(slot);  schedule()
    dispose(key):
        清空槽位；jobs.push({ type: "dispose", key })   # 进队列保证顺序
        schedule()
```

### 4.5 高亮响应合并

```
applyHighlight(state, res):
    if state and res.id <= state.id: return state                                  # R-06-08 乱序过滤
    return {
      id: res.id,
      generation: (state?.generation ?? 0) + (res.reset ? 1 : 0),
      language: res.language,
      stable:  res.reset ? res.stable : [...(state?.stable ?? []), ...res.stable],
      unstable: res.unstable,                       # unstable 始终整体替换
    }
```

---

## 5. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 是否用 Worker | 是 | — | 不用：主线程会被高亮阻塞；适合无大代码块的场景 |
| 传输层在途上限 | 每 key 1 个 | 固定 | 不建议改 |
| 围栏最小标记数 | 3 | 固定（markdown 规范） | — |
| 围栏最大缩进 | 3 空格 | 固定（markdown 规范） | — |
| 跨批次回看长度 | `markLength - 1` | 固定 | — |

---

## 6. 接口契约

```ts
interface MarkdownPipeline {
  /** 增量投影；previous 为上次结果 */
  project(previous: Projection | undefined, text: string, live: boolean): Projection
  /** 块是否可复用（渲染层的 memo 判据） */
  canReuse(current: Pick<Block, "mode" | "raw"> | undefined, next: Block): boolean
  /** 块 key */
  blockKey(owner: string, cacheKey: string | undefined, index: number, mode: string): string
}
```

---

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| 每个 delta 全量解析 | O(n²)，长回答后期每帧几十毫秒 |
| 不检测链接引用定义 | 前面的引用渲染错 |
| 不修补未闭合语法 | 看到 `**` 字面量，闭合时整块重排 |
| 未闭合链接补成链接 | 渲染出可点击的坏链接 |
| 围栏判定不拼回看字符 | 跨批次切分时代码块永不闭合 |
| 闭合围栏数量不要求 ≥ | 提前闭合，后面全渲染错 |
| 块 key 不含 mode | live→full 时复用错误的流式状态 |
| 传 HTML 字符串 | 序列化开销 + XSS 风险 + 整块重建 |
| 只做一级 latest-wins | 消息洪水或执行洪水 |
| `dispose` 不进队列 | 删掉正在用的状态导致崩溃 |
| 稳定块不做记忆化 | 切块白做，仍然每帧全渲染 |
| 未知语言不降级 | 整块渲染失败 |
| 全量 tokenize 不补换行 | 10 行代码渲染成 1 行 |

---

## 8. 分级实现路径

### 最小可用版（半天，不用 Worker）

只做**块投影 + 语法修补**（§4.1 + R-06-03）。约 60 行。

```tsx
const blocks = useMemo(() => splitBlocks(text, streaming), [text, streaming])
return blocks.map((b, i) => <MarkdownBlock key={`${i}:${b.mode}`} block={b} />)

const MarkdownBlock = React.memo(Inner, (a, b) =>
  a.block.mode === b.block.mode && a.block.raw === b.block.raw)
```

**这一步就能把长回答后期的每帧耗时从几十毫秒降到几毫秒**，因为稳定块的记忆化命中，只有最后一个块在重算。

### 中间版（+1 天）

加增量投影的快路径（§4.2）——大代码块流式时的主要收益来源。

### 完整版（+1–2 天）

Worker + 两级 latest-wins + 流式高亮 stable/unstable。

---

## 9. 验收

见 [`acceptance/SPEC-06.yaml`](../acceptance/SPEC-06.yaml)（22 条）。

围栏相关的四条（A-06-11 ~ A-06-14）建议优先写成单测——它们是最容易写错也最难在真实场景复现的部分。
