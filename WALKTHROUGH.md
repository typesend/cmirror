# CMirror Collaborative Workflow Walkthrough

This document follows **Alice**, a renderer engineer, through a real week of work on a game engine project called **Nova**. She is collaborating with three other developers:

- **Bob** — audio system
- **Carol** — input and config
- **Dave** — art and assets

The team uses a Windows file server at `\\buildbox` as their central repository. Each developer has a local copy on their own machine and a workspace they edit in.

---

## The Setup

Alice's `cmirror.config` lives at the root of her workspace and is itself tracked in the central repo (so everyone shares the same config):

```
// cmirror.config — Nova project

Workspace = "C:/dev/nova";
Local     = "C:/nova_local";
Central   = "//buildbox/repos/nova";

CentralCache      = "C:/nova_cache/central.cache";
CentralRetryTimes = 3;

Ignore = "*.obj;*.pdb;*.ilk;*.exe;*.dll;*.lib";
Ignore = "build/";

PreCheckin = "*.c;*.cpp;*.h" ckeyword "$File$ $Revision$ $Date$";

WorkspaceToLocalDiff = TimestampRepresentationTolerantStamp;
LocalToCentralDiff   = ByteByByte;
```

---

## Scene 1 — Alice Joins the Project

It's Monday morning. Alice has just been added to the team. She has cloned the `cmirror.config` from the file server manually, set up her directories, and is now running CMirror for the first time. Her workspace and local repo are completely empty; the central repo already has five source files and some assets checked in by Bob, Carol, and Dave over the past two weeks.

She opens a command prompt, navigates to her workspace, and runs:

```
C:\dev\nova> cmirror
```

```
CMirror v1.3c by Casey Muratori, Sean Barrett, and Jeff Roberts
NO WARRANTY IS EXPRESSED OR IMPLIED. USE AT YOUR OWN RISK.
This program and its source code are in public domain. For the latest
version or the source code, see http://www.mollyrocket.com/tools

Checked cmirror.cpp successfully.

Looking for config file at "C:/dev/nova/cmirror.config"...
Configuration:
  Workspace: "C:/dev/nova" (writable)
  Central: "//buildbox/repos/nova"
  Central cache: "C:/nova_cache/central.cache"
  Local: "C:/nova_local"
  Workspace <-> Local: "TimestampRepresentationTolerantStamp"
  Local <-> Central: "ByteByByte"
  File rules:
    Ignore - "*.obj"
    Ignore - "*.pdb"
    Ignore - "*.ilk"
    Ignore - "*.exe"
    Ignore - "*.dll"
    Ignore - "*.lib"
    Ignore - "build/"
    PreCheck[ckeyword] - "*.c"
    PreCheck[ckeyword] - "*.cpp"
    PreCheck[ckeyword] - "*.h"

Building directory structure:
  Workspace: 0
  Local: 0
  Central: 11

Synchronizing:
  Workspace <-> Local: 11
```

Because her workspace is empty there is nothing to push; **Phase 1 produces no operations**. CMirror skips the prompt and prints nothing for that phase, then moves straight to Phase 2.

