---
name: powershell7-safe-invocation-cn
description: 中文 PowerShell 7 安全调用规范。用于 Codex 在 Windows 上运行、编写、调试或审查 PowerShell 命令和脚本，尤其是 pwsh.exe、PowerShell 7 安装检查、原生命令参数、带引号路径、JSON/正则转义、Start-Process、文件操作、递归删除/移动/覆盖、shell 排错等场景。
---

# PowerShell 7 安全调用

## 基本原则

在 Windows 上优先使用 PowerShell 7：`pwsh.exe`。不要把 `powershell.exe` 当成 PowerShell 7；它通常是 Windows PowerShell 5.1。

执行前先判断环境：

```powershell
Get-Command pwsh -ErrorAction SilentlyContinue
$PSVersionTable.PSVersion
$PSNativeCommandArgumentPassing
```

如果没有找到 `pwsh`，或版本不是 7.x，先提醒用户安装 PowerShell 7，并说明效果会更稳定、更适合 Codex 执行脚本。不要静默退回 `powershell.exe` 执行复杂脚本；只有简单只读命令或用户明确允许时，才临时使用 `powershell.exe`。

## Codex 调用模板

短小、无复杂引号的命令可以使用：

```text
pwsh.exe -NoLogo -NoProfile -Command '<code>'
```

多行脚本、JSON、XML、正则、管道、重定向、复杂引号、非 ASCII 路径、文件批量变更、递归删除/移动/覆盖时，写临时 `.ps1`，再执行：

```text
pwsh.exe -NoLogo -NoProfile -NonInteractive -File script.ps1
```

临时脚本开头推荐：

```powershell
$ErrorActionPreference = 'Stop'
```

不要习惯性添加 `-ExecutionPolicy Bypass`。只有可信脚本确实被执行策略阻止，且用户或上下文允许时才添加。

## 原生命令

不要把原生命令拼成一个大字符串。每个参数都作为数组的一项：

```powershell
$exe = 'C:\Path With Spaces\tool.exe'
$args = @(
    '--input'
    'C:\Data Folder\input.json'
    '--name'
    'value with spaces'
    '--empty'
    ''
)

& $exe @args
$exitCode = $LASTEXITCODE
if ($exitCode -ne 0) {
    exit $exitCode
}
```

规则：

- 用 `&` 执行变量中的可执行文件路径。
- 原生命令后立即保存 `$LASTEXITCODE`。
- 不要丢弃空字符串参数。
- 不要使用 Bash 风格的 `\"` 转义。
- 不要用 `Invoke-Expression` 执行拼接出来的命令。
- 不要为了启动程序额外包一层 `cmd.exe /c`，除非确实需要 cmd 语义。

## Cmdlet 与文件路径

PowerShell cmdlet 使用 splatting 和终止错误：

```powershell
$params = @{
    LiteralPath = 'C:\Data[1]\input.txt'
    Destination = 'C:\Output'
    Force       = $true
    ErrorAction = 'Stop'
}

Copy-Item @params
```

规则：

- 真实路径优先用 `-LiteralPath`，只有故意使用通配符时才用 `-Path`。
- 不要用 `$LASTEXITCODE` 判断 cmdlet 成败。
- 需要失败即停止时设置 `$ErrorActionPreference = 'Stop'`，并给关键 cmdlet 加 `-ErrorAction Stop`。

## Start-Process

普通前台执行使用：

```powershell
& $exe @args
```

只有需要提权、新窗口、隐藏窗口、分离进程、文件关联打开等行为时才使用 `Start-Process`。Codex 启动后台服务或辅助进程时，除非用户明确需要可见窗口，优先隐藏窗口。

`Start-Process -ArgumentList` 会把数组合并成一个命令行字符串，不是可靠的结构化传参 API。复杂参数边界使用：

```powershell
$psi = [System.Diagnostics.ProcessStartInfo]::new()
$psi.FileName = $exe
$psi.UseShellExecute = $false
foreach ($arg in $args) {
    $psi.ArgumentList.Add($arg)
}
$process = [System.Diagnostics.Process]::Start($psi)
$process.WaitForExit()
if ($process.ExitCode -ne 0) {
    exit $process.ExitCode
}
```

## 文件变更安全

递归删除、移动或覆盖前必须：

- 解析预期根目录和目标路径的绝对路径。
- 验证目标位于预期根目录内。
- 拒绝空路径、磁盘根路径、用户主目录、工作区根目录等高风险目标，除非用户明确要求。
- 不要在 PowerShell 枚举路径后交给 `cmd.exe`、批处理或其他 shell 删除。

简单 `$target.StartsWith($root)` 不够，因为 `C:\WorkBackup` 会被误判成 `C:\Work` 的子路径。更多模式见 `references/powershell7-safe-patterns.md`。

## 何时读取参考文件

遇到以下情况，读取 `references/powershell7-safe-patterns.md`：

- 需要向用户解释如何安装或验证 PowerShell 7。
- 命令涉及多层引号、JSON、正则、here-string 或非 ASCII 路径。
- 需要处理 `Start-Process`、后台进程、stdout/stderr 分离捕获。
- 需要执行递归删除、移动、覆盖等高风险文件操作。
- PowerShell 语法或参数传递出现异常，需要排错。
