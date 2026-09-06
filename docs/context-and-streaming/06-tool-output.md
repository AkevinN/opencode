# 06 · 工具输出的上下文治理：2000 行 / 50KB 上限与落盘

源码：`packages/core/src/tool-output-store.ts`（211 行，本章逐行讲完）

---

## 1. 解决什么问题

coding agent 里最容易炸上下文的不是对话，是工具输出。一次 `cat` 大文件、一次 `npm test`、一次 `grep -r` 就能吐出几十万字符。

朴素做法的问题：

| 做法 | 问题 |
| --- | --- |
| 原样进历史 | 一次就吃掉半个上下文窗口，后面全靠压缩救 |
| 简单截断前 N 字符 | 测试输出的**结论在最后**（`3 failed, 42 passed`），截前面等于什么都没说 |
| 不给模型，让它自己再查 | 模型看不到就会反复重试同一个命令 |

opencode 的答案：

> **头尾采样 + 中间插一条指向完整文件的标记。** 模型看到开头、结尾和"完整内容在 `/path/to/tool_xxx`"，需要细节时可以用 `read` 工具去看那个文件。

---

## 2. 概念模型与不变量

**Model Tool Output**：进入会话历史、会被回放给模型的那份有界投影。
**Managed Tool Output File**：完整输出落盘的临时文件，路径回写进 `state.outputPaths`。

不变量：

1. **T1** 双重上限：行数 **和** 字节数，任一超限就触发落盘。
2. **T2** 采样保头保尾，中间用显式标记说明被截断且完整内容在哪。
3. **T3** 媒体内容（图片等）**不参与**文本上限计算，也不落盘，原样保留。
4. **T4** 落盘用 `flag: "wx"`（独占创建），文件名单调递增，绝不覆盖。
5. **T5** 保留期 7 天，每小时清理一次，全局只跑一个清理器。
6. **T6** 上限可配置，配置缺失时用默认值。

---

## 3. 常量与类型

```ts
export const MAX_LINES = 2_000
export const MAX_BYTES = 50 * 1024
export const RETENTION = Duration.days(7)
export const MANAGED_DIRECTORY = "tool-output"

export interface BoundInput {
  readonly sessionID: SessionSchema.ID
  readonly toolCallID: string
  readonly output: ToolOutput
}
export interface BoundResult {
  readonly output: ToolOutput
  readonly outputPaths: ReadonlyArray<string>
}
export interface Interface {
  readonly limits: () => Effect.Effect<{ readonly maxLines: number; readonly maxBytes: number }>
  readonly bound: (input: BoundInput) => Effect.Effect<BoundResult, Error>
  readonly cleanup: () => Effect.Effect<void>
}
```
> `packages/core/src/tool-output-store.ts:13-47`

配置覆盖（T6）：

```ts
const limits = Effect.fn("ToolOutputStore.limits")(function* () {
  if (Option.isNone(config)) return { maxLines: MAX_LINES, maxBytes: MAX_BYTES }
  const entries = yield* config.value.entries().pipe(Effect.catch(() => Effect.succeed([] as Config.Entry[])))
  const configured = Object.assign({}, ...entries.flatMap((entry) => (entry.type === "document" ? [entry.info.tool_output ?? {}] : [])))
  return { maxLines: configured.max_lines ?? MAX_LINES, maxBytes: configured.max_bytes ?? MAX_BYTES }
})
```
> `tool-output-store.ts:119-127`

---

## 4. 核心算法

### 4.1 `bound()` 主流程

```ts
const bound = Effect.fn("ToolOutputStore.bound")(function* (input: BoundInput) {
  const outputLimits = yield* limits()
  const media = input.output.content.filter((item) => item.type === "file")     // T3
  const text = input.output.content.filter((item) => item.type === "text")
  const contextual =
    input.output.content.length === 0
      ? yield* Effect.try({
          try: () => JSON.stringify(input.output.structured, null, 2) ?? String(input.output.structured),
          catch: (cause) => new StorageError({ operation: "encode", cause }),
        })
      : text.map((item) => item.text).join("")

  if (lineCount(contextual) <= outputLimits.maxLines &&
      Buffer.byteLength(contextual, "utf-8") <= outputLimits.maxBytes)
    return { output: input.output, outputPaths: [] }                            // 不超限：原样放行

  const outputPath = yield* write(contextual)
  const marker = `... output truncated; full content saved to ${outputPath} ...`

  return {
    output: {
      structured: input.output.structured,                                      // structured 永远完整保留
      content: [
        { type: "text" as const, text: boundedPreview(contextual, marker, outputLimits.maxLines, outputLimits.maxBytes) },
        ...media,                                                               // T3：媒体原样跟在后面
      ],
    },
    outputPaths: [outputPath],
  }
})
```
> `tool-output-store.ts:138-172`

