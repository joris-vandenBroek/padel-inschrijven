---
name: feedback-commit-push
description: Na elke wijziging altijd direct committen én pushen zonder eerst te vragen
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 2863171c-4434-40e7-93bf-e3369c315a9a
---

Na elke wijziging aan het project altijd direct committen én pushen, zonder eerst bevestiging te vragen. PowerShell en git push mogen zonder toestemming worden uitgevoerd.

**Why:** Joris verwacht dit als standaard werkwijze, consistent met andere projecten. De site draait op GitHub Pages dus pushen = direct live. Voorkeur was eerder opgeslagen in knltb-autoboek memory maar hoort hier thuis.

**How to apply:** Na elke edit of reeks edits meteen `git add`, `git commit` en `git push` uitvoeren zonder tussentijdse vraag.
