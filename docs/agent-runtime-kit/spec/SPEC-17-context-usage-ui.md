# SPEC-17 · 上下文用量可视化

> **优先级** P3 · **工作量** S · **依赖** SPEC-01（消息模型 / token 用量字段）
> **分析原文** [ch12 上下文用量可视化](../../context-and-streaming/12-context-usage-ui.md)
> **验收** [`acceptance/SPEC-17.yaml`](../acceptance/SPEC-17.yaml)（15 条）

**全规范中投入产出比最高的一项**：不到 200 行、零框架依赖、半天完成，直接回答用户「为什么变贵了」和「为什么它忘了」。

---

## 1. 目标与非目标

### 目标

长会话产品最常收到的两个问题：

- 「为什么突然变慢了 / 变贵了？」
- 「为什么它把前面说的忘了？」

答案都在上下文用量里，但 provider 只回一个 `input_tokens` 总数，**不告诉你这些 token 是被 system prompt、用户消息、模型回答还是工具输出吃掉的**。

方案：**用字符数估算各类占比，再按 provider 报告的真实 `input` 归一化。** 结果是一条堆叠条 + 一张统计表，用户一眼看出「哦，78% 是工具输出」。

这个能力对 agent 类产品几乎是必需的——**工具输出爆炸是最常见的上下文杀手**，没有可视化用户根本不知道该收敛哪里。

### 非目标

- 不做精确 tokenization。精确需要 provider 侧 tokenizer，成本高、跨 provider 不一致，而归一化已经吸收了系统性偏差。
- 不做历史趋势图（每轮用量随时间变化）。可以加，但不在本规范范围。
- 不负责压缩（SPEC-14）。可选联动：占用率高时提示用户手动压缩。

---

## 2. 领域模型

```ts
export type BreakdownKey = "system" | "user" | "assistant" | "tool" | "other"

export interface BreakdownSegment {
  key: BreakdownKey
  tokens: number
  width: number      // 百分比，未取整 —— 给 CSS width 用
  percent: number    // 百分比，保留一位小数 —— 给显示用
}

export interface ContextMetrics {
  message: AssistantMessage     // 取样的那条消息
  providerLabel: string         // 目录查不到时回退为 id
  modelLabel: string
  limit: number | undefined     // 模型窗口上限
  input: number                 // provider 报告的输入 token
  total: number                 // input + output + reasoning + cache.read + cache.write
  usage: number | null          // 占用百分比；limit 缺失时为 null
}
```

---

## 3. 规范条款

### R-17-01 总用量取最后一条有 token 记录的 assistant 消息 · **必须**

**要求**：从尾往前找第一条 `role === "assistant"` 且 `tokenTotal(msg) > 0` 的消息。**不得**累加所有消息。

**理由**：每一轮的 `input` 已经包含了整个历史。累加会得到一个几倍于窗口的荒谬数字。**最后一轮的 `input` 就是「当前上下文有多满」。**

**`tokenTotal` 的定义**：

```
tokenTotal(msg) = tokens.input + tokens.output + tokens.reasoning
                + tokens.cache.read + tokens.cache.write
```

反映「这一轮总共经手了多少 token」，用于算窗口占用率。**堆叠条用的是 `input`（纯输入侧），不是 `total`。**

---

### R-17-02 跳过没有 usage 的消息 · **必须**

**要求**：`tokenTotal <= 0` 的 assistant 消息（失败 / 中断的）跳过，继续往前找。

**理由**：否则最后一次失败的请求会让面板显示 0 token。

---

### R-17-03 各段 token 之和必须精确等于 `input` · **必须**

**要求**：分类占比是估算，但**总和必须精确等于** provider 报告的 `input`。

**理由**：条形图的宽度加起来永远是 100%，且各段比例反映真实结构。用户不会看到「加起来 137%」这种破绽。

---

### R-17-04 估算超过实际时按比例缩放，不足时差额归 `other` · **必须**

**要求**：