三个要点：

- **`content` 为空时退化到 `structured`**：有些工具只回结构化数据不回文本，也要参与上限计算。
- **`structured` 从不被截断**：它是给程序读的（UI 用它渲染 diff、文件列表），通常很小且结构化，截断会直接破坏解析。
- **媒体只过滤不计算**（T3）：图片是二进制引用，字节数和行数对它没意义。

### 4.2 `preview()`：头尾采样

```ts
const preview = (text: string, maxLines: number, maxBytes: number) => {
  const lines = text.split("\n")
  const headLines = Math.ceil(maxLines / 2)          // 上取整 → 头部略多
  const tailLines = Math.floor(maxLines / 2)
  const sampled =
    lines.length <= maxLines
      ? text
      : [lines.slice(0, headLines).join("\n"),
         ...(tailLines > 0 ? [lines.slice(lines.length - tailLines).join("\n")] : [])].join("\n")

  if (Buffer.byteLength(sampled, "utf-8") <= maxBytes) {
    return lines.length <= maxLines
      ? { head: sampled, tail: "" }
      : { head: lines.slice(0, headLines).join("\n"),
          tail: tailLines > 0 ? lines.slice(lines.length - tailLines).join("\n") : "" }
  }
  const headBytes = Math.ceil(maxBytes / 2)
  const tailBytes = Math.floor(maxBytes / 2)
  return { head: takePrefix(sampled, headBytes), tail: takeSuffix(sampled, tailBytes) }
}
```
> `tool-output-store.ts:74-96`

**两级采样**：先按行采样，还超字节再按字节采样。

字节采样是**按码点安全截断**的，不会切出半个 UTF-8 字符：

```ts
const takePrefix = (input: string, maximumBytes: number) => {
  let bytes = 0
  let content = ""
  for (const char of input) {                        // for...of 按码点迭代
    const size = Buffer.byteLength(char, "utf-8")
    if (bytes + size > maximumBytes) break
    content += char
    bytes += size
  }
  return content
}

const takeSuffix = (input: string, maximumBytes: number) => {
  let bytes = 0
  const content: string[] = []
  for (const char of Array.from(input).toReversed()) {
    const size = Buffer.byteLength(char, "utf-8")
    if (bytes + size > maximumBytes) break
    content.unshift(char)
    bytes += size
  }
  return content.join("")
}
```
> `tool-output-store.ts:50-72`

**用 `for...of` / `Array.from` 而不是 `slice(0, n)`** —— 后者按 UTF-16 code unit 切，会把 emoji 和中文切成乱码，模型看到乱码会困惑。

### 4.3 `boundedPreview()`：把标记也算进预算

```ts
const boundedPreview = (text: string, marker: string, maxLines: number, maxBytes: number) => {
  const markerOnly = takePrefix(marker, maxBytes).split("\n").slice(0, maxLines).join("\n")
  const markerBytes = Buffer.byteLength(marker, "utf-8")
  if (maxLines <= 4 || maxBytes <= markerBytes + 4) return markerOnly          // 预算太小：只放标记
  const bounded = preview(text, maxLines - 4, maxBytes - markerBytes - 4)      // 给标记留 4 行 + marker 字节
  return bounded.tail ? `${bounded.head}\n\n${marker}\n\n${bounded.tail}` : `${bounded.head}\n\n${marker}`
}
```
> `tool-output-store.ts:98-104`

**`- 4` 是给 `\n\n` × 2 留的位置**（标记前后各一个空行）。标记本身的字节也从预算里扣。极端小预算时退化成"只有标记"——至少模型知道有个文件可以去读。

最终模型看到的形态：

```
<前 998 行>

... output truncated; full content saved to /home/user/.local/share/opencode/tool-output/tool_01J8X... ...

<后 998 行>
```

### 4.4 落盘（T4）

```ts
const write = Effect.fn("ToolOutputStore.write")(function* (content: string) {
  const file = path.join(directory, `tool_${Identifier.ascending()}`)
  yield* fs.ensureDir(directory).pipe(Effect.mapError((cause) => new StorageError({ operation: "write", cause })))
  yield* fs.writeFileString(file, content, { flag: "wx" })
    .pipe(Effect.mapError((cause) => new StorageError({ operation: "write", cause })))
  return file
})
```
> `tool-output-store.ts:129-136`

- `flag: "wx"` = 独占创建，文件已存在则失败。配合单调递增文件名，**绝不覆盖已有输出**。
- 目录：`{global.data}/tool-output/`。

