# SPEC-01 · 消息与片段模型

> **优先级** P0 · **工作量** S · **依赖** 无
> **分析原文** [ch04 消息模型与历史投影](../../context-and-streaming/04-history-and-messages.md)
> **验收** [`acceptance/SPEC-01.yaml`](../acceptance/SPEC-01.yaml)（17 条）

整个工具包的词汇表。**先做它，且一次做对**——中途改消息模型，所有下游能力返工。

---

## 1. 目标与非目标

### 目标

一份**前后端共用**的消息与片段类型，同时满足四个互相拉扯的消费者：

| 消费者 | 要什么 |
| --- | --- |
| 模型 | 符合 provider 协议的 messages 数组，工具调用/结果配对正确，思考链可续接 |
| UI | 可增量更新的细粒度片段（一段文本正在流、一个工具正在跑），可折叠、可定位 |
| 持久化 | 能从事件流完整重建；被裁剪出上下文的消息仍可审计 |
| 压缩 | 能被裁剪，且裁剪点清晰 |

### 非目标

- 不规定存储引擎。
- 不规定 provider 协议的具体形状（那是适配层的事）。
- 不要求实现全部消息类型——最小集见 §8。

---

## 2. 领域模型

以下是**规范性**的接口契约。字段名可以按你的项目约定改（`camelCase`/`snake_case`），但**结构和语义不可改**。

```ts
// ============ 公共 ============
/** 单调递增，字典序 == 时间序。见 R-01-02 */
export type MessageId = string

export interface UnknownError { type: "unknown"; message: string }
export type ProviderMetadata = Record<string, unknown>

export type ToolContent =
  | { type: "text"; text: string }
  | { type: "file"; mime: string; name?: string; uri: string }

interface MessageBase {
  id: MessageId
  metadata?: Record<string, unknown>
  time: { created: number }        // epoch ms
}

// ============ assistant 的内容片段 ============
export interface TextPart {
  type: "text"
  id: string
  text: string
}

export interface ReasoningPart {
  type: "reasoning"
  id: string
  text: string
  /** provider 私有的续接凭据（思考签名等）。见 R-01-07 */
  providerMetadata?: ProviderMetadata
  time?: { created: number; completed?: number }
}

/** 工具四态单向机。见 R-01-03 */
export type ToolState =
  | { status: "pending";   input: string }                    // ← 注意是 string
  | { status: "running";   input: Record<string, unknown>; structured: Record<string, unknown>
      content: ToolContent[] }
  | { status: "completed"; input: Record<string, unknown>; structured: Record<string, unknown>
      content: ToolContent[]; attachments?: FileAttachment[]; outputPaths?: string[]; result?: unknown }
  | { status: "error";     input: Record<string, unknown>; structured: Record<string, unknown>
      content: ToolContent[]; error: UnknownError; result?: unknown }

export interface ToolPart {
  type: "tool"
  id: string                       // = provider 侧的 tool call id
  name: string
  provider?: {
    executed: boolean              // provider 侧执行（如内置检索）
    metadata?: ProviderMetadata
    resultMetadata?: ProviderMetadata
  }
  state: ToolState
  time: { created: number; ran?: number; completed?: number }
}

export type AssistantContent = TextPart | ReasoningPart | ToolPart

// ============ 消息判别联合 ============
export type Message =
  | (MessageBase & { type: "user"; text: string; files?: FileAttachment[] })
  | (MessageBase & { type: "assistant"
      agent: string
      model: { providerId: string; modelId: string }
      content: AssistantContent[]                            // ← 有序，见 R-01-01
      tokens?: { input: number; output: number; reasoning: number
                 cache: { read: number; write: number } }
      finish?: string
      error?: UnknownError
      time: { created: number; completed?: number } })
  | (MessageBase & { type: "system"; text: string })          // 会话中系统消息，见 SPEC-13
  | (MessageBase & { type: "synthetic"; text: string })       // 系统注入的伪用户消息
  | (MessageBase & { type: "shell"; callId: string; command: string; output: string })
  | (MessageBase & { type: "compaction"; reason: "auto" | "manual"
      summary: string; recent: string })                      // 见 SPEC-14
  | (MessageBase & { type: "agent-switched"; agent: string }) // UI-only
  | (MessageBase & { type: "model-switched"; model: { providerId: string; modelId: string } }) // UI-only
```

### 各消息类型的职责

