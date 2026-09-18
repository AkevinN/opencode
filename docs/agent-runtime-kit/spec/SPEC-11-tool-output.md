# SPEC-11 · 工具输出治理

> **优先级** P1 · **工作量** S · **依赖** SPEC-08
> **分析原文** [ch06 工具输出治理](../../context-and-streaming/06-tool-output.md)
> **验收** [`acceptance/SPEC-11.yaml`](../acceptance/SPEC-11.yaml)（18 条）

**性价比最高的一项**：S 工作量，但直接决定长会话能不能活下来。

---

## 1. 目标与非目标

### 目标

coding agent 里最容易炸上下文的不是对话，是工具输出。一次读大文件、一次跑测试、一次全仓搜索就能吐出几十万字符。

三种朴素做法：

| 做法 | 问题 |
| --- | --- |
| 原样进历史 | 一次就吃掉半个窗口，后面全靠压缩救 |
| 简单截断前 N 字符 | **测试输出的结论在最后**（`3 failed, 42 passed`），截前面等于什么都没说 |
| 不给模型让它自己再查 | 模型看不到就会反复重试同一个命令 |

**答案**：头尾采样 + 中间插一条指向完整内容的标记。

### 非目标

- 不规定存储位置。
- 不替代压缩（SPEC-14）——两者分工见 §5。

---

## 2. 领域模型

```ts
export interface BoundInput {
  sessionId: string
  toolCallId: string
  output: ToolOutput
}
export interface BoundResult {
  output: ToolOutput
  /** 超限落盘后的完整内容位置；未超限时为空数组 */
  outputPaths: string[]
}
```

---

## 3. 规范条款

### R-11-01 必须同时有行数和字节数两个上限 · **必须**

**要求**：任一超限即触发采样 + 落盘。

**理由**：只看行数挡不住「一行 10MB 的压缩 JSON」；只看字节挡不住「50 万行短文本」。

---

### R-11-02 采样必须保头保尾 · **必须**

**要求**：超限时取开头若干 + 结尾若干，中间插入显式标记。

**要求**：头部份额用**上取整**、尾部用下取整（头部略多）。

**理由**：命令输出的上下文在开头，结论在结尾。只保头等于丢掉结论。

---

### R-11-03 必须两级采样 · **必须**

**要求**：先按行采样；若结果仍超字节上限，再按字节采样。

---

### R-11-04 字节截断必须按码点安全 · **必须**

**要求**：按字节截断时逐**码点**累加字节数，不得按 UTF-16 code unit 切。

**理由**：`slice(0, n)` 会把中文和 emoji 切成乱码，模型看到乱码会困惑，且可能破坏后续的 JSON 解析。

---

### R-11-05 标记本身必须计入预算 · **必须**

**要求**：从行数和字节预算里扣掉标记占用（标记字节数 + 前后空行）再做采样。

**要求**：预算小到放不下标记时，**退化成只输出标记**——至少让模型知道有个地方可以去读。

---

### R-11-06 媒体内容不得参与文本上限计算 · **必须**

**要求**：图片等媒体内容从上限判定中排除，不落盘，原样保留在输出里。

**理由**：把图片的 base64 算进字节数会误触发落盘，且落盘毫无意义。

---

### R-11-07 结构化输出不得被截断 · **必须**

**要求**：`structured` 字段永远完整保留。

**理由**：它是给程序读的（UI 渲染 diff、文件列表），通常很小且结构化，截断会直接破坏解析。

---

### R-11-08 无文本内容时必须用结构化数据参与判定 · **必须**

**要求**：`content` 为空的工具（只回结构化数据），用其 JSON 序列化结果参与上限判定。

**理由**：漏掉这条，结构化大输出不受任何限制。

**要求**：序列化失败（循环引用）要转成有意义的错误。

---

### R-11-09 落盘必须独占创建且文件名单调 · **必须**

**要求**：用「不覆盖已存在」的创建标志 + 单调递增的文件名。

**理由**：覆盖会让另一个工具调用的输出消失。

---

### R-11-10 落盘路径必须回传到消息 · **必须**

**要求**：`outputPaths` 一路传到工具的完成状态里，供 UI 提供「查看完整输出」入口，也供模型用读取工具去看。

---

### R-11-11 必须有保留期清理且全局单例 · **应该**

**要求**：周期性清理超过保留期的落盘文件；清理器**全局只跑一个**。

**要求**：只删自己创建的文件（按文件名前缀识别）；每一步失败都 catch 后继续。

**理由**：每个工作目录一个清理器会重复扫描、互相删文件。一个文件删不掉不能让整个循环挂掉。

---

### R-11-12 治理必须只在结算层做一次 · **必须**

见 SPEC-08 `R-08-14`。工具内部的采集上限是另一回事。

---

### R-11-13 上限应可配置 · **应该**

**要求**：行数和字节上限可通过配置覆盖，缺失时用默认值。

---

## 4. 算法规范

