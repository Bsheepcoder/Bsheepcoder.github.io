# 分类：开发环境

共 1 篇文章

---

# Windows Terminal 实用指南：把一个窗口变成统一开发工作台
Date: 2026-06-30 | Tags: 开发环境, Windows Terminal, WSL, PowerShell | URL: https://bsheepcoder.github.io/2026/06/30/dev-windows-terminal/

## 前言

很多人的 Windows Terminal（后文简称 WT）只用了两成能力：开一个标签、敲命令、关掉。但在 **Windows + WSL + Jetson Nano + Docker + OpenCode + AI Agent** 这种多环境开发场景下，它完全可以配置成一个统一工作台——一个窗口管所有环境，几乎不用切窗口。

这篇文章是实用指南，不是功能清单。每条配置都附一句"为什么这么做"，末尾会从原理层批判它的真实边界，告诉你什么情况下才真需要换 WezTerm。

---

## 一、先理解：Windows Terminal 不是 Shell

这是最容易被忽略的认知前提。**Windows Terminal 本身不是 Shell，而是一个终端管理器（terminal emulator / multiplexer UI）。**

它只负责渲染字符、接收输入、管理标签和分屏。真正执行命令的是背后的 Shell：

```
Windows Terminal（渲染层）
        │
        ├── PowerShell 7      ← Shell
        ├── CMD               ← Shell
        ├── Git Bash          ← Shell
        ├── Ubuntu (WSL)      ← Shell
        ├── Debian (WSL)      ← Shell
        ├── ssh jetson        ← 远程 Shell
        ├── ssh vps           ← 远程 Shell
        └── Python / Node     ← REPL
```

**为什么这很重要**：理解了三层分离（终端是渲染层、Shell 是执行层、multiplexer 是会话层），你才能理解后面所有配置的本质——Profile 不是"换皮肤"，是"把不同 Shell + 连接参数打包成一个入口"。

它的最大价值用一句话概括：

> **一个窗口管理所有开发环境。**

---

## 二、装 PowerShell 7 并设为默认

不要用系统自带的 Windows PowerShell 5.1。

**为什么**：PowerShell 5.1 只随 Windows 更新，版本停滞；PowerShell 7 是独立安装的跨平台版本（基于 .NET），迭代更快、性能更好、对现代工具链（Oh My Posh、AI CLI、JSON 处理）支持更友好。

安装后，在 Windows Terminal 设置里把默认 Profile 改成 PowerShell 7。这样每次 `Ctrl+Shift+T` 新开标签就是 PS7，不用再手动选。

---

## 三、Profile：把每台机器抽象成一个入口

Windows Terminal 的 Profile 不是主题，是**一组启动参数的预设**——指定 Shell 路径、启动目录、图标、标题、配色。建议至少建这些：

| Profile | 用途 | 启动命令 |
|---------|------|---------|
| PowerShell 7 | 本机开发 | `pwsh.exe` |
| Ubuntu | WSL 开发 | `wsl.exe -d Ubuntu` |
| Jetson Nano | 远程开发 | `ssh jetson` |
| Docker | 容器管理 | `ssh docker-host` 或 `wsl docker` |
| Git Bash | 兼容 Unix 脚本 | `bash.exe` |

**为什么这么做**：没有 Profile，每次连 Jetson 都要手敲 `ssh jetson@192.168.31.10`；有了 Profile，一次点击直接进入。Profile 把"连接参数"这个重复劳动固化成了入口。

建好之后，用数字快捷键直接启动：

```
Ctrl+Shift+1   →  Profile 1（PowerShell）
Ctrl+Shift+2   →  Profile 2（Ubuntu）
Ctrl+Shift+3   →  Profile 3（Jetson）
...
```

数字对应 Profile 在下拉列表中的顺序。调好顺序后，全键盘操作，不用碰鼠标。

---

## 四、SSH Profile + ~/.ssh/config：远程会话归一

