# PowerShell 7 安全调用参考

## 1. PowerShell 7 安装与验证

在 Windows 上，`pwsh.exe` 是 PowerShell 7，`powershell.exe` 通常是 Windows PowerShell 5.1。安装 PowerShell 7 不会把 `powershell.exe` 自动变成 PowerShell 7。

验证命令：

```powershell
Get-Command pwsh -ErrorAction SilentlyContinue
pwsh -NoLogo -NoProfile -Command '$PSVersionTable.PSVersion; $PSNativeCommandArgumentPassing'
```

如果 `pwsh` 不存在，提醒用户安装 PowerShell 7。可给出简短建议：

```text
当前环境没有检测到 PowerShell 7。建议安装 PowerShell 7 后再执行复杂脚本，Codex 处理原生命令参数、引号、JSON 和跨平台行为会更稳定。安装后请确认 `pwsh.exe` 可在 PATH 中访问。
```

不要自行联网安装，除非用户明确要求。可以提示用户选择官方安装方式，例如 Microsoft Store、winget、MSI 安装包或企业软件分发。

常见安装命令示例，只有在用户明确要求安装时才执行：

```powershell
winget install --id Microsoft.PowerShell --source winget
```

## 2. `-Command` 与外层 shell 嵌套

从 PowerShell 中调用另一个 `pwsh -Command "..."` 时，外层双引号会先展开变量：

```powershell
# 容易出错：$LASTEXITCODE 可能被外层提前展开
pwsh -Command "$x = 1; $LASTEXITCODE"
```

简单场景优先使用外层单引号：

```powershell
pwsh -NoLogo -NoProfile -Command '$x = 1; $x'
```

复杂场景不要继续堆转义，写 `.ps1`：

```powershell
$ErrorActionPreference = 'Stop'
$data = [ordered]@{
    name = 'test'
    path = 'C:\Path With Spaces\data.json'
}
$data | ConvertTo-Json -Depth 10
```

执行：

```text
pwsh.exe -NoLogo -NoProfile -NonInteractive -File .\work\script.ps1
```

## 3. JSON、here-string 与编码

生成 JSON 时优先创建对象并序列化：

```powershell
$data = [ordered]@{
    name = $name
    path = $path
    flags = @('a', 'b')
}

$data |
    ConvertTo-Json -Depth 10 |
    Set-Content -LiteralPath $jsonPath -Encoding utf8
```

字面量多行文本使用单引号 here-string：

```powershell
$text = @'
{
  "name": "$literal"
}
'@
```

关闭标记必须独占一行，并从行首开始。

文本文件被其他工具消费时，显式指定编码：

```powershell
Set-Content -LiteralPath $path -Value $text -Encoding utf8
```

二进制内容使用字节 API：

```powershell
[System.IO.File]::WriteAllBytes($path, $bytes)
```

## 4. 原生命令参数调试

当怀疑参数被破坏时，打印参数和值长度：

```powershell
$args | ForEach-Object {
    '[{0}] Length={1}' -f $_, $_.Length
}
```

排错顺序：

1. 去掉 `cmd.exe /c`。
2. 去掉 `pwsh -Command` 的嵌套。
3. 去掉 `Invoke-Expression`。
4. 去掉手写嵌套引号。
5. 写最小 `.ps1` 文件。
6. 用 `& $exe @args` 调用原生命令。
7. 打印每个参数和长度。
8. 立即保存 `$LASTEXITCODE`。

注意：某些工具有特殊退出码约定，不要在不了解工具文档时机械套用非零即失败。

## 5. `Start-Process` 与后台进程

普通 CLI 程序：

```powershell
& $exe @args
```

后台或隐藏窗口：

```powershell
$process = Start-Process `
    -FilePath $exe `
    -ArgumentList $argumentString `
    -WindowStyle Hidden `
    -PassThru
```

复杂参数边界或需要分离 stdout/stderr 时，使用 `ProcessStartInfo`：

```powershell
$psi = [System.Diagnostics.ProcessStartInfo]::new()
$psi.FileName = $exe
$psi.UseShellExecute = $false
$psi.RedirectStandardOutput = $true
$psi.RedirectStandardError = $true

foreach ($arg in $args) {
    $psi.ArgumentList.Add($arg)
}

$process = [System.Diagnostics.Process]::Start($psi)
$stdout = $process.StandardOutput.ReadToEnd()
$stderr = $process.StandardError.ReadToEnd()
$process.WaitForExit()

if ($process.ExitCode -ne 0) {
    throw "Command failed with exit code $($process.ExitCode): $stderr"
}
```

## 6. 递归文件操作保护

对已存在目标：

```powershell
$root = (Resolve-Path -LiteralPath 'C:\ExpectedRoot').Path
$target = (Resolve-Path -LiteralPath $candidate).Path

$rootPrefix = $root.TrimEnd(
    [System.IO.Path]::DirectorySeparatorChar,
    [System.IO.Path]::AltDirectorySeparatorChar
) + [System.IO.Path]::DirectorySeparatorChar

if (-not $target.StartsWith(
    $rootPrefix,
    [System.StringComparison]::OrdinalIgnoreCase
)) {
    throw "Refusing to modify path outside expected root: $target"
}

Remove-Item -LiteralPath $target -Recurse -Force
```

对可能尚不存在的目标，用 `[System.IO.Path]::GetFullPath()` 归一化路径，再做同样的父子目录校验。

始终拒绝：

- 空字符串或 `$null`。
- 文件系统根目录，例如 `C:\`。
- 用户主目录。
- 工作区根目录本身，除非用户明确要求。
- 无法解析或明显超出预期范围的路径。

## 7. PowerShell 语法易错点

不要直接把 `foreach` 语句接管道：

```powershell
# 错误或不稳定
foreach ($item in $items) {
    [pscustomobject]@{ Name = $item.Name }
} | Format-Table
```

先收集结果：

```powershell
$results = foreach ($item in $items) {
    [pscustomobject]@{ Name = $item.Name }
}

$results | Format-Table
```

避免反引号续行，使用数组、hashtable、括号或自然换行。

## 8. 推荐决策顺序

优先选择最简单且安全的方式：

1. PowerShell cmdlet。
2. `& $exe @args`。
3. 临时 `.ps1` 加 `pwsh.exe -File`。
4. `ProcessStartInfo.ArgumentList`。
5. 需要特殊窗口、提权、分离进程时使用 `Start-Process`。
6. 只有确实需要 cmd 语义时使用 `cmd.exe /c`。
7. `Invoke-Expression` 只作为可信 PowerShell 源码的最后选择。
