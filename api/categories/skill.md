# 分类：Skill

共 5 篇文章

---

# Windows 桌面快捷方式：双击启动任意 CLI 工具
Date: 2026-07-01 | Tags: opencode, Skill, Windows, 快捷方式 | URL: https://bsheepcoder.github.io/2026/07/01/skill-opencode-shortcut/

## 痛点：每次开终端都要 cd

日常开发中，启动一个 CLI 工具（如 opencode、hexo、python REPL）往往要三步：打开终端 → `cd` 到项目目录 → 输入命令。步骤不多，但每天重复十几次就很烦。

Windows 的 `.lnk` 快捷方式不只指向 exe，还能携带参数、指定工作目录。配合 Windows Terminal 的 `-d` 参数和 PowerShell 的 `-NoExit`，可以做到**双击桌面图标 → 直接在项目目录启动 CLI 工具**。

本文以 opencode 为例，但方案适用于任何 CLI 工具——改一行命令即可迁移。

## 工作原理：四层调用链

双击 `.lnk` 后发生的事：

```
.lnk 快捷方式
   │  TargetPath = wt.exe
   │  Arguments  = -d "项目目录" powershell -NoExit -Command "opencode"
   ▼
Windows Terminal (wt.exe)
   │  -d "项目目录" 设定终端启动目录
   ▼
PowerShell 进程
   │  -NoExit 执行后不关窗
   │  -Command "opencode" 运行目标 CLI
   ▼
opencode TUI 启动（工作目录 = 项目目录）
```

每一层各司其职：

| 层 | 职责 | 关键参数 |
|----|------|---------|
| `.lnk` 文件 | 绑定图标、目标、工作目录 | `IconLocation` / `WorkingDirectory` |
| `wt.exe` | 启动终端并设定目录 | `-d "路径"` |
| `powershell` | 执行命令并控制退出行为 | `-NoExit -Command "..."` |
| 目标 CLI | 实际要用的工具 | `opencode` / `hexo s` 等 |

为什么是四层而不是直接 `.lnk` 指向 `opencode.exe`？因为 opencode 是 TUI 程序，需要一个终端宿主来承载界面；而 `wt.exe -d` 能在启动终端的同时设定目录，`-NoExit` 保证 CLI 退出后窗口不消失（便于看报错）。这三者组合才完整。

## 前置条件

| 条件 | 验证命令 | 说明 |
|------|---------|------|
| 目标 CLI 在 PATH | `Get-Command opencode` | npm 全局安装的工具默认在 PATH |
| Windows Terminal | `Get-Command wt` | Win11 自带，Win10 需从 Store 安装 |
| PowerShell 5.1+ | `$PSVersionTable.PSVersion` | Windows 系统自带 |

## 创建快捷方式

完整的 PowerShell 脚本，直接复制执行即可：

```powershell
$ws = New-Object -ComObject WScript.Shell
$desktop = [Environment]::GetFolderPath('Desktop')
$sc = $ws.CreateShortcut("$desktop\OpenCode (Hexo).lnk")
$sc.TargetPath = "$env:LOCALAPPDATA\Microsoft\WindowsApps\wt.exe"
$sc.Arguments  = '-d "E:\Nut\Hexo" powershell -NoExit -Command "opencode"'
$sc.WorkingDirectory = "E:\Nut\Hexo"
$sc.IconLocation = "$env:APPDATA\npm\node_modules\opencode-ai\bin\opencode.exe,0"
$sc.Save()
Write-Host "已创建：$desktop\OpenCode (Hexo).lnk"
```

执行后桌面会出现 `OpenCode (Hexo)` 图标，双击即在 `E:\Nut\Hexo` 目录启动 opencode。

## 参数逐项说明

### TargetPath —— 启动谁

```powershell
$sc.TargetPath = "$env:LOCALAPPDATA\Microsoft\WindowsApps\wt.exe"
```

指向 Windows Terminal。用 `$env:LOCALAPPDATA` 而非硬编码路径，确保跨用户通用。`wt.exe` 在此目录是个别名（零字节 reparse point），实际由 Store 版 wt 执行。

### Arguments —— 怎么启动

```powershell
$sc.Arguments = '-d "E:\Nut\Hexo" powershell -NoExit -Command "opencode"'
```

这是最关键的一行，三段含义：

