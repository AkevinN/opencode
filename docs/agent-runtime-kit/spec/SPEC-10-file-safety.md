# SPEC-10 · 文件访问安全与并发

> **优先级** P1 · **工作量** M · **依赖** SPEC-08, SPEC-09
> **分析原文** [ch14 §7 路径安全与并发](../../context-and-streaming/14-tools.md)
> **验收** [`acceptance/SPEC-10.yaml`](../acceptance/SPEC-10.yaml)（7 条）

只有 7 条断言，但**其中一条挡住的是真实的安全漏洞**（符号链接逃逸）。

---

## 1. 目标与非目标

### 目标

让 agent 能读写文件，但：

1. 不能逃逸出它被授权的目录
2. 并发写不互相覆盖
3. 编辑不破坏文件的既有格式特征（行尾、BOM）

### 非目标

- 不做沙箱隔离（那是 OS/容器层的事）。
- 不做版本控制集成。

---

## 2. 领域模型

```ts
export interface ResolvedTarget {
  /** 规范化后的真实路径（符号链接已解析）；或不存在路径挂在一个真实祖先目录下 */
  canonical: string
  /** 权限资源标识：内部路径用相对路径，外部路径用绝对路径。见 R-10-04 */
  resource: string
  /** 非空表示这是外部路径，需要额外授权 */
  external?: {
    action: "external_directory"
    directory: string
    resource: string
    save: string
  }
}

export type PathError =
  | { reason: "relative_escape" }        // 相对路径逃逸出根
  | { reason: "location_escape" }        // 符号链接逃逸
  | { reason: "non_directory_ancestor" } // 祖先不是目录

export class StaleContentError extends Error {}
```

---

## 3. 规范条款

### R-10-01 相对路径不得逃逸出工作根 · **必须**

**要求**：把相对路径基于工作根解析成绝对路径后，若字面上已不在根内 → 拒绝（`relative_escape`）。

**挡住**：`../../etc/passwd`。

---

### R-10-02 符号链接解析后必须重新校验 · **必须**

**要求**：对字面上在根内的路径，解析真实路径（`realpath`）后**再查一次**是否仍在根内；不在则拒绝（`location_escape`）。

**挡住**：`ln -s /etc ./config` 然后读 `config/passwd`。

**这是本规范里最容易漏的一条**——路径字面检查通过，但真实路径在根外。

---

### R-10-03 绝对外部路径必须要求额外授权 · **必须**

**要求**：显式的绝对路径落在根外时**不直接拒绝**，而是生成一个「外部目录访问」授权要求。工具必须先通过它，再通过自己的动作授权。

**要求**：授权顺序固定：**外部目录 → 本工具动作**。

**理由**：顺序反了会出现「用户批准了编辑，结果发现目录也要批」的二次弹窗。

**要求**：外部授权的资源粒度是**目录 + 通配**（如 `/abs/dir/*`），不是单个文件——用户批准一次就能在该目录工作。

---

### R-10-04 权限资源的表示法必须区分内外 · **必须**

| 位置 | 资源表示 |
| --- | --- |
| 根内 | **相对路径**（如 `src/index.ts`） |
| 根外 | **绝对路径**（如 `/etc/hosts`） |

**理由**：用户保存的「允许读 `src/*`」不会因为项目被移动到别的目录而失效。

---

### R-10-05 不存在的路径必须锚定到最近的存在祖先 · **必须**

**要求**：目标不存在时，向上逐级找最近的存在的目录，解析它的真实路径，再把剩余部分接回去。若找到的祖先不是目录 → 拒绝（`non_directory_ancestor`）。

**理由**：不这样做，创建新文件的场景（写入深层路径）会因为 `realpath` 失败而不可用。

---

### R-10-06 条件写必须在按路径的锁内比较并写入 · **必须**

**要求**：提供「仅当内容仍是期望值时才写」的操作，且**比较和写入在同一把按规范化路径的互斥锁内**。

**理由**：模型并发调多个编辑改同一文件时，不加锁会出现「都读到旧内容 → 都写 → 后写的覆盖先写的」，静默丢改动。