把每台远程机器建一个 Profile，配合 `~/.ssh/config` 把连接参数归一。

先写 SSH 配置（位于 `C:\Users\你的用户名\.ssh\config`）：

```ssh-config
Host jetson
    HostName 192.168.31.10
    User jetson
    IdentityFile ~/.ssh/id_ed25519

Host vps
    HostName your-vps-ip
    User root
    IdentityFile ~/.ssh/id_ed25519
```

然后 Windows Terminal 里建一个 Profile，命令行就写 `ssh jetson`，一点即连。

**为什么这么做**：没有 `~/.ssh/config`，每次连机器要敲 IP、用户名、密钥路径；有了它，`ssh jetson` 一行解决，而且 `scp`、`rsync`、`git` 都能复用这个别名。Profile + SSH config 是两层固化：config 归一连接参数，Profile 把连接参数变成可视入口。

建议给每台机器配独立图标和配色，一眼能认出现在在哪个环境——远程机器误敲 `rm -rf` 的代价远高于本地。

---

## 五、Pane 分屏：比 Tab 更该学

很多人只会开 Tab（标签页），其实 **Pane（分屏）** 更重要。

Tab 是切换关系（同时只看一个），Pane 是并列关系（同时看多个）。开发场景里，你经常需要同时盯两件事：一边跑服务、一边看日志；一边写代码、一边跑测试。

```
┌──────────────┬──────────────┐
│   Ubuntu     │  Jetson Nano  │
├──────────────┼──────────────┤
│ Docker Logs  │  AI Service   │
└──────────────┴──────────────┘
```

快捷键：

| 快捷键 | 作用 |
|--------|------|
| `Alt+Shift++` | 左右分屏（新 Pane 在右侧） |
| `Alt+Shift+-` | 上下分屏（新 Pane 在下方） |
| `Alt+Shift+D` | 复制当前 Profile 分屏 |
| `Alt+方向键` | 在 Pane 间移动焦点 |
| `Ctrl+Shift+W` | 关闭当前 Pane |

**为什么这么做**：Tab 解决的是"环境隔离"（一个 Tab 一个项目），Pane 解决的是"任务并列"（一个屏幕同时看多个任务）。真正的效率来自减少切换，Pane 让你一次看全，Tab 让你按需切。两者搭配才完整。

快捷键不顺手可以在设置里改，Windows Terminal 的快捷键是 JSON 配置，完全可定制。

---

## 六、多标签与 Command Palette

多标签用于按项目或按机器组织：

```
[ Project A ] [ Project B ] [ Jetson ] [ Docker ] [ Logs ]
```

每个标签内部再分 Pane。一个标签就是一个"工作区切片"。

另外一个被低估的功能是 **Command Palette**（命令面板）：

```
Ctrl+Shift+P
```

打开后可以搜索所有命令：新建 Tab、新建 Pane、切换 Profile、重命名标签、复制、粘贴、查找……类似 VS Code 的 `Ctrl+Shift+P`。

**为什么这么做**：命令面板是"快捷键的兜底"——记不住快捷键的操作，在这里搜一下就能执行，不用去翻菜单。对新手尤其友好，降低记忆负担。

---

## 七、启动即就位：自动开多 Tab/Pane

每次开 Terminal 都要手动点开三四个标签很烦。Windows Terminal 支持 **startup actions**，启动时自动执行命令。

在 `settings.json` 里配置：

```json
{
  "startupActions": "--maximized; new-tab -p \"PowerShell\" ; split-pane -V -p \"Ubuntu\" ; focus-pane -t 0 ; new-tab -p \"Jetson\" ; split-pane -H -p \"Docker\""
}
```

这段配置的效果：启动时窗口最大化，第一个标签左右分屏（左 PowerShell、右 Ubuntu），第二个标签上下分屏（上 Jetson、下 Docker）。打开 Terminal 就是你要的工作台，不用再点。