| 段 | 作用 |
|----|------|
| `-d "E:\Nut\Hexo"` | 传给 wt.exe，设定终端启动目录 |
| `powershell -NoExit -Command "opencode"` | wt.exe 启动 PowerShell，执行 `opencode` 后不关窗 |

外层用单引号包裹整个字符串，内层路径用双引号防空格断裂——这是 PowerShell 引号嵌套的标准写法。`-NoExit` 的作用：opencode 正常退出或崩溃时，窗口保持打开，能看到最后输出和报错。

### WorkingDirectory —— 进程在哪跑

```powershell
$sc.WorkingDirectory = "E:\Nut\Hexo"
```

设定 wt.exe 进程本身的工作目录。虽然 `wt -d` 已设定终端目录，但显式指定 `WorkingDirectory` 是好习惯——某些场景下 wt.exe 自身读取相对路径会用到它，两者一致避免歧义。

### IconLocation —— 图标从哪来

```powershell
$sc.IconLocation = "$env:APPDATA\npm\node_modules\opencode-ai\bin\opencode.exe,0"
```

从 `opencode.exe` 提取内嵌图标（`,0` 表示第一个图标资源）。npm 全局安装的工具通常带有可执行文件，其内嵌图标比 wt.exe 的终端图标更直观。

> opencode.exe 约 165 MB，但 `IconLocation` 只读取图标资源段，不会加载整个文件到内存。

## 验证

### 方式一：回读快捷方式属性

创建后用 COM 对象回读，确认各字段写入正确：

```powershell
$ws = New-Object -ComObject WScript.Shell
$sc = $ws.CreateShortcut("$([Environment]::GetFolderPath('Desktop'))\OpenCode (Hexo).lnk")
"TargetPath  : $($sc.TargetPath)"
"Arguments   : $($sc.Arguments)"
"WorkingDir  : $($sc.WorkingDirectory)"
"IconLocation: $($sc.IconLocation)"
```

预期输出：

```
TargetPath  : C:\Users\71041\AppData\Local\Microsoft\WindowsApps\wt.exe
Arguments   : -d "E:\Nut\Hexo" powershell -NoExit -Command "opencode"
WorkingDir  : E:\Nut\Hexo
IconLocation: C:\Users\71041\AppData\Roaming\npm\node_modules\opencode-ai\bin\opencode.exe,0
```

### 方式二：双击行为测试

双击桌面图标，依次确认：

1. Windows Terminal 窗口弹出，标题栏显示 `E:\Nut\Hexo`
2. PowerShell 提示符路径为 `E:\Nut\Hexo`
3. opencode TUI 自动启动
4. 退出 opencode 后窗口保持打开（`-NoExit` 生效）

## 进阶用法

### 钉到任务栏

把 `.lnk` 拖到任务栏即可。钉后右键图标能看到最近列表，适合多个项目快速切换。

### 无 Windows Terminal 时兜底

老版 Windows 10 若未装 wt.exe，可降级用系统自带控制台：

```powershell
$sc.TargetPath = "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
$sc.Arguments  = '-NoExit -Command "Set-Location ''E:\Nut\Hexo''; opencode"'
$sc.WorkingDirectory = "E:\Nut\Hexo"
```

注意 `Set-Location` 要在 `opencode` 前执行，因为 powershell.exe 没有 `-d` 参数。单引号嵌套用 `''` 转义。

### 退出即关窗

去掉 `-NoExit` 即可，opencode 退出后窗口立即关闭：

```powershell
$sc.Arguments = '-d "E:\Nut\Hexo" powershell -Command "opencode"'
```

适合无人值守场景，但崩溃信息会丢失，不推荐日常使用。

## 迁移到其他 CLI 工具

方案的核心价值在于**通用性**——把 `opencode` 替换为任意命令即可。下表列出常见迁移示例：

| 工具 | Arguments 命令段 | 图标来源 | 备注 |
|------|-----------------|---------|------|
| opencode | `opencode` | `opencode-ai\bin\opencode.exe` | 本文示例 |
| Hexo 本地预览 | `hexo s` | `hexo-cli\bin\hexo` 或默认 | 调试博客 |
| Python REPL | `python` | `python.exe` | 交互式编程 |
| VS Code 打开目录 | `code .` | `Code.exe` | `.` 表示当前目录 |
| Git 状态查看 | `git status` | git.exe | 查看仓库变更 |