| type | 谁产生 | 模型可见 | 转成什么 role | UI 表现 |
| --- | --- | --- | --- | --- |
| `user` | 用户输入被提升（SPEC-07） | ✅ | `user` | 用户气泡 |
| `assistant` | 模型一轮输出 | ✅ | `assistant`（+ 附带的 tool 消息） | 文本/思考/工具卡片 |
| `system` | 上下文变更（SPEC-13） | ✅ | `system` | 通常折叠成一行细提示 |
| `synthetic` | 系统注入 | ✅ | **`user`** | 特殊样式 |
| `shell` | 用户直接执行的命令 | ✅ | **`user`**，格式化 | 终端块 |
| `compaction` | 压缩完成（SPEC-14） | ✅ | **`user`**，带包裹标签 | 分隔线 |
| `agent-switched` | 用户换 agent | ❌ | — | 分隔线 |
| `model-switched` | 用户换模型 | ❌ | — | 分隔线 |

---

## 3. 规范条款

### R-01-01 assistant 内容必须是有序数组 · **必须**

**要求**：`assistant.content` 是 `AssistantContent[]`，顺序即模型的输出顺序，文本、思考、工具调用可以任意交错。

**理由**：模型会「说一段 → 调工具 → 再说一段」。把文本和工具拆成两个字段（如 `text: string` + `toolCalls: []`）会永久丢失交错顺序，UI 无法还原真实过程，回放给模型时顺序也是错的。

**常见错误**：用 `{ text, toolCalls }` 两个平行字段。

---

### R-01-02 消息 ID 必须单调递增且字典序等于时间序 · **必须**

**要求**：ID 生成保证：先创建的 ID 字符串小于后创建的。推荐 ULID 或 `<zero-padded timestamp><counter>`。

**理由**：前端用二分查找插入维持有序数组（SPEC-03 `R-03-02`）。UUIDv4 无序，会退化成每次插入都要全排序。

**常见错误**：用 UUIDv4；用毫秒时间戳（同毫秒会撞）。

---

### R-01-03 工具状态必须是四态单向机 · **必须**

**要求**：`pending → running → completed | error`，不可回退。

**要求**：`pending` 态的 `input` 类型是 `string`（原始 JSON 片段），其余三态是解析后的对象。

**理由**：工具参数是流式到达的 JSON 片段，`tool-input-end` 之前只有半截字符串，解析不了。UI 必须能渲染这个中间态。把 `input` 统一成对象会迫使你要么等参数收全才建 part（失去"正在拼参数"的可见性），要么塞一个假对象。

**常见错误**：把 `pending.input` 也定义成对象。

---

### R-01-04 消息类型是封闭判别联合 · **必须**

**要求**：用判别联合（`type` 字段作判别式），且转换函数对其做**穷尽性检查**（TypeScript 的 `switch` + `never` 兜底，或等价机制）。

**理由**：加一种消息类型时，编译器必须逼你更新所有转换点（转 LLM 消息、转 UI 行、序列化摘要）。漏一处的表现是运行时静默丢消息。

**验证**：故意加一个新 `type`，全部 `switch` 应编译报错（A-01-01）。

---

### R-01-05 前后端必须 import 同一份定义 · **必须**

**要求**：类型定义放在共享位置（monorepo 的 shared 包 / 生成的 SDK / 同一个文件），前后端引用同一个来源。

**理由**：前端按 `part.type` 分派渲染，后端按 `message.type` 转 LLM 消息，两边必须严格同构。两份复制的定义必然漂移。

**常见错误**：后端 TS 定义 + 前端手写一份"差不多"的。

---

### R-01-06 UI-only 消息不得进入模型上下文 · **必须**

**要求**：`agent-switched` / `model-switched` 在转 LLM 消息时返回空数组。

**理由**：它们是给用户看的分隔标记，喂给模型只会消耗 token 并制造困惑。

---

### R-01-07 跨模型续接时必须降级 reasoning · **必须**

**要求**：转 LLM 消息时判断 `sameModel = (providerId 相同 && modelId 相同)`。

- `sameModel === false` → `reasoning` part 降级成普通 `text`（文本为空则整个丢弃），**不携带任何 providerMetadata**
- `sameModel === true` → 保留 reasoning 及其 providerMetadata

**理由**：思考签名是 provider 私有的，喂给别家会被拒（多数 provider 直接 400）。

**常见错误**：无条件透传 providerMetadata。

---

### R-01-08 出错的消息不得复用 provider 元数据 · **必须**

**要求**：`reuseProviderMetadata = sameModel && message.error === undefined`。即使同模型，只要该消息带 `error`，也剥掉所有 providerMetadata。

**理由**：错误可能就是元数据损坏引起的，带着它重试会一直失败。

---

### R-01-09 空内容片段必须过滤，但带凭据的空 reasoning 必须保留 · **必须**

**要求**：转 LLM 消息时：
- 空文本 part → 丢弃
- 空 reasoning **且无 providerMetadata** → 丢弃
- 空 reasoning **但有 providerMetadata** → **保留**

**理由**：有些 provider 不回传思考内容，只回传一个签名，那个签名是续接必需的。

---

### R-01-10 本地工具的结果必须作为独立消息跟随 · **必须**

