---
name: runtime-verify
description: >-
  Use when verifying a finished agent-runtime-kit capability against its acceptance assertions -
  that is, when the user asks to check whether an implementation of SPEC-01 through SPEC-17 passes
  the assertions in docs/agent-runtime-kit/acceptance/SPEC-NN.yaml, asks for a pass/fail verdict on
  assertion ids like A-08-05, or asks to fill in the status field of those YAML files. Also use
  right after runtime-implement finishes a capability. Do not use for auditing code that has not
  been implemented yet (that is runtime-audit), for writing or fixing the implementation itself
  (that is runtime-implement), for general test writing, or for any project that does not contain
  docs/agent-runtime-kit.
---

# 运行时能力验收

按 `acceptance/SPEC-NN.yaml` 逐条验证实施结果。

## 铁律

**不准改实现代码。** 唯一允许改的是测试文件和验证报告。

发现测试本身写错了（断言写反、用错 API）可以修，但要在报告里说明。

**为什么**：一边验一边改，你会不自觉地把"改到能过"当成"验证通过"。

**不准为了通过而放宽标准。** 觉得某条断言不合理，如实记为失败并在报告的「我对断言本身的异议」里说明，不要自己改断言的含义。

**不准编造证据。** 「测试通过」要有实际跑过的输出；「手动验证通过」要写出实际执行的步骤和实际看到的结果；没条件跑就标 `blocked`。

完整流程见 `docs/agent-runtime-kit/prompts/04-verify.md`。

## 判定

四选一：`pass` / `fail` / `n/a` / `blocked`。

- `n/a` 的标准和审计一样严格：只有「架构上不存在那个概念」或「产品上明确不需要」。**「这次没实现」是 `fail`**。
- 每条 `fail` 要给：期望 / 实际 / 复现 / 定位 / 根因判断。**不要写修复方案。**

## 反作弊检查（必做）

对每条 `pass` 问自己：

1. **这个测试真的会失败吗？** 把实现改坏一行，测试应该变红。**至少对 3 条关键断言做这个变异测试**，并在报告里写明改坏了什么、是否变红。SPEC 第 10 节点名的「必须优先验证」那几条**必须**做。
2. **测试验的是断言说的那件事吗？** 断言说「逐字节相同」就要 `assert(a === b)`，不是 `assert(a.length === b.length)`。
3. **手动步骤是真的执行过的吗？** 还是你觉得"应该会这样"？

## 产出

- `verify-SPEC-NN.md` —— 结论（通过 / 有条件通过 / 不通过）+ 逐条结果 + 失败明细 + 变异测试记录 + blocked 项 + n/a 理由
- 更新 `acceptance/SPEC-NN.yaml` 的 `status` 字段，并新增 `evidence`

**不要改 YAML 的 `statement` / `kind` / `severity` / `group`** —— 那些是规范的一部分。

## 结论判定规则

- 全部必须级 `pass` 且无 `blocked` → **通过**
- 必须级全 `pass` 但有 `blocked` → **有条件通过**（列出待验项）
- 有任何必须级 `fail` → **不通过**
