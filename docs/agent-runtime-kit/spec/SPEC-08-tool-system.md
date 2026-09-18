# SPEC-08 · 工具系统

> **优先级** P1 · **工作量** L · **依赖** SPEC-01
> **分析原文** [ch14 工具系统](../../context-and-streaming/14-tools.md)
> **验收** [`acceptance/SPEC-08.yaml`](../acceptance/SPEC-08.yaml)（18 条）

---

## 1. 目标与非目标

### 目标

「给模型一组函数让它调用」要同时满足七件事：

| 需求 | 朴素做法的问题 |
| --- | --- |
| 模型要 JSON Schema | 手写 schema 和类型两份，必然漂移 |
| 输入要校验 | 模型会传畸形参数，不校验就在业务代码里崩 |
| 输出要给两个消费者 | 模型要文本、UI 要结构化，混在一起两边都难受 |
| 危险操作要授权 | 授权逻辑散落，规则无法统一配置 |
| 输出可能巨大 | 每个工具各自截断，行为不一致 |
| 要能被插件/MCP 扩展和覆盖 | 全局 Map 直接 set，卸载时无法恢复被覆盖的那个 |
| 定义要能按 agent 过滤 | 过滤与授权混在一起 → 「看不见但能调用」或「看得见但必失败」 |

### 非目标

- 不规定具体有哪些工具。
- 不规定 provider 的工具协议细节。
- 权限判定见 SPEC-09；文件安全见 SPEC-10；输出上限见 SPEC-11。

---

## 2. 领域模型

```ts
export interface ToolContext {
  sessionId: string
  agent: string
  assistantMessageId: string
  toolCallId: string
}

export type ToolContent =
  | { type: "text"; text: string }
  | { type: "file"; data: string; mime: string; name?: string }

export interface ToolOutput {
  structured: Record<string, unknown>
  content: ToolContent[]
}

/** 只允许抛这个；其它异常必须穿透 */
export class ToolFailure extends Error {}

/** 对外是不透明句柄，拿不到 execute / schema。见 R-08-01 */
export type Tool = { readonly __tool: unique symbol }

export interface ToolConfig<I, O, S = O> {
  description: string
  input: Codec<I>                    // { parse(u): I; jsonSchema(): object }
  output: Codec<O>
  structured?: Codec<S>
  toStructuredOutput?: (a: { input: I; output: O }) => S
  execute: (input: I, ctx: ToolContext) => Promise<O>
  toModelOutput?: (a: { input: I; output: O }) => ToolContent[]
}

export interface Materialization {
  definitions: ToolDefinition[]
  settle(input: { sessionId; agent; assistantMessageId; call: ToolCall }): Promise<Settlement>
}

export interface Settlement {
  result: { type: "value" | "error"; value: unknown }
  output?: ToolOutput
  outputPaths?: string[]
}
```

---

## 3. 规范条款

### R-08-01 工具值必须不可变且不可内省 · **必须**

**要求**：构造函数返回一个冻结的空对象，实现（`execute`、codec、定义派生、权限声明）存在旁路映射（`WeakMap` 或等价）里。调用方只能通过模块函数操作它。

**要求**：传入非工具值时抛出明确错误。

**理由**：防止调用方绕过结算管线直接调 `execute`（从而跳过输入校验、输出编码、上限治理）。

---

### R-08-02 必须有三份 schema · **必须**

| schema | 给谁 | 说明 |
| --- | --- | --- |
| `input` | 模型 | JSON Schema 进工具定义；结算时校验模型传的参数 |
| `output` | 内部 | 执行结果的完整领域类型，`toModelOutput` 的输入 |
| `structured`（可选） | UI / 历史 | `output` 的**瘦身投影** |

**理由**：典型场景——执行命令类工具的 `output` 含完整输出（可能几 MB），`structured` 只有 `{exit, truncated, timeout}`。UI 渲染卡片只需要后者，历史里存的也是后者。**大输出不会因为 UI 需要而被完整保留两份。**

---

### R-08-03 结算管线必须是固定五步 · **必须**