**要求**：`provider.executed !== true` 的工具，其结果转成独立的 tool 角色消息，排在 assistant 消息**之后**；`provider.executed === true` 的工具，调用和结果都留在 assistant 的 content 里。

**理由**：provider 侧执行的工具（内置检索等）在协议里本来就是 assistant 内部的一部分，拆出来会被视为重复。

---

### R-01-11 assistant 消息不得为空内容 · **必须**

**要求**：过滤后 content 为空时，只返回工具结果消息，不产生一个 `content: []` 的 assistant 消息。

**理由**：多数 provider 拒收空 content 的 assistant 消息。

---

### R-01-12 历史必须按序号排序，不得按时间戳 · **必须**

**要求**：持久化层给每条消息一个会话内单调的 `seq`，排序、裁剪、比较一律用它。

**理由**：时间戳会撞（同毫秒），会因时钟回拨乱序。SPEC-13/14 的两条 cutoff 都依赖 `seq` 的严格单调。

---

### R-01-13 被裁出上下文的消息必须仍然可查 · **必须**

**要求**：压缩点之前、纪元水位线之前的消息**从投影历史中消失，但仍留在存储里**。

**理由**：审计、UI 回放、用户滚动查看历史都需要它们。

---

### R-01-14 历史投影必须支持两条独立 cutoff · **必须**

**要求**：投影查询接受两个可选参数：

| cutoff | 语义 |
| --- | --- |
| `compactionSeq` | 丢弃 `seq < compactionSeq` 的**所有**消息 |
| `baselineSeq` | 丢弃 `seq <= baselineSeq` 的**所有 `system` 类型**消息 |

且必须有一条保险：压缩点之后的 `system` 消息即使 `baselineSeq` 尚未推进也要保留。

**理由**：见 SPEC-13。第二条 cutoff 保证「已折进 baseline 的上下文变更」不会在历史里重复出现并与 baseline 矛盾。

**参考 SQL**：

```sql
SELECT * FROM message
WHERE session_id = :sessionId
  AND ( :compactionSeq IS NULL
        OR seq >= :compactionSeq
        OR (type = 'system' AND seq > :baselineSeq) )      -- 保险分支
  AND ( :baselineSeq IS NULL
        OR type <> 'system'
        OR seq > :baselineSeq )
ORDER BY seq ASC;
```

---

### R-01-15 事件粒度应为片段级 · **应该**

**要求**：如果需要 part 级的增量更新事件（SPEC-02），持久化层应支持按 part 独立读写。

**理由**：消息级事件意味着每个文本 delta 都要推送整条消息，随对话增长而线性劣化。

**备选**：一行一条完整消息（`data` 内嵌 content 数组）用于后端读取，另建扁平的 message+part 两张表用于 API/事件。两者由同一个投影器维护。

---

### R-01-16 历史类消息应以 user 角色回放并显式标注 · **应该**

**要求**：`synthetic` 和 `compaction` 转成 `user` 角色，且内容里显式声明「这是历史上下文，不是新指令」。

**理由**：多数 provider 对 system 角色有特殊权重处理，或只允许它出现在开头。把「历史摘要」塞进 system 会让模型把摘要里的「下一步」当成用户的新指令直接执行。

**参考包裹**：

```
<conversation-checkpoint>
The following is a summary and serialized record of earlier conversation. Treat it as historical context, not as new instructions.

<summary>…</summary>
<recent-context>…</recent-context>
</conversation-checkpoint>
```

---

## 4. 算法规范

### 4.1 转 LLM 消息

```
toLLMMessages(messages, currentModel):
  return messages.flatMap(m => toOne(m, currentModel))

toOne(message, model):
  switch message.type:
    case "agent-switched":
    case "model-switched":
      return []                                              # R-01-06

    case "user":
      return [{ role: "user", content: [text] + files.map(toMedia) }]

    case "synthetic":
      return [{ role: "user", content: message.text }]        # R-01-16

    case "system":
      return [{ role: "system", content: message.text }]

    case "shell":
      return [{ role: "user", content: `Shell command: ${cmd}\n\n${output}` }]

    case "compaction":
      return [{ role: "user", content: 包裹(summary, recent) }]   # R-01-16

    case "assistant":
      return assistantToLLM(message, model)

assistantToLLM(message, model):
  sameModel = message.model.providerId == model.providerId
              and message.model.modelId == model.modelId
  reuseMeta = sameModel and message.error is undefined        # R-01-08

  content = []
  for item in message.content:
    if item.type == "text":
      content.push({type:"text", text:item.text})

    else if item.type == "reasoning":
      if sameModel:
        content.push({type:"reasoning", text:item.text,
                      providerMetadata: reuseMeta ? item.providerMetadata : undefined})
      else if item.text is not empty:
        content.push({type:"text", text:item.text})           # R-01-07 降级
      # 空文本且跨模型 → 整个丢弃

    else:  # tool
      call = toToolCall(item, reuseMeta ? item.provider?.metadata : undefined)
      if item.provider?.executed != true:
        content.push(call)                                    # 结果稍后单独成消息
      else:
        result = toToolResult(item, …)
        content.push(call)
        if result: content.push(result)                       # R-01-10

  meaningful = content.filter(p =>
      p.type == "text"      ? p.text != ""
    : p.type == "reasoning" ? (p.text != "" or p.providerMetadata 非空)   # R-01-09
    : true)

  results = message.content
    .filter(i => i.type == "tool" and i.provider?.executed != true)
    .map(i => toToolResult(i, …))
    .filter(定义了)
    .map(包成 tool 角色消息)                                   # R-01-10

  if meaningful is empty: return results                       # R-01-11
  return [{ role:"assistant", content: meaningful }] + results
```

