# powershell7-terminal

An agent skill that forces every terminal/shell command on Windows to run through **PowerShell 7 (`pwsh`)** instead of the tool's default PowerShell 5.1 wrapper — using `pwsh -EncodedCommand` so quoting, CJK text, and exit codes survive intact.

## Why

On Windows, most coding agents execute commands through a Windows PowerShell **5.1** wrapper shell, which causes real breakage:

- Native argument passing **strips embedded double quotes** → commands silently mutate (`Write-Output "hello world"` arrives as two words)
- Chinese / CJK text gets garbled by encoding mismatches
- Exit codes from `pwsh` are swallowed, so failures look like successes
- PS7-only syntax (`??`, `?:`, `ForEach-Object -Parallel`) can't be used directly

## What the skill does

- Wraps every command as a child `pwsh -NoProfile -EncodedCommand` process (Base64 UTF-16LE — safe for quotes, `<>&`, JSON, newlines, 中文)
- Provides a temp-file + `-File` pattern for long/multi-line scripts
- Propagates `$LASTEXITCODE` so the agent sees real failures
- Documents verified PS 5.1 vs PS 7 differences (`$IsWindows`, ternary, `?.`, `curl` alias, …) and a troubleshooting table

## Install

Clone and copy the `powershell7-terminal` folder into your agent's skills directory:

```bash
git clone https://github.com/wanming1122/powershell7-terminal.git
```

Then place it in a location your agent scans for skills, e.g. `~/.zcode/skills/`, `.claude/skills/`, or your tool's equivalent. The skill is defined by [SKILL.md](SKILL.md) with `locales/` for English and Chinese.

## Trigger

Activates whenever the agent is about to execute shell commands, scripts, git, npm, python, or any CLI tool on Windows — or on trigger phrases like 「用 PowerShell 7」「用 pwsh」「执行命令」「跑一下脚本」.