```
1. 校验并解析 input     失败 → ToolFailure("Invalid tool input: …")
2. execute(input, ctx)  只允许失败为 ToolFailure；中断/其它异常穿透
3. 编码 output          失败 → ToolFailure("Tool returned an invalid value for its output schema: …")
4. 若声明了 structured：编码 toStructuredOutput({input, output})；否则 structured = output
5. content = toModelOutput?.({input, output})
            ?? (typeof output === "string" ? [{type:"text", text: output}] : [])
返回 { structured, content }
```

**第 5 步的兜底是刻意的强制约定**：返回对象却不写 `toModelOutput` 的工具，**模型什么都看不到**。逼你显式决定给模型看什么。

**要求**：`file` 类 content 在这一步转成标准 URI（如 data URI），工具作者只写 `{type:"file", data, mime}`。

---

### R-08-04 只允许捕获工具失败，其它必须穿透 · **必须**

**要求**：结算层**只**捕获 `ToolFailure` 并转成错误结果回给模型。中断信号、其它异常、defect **必须**向上传播。

**理由**：把中断当成工具错误，执行循环就停不下来；把编程错误吞掉，问题会在下一轮以 provider 400 的形式出现且无从排查。

**常见错误**：用 `catch (e) { return error(e.message) }` 一网打尽。

---

### R-08-05 注册必须是作用域化的栈 · **必须**

**要求**：注册表结构是 `Map<name, Array<{token, registration}>>`：

- 同名注册**压栈**，生效的是栈顶
- 作用域关闭时**按 token 移除自己那条**，露出前一条
- 一批注册共享一个 token

**理由**：覆盖式 `Map.set` 在插件卸载时无法恢复被它覆盖的那个工具，会留下空洞。

---

### R-08-06 一批注册必须先全部校验再注册任何一个 · **必须**

**要求**：先校验全部名字合法，再执行注册；且注册与清理钩子登记之间不可被打断。

**理由**：一批要么全成功要么全失败，不留半截状态；被打断会泄漏一条无法回收的注册。

---

### R-08-07 工具名必须受限 · **必须**

**要求**：`/^[A-Za-z][A-Za-z0-9_-]{0,63}$/`（字母开头，长度 ≤ 64，只含字母数字下划线连字符）。

**理由**：多数 provider 对工具名有类似限制，不校验会在调用时才被 provider 拒绝。

---

### R-08-08 多层注册必须有确定的优先级 · **必须**

**要求**：存在多层注册来源时（如进程级应用注册 + 会话/位置级注册），明确规定谁覆盖谁，并在 materialize 时按该顺序叠加。

---

### R-08-09 必须有 materialize 快照 · **必须**

**要求**：每轮 provider 请求前把「本轮可见工具集」快照下来，之后本轮所有调用都对着这个快照。

**理由**：防止流式过程中注册表变动导致的错乱。

---

### R-08-10 必须做 stale 检测 · **必须**

**要求**：每个注册携带一个仅用于引用比较的身份对象。结算时重新查当前生效的注册，**引用不同**则返回「工具已过期」错误。

**要求**：区分两种错误文案：

| 情况 | 文案语义 |
| --- | --- |
| 本轮宣告过、但注册已被替换 | 「过期的工具调用」 |
| 从未注册过（模型捏造） | 「未知工具」 |

**理由**：工具在流式过程中被热替换（插件重载、MCP 重连）时，模型是按旧 schema 生成的参数，执行新工具很危险。两种文案让模型做不同的恢复。

---

### R-08-11 定义过滤与执行授权必须分离 · **必须**

**要求**：

- **定义过滤**（catalog visibility）：只把「被完全禁用」的整个工具从 `definitions` 里摘掉
- **执行授权**（execution authorization）：具体这次调用准不准，由工具内部在执行前判定

**「完全禁用」的精确判据**：最后一条匹配该权限动作的规则同时满足「资源为通配全部」且「效果为拒绝」。

**理由**：工具定义是**整体**的，没法给模型一个「只能读某目录」的 JSON Schema。所以：完全禁用 → 摘掉（省 token，也不会白试）；部分限制 → 保留定义，由叶子判定。

