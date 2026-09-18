# SPEC-09 · 权限与授权

> **优先级** P1 · **工作量** M · **依赖** SPEC-02, SPEC-08
> **分析原文** [ch14 §5 权限系统](../../context-and-streaming/14-tools.md)
> **验收** [`acceptance/SPEC-09.yaml`](../acceptance/SPEC-09.yaml)（10 条）

**即使是单用户本地应用也建议实现**——它同时是「操作确认」机制，不只是安全机制。

---

## 1. 目标与非目标

### 目标

让危险操作可控，**同时不把用户烦死**。这两个目标的平衡点全在 §3 的 `R-09-08`（资源与记忆分离）。

### 非目标

- 不做用户/角色体系（本规范是「单用户对 agent 的授权」）。
- 不做网络层鉴权。

---

## 2. 领域模型

```ts
export type Effect = "allow" | "deny" | "ask"

export interface Rule {
  action: string          // 权限动作，通常等于工具名
  resource: string        // 支持通配符
  effect: Effect
}

export interface PermissionRequest {
  id: string
  sessionId: string
  action: string
  /** 本次要授权的具体资源 */
  resources: string[]
  /** 用户点「总是」时要记住的模式。与 resources 故意不同，见 R-09-08 */
  save?: string[]
  metadata?: Record<string, unknown>
  /** 前端靠它把弹窗锚定到具体工具卡片 */
  source?: { type: "tool"; messageId: string; callId: string }
}

export type Reply = "once" | "always" | "reject"

/** 纯拒绝：必须是可识别的特殊类型，让执行循环能中断 */
export class DeclinedError extends Error {}
/** 拒绝并留言：变成工具错误文本回给模型 */
export class CorrectedError extends Error { constructor(readonly feedback: string) { super() } }
/** 规则直接拒绝 */
export class BlockedError extends Error { constructor(readonly rules: Rule[]) { super() } }
```

存储（保存的决策）：

```sql
CREATE TABLE saved_permission (
  id         TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,          -- 作用域边界，见 R-09-07
  action     TEXT NOT NULL,
  resource   TEXT NOT NULL
);
```

---

## 3. 规范条款

### R-09-01 规则求值必须是「最后匹配者赢」且默认为询问 · **必须**

**要求**：

```
evaluate(action, resource, ...rulesets):
    return rulesets.flat()
             .findLast(r => match(action, r.action) && match(resource, r.resource))
           ?? { action, resource: "*", effect: "ask" }
```

**理由**：
- `findLast` 让后定义的规则覆盖先定义的，符合「配置文件从通用到特殊」的书写习惯。
- **默认 `ask` 是唯一安全的默认值**。默认 allow 等于没有权限系统；默认 deny 会让未配置的工具全部不可用。

---

### R-09-02 通配符匹配必须处理三个细节 · **必须**

**要求**：

1. **路径分隔符归一化**：反斜杠统一成正斜杠后再匹配（Windows 路径与 glob 模式互通）
2. **尾部空格通配的特判**：模式 `xxx *` 必须同时匹配裸 `xxx`（无参数形式）
3. **Windows 上大小写不敏感**

**理由**：
- 不做第 2 条，用户保存的「允许 `git commit *`」对裸 `git commit` 失效，会再弹一次窗。
- 不做第 1 条，Windows 上所有路径规则都失效。

**要求**：转义正则元字符后再把 `*` 转 `.*`、`?` 转 `.`。

---

### R-09-03 agent 层的拒绝必须先单独判定且不可被保存规则覆盖 · **必须**

**要求**：判定分两步：

```
① 只用 agent 配置的规则判一次；任一资源 deny → 直接 deny，结束
② 叠加用户保存的规则，再判一次
```

**理由**：**这是整个权限系统的安全底线。** 不分两步的话，用户点一次「总是允许 rm -rf *」就能把 agent 配置里的硬约束彻底绕过。

---

### R-09-04 保存的决策只能是允许 · **必须**

**要求**：持久化的用户决策在转成规则时，效果**硬编码为 `allow`**。

**理由**：记忆只能放宽，不能收紧——收紧应该改 agent 配置。允许保存 `deny` 会产生一个绕过配置的隐藏状态。

---

### R-09-05 多资源必须取最保守结果 · **必须**

**要求**：一次授权请求带多个资源时：任一 deny → deny；任一 ask → ask；全 allow 才 allow。

---

### R-09-06 找不到 agent 时必须全部拒绝 · **必须**

**要求**：兜底规则集为 `[{action: "*", resource: "*", effect: "deny"}]`。

**理由**：兜底成 allow 是灾难性的默认值。