---

### R-10-07 内容已变必须给出可操作的错误 · **必须**

**要求**：条件写失败时，给模型的错误文案必须**告诉它下一步做什么**。

**参考**：`File changed after permission approval. Read it again before editing.`

**理由**：只说「写入失败」模型会盲目重试。

---

### R-10-08 编辑必须保留行尾风格 · **必须**

**要求**：读出原文件后检测行尾（是否含 `\r\n`），把输入的新旧字符串转换成同一风格再比较和替换。

**理由**：在 CRLF 文件里用 LF 的 `oldString` 匹配会永远找不到；写回时混用行尾会让整个文件在 diff 里全变。

---

### R-10-09 编辑必须保留 BOM 且只保留一个 · **必须**

**要求**：读时剥离 BOM 并记住是否存在；写回时按原状恢复，且保证最多一个。

---

### R-10-10 编辑的失败文案必须可操作 · **应该**

**要求**：每种失败给出明确的下一步。参考：

| 情况 | 文案要点 |
| --- | --- |
| 新旧字符串相同 | 「无变化可应用」 |
| 旧字符串为空 | 「不得为空，创建/覆盖请用写入工具」 |
| 找不到匹配 | 「必须完全匹配，包括空白和缩进」 |
| 多处匹配且未声明全替换 | 「提供更多上下文，或显式声明全部替换」 |
| 内容已变 | 「重新读取后再编辑」 |

**理由**：**每条都告诉模型下一步该做什么**，这是让 agent 能自我修复的关键。

---

### R-10-11 编辑应产出结构化差异信息 · **应该**

**要求**：编辑结果携带 `{file, patch, status, additions, deletions}` 供 UI 渲染 diff 与摘要。

---

## 4. 算法规范

### 4.1 路径解析

```
resolveTarget(inputPath, kind):
    isRelative = not isAbsolute(inputPath)
    absolute   = resolve(workRoot, inputPath)
    lexicallyInternal = contains(workRoot, absolute)

    if isRelative and not lexicallyInternal:
        throw PathError("relative_escape")                         # R-10-01

    resolved = resolvePathWithAncestor(absolute)                   # R-10-05

    if lexicallyInternal and not contains(realRoot, resolved.canonical):
        throw PathError("location_escape")                         # R-10-02 ← 最容易漏

    external = not lexicallyInternal
    resource = external ? slash(resolved.canonical)                # R-10-04
                        : slash(relative(realRoot, resolved.canonical) || ".")
    externalDir = (kind == "directory" and resolved.type == "Directory")
                  ? resolved.canonical : resolved.parentDirectory
    return {
      canonical: resolved.canonical,
      resource,
      external: external ? { action: "external_directory",
                             directory: externalDir,
                             resource: slash(join(externalDir, "*")),
                             save:     slash(join(externalDir, "*")) } : undefined,
    }

resolvePathWithAncestor(absolute):                                 # R-10-05
    existing = realpath(absolute) 或 undefined
    if existing: return { canonical: existing, type: stat(existing).type, parentDirectory: … }
    anchor = dirname(absolute)
    loop:
        canonical = realpath(anchor) 或 undefined
        if canonical:
            if stat(canonical).type != "Directory": throw PathError("non_directory_ancestor")
            return { canonical: resolve(canonical, relative(anchor, absolute)), parentDirectory: canonical }
        parent = dirname(anchor)
        if parent == anchor: throw PathError("non_directory_ancestor")
        anchor = parent
```

### 4.2 条件写

```
locks = KeyedMutex()                                               # 按 canonical 路径

writeIfUnchanged({ target, expected, content }):                   # R-10-06
    return locks.withLock(target.canonical)(不可中断地执行:
        current = readFileBytes(target.canonical)
        if not sameBytes(current, expected): throw StaleContentError(target.canonical)
        writeFile(target.canonical, content)
        return { existed: true, resource: target.resource }
    )
```

### 4.3 编辑流程