```
bound(input):
    limits = getLimits()                                           # R-11-13
    media  = input.output.content.filter(是媒体)                    # R-11-06
    text   = input.output.content.filter(是文本)
    contextual = input.output.content 为空
                 ? JSON.stringify(input.output.structured, null, 2)  # R-11-08
                 : text.map(t => t.text).join("")

    if lineCount(contextual) <= limits.maxLines
       and byteLength(contextual) <= limits.maxBytes:
        return { output: input.output, outputPaths: [] }            # 未超限，原样放行

    path   = writeExclusive(contextual)                            # R-11-09
    marker = `... output truncated; full content saved to ${path} ...`
    return {
      output: {
        structured: input.output.structured,                        # R-11-07 完整保留
        content: [{ type: "text", text: boundedPreview(contextual, marker, limits) }, ...media],
      },
      outputPaths: [path],                                          # R-11-10
    }

boundedPreview(text, marker, limits):                               # R-11-05
    markerBytes = byteLength(marker)
    if limits.maxLines <= 4 or limits.maxBytes <= markerBytes + 4:
        return 只放标记（自身也按上限截断）
    b = preview(text, limits.maxLines - 4, limits.maxBytes - markerBytes - 4)
    return b.tail ? `${b.head}\n\n${marker}\n\n${b.tail}` : `${b.head}\n\n${marker}`

preview(text, maxLines, maxBytes):                                  # R-11-02/03
    lines = text.split("\n")
    headLines = ceil(maxLines / 2);  tailLines = floor(maxLines / 2)
    sampled = lines.length <= maxLines ? text
              : lines[0..headLines].join("\n") + "\n" + lines[-tailLines..].join("\n")
    if byteLength(sampled) <= maxBytes:
        return lines.length <= maxLines ? { head: sampled, tail: "" }
                                        : { head: 头部行, tail: 尾部行 }
    return { head: takePrefixBytes(sampled, ceil(maxBytes/2)),      # R-11-04
             tail: takeSuffixBytes(sampled, floor(maxBytes/2)) }

takePrefixBytes(input, maxBytes):                                   # R-11-04 按码点
    bytes = 0;  out = ""
    for ch of input:                       # ← 按码点迭代，不是按 code unit
        size = byteLength(ch)
        if bytes + size > maxBytes: break
        out += ch;  bytes += size
    return out
# takeSuffixBytes 同理，从尾部反向累加
```

模型最终看到的形态：

```
<前 N 行>

... output truncated; full content saved to /abs/path/tool_01J8X… ...

<后 N 行>
```

---

## 5. 与压缩的分工

两处都在截断工具输出，**不要搞混**：

| | 本规范（结算时） | SPEC-14（压缩时） |
| --- | --- | --- |
| 何时 | 工具执行完，**写入历史之前** | 压缩时，序列化进摘要 |
| 上限量级 | 千行 / 几十 KB | 千**字符** |
| 策略 | 头尾采样 + 落盘 + 标记 | 简单取前 N 字符 |
| 目的 | 控制**单条**输出对上下文的占用 | 控制**摘要 prompt** 的长度 |

一条工具输出会**先后经过两次**。这不是重复——摘要要的是「做过什么」，历史要的是「结果是什么」。

---

## 6. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 行数上限 | `2000` | 200 – 5000 | 调大：单条输出占更多窗口；调小：更容易丢失中段信息 |
| 字节上限 | `50 KB` | 8 KB – 200 KB | 同上 |
| 保留期 | `7 天` | 1 – 30 天 | 调大：磁盘占用上升；调小：用户回看旧输出时文件已没 |
| 清理周期 | `1 小时` | 10 min – 24 h | — |
| 标记预留 | 4 行 + 标记字节 | 固定 | 见 R-11-05 |

---

## 7. 接口契约

```ts
interface ToolOutputStore {
  limits(): Promise<{ maxLines: number; maxBytes: number }>
  bound(input: BoundInput): Promise<BoundResult>
  cleanup(): Promise<void>
}
```

---

## 8. 反模式

| 反模式 | 后果 |
| --- | --- |
| 只截断前 N 字符 | 丢掉结论（测试输出最后一行） |
| 只看行数或只看字节 | 一行超长 / 多行短文本各挡不住一种 |
| 按 `slice` 截断 | 中文/emoji 乱码 |
| 标记不计入预算 | 实际输出超过上限 |
| 媒体计入字节判定 | 误触发落盘 |
| 截断 `structured` | 破坏 UI 解析 |
| `content` 为空时不判定 | 结构化大输出不受限 |
| 落盘用覆盖写 | 另一个调用的输出消失 |
| 每个工作目录一个清理器 | 重复扫描、互相删文件 |
| 清理时单个失败就中止 | 一个占用文件让整轮清理失效 |
| 与压缩的截断混为一谈 | 长输出在摘要里丢失中段 |

---

## 9. 分级实现路径

### 最小可用版（2 小时）

```ts
function boundToolOutput(text: string, save: (c: string) => string) {
  const lines = text.split("\n")
  if (lines.length <= MAX_LINES && byteLength(text) <= MAX_BYTES) return { text, paths: [] }
  const p = save(text)
  const marker = `... output truncated; full content saved to ${p} ...`
  const head = lines.slice(0, Math.ceil((MAX_LINES - 4) / 2)).join("\n")
  const tail = lines.slice(-Math.floor((MAX_LINES - 4) / 2)).join("\n")
  return { text: `${head}\n\n${marker}\n\n${tail}`, paths: [p] }
}
```

**砍掉字节级二次采样后**，遇到「行数不多但单行超长」会失效。如果你的工具可能产生超长单行（压缩 JSON、minified 代码），这一层不能砍。

### 完整版

两级采样 + 码点安全截断 + 媒体分离 + 结构化兜底 + 保留期清理。

---

## 10. 验收

见 [`acceptance/SPEC-11.yaml`](../acceptance/SPEC-11.yaml)（18 条）。

必做的两条：60KB 纯中文输出截断后无乱码且能正常 JSON 往返（A-11-06）；
边界值 `maxLines` 整 / `maxLines + 1` 的对照（A-11-01 ~ A-11-03）。