迁移时改动点：

1. `Arguments` 中 `-Command` 后的命令
2. `IconLocation` 指向对应工具的 exe
3. `.lnk` 文件名改为工具相关名称

> 一个项目可建多个快捷方式（opencode / hexo s / code .），共享同一个 `WorkingDirectory`，桌面放一组图标覆盖完整开发循环。

## 注意事项

- **路径固定**：`.lnk` 绑定的是创建时的绝对路径，项目迁移后需重新生成。快捷方式无法动态跟随"当前文件夹"。
- **同名覆盖**：重复执行脚本会覆盖同名 `.lnk`，无需先删除旧文件。
- **引号嵌套**：外层单引号、内层双引号是 PowerShell 处理路径空格的标准写法，改动时注意层级。
- **图标文件**：`IconLocation` 指向的 exe 必须存在，否则显示默认图标。卸载工具后图标会失效。
- **PATH 依赖**：`opencode` 命令依赖 PATH 中能找到它。若用 nvm 等版本管理器，切换 Node 版本后 PATH 可能变化，导致命令失效。

## 总结

这套方案的本质是**用 `.lnk` 把"开终端 + cd + 启动工具"三步压缩成双击一下**。四层调用链（`.lnk` → `wt.exe` → `powershell` → 目标 CLI）各管一件事，参数清晰可调。改一行命令就能迁移到任意 CLI 工具，改一个目录就能绑定不同项目。

对一个工具的熟练程度，往往就体现在把它缩短到几次按键。桌面快捷方式是最朴素的一种——但它真的省时间。


---

# 趋势观察准则：三维度判断框架
Date: 2026-06-26 | Tags: Skill, 趋势观察, 分析框架 | URL: https://bsheepcoder.github.io/2026/06/26/skill-trend-observation/

## 为什么需要框架

每次新技术出现，都会有人问同一个问题：**这个领域的窗口期还有多久？**

直播带货火了 3 年才被监管覆盖，社区团购只火了 1.5 年就迎来反垄断，加密货币交易所从野蛮生长到全面清退走了 4 年。窗口期越来越短，但判断"现在能不能进"始终靠直觉。

三维度判断框架把这个直觉拆成可量化的评估模型。

## 三维度判断框架

```
立法空白度 × 执法可行性 × 商业变现能力 = 窗口期价值
```

三者同时满足时，才是真正的"真空窗口期"。任一维度归零，窗口即关闭。

### 维度一：立法空白度

该领域是否有明确的法律条文规范。

| 评级 | 标准 |
|------|------|
| **高** | 无任何法规覆盖，法律未定义该行为 |
| **中** | 有上位法框架，但无实施细则或司法解释 |
| **低** | 法规明确禁止，仅执法尚未覆盖 |

判断方法：

1. 检索相关法律法规（刑法、行政法、行业监管条例）
2. 查看是否有近期立法动态（草案、征求意见稿）
3. 检索司法判例（裁判文书网、典型案例）

### 维度二：执法可行性

即使有法规，执法是否实际覆盖。

| 评级 | 标准 |
|------|------|
| **高盲区** | 取证极难、跨地域、涉案分散、执法成本远高于收益 |
| **中盲区** | 可执法但优先级低，需达到一定规模才触发 |
| **低盲区** | 技术监测成熟，大数据风控已覆盖 |

判断方法：

1. 评估取证难度（技术手段是否可追溯）
2. 评估执法成本（人力、技术、时间）
3. 查看近年打击案例数量和趋势
4. 评估是否被纳入专项整治范围

### 维度三：商业变现能力

是否有真实市场需求和付费意愿。

| 评级 | 标准 |
|------|------|
| **强** | 真实需求、付费意愿明确、可规模化 |
| **中** | 有需求但付费意愿弱，或难以规模化 |
| **弱** | 伪需求、无付费意愿、不可持续 |

判断方法：

1. 是否有真实用户痛点（非人为制造的伪需求）
2. 用户是否有付费能力和意愿
3. 能否形成可复制的商业模式
4. 边际成本是否递减

## 窗口期评估矩阵

