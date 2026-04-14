# AutoupdateForBetterbot

Version: `v1.0.0`

## English

### Overview

`AutoupdateForBetterbot` is a Windows updater for `Qwepplz/BetterBot`. Place `dist\UpdateBetterBot.exe` in a BetterBot folder, confirm the prompt, and it reads the GitHub repository tree and syncs upstream-managed files into the current directory without requiring Git.

### Features

- Syncs files from the default branch of `Qwepplz/BetterBot` into the folder that contains `UpdateBetterBot.exe`.
- Falls back to common branch names such as `main` and `master` if default-branch lookup fails.
- Overwrites local files only when content really changed, and skips unchanged files.
- Removes only files that were previously tracked by this updater and later removed upstream.
- Excludes root-level `README*`, `LICENSE*`, `LICENCE*`, and `LECENSE*` files from sync. If a local copy exactly matches upstream and is safe to touch, the old synced copy may be removed.
- Protects the updater itself and known helper files so it does not overwrite its own tooling.
- Uses isolated cache data per target directory and a mutex to prevent concurrent syncs for the same folder.
- Writes updated files atomically to reduce the risk of corruption during replacement.
- Keeps download/update progress compact by refreshing two status lines, with the progress bar on its own full-width line instead of printing one new console line for every file.
- Mirrors console output into a `log` folder beside `UpdateBetterBot.exe`, writes one timestamped file per day such as `log\UpdateBetterBot-2026-04-12.log`, and keeps appending to that day's file.
- Does not run shell commands, modify registry startup entries, or create scheduled tasks.

### Repository Layout

| Path | Description |
| --- | --- |
| `src\UpdateBetterBot\Program.cs` | Current main implementation: single-file C# updater source. |
| `tools\Build-UpdateBetterBot.bat` | Build script that produces `dist\UpdateBetterBot.exe` and optionally signs it. |
| `dist\UpdateBetterBot.exe` | Output directory for the built executable. |
| `log\UpdateBetterBot-YYYY-MM-DD.log` | Runtime log file written in the target folder's `log` directory, one file per day. |
| `legacy\_UpdateBetterBot.bat` | Legacy batch launcher kept for historical reference. |
| `legacy\_UpdateBetterBot.ps1` | Legacy PowerShell sync implementation kept for historical reference. |

### Build

Build on Windows with:

```bat
tools\Build-UpdateBetterBot.bat
```

The script first looks for `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe` and falls back to the `Framework` compiler if needed.

To skip the final pause:

```bat
tools\Build-UpdateBetterBot.bat --no-pause
```

Default output:

```text
dist\UpdateBetterBot.exe
```

### Optional Code Signing

If you have a real `.pfx` code-signing certificate, the EXE can be signed automatically after the build. The script looks for `signtool.exe` in `PATH` or common Windows SDK locations.

```bat
tools\Build-UpdateBetterBot.bat "C:\path\code-signing.pfx" "password"
```

```bat
tools\Build-UpdateBetterBot.bat --no-pause "C:\path\code-signing.pfx" "password"
```

Unsigned builds can still run, but because the program reads repository metadata over the network and updates local files, antivirus products or SmartScreen may still show heuristic warnings.

### Usage

1. Copy `dist\UpdateBetterBot.exe` into the BetterBot folder you want to sync.
2. Launch the program and confirm that the displayed target folder is correct.
3. Press `ENTER` to start syncing, or press `ESC` to exit.
4. Wait for the four stages to finish: read the repository tree, handle root README/LICENSE files, download updates, and remove upstream-deleted files.
5. If you need to troubleshoot a run, open today's file in the `log` folder beside the updater.

### Cache and State

Sync state is stored in the first available location below:

1. `BETTERBOT_SYNC_STATE`
2. `%LOCALAPPDATA%\BetterBotSync`
3. `%APPDATA%\BetterBotSync`
4. `%TEMP%\BetterBotSync`

Each target folder gets its own hashed subdirectory. Common state files include `tracked-files.txt` and `sync-state.json`.

### Safety Notes

- The updater refuses to touch paths outside the target folder.
- If a target path is actually a directory, the updater stops instead of overwriting it.
- Reparse-point paths are rejected to reduce the risk of accidentally touching linked locations.
- Temporary download files and replacement backups are cleaned up as much as possible after success or failure.

### Verify a Release File

```bat
certutil -hashfile dist\UpdateBetterBot.exe SHA256
```

You can use the resulting SHA-256 digest for release verification or pre-distribution checks.

## 中文

### 概览

`AutoupdateForBetterbot` 是一个面向 Windows 的 `Qwepplz/BetterBot` 更新器。将 `dist\UpdateBetterBot.exe` 放到 BetterBot 目录后，程序会在用户确认后从 GitHub 读取仓库树，并把受上游仓库管理的文件同步到当前目录，全程不依赖 Git。

