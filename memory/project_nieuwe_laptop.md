---
name: project-nieuwe-laptop
description: Wat mee te nemen bij overstap naar nieuwe werklaptop
metadata: 
  node_type: memory
  type: project
  originSessionId: 2863171c-4434-40e7-93bf-e3369c315a9a
---

Bij een nieuwe laptop:

- Projecten staan in git op `C:\projecten\` — na clonen direct beschikbaar.
- Memory staat in de repo zelf (`memory/`-map per project) en is dus al in git.
- Junctions aanmaken zodat Claude Code de memory vindt (zie Glassart and Design-sessie voor het mklink-commando per project).
- `C:\Users\broek01\.claude.json` en `C:\Users\broek01\.claude\settings.json` handmatig meenemen voor globale instellingen en plugins.

**Why:** Memory stond vroeger alleen lokaal in `~/.claude/projects/`, maar is nu per project in git opgeslagen en via een junction gekoppeld — zie ook [[feedback-werklocatie]].

**How to apply:** Bij laptopovergang: projecten clonen, junctions aanmaken, globale Claude-config kopiëren. De NAS is geen werklocatie meer.
