# UpdateBetterBot

Version: `v1.0.0`

`UpdateBetterBot` is a Windows updater that syncs files from `Qwepplz/Betterbot` into the folder that contains `UpdateBetterBot.exe`. The target machine does not need Git; the updater reads repository metadata and file contents directly over HTTPS, trying Gitee first and GitHub second.

## Contents

| Language | Link |
| --- | --- |
| English | [English Guide](#english-guide) |
| 中文 | [中文说明](#中文说明) |

## English Guide

### Purpose

| Item | Description |
| --- | --- |
| Goal | Sync repository-managed `Betterbot` files into the folder where `UpdateBetterBot.exe` is running. |
| Platform | Windows. |
| Git requirement | None on the target machine. |
| Upstream repository | `Qwepplz/Betterbot`. |
| Access order | Gitee mirror first, matching GitHub repository second. |
| Order meaning | Both sources are treated as equivalent entrances for the same content; the order is only about network convenience, not content priority. |

### Quick Start

| Step | Action |
| --- | --- |
| 1 | Build with `tools\Build-UpdateBetterBot.bat`. |
| 2 | Get the output file at `dist\UpdateBetterBot.exe`. |
| 3 | Copy `dist\UpdateBetterBot.exe` into the BetterBot folder you want to sync. |
| 4 | Run `UpdateBetterBot.exe`. |
| 5 | Press `ENTER` to start syncing, or press `ESC` to exit. |
| 6 | If a run fails, inspect `log\UpdateBetterBot-YYYY-MM-DD.log` beside the updater. |

### Startup Prompt

| Input | Action |
| --- | --- |
| `ENTER` | Start syncing the current folder shown by the program. |
| `ESC` | Exit immediately without syncing. |
| Input unavailable | If key input cannot be read, the updater continues into sync automatically. |

### Sync Sources and Network Strategy

| Data | First source | Second source |
| --- | --- | --- |
| Repository metadata | `https://gitee.com/api/v5/repos/SaUrrr/Betterbot` | `https://api.github.com/repos/Qwepplz/Betterbot` |
| Repository tree | `https://gitee.com/api/v5/repos/SaUrrr/Betterbot/git/trees/<branch>?recursive=1` | `https://api.github.com/repos/Qwepplz/Betterbot/git/trees/<branch>?recursive=1` |
| Raw file download | `https://gitee.com/SaUrrr/Betterbot/raw/<branch>/<path>` | `https://raw.githubusercontent.com/Qwepplz/Betterbot/<branch>/<path>` |

- The updater first queries the default branch of the repository.
- If default-branch lookup fails, it still tries common branch names such as `main` and `master`.
- Repository trees must be complete. If a remote API returns a truncated tree, sync is aborted because deletion would no longer be safe.
- Network requests use TLS 1.2 with a fixed `15000` ms timeout.

### Sync Flow

| Stage | Console label | Description |
| --- | --- | --- |
| 1 | `[1/4] Reading repository tree...` | Reads the default/common branch and the full remote file tree. |
| 2 | `[2/4] Removing repo README/LICENSE when safe...` | Handles root README/LICENSE files specially so local docs are not blindly overwritten. |
| 3 | `[3/4] Downloading and updating files...` | Adds missing files, updates changed files, and skips unchanged files through cache and Git blob checks. |
| 4 | `[4/4] Removing files deleted upstream...` | Deletes only files that were previously tracked by this updater and are now gone upstream. |

### File Rules

| File type | Behavior |
| --- | --- |
| Normal tracked file | Added if missing, updated if changed, skipped if unchanged through cache comparison or Git blob SHA-1 verification. |
| Root `README*`, `LICENSE*`, `LICENCE*`, `LECENSE*` | Not treated as normal sync files; an old synced copy may be removed only when it exactly matches upstream and is safe to touch. |
| File deleted upstream | Removed only if it appears in the updater's previous manifest/state. |
| Updater executable and helper files | Protected from overwrite and deletion. |
| Existing log files | Protected from sync operations. |

### Safety Guarantees

| Area | Behavior |
| --- | --- |
| Target boundary | Refuses to touch paths outside the selected target folder. |
| Reparse points | Refuses to touch paths that pass through a reparse point. |
| Directory conflicts | Stops if a target file path is actually an existing directory. |
| Concurrent runs | Uses a named mutex per target folder to prevent simultaneous syncs into the same location. |
| File verification | Verifies each downloaded file against the Git blob SHA-1 from the remote tree. |
| Replacement mode | Uses staging files plus `File.Replace` / `File.Move` for atomic-style replacement. |
| Cleanup | Removes stale `.__betterbot_sync_staging__*` and `.__betterbot_sync_backup__*` files on startup when safe. |
| System behavior | Does not run shell commands, modify registry startup entries, or create scheduled tasks. |

Protected local targets include:

| Type | Description |
| --- | --- |
| Running executable | The updater protects the actual executable path it is currently running from. |
| Existing local helper files | The updater also protects `_UpdateBetterBot.bat`, `_UpdateBetterBot.ps1`, `UpdateBetterBot.cs`, `Build-UpdateBetterBot.bat`, `Build-UpdateBetterBot.cmd`, and `UpdateBetterBot.exe` if they already exist in the target folder. |
| Existing log files | Files already inside the target folder's `log` directory are protected. |

### Cache, State, and Logs

| Item | Description |
| --- | --- |
| State root priority | `BETTERBOT_SYNC_STATE` → `%LOCALAPPDATA%\BetterBotSync` → `%APPDATA%\BetterBotSync` → `%TEMP%\BetterBotSync` |
| Target isolation | The updater hashes the target folder path with SHA-256 and stores state under that hash. |
| State files | `tracked-files.txt` and `sync-state.json` |
| Cached metadata | Remote blob SHA-1, local file length, and local last-write UTC ticks |
| Log directory | `log` beside `UpdateBetterBot.exe` |
| Log naming | One file per day, such as `log\UpdateBetterBot-2026-04-14.log` |
| Log format | Each line is timestamped as `yyyy-MM-dd HH:mm:ss.fff` |

### Repository Layout

Only tracked project files that currently exist in this repository are listed below.

| Path | Description |
| --- | --- |
| `README.md` | Project documentation. |
| `LICENSE` | License file. |
| `src\UpdateBetterBot\Program.cs` | Single-file C# implementation containing the startup prompt, sync logic, network access, safety checks, progress display, cache handling, and logging. |
| `tools\Build-UpdateBetterBot.bat` | Windows build script for `dist\UpdateBetterBot.exe`, with optional code signing. |

### Runtime-generated Paths

These paths are created locally during build or sync and are not tracked as repository files.

| Path | Description |
| --- | --- |
| `dist\UpdateBetterBot.exe` | Default local build output produced by `tools\Build-UpdateBetterBot.bat`. |
| `log\` | Runtime log directory created beside `UpdateBetterBot.exe` when logging is available. |
| `<state-root>\<target-hash>\tracked-files.txt` | Legacy tracked-file manifest for the selected target folder. |
| `<state-root>\<target-hash>\sync-state.json` | Current JSON cache/state for the selected target folder. |

### Build

Build on Windows with:

```bat
tools\Build-UpdateBetterBot.bat
```

Skip the final pause with:

```bat
tools\Build-UpdateBetterBot.bat --no-pause
```

The script first looks for:

```text
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe
```

If that compiler is not available, it falls back to:

```text
C:\Windows\Microsoft.NET\Framework\v4.0.30319\csc.exe
```

Build references:

| Reference | Purpose |
| --- | --- |
| `System.Web.Extensions.dll` | `JavaScriptSerializer` JSON parsing |
| `System.Core.dll` | LINQ and related base functionality |

Default output:

```text
dist\UpdateBetterBot.exe
```

### Optional Code Signing

If you have a real `.pfx` code-signing certificate, the build script can sign the generated EXE automatically:

```bat
tools\Build-UpdateBetterBot.bat "C:\path\code-signing.pfx" "password"
```

Or together with `--no-pause`:

```bat
tools\Build-UpdateBetterBot.bat --no-pause "C:\path\code-signing.pfx" "password"
```

The script searches `PATH` and common Windows SDK locations for `signtool.exe`.

### Verify a Build Artifact

```bat
certutil -hashfile dist\UpdateBetterBot.exe SHA256
```

Use the resulting SHA-256 digest for release verification or pre-distribution checks.

### Troubleshooting

| Symptom | Suggestion |
| --- | --- |
| `C# compiler was not found.` | Install or enable a .NET Framework 4.x compiler on Windows. |
| `Another BetterBot sync is already running for this folder.` | Close the other updater instance for the same target folder and retry. |
| `Cannot create sync state directory. Last error: ...` | Set `BETTERBOT_SYNC_STATE` to a writable folder or fix permissions for `%LOCALAPPDATA%`, `%APPDATA%`, or `%TEMP%`. |
| `Cannot read repository tree.` | Retry later or from another network; sync stops intentionally when the remote tree cannot be trusted. |
| `All download attempts failed for ...` | Check access to Gitee/GitHub raw endpoints, proxy/network restrictions, or antivirus interference. |
| `Sync cache unreadable. Rebuilding...` | The updater will rebuild cache from current files; rerun only if later steps still fail. |
| `Refusing to touch a path outside the target folder` | Check where the updater was copied and whether the resolved path still points inside the intended target. |
| `Refusing to touch a reparse point path` | Remove or replace junctions / symlinks / reparse points in the target path and retry. |
| Output is not enough to diagnose a failure | Open the current day's file in `log\`. |

## 中文说明

### 项目定位

| 项目 | 说明 |
| --- | --- |
| 程序目标 | 将上游仓库 `Betterbot` 受管理的文件同步到 `UpdateBetterBot.exe` 所在目录。 |
| 目标平台 | Windows。 |
| 目标机器依赖 | 不需要安装 Git。 |
| 当前上游仓库 | `Qwepplz/Betterbot`。 |
| 默认访问顺序 | 先访问 Gitee 镜像，再访问 GitHub 对应仓库。 |
| 顺序含义 | 两组地址被视为内容应保持一致的同步入口；先后顺序只用于访问便利，不表示主备或内容优先级。 |

### 快速开始

| 步骤 | 操作 |
| --- | --- |
| 1 | 在仓库根目录运行 `tools\Build-UpdateBetterBot.bat`。 |
| 2 | 构建成功后得到 `dist\UpdateBetterBot.exe`。 |
| 3 | 将 `dist\UpdateBetterBot.exe` 复制到需要同步的 BetterBot 目录。 |
| 4 | 运行 `UpdateBetterBot.exe`。 |
| 5 | 按 `ENTER` 开始同步，或按 `ESC` 退出。 |
| 6 | 如果同步失败，请查看更新器旁边的 `log\UpdateBetterBot-YYYY-MM-DD.log`。 |

### 启动提示

| 输入 | 行为 |
| --- | --- |
| `ENTER` | 开始同步程序当前显示的目标目录。 |
| `ESC` | 立即退出，不执行同步。 |
| 无法读取输入 | 如果无法读取按键输入，程序会自动继续执行同步。 |

### 同步源与网络策略

| 数据 | 第一访问入口 | 第二访问入口 |
| --- | --- | --- |
| 仓库信息 | `https://gitee.com/api/v5/repos/SaUrrr/Betterbot` | `https://api.github.com/repos/Qwepplz/Betterbot` |
| 仓库树 | `https://gitee.com/api/v5/repos/SaUrrr/Betterbot/git/trees/<branch>?recursive=1` | `https://api.github.com/repos/Qwepplz/Betterbot/git/trees/<branch>?recursive=1` |
| 原始文件下载 | `https://gitee.com/SaUrrr/Betterbot/raw/<branch>/<path>` | `https://raw.githubusercontent.com/Qwepplz/Betterbot/<branch>/<path>` |

- 程序会先查询仓库默认分支。
- 如果默认分支查询失败，还会继续尝试常见分支名 `main` 与 `master`。
- 仓库树必须完整返回；如果远端 API 返回截断树，程序会终止同步，因为那样删除逻辑将不再安全。
- 网络请求使用 TLS 1.2，并带固定 `15000` 毫秒超时。

### 同步流程

| 阶段 | 控制台标识 | 说明 |
| --- | --- | --- |
| 1 | `[1/4] Reading repository tree...` | 读取默认/候选分支以及完整远端文件树。 |
| 2 | `[2/4] Removing repo README/LICENSE when safe...` | 特殊处理根目录 README/LICENSE 文件，避免粗暴覆盖本地说明文档。 |
| 3 | `[3/4] Downloading and updating files...` | 新增缺失文件、更新已变化文件，并通过缓存与 Git blob 校验跳过未变化文件。 |
| 4 | `[4/4] Removing files deleted upstream...` | 只删除此前由本更新器跟踪、且当前已从上游移除的文件。 |

### 文件处理规则

| 文件类型 | 处理方式 |
| --- | --- |
| 普通跟踪文件 | 不存在则新增，内容变化则更新，未变化则通过缓存比较或 Git blob SHA-1 校验直接跳过。 |
| 根目录 `README*`、`LICENSE*`、`LICENCE*`、`LECENSE*` | 不按普通同步文件处理；只有在本地副本与上游完全一致且安全可触碰时，旧同步副本才可能被移除。 |
| 上游已删除文件 | 仅当该文件出现在此前的清单/状态记录里时才会删除。 |
| 更新器自身与辅助文件 | 受保护，不会被覆盖或删除。 |
| 现有日志文件 | 受保护，不参与同步操作。 |

### 安全保证

| 领域 | 行为 |
| --- | --- |
| 目标目录边界 | 拒绝触碰目标目录之外的路径。 |
| 重解析点 | 拒绝触碰经过 reparse point 的路径。 |
| 目录冲突 | 如果目标文件路径实际是一个已有目录，程序会停止。 |
| 并发运行 | 针对每个目标目录使用命名互斥锁，避免同时写入同一位置。 |
| 文件校验 | 每个下载文件都会用远端仓库树提供的 Git blob SHA-1 做校验。 |
| 替换方式 | 使用 staging 临时文件加 `File.Replace` / `File.Move` 进行原子式替换。 |
| 启动清理 | 启动时会在安全前提下清理残留的 `.__betterbot_sync_staging__*` 与 `.__betterbot_sync_backup__*` 文件。 |
| 系统行为 | 不会执行 shell 命令、不会修改注册表启动项、不会创建计划任务。 |

受保护的本地目标包括：

| 类型 | 说明 |
| --- | --- |
| 当前运行的可执行文件 | 程序会保护自己当前实际运行的 EXE 路径。 |
| 已存在的本地辅助文件 | 如果目标目录里已经存在 `_UpdateBetterBot.bat`、`_UpdateBetterBot.ps1`、`UpdateBetterBot.cs`、`Build-UpdateBetterBot.bat`、`Build-UpdateBetterBot.cmd`、`UpdateBetterBot.exe`，程序也会保护它们。 |
| 已存在的日志文件 | 目标目录 `log` 目录内已有的文件会被保护。 |

### 缓存、状态与日志

| 项目 | 说明 |
| --- | --- |
| 状态根目录优先级 | `BETTERBOT_SYNC_STATE` → `%LOCALAPPDATA%\BetterBotSync` → `%APPDATA%\BetterBotSync` → `%TEMP%\BetterBotSync` |
| 目标目录隔离 | 程序会对目标目录路径做 SHA-256 哈希，并将状态存放到对应子目录。 |
| 状态文件 | `tracked-files.txt` 与 `sync-state.json` |
| 缓存内容 | 远端 blob SHA-1、本地文件长度、本地最后写入 UTC ticks |
| 日志目录 | `UpdateBetterBot.exe` 同级的 `log` 目录 |
| 日志命名 | 按天生成一个文件，例如 `log\UpdateBetterBot-2026-04-14.log` |
| 日志格式 | 每行都会带 `yyyy-MM-dd HH:mm:ss.fff` 时间戳 |

### 仓库结构

这里只列出当前仓库中实际受版本控制的项目文件。

| 路径 | 说明 |
| --- | --- |
| `README.md` | 项目说明文档。 |
| `LICENSE` | 许可证文件。 |
| `src\UpdateBetterBot\Program.cs` | 单文件 C# 主实现，包含启动提示、同步逻辑、网络访问、安全校验、进度显示、缓存处理和日志逻辑。 |
| `tools\Build-UpdateBetterBot.bat` | Windows 构建脚本，用于生成 `dist\UpdateBetterBot.exe`，并支持可选代码签名。 |

### 运行时生成路径

以下路径是在本地构建或同步时生成的，不属于仓库已提交文件。

| 路径 | 说明 |
| --- | --- |
| `dist\UpdateBetterBot.exe` | `tools\Build-UpdateBetterBot.bat` 生成的默认本地构建产物。 |
| `log\` | 运行时在 `UpdateBetterBot.exe` 同级创建的日志目录。 |
| `<状态根目录>\<目标目录哈希>\tracked-files.txt` | 面向当前目标目录的旧式跟踪清单。 |
| `<状态根目录>\<目标目录哈希>\sync-state.json` | 面向当前目标目录的当前 JSON 缓存/状态文件。 |

### 构建

在 Windows 上构建：

```bat
tools\Build-UpdateBetterBot.bat
```

如需跳过最后的暂停：

```bat
tools\Build-UpdateBetterBot.bat --no-pause
```

脚本会优先查找：

```text
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe
```

若不存在，则回退到：

```text
C:\Windows\Microsoft.NET\Framework\v4.0.30319\csc.exe
```

构建引用：

| 引用 | 用途 |
| --- | --- |
| `System.Web.Extensions.dll` | 使用 `JavaScriptSerializer` 解析 JSON |
| `System.Core.dll` | LINQ 等基础能力 |

默认输出：

```text
dist\UpdateBetterBot.exe
```

### 可选代码签名

如果你有真实可用的 `.pfx` 代码签名证书，构建脚本可以在生成 EXE 后自动签名：

```bat
tools\Build-UpdateBetterBot.bat "C:\path\code-signing.pfx" "password"
```

也可以与 `--no-pause` 组合使用：

```bat
tools\Build-UpdateBetterBot.bat --no-pause "C:\path\code-signing.pfx" "password"
```

脚本会在 `PATH` 与常见 Windows SDK 安装路径中查找 `signtool.exe`。

### 校验构建产物

```bat
certutil -hashfile dist\UpdateBetterBot.exe SHA256
```

得到的 SHA-256 摘要可用于发布校验或分发前自检。

### 常见问题

| 现象 | 建议 |
| --- | --- |
| `C# compiler was not found.` | 在 Windows 上安装或启用 .NET Framework 4.x 编译器。 |
| `Another BetterBot sync is already running for this folder.` | 关闭同一目标目录下的另一个更新器实例后重试。 |
| `Cannot create sync state directory. Last error: ...` | 设置一个可写的 `BETTERBOT_SYNC_STATE`，或检查 `%LOCALAPPDATA%`、`%APPDATA%`、`%TEMP%` 的权限。 |
| `Cannot read repository tree.` | 稍后重试或切换网络；程序在无法信任远端树时会主动停止。 |
| `All download attempts failed for ...` | 检查 Gitee / GitHub 原始文件地址可达性、代理/网络限制或杀毒软件干扰。 |
| `Sync cache unreadable. Rebuilding...` | 程序会按当前文件重建缓存；只有在后续步骤仍失败时才需要再次处理。 |
| `Refusing to touch a path outside the target folder` | 检查更新器被复制到的位置，以及解析后的路径是否仍在预期目标目录内。 |
| `Refusing to touch a reparse point path` | 去掉或替换目标路径中的 junction / symlink / reparse point 后再试。 |
| 控制台信息不足以定位错误 | 打开 `log` 目录中的当日日志文件。 |
