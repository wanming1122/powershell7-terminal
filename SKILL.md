---
name: powershell7-terminal
description: Run every terminal/shell command on this Windows machine through PowerShell 7 (pwsh) instead of the default PowerShell 5.1 shell. Wraps commands with pwsh -EncodedCommand so quoting, CJK text, and exit codes survive. Use whenever the agent is about to execute shell commands, scripts, git, npm, python, or any CLI tool on Windows. Trigger phrases: 用 PowerShell 7, 用 pwsh, 执行命令, 跑一下脚本, 终端命令, PowerShell 7. Do not use when only reading or editing files with dedicated file tools.
---

# PowerShell 7 Terminal

## Important

**Every terminal call must execute inside PowerShell 7 (`pwsh`), never directly in the tool's default shell.**

The terminal tool's own shell is Windows PowerShell **5.1**. That is the wrapper only — the actual work must run as a child `pwsh` process. If your agent's terminal tool also wraps commands in Windows PowerShell 5.1 (as most Windows coding agents do), everything below applies the same way.

Verified environment:

| Item | Value |
|------|-------|
| Tool shell (wrapper) | Windows PowerShell 5.1.26100 Desktop |
| Target shell | PowerShell 7.6.x Core |
| `pwsh` on PATH | yes |
| Full path fallback | `%LOCALAPPDATA%\Microsoft\WindowsApps\pwsh.exe` (i.e. `C:\<you>\AppData\Local\Microsoft\WindowsApps\pwsh.exe`) |

## Instructions

### Step 1: Pick an invocation pattern

Do **not** call `pwsh -Command '..."inner"...'`. PowerShell 5.1's native argument passing **strips embedded double quotes**, so the command silently mutates (verified: `Write-Output "hello world"` becomes two separate words).

Use one of these three patterns instead.

**Pattern A — default for almost everything. `-EncodedCommand` (safe for any content):**

```powershell
$cmd = 'YOUR COMMAND HERE'
$b64 = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
& pwsh -NoProfile -EncodedCommand $b64
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
```

Handles Chinese, quotes, `<>&`, JSON, and newlines. Exit codes propagate.

**Pattern B — long or multi-line scripts. Temp file + `-File`:**

```powershell
$tmp = Join-Path $env:TEMP "mimo_pwsh_$(Get-Random).ps1"
@'
# your script body
'@ | Set-Content -Path $tmp -Encoding UTF8
& pwsh -NoProfile -File $tmp
$code = $LASTEXITCODE
Remove-Item $tmp -Force
if ($code -ne 0) { exit $code }
```

**Pattern C — trivial one-liners only. No embedded double quotes:**

```powershell
& pwsh -NoProfile -Command 'Get-Location'
```

If the body needs a string literal, use PowerShell single quotes inside. Never nest double quotes inside a `-Command` string.

### Step 2: Propagate the exit code

Always end with `if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }` (Pattern A) or the Pattern B equivalent. Without it the tool sees success even when `pwsh` failed.

### Step 3: Write PowerShell 7 syntax in the body

Script bodies run under `pwsh`, so PS7-only syntax is allowed — and PS5.1-only habits will break. Differences verified on this machine:

| Feature | PS 5.1 (wrapper) | PS 7.x (pwsh) |
|---------|------------------|-----------------|
| Edition | Desktop | Core |
| `$IsWindows` | does not exist | `$true` |
| `??` / `??=` | parse error | works |
| ternary `?:` | parse error | works |
| `if` as expression | works | works |
| `?.` null-conditional | n/a | **not supported** (still throws) |
| `curl` alias | present (→ `Invoke-WebRequest`) | absent (real `curl.exe`) |
| `Get-WmiObject` | present | present |
| `ForEach-Object -Parallel` | no | yes |

Notes:

- Use `??` / `??=` / `?:` freely inside `pwsh`. Do **not** use `?.` — it is not implemented as of 7.6.x.
- `curl` inside `pwsh` is the real `curl.exe`. Inside the 5.1 wrapper it is `Invoke-WebRequest` — another reason to go through `pwsh`.
- `$IsWindows` only exists under `pwsh`. In the wrapper use `[Environment]::OSVersion` or `$env:OS` if you must.

## Examples

**User: "跑一下这个脚本"**

```powershell
$cmd = @'
Get-ChildItem -Path . -File |
  Select-Object -First 5 Name, Length
'@
$b64 = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
& pwsh -NoProfile -EncodedCommand $b64
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
```

**User: "用 PowerShell 7 看看当前目录"**

```powershell
& pwsh -NoProfile -Command 'Get-Location'
```

**User: "查一下 git 状态"**

```powershell
$cmd = 'git status --short'
$b64 = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
& pwsh -NoProfile -EncodedCommand $b64
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
```

## Notes

- Each terminal call starts a **fresh process**. Environment variables set inside `pwsh` do not persist to the next call. Pass state through files or arguments.
- `pwsh` is on PATH. If `pwsh` is not found, fall back to the full path above.
- Never wrap the work in `powershell.exe` (5.1). That defeats the purpose of this skill.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Command splits into multiple output lines, or "not recognized as a name" | double quotes stripped by 5.1 native arg passing | switch to Pattern A (`-EncodedCommand`) |
| Chinese text garbled | wrong encoding | use `[Text.Encoding]::Unicode` (UTF-16LE) in Pattern A |
| `UnexpectedToken: ??` or `?` | body ran in the 5.1 wrapper, not `pwsh` | ensure the body goes through `& pwsh ...` |
| Exit code always 0 | forgot to propagate | add `if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }` |
| `pwsh` not recognized | PATH lost in a non-login shell | use the full path fallback above |
| Cmdlet failed (e.g. `Get-Content` on a missing file) but exit code is still 0 | cmdlet non-terminating errors do not set `$LASTEXITCODE` | end the script body with `if (-not $?) { exit 1 }`, or use try/catch with explicit `exit 1` |