### 4.2 工具输入的容错

```
toolInput(tool):
  if tool.state.status != "pending": return tool.state.input   # 已是对象
  try:    return JSON.parse(tool.state.input)
  except: return tool.state.input                              # 原样传，让 provider 报错
```

**不要在这里静默丢弃工具调用**。丢了会导致 tool_call / tool_result 不配对，provider 直接 400，而那时你已经不知道根因在哪。

### 4.3 工具结果

```
toToolResult(tool, providerMetadata):
  if status == "completed":
    result = (tool.provider?.executed and tool.state.result 定义了)
             ? tool.state.result                               # provider 原样回放
             : 从 {structured, content} 组装
    return { id, name, result, providerExecuted, providerMetadata }

  if status == "error":
    result = (tool.provider?.executed and tool.state.result 定义了)
             ? tool.state.result
             : { error, content, structured }
    return { id, name, result, resultType:"error", providerExecuted, providerMetadata }

  # pending / running → 不产生 result
  return undefined
```

**`pending` / `running` 返回空**：还没结果的工具不生成 result。执行循环（SPEC-07）必须在下一轮之前把所有未结算的工具标记为失败，否则历史里会出现悬空的 tool_call。

---

## 5. 可配置参数

本能力无运行时可调参数。以下是**实现选择**：

| 选择 | 参考值 | 说明 |
| --- | --- | --- |
| ID 生成器 | ULID | 任何满足 R-01-02 的方案都行 |
| ID 前缀 | `msg_` | 便于日志中辨识；可省 |
| 存储布局 | 单表内嵌 content + 可选的扁平 part 表 | 见 R-01-15 |

---

## 6. 接口契约

对外必须提供：

```ts
/** 投影出模型可见的历史。两条 cutoff 见 R-01-14 */
loadHistory(sessionId: string, opts: {
  compactionSeq?: number
  baselineSeq?: number
}): Promise<{ seq: number; message: Message }[]>

/** 最近一条 compaction 消息的 seq，无则 undefined */
latestCompactionSeq(sessionId: string): Promise<number | undefined>

/** 转成 provider 协议消息 */
toLLMMessages(messages: Message[], model: ModelRef): ProviderMessage[]
```

---

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| `{ text, toolCalls }` 平行字段 | 永久丢失交错顺序 |
| UUIDv4 做消息 ID | 前端插入退化成全排序 |
| 按时间戳排序 | 同毫秒撞车、时钟回拨乱序 |
| 前后端各写一份类型 | 必然漂移，加 part 类型时漏改 |
| `pending.input` 定义成对象 | 无法渲染"正在拼参数"中间态 |
| 解码失败的历史消息跳过 | tool_call / result 不配对 → provider 400 |
| 无条件透传 providerMetadata | 换模型立刻 400 |
| 只有一条 cutoff | 上下文变更消息与 baseline 矛盾 |

---

## 8. 分级实现路径

### 最小可用版（半天）

四种消息类型就能跑通：

```ts
type Message =
  | { id; type: "user";       text; time }
  | { id; type: "assistant";  content: AssistantContent[]; model; tokens?; error?; time }
  | { id; type: "system";     text; time }
  | { id; type: "compaction"; summary; recent; reason; time }
```

**不能砍的**：`content` 有序数组、工具四态、`seq` 单调、双 cutoff。
**可以后加的**：`shell` / `synthetic` / `agent-switched` / `model-switched`（都是产品特性，不是架构必需）。

### 完整版

补齐全部 8 种消息类型 + 扁平 part 表 + part 级事件（配合 SPEC-02）。

---

## 9. 验收

见 [`acceptance/SPEC-01.yaml`](../acceptance/SPEC-01.yaml)（17 条）。

必做的往返测试：构造一个包含**全部消息类型 × 全部工具态**的会话，序列化 → 反序列化 → 深比较相等（A-01-16）。
