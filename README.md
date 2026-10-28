# win-powershell-next

让 coding agent 在 Windows 上一次写对 shell 命令的 Agent Skill。

面向 AI coding agent 的 Windows 命令执行规范，执行环境以 **Windows PowerShell 5.1（`powershell.exe`）** 为准。核心目标是**减少因 shell 语法、引号、编码、退出码误判导致的试错**——这些问题在 Windows 上尤其高发，且每次失败都要消耗一轮上下文。

## 为什么需要它

Agent 在 Windows 上最常见的失败模式：

- 把 `rm -rf`、`ls -la`、`cp -r`、`VAR=x cmd` 直接搬过来 → 参数解析报错
- 用 `cat` / `find` / `where` → 别名歧义或调错可执行文件
- 写 `a && b` / `$x ? 1 : 2` → 5.1 直接语法错误
- `>` 重定向写出 UTF-16LE → 后续读取中文乱码
- `-Encoding utf8NoBOM` → 5.1 不认这个值
- 只看 `$?` 不看 `$LASTEXITCODE` → 原生命令失败被当成成功
- `Read-Host` / `git` 凭据提示 / `winget` 首次协议 → 命令永久挂起
- 用 `Start-Process` 想拿 stdout → 什么都拿不到

本 skill 把这些坑固化成可查的规则和模板，agent 加载后无需现场推导。

## 特性

- **以 5.1 为基准**：默认 `powershell.exe`，明确不强制 PowerShell 7，并给出「5.1 vs 7」禁用清单（`&&`、`||`、三元、`??`、`utf8NoBOM`、`$PSStyle`、`$IsWindows`、`-Parallel` 等）
- **Unix → PowerShell 速查表**：覆盖 30+ 高频命令映射，每行标注 5.1 的坑
- **引号与 here-string 规范**：单双引号选择、反引号转义、here-string 边界规则
- **编码与 BOM 指南**：5.1 的 UTF-16LE 默认值、`utf8` 带 BOM、无 BOM 写法、`.ps1` 必须带 BOM
- **退出码与错误处理模板**：`$LASTEXITCODE` vs `$?`；5.1 没有 `$PSNativeCommandUseErrorActionPreference`，必须手动判断
- **超时与交互防护**：`GIT_TERMINAL_PROMPT`、`winget --disable-interactivity`、`Start-Job` + `Wait-Job -Timeout`
- **检索优先 `rg`**：含 `-F`、`-uu`、退出码语义（`1` = 无匹配 ≠ 错误）、不在 PATH 时的全路径调用

## 目录结构

```text
win-powershell-next/
├── SKILL.md          # 技能本体（frontmatter + 指令正文）
└── README.md         # 本文件
```

如后续需要拆分，可增加 `references/`（按需加载的长文档）或 `scripts/`（可执行辅助脚本）。SKILL.md 建议保持在 500 行以内。

## 安装

把 `win-powershell-next/` 整个目录放进所用 Agent 的技能目录即可。常见位置：

| Agent       | 技能目录                            |
| ----------- | ----------------------------------- |
| Claude Code | `~/.claude/skills/`                 |
| Cursor      | `.cursor/skills/`                   |
| Gemini CLI  | `~/.gemini/skills/`                 |
| 其他        | 遵循各 Agent 文档中的 "skills" 路径 |

确保目录名与 frontmatter 中的 `name` 一致（`win-powershell-next`），否则技能无法加载。

验证：让 agent 生成一条 Windows 命令，观察它是否使用 `powershell.exe` 能跑的 5.1 语法（没有 `&&`、没有 `utf8NoBOM`）、是否优先 `rg`。

## 使用

Skill 由 Agent 根据 `description` 自动触发，无需手动调用。触发场景包括：

- 在 Windows 上生成或执行任何 shell 命令
- 宿主 shell 是 PowerShell / cmd / Git Bash / WSL，agent 可能选错 shell
- 涉及文件检索、文本搜索、命令退出码判断

## 环境要求

| 组件       | 要求                  | 说明                                              |
| ---------- | --------------------- | ------------------------------------------------- |
| PowerShell | 5.1（`powershell.exe`） | Windows 8.1 / Server 2012 R2 起系统自带，无需安装 |
| ripgrep    | 任意较新版本（可选）  | `winget install --id BurntSushi.ripgrep.MSVC -e`   |

- **不需要 PowerShell 7**。装了也不冲突，本 skill 只用 5.1 语法。
- rg 缺失时降级为 `Get-ChildItem | Select-String`，并在回复里说明降级原因。
- 安装任何东西前，skill 会要求 agent **先征得用户同意**，不会擅自运行。

## 从 1.x 升级

1.x 强制 `pwsh` 7。2.0 起改为以 `powershell.exe`（5.1）为准：

- 命令模板里的 `pwsh` 全部替换为 `powershell.exe`，并加 `-ExecutionPolicy Bypass`
- 正文示例改为 5.1 语法，删除 `&&` / `||` / 三元 / `utf8NoBOM` / `$PSStyle` 等 7 专有写法
- 环境自检不再要求 major = 7，改为识别 `PSEdition`
- 原「5.1 兜底」章节升级为「5.1 与 7 的语法差异（禁用清单）」，成为正文核心

## 格式兼容性

`SKILL.md` 遵循 [Agent Skills Specification](https://agentskills.io/specification)：

- frontmatter 仅使用标准字段（`name`、`description`），可选 `license` / `metadata`
- `name` 匹配 `^[a-z0-9]+(-[a-z0-9]+)*$`，且与父目录同名
- 正文采用渐进式披露：frontmatter 常驻上下文，正文按需加载

## 维护

改动 SKILL.md 时请同步：

1. 更新 frontmatter 中的 `version`
2. 确认 `description` 仍覆盖所有触发关键词（它是 Agent 决定是否加载的唯一依据）
3. 新增规则时优先补充速查表或可复制模板，而非再写一段原则说明
4. **任何新增示例都必须在 `powershell.exe` 5.1 上实测可跑**，不要引入 7 专有语法

## License

MIT
