# CMirror v1.3c

A simple source code control utility for Windows that performs **three-way file synchronization** across a workspace, a local repository, and a central (server) repository.

Originally written by Casey Muratori, with contributions from Sean Barrett and Jeff Roberts. Released into the public domain.

> **WARNING:** The authors make **no warranty** as to the reliability, suitability, or usability of this software. It works with files — probably files that are important to you. There might be bad bugs in here. It could delete them all. CMirror is not a substitute for making backups.

---

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Repository Layout](#repository-layout)
- [File Versioning](#file-versioning)
- [Building](#building)
- [Usage](#usage)
- [Configuration File](#configuration-file)
  - [Directory Settings](#directory-settings)
  - [File Rules](#file-rules)
  - [Sync Behaviour](#sync-behaviour)
  - [Diff Methods](#diff-methods)
  - [Output Options](#output-options)
  - [Custom Commands](#custom-commands)
- [Sync Modes](#sync-modes)
- [Conflict Handling](#conflict-handling)
- [Pre-Checkin Hooks & ckeyword](#pre-checkin-hooks--ckeyword)
- [Performance: Central Cache](#performance-central-cache)
- [Command-Line Reference](#command-line-reference)
- [Example Configuration](#example-configuration)

---

## Overview

CMirror solves a specific workflow: you have one or more developers each with a **local machine**, each talking to a shared **central server**, and each doing active development in a **workspace** directory. CMirror keeps all three in sync, tracking file versions so you can roll back changes and detect when two parties have edited the same file simultaneously.

It is deliberately simple — no branches, no merging of file contents, no network protocol. It works entirely through the filesystem, which makes it robust on Windows network shares (SMB/UNC paths).

---

## How It Works

CMirror performs a two-phase synchronization on every run:

```
Phase 1: Workspace <-> Local
   Detect new, changed, or deleted files in the workspace.
   Propagate those changes into the local repository.

Phase 2: Local <-> Central
   Push new local versions to the central repository.
   Pull new central versions (from other developers) into local.
```

Before executing any file operations CMirror prints a summary and prompts the user to confirm. After confirmation it executes every queued operation atomically (from the user's perspective).

---

## Repository Layout

Each repository directory (local and central) stores **versioned copies** of files alongside the originals using a naming convention described in the next section. The workspace directory contains only the plain, unversioned working copies of files.

```
project/
├── cmirror.config          ← CMirror configuration file
├── workspace/              ← Your working files (no version numbers)
│   ├── main.c
│   └── utils.h
├── local/                  ← Local repository (versioned files)
│   ├── main,cm00000001.c
│   ├── main,cm00000002.c
│   └── utils,cm00000001.h
└── central/                ← Central/server repository (versioned files)
    ├── main,cm00000001.c
    ├── main,cm00000002.c
    └── utils,cm00000001.h
```

---

## File Versioning

CMirror embeds the version number directly into the filename using a `,cm` infix before the last file extension:

| File in workspace | Versioned copy (new style) |
|---|---|
| `main.c`           | `main,cm00000003.c`         |
| `README`           | `README,cm00000001`         |
| `data.tar.gz`      | `data,cm00000002.tar.gz`    |

Deleted versions use a `d` prefix on the version number:

| Description | Filename |
|---|---|
| Version 4 (live)    | `main,cm00000004.c`   |
| Version 4 (deleted) | `main,cmd00000004.c`  |

An older "legacy" naming convention (`filename,NNN`) is also recognized for backwards compatibility.

Version numbers are monotonically increasing integers. The highest version number present in the repository is the current version. CMirror never overwrites an existing version; it always adds a new numbered copy.

---

## Building

CMirror is a single C/C++ source file for Windows. There is no build system; compile with any MSVC-compatible compiler:

**Visual Studio (command line):**
```
cl /O2 cmirror.cpp
```

**Visual Studio 2022 (project):**
Add `cmirror.cpp` and `stb.h` to a new empty C++ console project and build.

Dependencies:
- `stb.h` — Sean Barrett's single-header utility library (included in the repo)
- Windows SDK (Win32 API — no extra libraries beyond the standard CRT and `kernel32`)

`ckeyword.cpp` is a standalone utility; build it separately:
```
cl /O2 ckeyword.cpp
```

---

## Usage

Run CMirror from any directory inside your workspace:

```
cmirror [config-file] [-config "key = value; ..."]
```

CMirror searches upward from the current directory for a file named `cmirror.config`. You can override this by passing an explicit path as the first argument.

The `-config` flag lets you pass extra configuration statements inline on the command line (applied after the config file is parsed).

**Typical workflow:**

```bat
cd C:\projects\myproject
cmirror
```

CMirror will:
1. Locate and parse `cmirror.config`
2. Scan all three directories
3. Print the operations it intends to perform
4. Prompt you to confirm (`y` / `n` / `a` for "always continue")
5. Execute the operations

---

## Configuration File

CMirror looks for a file named `cmirror.config` starting from the current directory and walking up the directory tree. It can also reside in a directory named `cmirror.config` (in which case CMirror looks for the file inside that directory).

The config file uses a simple `key = value;` syntax. Comments begin with `//`. Strings can be quoted with `"` or `'`.

```
// This is a comment
Central = "//server/share/central";
Local   = "C:/local_repo";
Ignore  = "*.obj;*.exe";
```

### Directory Settings

| Key | Type | Default | Description |
|---|---|---|---|
| `Workspace` | path | current directory | Working directory (plain files, no versions) |
| `Local` | path | *(required)* | Local repository directory |
| `Central` | path | *(required)* | Central (server) repository directory |
| `CentralCache` | path | `""` | Path for the central directory cache file (see [Performance](#performance-central-cache)) |

**Retry on network failure:** If a repository directory is temporarily unavailable (e.g. a network share), CMirror will retry access before giving up:

| Key | Type | Default | Description |
|---|---|---|---|
| `LocalRetryTimes` | integer | `0` | Number of extra attempts to access the local directory |
| `WorkspaceRetryTimes` | integer | `0` | Number of extra attempts to access the workspace directory |
| `CentralRetryTimes` | integer | `0` | Number of extra attempts to access the central directory |

### File Rules

Rules are evaluated in order; the **last matching rule** for a given setting wins.

Multiple wildcard patterns can be combined in a single rule by separating them with semicolons:

```
Ignore = "*.obj;*.pdb;*.ilk";
```

#### `Ignore` — Exclude files from sync

```
Ignore = "*.obj";
Ignore = "build/*";
```

Files matching the pattern are treated as if they do not exist. They are never copied, versioned, or deleted by CMirror.

#### `Allow` — Re-include previously ignored files

```
Ignore = "*";
Allow  = "*.c;*.h";
```

Overrides a prior `Ignore` rule. Useful for an allowlist approach: ignore everything, then explicitly allow specific extensions.

#### `IgnoreConflicts` — Suppress conflict halting for a pattern

```
IgnoreConflicts = "generated/*.h";
```

By default, if CMirror detects that a file has been changed on both the local and central sides simultaneously (a conflict), it halts and asks the user what to do. `IgnoreConflicts` makes CMirror silently resolve such conflicts by preferring the central version.

#### `HaltOnConflicts` — Restore default conflict behaviour

```
HaltOnConflicts = "*.c";
```

#### `VersionCap` — Limit the number of stored versions

```
VersionCap = "*.log" 5 2;
```

Format: `VersionCap = "pattern" <keep-count> <keep-count-if-deleted>;`

- `keep-count`: Maximum number of versions to retain while the file exists.
- `keep-count-if-deleted`: Maximum number of versions to retain after the file has been deleted from the workspace.

Older versions beyond the cap are purged on the next sync.

#### `PreCheckin` — Run a command before checking in a file

```
PreCheckin = "*.c" ckeyword "$File$ $Revision$ $Date$";
```

Format: `PreCheckin = "pattern" <command> "<parameters>";`

When CMirror detects that a workspace file has changed, it runs `<command>` with the parameters expanded (see [Pre-Checkin Hooks](#pre-checkin-hooks--ckeyword)) before copying the file into the local repository.

### Sync Behaviour

| Key | Type | Default | Description |
|---|---|---|---|
| `WritableWorkspace` | bool | `true` | Whether CMirror is allowed to modify files in the workspace (e.g. pull new versions from central into workspace) |
| `SyncWorkspace` | bool | `true` | Whether to run Phase 1 (Workspace ↔ Local) |
| `SyncCentral` | bool | `true` | Whether to run Phase 2 (Local ↔ Central) |
| `ContinueAlways` | bool | `false` | Skip all confirmation prompts; execute everything automatically |

### Diff Methods

CMirror supports four strategies for determining whether two copies of a file are identical. The default is `TimestampRepresentationTolerantStamp`.

| Value | Description |
|---|---|
| `ByteByByte` | Full binary content comparison (slow but definitive) |
| `ModificationStamp` | Compare Windows `FILETIME` modification timestamps exactly |
| `TimestampRepresentationTolerantStamp` | Compare timestamps with tolerance for FAT/NTFS precision differences *(default)* |
| `TRTSWithByteFallback` | Tolerant timestamp comparison; falls back to byte-by-byte if timestamps match but are suspicious |

Configure per sync phase:

```
WorkspaceToLocalDiff = TimestampRepresentationTolerantStamp;
LocalToCentralDiff   = ByteByByte;
```

### Output Options

| Key | Type | Default | Description |
|---|---|---|---|
| `SyncPrintThreshold` | integer | `256` | Print individual file operations only when the total count is below this threshold |
| `PrintReason` | bool | `true` | Include the reason code alongside each printed operation |
| `SummarizeSync` | bool | `true` | Print a directory-level summary of sync operations |
| `SummarizeSyncIfNotPrinted` | bool | `true` | Show the summary even if individual operations were suppressed by the threshold |
| `SummarizeDirs` | bool | `true` | Include per-directory breakdown in the summary |
| `SummarizeDirsIfNotPrinted` | bool | `true` | Show the directory breakdown even when individual operations are suppressed |
| `SuppressIgnores` | bool | `false` | Do not print a message for each ignored file |
| `SummarizeIgnores` | bool | `false` | Print a single count of ignored files instead of per-file messages |
| `LineLength` | integer | `0` | Wrap long output lines at this column (0 = no wrapping) |

### Custom Commands

For advanced setups where filesystem copy and delete are not sufficient (e.g. copy-on-write volumes, quota enforcement):

| Key | Type | Description |
|---|---|---|
| `CopyCommand` | string | Shell command to use instead of the built-in file copy |
| `DeleteCommand` | string | Shell command to use instead of the built-in file delete |

---

## Sync Modes

CMirror supports four operating modes. The mode is set on the command line rather than in the config file (the internal `MirrorMode` enum is not yet exposed as a config key in v1.3c).

| Mode | Description |
|---|---|
| `FullSync` *(default)* | Full three-way synchronisation: Workspace ↔ Local, then Local ↔ Central |
| `LocalSync` | Sync Local ↔ Central only (skip the workspace phase) |
| `SidewaysSync` | Sync Workspace ↔ Central only (bypass local) |
| `Checkpoint` | Create a new version checkpoint without syncing to central |

---

## Conflict Handling

A **conflict** occurs when a file has been modified in both the local repository (by you) and the central repository (by someone else) since the last sync.

By default (`HaltOnConflicts`) CMirror stops and prompts:

```
Conflict detected in "src/engine.c". Continue? (y/n/a)
```

- `y` — resolve this one conflict and continue
- `n` — abort the sync
- `a` — resolve all conflicts automatically for the rest of this run

To suppress prompts entirely for specific file patterns, use `IgnoreConflicts` in the config file.

---

## Pre-Checkin Hooks & ckeyword

The `PreCheckin` rule lets you run an arbitrary command on a file before CMirror checks it into the local repository. The command is invoked as:

```
<command> <parameters> <workspace-file> <repo-relative-name> <version-number>
```

The canonical use case is **keyword substitution** — embedding metadata like filename, date, and revision number into source file comments. The included `ckeyword` tool handles this:

```
PreCheckin = "*.c;*.h" ckeyword "$File$ $Revision$ $Date$";
```

### ckeyword keywords

`ckeyword` scans the file for the following placeholder strings and replaces them with live values:

| Keyword | Replaced with |
|---|---|
| `$File$` | Repository-relative path of the file |
| `$Revision$` | Version number being checked in |
| `$Date$` | File modification date/time (from `stat`) |
| `$Notice$` | Copyright/notice text (passed via config parameters) |

`ckeyword` automatically detects binary files (by looking for low control characters) and skips them without modification.

---

## Performance: Central Cache

Scanning a large central repository over a slow network share on every run can be slow. The `CentralCache` option writes the central directory listing to a local file so subsequent runs can load it without a full scan:

```
CentralCache = "C:/local_cache/central.cache";
```

CMirror automatically adds an `Ignore` rule for the cache file itself so it is never synced into the repository.

The cache is invalidated and rebuilt automatically whenever CMirror modifies the central repository.

---

## Command-Line Reference

```
cmirror [<config-file>] [-config "<key = value; ...>"]
```

| Argument | Description |
|---|---|
| `<config-file>` | Explicit path to a `cmirror.config` file. Overrides the automatic upward search. |
| `-config "..."` | Inline configuration statements (applied after the config file). Multiple `-config` flags are allowed. |

**Examples:**

```bat
rem Use default config discovery
cmirror

rem Use a specific config file
cmirror C:\projects\myproject\cmirror.config

rem Override the workspace path on the command line
cmirror -config "Workspace = 'C:/dev/myproject';"

rem Skip central sync (workspace-to-local only)
cmirror -config "SyncCentral = false;"
```

---

## Example Configuration

```
// cmirror.config — example project configuration

// Repository paths
Workspace = "C:/dev/myproject";
Local     = "C:/local_repo/myproject";
Central   = "//fileserver/repos/myproject";

// Cache the central directory listing locally for faster scans
CentralCache = "C:/local_cache/myproject_central.cache";

// Retry network access up to 3 times before giving up
CentralRetryTimes = 3;

// Ignore build artefacts
Ignore = "*.obj;*.pdb;*.ilk;*.exe;*.dll;*.lib";
Ignore = "build/";

// Keyword substitution for source files
PreCheckin = "*.c;*.cpp;*.h" ckeyword "$File$ $Revision$ $Date$";

// Keep at most 20 versions of log files; keep only 2 after deletion
VersionCap = "*.log" 20 2;

// Generated headers may conflict safely; don't halt on them
IgnoreConflicts = "generated/*.h";

// Tolerant timestamp diff workspace→local; byte-for-byte local→central
WorkspaceToLocalDiff = TimestampRepresentationTolerantStamp;
LocalToCentralDiff   = ByteByByte;

// Skip confirmation prompts in CI
// ContinueAlways = true;
```

---

## License

Public domain. See [LICENSE](LICENSE) for details. The authors make no warranty — use at your own risk.

For the latest version or source code, see http://www.mollyrocket.com/tools