---

### R-09-07 保存的决策必须按项目作用域隔离 · **必须**

**理由**：在 A 项目允许的危险命令不应泄漏到 B 项目。

---

### R-09-08 授权资源与记忆模式必须分离 · **必须**

**要求**：授权请求携带两个数组：

| 字段 | 含义 |
| --- | --- |
| `resources` | 本次要授权的**具体**资源 |
| `save` | 用户点「总是」时要记住的**模式** |

**参考策略**（按你的产品调整，但分离这件事不能省）：

| 工具类 | `resources` | `save` | 效果 |
| --- | --- | --- | --- |
| 读文件 | 具体路径 | `["*"]` | 批准一次 → 读所有文件都不再问 |
| 写文件 | 具体路径 | `["*"]` | 同上 |
| 执行命令 | 具体命令 | 具体命令 | 逐条批准 |
| 加载技能 | 技能名 | 技能名 | 只记住这一个 |
| 外部目录访问 | 目录 + 通配 | 同 | 记住整个目录 |

**理由**：**这是「用户体验」和「安全」的平衡点，也是本规范最需要按你自己产品调的一处。**
读文件是低风险高频操作，问一次就该记住全部；执行命令是高风险，只记住具体那条。
不分离的话，要么烦死用户（每个文件都问），要么一次点击就把整个沙箱打开了。

---

### R-09-09 拒绝必须在会话内传染 · **必须**

**要求**：用户拒绝一个请求时，**同会话所有待处理的授权请求一并拒绝**。

**理由**：模型一次调了 5 个工具，用户说「不」的时候意思是全停，而不是想点 5 次「不」。

---

### R-09-10 「总是允许」必须触发重新判定 · **必须**

**要求**：保存规则后，遍历所有待处理请求，对每个**重新跑一遍完整判定**（含 agent deny 检查），满足 allow 的自动放行。

**理由**：不重新判定而是简单放行同 action 的请求，会绕过 `R-09-03` 的安全底线。

---

### R-09-11 纯拒绝与带留言拒绝必须语义不同 · **必须**

| 拒绝方式 | 错误类型 | 执行循环的反应 |
| --- | --- | --- |
| 纯拒绝 | `DeclinedError`（**可识别的特殊类型**） | **中断整个执行循环** |
| 拒绝并留言 | `CorrectedError` | 变成工具错误文本回给模型 |

**理由**：纯拒绝喂回模型，它会换个说法再试一次（见 SPEC-07 `R-07-13`）。

**实现提示**：`DeclinedError` 要用执行循环能检测到的方式传播（不是普通的业务失败）。

---

### R-09-12 授权请求必须携带来源以便 UI 锚定 · **应该**

**要求**：`source: { type: "tool", messageId, callId }`，且叶子工具**统一这样构造**。

**理由**：前端靠这两个 id 把权限弹窗锚定到具体那张工具卡片上，而不是弹一个孤立的全局对话框。

---

### R-09-13 待处理请求必须在服务关闭时全部拒绝 · **应该**

**要求**：进程/作用域关闭时，把所有挂起的等待以 `DeclinedError` 结束。

**理由**：否则这些 Promise 永远不 resolve，执行循环挂死。

---

## 4. 算法规范

### 4.1 判定

```
evaluateInput(input):
    agentRules = 取 agent 配置的规则（找不到 agent → 全 deny 兜底）    # R-09-06
    if input.resources 任一在 agentRules 下判定为 deny:                # R-09-03 ①
        return { effect: "deny", rules: agentRules }
    all = agentRules + savedRules(projectId)                          # R-09-03 ②
    effects = input.resources.map(r => evaluate(input.action, r, all).effect)
    effect = effects 含 deny ? "deny"
           : effects 含 ask  ? "ask"
           : "allow"                                                  # R-09-05
    return { effect, rules: all }
```

### 4.2 断言（阻塞直到用户回答）

```
assertPermission(input):
    result = evaluateInput(input)
    if result.effect == "deny":  throw BlockedError(相关规则)
    if result.effect == "allow": return

    id = newId()
    request = { id, ...input }
    promise = new Promise((resolve, reject) => pending.set(id, { request, resolve, reject }))
    publish("permission.asked", request)                              # → UI
    return promise                                                     # 阻塞
```

### 4.3 回复

