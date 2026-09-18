---
name: runtime-implement
description: >-
  Use when implementing one capability from the agent-runtime-kit specifications in this project -
  that is, when the user names a SPEC id (SPEC-01 through SPEC-17) or a capability it covers
  (message model with durable events, SSE event transport, client-side delta sync, timeline and
  scroll anchoring, process rendering, streaming markdown, input orchestration and interjection,
  tool system, permission rules, file-edit safety, tool-output governance, system context sources,
  context epoch, compaction, skills, memory layers, context-usage UI) and asks to build it. Do not
  use for auditing an existing implementation (that is runtime-audit), for verifying a finished one
  (that is runtime-verify), for fixing an implementation that already exists but deviates from spec
  (that is prompts/05-optimize.md), or for any project that does not contain docs/agent-runtime-kit.
---

# 运行时能力实施

一次实施**一个** SPEC。同时做多个会让每一个都成为半成品。

## 开始前必须做的三件事

1. **读 `spec/SPEC-NN-*.md` 全文**，不跳读。重点：第 3 节条款、第 4 节算法、第 7/8 节边界情况与反模式。
2. **读 `acceptance/SPEC-NN.yaml`** —— 在写第一行代码之前就知道验收标准。
3. **确认依赖已满足**（SPEC 头部标了）。依赖未实现 → 停下来问，不要"先做个简化版凑合"。
   **SPEC-12 和 SPEC-13 必须一起实现**，被要求单独做其中一个时先确认。

完整流程见 `docs/agent-runtime-kit/prompts/03-implement.md`。

## 顺序

**步骤 1：先出设计方案，等用户确认，不要直接写代码。**

方案包含：适配映射表（规范概念 → 本项目用什么）、文件改动清单（每行标涉及条款）、数据层改动、新增依赖（要批准）、选择偏离的条款、本次不实现的部分。

**步骤 2：写代码。**

- **最小 diff**，只改必需的地方
- **跟随宿主约定**——命名、错误处理、日志、测试框架一律跟项目走。规范里的示例代码是表达语义用的，不是风格模板
- 在「不这么写就会被改错」的关键分支上加一行注释标条款号和它防的是什么
- 第 7 节边界情况表逐行实现；第 8 节反模式表当 checklist

**步骤 3：自测。** 按 YAML 的 `kind`：`unit` 写成真实测试（用项目自己的测试框架），`manual` 写出精确的手动步骤。

**步骤 4：交付。** 输出改动清单 + 条款覆盖表（必须级 N/M）+ 测试对照 + 已知限制。

## 硬约束

- 不加依赖不问
- 不顺手重构无关代码
- 不"顺便"实现别的 SPEC
- 不改格式化配置 / lint 规则
- 交付说明里不要把「未实现」写成「已实现」
- 做最小可用版时，SPEC 第 9 节「绝对不能砍」的那些一条都不能砍

## 遇到冲突

规范与现有架构冲突 → **不要自己选一边**，把选项、代价、你的推荐摊开给用户。
规范没覆盖的情况 → **停下来问**，给出判断和理由，让用户做选择题。