| 情况 | 处理 | `other` 的语义 |
| --- | --- | --- |
| 估算 ≤ 实际 | 各项保持估算值 | **没算到的部分**：工具定义 schema、消息元数据、provider 自己加的包装、tokenizer 的实际开销 |
| 估算 > 实际 | 各项乘以 `input / estimated`，向下取整 | **只是取整余数**（很小） |

缩放后用 `Math.max(0, input - total)` 兜底，保证 `other` 非负。

**理由**：`chars / 4` 对中文会严重低估字符→token 比（一个汉字通常 ≈ 1 token 但只占 1 字符），所以中文场景下估算超过实际很常见。缩放路径不是边缘情况。

---

### R-17-05 零值分类不显示 · **必须**

**要求**：`tokens === 0` 的分类既不出现在条形图里，也不出现在图例里。

**理由**：图例里出现 `system 0%` 是噪音。

---

### R-17-06 缺失数据显示 `—` · **必须**

**要求**：`undefined` / `null` 统一格式化为 `—`（em dash），**不得**显示 `0`、`NaN%`、`Infinity%` 或 `Unknown`。

**理由**：`0` 和「没数据」是两件不同的事，混在一起用户无法判断。

---

### R-17-07 `input === 0` 时返回空数组 · **必须**

**要求**：入口处先判 `if (!input) return []`。

**理由**：后续全部是除以 `input` 的运算，不判会得到一片 `NaN`。

---

### R-17-08 provider / model 标签查不到时回退为 id · **必须**

**要求**：目录里查不到就显示 id，**不要显示 "Unknown"**。

**理由**：id 至少是可查的信息。"Unknown" 是纯粹的信息损失。

---

### R-17-09 `limit` 缺失时占用率为 `null` · **必须**

**要求**：模型不在目录里 → `limit` 为 `undefined` → `usage` 为 `null` → 显示 `—`。**不得**猜一个默认窗口大小。

**理由**：猜错的百分比比没有百分比更有害——用户会据此做决策。

---

### R-17-10 `width` 不取整、`percent` 取一位小数 · **必须**

**要求**：两个百分比字段分开算：

```
width   = (tokens / input) * 100                          // CSS 用，全精度
percent = Math.round((tokens / input) * 100 * 10) / 10    // 显示用，一位小数
```

**理由**：取整的 width 累加会有误差，导致条形图右侧留白。

---

### R-17-11 pending 状态的工具用 `raw` 长度 · **必须**

**要求**：工具参数还没解析完时（`status === "pending"`）用 `state.raw.length`，**不得**访问 `state.output`。

**理由**：那个字段此时是 `undefined`，会让整个统计抛错或算出 `NaN`。

**完整的状态分支**：

| status | 计入 tool 的字符数 |
| --- | --- |
| `pending` | `inputChars + state.raw.length` |
| `running` | `inputChars` |
| `completed` | `inputChars + state.output.length` |
| `error` | `inputChars + state.error.length` |

---

### R-17-12 重算只依赖标量，不依赖 parts 内容 · **必须**

**要求**：memo 的依赖列表只有四个标量：

```
[取样消息的 id, 它的 input, 消息总数, systemPrompt]
```

**明确不依赖 `parts` 本身。**

**理由**：否则流式期间每帧都要遍历全部消息做字符统计——那是 O(全部内容) 的操作，500 条消息的会话会直接卡死。

**React 注意**：`exhaustive-deps` lint 会警告缺少 `messages` / `parts`。**必须显式禁用并写注释说明这是刻意的性能取舍**，否则下一个人会"修好"它。

---

### R-17-13 配色复用语法高亮调色板 · **应该**

**要求**：

```ts
const COLOR: Record<BreakdownKey, string> = {
  system:    "var(--syntax-info)",
  user:      "var(--syntax-success)",
  assistant: "var(--syntax-property)",
  tool:      "var(--syntax-warning)",     // ← 工具输出是最该警惕的一类
  other:     "var(--syntax-comment)",
}
```

**理由**：自动跟随主题切换、与代码块视觉一致、深浅色模式的对比度都已验证过。`tool` 用 warning 色本身就是个信号。

