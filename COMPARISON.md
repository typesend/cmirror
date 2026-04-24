# CMirror vs. Git

## Where CMirror Wins

**Binary files are first-class.** CMirror treats a PNG and a `.c` file identically — copy it, version it, done. Git technically handles binaries but they bloat the pack files, diffs are useless, and LFS is a whole separate system to set up. For a game team syncing textures, sounds, and meshes alongside code, CMirror just works.

**Network shares are the transport.** It runs entirely over a filesystem — SMB, UNC paths, a NAS, whatever. No server process to configure, no SSH keys, no port forwarding. If you can open the folder in Explorer, CMirror can sync to it.

**History is self-evident without tools.** You can open the local repo in Explorer and see `renderer,cm00000005.c` sitting there. No `git log`, no object database, no need to extract anything. Very useful in a studio environment where not everyone is a programmer.

**No content merging means no surprise merges.** Git's auto-merge is mostly great, but when it goes wrong it goes badly wrong. CMirror never silently produces a merged file — you always know exactly what happened to your file.

**Simpler mental model for small teams.** Workspace / local / central maps directly to physical directories on disk. There's no DAG, no staging area, no detached HEAD, no rebase vs. merge debate. The target audience is a 2–5 person team who just want their files backed up and shareable.

---

## Where Git Wins (Most of the Time)

**Branches.** CMirror has none. Everyone works on the same linear history. You can't isolate an experimental feature from the main line without coordination — you either commit it or you don't.

**Content merging.** If Alice adds a function at the top of a file and Carol adds one at the bottom, Git auto-merges them correctly. CMirror would flag it as a conflict that a human has to resolve manually.

**Commit messages.** CMirror has no concept of annotating *why* something changed. The version history is just numbered file copies with no metadata beyond the modification timestamp. Six months later, `renderer,cm00000006.c` tells you nothing about what was fixed.

**Cross-platform.** CMirror is Windows-only (Win32 API throughout). Git runs everywhere.

**Distributed.** Every Git clone is a full backup. With CMirror, if the central file server dies and nobody has recently synced their local, you lose history. The local repo is a partial mirror, not a full one.

**Integrity.** Git content-addresses everything with SHA hashes, so corruption is detectable. CMirror just assumes the filesystem is honest.

**Scales to large teams.** CMirror scanning a 50,000-file repo over a network share would be painfully slow (hence the cache hack). Git's pack format handles large repos efficiently.

**Tooling ecosystem.** GitHub, GitLab, VS Code integration, `git bisect`, `git blame`, `git stash` — none of that applies here.

---

## The Honest Summary

CMirror is roughly what version control looked like before Linus wrote Git — a refined take on the "put numbered copies in a shared folder" approach that many small studios still use informally. It's a real tool with real safety guarantees (the `EOLLFO` sentinel, the wrapped file ops, the user confirmation prompts), but it's solving a narrower problem: *keep a small team's files in sync on a Windows network, including binary assets, without anyone needing to learn a VCS.*

For a 3-person game studio sharing assets over a NAS in 2007, it's arguably better than Git for that use case. For almost anything else today — especially code, open source, or teams larger than ~5 — Git wins decisively.