```
replyPermission(requestId, reply, message?):
    item = pending.get(requestId);  if not item: throw NotFound
    publish("permission.replied", { requestId, reply, sessionId })

    if reply == "reject":                                             # R-09-09 拒绝传染
        item.reject(message ? CorrectedError(message) : DeclinedError())   # R-09-11
        pending.delete(requestId)
        for (id, other) in pending:
            if other.sessionId != item.sessionId: continue
            publish("permission.replied", { requestId: id, reply: "reject", ... })
            other.reject(DeclinedError())
            pending.delete(id)
        return

    if reply == "always" and item.request.save 非空:                   # R-09-04 只存 allow
        saveRules(projectId, item.request.action, item.request.save)
    item.resolve()
    pending.delete(requestId)
    if reply != "always": return

    saved = savedRules(projectId)                                     # R-09-10 允许传染
    for (id, other) in pending:
        agentRules = 取 other 的 agent 规则
        if other.resources 任一在 agentRules 下 deny: continue          # ← 安全底线仍然生效
        eff = agentRules + saved
        if not other.resources.every(r => evaluate(other.action, r, eff).effect == "allow"): continue
        publish("permission.replied", { requestId: id, reply: "always", ... })
        other.resolve()
        pending.delete(id)
```

### 4.4 通配符匹配

```
match(input, pattern):
    normalized = input.replace(所有 "\\" → "/")
    escaped = pattern.replace(所有 "\\" → "/")
                     .escapeRegexMetaChars()          # . + ^ $ { } ( ) | [ ] \
                     .replace(所有 "*" → ".*")
                     .replace(所有 "?" → ".")
    if escaped 以 " .*" 结尾:                          # R-09-02 特判
        escaped = escaped[0..-3] + "( .*)?"
    flags = isWindows ? "si" : "s"
    return new RegExp("^" + escaped + "$", flags).test(normalized)
```

---

## 5. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 默认效果 | `ask` | 固定 | 见 R-09-01 |
| 记忆作用域 | 项目 | 项目 / 全局 / 会话 | 全局：跨项目泄漏；会话：每次新会话重问 |
| 读类工具的 `save` | `["*"]` | `["*"]` / 具体路径 | 具体路径会让用户被每个文件问一次 |
| 命令类工具的 `save` | 具体命令 | 具体 / 前缀模式 | 前缀模式更方便但风险更高 |
| 拒绝传染范围 | 会话内 | 会话 / 全局 | 全局会误伤其它会话 |

---

## 6. 接口契约

```ts
interface Permission {
  /** 阻塞直到判定完成；deny 抛 BlockedError，拒绝抛 DeclinedError/CorrectedError */
  assert(input: Omit<PermissionRequest, "id"> & { id?: string }): Promise<void>
  /** 只判定不阻塞，返回 effect（用于 UI 预览） */
  ask(input: Omit<PermissionRequest, "id">): Promise<{ id: string; effect: Effect }>
  reply(input: { requestId: string; reply: Reply; message?: string }): Promise<void>
  forSession(sessionId: string): Promise<PermissionRequest[]>
}
```

---

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| 默认 allow | 等于没有权限系统 |
| 默认 deny | 未配置的工具全不可用 |
| 不分两步判定 | 一次「总是允许」打开整个沙箱 |
| 保存的规则允许是 deny | 产生绕过配置的隐藏状态 |
| 找不到 agent 兜底成 allow | 灾难 |
| 记忆不按项目隔离 | 跨项目泄漏危险授权 |
| `resources` 与 `save` 不分离 | 要么烦死用户，要么一键开沙箱 |
| 拒绝不传染 | 用户要点 5 次「不」 |
| 「总是」后不重新判定 | 绕过 agent deny |
| 纯拒绝当成工具错误 | 模型换个说法再试 |
| 带留言拒绝也中断循环 | 用户的意见传不到模型 |
| 请求不带来源 | 弹窗无法锚定到工具卡片 |
| 关闭时不拒绝挂起请求 | 执行循环挂死 |
| 通配符不做尾部特判 | 保存的规则对无参数命令失效 |

---

## 8. 分级实现路径

### 最小可用版（半天）

规则模型（`Rule[]` + `findLast` + 默认 ask）+ `assert` 阻塞 + `reply` 三态。

**不能砍的**：两步判定（R-09-03）、`resources`/`save` 分离（R-09-08）、纯拒绝的特殊语义（R-09-11）。

### 完整版

加拒绝传染、允许传染、项目作用域记忆、来源锚定、关闭时清理。

---

## 9. 验收

见 [`acceptance/SPEC-09.yaml`](../acceptance/SPEC-09.yaml)（10 条）。

**必须优先验证的一条**：agent 配了 `deny`，用户先前保存过对应的 `allow` → 仍然 deny（A-09-02）。
这是整个权限系统的安全底线，写错了其余全部条款都白搭。
