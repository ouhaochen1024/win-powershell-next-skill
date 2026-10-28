---
name: win-powershell-next
version: 2.0.0
display_name: Windows 命令执行（powershell.exe / 5.1）
display_name_en: Win PowerShell (5.1)
description: Windows command-execution guide for coding agents. Target Windows PowerShell 5.1 (powershell.exe) by default and write 5.1-compatible syntax only — PowerShell 7 must never be a prerequisite. No && / || chaining, no ternary, no ??, no utf8NoBOM, no $PSStyle, no $IsWindows. Covers Unix-to-PowerShell mapping, quoting and here-strings, encoding and BOM traps, exit codes, timeout guards, and rg-based search. Trigger when generating or executing any shell command on Windows, or when the host shell is PowerShell / cmd / Git Bash / WSL. Also trigger when the user asks to find files, search text, delete or recreate a directory, read/write files, or check command exit codes on Windows.
description_zh: Windows 命令执行规范 —— 以 powershell.exe（5.1）为准，只写 5.1 可运行语法，Unix→PowerShell 对照，引号与 here-string，编码与 BOM，退出码与错误处理，超时防护，检索优先 rg
description_en: Windows PowerShell 5.1 (powershell.exe) command execution and rg-based search guide for coding agents
---

# Win PowerShell Next

在 Windows 上生成或执行任何 shell 命令时遵循本 skill。执行环境默认是 **Windows PowerShell 5.1（`powershell.exe`，系统自带）**，不要求、不假设存在 PowerShell 7。核心目标：**一次写对，不靠试错**。

## 1. 版本策略（最重要）

1. 目标解释器是 `powershell.exe`（Windows PowerShell 5.1，`$PSVersionTable.PSEdition` 为 `Desktop`）。它随系统预装，开箱可用，所以**默认就是它**。
2. **不强制 PowerShell 7**：不提示安装 `pwsh`，不因为「7 更新更快」就换解释器，不把 `pwsh` 写进命令模板。
3. **只写一套语法**：即使宿主恰好是 pwsh 7，也按 5.1 语法写。5.1 是 7 的子集，5.1 语法在 7 上照跑；反过来不成立。混写两套方言是错误的主要来源。
4. 自查：一旦准备写 `&&`、`||`、`?:`、`??`、`??=`、`utf8NoBOM`、`$PSStyle`、`$IsWindows`、`-Parallel`、`-AsHashtable`，立刻换成 §3 的 5.1 写法。
5. 个别场景确实只能靠 7（例如需要 `ForEach-Object -Parallel` 真并行、`ConvertFrom-Json -AsHashtable` 处理深层结构）：先说明理由并征得用户同意，**只对那一段**用 `pwsh.exe` 单独执行，不要整体切换解释器，也不要因此要求全局安装 7。

## 2. 硬规则

1. 宿主 shell 是 PowerShell 就直接用；需要独立进程时用 `powershell.exe -NoLogo -NoProfile -NonInteractive`。`-NoProfile` 防止用户 profile 改别名 / 改 `$ErrorActionPreference` / 打印横幅；`-NonInteractive` 防止卡在 `Read-Host`。
2. 5.1 默认 ExecutionPolicy 常拦住本地 `.ps1`（`RemoteSigned` / `Restricted`），执行脚本加 `-ExecutionPolicy Bypass`。
3. 用 PowerShell 原生语法，不照搬 Unix 命令。对照表见 §4。
4. 文本检索用 `rg`，文件枚举用 `rg --files`。
5. 命令失败先看错误原文、`$LASTEXITCODE`、引号与路径，不要为了「绕过去」切换 shell。
6. 写文件一律显式 `-Encoding`（§7），否则 5.1 默认写出 UTF-16LE。

## 3. 5.1 与 7 的语法差异（禁用清单）