| 立法空白 | 执法盲区 | 变现能力 | 综合判断 |
|---------|---------|---------|---------|
| 高 | 高 | 强 | **真空窗口期**——可切入，但窗口短 |
| 高 | 高 | 中 | **观察期**——等待变现模式成熟 |
| 高 | 中 | 强 | **灰色期**——有风险但可操作 |
| 中 | 高 | 强 | **过渡期**——法规迟早跟进 |
| 中 | 中 | 强 | **高危期**——随时可能被打击 |
| 低 | — | — | **已关闭**——不建议进入 |
| — | 低 | — | **已关闭**——不建议进入 |

## 中国市场的窗口期规律

```
新业态出现 → 法律空白期（1-2年）→ 立法跟进 → 执法覆盖 → 窗口关闭
```

三条铁律：

- **窗口越来越短**：立法密度加大，反应速度加快
- **规模即风险**：一旦规模大到引起注意，立法和打击通常在 1-2 年内跟进
- **技术迭代快于法律**：但法律一旦跟上，往往溯及既往

近年窗口期缩短案例：

| 领域 | 空白期 | 监管节点 |
|------|--------|---------|
| 直播带货 | ~3 年（2016-2019） | 《网络直播营销管理办法》出台 |
| 社区团购 | ~1.5 年（2020-2021） | 反垄断介入，"九不得"出台 |
| 加密货币交易所 | ~4 年（2017-2021） | 全面清退 |
| AI 换脸 | ~1 年（2023） | 2024 年专项打击 |

## 实践：维护观察清单

跟踪趋势时，按以下结构记录：

```yaml
领域: "AI 生成内容"
首次出现时间: 2022-11
当前阶段: 立法跟进期
立法空白度: 中
执法可行性: 中
变现能力: 强
窗口期评估: 过渡期
关键信号:
  - 立法动态: "《生成式AI服务管理办法》已征求意见"
  - 执法案例: "2024年AI换脸诈骗首批判例"
  - 市场规模: "AIGC市场规模快速增长"
预计窗口关闭: 2025-2026
```

每个领域维护一张卡片，定期更新三个维度的评级变化。当任一维度从高降到中、或从中降到低时，是重要的退出信号。

## 适用范围

这个框架不局限于灰产分析，同样适用于：

- **创业机会评估**：判断一个新兴赛道是否值得切入
- **投资判断**：评估赛道窗口期和退出时机
- **技术趋势分析**：预判技术从灰色到合规的演进路径
- **职业选择**：判断新兴岗位的可持续性

## 注意事项

- 本框架仅用于趋势观察和分析，不构成任何商业或法律建议
- 立法空白不等于合法，执法盲区不等于安全
- 任何分析都应优先考虑法律合规风险
- 框架基于中国市场观察，其他法域的规律可能不同


---

# 提示词写作技巧：用 XML 结构构建可复用提示词
Date: 2026-06-21 | Tags: Prompt, GPT-5.5, XML结构, Skill | URL: https://bsheepcoder.github.io/2026/06/21/skill-prompt-writing/

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


---

# Kokoro TTS 本地语音合成引擎集成指南
Date: 2026-06-18 | Tags: AI, Skill, TTS | URL: https://bsheepcoder.github.io/2026/06/18/skill-kokoro-tts/

## 概述

Kokoro TTS 是一个高质量的开源 AI 语音合成引擎，支持本地部署，提供与 OpenAI TTS API 兼容的接口。它能够将文本转换为自然的语音，适用于聊天机器人语音消息、有声内容生成、无障碍辅助等场景。

本文以 derder-bot 项目中的 `kokoro-tts` skill 为例，介绍完整的集成方案。

### 核心特性

- **高质量语音**：基于 Kokoro 模型，生成自然流畅的人声
- **本地部署**：数据不出本地，隐私安全有保障
- **OpenAI 兼容 API**：`/v1/audio/speech` 接口，迁移成本低
- **多音色支持**：内置美式/英式男女声共 20+ 种音色
- **参数可调**：支持语速调节（0.25x ~ 4.0x）

## 工作原理

```
用户文本 → Node.js 脚本 → HTTP POST → Kokoro TTS API → MP3 音频 → 机器人发送语音消息
```

### 架构示意

```
┌─────────────┐     HTTP POST      ┌──────────────────┐     MP3      ┌──────────┐
│  tts.js     │  ──────────────→   │  Kokoro TTS API  │  ────────→   │  media/  │
│  (客户端)   │  JSON body         │  (localhost:8880) │              │  *.mp3   │
└─────────────┘                    └──────────────────┘              └──────────┘
```

