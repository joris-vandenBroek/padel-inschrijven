---
name: project-nieuwe-laptop
description: Wat mee te nemen bij overstap naar nieuwe werklaptop
metadata: 
  node_type: memory
  type: project
  originSessionId: 2863171c-4434-40e7-93bf-e3369c315a9a
---

Bij een nieuwe laptop moeten de volgende Claude-bestanden handmatig worden gekopieerd van de oude laptop:

- `C:\Users\broek01\.claude\` — globale map met memory, instellingen en plugins
- `C:\Users\broek01\.claude.json` — authenticatie/globale config
- `C:\Users\broek01\.claude\settings.json` — globale instellingen

**Why:** Deze bestanden staan lokaal en gaan niet automatisch mee. Ze bevatten projectgeheugen, permissie-instellingen en plugins.

**How to apply:** Wanneer de gebruiker vraagt welke bestanden gekopieerd moeten worden bij een nieuwe laptop, altijd deze bestanden noemen.

De projecten zelf staan al op de NAS (`\\MyCloudEX2Ultra\Transmission\ETV-Volley\`) en zijn direct beschikbaar na installatie van Claude Code.
