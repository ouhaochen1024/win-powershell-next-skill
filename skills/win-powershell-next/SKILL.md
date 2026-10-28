---
name: win-powershell-next
version: 1.2.0
display_name: Windows 命令执行 Next
display_name_en: Win PowerShell Next
description: Windows command-execution guide for coding agents. Always use PowerShell 7 (pwsh), never powershell.exe (5.1) / cmd / Git Bash / WSL for normal commands. Covers Unix-to-PowerShell syntax mapping, quoting and here-strings, exit codes, UTF-8 output, timeout guards, and rg-based search. Trigger when generating or executing any shell command on Windows, or when the host shell is PowerShell / cmd / Git Bash / WSL. Also trigger when the user asks to find files, search text, delete or recreate a directory, read/write files, or check command exit codes on Windows.
description_zh: Windows 命令执行规范 —— 强制 PowerShell 7，Unix→PowerShell 语法对照，引号与 here-string，退出码与错误处理，UTF-8 输出，超时防护，检索优先 rg

description_en: Windows PowerShell 7 command execution and rg-based search guide for coding agents
---

# Win PowerShell Next

在 Windows 上生成或执行任何 shell 命令时遵循本 skill。核心目标：**一次写对，不靠试错**。

## 1. 硬规则

1. 使用 PowerShell 7（`pwsh`）。不调用 `powershell.exe`（会降级到 5.1）、`cmd.exe`、Git Bash、WSL 来执行常规命令。
2. 确需独立进程时用 `pwsh.exe -NoLogo -NoProfile -NonInteractive`。`-NoProfile` 防止用户 profile 改别名/`$ErrorActionPreference`/打印横幅；`-NonInteractive` 防止卡在 `Read-Host`。
3. 用 PowerShell 原生语法，不照搬 Unix 命令。对照表见 §3。
4. 文本检索用 `rg`，文件枚举用 `rg --files`。
5. 命令失败先看错误原文、`$LASTEXITCODE`、引号和路径，不要为了“绕过去”切换 shell。

## 2. 环境自检（首次会话跑一次）

```powershell
"pwsh major: $($PSVersionTable.PSVersion.Major)"          # 期望 7
"edition:    $($PSVersionTable.PSEdition)"                # 7 = Core，5.1 = Desktop
$rg = Get-Command rg -ErrorAction SilentlyContinue
"rg path:    $(if ($rg) { $rg.Source } else { 'MISSING' })"
```

- **只有 5.1（major = 5）** → 按 §12 兜底；若用户同意，仍优先安装 pwsh 7。
- **pwsh 缺失** → 征得用户同意后：`winget install --id Microsoft.PowerShell --exact --source winget --accept-source-agreements --accept-package-agreements --disable-interactivity`
- **rg 缺失** → 征得用户同意后：`winget install --id BurntSushi.ripgrep.MSVC --exact --source winget --accept-source-agreements --accept-package-agreements --disable-interactivity`
- 安装前必须征求用户同意。rg 不可用时降级为 `Get-ChildItem | Select-String`，并在回复中说明降级原因；rg 已安装但不在 PATH 时见 §9。

## 3. Unix → PowerShell 速查

