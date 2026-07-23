## Skill 不是提示词模板，而是 Agent 的可发现知识

很多人把 Skill 理解为"保存下来的提示词模板"，这低估了它的价值。提示词模板是死的——需要人记得去用；Skill 是活的——Agent 自己能发现、判断何时用、按需加载。

这个区别的根源在于 **可发现性（Discoverability）**。Agent 启动时不会加载所有 Skill 全文，只加载每个 Skill 的 `name` + `description`（约 100 token）。当用户提出请求时，Agent 在这些元数据中匹配意图，决定是否激活某个 Skill。只有被激活的 Skill，其完整正文才进入上下文窗口。

这意味着：**Skill 的质量取决于两个独立环节——被发现的概率（description 质量）和被激活后的效果（正文质量）**。两者缺一不可。

## 平台格局：一份 SKILL.md，六个平台

当前主流 AI 编码助手都采用相似的 Skill 规范，核心格式趋同：

| 平台 | 存储路径 | 触发方式 | 扩展能力 |
|------|---------|---------|---------|
| Qoder | `~/.qoder/skills/` | `/skill-name` 或自动 | 最小字段（name + description） |
| Claude Code | `~/.claude/skills/` | `/skill-name` 或自动 | 最丰富（工具授权、子代理、动态上下文） |
| Agent Skills 标准 | 任意目录 | 由客户端决定 | 开放标准（agentskills.io） |
| ClawHub/OpenClaw | OpenClaw 目录 | 守护进程 | Anthropic Skills 标准 |
| CCSwitch | 各助手对应目录 | 透传 | 跨平台管理工具 |
| SkillHub | 社区发布 | 社区平台 | Agent Skills 标准 |

所有平台共享 `SKILL.md` 核心格式——frontmatter（YAML 元数据）+ Markdown 正文。差异在于扩展字段和触发机制。**跨平台兼容的最低门槛是仅用 `name` + `description` 两个字段。**

## Description：Skill 的"搜索引擎优化"

### 为什么 description 决定生死

Agent 的 Skill 发现机制类似于搜索引擎：用户输入查询 → Agent 将查询与所有 Skill 的 description 做语义匹配 → 选中最相关的 Skill 激活。如果 description 写得模糊，Skill 永远不会被激活，再好的正文也没用。

### 好的 description 遵循 WHAT + WHEN 原则

```yaml
# 差：没有 WHEN，Agent 不知道何时触发
description: 帮助处理文档。

# 好：WHAT + WHEN + 触发关键词
description: 从 PDF 文件提取文本和表格，填写表单，合并文档。当处理 PDF 文件或用户提到 PDF、表单、文档提取时使用。
```

从 Token 角度分析：description 中的关键词（如 "PDF"、"表单"、"文档提取"）是 Agent 匹配用户意图的锚点。用户的查询中如果出现这些词，注意力匹配分数会显著提高。**关键词越具体，匹配越精准。**

### 第三人称而非第一人称

```yaml
# 差：第一人称，不够客观
description: 我可以帮你分析 git diff 并生成 commit message。

# 好：第三人称，客观描述
description: 分析 git diff 生成符合规范的 commit message。当用户要求帮助写提交信息、审查暂存变更或准备提交代码时使用。
```

Agent 在匹配时以任务为导向，第三人称描述更贴近"这是一个工具/能力"的语义框架，而非"这是一个角色"。

### OpenAI 的实践验证

OpenAI GPT-5.5 指南强调 **outcome-first** 描述——描述目标结果而非过程。这与 Skill description 的 WHAT 原则一致：

```yaml
# 差：描述过程
description: 首先读取文件，然后解析内容，接着验证格式，最后输出结果。

# 好：描述结果
description: 验证数据文件格式和完整性。当检查 CSV/JSON/YAML 数据文件、验证数据模式或排查数据问题时使用。
```

## 渐进式披露：Token 经济学的核心策略

### 三层加载模型

Skill 的设计遵循**渐进式披露**（Progressive Disclosure）原则，本质是上下文窗口的分层管理：

| 层级 | 内容 | Token 开销 | 加载时机 |
|------|------|-----------|---------|
| 元数据 | name + description | ~100 token | 启动时全量加载 |
| 指令 | SKILL.md 正文 | <5000 token（推荐） | 激活时加载 |
| 资源 | scripts/、references/、assets/ | 按需 | Agent 主动读取时加载 |

这个设计与 OpenAI 的 **Prompt Caching** 原理相通：高频不变的内容（元数据）始终在内存，低频变化的内容（正文）按需加载，极大内容（资源文件）仅在需要时才进入上下文。

### SKILL.md 应保持在 500 行以内

```markdown
# 差：SKILL.md 包含 800 行详细 API 文档

# 好：SKILL.md 保持 500 行以内
## 详细资源
- 完整 API 文档见 [reference.md](reference.md)
- 使用示例见 [examples.md](examples.md)
```