derder-bot（基于 OpenClaw）通过读取脚本输出的 `MEDIA:` 行，自动将生成的 MP3 作为语音附件发送给用户。

## 环境配置

### 环境变量

Kokoro TTS skill 通过 `KOKORO_API_URL` 环境变量定位 API 服务：

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `KOKORO_API_URL` | `http://localhost:8880/v1/audio/speech` | Kokoro TTS API 地址 |

### 配置方式

在项目根目录的 `.env` 文件中添加：

```env
KOKORO_API_URL=http://your-server:port/v1/audio/speech
```

若使用本地部署的 Kokoro 实例，保持默认值即可，无需额外配置。

## 使用方法

### 命令格式

```bash
node skills/kokoro-tts/scripts/tts.js "<text>" [voice] [speed]
```

### 参数说明

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `text` | 是 | — | 要朗读的文本，需用引号包裹 |
| `voice` | 否 | `af_heart` | 音色 ID |
| `speed` | 否 | `1.0` | 语速，范围 0.25 ~ 4.0 |

### 使用示例

默认音色朗读：

```bash
node skills/kokoro-tts/scripts/tts.js "Hello, this is a test message."
```

指定音色（美式女声 nova）：

```bash
node skills/kokoro-tts/scripts/tts.js "Hello Ed, this is Theosaurus speaking." af_nova
```

调整语速（0.5 倍速慢读）：

```bash
node skills/kokoro-tts/scripts/tts.js "慢速朗读示例" af_heart 0.5
```

### 输出格式

脚本执行成功后，输出一行 `MEDIA:` 前缀的文件路径：

```
MEDIA: media/tts_1706745000000.mp3
```

OpenClaw / derder-bot 会自动识别该格式，并将文件作为语音附件发送。

## 脚本源码解析

以下是 `scripts/tts.js` 的完整实现：

```javascript
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');

// 配置：优先从环境变量读取 API 地址
const API_URL = process.env.KOKORO_API_URL || 'http://localhost:8880/v1/audio/speech';
const DEFAULT_VOICE = 'af_heart';
const DEFAULT_SPEED = 1.0;

// 解析命令行参数
const args = process.argv.slice(2);
if (args.length === 0) {
  console.error('Usage: node tts.js <text> [voice] [speed]');
  process.exit(1);
}

const text = args[0];
const voice = args[1] || DEFAULT_VOICE;
const speed = parseFloat(args[2] || DEFAULT_SPEED);

// 确保 media 目录存在
const mediaDir = path.join(process.cwd(), 'media');
if (!fs.existsSync(mediaDir)) {
  fs.mkdirSync(mediaDir, { recursive: true });
}

// 生成唯一文件名
const timestamp = Date.now();
const filename = `tts_${timestamp}.mp3`;
const filePath = path.join(mediaDir, filename);

async function generateSpeech() {
  try {
    // 调用 Kokoro TTS API（OpenAI 兼容格式）
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        input: text,           // 要朗读的文本
        voice: voice,          // 音色 ID
        speed: speed,          // 语速
        response_format: 'mp3', // 输出格式
        model: 'kokoro',       // 模型名称
        stream: false           // 非流式返回
      }),
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.status} ${response.statusText}`);
    }

    // 将音频写入文件
    const buffer = await response.arrayBuffer();
    fs.writeFileSync(filePath, Buffer.from(buffer));

    // 输出 OpenClaw 可识别的格式
    console.log(`MEDIA: ${path.relative(process.cwd(), filePath)}`);

  } catch (error) {
    console.error('Error generating speech:', error.message);
    process.exit(1);
  }
}

