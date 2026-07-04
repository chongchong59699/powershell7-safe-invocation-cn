# 项目概述
本项目是可发布到 GitHub 的 Codex skill 项目，封装 `powershell7-safe-invocation-cn` 中文 PowerShell 7 安全调用规范。

# 技术栈
- Codex Skill
- Markdown
- YAML
- PowerShell 7

# 启动方式
无应用启动流程。使用时将 `powershell7-safe-invocation-cn/` 子目录复制到 Codex skills 目录，例如 `~/.codex/skills/powershell7-safe-invocation-cn`，然后重启 Codex。

# 目录结构
- `README.md`: GitHub 项目说明，介绍 skill、安装方式、触发方式、使用效果示例。
- `.gitignore`: Git 忽略规则。
- `PROJECT_MEMORY.md`: 项目记忆文档。
- `powershell7-safe-invocation-cn/SKILL.md`: Codex skill 主说明文件。
- `powershell7-safe-invocation-cn/agents/openai.yaml`: Codex UI 元数据。
- `powershell7-safe-invocation-cn/references/powershell7-safe-patterns.md`: PowerShell 7 安全调用详细参考。

# 核心模块
- `powershell7-safe-invocation-cn`: 实际可安装 skill。用于指导 Codex 在 Windows 上安全执行 PowerShell 7 命令，处理引号、路径、原生命令参数、Start-Process、错误处理和高风险文件操作。

# 数据库说明
无数据库。

# 外部依赖
- Codex skills 机制。
- Windows PowerShell 7，即 `pwsh.exe`。

# 历史变更
- 2026-07-04：从本地已评估 skill 整理为 GitHub 可发布项目结构。
- 2026-07-04：补充 README，包含 skill 介绍、安装方式、触发方式、使用示例和实测效果。

# 注意事项
- skill 本体目录内不放 README，保持 Codex skill 目录简洁；GitHub 用户文档放在仓库根目录。
- 复杂 PowerShell 命令优先写 `.ps1` 后通过 `pwsh.exe -NoLogo -NoProfile -NonInteractive -File` 执行。
- 原生命令参数使用数组传递，复杂进程边界使用 `ProcessStartInfo.ArgumentList`。