**要求**：注册表模块**不得**依赖权限服务（依赖图可验证）。

---

### R-08-12 允许多个工具共享一个权限动作 · **应该**

**要求**：提供一个装饰方式，让若干工具声明同一个权限动作（典型：所有写文件类工具共享一个 `edit` 动作）。

**理由**：用户配「禁止编辑」时应该一次禁掉全部写入类工具，而不是逐个列举。

---

### R-08-13 工具定义应按名字缓存 · **应该**

**要求**：JSON Schema 生成结果按注册名缓存。

**理由**：同一个工具值可能以不同名字注册（MCP 前缀、别名）。schema 生成不便宜，每轮 materialize 重算是浪费。

**要求**：仅在存在引用定义时才输出 `$defs`（空的 `$defs: {}` 在某些 provider 上会报错）。

---

### R-08-14 输出上限必须只在结算层统一施加 · **必须**

**要求**：模型可见输出的截断/落盘（SPEC-11）**只在结算层做一次**，工具内部不做。

**说明**：工具内部可以有**采集**上限（如命令执行的内存捕获上限），那是另一回事，两者不能混。

---

### R-08-15 叶子必须遵循统一的实现模板 · **应该**

见 §4.3。核心是：依赖在构造期取好、权限来源固定构造、授权顺序固定、只翻译预期错误。

---

## 4. 算法规范

### 4.1 构造

```
makeTool(config):
    tool = 冻结的空对象
    cache = {}
    runtimes.set(tool, {
        definition(name):
            if cache[name]: return cache[name]                      # R-08-13
            def = { name, description: config.description,
                    inputSchema: config.input.jsonSchema(),
                    outputSchema: (config.structured ?? config.output).jsonSchema() }
            cache[name] = def;  return def

        settle(call, ctx):                                          # R-08-03
            try:    input = config.input.parse(call.input)
            catch:  throw ToolFailure("Invalid tool input: …")

            output = await config.execute(input, ctx)                # ToolFailure 向上抛

            try:
                structured = config.structured && config.toStructuredOutput
                             ? config.structured.parse(config.toStructuredOutput({input, output}))
                             : output
            catch: throw ToolFailure("Tool returned an invalid value for its output schema: …")

            content = config.toModelOutput?.({input, output})
                      ?? (typeof output === "string" ? [{type:"text", text: output}] : [])
            return { structured, content }
    })
    return tool
```

### 4.2 注册与 materialize

```
register(tools):                                                    # R-08-05/06
    先校验全部名字                                                   # R-08-07
    不可中断段:
        token = {}
        for (name, tool) in tools:
            local[name] = (local[name] ?? []) + [{ token, registration: { identity: {}, tool } }]
        登记清理钩子: for name: local[name] = local[name].filter(r => r.token != token)
                                （空了就删 key）

materialize(permissionRules):                                       # R-08-09
    regs = Map(应用级注册)
    for (name, stack) in local:
        top = stack.last?.registration
        if top: regs.set(name, top)                                 # R-08-08 上层覆盖
    for (name, reg) in regs:
        if 完全禁用(permissionOf(reg.tool, name), permissionRules): regs.delete(name)   # R-08-11
    return {
      definitions: regs.map((reg, name) => definition(name, reg.tool)),
      settle: input => {
          reg = regs.get(input.call.name)
          if not reg: return { result: { type:"error", value: `Unknown tool: ${input.call.name}` } }
          return settleWith(input, reg.identity)                    # 带 identity 做 stale 检测
      },
    }

完全禁用(action, rules):
    rule = rules 中最后一条匹配 action 的
    return rule?.resource == "*" and rule.effect == "deny"

settleWith(input, advertised):                                      # R-08-10
    reg = 当前生效的注册(input.call.name)
    if not reg:
        return error(advertised ? "Stale tool call: …" : "Unknown tool: …")
    if advertised and reg.identity !== advertised:
        return error("Stale tool call: …")
    try:
        output = await settle(reg.tool, input.call, ctx)
    catch e:
        if e instanceof ToolFailure: return { result: { type:"error", value: e.message } }   # R-08-04
        throw e                                                     # 中断/其它穿透
    bounded = boundToolOutput(output)                               # R-08-14 → SPEC-11
    return { result: toResult(bounded.output), output: bounded.output, outputPaths: bounded.paths }
```