---

### R-17-14 堆叠条必须有无障碍标签 · **必须**

**要求**：`role="img"` + `aria-label` 描述整条的构成（如 `"system 12%, user 5%, tool 78%, other 5%"`）。

**理由**：否则屏幕阅读器读到的是一串空 div。

---

### R-17-15 数字走本地化格式 · **应该**

**要求**：所有数字用 `toLocaleString(locale)`。

**理由**：中文环境下没有千分位的六位数字很难读。

---

## 4. 算法规范

### 4.1 取样

```
tokenTotal(msg) = msg.tokens.input + msg.tokens.output + msg.tokens.reasoning
                + msg.tokens.cache.read + msg.tokens.cache.write

lastAssistantWithTokens(messages):
    for i from messages.length-1 down to 0:
        msg = messages[i]
        if msg.role != "assistant": continue
        if tokenTotal(msg) <= 0: continue                  # R-17-02
        return msg
    return undefined

buildMetrics(messages, providers):
    message = lastAssistantWithTokens(messages)            # R-17-01
    if !message: return undefined                          # 空态
    provider = providers.find(p => p.id == message.providerID)
    model    = provider?.models[message.modelID]
    limit    = model?.limit.context
    total    = tokenTotal(message)
    return {
        message,
        providerLabel: provider?.name ?? message.providerID,    # R-17-08
        modelLabel:    model?.name    ?? message.modelID,
        limit,
        input: message.tokens.input,
        total,
        usage: limit ? round((total / limit) * 100) : null,     # R-17-09
    }
```

### 4.2 字符数估算

```
estimateTokens(chars) = ceil(chars / 4)

charsFromUserPart(part):
    text  → part.text.length
    file  → part.source?.text.value.length ?? 0
    agent → part.source?.value.length ?? 0
    其它  → 0

charsFromAssistantPart(part) -> { assistant, tool }:
    text      → { assistant: part.text.length, tool: 0 }
    reasoning → { assistant: part.text.length, tool: 0 }
    非 tool   → { assistant: 0, tool: 0 }
    tool:
        inputChars = Object.keys(part.state.input ?? {}).length * 16    # ← 键数 × 16 粗估
        pending    → { assistant: 0, tool: inputChars + state.raw.length }     # R-17-11
        completed  → { assistant: 0, tool: inputChars + state.output.length }
        error      → { assistant: 0, tool: inputChars + state.error.length }
        running    → { assistant: 0, tool: inputChars }
```

**三个约定**：

- **`chars / 4`** 是英文文本的经验值。中文会低估，但 R-17-04 的归一化会吸收系统性偏差。
- **工具参数按「键数 × 16」估**，不按 JSON 字符串长度。理由：参数的 key 名和结构本身有固定开销，而值的长度差异极大；这个粗估在参数不大时够用（值很大时会落到 `output` 那一侧）。
- **reasoning 算在 assistant 里**。可以拆出来单独一类，看你的产品要不要强调思考成本。

### 4.3 归一化（本模块的精髓）

```
estimateContextBreakdown({ messages, parts, input, systemPrompt }):
    if !input: return []                                          # R-17-07

    counts = { system: systemPrompt?.length ?? 0, user: 0, assistant: 0, tool: 0 }
    for msg in messages:
        p = parts[msg.id] ?? []
        if msg.role == "user":
            counts.user += sum(p.map(charsFromUserPart))
        else if msg.role == "assistant":
            for part in p:
                n = charsFromAssistantPart(part)
                counts.assistant += n.assistant
                counts.tool      += n.tool

    tokens = mapValues(counts, estimateTokens)
    estimated = tokens.system + tokens.user + tokens.assistant + tokens.tool

    if estimated <= input:
        return build({ ...tokens, other: input - estimated }, input)      # R-17-04 差额归 other

    scale  = input / estimated                                            # R-17-04 按比例缩放
    scaled = mapValues(tokens, t => floor(t * scale))
    total  = scaled.system + scaled.user + scaled.assistant + scaled.tool
    return build({ ...scaled, other: max(0, input - total) }, input)      # 取整余数归 other

build(t, input):
    return ["system", "user", "assistant", "tool", "other"]
        .map(key => ({ key, tokens: t[key] }))
        .filter(x => x.tokens > 0)                                        # R-17-05
        .map(x => ({
            key:     x.key,
            tokens:  x.tokens,
            width:   (x.tokens / input) * 100,                            # R-17-10 不取整
            percent: round((x.tokens / input) * 100 * 10) / 10,           # 一位小数
        }))
```

