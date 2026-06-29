---
name: powershell-safety
description: Use when running or composing Windows PowerShell commands, especially through Codex shell_command on Windows. Trigger for paths with brackets or spaces, Nuxt/Vue dynamic route files, Get-Content, Select-String, rg, git pathspecs, environment variables, quoting, native command arguments, deletion, moving, command chaining, exit codes, or any task where PowerShell syntax may differ from Bash or cmd.
---

# PowerShell Safety

## Core Rule

When using Windows PowerShell, assume that paths, wildcards, quoting, native commands, and file operations can behave differently from Bash. Before writing a command:

1. Use single quotes for paths and patterns unless interpolation is required.
2. Use `-LiteralPath` with PowerShell cmdlets when targeting an exact file or directory.
3. Do not pass `-LiteralPath` to native commands such as `rg`, `git`, `pnpm`, `node`, or `python`.
4. Use `apply_patch` for source edits. Do not write source files with shell redirection, here-docs, or `Set-Content`.
5. For delete or move operations, resolve and inspect the absolute path first.
6. Do not use Bash syntax by habit: avoid `export`, `grep`, `cat <<EOF`, `rm -rf`, and `VAR=value command`.

## Paths And Wildcards

PowerShell treats `[]` as wildcard character classes. This breaks common Nuxt and Vue route files such as `[address].vue`.
PowerShell also treats unquoted parentheses as syntax. This breaks route-group directories such as `app\pages\(home)`, even for read-only commands.

Use this:

```powershell
Get-Content -Raw -LiteralPath 'app\pages\user\[address].vue'
Select-String -LiteralPath 'app\pages\user\[address].vue' -Pattern 'BasicVirtualTable'
Test-Path -LiteralPath 'app\pages\user\[address].vue'
```

Do not use this:

```powershell
Get-Content -Raw app\pages\user\[address].vue
Select-String app\pages\user\[address].vue -Pattern 'BasicVirtualTable'
```

Use `-LiteralPath` with these PowerShell cmdlets when the path is exact or may contain special characters:

- `Get-Content`
- `Set-Content`
- `Add-Content`
- `Copy-Item`
- `Move-Item`
- `Remove-Item`
- `Rename-Item`
- `Test-Path`
- `Get-Item`
- `Get-ChildItem`
- `Select-String`
- `Resolve-Path`

Remember: `rg`, `git`, `pnpm`, `node`, `python`, and most CLIs are native commands. They do not understand `-LiteralPath`.

For native commands, quote paths that contain `[]`, `()`, spaces, or other special characters. Put command options before `--`, then pass quoted paths:

```powershell
rg -n -S 'WorldCupMarketCard' -- 'app/pages/(home)' 'app/components/market'
```

Avoid unquoted route-group paths. PowerShell may parse the parentheses before `rg` ever sees the argument:

```powershell
rg -n 'WorldCupMarketCard' app\pages\(home\) -S
```

## ripgrep

Use `rg` before slower alternatives.

Search text in a directory:

```powershell
rg -n 'useAuthEventSource|EventSource' app -S
```

Search a literal string:

```powershell
rg -n -F 'route.params.address' app
```

Search a single file. Put the path after `--`:

```powershell
rg -n 'BasicVirtualTable' -- 'app/pages/user/[address].vue'
```

Put all `rg` options before `--`. After `--`, `rg` treats every following token as a path, so an option like `-S` becomes a filename:

```powershell
rg -n -S 'CHART_API|apiBase' -- '.env.dev' '.env.test' 'nuxt.config.ts'
```

Do not put options after `--`:

```powershell
rg -n 'CHART_API|apiBase' -- '.env.dev' '.env.test' 'nuxt.config.ts' -S
```

This also applies to `--glob` / `-g`. Do not append globs after `--` and path arguments; PowerShell/native execution will pass them as literal path tokens, and `rg` may report errors like `rg: --glob: The system cannot find the file specified` or `*.vue: The filename, directory name, or volume label syntax is incorrect`.

Use this:

```powershell
rg -n -S --glob '*.vue' --glob '*.ts' 'createEventDetailTo\(' -- app\components app\pages app\utils
```

Do not use this:

```powershell
rg -n -S 'createEventDetailTo\(' -- app\components app\pages app\utils --glob '*.vue' --glob '*.ts'
```

Do not pass quoted wildcard paths such as `'.env*'` to native commands. PowerShell will not expand the glob for `rg`, and `rg` will try to open a literal path containing `*` on Windows. Use `rg` globs or enumerate concrete files first:

```powershell
rg -n -S 'CHART_API|apiBase' -g '.env*' -- .
Get-ChildItem -LiteralPath '.' -Filter '.env*' -File | ForEach-Object { $_.FullName }
```

List files:

```powershell
rg --files app
rg --files -g '*.vue' app
rg --files app | rg 'Basic.*Table'
```

