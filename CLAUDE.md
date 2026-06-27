# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project overview

Single-file standalone HTML app (`inschrijflijst.html`) for weekly padel/tennis sign-up sessions. No build step, no framework, no server. Hosted on GitHub Pages. Data lives in Firebase Realtime Database.

**Lokale locatie:** `\\MyCloudEX2Ultra\Transmission\ETV-Volley\padel-inschrijven`  
**Live URL (vrijdag):** https://tinyurl.com/padel-inschrijven  
**Live URL (dinsdag):** https://tinyurl.com/LossePols

## Architecture

Everything is in one self-contained file: HTML, CSS custom properties, vanilla JS. A separate `dinsdag.html` wrapper exists solely for the WhatsApp og:title preview — it redirects immediately to `inschrijflijst.html?event=dinsdag_losse_pols`.

### Event types

Each session is tied to an event type (e.g. `vrijdagochtend`, `dinsdag_losse_pols`). The active event type is determined by the `?event=` URL parameter; without it the default (`DEFAULT_EVENT_TYPE = "vrijdagochtend"`) is used.

Event type config lives in Firebase at `instellingen/eventTypes/<id>` and is mirrored in the in-memory `EVENT_TYPES` object. Fields per type:

```
id, label {nl, en}, eyebrow {nl, en}, dag, startTijd, eindTijd,
verloopUur, mainSize, reserveSize, sport, shareUrl?
```

`sport` can be `"padel"`, `"tennis"`, or `"beide"` (two-column layout).

### Firebase data structure

```
instellingen/
  admins/
    "Naam": { ww: "wachtwoord", eventTypes: "*" | ["id", …] }
  actiefPerType/
    vrijdagochtend: "YYYY-MM-DD"
    dinsdag_losse_pols: "YYYY-MM-DD"
  vasteDeelnemersPerType/
    vrijdagochtend: ["Naam", …]
    dinsdag_losse_pols_padel: ["Naam", …]   ← beide-sport
    dinsdag_losse_pols_tennis: ["Naam", …]  ← beide-sport
  eventTypes/
    <id>/ { …config fields… }

sessies/
  YYYY-MM-DD/
    title, time, eventType, aangemaaktOp
    main/ { 0..N: naam }          ← enkelvoudig sport
    reserve/ { 0..M: naam }
    main_padel/ …                 ← beide-sport
    reserve_padel/ …
    main_tennis/ …
    reserve_tennis/ …
```

### State variables (JS)

```js
let identity        // naam opgeslagen in localStorage
let isAdmin         // true na succesvol inloggen
let adminNaam       // naam van ingelogde beheerder
let adminEventTypes // "*" = superadmin, string[] = toegestane event type IDs
let activeDatum     // actieve datum per type in Firebase
let currentDate     // datum die de beheerder lokaal bekijkt
let currentSession  // snapshot van de huidige sessie
let eventConfig     // actief event type object
let EVENT_TYPES     // alle event types geladen uit Firebase
let sessieListenerRef  // actieve Firebase .on() listener — altijd .off() voor je wisselt
```

### Key flows

- **Inschrijven**: `joinFirstFree(sport?)` → Firebase transaction → slot claimen
- **Uitschrijven**: `confirmRemove` → Firebase transaction → reserve schuift automatisch door
- **Admin login**: `checkAdminPw()` → vergelijkt met `ADMINS[naam].ww` → zet `isAdmin`, `adminNaam`, `adminEventTypes`
- **Nieuwe lijst**: `nieuweLijstAanmaken()` → schrijft sessie naar Firebase, vaste deelnemers worden automatisch ingevuld
- **Lijst activeren**: `maakLijstActief()` → schrijft naar `instellingen/actiefPerType/<type>`
- **Lijst verwijderen**: `verwijderHuidigeLijst()` → `.off()` listener eerst, dan `.remove()` — anders herschrijft de listener de sessie meteen
- **Beide-sport render**: `renderBeide()` + `createBeideSlot()` — slots worden afwisselend (padel, tennis) als directe grid-children geplaatst zodat CSS Grid automatisch gelijke rijhoogte garandeert

### Role-based access control

Admins hebben een `eventTypes` veld: `"*"` = superadmin (Joris van den Broek), array van IDs = beperkte toegang. Helper `heeftToegang(id)` controleert dit. Secties "Beheerders" en "Evenement types" zijn alleen zichtbaar voor superadmin (`.superadmin-only` class). Dropdowns voor nieuwe lijst, andere lijst openen en vaste deelnemers worden gefilterd op `heeftToegang()`.

### Migratie oud admins-formaat

Bij het laden wordt het oude flat-string formaat `{ "Naam": "ww" }` automatisch gemigreerd naar `{ "Naam": { ww: "...", eventTypes: … } }`. Joris van den Broek krijgt `"*"`, anderen krijgen alle huidige event types.

### WhatsApp share

`shareLink()` gebruikt `eventConfig.shareUrl` als die is ingesteld, anders de basis-URL met `?event=` parameter. De `dinsdag.html` wrapper zorgt dat WhatsApp de juiste og:title preview toont voor de dinsdag tinyurl.

### Tijdvelden

Start- en eindtijd worden opgeslagen als `"9:00"` / `"10:30"` (geen leading zero). Het formulier gebruikt `type="time"` inputs; `toTimeInput()` / `fromTimeInput()` converteren tussen beide formaten.
