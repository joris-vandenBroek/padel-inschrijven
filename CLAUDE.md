# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-file standalone HTML app (`inschrijflijst.html`) for a weekly padel signup list. No build step, no server, no dependencies beyond Google Fonts. Open the file directly in a browser.

## Architecture

Everything is in one self-contained file: HTML structure, CSS custom properties, and vanilla JS. No frameworks.

### State model
All state lives in `localStorage` with versioned keys (e.g. `inschrijflijst_vr5jun_state_v7`). When the `VERSION` constant is bumped, old keys are cleaned up on load and the initial hardcoded list is restored.

- `state` — `{ main: string[], reserve: string[] }` — the full signup list
- `mySlots` — `[{ listType, index }]` — which slots this browser "owns" (for unsubscribe rights)
- `identity` — the display name saved locally (persists across sessions, not versioned)
- `settings` — `{ title, time }` — date/time shown in the header

### Key flows
- **Inschrijven**: `joinFirstFree(listType)` → finds first empty slot → `openJoinModal` → `confirmJoin` → pushes to `mySlots`
- **Uitschrijven**: `removeName` → confirms → `doRemove` → `shiftListUp` → if main list, promotes first reservist automatically
- **Admin mode**: password `padel` (hardcoded) → `isAdmin = true` → can remove any slot and set next Friday date
- **Identity claim**: on load or name change, `claimExistingSlots` matches existing names case-insensitively so users can reclaim their spot across browsers

### Resetting for a new week
Change `VERSION` (e.g. `v7` → `v8`) and update `initialMain`/`initialReserve` with the new starting list. Update the hardcoded date strings in HTML header, `copyWhatsApp()`, and `shareLink()`.

### Admin password
The password is `padel`, stored in plaintext as `ADMIN_PW`. This is intentional — it's low-stakes access control for a friendly group.