**为什么这么做**：把"准备环境"这个动作从每次手动执行变成一次配置永久生效。这是从"用工具"到"配置工具"的一步——工具的配置成本应该是一次性的，不该每次重复付。

---

## 八、主题与字体

Windows Terminal 内置多套配色，也可自定义。常见主题：

```
Dracula / Nord / Tokyo Night / Catppuccin / One Dark / Gruvbox
```

字体建议（等宽 + 编程连字支持）：

| 字体 | 特点 |
|------|------|
| Cascadia Code | 微软官方，WT 默认，支持连字 |
| JetBrains Mono | JetBrains 出品，清晰锐利 |
| Maple Mono | 中文显示友好 |
| FiraCode | 经典编程字体，连字丰富 |

再开启 **Ligatures**（连字）和安装 **Nerd Font** 版本（含图标字符）：

```
Cascadia Code NF
JetBrainsMono Nerd Font
```

**为什么这么做**：Nerd Font 内嵌大量图标（git 分支、文件夹、语言 logo），配合 Oh My Posh 能在提示符里显示图形化状态。不开 Nerd Font，提示符里的图标会变成豆腐块方块。字体不是审美问题，是功能依赖。

---

## 九、提示符：Oh My Posh 与 Starship 怎么选

裸的 PowerShell 提示符只有路径和 `>`。装上提示符引擎后，能显示：

```
PS D:\Code\Hexo  git:main ✚  Node:18  Python:3.12  耗时:2.3s
```

两个主流选择：

| | Oh My Posh | Starship |
|---|------------|----------|
| 平台 | Windows 原生，与 PowerShell 深度集成 | 跨平台，Win/Linux/Mac 统一 |
| 配置 | JSON / YAML | TOML |
| 生态 | Windows 社区主流 | 全平台统一 |
| 适合 | 只用 Windows + WSL | 多平台切换，要统一体验 |

**怎么选**：

- 只在 Windows + WSL 开发 → **Oh My Posh**，与 PowerShell 集成最深，配置示例最多
- 经常在 Windows / Linux / Mac 之间切换，要提示符完全一样 → **Starship**，一份配置走天下

两者都能显示 Git 分支、Git 状态、Node/Python 版本、K8s 上下文、AWS profile、电池电量等。选哪个看你的平台跨度，不看功能多少。

---

## 十、推荐布局：覆盖 Win+WSL+Jetson+Docker 的四宫格

针对多环境开发，建议固定一个布局，形成肌肉记忆：

```
┌─────────────────────────────────────┐
│  Tab1: 开发                          │
│                                     │
│   Ubuntu (WSL)  │  PowerShell       │
│─────────────────┼───────────────────│
│   Jetson SSH    │  Docker Logs      │
└─────────────────────────────────────┘

  Tab2: OpenCode / AI CLI
  Tab3: Git 操作
  Tab4: Server / 杂项
```

**为什么这么分**：

- 左上 Ubuntu 写代码，右上 PowerShell 跑 Windows 侧命令（`hexo`、`npm`）——本地两个环境并排
- 左下 Jetson 看远程状态，右下 Docker 看容器日志——远程两个监控并排
- 上下分界是"本地 / 远程"，左右分界是"主操作 / 辅助监控"

这种布局的好处是**视线不需要离开屏幕中心**，所有关键信息都在视野内。配合第七节的 startup actions，每次打开 Terminal 自动就位。

---

## 十一、与 AI / CLI 工具协同

Windows Terminal 不限制任何 CLI 工具，以下都能直接跑：

- **OpenCode / Claude Code / Gemini CLI** —— AI Agent 终端工具
- **GitHub CLI**（`gh`）—— PR、Issue、Actions 管理
- **Docker CLI** —— 容器管理
- **Git** —— 版本控制
- **Node.js / Python** —— 运行时 REPL