### 4.4 格式化

```
formatter(locale):
    number(v)  = v == null ? "—" : v.toLocaleString(locale)               # R-17-06 / R-17-15
    percent(v) = v == null ? "—" : v.toLocaleString(locale) + "%"
    time(v)    = !v ? "—" : formatDateTime(v, locale)
```

### 4.5 重算时机

```
breakdown = memo(
    deps = [metrics?.message.id, metrics?.input, messages.length, systemPrompt],   # R-17-12
    compute = () => estimateContextBreakdown({ messages, parts,
                                              input: metrics?.input ?? 0, systemPrompt })
)
```

React：

```ts
// eslint-disable-next-line react-hooks/exhaustive-deps -- 刻意不依赖 messages/parts：
// 流式期间每帧重算是 O(全部内容)，500 条消息的会话会卡死。只在取样消息切换时重算。
const breakdown = useMemo(
  () => estimateContextBreakdown({ messages, parts, input: ctx?.input ?? 0, systemPrompt }),
  [ctx?.message.id, ctx?.input, messages.length, systemPrompt],
)
```

---

## 5. 视觉规格

### 5.1 结构

```
┌──────────────────────────────────────────────────────┐
│ ████ ██ ████████████████████████████████████ ███     │  ← 堆叠条，高 8px，圆角 4px
└──────────────────────────────────────────────────────┘
 ■ System 12.3%   ■ User 4.1%   ■ Tool 78.2%   ■ Other 5.4%    ← 图例，flex-wrap
```

图例用 `flex-wrap`——窄面板下自动换行，**不做水平滚动**。

### 5.2 统计表（16 行）

| 行 | 值 |
| --- | --- |
| 会话 | 标题（回退为 id） |
| 消息数 | 本地化数字 |
| Provider | `providerLabel` |
| 模型 | `modelLabel` |
| 窗口上限 | `number(limit)` |
| 总 token | `number(total)` |
| 占用率 | `percent(usage)` |
| 输入 token | `number(input)` |
| 输出 token | `number(tokens.output)` |
| 推理 token | `number(tokens.reasoning)` |
| 缓存读/写 | `"1,234 / 567"`（**读在前**，一眼看出命中率） |
| 用户消息数 | 本地化数字 |
| 助手消息数 | 本地化数字 |
| 总成本 | 货币格式 |
| 会话创建时间 | `time(...)` |
| 最后活动时间 | `time(...)` |

### 5.3 可选联动

`usage > 80%` 时给条形图加警示态，并提示用户可以手动压缩（SPEC-14）。

---

## 6. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 字符→token 比 | `chars / 4` | 2–6，或接真 tokenizer | 主要影响缩放前的相对比例；归一化会吸收绝对偏差。中文为主的产品可试 `/2` |
| 工具参数估算 | 键数 × 16 | 8–32，或改用 JSON 长度 | 参数普遍很大时改用 JSON 长度更准 |
| reasoning 归属 | 并入 assistant | 可拆为独立分类 | 拆出来能让用户看见思考成本 |
| 条形高度 | 8px | 4–16px | — |
| 警示阈值 | 80% | 60–90% | 调低会过早打扰用户 |
| 百分比精度 | 1 位小数 | 0–2 位 | 0 位在小分类上会全显示 0% |

---

## 7. 边界情况