### 4.3 叶子实现模板

```
registerTool(name, makeTool({
    description,                       # 给模型看：参数语义 + 边界 + 默认值
    input, output, structured?, toStructuredOutput?, toModelOutput,
    execute: async (input, ctx) => {
        # ① 权限来源固定这样构造 —— 前端靠这两个 id 把弹窗锚定到具体工具卡片
        const source = { type: "tool", messageId: ctx.assistantMessageId, callId: ctx.toolCallId }

        # ② 先解析路径 → 再要外部访问授权 → 再要本工具授权（顺序不可颠倒，见 SPEC-10）
        const target = await resolvePath(input.path, { kind: "file" })
        if (target.external) await assertPermission({ ...externalPermission(target.external), ...ctx, source })
        await assertPermission({ action: name, resources: [target.resource], save: ["*"], ...ctx, source })

        # ③ 副作用
        return { ... }
    },
}))
# ④ 只把预期错误翻译成 ToolFailure，其余穿透
```

**依赖必须在构造期取好并被 executor 闭包捕获**，不要在 `execute` 里现取——那会让每次调用都过一遍依赖解析。

---

## 5. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 工具名长度上限 | `64` | 与 provider 限制对齐 | 超限会被 provider 拒 |
| 定义缓存 | 按名缓存 | — | 关闭会让每轮重算 schema |
| 共享权限动作 | 写文件类共享一个 | 按产品定 | 分得太细用户配置负担重 |

---

## 6. 接口契约

```ts
interface ToolRegistry {
  register(tools: Record<string, Tool>): Promise<{ unregister(): void }>
  materialize(rules?: PermissionRule[]): Promise<Materialization>
}

// 模块级函数（工具值不可内省，只能这样操作）
function toolDefinition(name: string, tool: Tool): ToolDefinition
function toolPermission(tool: Tool, fallbackName: string): string
function settleTool(tool: Tool, call: ToolCall, ctx: ToolContext): Promise<ToolOutput>
function withPermission(tool: Tool, action: string): Tool
```

---

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| 工具值可内省 | 调用方绕过结算管线，跳过校验与上限 |
| 只有 input/output 两份 schema | 大输出被完整存两份 |
| 返回对象却不写 `toModelOutput` 却期待模型看到 | 模型收到空内容（这是刻意设计，要显式决定） |
| `catch (e)` 一网打尽 | 中断变成工具错误，循环停不下来 |
| 覆盖式 `Map.set` 注册 | 插件卸载后留下空洞 |
| 先注册再校验名字 | 留下半批注册 |
| 注册表依赖权限服务 | 定义过滤与执行授权耦合 |
| 逐资源过滤工具定义 | 做不到（schema 是整体的），会导致看得见但必失败 |
| 不做 stale 检测 | 用新 schema 执行旧参数 |
| stale 与 unknown 同文案 | 模型无法区分该重试还是放弃 |
| 每轮重算 JSON Schema | 无谓 CPU |
| 空的 `$defs: {}` | 某些 provider 报错 |
| 叶子内部各自截断输出 | 行为不一致 |
| 授权顺序颠倒 | 二次弹窗 |

---

## 8. 分级实现路径

### 最小可用版（1 天）

`makeTool` + 单层注册表 + materialize + 五步结算 + 只捕获 `ToolFailure`。

**不能砍的**：input 校验、`ToolFailure` 与其它错误的区分、输出上限统一在结算层。

### 完整版

作用域注册栈 + stale 检测 + `structured` 投影 + 共享权限动作 + 定义缓存。

---

## 9. 验收

见 [`acceptance/SPEC-08.yaml`](../acceptance/SPEC-08.yaml)（18 条）。

最容易被跳过、也最重要的两条：`T5` 的两个对照断言（完全禁用摘掉 vs 部分限制保留，A-08-09/10），
以及 `T8` 的中断穿透（A-08-14/15）。