### 4.5 清理（T5）

```ts
const cleanup = Effect.fn("ToolOutputStore.cleanup")(function* () {
  const entries = yield* fs.readDirectory(directory).pipe(Effect.catch(() => Effect.succeed([])))
  const cutoff = Date.now() - Duration.toMillis(RETENTION)
  for (const entry of entries) {
    if (!entry.startsWith("tool_")) continue                     // 只删自己的文件
    const file = path.join(directory, entry)
    const info = yield* fs.stat(file).pipe(Effect.catch(() => Effect.void))
    const modified = info?.mtime.pipe(Option.map((date) => date.getTime()), Option.getOrElse(() => 0))
    if (modified !== undefined && modified < cutoff) yield* fs.remove(file).pipe(Effect.catch(() => Effect.void))
  }
})

/** Runs retention scanning once globally rather than once per active Location. */
export const cleanupLayer = Layer.effectDiscard(
  Effect.gen(function* () {
    const store = yield* Service
    yield* store.cleanup().pipe(Effect.repeat(Schedule.spaced(Duration.hours(1))), Effect.forkScoped)
  }),
)
```
> `tool-output-store.ts:176-207`

**每一步都 catch 后继续**——清理是尽力而为，一个文件删不掉不能让整个循环挂掉。**全局只跑一个**（`makeGlobalNode`），不是每个工作目录一个。

### 4.6 落盘路径怎么传到历史

`outputPaths` 从 `bound()` 出来后，经工具 settle 一路传到事件：

```ts
Effect.flatMap((settlement) =>
  publish(
    LLMEvent.toolResult({ id: event.id, name: event.name, result: settlement.result, output: settlement.output }),
    settlement.outputPaths ?? [],                              // ← 第二个参数
  ),
)
```
> `packages/core/src/session/runner/llm.ts:262-269`

```ts
yield* events.publish(SessionEvent.Tool.Success, {
  sessionID: input.sessionID, timestamp: yield* timestamp,
  assistantMessageID: tool.assistantMessageID, callID: event.id,
  ...result,
  outputPaths,                                                 // ← 落进 ToolStateCompleted.outputPaths
  ...(provider.executed ? { result: event.result } : {}),
  provider,
})
```
> `packages/core/src/session/runner/publish-llm-event.ts:364-372`

最终躺在 `ToolStateCompleted.outputPaths?: string[]`（`packages/schema/src/session-message.ts:101`），UI 可以据此提供「查看完整输出」入口。

---

## 5. 与压缩的分工

两处都在截断工具输出，别搞混：

| | 本章（ToolOutputStore） | 第 03 章（Compaction） |
| --- | --- | --- |
| 何时 | 工具刚执行完，**写入历史之前** | 压缩时，序列化进摘要 |
| 上限 | 2000 行 / 50KB | 2000 **字符** |
| 策略 | 头尾采样 + 落盘 + 标记 | 简单取前 2000 字符 + `[truncated]` |
| 目的 | 控制**单条**输出对上下文的占用 | 控制**摘要 prompt**的长度 |
| 代码 | `tool-output-store.ts:98` | `compaction.ts:85` |

一条工具输出会**先后经过两次**：写历史时被本章治理，被压缩时再被砍到 2000 字符。这不是重复——摘要要的是"做过什么"，2000 字符足够；历史要的是"结果是什么"，需要 50KB。

---

## 6. 边界情况与失败模式

| 场景 | opencode 怎么处理 | 你自己实现时必须处理 |
| --- | --- | --- |
| 输出含中文 / emoji，恰好在字节边界 | `takePrefix` / `takeSuffix` 按码点迭代 | `slice(0, n)` 会切出乱码 |
| 输出只有 structured 没有 text | 用 `JSON.stringify(structured, null, 2)` 参与计算 | 漏掉这条，结构化大输出不受任何限制 |
| 输出含图片 | 媒体不计入上限，不落盘，原样跟在文本后 | 把图片 base64 算进字节数会误触发落盘 |
| `maxLines` 配成 2 | `boundedPreview` 检测 `maxLines <= 4` → 只返回标记 | 不检测会算出负数切片 |
| 磁盘满 / 无写权限 | `StorageError{operation:"write"}` 向上传播，工具 settle 失败 | 别静默降级成"直接截断"——用户会永久失去那份输出 |
| 文件名冲突 | `flag: "wx"` 直接失败 | 用 `w` 会覆盖别的工具的输出 |
| 清理时某文件被占用 | `catch` 后跳过，继续下一个 | 抛错会中止整轮清理 |
| 目录里有非 `tool_` 开头的文件 | `continue` 跳过 | 别 `rm -rf` 整个目录 |
| 多个工作目录同时运行 | 清理器 `makeGlobalNode` 全局单例 | 每个目录一个清理器会重复扫描、互相删文件 |
| `structured` 序列化失败（循环引用） | `StorageError{operation:"encode"}` | `JSON.stringify` 抛错要转成有意义的错误 |