| pwsh 7 写法                                    | 5.1 等价写法                                                                 |
| ---------------------------------------------- | ---------------------------------------------------------------------------- |
| `a && b`                                       | `a; if ($?) { b }`                                                           |
| `a \|\| b`                                     | `a; if (-not $?) { b }`                                                      |
| `$r = $c ? 'y' : 'n'`                          | `if ($c) { $r = 'y' } else { $r = 'n' }`                                     |
| `$x ?? 'default'` / `$x ??= 'default'`         | `if ($null -eq $x) { $x = 'default' }`                                       |
| `-Encoding utf8NoBOM`                          | `[System.IO.File]::WriteAllText($p, $s, (New-Object System.Text.UTF8Encoding($false)))`，或 `-Encoding utf8`（带 BOM） |
| `$PSStyle.OutputRendering = 'PlainText'`       | `$env:NO_COLOR = '1'`                                                        |
| `$IsWindows` / `$IsLinux`                      | `$PSVersionTable.PSEdition -eq 'Desktop'`                                    |
| `$PSNativeCommandUseErrorActionPreference`     | 手动判断 `$LASTEXITCODE`（§9）                                               |
| `ForEach-Object -Parallel { }`                 | `ForEach-Object { }` 串行，或 `Start-Job` + `Wait-Job`                       |
| `Get-Content -AsByteStream`                    | `Get-Content -Encoding Byte`                                                 |
| `ConvertFrom-Json -AsHashtable`                | `ConvertFrom-Json` 后遍历 `$o.PSObject.Properties`                           |
| `Join-Path -Path a -ChildPath b -AdditionalChildPath c` | `Join-Path (Join-Path $a $b) $c`                                    |
| `Test-Connection -TargetName x`                | `Test-Connection -ComputerName x`                                            |
| `Select-String -Raw`                           | `(Select-String -LiteralPath $p -Pattern $re).Line`                          |
| `ConvertTo-Json -AsArray`                      | `ConvertTo-Json -Depth 10`，单元素要强制数组就 `@($items) \| ConvertTo-Json -Depth 10` |
| `Test-Json` 这个 cmdlet                        | 5.1 没有，用 `ConvertFrom-Json` 包 `try/catch` 判断                          |
| `Invoke-RestMethod -SkipCertificateCheck`      | 不要用；确需自签证书时改 `[Net.ServicePointManager]::ServerCertificateValidationCallback` |

补充 5.1 事实，写代码前先确认：

- `::new()` 语法 5.0 起可用，但团队里混用版本时更稳的是 `New-Object`。
- `$PSVersionTable.PSVersion`、`Where-Object` 简化语法、`[pscustomobject]@{}`、`class`、`Expand-Archive` 都可用。
- `Get-Content -Tail`、`-TotalCount`、`-Raw` 都可用。
- **不要过度禁用**：5.1 也支持 `Set-Location -LiteralPath`、`Select-Object -SkipLast`、`Get-ChildItem -File/-Directory/-Depth`、`-in`/`-notin`、`[WildcardPattern]::Escape()`。别为了「怕不兼容」绕远路写出更脆的代码。
- 5.1 的 `-Command` 参数把整串交给内置解析器，引号/`$`/`;` 很容易被吞，复杂逻辑一律落 `.ps1` 再 `-File`。

## 4. Unix → PowerShell 速查（5.1 版）

