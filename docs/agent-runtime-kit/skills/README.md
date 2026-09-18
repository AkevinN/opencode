# skills/ · 把提示词装成技能（可选）

> **这一层是可选的。** 直接复制 `prompts/` 下的 Markdown 贴给 AI 完全够用。
> 只有当你**反复**在多个项目、多个能力上跑这套流程时，装成技能才划算——省掉每次翻文件、贴全文的动作。

---

## 1. 装哪儿

三个技能对应三份提示词里**最常重复执行**的：

| 技能 | 对应提示词 | 什么时候用 |
| --- | --- | --- |
| `runtime-audit` | `prompts/01-audit.md` | 审一个能力的现状 |
| `runtime-implement` | `prompts/03-implement.md` | 实施一个能力 |
| `runtime-verify` | `prompts/04-verify.md` | 验收一个能力 |

`00-orchestrator`（一次性）、`02-plan`（一次性）、`05-optimize`（低频）**不建议装成技能**——它们每个项目只跑一两次，贴全文更清楚，而且它们要求的交互（问五个问题、等确认）不适合被"顺手触发"。

## 2. 安装

按你的 AI 编码助手的技能约定放置。常见的两种：

**目录布局**（opencode / Claude Code 等）：

```
<你的项目>/.opencode/skill/          # 或 .claude/skills/
  runtime-audit/SKILL.md
  runtime-implement/SKILL.md
  runtime-verify/SKILL.md
```

**配置声明**：

```jsonc
{
  "skills": ["./docs/agent-runtime-kit/skills"]
}
```

装完之后，工具包的 `spec/` 和 `acceptance/` 目录必须仍然可达——技能正文里引用的是**相对于项目根**的路径。如果你把工具包放在别处，改 SKILL.md 里的路径常量。

## 3. 关于 description 的写法

三个 SKILL.md 的 `description` 都是按 SPEC-15 §7 的三段结构写的：**正向触发条件 + 补充触发条件 + 明确排除条件**。

排除条件那一段尤其重要——没有它，模型会在任何提到"审计""验证"的对话里触发这些技能。

## 4. 自检

装完后验证三件事：

1. 新开一个会话，问「我们项目的事件传输符合规范吗」→ 应该触发 `runtime-audit`
2. 问「帮我看下这段 CSS」→ **不应该**触发任何一个
3. 触发后，技能正文里引用的 `spec/SPEC-NN-*.md` 路径能被读到
