# CMirror — Architecture & Developer Guide

This document describes the internal design of `cmirror.cpp` for developers who want to understand, debug, or extend the codebase.

---

## Table of Contents

- [Source Files](#source-files)
- [High-Level Data Flow](#high-level-data-flow)
- [Core Data Structures](#core-data-structures)
  - [Repositories and file_type](#repositories-and-file_type)
  - [file_version](#file_version)
  - [file_record](#file_record)
  - [directory_entry and directory_contents](#directory_entry-and-directory_contents)
  - [file_rule and file_rule_list](#file_rule-and-file_rule_list)
  - [mirror_config](#mirror_config)
  - [sync_operation and sync_context](#sync_operation-and-sync_context)
- [Configuration Parsing](#configuration-parsing)
- [Directory Scanning](#directory-scanning)
  - [Versioned filename format](#versioned-filename-format)
  - [Cache support](#cache-support)
- [Synchronization Logic](#synchronization-logic)
  - [Phase 1 — Workspace ↔ Local](#phase-1--workspace--local)
  - [Phase 2 — Local ↔ Central](#phase-2--local--central)
  - [Sync reasons](#sync-reasons)
  - [Operation types](#operation-types)
- [Operation Execution](#operation-execution)
- [File Comparison (Diff Methods)](#file-comparison-diff-methods)
- [File Rule Evaluation](#file-rule-evaluation)
- [Safety Architecture](#safety-architecture)
- [Pre-Checkin Hooks](#pre-checkin-hooks)
- [ckeyword.cpp](#ckeywordcpp)
- [string_table Memory Model](#string_table-memory-model)
- [stb.h](#stbh)
- [Source Self-Check (CheckCmirrorSource)](#source-self-check-checkcirrorsource)

---

## Source Files

| File | Purpose |
|---|---|
| `cmirror.cpp` | Entire synchronization engine (~3 900 lines) |
| `ckeyword.cpp` | Stand-alone keyword substitution utility (~380 lines) |
| `stb.h` | Sean Barrett's single-header C utility library (BST, arrays, string helpers) |

The project intentionally has no build system. Both `.cpp` files are compiled independently.

---

## High-Level Data Flow

```
main()
 │
 ├─ ProcessCommandLine()         parse argv (first pass: find config file path)
 ├─ FindConfigFile()             walk up directory tree for cmirror.config
 ├─ ParseConfigFile()            tokenize & interpret config → mirror_config
 ├─ ProcessCommandLine()         parse argv again (second pass: apply -config overrides)
 ├─ BuildDirectory()             scan workspace + local + central → directory_contents BST
 │    ├─ BuildDirectoryFromCache()   (central only, when CentralCache is set)
 │    └─ BuildDirectoryRecursive()  (Win32 FindFirstFile/FindNextFile walk)
 │         └─ ProcessFilename()     decode versioned filename → insert into BST
 │
 ├─ Synchronize()
 │    ├─ [Phase 1] SynchronizeLocalAndWorkspace()   → queue sync_operations
 │    │    ├─ PromptSyncContext()                   ask user y/n/a
 │    │    └─ ExecuteSyncContext()                  run operations
 │    └─ [Phase 2] SynchronizeLocalAndCentral()    → queue sync_operations
 │         ├─ PromptSyncContext()
 │         └─ ExecuteSyncContext()
 │
 └─ WriteDirectoryToCache()      (if CentralCache configured)
```

---

## Core Data Structures

### Repositories and `file_type`

```c
enum file_type {
    NullFileType,
    WorkspaceFileType,   // developer's working directory
    LocalFileType,       // local versioned repository
    CentralFileType,     // central/server versioned repository
    OnePastLastFileType  // sentinel for array sizing
};
```

Most arrays and two-dimensional tables in the codebase are indexed by `file_type`. The `NullFileType` slot is intentionally unused; valid indices are 1–3.

---

### `file_version`

Represents one versioned copy of a file in a repository:

```c
struct file_version {
    int          VersionNumber;      // monotonically increasing integer
    bool         Deleted;            // true if this is a "deleted" marker
    unsigned int dwHighDateTime;     // Win32 FILETIME (high word)
    unsigned int dwLowDateTime;      // Win32 FILETIME (low word)
    bool         OldStyle;           // true for legacy ",NNN" naming
};
```

`file_version` nodes are stored in a BST keyed on `VersionNumber` using the `stb_bst_parent` macro (from `stb.h`). The tree type is `file_versions` and the generated accessor prefix is `Version`.

---

### `file_record`

Tracks all known versions of one logical file within one repository:

```c
struct file_record {
    bool          Present;      // at least one version exists in this repo
    bool          Deleted;      // the most recent version is a deletion marker
    int           MinVersion;   // lowest version number seen
    int           MaxVersion;   // highest version number seen (= "current")
    file_versions Versions;     // BST of file_version nodes
};
```

`Present` and `Deleted` are derived from the version tree during the scan and used as quick checks by the sync logic.

---

### `directory_entry` and `directory_contents`

```c
struct directory_entry {
    char        *Name;                       // logical filename (no version infix)
    file_record  File[OnePastLastFileType];  // one record per repository
};
```

`directory_entry` nodes form a BST keyed on `Name` (case-insensitive). The tree type is `directory_contents` with accessor prefix `Dir`.

The entire state of all three repositories is represented as a single `directory_contents` tree rooted at `RootContents`. Every logical file that appears in *any* repository (or the workspace) has exactly one `directory_entry`.

---

### `file_rule` and `file_rule_list`

```c
struct file_rule {
    char             *WildCard;                    // glob pattern
    int               WildCardLength;
    conflict_behavior Behavior;                    // HaltOnConflicts | IgnoreConflicts
    ignore_behavior   Ignore;                      // NoIgnoreEffect | DoIgnore | DoNotIgnore
    bool              CapVersionCount;
    int               MaxVersionCount;             // max live versions to keep
    int               MaxVersionCountIfDeleted;    // max versions to keep after deletion
    char             *PreCheckinCommand;           // executable to invoke
    char             *PreCheckinCommandParameters; // parameter template
};

typedef file_rule* file_rule_list;  // stb dynamic array
```

Rules are stored in declaration order (a `stb` dynamic array). The `Accept()` function walks the rule list in order for a given filename, applying the last matching rule for each attribute (ignore behavior, conflict behavior, etc.).

Semicolon-separated wildcard lists in a single config statement are expanded by `ExpandRule()` / `MakeIgnoreRule()` into individual `file_rule` entries.

---

### `mirror_config`

The global configuration object built by `ParseConfigFile()` and `ProcessCommandLine()`:

```c
struct mirror_config {
    file_rule_list  FileRuleList;
    mirror_mode     MirrorMode;
    diff_method     DiffMethod[OnePastLastFileType][OnePastLastFileType];
    char           *Directory[OnePastLastFileType];
    char           *CentralCache;
    char           *CopyCommand;
    char           *DeleteCommand;
    bool            WritableWorkspace;
    bool            ContinueAlways;
    bool            SyncWorkspace;
    bool            SyncCentral;
    int             RetryTimes[OnePastLastFileType];
    bool            SuppressIgnores;
    bool            SummarizeIgnores;
    int             SyncPrintThreshold;
    bool            PrintReason;
    bool            SummarizeSync;
    bool            SummarizeSyncIfNotPrinted;
    bool            SummarizeDirs;
    bool            SummarizeDirsIfNotPrinted;
    int             LineLength;
};
```

`DiffMethod` is a symmetric 4×4 matrix; only the two meaningful pairs `[Workspace][Local]` and `[Local][Central]` are set by the config parser.

---

### `sync_operation` and `sync_context`

```c
struct sync_operation {
    sync_operation_type OperationType; // VersionCopy | VersionDelete | VersionMove
                                       // | MarkAsDeleted | PreCheckin
    sync_reason         Reason;        // why this operation was queued (12 values)
    directory_entry    *Entry;         // the logical file being acted on
    file_type           EntryAType;    // source repository
    file_type           EntryBType;    // destination repository (or same as A)
    int                 VersionIndex;  // version number being acted on
    bool                Overwrite;     // whether the destination may be overwritten
    char               *Command;       // PreCheckin: executable
    char               *Parameters;   // PreCheckin: parameter template
};

struct sync_context {
    mirror_config  *Config;
    bool            ContinueAlways;
    sync_operation *Operations;    // stb dynamic array of queued operations
};
```

The synchronization functions never touch the filesystem directly. They build a list of `sync_operation` objects in a `sync_context`. `PromptSyncContext()` prints the list and asks the user to confirm. `ExecuteSyncContext()` then executes every operation by calling `Execute()`.

---

## Configuration Parsing

The config parser (`ParseConfigFile()`) is a hand-written recursive descent tokenizer. There is no external grammar file.

**Token types:** `Identifier`, `Equals`, `String`, `Integer`, `SemiColon`, `EndOfStream`, `Unrecognized`

**Lexer (`GetToken()`):**
- Skips whitespace and `//` line comments
- Handles single- and double-quoted strings (via `ParseString()`)
- Reads identifiers (alpha-only start, alphanumeric continuation)
- Reads integers with `atoi`

**Parser:** Processes one `Identifier = Value;` statement at a time. Unrecognized keys produce a warning but do not abort parsing. Malformed lines are skipped to the next line via `SkipToNextLine()`.

`TrueOrFalse()` accepts `true`/`yes`/`1` and `false`/`no`/`0` (case-insensitive).

The same `ParseConfigFile()` function is called for inline `-config "..."` command-line arguments by re-using a `stream` struct pointed at the argument string.

---

## Directory Scanning

### Versioned filename format

Repository filenames encode version information using a `,cm` infix inserted immediately before the last `.` extension separator:

```
<basename>,cm<version><.ext>       — live version
<basename>,cmd<version><.ext>      — deleted marker version
```

Where `<version>` is a zero-padded 8-digit decimal integer (`%08d` for versions > 9999, `%04d` otherwise — legacy files may have shorter padding).

Files without the `,cm` infix in local/central are ignored (workspace files never have a version infix).

`ProcessFilename()` decodes this format:

1. Finds the last `,` in the filename.
2. Checks for `cm` prefix → new-style naming.
3. Checks for `d` prefix after `cm` → sets `Deleted = true`.
4. Reads the version integer with `atoi`.
5. Reconstructs the logical filename by splicing the extension over the `,cm…` segment.
6. Inserts or updates the `directory_entry` in the BST.

Old-style files (comma followed directly by a digit, no `cm`) are also recognized and flagged with `OldStyle = true`.

### Cache support

`WriteDirectoryToCache()` serializes the `directory_contents` BST for one repository to a binary flat file. `BuildDirectoryFromCache()` reads it back, bypassing the Win32 directory walk.

The cache is only used for the **central** repository. It is always rebuilt after CMirror makes any change to central. The cache file is automatically ignored by adding an `Ignore` rule in `InitializeCacheIgnore()`.

---

## Synchronization Logic

### Phase 1 — Workspace ↔ Local

`SynchronizeLocalAndWorkspace()` iterates every `directory_entry` in the BST and classifies the relationship between the workspace file and the local repository:

| Situation | Action queued |
|---|---|
| File is new in workspace (not in local) | `VersionCopyOperation` workspace → local (reason: `WorkspaceNewReason`) |
| File changed in workspace vs. latest local | `PreCheckinOperation` (if rule), then `VersionCopyOperation` workspace → local (`WorkspaceChangedReason`) |
| File deleted from workspace | `VersionCopyOperation` marking deletion in local (`WorkspaceDeletedReason`) |
| Local has a newer version than workspace | `VersionCopyOperation` local → workspace (`LocalNonLatestReason`) |
| Only local has the file (workspace missing) | Copy latest local version to workspace (`LocalOnlyLatestReason`) |
| Version cap exceeded | `VersionDeleteOperation` oldest versions (`VersionCapReason`) |

"Changed" is determined by `AreEquivalent()` using the configured `DiffMethod[Workspace][Local]`.

### Phase 2 — Local ↔ Central

`SynchronizeLocalAndCentral()` compares local and central per-file:

| Situation | Action queued |
|---|---|
| Local has versions central doesn't | Copy missing local versions to central |
| Central has versions local doesn't | Copy missing central versions to local (and to workspace if `WritableWorkspace`) |
| New file only in central | Pull all versions from central into local + workspace (`CentralNewReason`) |
| New file only in local | Push all local versions to central (`LocalOnlyNonLatestReason` / `LocalOnlyLatestReason`) |
| Latest version deleted in central | Propagate deletion to local + workspace (`CentralLatestDeletedReason`) |
| Conflict (both sides have changes since last common ancestor) | Halt or ignore per rule (`LocalCentralConflictIgnoreReason`) |
| Version cap exceeded | Purge oldest versions (`VersionCapReason`) |

### Sync reasons

The `sync_reason` enum has 12 values. Each operation records *why* it was enqueued so that `Print()` can display a human-readable explanation:

```
PreCheckinReason
WorkspaceNewReason
WorkspaceChangedReason
WorkspaceDeletedReason
LocalNonLatestReason
LocalOnlyNonLatestReason
LocalOnlyLatestReason
LocalCentralConflictIgnoreReason
VersionCapReason
CentralNewReason
CentralOnlyLatestReason
CentralLatestDeletedReason
```

### Operation types

```
VersionCopyOperation    — copy a file from one repo to another
VersionDeleteOperation  — delete a specific versioned file
VersionMoveOperation    — rename/move a versioned file (e.g. for old-style migration)
MarkAsDeletedOperation  — write a deletion-marker version without removing live files
PreCheckinOperation     — invoke an external command on a workspace file
```

---

## Operation Execution

`ExecuteSyncContext()` calls `Execute()` for each `sync_operation` in order.

`Execute()` switches on `OperationType`:

- **VersionCopyOperation:** calls `CopyFileWrapped()` which in turn calls `Win32CopyFile()`. Creates any missing intermediate directories with `EnsureDirectoryExistsForFile()`. On failure may retry (for network paths).
- **VersionDeleteOperation:** calls `DeleteFileWrapped()` which calls `DeleteFileLowLevel()`. If deletion fails (e.g. file locked), the file is moved to the Windows temp directory as a fallback.
- **VersionMoveOperation:** calls `MoveFileWrapped()`.
- **MarkAsDeletedOperation:** copies the workspace file to a `cmd`-prefixed version in the repository.
- **PreCheckinOperation:** calls `ExecuteSystemCommand()` → Win32 `CreateProcess`.

---

## File Comparison (Diff Methods)

`AreEquivalent(a, b)` delegates to `AreEquivalentMethod(a, b, method)`:

| `diff_method` value | Implementation |
|---|---|
| `ByteByByteDiff` | `FilesAreEquivalentByteForByte()` — opens both files, reads in 64 KB blocks, compares with `memcmp` |
| `ModificationStampDiff` | `FilesHaveEquivalentModificationStamps()` — compares `FILETIME` from `GetFileAttributesEx` exactly |
| `TimestampRepresentationTolerantStampDiff` | Same as above but allows a 2-second tolerance (handles FAT-to-NTFS timestamp rounding) |
| `TRTSWithByteFallbackDiff` | Tolerant timestamp first; if timestamps differ by exactly the tolerance boundary, falls back to byte comparison |

The default is `TimestampRepresentationTolerantStampDiff` for both phase pairs.

---

## File Rule Evaluation

`Accept(RuleList, Name)` determines whether a file should be included in synchronization:

1. Starts with `included = true`.
2. Walks `FileRuleList` in order.
3. For each rule where `WildCardMatch(Name, rule.WildCard)` is true:
   - If `rule.Ignore == DoIgnore` → set `included = false`
   - If `rule.Ignore == DoNotIgnore` → set `included = true`
4. Returns `included`.

`WildCardMatch()` is a simple recursive glob matcher supporting `*` (any sequence of characters) and `?` (any single character).

For conflict behavior and version capping, the sync logic calls `UpdateRuleFor()` to find the *last* matching rule for the specific attribute, rather than the first. This means later rules in the config file take precedence.

---

## Safety Architecture

A deliberate safety mechanism groups all raw filesystem mutation calls (`MoveFile`, `DeleteFile`, `CopyFile`) in the first ~250 lines of `cmirror.cpp`, wrapped in `...LowLevel()` and `...Wrapped()` helper functions. Everything past the `// EOLLFO` comment is forbidden from calling the raw Win32 functions directly.

`CheckCmirrorSource()` (called at startup) reads `cmirror.cpp` itself from disk, finds the `// EOLLFO` marker, and scans the text after it for bare calls to `MoveFile`, `DeleteFile`, `CopyFile`, `remove`, and `rename`. If any are found, it prints a warning to stdout. This is a developer-time consistency check, not a security check.

The wrapper functions add:
- **Repository validation:** verify the file path is inside an expected repository directory before mutating it.
- **Filename validation:** check that local/central filenames contain the `,cm` infix (workspace files never should).
- **Fallback on failure:** failed deletes are relocated to `%TEMP%` instead of leaving locked files in the repository.
- **Interactive confirmation:** for unusual but possibly legitimate operations, `ShouldContinue()` prompts the user.

---

## Pre-Checkin Hooks

When `SynchronizeLocalAndWorkspace()` detects a changed or new workspace file and a `PreCheckinCommand` rule matches, it queues a `PreCheckinOperation` *before* the copy operation.

`Execute()` invokes the command as:

```
<command> <parameters> <workspace-file-path> <repo-relative-name> <version-number>
```

The process is run with `CreateProcess` (no shell). CMirror waits for it to complete before proceeding. The command is expected to modify the workspace file in place (e.g. expand keywords), and CMirror then copies the modified file into the local repository.

---

## ckeyword.cpp

`ckeyword` is a stand-alone C program (~380 lines) for keyword substitution. It is invoked by CMirror as a pre-checkin hook.

**Command line:**
```
ckeyword [keyword-list] <source-file> <repo-relative-name> <version-number>
```

`keyword-list` is a space-separated string from the `PreCheckin` parameters field. It controls which keywords are active (e.g. `"$File$ $Revision$ $Date$"`).

**Processing model:**

`keyword_context` holds I/O state. The main loop reads the source file one character at a time via `Input()` and writes to a temporary output file via `Output()`. When a `$` is detected, the code attempts to match one of the four keyword strings character by character. On a full match the keyword is replaced with the live value; on a partial match the characters buffered so far are flushed as literals.

The source file is overwritten atomically: `ckeyword` writes to a temp file then renames it over the original.

**Binary detection:** Any byte in the range 0x00–0x09 or 0x10–0x1F triggers `SuspectedBinary = true`, causing the file to be passed through unchanged.

**Typo detection:** If `$` is followed by text that looks like a keyword start but does not match any known keyword, `TypoSuspected` is set and a warning is printed.

---

## string_table Memory Model

CMirror uses a simple arena allocator (`string_table`) for all string data parsed from the config file and directory scans. Strings are written sequentially into a large fixed buffer (`StoreCurrent` pointer). They are never freed individually — the entire arena is valid for the lifetime of the program.

This means `mirror_config`, `directory_entry`, and `file_rule` members that are `char*` point directly into the arena and must not be `free()`d.

---

## stb.h

`stb.h` is Sean Barrett's single-header utility library. CMirror uses the following facilities:

| Macro / function | Used for |
|---|---|
| `stb_bst_parent` | Declare intrusive BST types (`file_versions`, `directory_contents`) |
| `stb_arr_*` | Dynamic arrays for `file_rule_list` and `sync_operation` arrays |
| `stb_filec()` | Read an entire file into a malloc'd buffer |
| `stb_tokens()` | Split semicolon-separated wildcard strings |
| `stb_prefix()` / `stb_strichr()` | String utilities in `CheckCmirrorSource` / `ShouldContinue` |

`#define STB_DEFINE` is set once in `cmirror.cpp` before including the header to emit the implementation.

---

## Source Self-Check (`CheckCmirrorSource`)

```c
void CheckCmirrorSource(void);
```

Called from `main()` before any sync work begins. Reads `cmirror.cpp` from the current directory (silently skips if not found), locates the `// EOLLFO` sentinel comment, then searches the text after it for unwrapped calls to `MoveFile`, `DeleteFile`, `CopyFile`, `remove`, and `rename`. A warning is printed for each violation found.

This is a "belt and suspenders" developer guard: it catches the case where someone adds a file operation below the safety boundary without going through the wrapped helpers. It is not a cryptographic or security check — it is a simple `strstr` scan, so it does not understand C syntax and will also trigger on comments or strings.