---

## 7. 移植到你自己的项目

### 7.1 最小可用版

```ts
const MAX_LINES = 2000, MAX_BYTES = 50 * 1024

function boundToolOutput(text: string, saveFile: (c: string) => string) {
  const lines = text.split("\n")
  if (lines.length <= MAX_LINES && Buffer.byteLength(text) <= MAX_BYTES) return { text, paths: [] }
  const p = saveFile(text)
  const marker = `... output truncated; full content saved to ${p} ...`
  const head = lines.slice(0, Math.ceil((MAX_LINES - 4) / 2)).join("\n")
  const tail = lines.slice(-Math.floor((MAX_LINES - 4) / 2)).join("\n")
  return { text: `${head}\n\n${marker}\n\n${tail}`, paths: [p] }
}
```

砍掉字节级二次采样后，遇到"行数不多但每行超长"（比如一行 10MB 的压缩 JSON）就会失效。**如果你的工具可能产生超长单行，字节采样不能砍。**

### 7.2 落地步骤

1. 定常量：`MAX_LINES = 2000`、`MAX_BYTES = 50 * 1024`、`RETENTION = 7 天`。
2. 实现 `takePrefix` / `takeSuffix`：**必须**用 `for...of` / `Array.from` 按码点走。
3. 实现 `preview(text, maxLines, maxBytes)`：先按行、再按字节的两级采样，头部 `ceil`、尾部 `floor`。
4. 实现 `boundedPreview(text, marker, maxLines, maxBytes)`：从预算里扣掉 marker 字节 + 4。
5. 实现 `write(content)`：`{data}/tool-output/tool_${ulid()}`，`flag: "wx"`，先 `mkdir -p`。
6. 实现 `bound(output)`：分离 media / text，`content` 为空则用 `structured` 的 JSON，判上限，超了就落盘 + 预览。
7. 把 `outputPaths` 一路传到 `ToolStateCompleted.outputPaths`，UI 加「查看完整输出」入口。
8. 起一个每小时的清理定时器，**全局单例**，逐文件 try/catch。

### 7.3 你需要自己提供的依赖

| 依赖 | 用途 | 建议 |
| --- | --- | --- |
| 数据目录 | 落盘位置 | `~/.local/share/<app>/tool-output`（遵循 XDG） |
| 单调 ID | 文件名 | `ulid()`，天然按时间排序，方便清理 |
| 定时器 | 保留期清理 | `setInterval(cleanup, 3600_000)`，进程级单例 |
| 配置系统 | 覆盖上限 | 可选，先硬编码 |

---

## 8. 验收清单

- [ ] **T1** 2001 行、10KB 的输出 → 触发落盘（行数超限）
- [ ] **T1** 100 行、51KB 的输出 → 触发落盘（字节超限）
- [ ] **T1** 2000 行、50KB 整 → **不**触发（边界是 `<=`）
- [ ] **T2** 截断后的文本同时包含原文首行和末行，中间有 marker
- [ ] **T2** marker 里的路径指向的文件内容 === 原始完整输出
- [ ] **UTF-8 安全** 输出是 60KB 纯中文 → 截断后无乱码字符，能正常 `JSON.parse` 往返
- [ ] **UTF-8 安全** 输出末尾是 emoji → `takeSuffix` 不切出半个代理对
- [ ] **T3** 输出含 1 张图 + 100 行文本 → 图片原样保留在 content 里，不计入字节判定
- [ ] **structured 完整** 超限时 `output.structured` 与输入完全相同
- [ ] **空 content** 只有 structured 的输出 → 用其 JSON 参与上限判定
- [ ] **T4** 连续 100 次落盘 → 100 个不同文件，无覆盖
- [ ] **T4** 手工预创建同名文件 → `wx` 失败并抛 `StorageError`
- [ ] **T5** 造一个 8 天前 mtime 的 `tool_xxx` → 清理后消失
- [ ] **T5** 造一个 6 天前的 → 保留
- [ ] **T5** 造一个 `other_file` → 永不被删
- [ ] **极小预算** `maxLines = 2` → 返回值只含 marker，不抛错
- [ ] **T6** 配置 `tool_output.max_lines = 500` → 501 行即触发
- [ ] **传导** 落盘后 `ToolStateCompleted.outputPaths` 含该路径，SSE 事件里也有