从注意力机制分析：上下文窗口中的 token 越多，每个 token 的平均注意力权重越低。800 行的 Skill 正文会稀释关键指令的注意力信号，导致 Agent "忘记"或忽略部分规则。500 行是经验上的甜蜜点——足够传达完整指令，又不至于淹没注意力。

### 文件引用仅一级深度

```markdown
# 正确：一级引用
详见 [reference.md](reference.md) 获取完整 API 文档。

# 错误：多级引用链
详见 [reference.md](reference.md)，其中引用了 [api-spec.md](../specs/api-spec.md)
```

Agent 读取配套文件是按需的、有成本的（每次读取消耗 token）。多级引用链会导致 Agent 需要多次读取才能获取完整信息，每次读取都挤占上下文窗口。一级深度确保 Agent 一次读取即可获得所需信息。

## 自由度匹配：Outcome-first 在 Skill 中的应用

### 任务脆弱性决定自由度

OpenAI GPT-5.5 指南的核心洞察——**outcome-first 优于 process-heavy**——在 Skill 设计中同样适用，但需要根据任务脆弱性调整：

| 自由度 | 适用场景 | 实现方式 | 示例 |
|--------|---------|---------|------|
| **高**（文本指令） | 多种有效方案 | 描述目标和原则 | 代码审查指南 |
| **中**（伪代码/模板） | 有首选模式 | 提供模板但允许变化 | 报告生成 |
| **低**（精确脚本） | 脆弱操作 | 精确步骤 + 验证脚本 | 数据库迁移 |

### 高自由度 Skill：给原则，不给步骤

```markdown
# 代码审查 Skill（高自由度）
## 审查清单
- [ ] 逻辑正确，处理边界情况
- [ ] 无安全漏洞（SQL 注入、XSS 等）
- [ ] 遵循项目代码风格
- [ ] 错误处理完善

## 反馈格式
- **严重**：合并前必须修复
- **建议**：考虑改进
```

这对应 GPT-5.5 的 outcome-first——描述"好的审查长什么样"，让 Agent 自己决定如何审查。

### 低自由度 Skill：给步骤 + 验证循环

```markdown
# 数据库迁移 Skill（低自由度）
## 步骤
1. 分析当前数据库结构
2. 根据需求生成迁移 SQL
3. 运行验证脚本检查语法
4. 输出迁移文件

## 验证
生成后立即运行验证：
python3 scripts/validate_migration.py output/migration.sql
若验证失败，修正后重新验证，直到通过。
```

数据库迁移是脆弱操作——一个错误可能导致数据丢失。这里 **process-heavy 是对的**，因为一致性比灵活性重要。GPT-5.5 指南说的"避免过程堆叠"针对的是模型自身推理能力足够强的场景，而非脆弱操作。

### 判断标准：如果出错代价高，用低自由度

```
任务出错代价高吗？
├── 是（数据库、部署、删除） → 低自由度（精确脚本 + 验证循环）
└── 否（审查、建议、生成）
    ├── 模型推理足够强 → 高自由度（原则 + 清单）
    └── 格式一致性重要 → 中自由度（模板）
```

## 结构化设计：五种常见模式

### 模板模式

提供输出格式模板，适用于格式一致性要求高的场景。与 OpenAI 推荐的 `# Output` 段落对应：

```markdown
## 输出格式
{type}({scope}): {subject}

{body}
```

### 示例模式（Few-Shot）

提供输入/输出示例。从 Token 角度，每个示例约消耗 50-200 token，1 个示例锚定格式性价比最高：

```markdown
## 示例
**输入**：添加了 JWT 认证中间件
**输出**：
feat(auth): 添加 JWT 认证中间件
```

### 工作流模式

分解为清晰步骤 + 检查清单。适用于多步骤任务：

```markdown
## 步骤
1. 运行测试套件
2. 构建应用
3. 推送至部署目标
4. 验证部署成功
```

### 条件工作流模式

引导通过决策点，适用于多分支流程。与 GPT-5.5 的"决策规则"（而非绝对规则）对应：

```markdown
## 决策规则
- 搜索后再回答，除非你已有足够的上下文证据
- 不确定时询问用户，而非猜测
```

### 反馈循环模式

实现验证循环，适用于质量关键任务。与 GPT-5.5 的"验证循环"原则对应：

```markdown
## 验证
生成后立即运行验证脚本。若验证失败，修正后重新验证，直到通过。
```

## Claude Code 的进阶能力

Claude Code 的 Skill 规范最丰富，提供了几个其他平台没有的能力：

### 动态上下文注入

```markdown
## 当前变更
!`git diff HEAD`
```

`!`command`` 在 Skill 内容发送给 Agent 前执行，输出替换占位符。这相当于 **预处理 token 注入**——在 Agent 看到指令之前，先把上下文数据准备好，避免 Agent 花额外步骤去获取。

### 子代理执行

```yaml
context: fork
agent: Explore
```

`context: fork` 让 Skill 在隔离的子代理中运行，不污染主对话上下文。从 Token 角度，这相当于**独立的上下文窗口**——子代理的推理过程不占用主窗口的 token 预算。