| Unix 写法              | PowerShell 写法                                     | 备注                                             |
| ---------------------- | --------------------------------------------------- | ------------------------------------------------ |
| `a && b`               | `a; if ($?) { b }`                                  | pwsh 7 可直接用 `&&`，5.1 不行                   |
| `a \|\| b`             | `a; if (-not $?) { b }`                             | 同上                                             |
| `VAR=x cmd`            | `$env:VAR = 'x'; cmd`                               | 仅对子进程生效                                   |
| `export VAR=x`         | `$env:VAR = 'x'`                                    |                                                  |
| `echo $VAR`            | `$env:VAR`                                          |                                                  |
| `2>/dev/null`          | `2>$null`                                           |                                                  |
| `cmd > f 2>&1`         | `cmd *> f`                                          | 合并所有流                                       |
| `cat f`                | `Get-Content f`                                     | `cat -n` 会报错                                  |
| `head -n 10`           | `Get-Content f -TotalCount 10`                      | 或 `Select-Object -First 10`                     |
| `tail -n 10`           | `Get-Content f -Tail 10`                            |                                                  |
| `grep -r p .`          | `rg p .`                                            |                                                  |
| `find . -name '*.ps1'` | `rg --files -g '*.ps1'`                             | 不要用 `find`（会调 `find.exe`）                 |
| `which x`              | `Get-Command x`                                     | 不要用 `where`（是 `Where-Object` 别名）         |
| `touch f`              | `New-Item -ItemType File -Path f -Force`            | `New-Item` 无 `-LiteralPath`，见 §8              |
| `rm -rf d`             | `Remove-Item -Recurse -Force d`                     | `-rf` 会被当成参数名报错                         |
| `cp -r a b`            | `Copy-Item -Recurse a b`                            | 同上                                             |
| `ls -la`               | `Get-ChildItem -Force`                              | `-la` 会报错                                     |
| `mkdir -p a/b`         | `New-Item -ItemType Directory -Path a/b -Force`     | `New-Item` 无 `-LiteralPath`，见 §8              |
| `ps aux`               | `Get-Process`                                       |                                                  |
| `kill -9 $pid`         | `Stop-Process -Id $pid -Force`                      |                                                  |
| `env`                  | `Get-ChildItem Env:`                                |                                                  |
| `diff a b`             | `Compare-Object (Get-Content a) (Get-Content b)`    |                                                  |
| `sed -i 's/a/b/g' f`   | `(Get-Content f) -replace 'a','b' \| Set-Content f` |                                                  |
| `xargs`                | `ForEach-Object`                                    |                                                  |
| `curl url`             | `curl.exe url` 或 `Invoke-RestMethod url`           | pwsh 7 无 `curl` 别名                            |
| `wget url`             | `Invoke-WebRequest -Uri url -OutFile f`             | pwsh 7 无 `wget` 别名                            |
| `sha256sum f`          | `Get-FileHash f`                                    |                                                  |
| `date`                 | `Get-Date`                                          |                                                  |
| `sleep 5`              | `Start-Sleep -Seconds 5`                            |                                                  |
| `\` 续行               | 反引号 `` ` `` 在行尾                               | 反引号后不能有空格；复杂命令改用括号或 splatting |
| `$?` 取退出码          | `$LASTEXITCODE`                                     | `$?` 是布尔值                                    |
| `read -p`              | `Read-Host`                                         | agent 不要用，会挂起                             |

## 4. 引号与 here-string

- **默认单引号**：字面量，不展开。含 `$`、反引号的正则/模式一律单引号。

- **需要展开时用双引号**：`"$name"`。相邻有歧义写 `"${name}.txt"`。

- **双引号内转义用反引号**：`` `n `` `` `t `` `` `" `` `` `$ ``。不认 `\n`、`\"`。

- **多行文本用 here-string**，不用 Bash heredoc：

  ```powershell
  @'
  字面量，不展开
  '@
  ```

  `@'` / `@"` 必须在行尾，`'@` / `"@` 必须在行首（前面不能有缩进），否则报错。

- 传给原生命令且含空格/`;`/`&`/`$` 的参数必须加引号。复杂场景用停止解析符：`rg --% --pattern "a b"`。

## 5. 管道、语句块、输出

- 把 `foreach`/`if` 块输出接管道时用 `& { ... } | ...`，不要在 `}` 后直接接 `|`。

- `$()` 用于表达式上下文，`(...)` 用于取单个值。

- **`Format-Table` / `Format-List` 之后不能再接 `Where-Object` / `Select-String`**——输出已变成格式化字符串。格式化只放在最后。

- 需要捕获输出用 `Write-Output`，不要用 `Write-Host`。

- 空输入陷阱：`Get-ChildItem | Measure-Object` 在目录为空时下游整段不执行。要统计条目用 `@(Get-ChildItem -LiteralPath $p).Count`，不要接 `Measure-Object` + `ForEach-Object`。

- 大输出先截断，避免撑爆上下文：

  ```powershell
  ... | Select-Object -First 50
  ... | Out-String -Width 200
  ```

  或重定向到临时文件再用 `rg` 查。

## 6. 退出码与错误处理