### 功能特性

- 把 `Qwepplz/BetterBot` 默认分支中的文件同步到 `UpdateBetterBot.exe` 所在目录。
- 如果默认分支查询失败，会继续尝试常见分支名，例如 `main` 和 `master`。
- 只有在文件内容确实变化时才覆盖本地文件；未变化文件会直接跳过。
- 只会删除此前由本更新器跟踪、且已从上游仓库移除的文件。
- 根目录中的 `README*`、`LICENSE*`、`LICENCE*`、`LECENSE*` 文件默认不参与同步；若本地文件与上游完全一致且可安全处理，旧的同步副本会被移除。
- 会保护更新器自身及常见辅助文件，避免把更新器覆盖掉。
- 每个目标目录都有独立缓存，并通过互斥锁防止同一目录同时运行多个同步任务。
- 更新后的文件会以原子方式写入，尽量降低写入中断导致文件损坏的风险。
- 下载/更新阶段的文件级进度会以两行控制台状态原地刷新，其中进度条单独占满一整行，避免每个文件都新增一行输出。
- 会把控制台输出同时写入 `UpdateBetterBot.exe` 同目录下的 `log` 文件夹，并按日期生成日志文件（例如 `log\UpdateBetterBot-2026-04-12.log`）；每行都会带时间戳，并持续追加到当天文件。
- 程序不会执行 shell 命令、不会修改注册表启动项，也不会创建计划任务。

### 仓库结构

| 路径 | 说明 |
| --- | --- |
| `src\UpdateBetterBot\Program.cs` | 当前主实现，使用 C# 编写的单文件更新器源码。 |
| `tools\Build-UpdateBetterBot.bat` | 构建脚本，可生成 `dist\UpdateBetterBot.exe`，也支持可选签名。 |
| `dist\UpdateBetterBot.exe` | 已构建的可执行文件输出目录。 |
| `log\UpdateBetterBot-YYYY-MM-DD.log` | 运行时日志文件，写在目标目录下的 `log` 文件夹中，并按天分文件。 |
| `legacy\_UpdateBetterBot.bat` | 旧版批处理启动器，保留作历史参考。 |
| `legacy\_UpdateBetterBot.ps1` | 旧版 PowerShell 同步实现，保留作历史参考。 |

### 构建

在 Windows 环境下运行：

```bat
tools\Build-UpdateBetterBot.bat
```

脚本会优先使用 `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe`，如果不存在则回退到 `Framework` 目录下的编译器。

如果希望构建结束后不暂停窗口：

```bat
tools\Build-UpdateBetterBot.bat --no-pause
```

默认输出文件：

```text
dist\UpdateBetterBot.exe
```

### 可选代码签名

如果你有真实可用的代码签名证书 `.pfx`，可以在构建后自动对 EXE 签名。脚本会尝试从 `PATH` 或常见 Windows SDK 安装目录中寻找 `signtool.exe`。

```bat
tools\Build-UpdateBetterBot.bat "C:\path\code-signing.pfx" "password"
```

```bat
tools\Build-UpdateBetterBot.bat --no-pause "C:\path\code-signing.pfx" "password"
```

即使是未签名版本，也可以正常运行；但由于程序会联网读取仓库树并更新本地文件，仍可能触发杀毒软件或 SmartScreen 的启发式警告。

### 使用方法

1. 将 `dist\UpdateBetterBot.exe` 复制到需要同步的 BetterBot 目录中。
2. 双击运行程序，确认目标目录无误。
3. 按 `ENTER` 开始同步，或按 `ESC` 直接退出。
4. 等待程序完成四个阶段：读取仓库树、处理根目录 README/LICENSE、下载更新文件、删除上游已移除文件。
5. 如果需要排查运行问题，请打开与更新器同目录下 `log` 文件夹中的当日日志。

### 缓存与状态

同步状态会写入以下第一个可用位置：

1. `BETTERBOT_SYNC_STATE`
2. `%LOCALAPPDATA%\BetterBotSync`
3. `%APPDATA%\BetterBotSync`
4. `%TEMP%\BetterBotSync`

每个目标目录会生成独立的哈希子目录，常见状态文件包括 `tracked-files.txt` 和 `sync-state.json`。

### 安全说明

- 程序拒绝处理目标目录之外的路径。
- 如果某个目标路径实际上是目录，程序会中止而不是覆盖它。
- 程序会检查并避开重解析点路径，减少误操作链接路径的风险。
- 临时下载文件和替换备份文件会在成功或失败后尽量清理。

### 校验发布文件

```bat
certutil -hashfile dist\UpdateBetterBot.exe SHA256
```

你可以把得到的 SHA-256 摘要用于发布校验或分发前自检。
