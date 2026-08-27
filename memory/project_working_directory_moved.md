---
name: project-working-directory-moved
description: "padel-inschrijven's working copy moved from L:\\ETV-Volley to C:\\Projecten\\ETV-Volley on 2026-08-27"
metadata: 
  node_type: memory
  type: project
  originSessionId: 89bceb6c-09d0-4e76-9d14-b864840c4ebf
  modified: 2026-08-27T10:38:08.513Z
---

The working copy for this project moved from `L:\ETV-Volley\padel-inschrijven` to
`C:\Projecten\ETV-Volley\padel-inschrijven` on 2026-08-27. Always work in the
`C:\Projecten` copy going forward — the user explicitly asked for this.

`L:\ETV-Volley\padel-inschrijven` (= `\\192.168.1.21\Transmission\ETV-Volley\padel-inschrijven`,
also referenced by hostname as `\\MyCloudEX2Ultra\Transmission\...` in CLAUDE.md) is now
**only** a backup destination, kept in sync by `scripts/backup.ps1` — do not edit files there.

**Why:** the user copied the whole `L:\ETV-Volley` folder (this project plus a sibling
`knltb-autoboek` project) to `C:\Projecten\ETV-Volley` and wants L: reserved as a
network-drive backup target, matching the pattern already used in [[backup_script_pattern]]
for the Glassart and design / Duskie projects.

**How to apply:** if a Bash/PowerShell command's cwd ever defaults back to the L: path
(observed to happen after tool calls reset the shell), explicitly `cd` to
`C:\Projecten\ETV-Volley\padel-inschrijven` before running project commands — don't rely on
the persisted cwd. CLAUDE.md's "Werkmap" line documents this move.
