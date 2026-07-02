## 提示词不是聊天，是结构化文本

很多人写提示词像聊天：想到什么写什么，角色、任务、格式混在一起。结果是同一个任务每次都要重新写，AI 每次理解都不一样。

提示词应该是一段**结构化文本**——用 XML 标签划分内容边界，每个标签有明确职责。这样写出来的提示词可复用、可检索、AI 提取后可直接使用。

> 本文聚焦"怎么写"，理论分析见 [什么是好的提示词：从 Token 原理到开源实践](/2026/06/18/ai-prompt-engineering/)

## 为什么用 XML 标签

Markdown 标记结构层级（标题、列表），XML 标记内容边界（`<role>...</role>`）。两者互补：

| 需求 | 用什么 | 示例 |
|------|--------|------|
| 划分结构层级 | Markdown 标题/列表 | `## 审查维度` |
| 划分内容边界 | XML 标签 | `<instructions>...</instructions>` |
| 包裹动态数据 | XML 标签 | `<diff>{{DIFF}}</diff>` |

XML 标签在 Token 层面创造了明确的语义边界，AI 的注意力能精确定位到每个区块，不会把"角色"和"指令"混为一谈。

## 6 步写作流程

### Step 1: 定义角色 `<role>`

1-2 句定义 AI 的身份、经验、职责。不写风格（风格放 `<personality>`）。

```xml
<role>
资深代码审查专家，10 年以上多语言项目审查经验。
</role>
```

**常见错误**：把风格写进角色（"你是一个友善的、耐心的助手"）。角色是"做什么的"，不是"什么样的"。

### Step 2: 设定目标 `<goal>`

用一句话描述终点，不描述过程。这是 outcome-first 的核心——告诉 AI 要到达哪里，而不是怎么走。

```xml
<goal>
端到端完成 PR 代码审查，输出可操作的修复建议。
</goal>
```

**常见错误**：把步骤写进目标（"首先检查安全，然后检查性能..."）。步骤放 `<instructions>`。

### Step 3: 写指令 `<instructions>`

列出 AI 必须覆盖的维度或检查项。用 Markdown 列表组织，每项简洁明确。

```xml
<instructions>
审查维度（每条 diff 变更必须覆盖）：

1. 安全
   - 注入攻击（SQL/XSS/CSRF）
   - 敏感信息泄露
2. 性能
   - N+1 查询
   - 循环内 IO 操作
3. 可维护性
   - 命名是否表意
   - 函数职责是否单一
</instructions>
```

**关键原则**：
- 用正面表述（"使用参数化查询"优于"不要拼接 SQL"）
- 关键词一致（通篇用"审查"，不混用"审查/检查/分析"）

### Step 4: 定义输出格式 `<output_format>`

用具体示例或模板定义输出，不用"格式好看一点"这类模糊描述。

```xml
<output_format>
按严重程度分组，每组内按行号排序：

🔴 严重（阻塞合并）
- `文件:行号` 问题描述 → 修复建议

🟡 警告（不阻塞）
- `文件:行号` 问题描述 → 修复建议

无问题的级别输出「无」。
</output_format>
```

### Step 5: 加停止规则 `<stop_rules>`

定义何时停止、何时重试、何时回退。避免 AI 无限循环或过度工作。

```xml
<stop_rules>
- diff 为空：输出「无法读取」并停止
- diff > 500 行：审查前 300 行，提示拆分
- 不确定是否为问题：归入建议，注明「需人工确认」
- 三个维度均已覆盖即完成
</stop_rules>
```

**关键原则**：判断类决策用决策规则（带理由），不用绝对规则（ALWAYS/NEVER）。

### Step 6: 加示例 `<examples>`（可选）

1 个输入输出对锚定格式，性价比最高。超过 3 个示例 Token 成本陡增，仅格式复杂时使用。

```xml
<examples>
<example id="input">
（简短输入示例）
</example>

<example id="output">
（期望输出示例）
</example>
</examples>
```

## 可选标签

按需添加，不强制全套使用：

| 标签 | 用途 | 何时用 |
|------|------|--------|
| `<context>` | 背景信息、动态数据 | 需要注入变量时 |
| `<personality>` | 语气、风格 | 角色需要特定风格时 |
| `<success_criteria>` | 成功标准 | 需要可验证的完成条件时 |
| `<constraints>` | 约束条件 | 需要限定范围或安全边界时 |

## 变量设计

需要动态注入的内容用 `{{变量名}}` 标记，放在 `<context>` 末尾（便于 Prompt Caching）。

```xml
<context>
生产环境项目的 PR，待审查代码差异：

<diff>
{{DIFF}}
</diff>
</context>
```

在文章的"变量"表格中说明每个变量的位置和示例：

| 变量 | 位置 | 说明 | 示例 |
|------|------|------|------|
| `{{DIFF}}` | `<context>` 内的 `<diff>` | 代码差异 | `git diff` 输出 |

## 完整结构

```xml
<prompt>
<role>身份和职责</role>
<personality>语气和风格（可选）</personality>
<goal>终点描述</goal>
<success_criteria>可验证的完成条件（可选）</success_criteria>
<instructions>必须覆盖的维度或检查项</instructions>
<constraints>范围和安全边界（可选）</constraints>
<context>
动态数据：
<variable>{{VARIABLE}}</variable>
</context>
<examples>输入输出对（可选）</examples>
<output_format>输出模板</output_format>
<stop_rules>停止和回退规则</stop_rules>
</prompt>
```

**顺序原则**：静态内容前置（role/goal/instructions 可缓存），动态内容后置（context/examples 每次变化）。标签顺序不强制，但保持一致便于维护。

## 质量检查清单

- [ ] `<role>` 只写身份，不写风格
- [ ] `<goal>` 描述终点，不描述过程
- [ ] `<instructions>` 用正面表述
- [ ] `<output_format>` 有具体模板，无模糊描述
- [ ] `<stop_rules>` 用决策规则，不用绝对规则
- [ ] 全文关键词一致
- [ ] 动态变量在 `<context>` 末尾
- [ ] 示例不超过 3 个
- [ ] 用 `<prompt>` 标签包裹整体

## 参考资料

- [什么是好的提示词：从 Token 原理到开源实践](/2026/06/18/ai-prompt-engineering/) — Token、注意力、上下文窗口的底层原理
- [OpenAI GPT-5.5 Prompting Guide](https://developers.openai.com/api/docs/guides/prompt-guidance) — outcome-first、停止条件、检索预算
- [Anthropic Prompt Engineering](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — XML 标签结构化、前缀预填