| Unix 写法              | PowerShell 写法                                         | 备注                                                     |
| ---------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| `a && b`               | `a; if ($?) { b }`                                      | **5.1 不支持 `&&`**，写 `&&` 直接语法错误                |
| `a \|\| b`             | `a; if (-not $?) { b }`                                 | 同上                                                     |
| `a \| b`               | `a \| b`                                                | 管道通用                                                 |
| `VAR=x cmd`            | `$env:VAR = 'x'; cmd`                                   | 仅对子进程生效                                           |
| `export VAR=x`         | `$env:VAR = 'x'`                                        |                                                          |
| `echo $VAR`            | `$env:VAR`                                              |                                                          |
| `2>/dev/null`          | `2>$null`                                               |                                                          |
| `cmd > f 2>&1`         | `cmd *> f`                                              | 合并所有流；`>` 默认 UTF-16LE，见 §7                      |
| `cat f`                | `Get-Content f`                                         | `cat -n` 会报错                                          |
| `head -n 10`           | `Get-Content f -TotalCount 10`                          | 或 `Select-Object -First 10`                             |
| `tail -n 10`           | `Get-Content f -Tail 10`                                |                                                          |
| `grep -r p .`          | `rg p .`                                                |                                                          |
| `find . -name '*.ps1'` | `rg --files -g '*.ps1'`                                 | 不要用 `find`（会调 `find.exe`）                         |
| `which x`              | `Get-Command x`                                         | 不要用 `where`（是 `Where-Object` 别名）                 |
| `touch f`              | `New-Item -ItemType File -Path f -Force`                | `New-Item` 无 `-LiteralPath`，见 §8                      |
| `rm -rf d`             | `Remove-Item -Recurse -Force d`                         | `-rf` 会被当成参数名报错                                 |
| `cp -r a b`            | `Copy-Item -Recurse a b`                                | 同上                                                     |
| `ls -la`               | `Get-ChildItem -Force`                                  | `-la` 会报错                                             |
| `mkdir -p a/b`         | `New-Item -ItemType Directory -Path a/b -Force`         | `New-Item` 无 `-LiteralPath`，见 §8                      |
| `ps aux`               | `Get-Process`                                           |                                                          |
| `kill -9 $pid`         | `Stop-Process -Id $pid -Force`                          |                                                          |
| `env`                  | `Get-ChildItem Env:`                                    |                                                          |
| `diff a b`             | `Compare-Object (Get-Content a) (Get-Content b)`        |                                                          |
| `sed -i 's/a/b/g' f`   | `(Get-Content f) -replace 'a','b' \| Set-Content f -Encoding utf8` | 忘了 `-Encoding` 会把文件写成 UTF-16LE          |
| `xargs`                | `ForEach-Object`                                        |                                                          |
| `curl url`             | `curl.exe url` 或 `Invoke-RestMethod url`               | **5.1 里 `curl` 是 `Invoke-WebRequest` 别名**，必须写 `curl.exe` |
| `wget url`             | `Invoke-WebRequest -Uri url -OutFile f`                 | **5.1 里 `wget` 是别名**，不是真 wget                    |
| `sha256sum f`          | `Get-FileHash f`                                        |                                                          |
| `date`                 | `Get-Date`                                              |                                                          |
| `sleep 5`              | `Start-Sleep -Seconds 5`                                |                                                          |
| `\` 续行               | 反引号 `` ` `` 在行尾                                   | 反引号后不能有空格；复杂命令改用括号或 splatting         |
| `$?` 取退出码          | `$LASTEXITCODE`                                         | `$?` 是布尔值                                            |
| `read -p`              | `Read-Host`                                             | agent 不要用，会挂起                                     |

## 5. 引号与 here-string

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

- 传给原生命令且含空格 / `;` / `&` / `$` 的参数必须加引号。复杂场景用停止解析符：`rg --% --pattern "a b"`。
- 从 `cmd` 调 `powershell.exe -Command` 时，外层只能双引号，内层一律用 PowerShell 单引号包路径：`powershell.exe -NoProfile -Command "Get-ChildItem 'C:\Program Files'"`。

## 6. 管道、语句块、输出

- 把 `foreach` / `if` 块输出接管道时用 `& { ... } | ...`，不要在 `}` 后直接接 `|`。
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

## 7. 编码与 BOM（5.1 高发坑）

5.1 的默认编码和 7 完全不同，中文环境几乎每次踩：

- `>`、`>>`、`Out-File`、`Set-Content` **默认 UTF-16LE**。写文本必须显式 `-Encoding utf8`（5.1 的 `utf8` = **带 BOM**）。
- `-Encoding` 在 5.1 的合法值（`Out-File` / `Set-Content`）：`unknown`、`string`、`unicode`、`bigendianunicode`、`utf8`、`utf7`、`utf32`、`ascii`、`default`（ANSI）、`oem`。**没有 `utf8NoBOM`，传了直接 ValidateSet 报错**。读写二进制用 `Get-Content -Encoding Byte`（写用 `[System.IO.File]::WriteAllBytes()`，`Set-Content -Encoding Byte` 只接受 byte 类型的值）。
- 要写无 BOM 的 UTF-8：

  ```powershell
  $utf8NoBom = New-Object System.Text.UTF8Encoding($false)
  [System.IO.File]::WriteAllText('C:\tmp\a.json', $json, $utf8NoBom)
  ```