**为什么单独提**：终端是渲染层，不关心上面跑什么。AI CLI 工具（OpenCode、Claude Code）和传统工具（git、docker）在终端眼里没区别，都能开 Pane 并排跑。把 AI Agent 放一个 Pane、把要操作的代码放另一个 Pane，就是最简单的"AI + 人工"协同工作流。

---

## 十二、评分与边界：为什么 Workspace 和 AI 只有两颗星

| 能力 | 评分 | 说明 |
|------|------|------|
| 分屏 | ⭐⭐⭐⭐☆ | 够用，但无法保存布局为命名工作区 |
| 多标签 | ⭐⭐⭐⭐⭐ | 完整 |
| SSH | ⭐⭐⭐⭐⭐ | Profile + ssh config，一点即连 |
| WSL | ⭐⭐⭐⭐⭐ | 原生集成，无感切换 |
| 性能 | ⭐⭐⭐⭐⭐ | 基于 DirectX 渲染，流畅 |
| 可定制 | ⭐⭐⭐⭐☆ | JSON 配置灵活，但不可编程 |
| Workspace | ⭐⭐☆☆☆ | **弱项** |
| AI 能力 | ⭐⭐☆☆☆ | 依赖外部 CLI |

**Workspace 为什么弱**：Windows Terminal 没有"会话持久化"。你分了四个 Pane、跑了三个长任务，一旦关闭 Terminal——所有 Pane 布局消失，长任务中断。它只有 `startupActions` 这种"启动时重建布局"的静态方案，没有 tmux 那种 `detach`（脱离会话）/ `attach`（重新接入）的动态会话保持能力。

> **这就是为什么远程长期任务必须用 tmux**——tmux 跑在 Jetson 或 Linux 服务器上，Terminal 关了 tmux 会话还在，下次 `tmux attach` 接回来，任务没断。Windows Terminal 本身替代不了这个能力，因为它只是渲染层，会话生命周期绑在终端进程上。

**AI 能力为什么两颗星**：不是 WT 不支持 AI，而是终端本质上就是渲染层，AI 能力来自外部 CLI（OpenCode、Claude Code、GitHub Copilot CLI）。WT 没有内置 AI 集成（如自然语言转命令、上下文感知补全），这块完全靠外部工具补。这是架构决定的——终端不该承担 AI 逻辑，但它也没提供与 AI 工具深度集成的原生接口（如把 Pane 上下文喂给 AI）。

**可定制为什么四颗星不是五星**：配置是 JSON，不是代码。JSON 能声明"做什么"，但不能写逻辑（条件判断、循环、函数）。对比 WezTerm 用 Lua 配置，可以写函数动态生成窗口布局、根据时间切主题、根据项目自动开 Pane。Windows Terminal 的 JSON 是静态声明，够用但不可编程。

---

## 结语：什么情况下才真需要换 WezTerm

把以上配置做完，Windows Terminal 足以覆盖绝大多数开发需求。真正需要切换到 WezTerm 或其他终端，通常是因为以下触发条件：

1. **需要持久化工作区**：要保存多个命名布局、随时切换、关闭终端后会话不丢——WT 没有，tmux + 任意终端或 WezTerm 的工作区管理更适合
2. **需要可编程配置**：要用代码动态生成窗口布局、根据项目自动开 Pane、写条件逻辑——WezTerm 的 Lua 配置能做到，WT 的 JSON 做不到
3. **跨平台统一配置**：在 Win/Linux/Mac 之间频繁切换，要一份配置走天下——WezTerm 或 Alacritty + Starship 更合适
4. **需要更强的渲染控制**：GPU 渲染调优、自定义着色器——WezTerm 的渲染管线更灵活

如果只是以上条件之外的需求——多环境管理、SSH、WSL、分屏、提示符、主题——Windows Terminal 配好后体验已经很好，**切换的收益不一定覆盖迁移成本**。

工具的配置成本应该是一次性的。把 Windows Terminal 配到位，比追逐"最强终端"更值得。


