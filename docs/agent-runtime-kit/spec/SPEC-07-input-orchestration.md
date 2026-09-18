# SPEC-07 · 输入编排与执行循环

> **优先级** P1 · **工作量** M · **依赖** SPEC-01, SPEC-02
> **分析原文** [ch05 输入收件箱与 Drain 循环](../../context-and-streaming/05-input-and-drain.md)
> **验收** [`acceptance/SPEC-07.yaml`](../acceptance/SPEC-07.yaml)（16 条）

---

## 1. 目标与非目标

### 目标

让「用户随时能发消息」和「模型一轮不被打断」同时成立。

三种朴素做法都不对：

| 做法 | 问题 |
| --- | --- |
| 直接 append 到消息数组 | 当前请求已发出，改数组没用；下一轮突然多出一条来路不明的消息 |
| 禁用输入框 | 用户经常需要在 agent 跑偏时立刻纠正 |
| 中断当前请求再重发 | 已产生的工具结果全丢，成本浪费，用户看到内容消失 |

### 非目标

- 不做分布式执行（单进程串行即可满足全部条款）。
- 不规定 UI 上排队消息怎么展示。

---

## 2. 领域模型

```ts
export type Delivery = "steer" | "queue"

export interface AdmittedPrompt {
  /** 受理时就定下，提升后直接作为消息 id。见 R-07-02 */
  id: string
  sessionId: string
  prompt: { text: string; files?: FileAttachment[] }
  delivery: Delivery
  admittedSeq: number
  /** null/undefined = 尚未提升 */
  promotedSeq?: number
  timeCreated: number
}
```

```sql
CREATE TABLE session_input (
  id           TEXT PRIMARY KEY,
  session_id   TEXT NOT NULL REFERENCES session(id) ON DELETE CASCADE,
  prompt       TEXT NOT NULL,            -- JSON
  delivery     TEXT NOT NULL,            -- 'steer' | 'queue'
  admitted_seq INTEGER NOT NULL,
  promoted_seq INTEGER,                  -- NULL = pending
  time_created INTEGER NOT NULL
);
CREATE INDEX        idx_input_pending  ON session_input(session_id, promoted_seq, delivery, admitted_seq);
CREATE UNIQUE INDEX idx_input_admitted ON session_input(session_id, admitted_seq);
CREATE UNIQUE INDEX idx_input_promoted ON session_input(session_id, promoted_seq);
```

三个索引各有分工：复合索引服务全部待处理查询（字段顺序即查询谓词顺序）；两个唯一索引是原子提升的数据库级保障。

---

## 3. 规范条款

### R-07-01 受理与进入历史必须分离 · **必须**

**要求**：提交接口只把输入写进收件箱表并**立刻返回**，不等模型。此时它是「已受理的待办输入」，模型看不见。

**理由**：这是整个能力的地基。不分离就无法同时满足「随时能发」和「一轮不被打断」。

---

### R-07-02 输入 id 由客户端生成并复用为消息 id · **应该**

**要求**：客户端提交时自带 id；提升后该 id 直接作为 user 消息的 id。

**理由**：让乐观更新变得简单——前端本地立刻插入一条 pending 消息，服务端提升后广播的 user 消息 id 相同，前端原地替换，不闪烁不重复。

---

### R-07-03 受理必须幂等 · **必须**

**要求**：同一个 id 重复提交返回已存在的那条，不产生第二条。

**要求**：两道保险——前置存在性检查 + 唯一约束冲突后重查兜底（并发提交时前置检查可能都不命中）。

---

### R-07-04 提升必须原子且不可重复 · **必须**

**要求**：提升 = 在**一个事务**里完成「把 `promoted_seq` 从 NULL 置为具体值」+「产生 user 消息」。

**要求**：UPDATE 必须带 `WHERE promoted_seq IS NULL` 条件（数据库级 compare-and-swap）。

**理由**：并发下只有一个 writer 能成功，败者走「重查 + 校验一致」路径。用「先查后写」会双写。

---

### R-07-05 两种投递语义必须区分 · **必须**

| delivery | 语义 | 提升时机 |
| --- | --- | --- |
| `steer`（插话/转向） | 「现在就让模型知道」 | 当前执行还需要继续时，在**下一个安全边界**提升 |
| `queue`（排队） | 「等这轮彻底做完再说」 | 当前执行即将空闲时才提升，且**一次只提升一条** |

---

### R-07-06 steer 提升必须带 cutoff · **必须**

**要求**：提升 steer 时只提升 `admitted_seq <= cutoff` 的，`cutoff` = **本轮开始时**的最新序号。

**理由**：提升过程本身会产生事件、耗时。没有 cutoff 的话，提升过程中新到达的 steer 也会被卷进这一轮——这个集合在用户手速快时可能永不收敛。cutoff 把「本轮要处理的输入」在开始瞬间冻结成有限集合。

---

### R-07-07 queue 每次只提升一条 · **必须**

**理由**：一次全放会让模型在同一轮收到多个互相冲突的任务。

