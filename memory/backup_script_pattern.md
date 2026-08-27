---
name: backup-script-pattern
description: "User's established 3-layer backup convention (GitHub + git bundle + robocopy mirror), used across Glassart and design, Duskie, and padel-inschrijven"
metadata: 
  node_type: memory
  type: user
  originSessionId: 89bceb6c-09d0-4e76-9d14-b864840c4ebf
  modified: 2026-08-27T11:43:32.572Z
---

Across multiple projects (Glassart and design, Duskie, padel-inschrijven) the user has the
same `scripts/backup.ps1` convention for backing up a git project to their NAS
(`\\192.168.1.21\Transmission\...`, drive-lettered as `L:` in an interactive session).
Reuse this pattern whenever asked to add a backup script to a new project, rather than
designing one from scratch.

Three layers, each with a distinct purpose:
1. **GitHub** — primary backup of everything in git (user pushes manually).
2. **git bundle** (`--all`, write to `.nieuw` temp name, `git bundle verify`, then
   `Move-Item -Force`) — the complete history as one file, in case GitHub is unreachable.
   Never overwrite a working bundle with a half-written or corrupt one.
3. **robocopy `/MIR`** of the working tree — catches everything git doesn't track
   (`.env*`, secrets, `.claude/`, scratch dirs). `.git` itself is deliberately excluded
   from the mirror (robocopy isn't git-aware — `/MIR` on a `.git` that diverges from the
   source, e.g. a stray branch, silently deletes those objects; the bundle is the safe
   substitute). Build-artifact dirs (`node_modules`, `.next`, `out`, `.worktrees`) are also
   excluded when a project has them.

**Why this shape:** discovered by direct experience — an early setup that mirrored `.git`
lost commits that existed only in the NAS copy (an old branch), which is why `.git` is now
always bundle-only, never mirrored.

**Operational details worth repeating in any new instance:**
- Log to a file inside the project directory (not `%LOCALAPPDATA%`), added to
  `.gitignore` and excluded from the robocopy mirror — a scheduled task can run in a
  different environment than the interactive session, so a log written elsewhere becomes
  unfindable.
- Use the UNC path (`\\192.168.1.21\Transmission\...`), never the `L:` drive letter — drive
  mappings are per-session, and a scheduled task running while the user isn't logged in
  won't see `L:`.
- Check NAS reachability with a 3-second raw TCP connect to port 445 before any
  `Test-Path` on the UNC share — `Test-Path` on an unreachable share can hang for tens of
  seconds on the SMB timeout.
- If the NAS is simply unreachable (off the home network), log and `exit 0` — not an
  error. Treating "not on home network" as a failure trains the user to ignore red exit
  codes from the scheduled task, which defeats the point of monitoring it.
- Robocopy exit codes 0-7 are success (0 = no changes, 1-7 = files copied/mismatched-but-
  fine); only 8+ is a real failure — translate this explicitly in any script that reports
  robocopy's exit code, since raw robocopy codes read wrong under the usual "0 = success"
  assumption.

Reference implementations: `C:\Projecten\Glassart and design\scripts\backup.ps1`,
`C:\Projecten\Duskie\scripts\backup.ps1`, `C:\Projecten\ETV-Volley\padel-inschrijven\scripts\backup.ps1`,
`C:\Projecten\ETV-Volley\knltb-autoboek\scripts\backup.ps1`.
See also [[project_working_directory_moved]].

**Scheduling:** each project gets its own Windows Scheduled Task named "<Project> back-up",
daily at 17:30, `LogonType Interactive` / `RunLevel Limited` for user `broek01`,
`MultipleInstances IgnoreNew`, `StartWhenAvailable` (catches up if the user wasn't logged
in at 17:30), `ExecutionTimeLimit` 30 min. As of 2026-08-27 all four projects (Glassart,
Duskie, padel-inschrijven, knltb-autoboek) have this task registered. When a new project
gets a backup.ps1, register its scheduled task the same way rather than asking the user to
do it manually — but confirm first, since registering a Scheduled Task is a standing/
persistent configuration change.