```
edit(input, ctx):
    早退校验:                                                       # R-10-10
        oldString == newString → ToolFailure("No changes to apply: …")
        oldString == ""        → ToolFailure("oldString must not be empty. Use write to …")

    target = resolveTarget(input.path, "file")
    if target.external: assertPermission(外部目录授权)               # R-10-03 顺序
    assertPermission({ action: "edit", resources: [target.resource], save: ["*"] })

    source   = readFileBytes(target.canonical)
    text     = decodeUtf8StripBom(source)                          # R-10-09
    ending   = detectLineEnding(text.body)                         # R-10-08
    oldStr   = convertLineEnding(input.oldString, ending)
    newStr   = convertLineEnding(input.newString, ending)

    count = countOccurrences(text.body, oldStr)
    if count == 0: ToolFailure("Could not find oldString … must match exactly, including whitespace and indentation.")
    if count > 1 and not input.replaceAll:
        ToolFailure("Found multiple exact matches … provide more surrounding context or set replaceAll to true.")

    replaced = input.replaceAll ? replaceAll(...) : replaceFirst(...)
    counts   = diffLineCounts(text.body, replaced)                 # R-10-11
    result   = writeIfUnchanged({ target, expected: source,
                                  content: joinBom(replaced, text.hadBom) })
    return { files: [{ file: result.resource, patch, status: "modified", ...counts }],
             replacements: count }
```

---

## 5. 可配置参数

| 参数 | 参考默认值 | 可调范围 | 调大 / 调小的影响 |
| --- | --- | --- | --- |
| 工作根 | 会话的工作目录 | — | 决定「内部/外部」的边界 |
| 外部授权粒度 | 目录 + `/*` | 目录 / 单文件 | 单文件会让用户被每个文件问一次 |
| 锁粒度 | 规范化路径 | 路径 / 目录 / 全局 | 全局会让并发编辑串行化 |
| 是否允许外部路径 | 允许（需授权） | 允许 / 完全禁止 | 完全禁止更安全但限制了跨仓库工作 |

---

## 6. 接口契约

```ts
interface PathResolver {
  /** 解析并派生权限资源；不做授权 */
  resolve(input: { path: string; kind?: "file" | "directory" }): Promise<ResolvedTarget>
}

interface FileMutation {
  create(input: { target; content }): Promise<WriteResult>            // 不覆盖已存在
  write(input: { target; content }): Promise<WriteResult>
  writeIfUnchanged(input: { target; expected: Uint8Array; content }): Promise<WriteResult>
  remove(input: { target }): Promise<RemoveResult>
}
```

---

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| 只做字面路径检查 | **符号链接逃逸**（真实安全漏洞） |
| 绝对外部路径直接拒 | 无法跨仓库工作 |
| 外部授权粒度是单文件 | 用户被每个文件问一次 |
| 授权顺序颠倒 | 二次弹窗 |
| 内部路径用绝对路径做资源 | 项目移动后所有保存的规则失效 |
| 不存在路径直接 stat 失败 | 无法创建新文件 |
| 条件写不加锁 | 并发编辑静默丢改动 |
| 只说「写入失败」 | 模型盲目重试 |
| 不检测行尾 | CRLF 文件永远匹配不上；写回后整文件 diff |
| 不保留 BOM | 某些工具链读不出文件 |
| 失败文案不给下一步 | agent 无法自我修复 |

---

## 8. 分级实现路径

### 最小可用版（半天）

三道路径检查（R-10-01/02/05）+ 条件写 + 按路径锁。

**绝对不能砍 R-10-02**（符号链接校验）。

### 完整版

加外部目录授权、行尾/BOM 保留、结构化 diff、全套可操作文案。

---

## 9. 验收

见 [`acceptance/SPEC-10.yaml`](../acceptance/SPEC-10.yaml)（7 条）。

**A-10-02（符号链接逃逸）必须写成自动化测试**：在测试目录里创建一个指向外部的符号链接，尝试读取，断言被拒绝。
手工验证容易遗漏，而这条一旦失效就是可被利用的漏洞。