generateSpeech();
```

### 关键设计点

1. **环境变量优先**：`KOKORO_API_URL` 未设置时回退到 `localhost:8880`，本地开发零配置
2. **自动创建目录**：`media/` 目录不存在时自动递归创建
3. **时间戳命名**：用 `Date.now()` 生成唯一文件名，避免覆盖
4. **MEDIA 协议**：输出格式严格遵循 OpenClaw 的 `MEDIA:` 协议，实现机器人自动发送
5. **错误处理**：API 异常时输出错误信息并 `exit(1)`，便于上层捕获

## API 接口说明

Kokoro TTS 提供 OpenAI 兼容的 `/v1/audio/speech` 接口。

### 请求

```
POST /v1/audio/speech
Content-Type: application/json
```

请求体：

```json
{
  "input": "要朗读的文本",
  "voice": "af_heart",
  "speed": 1.0,
  "response_format": "mp3",
  "model": "kokoro",
  "stream": false
}
```

### 响应

返回二进制 MP3 音频流，`Content-Type: audio/mpeg`。

### curl 调用示例

```bash
curl -X POST http://localhost:8880/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"input":"Hello world","voice":"af_heart","speed":1.0}' \
  --output output.mp3
```

## 可用音色

### 美式英语 · 女声（American Female）

| 音色 ID | 风格 |
|---------|------|
| `af_heart` | 默认，温暖亲切 |
| `af_alloy` | 通用 |
| `af_bella` | 活泼 |
| `af_jessica` | 清晰 |
| `af_kore` | 柔和 |
| `af_nicole` | 沉稳 |
| `af_nova` | 专业 |
| `af_river` | 自然 |
| `af_sarah` | 优雅 |
| `af_sky` | 明亮 |

### 美式英语 · 男声（American Male）

| 音色 ID | 风格 |
|---------|------|
| `am_adam` | 浑厚 |
| `am_echo` | 回响 |
| `am_eric` | 标准 |
| `am_fenrir` | 低沉 |
| `am_liam` | 年轻 |
| `am_michael` | 成熟 |
| `am_onyx` | 深沉 |
| `am_puck` | 活泼 |
| `am_santa` | 欢乐 |

### 英式英语 · 女声（British Female）

| 音色 ID | 风格 |
|---------|------|
| `bf_alice` | 优雅 |
| `bf_emma` | 清新 |
| `bf_lily` | 甜美 |

### 英式英语 · 男声（British Male）

| 音色 ID | 风格 |
|---------|------|
| `bm_daniel` | 稳重 |
| `bm_fable` | 叙事 |
| `bm_george` | 厚重 |
| `bm_lewis` | 轻快 |

### 推荐音色

| 场景 | 推荐音色 | 理由 |
|------|---------|------|
| 默认使用 | `af_heart` | 温暖亲切，适用性广 |
| 专业播报 | `af_nova` | 清晰专业 |
| 浑厚男声 | `am_adam` | 低沉有力 |
| 英式口音 | `bf_alice` | 优雅英式女声 |

## 实际应用场景

### 1. 聊天机器人语音消息

在 derder-bot 中，当用户请求"说"某段文字时，skill 自动调用并返回语音：

```
用户：说一下"你好，欢迎使用"
Bot：[语音消息] 你好，欢迎使用
```

### 2. 有声内容生成

批量将文章转为音频，配合 RSS 日报生成语音播报：

```bash
node skills/kokoro-tts/scripts/tts.js "$(cat article.md)" af_nova
```

### 3. 无障碍辅助

为视障用户朗读屏幕内容，或为阅读困难的用户提供语音版本。

### 4. 多语言内容

配合翻译 API，实现"翻译 + 朗读"一体化工作流。

## 部署 Kokoro TTS 服务

### Docker 部署（推荐）

```bash
docker run -d \
  --name kokoro-tts \
  -p 8880:8880 \
  --gpus all \
  kokoro-tts:latest
```

### 验证服务

```bash
curl http://localhost:8880/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"input":"test","voice":"af_heart"}' \
  --output test.mp3