- **`.ps1` 里含中文就必须存成 UTF-8 with BOM**，否则 5.1 按 ANSI 解码，中文变乱码甚至语法报错。用工具写脚本文件时确认这一点；不确定就把脚本文本写成纯 ASCII。
- 读文件显式指定编码：`Get-Content -LiteralPath $p -Raw -Encoding utf8`。不带 `-Encoding` 读无 BOM 的 UTF-8 会按 ANSI 解码。
- 原生命令输出中文乱码时：

  ```powershell
  [Console]::OutputEncoding = [System.Text.Encoding]::GetEncoding(65001)
  $OutputEncoding           = [System.Text.Encoding]::UTF8
  ```

  或先 `chcp 65001`（只在当前会话有效）。
- 解析输出时关掉 ANSI 颜色：`$env:NO_COLOR = '1'`（5.1 没有 `$PSStyle`）。
- git 状态里中文文件名显示成 `\344\270\255`：`git config core.quotepath false`。

## 8. 外部程序与路径

- 调用带空格路径的 exe 用调用运算符：`& "C:\Program Files\App\app.exe" --flag`。
- **不要用 `Start-Process` 捕获输出**——它默认不把 stdout 接回管道。需要输出就用 `&`；确实要新窗口/提权才用 `Start-Process -Wait -NoNewWindow`，并用 `-RedirectStandardOutput` 落盘。
- 拼接路径用 `Join-Path`（5.1 没有 `-AdditionalChildPath`，多段就嵌套调用），取绝对路径用 `Resolve-Path` / `Convert-Path`。
- 长路径用 `\\?\` 前缀。

### 路径与 `-LiteralPath`

路径含 `[` `]` 时，读取类 cmdlet 会把方括号当通配符，导致静默找不到文件。规则如下：

- **支持 `-LiteralPath`** 的常用 cmdlet：`Get-Item`、`Get-ChildItem`、`Get-Content`、`Set-Content`、`Add-Content`、`Test-Path`、`Remove-Item`、`Copy-Item`、`Move-Item`、`Rename-Item`、`Get-FileHash`、`Select-String`、`Resolve-Path`。这些一律优先用 `-LiteralPath`。
- **例外 1：`New-Item` 没有 `-LiteralPath`**（只有 `-Path`）。路径含 `[]` 时用：

  ```powershell
  New-Item -ItemType Directory -Path ([WildcardPattern]::Escape($p)) -Force
  # 或完全绕过通配符（推荐）：
  [System.IO.Directory]::CreateDirectory($p) | Out-Null
  New-Item -ItemType File -Path ([WildcardPattern]::Escape($f)) -Force
  # 文件等价写法：
  [System.IO.File]::WriteAllText($f, '')
  ```

- `-Path` 支持通配符，`-LiteralPath` 不支持。不确定且 cmdlet 支持时优先 `-LiteralPath`（`Set-Location -LiteralPath` 在 5.1 也可用）。

## 9. 退出码与错误处理

- `$LASTEXITCODE`：最后一条**原生命令**的退出码。cmdlet 不设置它。
- `$?`：最后一条命令是否成功（布尔）。管道只反映最后一段。
- `$ErrorActionPreference = 'Stop'` 只影响 cmdlet，**不影响原生命令**。
- 5.1 **没有** `$PSNativeCommandUseErrorActionPreference`：原生命令返回非零不会抛异常、也不会中断脚本。必须自己判断 `$LASTEXITCODE`，否则失败会被当成成功。

agent 执行外部命令的推荐模板：

```powershell
$ErrorActionPreference = 'Continue'
& rg --files -g '*.ps1' .
if ($LASTEXITCODE -gt 1) {
    Write-Output "rg failed: $LASTEXITCODE"
    exit $LASTEXITCODE
}
```

（若 `rg` 不在 PATH，把 `& rg` 换成 `& 'C:\full\path\rg.exe'`，见 §10。）

检测命令是否存在，不要靠「跑一下试试」：

```powershell
if (-not (Get-Command rg -ErrorAction SilentlyContinue)) { ... }
```

## 10. 检索（rg）

- 先探测：`Get-Command rg -ErrorAction SilentlyContinue`。**不在 PATH 但已知安装位置时，用调用运算符传全路径**：`& 'C:\Users\me\scoop\shims\rg.exe' --files`，不要裸写 `rg`。常见非 PATH 位置：scoop shims、cargo `%USERPROFILE%\.cargo\bin`、手动解压目录。
- 字面量搜索加 `-F`，避免 Windows 路径反斜杠被当正则转义。
- 常用参数：`-i` 忽略大小写、`-l` 只列文件、`-n` 行号、`-C 3` 上下文、`-t py` 按类型、`-g '*.ts'` 按 glob、`--max-columns 300` 防超长行。
- 默认遵守 `.gitignore` 且跳过隐藏文件。需要时加 `-uu`（`--no-ignore --hidden`），二进制加 `-a`，多行匹配加 `-U`。
- **不要**把含 `*` 的路径直接当 PATH 参数：`rg 'src/*.ps1'` 是错的，用 `rg -g 'src/*.ps1' pattern`。
- 按文件名找：`rg --files | rg -F name`。
- 大仓库加 `--max-filesize 5M`，避免卡在大文件上。
- 退出码：`0` 有匹配、`1` 无匹配（**正常**，不是错误）、`2` 执行错误（路径不存在、正则非法等）。
- rg 不可用时降级：`Get-ChildItem -Recurse -File | Select-String -Pattern $re`，并在回复里说明降级原因。

## 11. 超时与交互防护

以下设置能避免命令永久挂起：

```powershell
$ProgressPreference = 'SilentlyContinue'      # Invoke-WebRequest 提速且不刷进度
$env:GIT_PAGER = 'cat'
$env:GIT_TERMINAL_PROMPT = '0'                # git 不弹凭据提示
```

- 不用 `Read-Host`、`more`、`Out-Host -Paging`。
- `winget` 加 `--accept-source-agreements --accept-package-agreements --disable-interactivity`。
- 删除/覆盖类 cmdlet 加 `-Confirm:$false`。
- agent 侧给每条命令设超时；PowerShell 内兜底用 `Start-Job` + `Wait-Job -Timeout 30`（`Start-ThreadJob` 5.1 默认没有，别用）。

## 12. 从非 PowerShell 宿主调用

当宿主是 cmd / Git Bash / WSL 时，把复杂命令写成临时 `.ps1`，用 `-File` 执行，避免多层引号地狱：

```text
powershell.exe -NoLogo -NoProfile -NonInteractive -ExecutionPolicy Bypass -File C:\tmp\task.ps1
```

必须用 `-Command` 时：

- **cmd**：`powershell.exe -NoLogo -NoProfile -Command "Get-ChildItem 'C:\Program Files'"`。cmd 不认单引号，内层路径用 PowerShell 单引号包住。
- **Git Bash**：设 `MSYS_NO_PATHCONV=1` 防止 `/c/...` 被自动转换；路径传 `C:/...` 形式给 `powershell.exe`。
- **WSL**：调 `powershell.exe`，路径用 `wslpath -w` 转成 Windows 路径再传。
- 5.1 的 `-Command` 解析比 7 更脆弱（`$`、`;`、`"` 会被吞）。串长超过一行或含这些字符，直接写 `.ps1`，别硬拼。

## 13. 失败排查顺序

1. 读错误原文，不要猜。
2. `$LASTEXITCODE` / `$?`。
3. 引号是否被 PowerShell 吃掉——`Write-Output` 打印实际参数确认。
4. `Test-Path -LiteralPath` 确认路径存在（`New-Item` 场景改用 `-Path` + `[WildcardPattern]::Escape()`，见 §8）。
5. `Get-Command` 确认命令存在；原生命令不在 PATH 时用调用运算符传全路径（§10）。
6. 编码与版本（§7）：是不是把 5.1 当 7 写了（`&&`、`utf8NoBOM`、`$PSStyle`），或者 `.ps1` 少了 BOM。
7. 仍失败再考虑换调用方式，不要直接切到 cmd/bash。