```
  Workspace <-> Local: 11

CA  copy: //buildbox/repos/nova/assets/sprites/hero,cm00000001.png
       -> C:/nova_local/assets/sprites/hero,cm00000001.png
CA  copy: //buildbox/repos/nova/assets/sprites/hero,cm00000001.png
       -> C:/dev/nova/assets/sprites/hero.png
CA  copy: //buildbox/repos/nova/assets/sprites/old_hero,cm00000001.png
       -> C:/nova_local/assets/sprites/old_hero,cm00000001.png
CA  copy: //buildbox/repos/nova/assets/sprites/old_hero,cm00000001.png
       -> C:/dev/nova/assets/sprites/old_hero.png
CA  copy: //buildbox/repos/nova/build,cm00000002.bat
       -> C:/nova_local/build,cm00000002.bat
CA  copy: //buildbox/repos/nova/build,cm00000001.bat
       -> C:/nova_local/build,cm00000001.bat
CA  copy: //buildbox/repos/nova/build,cm00000002.bat
       -> C:/dev/nova/build.bat
CA  copy: //buildbox/repos/nova/src/audio,cm00000002.c
       -> C:/nova_local/src/audio,cm00000002.c
CA  copy: //buildbox/repos/nova/src/audio,cm00000001.c
       -> C:/nova_local/src/audio,cm00000001.c
CA  copy: //buildbox/repos/nova/src/audio,cm00000002.c
       -> C:/dev/nova/src/audio.c
CA  copy: //buildbox/repos/nova/cmirror,cm00000001.config
       -> C:/nova_local/cmirror,cm00000001.config
CA  copy: //buildbox/repos/nova/cmirror,cm00000001.config
       -> C:/dev/nova/cmirror.config
CA  copy: //buildbox/repos/nova/src/engine,cm00000003.c
       -> C:/nova_local/src/engine,cm00000003.c
CA  copy: //buildbox/repos/nova/src/engine,cm00000002.c
       -> C:/nova_local/src/engine,cm00000002.c
CA  copy: //buildbox/repos/nova/src/engine,cm00000001.c
       -> C:/nova_local/src/engine,cm00000001.c
CA  copy: //buildbox/repos/nova/src/engine,cm00000003.c
       -> C:/dev/nova/src/engine.c
CA  copy: //buildbox/repos/nova/src/renderer,cm00000005.c
       -> C:/nova_local/src/renderer,cm00000005.c
CA  copy: //buildbox/repos/nova/src/renderer,cm00000004.c
       -> C:/nova_local/src/renderer,cm00000004.c
CA  copy: //buildbox/repos/nova/src/renderer,cm00000003.c
       -> C:/nova_local/src/renderer,cm00000003.c
CA  copy: //buildbox/repos/nova/src/renderer,cm00000002.c
       -> C:/nova_local/src/renderer,cm00000002.c
CA  copy: //buildbox/repos/nova/src/renderer,cm00000001.c
       -> C:/nova_local/src/renderer,cm00000001.c
CA  copy: //buildbox/repos/nova/src/renderer,cm00000005.c
       -> C:/dev/nova/src/renderer.c
CA  copy: //buildbox/repos/nova/src/config,cm00000004.h
       -> C:/nova_local/src/config,cm00000004.h
CA  copy: //buildbox/repos/nova/src/config,cm00000003.h
       -> C:/nova_local/src/config,cm00000003.h
CA  copy: //buildbox/repos/nova/src/config,cm00000002.h
       -> C:/nova_local/src/config,cm00000002.h
CA  copy: //buildbox/repos/nova/src/config,cm00000001.h
       -> C:/nova_local/src/config,cm00000001.h
CA  copy: //buildbox/repos/nova/src/config,cm00000004.h
       -> C:/dev/nova/src/config.h

Continue?  (y/n/a)
```

Alice types `y`. CMirror starts executing:

```
  Executing: 27 / 27

```

Her workspace now looks exactly like what the rest of the team has been working with. She has every historical version stored locally and the latest of each file in her workspace directory.

> **What the `CA` prefix means:** `CA` = *Central Added* — a file (or version) that exists only in central and needs to be propagated to local and workspace. CMirror copies all historical versions into local so Alice can roll back, and puts only the latest version in her workspace as a plain file.

---

## Scene 2 — Alice Edits the Renderer

It's Tuesday afternoon. Alice has spent the morning reading the code and has now implemented a new depth-sorting pass in `src/renderer.c`. She saves the file and runs CMirror to check in her work:

```
C:\dev\nova> cmirror
```

```
CMirror v1.3c by Casey Muratori, Sean Barrett, and Jeff Roberts
NO WARRANTY IS EXPRESSED OR IMPLIED. USE AT YOUR OWN RISK.
...

Looking for config file at "C:/dev/nova/cmirror.config"...
Configuration:
  Workspace: "C:/dev/nova" (writable)
  Central: "//buildbox/repos/nova"
  Central cache: "C:/nova_cache/central.cache"
  Local: "C:/nova_local"
  Workspace <-> Local: "TimestampRepresentationTolerantStamp"
  Local <-> Central: "ByteByByte"
  File rules:
    ...

Building directory structure:
  Workspace: 6
  Local: 23
  Central: 23    (using compressed directory cache)

Synchronizing:
  Workspace <-> Local: 6
```

CMirror's timestamp comparison detects that `renderer.c` in Alice's workspace is newer than version 5 in her local repo. It queues the pre-checkin hook first, then a copy:

```
    pre-checkin: ckeyword $File$ $Revision$ $Date$ C:/dev/nova/src/renderer.c
wc  copy: C:/dev/nova/src/renderer.c
       -> C:/nova_local/src/renderer,cm00000006.c

Continue?  (y/n/a)
```

Alice types `y`:

```
  Executing: 2 / 2
```

What just happened behind the scenes:

1. **`ckeyword`** ran on `C:/dev/nova/src/renderer.c` and expanded any `$File$`, `$Revision$`, and `$Date$` keywords she had in her comments — for example, a line like `/* $Revision$ */` became `/* 6 */`.
2. The modified workspace file was then **copied to** `C:/nova_local/src/renderer,cm00000006.c`.

Now Phase 2 pushes her new version to the server:

```
ln  copy: C:/nova_local/src/renderer,cm00000006.c
       -> //buildbox/repos/nova/src/renderer,cm00000006.c

Continue?  (y/n/a)
```

> **What `ln` means:** `ln` = *Local only, newest version* — the local repo has a version that central does not yet have. CMirror pushes it up.

Alice types `y`:

```
  Executing: 1 / 1

```

Done. Version 6 of `renderer.c` is now in central. Bob, Carol, and Dave will get it the next time they run CMirror.

---

## Scene 3 — Alice Pulls in Her Teammates' Changes

It's Wednesday morning. While Alice was working on the renderer yesterday, Bob pushed a new version of `audio.c` (v3) and also added a new file, `audio_utils.c` (v1). Carol bumped `config.h` to v5 with a new `MAX_ENTITIES` constant.

Alice runs CMirror again before starting her day's work:

```
C:\dev\nova> cmirror
```

```
CMirror v1.3c by Casey Muratori, Sean Barrett, and Jeff Roberts
...

Building directory structure:
  Workspace: 6
  Local: 24
  Central: 28    (using compressed directory cache)

Synchronizing:
  Workspace <-> Local: 7
```

Alice hasn't touched anything overnight, so Phase 1 produces no operations. CMirror moves straight to Phase 2:

```
CA  copy: //buildbox/repos/nova/src/audio_utils,cm00000001.c
       -> C:/nova_local/src/audio_utils,cm00000001.c
CA  copy: //buildbox/repos/nova/src/audio_utils,cm00000001.c
       -> C:/dev/nova/src/audio_utils.c
CC  copy: //buildbox/repos/nova/src/audio,cm00000003.c
       -> C:/nova_local/src/audio,cm00000003.c
CC  copy: //buildbox/repos/nova/src/audio,cm00000003.c
       -> C:/dev/nova/src/audio.c
CC  copy: //buildbox/repos/nova/src/config,cm00000005.h
       -> C:/nova_local/src/config,cm00000005.h
CC  copy: //buildbox/repos/nova/src/config,cm00000005.h
       -> C:/dev/nova/src/config.h

Continue?  (y/n/a)
```

> **`CA` vs `CC`:** `CA` (*Central Added*) means the file didn't exist in local at all — it's brand new from central. `CC` (*Central only, newest version*) means the file already existed in local but central has a newer version that local is missing.

Alice types `y`:

```
  Executing: 6 / 6

```

When she opens her editor she finds `src/audio_utils.c` is now present, `src/audio.c` is Bob's latest version, and `src/config.h` already has Carol's new `MAX_ENTITIES` constant. She's caught up with no manual steps.

---

## Scene 4 — A Conflict

It's Thursday. Alice needed to add a `RENDERER_MAX_LAYERS` constant to `config.h` for the depth-sorting pass she shipped on Tuesday. She edits the file and saves it. Meanwhile — unknown to her — Carol had also been working on `config.h` this morning to add audio buffer size constants, and Carol ran CMirror first, pushing her version as v6 to central.

Alice now runs CMirror:

```
C:\dev\nova> cmirror
```

```
CMirror v1.3c by Casey Muratori, Sean Barrett, and Jeff Roberts
...

Building directory structure:
  Workspace: 6
  Local: 24
  Central: 29    (using compressed directory cache)

Synchronizing:
  Workspace <-> Local: 6
```

Phase 1 sees that `config.h` in Alice's workspace is newer than v5 (the version local knows about). It queues a check-in:

```
    pre-checkin: ckeyword $File$ $Revision$ $Date$ C:/dev/nova/src/config.h
wc  copy: C:/dev/nova/src/config.h
       -> C:/nova_local/src/config,cm00000006.h

Continue?  (y/n/a)
```

Alice types `y`. Her local repo now has a v6 of `config.h` with her `RENDERER_MAX_LAYERS` constant.

```
  Executing: 2 / 2
```

Phase 2 runs. CMirror compares local and central. It sees:

- Local `config.h` max version: **6** (Alice's change)
- Central `config.h` max version: **6** (Carol's change, pushed earlier)

Both sides have a version 6, but when CMirror does its byte-for-byte comparison (`LocalToCentralDiff = ByteByByte`) it finds the files are **different**. That's a conflict — two developers independently created "version 6" with different content.

```
C!  copy: C:/nova_local/src/config,cm00000006.h
       -> //buildbox/repos/nova/src/config,cm00000006.h

conflict between local & central - config.h - local version 6 differs from central version 6
About to copy 'C:/nova_local/src/config,cm00000006.h' over existing '//buildbox/repos/nova/src/config,cm00000006.h'.
Continue?  (y/n/a)
```

Alice types `n`. She does **not** want to blindly overwrite Carol's changes.

```

```

CMirror exits without touching the central file. Alice messages Carol on the team chat: *"Hey, I hit a conflict on config.h — I added RENDERER_MAX_LAYERS, looks like you were in there too. Can you send me your version?"*

Carol pastes her additions. Alice manually merges both sets of constants into her local `config.h`, saves, and runs CMirror again. This time Phase 1 sees the file has changed again (it's newer than local v6) and creates v7:

```
    pre-checkin: ckeyword $File$ $Revision$ $Date$ C:/dev/nova/src/config.h
wc  copy: C:/dev/nova/src/config.h
       -> C:/nova_local/src/config,cm00000007.h

Continue?  (y/n/a)
y
  Executing: 2 / 2

ln  copy: C:/nova_local/src/config,cm00000007.h
       -> //buildbox/repos/nova/src/config,cm00000007.h

Continue?  (y/n/a)
y
  Executing: 1 / 1

```

The merged v7 is now in central. The conflict is resolved. Carol runs CMirror on her machine and automatically gets v7 in her workspace.

---

## Scene 5 — Dave Deletes an Obsolete Asset

It's Friday. Dave has replaced the old placeholder sprite with the final art. He deleted `assets/sprites/old_hero.png` from his workspace and ran CMirror, which pushed a deletion marker to central.

When Alice runs her end-of-week sync:

```
C:\dev\nova> cmirror
```

```
CMirror v1.3c by Casey Muratori, Sean Barrett, and Jeff Roberts
...

Building directory structure:
  Workspace: 7
  Local: 27
  Central: 28    (using compressed directory cache)

Synchronizing:
  Workspace <-> Local: 7
```

Phase 1 is quiet — Alice hasn't edited anything since Thursday's merge.

Phase 2 detects that the latest version of `old_hero.png` in central is a deletion marker (a zero-byte file named `old_hero,cmd00000002.png`), while local still has v1 as a live file:

```
CD  copy: //buildbox/repos/nova/assets/sprites/old_hero,cmd00000002.png
       -> C:/nova_local/assets/sprites/old_hero,cmd00000002.png
CD  copy: //buildbox/repos/nova/assets/sprites/old_hero,cmd00000002.png
       -> C:/dev/nova/assets/sprites/old_hero.png

Continue?  (y/n/a)
```

> **`CD` = Central Deleted.** The newest version in central is a deletion marker. CMirror copies the deletion marker into local (so local's history reflects the deletion) and **overwrites** the live file in the workspace with the zero-byte marker, effectively deleting it from Alice's working directory.

Alice types `y`:

```
  Executing: 2 / 2

```

`old_hero.png` is gone from her workspace and local. The version history in both local and central shows versions 1 (live) and 2 (deleted), so if the team ever needs to retrieve the old sprite they can find it in local or central as `old_hero,cm00000001.png`.

---

## Summary

Here is a quick reference of the reason codes Alice saw during the week and what each means:

| Code | Full name | When it appears |
|---|---|---|
| `CA` | Central Added | A file (or version) exists only in central; pulled into local and workspace |
| `CC` | Central only, Current | Central has a newer version that local is missing; pulled into local and workspace |
| `CD` | Central Deleted | Central's newest version is a deletion marker; propagated to local and workspace |
| `wc` | Workspace Changed | Workspace file is newer than the latest local version; checked into local |
| `WA` | Workspace Added | A new file appeared in the workspace; checked into local for the first time |
| `WD` | Workspace Deleted | A file was removed from the workspace; a deletion marker is created in local |
| `ln` | Local only, newest | Local has a version central doesn't have yet; pushed to central |
| `C!` | Conflict | Both local and central have a version with the same number but different content |

And the operation verbs:

| Verb | Meaning |
|---|---|
| `copy` | Copy a file from one repository to another |
| `delete` | Delete a specific versioned file (usually from a version cap purge) |
| `move` | Rename a legacy-format versioned file to the current format |
| `d-mark` | Write a zero-byte deletion marker (no live file is removed yet) |
| `pre-checkin` | Run the configured hook command on the workspace file before copying |