- `$LASTEXITCODE`：最后一条**原生命令**的退出码。cmdlet 不设置它。
- `$?`：最后一条命令是否成功（布尔）。管道只反映最后一段。
- `$ErrorActionPreference = 'Stop'` 只影响 cmdlet，**不影响原生命令**。
- pwsh 7.3+ 可设 `$PSNativeCommandUseErrorActionPreference = $true` 让原生命令非零也触发 ErrorAction。

agent 执行外部命令的推荐模板：

```powershell
$ErrorActionPreference = 'Continue'
& rg --files -g '*.ps1' .
"LASTEXITCODE=$LASTEXITCODE"
```

（若 `rg` 不在 PATH，把 `& rg` 换成 `& 'C:\full\path\rg.exe'`，见 §9。）

检测命令是否存在，不要靠“跑一下试试”：

```powershell
if (-not (Get-Command rg -ErrorAction SilentlyContinue)) { ... }
```

## 7. 编码（中文环境必看）

- pwsh 7 默认 UTF-8（无 BOM）；**5.1 的 `>` / `Out-File` 默认 UTF-16LE**。

- 写文件显式指定：`... | Out-File -Encoding utf8NoBOM`（7）或 `-Encoding utf8`（5.1，带 BOM）。**`utf8NoBOM` 是 7 专有值，5.1 不认，见 §12。**

- 读文件显式指定：`Get-Content -Raw -Encoding utf8`。5.1 读无 BOM 的 UTF-8 会按 ANSI 解码，中文乱码。

- 原生命令输出中文乱码时设置：

  ```powershell
  [Console]::OutputEncoding = [Text.UTF8Encoding]::new($false)
  $OutputEncoding             = [Text.UTF8Encoding]::new($false)
  ```

- 解析输出时关闭 ANSI 颜色（pwsh 7.2+）：`$PSStyle.OutputRendering = 'PlainText'`，或设 `$env:NO_COLOR = '1'`。

## 8. 外部程序与路径