Avoid:

```powershell
rg -LiteralPath 'app\pages\user\[address].vue'
rg -n 'CHART_API' -- '.env*'
rg -n 'CHART_API' -- '.env.dev' -S
grep -R 'foo' app
```

## Git Pathspecs

Git does not support PowerShell `-LiteralPath`. For files with `[]`, use Git literal pathspecs.

Use this:

```powershell
git diff -- ':(literal)app/pages/user/[address].vue'
git add -- ':(literal)app/pages/user/[address].vue'
git log --oneline -- ':(literal)app/pages/user/[address].vue'
```

For ordinary paths:

```powershell
git diff -- app/components/BasicVirtualTable.vue
git add -- app/components/BasicVirtualTable.vue
git status --short
```

`git diff --stat`, `git diff --name-only`, and ordinary `git diff` do not show untracked files. Always pair them with `git status --short` when reviewing worktree changes, especially after creating new files:

```powershell
git status --short
git diff --stat
git diff --name-only
```

If `git status --short` shows `??`, inspect those files directly; do not assume the diff commands covered them.

Never use destructive Git commands such as `git reset --hard`, `git checkout --`, or `git clean -fd` unless the user explicitly asked for that exact operation. If the worktree is dirty, inspect with `git status --short` and only stage files changed for the current task.

## Reading Files

Read a full file:

```powershell
Get-Content -Raw -LiteralPath 'app\components\BasicSelect.vue'
```

Preview a large file:

```powershell
Get-Content -LiteralPath 'large-output.txt' -TotalCount 80
```

`-TotalCount 80` means "first 80 lines"; 80 is only an example, not a default rule. Choose the count based on file size and task context. Use a small preview for orientation, a larger preview when definitions span farther, and `-Raw` only when the whole file is needed.

Do not combine `-Raw` with `Select-Object -First` to preview a file. `-Raw` returns the whole file as one string, so `Select-Object -First 1` still emits the entire file:

```powershell
Get-Content -Raw -LiteralPath 'large-output.txt' | Select-Object -First 1
```

Read a line range. PowerShell arrays are zero-based:

```powershell
$lines = Get-Content -LiteralPath 'app\pages\user\[address].vue'
$lines[120..160]
```

For a middle window without loading into a variable, use `Select-Object -Skip/-First`. Remember that `-Skip 120 -First 40` starts after 120 emitted lines and prints the next 40 lines; it does not print line numbers:

```powershell
Get-Content -LiteralPath 'app\pages\user\[address].vue' | Select-Object -Skip 120 -First 40
```

Print line numbers:

```powershell
$i = 0
Get-Content -LiteralPath 'app\pages\user\[address].vue' | ForEach-Object {
  $i++
  if ($i -ge 120 -and $i -le 160) { "${i}: $_" }
}
```

Find text with context:

```powershell
Select-String -LiteralPath 'app\pages\user\[address].vue' -Pattern 'loadMore' -Context 3,3
```

For large repo searches, prefer `rg`. For exact single-file PowerShell searches, use `Select-String -LiteralPath`.

## Quoting

Use single quotes by default. They prevent PowerShell from expanding `$`, backticks, and most special characters:

```powershell
rg -n 'const value = ref\(' app
git commit -m 'feat(user): use virtual tables for profile lists'
```

Use double quotes only when variable interpolation is intended:

```powershell
"branch=$branchName"
```

Avoid backtick escaping unless there is no cleaner option. Backticks are easy to miss in Markdown, JSON, and nested shell commands.

## Native Commands Vs Cmdlets

PowerShell cmdlets receive objects and support PowerShell-style parameters:

```powershell
Get-ChildItem -LiteralPath 'app\pages\user' -Filter '*.vue'
Remove-Item -LiteralPath $target -Recurse -Force
```

Native commands receive string arguments and use their own parameter syntax:

```powershell
pnpm run typecheck
git diff -- ':(literal)app/pages/user/[address].vue'
rg -n 'pattern' -- 'app/pages/user/[address].vue'
```

When the executable path itself is quoted, use PowerShell's call operator `&`. A quoted executable path without `&` is parsed as a string expression, and the next quoted argument can trigger `Unexpected token ... in expression or statement`.

Use this:

```powershell
& 'C:\Users\admin\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe' 'script.py' 'arg'
```

Do not use this:

```powershell
'C:\Users\admin\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe' 'script.py' 'arg'
```

Do not mix the models:

```powershell
rg -LiteralPath 'file'
git add -LiteralPath 'file'
pnpm -LiteralPath 'package.json'
```

## Command Chaining

Avoid long chains for state-changing work. Separate commands are easier to inspect and safer after failures.

Acceptable for small read-only checks:

```powershell
git status --short; git branch --show-current
```

Do not pipe directly from a PowerShell statement block such as `foreach (...) { ... }`. It can fail with `An empty pipe element is not allowed`. Collect results first or use `ForEach-Object`:

```powershell
$rows = @()
foreach ($file in $files) {
  $rows += [PSCustomObject]@{ Path = $file }
}
$rows | Format-Table -AutoSize

$files | ForEach-Object {
  [PSCustomObject]@{ Path = $_ }
} | Format-Table -AutoSize
```

Prefer separate commands for staging, committing, pushing, deleting, and moving:

```powershell
git add -- ':(literal)app/pages/user/[address].vue'
git commit -m 'feat(user): use virtual tables for profile lists'
git push origin dev
```

Read the tool-provided exit code first. In interactive PowerShell:

- `$?` reports whether the previous command succeeded.
- `$LASTEXITCODE` reports the exit code of the previous native command.

## Environment Variables

Set an environment variable for the current PowerShell process:

```powershell
$env:NODE_ENV = 'development'
pnpm run dev
```

Read it:

```powershell
$env:NODE_ENV
```

Do not use Bash syntax:

```powershell
export NODE_ENV=development
NODE_ENV=development pnpm run dev
```

## Writing Files And Encoding

Do not write source files with shell tricks:

```powershell
cat > file.ts
@'
content
'@ | Set-Content file.ts
```

Reasons:

- Windows PowerShell 5.1 and PowerShell 7 differ in default encoding.
- Here-strings often add unwanted newlines.
- `$` and quotes can be interpolated accidentally.
- Large shell-written files are hard to review.

Use `apply_patch` for source edits. Use formatting, generation, lint, and build commands normally.

If a temporary text file is unavoidable, specify encoding explicitly:

```powershell
Set-Content -LiteralPath 'tmp.txt' -Value $content -Encoding UTF8
```

## Deleting And Moving

Before recursive delete or move, resolve and inspect the target:

```powershell
$target = Resolve-Path -LiteralPath 'path\to\target'
$target.Path
```

Only after confirming the target is inside the intended directory:

```powershell
Remove-Item -LiteralPath $target.Path -Recurse -Force
Move-Item -LiteralPath $source.Path -Destination $destination.Path
```

Do not enumerate paths in PowerShell and then pass them as strings to `cmd /c del`, `bash -c rm`, or another shell for deletion or moving. Keep the operation in one shell, preferably native PowerShell cmdlets with `-LiteralPath`.

## JSON, Regex, And Special Characters

Wrap JSON in single quotes when possible:

```powershell
'{"name":"xmarket","enabled":true}'
```

For literal text, avoid regex unless needed:

```powershell
rg -n -F 'literal[text]' app
Select-String -LiteralPath 'file.vue' -SimpleMatch 'route.params.address'
```

For regex, still prefer single quotes:

```powershell
rg -n 'defineModel<[^>]+>' app
```

## Common Translations

| Task | Use |
| --- | --- |
| Read a dynamic route file | `Get-Content -Raw -LiteralPath 'app\pages\user\[address].vue'` |
| Search text | `rg -n 'pattern' app -S` |
| Search literal text | `rg -n -F 'literal[text]' app` |
| Search one file with rg | `rg -n 'pattern' -- 'app/pages/user/[address].vue'` |
| Search env files with rg | `rg -n -S 'pattern' -g '.env*' -- .` |
| Preview a large file | `Get-Content -LiteralPath 'large-output.txt' -TotalCount 80` |
| Read line range | `$lines = Get-Content -LiteralPath 'file'; $lines[10..30]` |
| Set env var | `$env:NAME = 'value'` |
| Git diff for `[file]` | `git diff -- ':(literal)app/pages/user/[address].vue'` |
| Git add for `[file]` | `git add -- ':(literal)app/pages/user/[address].vue'` |
| Complete changed-file overview | `git status --short` plus `git diff --name-only` |
| Delete exact file | `Remove-Item -LiteralPath 'path'` |
| Recursive delete directory | `Resolve-Path -LiteralPath 'path'`, inspect it, then `Remove-Item -LiteralPath $resolved.Path -Recurse -Force` |

## Preflight Checklist

Before running a PowerShell command, ask:

1. Does any path contain `[]`, spaces, parentheses, non-ASCII characters, or glob characters?
2. Is this a PowerShell cmdlet or a native command?
3. Should the command use `-LiteralPath`, or should the path be passed after `--`?
4. If the executable path is quoted, did I prefix it with `&`?
5. Will the command write, delete, move, stage, commit, or push anything?
6. Is any part of the command actually Bash syntax?
7. Could the output be huge, and should it use a context-sized `-TotalCount`, a line range, or an `rg` filter instead of `-Raw`?
8. Am I relying on `git diff` output and accidentally ignoring untracked files? Check `git status --short`.
9. Is the output clean enough to interpret and summarize for the user?

If uncertain, run a read-only inspection command first, then perform the state-changing command only after the target is clear.