```

若返回有效的 MP3 文件，说明服务正常运行。

## 总结

Kokoro TTS 提供了轻量、高质量的本地语音合成方案。通过 OpenAI 兼容 API 和简洁的 Node.js 脚本，可以快速集成到各类聊天机器人项目中。核心优势在于：

- 零云端依赖，数据完全本地化
- OpenAI API 兼容，迁移成本低
- 20+ 音色覆盖主流场景
- 与 OpenClaw `MEDIA:` 协议无缝对接

## 参考

- [Kokoro TTS GitHub](https://github.com/kokoro-engage/kokoro-tts)
- [OpenAI TTS API 文档](https://platform.openai.com/docs/guides/text-to-speech)
- derder-bot 项目中的 `skills/kokoro-tts/` 目录


---

# RSS 日报生成工作流
Date: 2026-06-18 | Tags: 信息获取, RSS, Skill | URL: https://bsheepcoder.github.io/2026/06/18/skill-rss-daily/

## 适用场景

- 用户要求生成 RSS 日报、每日资讯、今日新闻
- 用户要求从 RSS 源抓取并筛选重要新闻
- 用户要求生成信息简报

## RSS 源清单

完整源清单参见文章：[RSS 源大全](https://bsheepcoder.github.io/2026/06/17/rss-source-collection/)

### 实际使用的源（已验证可访问）

| 领域 | 源名称 | RSS 地址 |
|------|--------|---------|
| AI 官方 | OpenAI Blog | `https://openai.com/blog/rss.xml` |
| AI 官方 | Google DeepMind | `https://deepmind.google/blog/rss.xml` |
| AI 官方 | Google AI Blog | `https://blog.google/technology/ai/rss/` |
| 学术 | arXiv AI | `https://rss.arxiv.org/rss/cs.AI` |
| 学术 | arXiv ML | `https://rss.arxiv.org/rss/cs.LG` |
| 学术 | arXiv NLP | `https://rss.arxiv.org/rss/cs.CL` |
| 学术 | arXiv CV | `https://rss.arxiv.org/rss/cs.CV` |
| 顶刊 | Nature | `https://www.nature.com/nature/articles.rss` |
| 科普 | ScienceDaily | `https://www.sciencedaily.com/rss/all.xml` |
| 科普 | New Scientist | `https://www.newscientist.com/section/news/feed/` |
| 技术 | Hacker News | `https://hnrss.org/frontpage` |
| 技术 | Hacker News AI | `https://hnrss.org/newest?q=AI` |
| 技术 | GitHub Blog | `https://github.blog/feed/` |
| 技术 | Ars Technica | `https://feeds.arstechnica.com/arstechnica/index` |
| 科技商业 | TechCrunch | `https://techcrunch.com/feed/` |
| 开发者 | Stack Overflow Blog | `https://stackoverflow.blog/feed/` |
| 开发者 | Dev.to | `https://dev.to/feed/` |
| 编程语言 | Rust Blog | `https://blog.rust-lang.org/feed.xml` |
| 编程语言 | Go Blog | `https://go.dev/blog/feed.atom` |
| 编程语言 | Python Blog | `https://blog.python.org/feeds/posts/default` |
| 编程语言 | TypeScript Blog | `https://devblogs.microsoft.com/typescript/feed/` |
| 编程语言 | Kotlin Blog | `https://blog.jetbrains.com/kotlin/feed/` |
| 中文博客 | 阮一峰 | `https://www.ruanyifeng.com/blog/atom.xml` |
| 中文博客 | OneV's Den | `https://onevcat.com/atom.xml` |
| 新闻 | NPR News | `https://feeds.npr.org/1001/rss.xml` |
| 新闻 | NHK World | `https://www3.nhk.or.jp/rss/news/cat0.xml` |
| 安全 | BleepingComputer | `https://www.bleepingcomputer.com/feed/` |

## 抓取策略

### 批量抓取

1. 将所有源分为 5-6 批，每批 5-6 个源
2. 使用 webfetch 工具并行抓取每批
3. 超时设置 15 秒
4. 失败的源跳过，不阻塞其他源

### 抓取流程

```
第1批：学术源（arXiv x4 + Nature + ScienceDaily）
第2批：AI 官方 + 科普（OpenAI + DeepMind + Google AI + New Scientist）
第3批：技术媒体（HN + GitHub + Ars Technica + TechCrunch + Stack Overflow）
第4批：编程语言（Rust + Go + Python + TypeScript + Kotlin）
第5批：中文 + 新闻 + 安全（阮一峰 + OneV's Den + NPR + NHK + BleepingComputer）
```

## 筛选标准

### 筛选维度（按权重排序）

1. **重要程度**（50%）：是否为突破性进展、重大事件、影响广泛
2. **领域多样性**（30%）：确保覆盖 AI、科学、安全、商业、新闻等多个领域
3. **时效性**（20%）：优先选择 24-48 小时内的新闻

### 筛选数量

- 固定选 **20 条**
- 按领域分组，不按死板的编号列表

### 排序规则

按重要程度分组排列，组内按时间排序。