- 调用带空格路径的 exe 用调用运算符：`& "C:\Program Files\App\app.exe" --flag`。
- **不要用 `Start-Process` 捕获输出**——它默认不把 stdout 接回管道。需要输出就用 `&`；确实要新窗口/提权才用 `Start-Process -Wait -NoNewWindow`，并用 `-RedirectStandardOutput` 落盘。
- 拼接路径用 `Join-Path`，取绝对路径用 `Resolve-Path` / `Convert-Path`。
- 长路径用 `\\?\` 前缀。

### 路径与 `-LiteralPath`

路径含 `[` `]` 时，读取类 cmdlet 会把方括号当通配符，导致静默找不到文件。规则如下：

- **支持 `-LiteralPath`** 的常用 cmdlet：`Get-Item`、`Get-ChildItem`、`Get-Content`、`Set-Content`、`Add-Content`、`Test-Path`、`Remove-Item`、`Copy-Item`、`Move-Item`、`Rename-Item`、`Get-FileHash`、`Select-String`、`Resolve-Path`。这些一律优先用 `-LiteralPath`。

- **例外：`New-Item` 没有 `-LiteralPath`**（只有 `-Path`）。路径含 `[]` 时用：

  ```powershell
  New-Item -ItemType Directory -Path ([WildcardPattern]::Escape($p)) -Force
  # 或完全绕过通配符（推荐）：
  [System.IO.Directory]::CreateDirectory($p) | Out-Null
  New-Item -ItemType File -Path ([WildcardPattern]::Escape($f)) -Force
  # 文件等价写法：
  [System.IO.File]::WriteAllText($f, '')
  ```

- `-Path` 支持通配符，`-LiteralPath` 不支持。不确定且 cmdlet 支持时优先 `-LiteralPath`。

## 9. 检索（rg）

- 先探测：`Get-Command rg -ErrorAction SilentlyContinue`。**不在 PATH 但已知安装位置时，用调用运算符传全路径**：`& 'C:\Users\me\scoop\shims\rg.exe' --files`，不要裸写 `rg`。常见非 PATH 位置：scoop shims、cargo `%USERPROFILE%\.cargo\bin`、手动解压目录。
- 字面量搜索加 `-F`，避免 Windows 路径反斜杠被当正则转义。
- 常用参数：`-i` 忽略大小写、`-l` 只列文件、`-n` 行号、`-C 3` 上下文、`-t py` 按类型、`-g '*.ts'` 按 glob、`--max-columns 300` 防超长行。
- 默认遵守 `.gitignore` 且跳过隐藏文件。需要时加 `-uu`（`--no-ignore --hidden`），二进制加 `-a`，多行匹配加 `-U`。
- **不要**把含 `*` 的路径直接当 PATH 参数：`rg 'src/*.ps1'` 是错的，用 `rg -g 'src/*.ps1' pattern`。
- 按文件名找：`rg --files | rg -F name`。
- 大仓库加 `--max-filesize 5M`，避免卡在大文件上。
- 退出码：`0` 有匹配、`1` 无匹配（**正常**，不是错误）、`2` 执行错误（路径不存在、正则非法等）。

## 10. 超时与交互防护

以下设置能避免命令永久挂起：

```powershell
$ProgressPreference = 'SilentlyContinue'      # Invoke-WebRequest 提速且不刷进度
$env:GIT_PAGER = 'cat'
$env:GIT_TERMINAL_PROMPT = '0'                # git 不弹凭据提示
```

- 不用 `Read-Host`、`more`、`Out-Host -Paging`。
- `winget` 加 `--accept-source-agreements --accept-package-agreements --disable-interactivity`。
- 删除/覆盖类 cmdlet 加 `-Confirm:$false`。
- agent 侧给每条命令设超时；PowerShell 内兜底可用 `Start-Job` + `Wait-Job -Timeout 30`。

## 11. 从非 PowerShell 宿主调用

当宿主是 cmd / Git Bash / WSL 时，把复杂命令写成临时 `.ps1`，用 `-File` 执行，避免多层引号地狱：

```text
pwsh -NoLogo -NoProfile -NonInteractive -File C:\tmp\task.ps1
```

必须用 `-Command` 时：

- **cmd**：`pwsh -NoLogo -NoProfile -Command "Get-ChildItem 'C:\Program Files'"`。cmd 不认单引号，内层路径用 PowerShell 单引号包住。
- **Git Bash**：设 `MSYS_NO_PATHCONV=1` 防止 `/c/...` 被自动转换；路径传 `C:/...` 形式给 pwsh。
- **WSL**：用 `pwsh.exe`，路径用 `wslpath -w` 转成 Windows 路径再传。

## 12. 5.1 兜底

pwsh 确实不可用且用户不同意安装时，显式告知用户并遵守 5.1 限制：

- 不支持 `&&`、`||`、`??`、三元 `?:`、`-Parallel`、`$PSStyle`、`$PSNativeCommandArgumentPassing`。
- **`-Encoding utf8NoBOM` 是 pwsh 7 专有值，5.1 不认**。5.1 写文件用 `-Encoding utf8`（带 BOM）；`Set-Content` / `Out-File` 不带 `-Encoding` 时默认 UTF-16LE。
- `>` / `Out-File` 默认 UTF-16LE，必须显式 `-Encoding utf8`。
- 读无 BOM 的 UTF-8 文件时 `Get-Content -Raw -Encoding utf8` 可用；不带 `-Encoding` 会按 ANSI 解码导致中文乱码。
- `curl` / `wget` 是 `Invoke-WebRequest` 别名，要用真 curl 写 `curl.exe`。
- 判断平台用 `$PSVersionTable.PSEdition`（5.1 = `Desktop`，7 = `Core`），不要用 `$IsWindows`（5.1 没有）。
- 安装 pwsh 后 `powershell.exe` 仍指向 5.1，**必须显式调用 `pwsh.exe`** 才能进 7。

## 13. 失败排查顺序

1. 读错误原文，不要猜。
2. `$LASTEXITCODE` / `$?`。
3. 引号是否被 PowerShell 吃掉——`Write-Output` 打印实际参数确认。
4. `Test-Path -LiteralPath` 确认路径存在（`New-Item` 场景改用 `-Path` + `[WildcardPattern]::Escape()`，见 §8）。
5. `Get-Command` 确认命令存在；原生命令不在 PATH 时用调用运算符传全路径（§9）。
6. 版本与编码（§7、§12）。
7. 仍失败再考虑换调用方式，不要直接切到 cmd/bash。