### 工具预授权

```yaml
allowed-tools: Bash(git *) Read Grep
```

预授权工具避免了 Agent 每次使用工具时都请求用户确认，减少交互开销。但要注意 **最小权限原则**——只授权 Skill 完成任务必需的工具。

## 反模式：什么让 Skill 变差

### 反模式 1：描述模糊

```yaml
# 差
description: 帮助处理文档。

# 好
description: 从 PDF 文件提取文本和表格，填写表单，合并文档。当处理 PDF 文件或用户提到 PDF、表单、文档提取时使用。
```

没有关键词的 description 就像没有 SEO 的网页——永远不会被发现。

### 反模式 2：选项过多

```markdown
# 差
你可以使用 pypdf，或 pdfplumber，或 PyMuPDF，或 pdf2image...

# 好
使用 pdfplumber 提取文本。
对于需要 OCR 的扫描件，使用 pdf2image + pytesseract。
```

GPT-5.5 指南明确警告：**选项过多会缩小模型的搜索空间，导致机械输出**。提供默认方案 + 逃生舱，而非列举所有可能。

### 反模式 3：Windows 路径

```markdown
# 差
运行脚本：scripts\helper.py

# 好
运行脚本：scripts/helper.py
```

所有平台统一使用正斜杠 `/`。Windows 反斜杠在 Linux/macOS 上无效，且在 Markdown 中可能被转义。

### 反模式 4：时效性信息

```markdown
# 差
如果你在 2025 年 8 月前操作，使用旧 API。

# 好
## 当前方法
使用 v2 API 端点。

## 旧模式（已弃用）
<details>
<summary>旧版 v1 API</summary>
...
</details>
```

Skill 应当是时间无关的。用"当前/旧版"结构替代日期，让 Skill 不会随时间失效。

### 反模式 5：SKILL.md 过长

当 SKILL.md 超过 500 行时，Agent 的注意力被稀释，关键指令可能被忽略。将详细内容移至配套文件，SKILL.md 只保留核心指令和文件引用。

### 反模式 6：术语不一致

```markdown
# 差：混用 URL/route/path/endpoint 指同一概念
# 好：选定一个术语贯穿全文
```

注意力机制中，同一概念用不同词表达会分散注意力锚点。统一术语能让所有相关 token 聚焦在同一语义上。

## 质量评估清单

生成 Skill 后逐项检查：

### 核心质量

- [ ] description 具体且包含关键词
- [ ] description 同时包含 WHAT 和 WHEN
- [ ] 使用第三人称
- [ ] SKILL.md 正文 ≤500 行
- [ ] 全文术语一致
- [ ] 示例具体而非抽象

### 结构

- [ ] 文件引用仅一级深度
- [ ] 渐进式披露使用得当
- [ ] 工作流有清晰步骤
- [ ] 无时效性信息

### 自由度匹配

- [ ] 脆弱操作用低自由度（精确脚本 + 验证）
- [ ] 开放任务用高自由度（原则 + 清单）
- [ ] 格式一致用中自由度（模板）

### 平台适配

- [ ] frontmatter 字段符合目标平台规范
- [ ] 存储路径正确
- [ ] 触发方式符合用户选择
- [ ] 使用正斜杠路径

### 包含脚本时

- [ ] 脚本解决实际问题
- [ ] 依赖已文档化
- [ ] 错误处理明确且有帮助

## Skill 与提示词工程的关系

Skill 本质上是**结构化的、可发现的、可复用的提示词**。前文《什么是好的提示词》中分析的所有原理都适用于 Skill：

| 提示词原理 | 在 Skill 中的应用 |
|-----------|-----------------|
| 信息密度最大化 | SKILL.md ≤500 行，每段证明其 token 成本 |
| 注意力聚焦 | 关键规则放开头，详细内容外置 |
| 上下文窗口分配 | 渐进式披露三层模型 |
| Outcome-first | 高自由度 Skill 描述目标而非过程 |
| 消息角色分层 | description（元数据）与正文（指令）分离 |
| 正面表述 | "使用 const/let" 优于 "不要用 var" |
| 分隔符划界 | Markdown 结构 + 配套文件分离 |

**Skill 是提示词工程的工程化形态**——它把一次性的提示词变成可维护、可发现、可复用的知识资产。

## 参考资料

- [Qoder Skill Generator](https://github.com/qoder/skill-generator) — 跨平台 Skill 生成规范（本文核心素材来源）
- [OpenAI GPT-5.5 Prompting Guide](https://developers.openai.com/api/docs/guides/prompt-guidance) — outcome-first、停止条件、验证循环
- [Claude Code Skills Documentation](https://docs.anthropic.com/en/docs/claude-code) — 动态上下文、子代理、工具授权
- [Agent Skills 开放标准](https://agentskills.io) — 跨平台 Skill 规范
- [OpenAI Skills Repository](https://github.com/openai/skills) — OpenAI 官方 Skill 实践
