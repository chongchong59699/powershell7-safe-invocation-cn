# PowerShell 7 Safe Invocation CN

中文 Codex Skill：帮助 Codex 在 Windows 上更安全、稳定地运行、编写、调试和审查 PowerShell 7 命令。

这个 skill 重点解决 Codex 执行 PowerShell 时常见的高频错误：

- 把 `powershell.exe` 误认为 PowerShell 7。
- 嵌套 `pwsh -Command "..."` 时外层 PowerShell 提前展开变量。
- 对带空格、方括号、`&` 的真实路径误用 `-Path`。
- 把原生命令拼成一个大字符串，导致参数边界丢失。
- 误以为 `Start-Process -ArgumentList` 是可靠的结构化传参 API。
- cmdlet 错误没有用 `-ErrorAction Stop`，失败后脚本仍继续运行。
- 递归删除、移动、覆盖文件时缺少路径边界校验。

## 目录结构

```text
powershell7-safe-invocation-cn/
├── README.md
├── PROJECT_MEMORY.md
├── .gitignore
└── powershell7-safe-invocation-cn/
    ├── SKILL.md
    ├── agents/
    │   └── openai.yaml
    └── references/
        └── powershell7-safe-patterns.md
```

实际可安装的 Codex skill 是子目录：

```text
powershell7-safe-invocation-cn/
```

也就是包含 `SKILL.md` 的那一层目录。

## 适用场景

当你希望 Codex 在 Windows 上执行以下任务时，可以使用这个 skill：

- 运行 PowerShell 7 命令或 `.ps1` 脚本。
- 编写带 JSON、正则、here-string、非 ASCII 路径的 PowerShell 脚本。
- 调用原生命令，例如 Java、Maven、Git、Node、Python、CLI 工具。
- 启动后台服务、隐藏窗口进程，或需要捕获 stdout/stderr。
- 处理带空格、方括号、特殊字符的文件路径。
- 执行递归删除、移动、覆盖等高风险文件操作。
- 排查 PowerShell 参数传递、引号、退出码和错误处理问题。

## 安装方式

### 方式一：复制到 Codex skills 目录

将本仓库中的 skill 子目录复制到 Codex skills 目录：

```powershell
$source = 'E:\java\workspace\powershell7-safe-invocation-cn\powershell7-safe-invocation-cn'
$dest = Join-Path $HOME '.codex\skills\powershell7-safe-invocation-cn'

if (Test-Path -LiteralPath $dest) {
    throw "Skill already exists: $dest"
}

Copy-Item -LiteralPath $source -Destination $dest -Recurse
```

安装后重启 Codex，让新 skill 被自动发现。

### 方式二：从 GitHub 克隆后复制

发布到 GitHub 后，可以这样安装：

```powershell
git clone https://github.com/chongchong59699/powershell7-safe-invocation-cn.git

$source = '.\powershell7-safe-invocation-cn\powershell7-safe-invocation-cn'
$dest = Join-Path $HOME '.codex\skills\powershell7-safe-invocation-cn'
Copy-Item -LiteralPath $source -Destination $dest -Recurse
```

### 方式三：使用 Codex skill installer

如果你的 Codex 环境支持从 GitHub 安装 skill，可以使用类似命令：

```text
安装 https://github.com/chongchong59699/powershell7-safe-invocation-cn/tree/main/powershell7-safe-invocation-cn
```

## 如何触发

安装并重启 Codex 后，可以在请求中直接写：

```text
使用 $powershell7-safe-invocation-cn 帮我安全执行这个 PowerShell 脚本
```

也可以用自然语言触发，例如：

```text
请在 Windows 上用 PowerShell 7 执行这个命令，注意路径、引号和错误处理。
```

## 使用效果示例

### 示例一：避免外层变量提前展开

不推荐：

```powershell
pwsh -Command "$PSVersionTable.PSVersion.ToString()"
```

这类写法从 PowerShell 外层再调用内层 `pwsh` 时，`$PSVersionTable` 可能先被外层展开，导致内层收到错误脚本。

推荐：

```powershell
pwsh.exe -NoLogo -NoProfile -Command '$PSVersionTable.PSVersion.ToString()'
```

复杂脚本推荐写入 `.ps1`：

```powershell
pwsh.exe -NoLogo -NoProfile -NonInteractive -File .\work\script.ps1
```

### 示例二：保留原生命令参数边界

不推荐：

```powershell
$cmd = "java -jar C:\Tools With Spaces\app.jar --name `"hello world`""
Invoke-Expression $cmd
```

推荐：

```powershell
$exe = 'java'
$args = @(
    '-jar'
    'C:\Tools With Spaces\app.jar'
    '--name'
    'hello world'
)

& $exe @args
$exitCode = $LASTEXITCODE
if ($exitCode -ne 0) {
    exit $exitCode
}
```

### 示例三：正确处理真实路径

不推荐：

```powershell
Get-Content -Path 'C:\Data[1]\input.txt'
```

推荐：

```powershell
Get-Content -LiteralPath 'C:\Data[1]\input.txt' -ErrorAction Stop
```

说明：真实路径优先使用 `-LiteralPath`，只有故意使用通配符时才使用 `-Path`。

### 示例四：复杂进程参数使用 ProcessStartInfo

不推荐把复杂参数交给 `Start-Process -ArgumentList` 数组，因为它会合并成一个命令行字符串。

推荐：

```powershell
$psi = [System.Diagnostics.ProcessStartInfo]::new()
$psi.FileName = 'pwsh.exe'
$psi.UseShellExecute = $false
$psi.RedirectStandardOutput = $true
$psi.RedirectStandardError = $true

foreach ($arg in @('-NoLogo', '-NoProfile', '-File', '.\script.ps1', 'value with spaces')) {
    [void]$psi.ArgumentList.Add($arg)
}

$process = [System.Diagnostics.Process]::Start($psi)
$stdout = $process.StandardOutput.ReadToEnd()
$stderr = $process.StandardError.ReadToEnd()
$process.WaitForExit()

if ($process.ExitCode -ne 0) {
    throw "Command failed with exit code $($process.ExitCode): $stderr"
}
```

## 实测效果

在本地 PowerShell 7.6.2 环境下，对安装前后进行了同一组测试：

| 阶段 | 通过 | 真实失败 | 预期风险失败 |
| --- | ---: | ---: | ---: |
| 安装前 | 9 | 4 | 2 |
| 安装后 | 13 | 0 | 2 |

安装后真实失败从 4 个降为 0 个。剩余 2 个是故意保留的风险证明：

- 外层双引号会导致 `$PSVersionTable` 提前展开。
- 对包含方括号的真实路径使用 `-Path` 会触发通配符语义。

## 发布建议

上传 GitHub 时建议仓库结构保持当前形式：

- 仓库根目录放 `README.md`，方便用户阅读和安装。
- skill 本体只保留 `SKILL.md`、`agents/`、`references/` 等必要文件。
- 不要把测试日志、临时目录、IDE 配置、压缩包提交到仓库。

## License

本项目基于 [MIT License](LICENSE) 开源。