**要求**：从空闲恢复时，排队的那一条和所有待处理的 steer 一起提升，顺序是 **queue 先、steer 后**（steer 是对 queue 那条的补充说明）。

---

### R-07-08 有输入被提升时必须重置轮次配额 · **必须**

**要求**：本次边界上提升了任意数量的输入 → 步数配额重置为 1；**提升多条也只重置一次**。

**理由**：新指令意味着新任务，不该继承上一个任务已消耗的步数。按条数重置会让配额彻底失效。

---

### R-07-09 一个会话同时只能有一个执行循环 · **必须**

**要求**：按会话 id 串行化。不同会话可并行。

**理由**：两个循环会交错写历史，工具调用/结果配对错乱。

**要求**：提供三种语义不同的入口：

| 操作 | 空闲时 | 忙碌时 |
| --- | --- | --- |
| `run(key)` | 启动（`force = true`），等它结束 | **等当前结束**，不排新的 |
| `wake(key)` | 启动（`force = false`） | 置 `pendingWake`，**多次 wake 合并成一次后继** |
| `interrupt(key)` | 无操作 | 置停止标志、清 `pendingWake`、中断并等清理 |

**要求**：`wake` 的合并必须用布尔标志，不能用计数器（计数器会跑 N 轮空转）。

---

### R-07-10 执行循环必须是双层的 · **必须**

```
外层 while (shouldRun):          # 处理 queue
    内层 while (needsContinuation):   # 处理 step
        result = runTurn(sessionId, promotion, step)
        needsContinuation = result.needsContinuation
        step = result.step + 1
        promotion = "steer"                                   # 第二轮起固定 steer
        if not needsContinuation:
            needsContinuation = hasPending(sessionId, "steer")  # 兜底
    shouldRun = hasPending(sessionId, "queue")
    promotion = shouldRun ? "queue" : undefined
```

**四个要点缺一不可**：

1. 开头 `hasSteer ? false : hasQueue` —— 有 steer 时不查 queue，steer 优先级更高
2. 内层末尾 `promotion = "steer"` —— 第一轮可能是 queue，之后每轮都检查 steer。**这就是「模型跑着的时候用户插话，下一步就能看到」**
3. 内层末尾的兜底 `needsContinuation = hasPending(steer)` —— 模型说完了（无工具调用要继续），但这期间用户插了话 → **继续跑而不是空闲**
4. 外层的存在 —— 内层跑到没 steer 了，看有没有排队的 queue；有就再来一轮外层，**每次只放行一条**

---

### R-07-11 继续条件必须由「有本地工具被调用且无 provider 错误」决定 · **必须**

**要求**：`needsContinuation = (本轮有非 provider 执行的工具调用) && (无 provider 错误)`。

---

### R-07-12 执行循环开始时必须清理悬空工具 · **必须**

**要求**：进入循环前扫描历史，把状态为 `pending` / `running` 的工具标记为失败（错误文案如「工具执行被中断」）。

**理由**：崩溃/重启后历史里会留下悬空的 tool_call，下一轮 provider 直接 400。

---

### R-07-13 用户拒绝授权必须中断整个执行循环 · **必须**

**要求**：区分两种拒绝：

| 拒绝方式 | 处理 |
| --- | --- |
| 纯拒绝（无留言） | **中断整个执行循环**，不把拒绝当成工具输出喂回模型 |
| 拒绝并留言 | 变成工具错误文本回给模型，让它按意见调整 |

**理由**：纯拒绝喂回模型，它会换个说法再试一次。

**实现提示**：纯拒绝用一个可识别的特殊错误类型（不是普通失败），让执行循环能检测到并中断。

---

### R-07-14 会话位置变化时必须中止本轮 · **应该**

**要求**：每轮开始时校验会话的执行位置（工作目录/工作区）仍与当前一致，不符则中止。

**理由**：在错误的工作目录里跑工具会破坏用户文件。

---

### R-07-15 崩溃恢复只能从持久状态重建 · **必须**

**要求**：执行循环**没有持久身份**。恢复时只能从「待处理输入 + 投影历史 + 工具状态」重建，不得依赖内存中的循环状态。

**理由**：待办输入放内存队列，进程一挂就全丢。

---

## 4. 算法规范

### 4.1 受理

```
admit(id, sessionId, prompt, delivery):
    existing = find(id)
    if existing: return existing                              # R-07-03 前置检查
    seq = nextSeq(sessionId)                                  # 与消息 seq 同源
    try:
        INSERT INTO session_input(...) VALUES(..., promoted_seq = NULL)
        publish(PromptAdmitted, {...})
        return 新建的
    catch 唯一约束冲突:
        return find(id) ?? rethrow                            # R-07-03 兜底
```

**HTTP handler**：`admit()` → `coordinator.wake(sessionId)` → **立刻返回**（202 之类），不等模型。

### 4.2 提升