## 文章格式

### 文件命名

```
rss-daily-YYYY-MM-DD.md
```

### Front-Matter

```yaml
---
title: "RSS 日报 · YYYY-MM-DD"
date: YYYY-MM-DD 08:00:00
categories:
  - [技术, 信息获取]
tags:
  - RSS
  - 日报
  - 信息获取
description: "YYYY年M月D日 RSS 日报，从 N 个可信源中精选 20 条重要资讯，涵盖 AI、科学、安全、商业、国际新闻等领域，按重要程度排列。"
---
```

### 正文结构（关键：美观 + 分析师解读）

**核心原则**：不是干巴巴的新闻列表，而是有温度、有洞察的信息简报。每条新闻都要有分析师视角的解读，帮助读者理解"为什么这很重要"。

```markdown
> **今日导语**：一段 2-3 句的开篇语，概括今日信息主旋律，像专业分析师的晨会开场。
>
> 📡 数据源：N 个 | 📊 筛选：20 条 | 🕐 截止时间
>
> 完整源清单：[RSS 源大全](/2026/06/17/rss-source-collection/)

---

## 🤖 AI 前沿

### 标题（用 emoji 或自然标题，不用编号）

> **来源**：xxx · **时间**：xxx

新闻摘要（2-3 句概括事实，客观陈述）。

**分析师解读**：这段是关键。用 1-2 句话解释为什么这条新闻重要、对行业意味着什么、读者应该关注什么。要有观点，不是复述新闻。可以用比喻、对比、历史参照等方式增强可读性。

🔗 [阅读原文](url)

---

### 下一条标题
...（同上格式）

---

## 🔬 科学突破

### 标题
...（同上格式）

---

## 🔒 安全警报

### 标题
...

---

## 💼 商业与资本

### 标题
...

---

## 🌍 国际视野

### 标题
...

---

## 📱 产品与生态

### 标题
...

---

## 📊 今日全景

| 领域 | 条数 | 亮点 |
|------|------|------|
| AI 前沿 | N | 一句话亮点 |
| 科学突破 | N | ... |
| 安全警报 | N | ... |
| ... | ... | ... |

> **明日关注**：1-2 句提示明天值得跟踪的事件或趋势。
```

### 排版要点

1. **分组而非编号**：用 `## 🤖 AI 前沿`、`## 🔬 科学突破` 等 emoji 标题分组，不用 `## 1.`、`## 2.` 死板编号
2. **每条新闻三段式**：摘要（事实）→ 分析师解读（洞察）→ 链接
3. **分析师解读必须有观点**：不是"这很重要"，而是"这意味着 X 格局可能改变，因为 Y"
4. **开篇有导语**：像晨报一样概括今日主旋律
5. **结尾有全景表 + 明日关注**：帮助读者快速回顾和前瞻
6. **适当使用加粗、引用块**：增强视觉层次
7. **emoji 适度**：分组标题用 emoji，正文不用

### 分析师解读写作指南

好的解读示例：
> **分析师解读**：SpaceX 短暂超越亚马逊不是噱头——它标志着"太空基建"正在被资本市场重新定价为"科技平台"而非"航天承包商"。当一家公司的估值逻辑从"发射次数"转向"星际基础设施"，估值天花板就消失了。

差的解读示例（禁止）：
> **分析师解读**：这条新闻非常重要，值得大家关注。

解读的角度可以是：
- 行业格局变化（谁赢谁输）
- 技术趋势信号（风向标意义）
- 资本市场逻辑（估值/投资含义）
- 社会影响（伦理/政策含义）
- 历史对比（类似事件参照）

## 部署

生成文章后执行标准部署流程：

```powershell
npx hexo clean; if ($?) { npx hexo g }; if ($?) { ... deploy }
```

## 注意事项

- 抓取量很大（arXiv 单源可返回 500KB+），webfetch 会被截断，需关注截断内容
- 部分源可能临时不可访问（网络/代理），跳过即可
- 日报日期为生成日期的第二天（如 6 月 17 日晚上抓取，生成 6 月 18 日日报）
- 不要包含重复新闻（多个源报道同一事件时只保留一条，选择可信度最高的源）
- 加密文章不参与日报
- **每条新闻必须有分析师解读，不能只列事实**
- **分组标题用 emoji，正文不用编号**


