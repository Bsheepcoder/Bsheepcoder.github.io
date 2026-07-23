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