| 场景 | 正确处理 | 错误处理的后果 |
| --- | --- | --- |
| 会话还没有 assistant 消息 | 返回 `undefined`，面板显示空态 | 渲染出一堆 `NaN%` |
| 最后一条 assistant 失败（无 usage） | 跳过，往前找 | 显示 0 token |
| 模型不在目录里 | `usage` 为 `null` → 显示 `—` | 算出 `Infinity%` |
| `input` 为 0 | 返回空数组 | 除零得到 `NaN` |
| 估算超过实际（中文常见） | 按比例缩放 | 条形图超过 100% |
| 缩放取整后有余数 | 归 `other`，`max(0, ...)` 兜底 | 条形图右侧留白 |
| 某分类为 0 | 过滤掉 | 图例里出现 `0%` 的项 |
| 流式期间频繁重算 | memo 只依赖 4 个标量 | 每帧遍历全部消息，长会话卡死 |
| 工具还在 pending | 用 `state.raw.length` | 访问 `state.output` 得 undefined |
| 数字本地化 | 全部走 `toLocaleString` | 中文环境下没有千分位 |

---

## 8. 反模式

| 反模式 | 后果 |
| --- | --- |
| 累加所有消息的 input | 得到几倍于窗口的荒谬数字 |
| 不归一化，直接显示估算值 | 加起来 137%，用户不再信任这个面板 |
| `limit` 缺失时猜一个默认窗口 | 用户据错误百分比做决策 |
| 缺失数据显示 `0` | 与真实的 0 无法区分 |
| 目录查不到显示 "Unknown" | 纯粹的信息损失 |
| `width` 取整 | 条形图右侧留白 |
| memo 依赖 parts | 流式期间每帧 O(全部内容)，长会话卡死 |
| 只加 `// eslint-disable` 不写原因 | 下一个人会"修好"它，性能回归 |
| 自定义一套配色 | 主题切换失效，深色模式对比度不达标 |
| 堆叠条没有 aria-label | 屏幕阅读器读到一串空 div |

---

## 9. 分级实现路径

### 最小可用版（半天）

`estimateContextBreakdown` + `buildMetrics` + 堆叠条 + 图例。约 200 行，**零框架依赖**，核心函数可以逐字复制。

可以先砍掉：统计表（只留条形图已经能回答大部分问题）、本地化格式（先用 `toLocaleString()` 无参数版）、警示联动。

**绝对不能砍**：归一化（R-17-03/04）、memo 依赖裁剪（R-17-12）、`—` 兜底（R-17-06）。

### 完整版

补齐 16 行统计表、本地化、无障碍、压缩联动。

### 落地步骤

1. 复制 `estimateContextBreakdown`（§4.3）。
2. 实现 `buildMetrics`：从后往前找最后一条有 token 的 assistant 消息（§4.1）。
3. 实现格式化器：`—` 兜底 + `toLocaleString(locale)`（§4.4）。
4. 渲染堆叠条 + 图例 + 统计表。配色复用你的语法高亮调色板。
5. **memo 只依赖 4 个标量**，加 lint 禁用注释说明这是刻意的。
6. 可选：`usage > 80%` 时加警示态 + 手动压缩入口。

### 你需要自己提供的依赖

| 依赖 | 用途 | 备注 |
| --- | --- | --- |
| 模型目录（context limit） | 算占用百分比 | 缺失时降级为「不显示百分比」，**不要猜** |
| system prompt 文本 | 估算 system 段 | 拿不到就传 `undefined`，差额自动进 `other` |
| 语法高亮调色板 | 配色 | 或自定义 5 个色，注意深浅色对比度 |

---

## 10. 验收

见 [`acceptance/SPEC-17.yaml`](../acceptance/SPEC-17.yaml)（15 条）。

**必须优先验证**：
- 各段 `tokens` 之和 === provider 报告的 `input`（A-17-05）——本规范的核心不变量。
- 灌入大量中文让估算超过实际 → 各项按比例缩小，总和仍等于 `input`（A-17-07）。
- 500 条消息的会话流式期间 breakdown 不在每帧重算（A-17-13，打点验证）。
