---
name: runtime-audit
description: >-
  Use when auditing this project's AI conversation / agent runtime capabilities against the
  agent-runtime-kit specifications - that is, when the user asks whether an existing implementation
  of message models, event transport, client sync, timeline rendering, streaming markdown, input
  orchestration, tool systems, permissions, file safety, tool-output governance, system context,
  context epochs, compaction, skills, memory layers, or context-usage UI conforms to a SPEC-NN
  clause set, or asks for a gap report against those specs. Also use when the user references a
  clause id like R-08-03 or an assertion id like A-08-05 and wants to know the current state of the
  code. Do not use for general code review, for implementing or fixing those capabilities (that is
  runtime-implement), for verifying a finished implementation (that is runtime-verify), or for any
  project that does not contain docs/agent-runtime-kit.
---

# 运行时能力审计

对照 `docs/agent-runtime-kit/spec/` 的规范条款，判定本项目「有没有、对不对」，并为每个判定给出代码证据。

## 铁律

**只诊断，不开方，不改文件。**

- 不写「建议改成…」「可以引入…」。一个字都不要写。方案是 `02-plan` 的事。
- 本阶段唯一允许写入的文件是 `audit-report.md` 和 `audit.json`。
- 找不到就写「未找到」。判定为「满足」必须有能贴出来的 `file:line`。

**为什么**：审计和设计混在一起，你会一边看一边在脑子里改，然后倾向于把「想改的地方」标成缺失、把「懒得改的地方」标成符合。

## 流程

完整流程见 `docs/agent-runtime-kit/prompts/01-audit.md`（**开始前先读它**）。要点：

1. **先问审计范围**：全量 17 项 / 分层（P0+P1）/ 定向。除非用户明确要全量，建议分层。
2. **每个 SPEC 单独审，分段输出**，不要全审完再一起给。
3. 每个 SPEC：读 `spec/SPEC-NN-*.md` 第 3 节 + `acceptance/SPEC-NN.yaml` → 在项目里定位实现 → 逐条判定。
4. 判定四选一：`满足` / `部分满足` / `不满足` / `不适用`。
   - **「不适用」只有两种合法用法**：架构上不存在那个概念；产品上明确不需要。
   - 「我们还没做到」是 `不满足`，不是 `不适用`。
5. 每条不满足的**必须**级条款要写**具体失败场景**，不是「可能有性能问题」。

## 产出

- `audit-report.md` —— 总览表 + 风险清单 + 逐 SPEC 明细 + 依赖图阻塞点 + 审计盲区
- `audit.json` —— 机器可读，供 `02-plan` 消费。格式见提示词 §4。

## 自检

- [ ] 每条「满足」都有 `file:line`
- [ ] 每条「不适用」写了前提为什么不成立
- [ ] 报告里没有任何「建议」「应该改成」
- [ ] 「审计盲区」不是空的——静态审计必然有看不到的东西