```
promoteSteers(sessionId, cutoff):                             # R-07-06
    rows = SELECT * FROM session_input
           WHERE session_id = ? AND promoted_seq IS NULL
             AND delivery = 'steer' AND admitted_seq <= cutoff
           ORDER BY admitted_seq ASC
    return publishPrompted(rows)

promoteNextQueued(sessionId):                                 # R-07-07
    row = SELECT * FROM session_input
          WHERE session_id = ? AND promoted_seq IS NULL AND delivery = 'queue'
          ORDER BY admitted_seq ASC LIMIT 1
    return row ? publishPrompted([row]) : false

publishPrompted(rows):
    for row in rows:
        transaction:                                          # R-07-04 原子
            updated = UPDATE session_input
                      SET promoted_seq = :seq
                      WHERE id = :id AND promoted_seq IS NULL  # ← CAS
                      RETURNING *
            if not updated:                                   # 竞态败者
                stored = find(id)
                if stored 与期望一致: continue                 # 幂等成功
                else: fail
            INSERT 一条 user 消息（id = row.id，时间戳用 row.time_created）
    return rows.length
```

**时间戳用受理时间，不是提升时间**——用户看到的应该是他按下回车的时刻。

### 4.3 每轮开始的提升

```
runTurn(sessionId, promotion, step):
    校验会话位置                                              # R-07-14
    初始化上下文纪元（见 SPEC-13，必须在提升之前）

    cutoff = 当前最新序号
    promoted = 0
    if promotion == "steer": promoted = promoteSteers(sessionId, cutoff)
    if promotion == "queue":
        promoted += promoteNextQueued(sessionId) ? 1 : 0
        promoted += promoteSteers(sessionId, cutoff)          # queue 与 steer 一起提升
    if promoted > 0: step = 1                                 # R-07-08 只重置一次

    准备上下文纪元（见 SPEC-13，必须在提升之后）
    组装历史 → 组装请求 → 压缩预检（SPEC-14）→ 调用 provider
    ...
    return { needsContinuation, step }
```

### 4.4 会话级串行

```
active = Map<key, { promise, pendingWake, stopping }>

wake(key):
    e = active.get(key)
    if e: e.pendingWake = true; return e.promise              # R-07-09 布尔合并
    return start(key, force = false)

run(key):
    e = active.get(key)
    if e: return e.promise                                    # 等当前，不排新的
    return start(key, force = true)

start(key, force):
    entry = { pendingWake: false, stopping: false }
    entry.promise = (async () => {
        try: await drain(key, force)
        finally:
            again = entry.pendingWake and not entry.stopping
            active.delete(key)
            if again: await start(key, false)                  # 合并的后继
    })()
    active.set(key, entry)
    return entry.promise
```

---

## 5. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 默认投递语义 | `queue` | — | 设成 `steer` 会让用户的每条消息都插进当前轮 |
| 每轮 queue 放行数 | `1` | 固定 | 见 R-07-07 |
| 步数配额上限 | 按 agent 配置 | 5 – 50 | 调小：复杂任务做不完；调大：跑偏时烧更多钱 |
| 串行粒度 | 会话 | 会话 / 全局 | 全局会让多会话互相阻塞 |

---

## 6. 接口契约

```ts
interface InputInbox {
  admit(input: { id: string; sessionId: string; prompt: Prompt; delivery: Delivery }): Promise<AdmittedPrompt>
  hasPending(sessionId: string, delivery: Delivery): Promise<boolean>
  promoteSteers(sessionId: string, cutoff: number): Promise<number>
  promoteNextQueued(sessionId: string): Promise<boolean>
}

interface RunCoordinator {
  run(sessionId: string): Promise<void>
  wake(sessionId: string): Promise<void>
  interrupt(sessionId: string): Promise<void>
}
```

---

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| 提交接口等模型返回 | 用户界面卡住，无法插话 |
| 待办输入放内存队列 | 进程重启全丢 |
| 提升不做 CAS | 并发下同一条消息进历史两次 |
| steer 提升不带 cutoff | 高频输入让本轮永不收敛 |
| queue 一次全放 | 模型收到多个冲突任务 |
| 按条数重置步数配额 | 配额失效 |
| 内层末尾不查 steer | 用户插话要等下次触发 |
| 用计数器合并 wake | 跑 N 轮空转 |
| 不清理悬空工具 | 重启后 provider 400 |
| 纯拒绝当成工具错误 | 模型换个说法再试 |
| 拒绝留言也中断循环 | 用户的修改意见传不到模型 |
| 时间戳用提升时间 | 用户看到错误的发送时间 |

---

## 8. 分级实现路径

### 最小可用版（1 天）

只做 `queue`，不做 `steer`：

```ts
async function submitPrompt(sessionId, id, prompt) {
  await insertInput({ id, sessionId, prompt, delivery: "queue", promotedSeq: null, ... })
  coordinator.wake(sessionId)          // 不等结果
}
```

**不能砍的**：收件箱表本身、`promoted_seq` 的 CAS、会话级串行。

### 完整版

加 `steer` + cutoff + 双层循环 + 三种协调器入口 + 悬空工具清理。

---

## 9. 验收

见 [`acceptance/SPEC-07.yaml`](../acceptance/SPEC-07.yaml)（16 条）。

最关键的一条端到端场景：模型跑工具期间发一条 steer → 下一轮就被提升且模型的回答体现了它（A-07-04）。
