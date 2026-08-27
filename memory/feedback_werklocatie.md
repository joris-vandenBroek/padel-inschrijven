---
name: feedback-werklocatie
description: "Werk voor padel-inschrijven vanuit de NAS-locatie, geen losse lokale C:-kopie"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 60ff611d-238a-4e8e-bbd6-6c5e61aaead9
  modified: 2026-08-08T13:59:31.244Z
---

Gebruik voor dit project standaard `L:\ETV-Volley\padel-inschrijven` (UNC: `\\MyCloudEX2Ultra\Transmission\ETV-Volley\padel-inschrijven`) als werklocatie, niet een losse kopie onder `C:\Users\broek01\...`.

**Why:** Er bestond een losse kopie op `C:\Users\broek01\padel-inschrijven` die uit sync raakte met de NAS-versie (6 commits verschil). Na het gelijktrekken is besloten die lokale kopie te verwijderen — de NAS-map is de enige echte werklocatie, zie ook [[project-nieuwe-laptop]].

**How to apply:** Als een sessie start vanuit een lokale `C:`-kopie van dit project, wijs de gebruiker erop dat de canonieke locatie op de NAS staat en dat een aparte lokale checkout dubbel werk/desync-risico oplevert.
