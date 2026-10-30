# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

> ⚠️ **Working-directory check — do this first, before anything else.** If this
> session's working directory is under `L:\ETV-Volley\padel-inschrijven` (or its UNC
> form `\\192.168.1.21\Transmission\ETV-Volley\padel-inschrijven` /
> `\\MyCloudEX2Ultra\Transmission\ETV-Volley\padel-inschrijven`), that is the **backup
> destination only** (kept in sync by `scripts/backup.ps1`), not the working copy.
> Immediately tell the user they appear to have started a session in the backup
> location by accident, and ask them to restart in
> `C:\Projecten\ETV-Volley\padel-inschrijven` instead. Do not make edits in the L:
> copy — changes there get silently overwritten by the next scheduled backup run.

## Project overview

Single-file standalone HTML app (`inschrijflijst.html`) for weekly padel/tennis sign-up sessions. No build step, no framework, no server. Hosted on GitHub Pages. Data lives in Firebase Realtime Database.

**Werkmap:** `C:\Projecten\ETV-Volley\padel-inschrijven` (verplaatst hierheen op 2026-08-27 vanaf `L:\ETV-Volley\padel-inschrijven` / `\\MyCloudEX2Ultra\Transmission\ETV-Volley\padel-inschrijven`, wat nu alleen nog de back-up-bestemming is — zie `scripts/backup.ps1`; werk niet meer op L:).  
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

`sport` can be `"padel"`, `"tennis"`, `"beide"` (two-column padel/tennis layout, shared `mainSize`/`reserveSize`), or `"split"` (two-column layout where both columns are the same sport at different times/capacities). A `"split"` type has no top-level `mainSize`/`reserveSize` — instead:

```
blokA: { label {nl, en}, mainSize }
blokB: { label {nl, en}, mainSize }
```

Reserve is uncapped for both blocks in `"split"` mode (unlike `"beide"`, where reserve is capped by `reserveSize`).

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
    vrijdagmorgen_2_blokken_a: ["Naam", …]  ← split-sport, blok A
    vrijdagmorgen_2_blokken_b: ["Naam", …]  ← split-sport, blok B
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
    main_a/ { 0..N: naam }         ← split-sport, blok A (reserve_a onbeperkt)
    reserve_a/ …
    main_b/ …                     ← split-sport, blok B (reserve_b onbeperkt)
    reserve_b/ …
```

Sessies zijn alleen op datum gesleuteld (`sessies/<YYYY-MM-DD>`), niet op event type — twee event types die op dezelfde dag vallen (zelfde `dag` waarde) delen dus dezelfde Firebase-node. `nieuweLijstAanmaken()` waarschuwt en blokkeert als een datum al in gebruik is door een ander event type; `switchToDatum(datum, expectedEventType)` doet dezelfde check bij het automatisch wisselen naar de actieve datum (zie Key flows).

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
- **Split-sport render**: `renderSplit()` + `createSplitSlot()` — analoog aan beide-sport, maar beide kolommen zijn dezelfde sport op andere tijden/capaciteiten (`blokA`/`blokB`); reserve is onbeperkt (`reserveArray()` i.p.v. `slotArray()`)
- **Deelnemers tellen**: `telDeelnemersSessie(session, prefix)` — helper die main/reserve telt over alle drie sessievormen (enkelvoudig, beide, split) heen, o.a. gebruikt in de admin-dropdowns voor "andere lijst openen"
- **Datumbotsing tussen event types**: omdat sessies alleen op datum gesleuteld zijn, checkt `nieuweLijstAanmaken()` of de gekozen datum al in gebruik is door een ánder event type en blokkeert dan het overschrijven; `switchToDatum(datum, expectedEventType)` doet dezelfde check wanneer een sessie automatisch geopend wordt via de actieve-datum listener (niet bij bewust "andere lijst" browsen — dat geeft geen `expectedEventType` mee)

### Verbinding en herstel (belangrijk)

De pagina herlaadt zichzelf **nooit** automatisch. Een eerdere versie deed dat na een
verbindingstimeout (`location.reload()`); bij een stilgevallen verbinding werd dat een
eindeloze lus — elke 8 seconden opnieuw ~600 KB aan scripts. Op iOS is zo'n lus
nauwelijks te verlaten omdat Safari en de WhatsApp-browser de pagina bij het heropenen
herstellen, waardoor gebruikers hun geschiedenis moesten wissen om hun toestel terug te
krijgen. **Voeg hier dus nooit een automatische reload toe.**

- `herverbind(knop?)` — wist de `firebase:host:`-cache en doet `goOffline()` + `goOnline()`,
  zonder de pagina te herladen. Dit is de enige herstelroute.
- `db.ref(".info/connected")` stuurt `isVerbonden` / `heeftOoitVerbonden` en toont of
  verbergt `#connBanner`. De lijst blijft zichtbaar en bruikbaar als de verbinding wegvalt.
- Na 10 seconden zonder eerste verbinding vervangt `#loadingOverlay` zijn inhoud door een
  foutmelding met een handmatige knop.
- **Transport**: Firebase gebruikt standaard een WebSocket. Sommige netwerken en
  iCloud Private Relay laten die niet door — hij opent nooit en meldt geen fout.
  Firebase valt dan zelf terug op long-polling over HTTPS, maar doet daar 10 tot 30
  seconden over. Precies dat maakte de oude reload na 8 seconden zo schadelijk: die
  brak de terugval telkens af, waardoor de verbinding nooit tot stand kwam. Lukte de
  eerste verbinding trager dan `TRAAG_MS`, dan onthoudt `onthoudTransport()` dat in
  `padel_longpolling_sinds` en start het volgende bezoek meteen met
  `db.INTERNAL.forceLongPolling()`. Na een week vervalt dat, zodat iemand die Private
  Relay uitzet weer op WebSockets uitkomt.
- Tijdlijn van het laadscherm: 8 s → tekst "duurt langer dan normaal" (de terugval
  loopt nog), 35 s → foutmelding met handmatige knop. Verhoog die 35 s niet zonder
  reden en verlaag hem zeker niet onder ~30 s.
- `store.lokaal` / `store.sessie` — wrapper om localStorage/sessionStorage die nooit gooit.
  In Lockdown Mode en sommige in-app browsers gooien die APIs; een ongevangen fout sloopte
  het hele script. Gebruik altijd deze wrapper, nooit `localStorage` direct.

### In-app browsers

`inAppBrowserNaam()` herkent WhatsApp, Facebook, Instagram, LINE, Snapchat en LinkedIn en
toont een **zachte, wegklikbare** banner (`#inappBanner`) — het blokkeert niets. Detectie
via de user agent is principieel onbetrouwbaar: de iOS-UA van WhatsApp bevat gewoon
`Version/16.3`, en SFSafariViewController is per ontwerp niet van echte Safari te
onderscheiden. Bouw hier dus nooit een blokkade op; de app moet het overal gewoon doen.

### Role-based access control

Admins hebben een `eventTypes` veld: `"*"` = superadmin (Joris van den Broek), array van IDs = beperkte toegang. Helper `heeftToegang(id)` controleert dit. Secties "Beheerders" en "Evenement types" zijn alleen zichtbaar voor superadmin (`.superadmin-only` class). Dropdowns voor nieuwe lijst, andere lijst openen en vaste deelnemers worden gefilterd op `heeftToegang()`.

### Migratie oud admins-formaat

Bij het laden wordt het oude flat-string formaat `{ "Naam": "ww" }` automatisch gemigreerd naar `{ "Naam": { ww: "...", eventTypes: … } }`. Joris van den Broek krijgt `"*"`, anderen krijgen alle huidige event types.

### WhatsApp share

`shareLink()` gebruikt `eventConfig.shareUrl` als die is ingesteld, anders de basis-URL met `?event=` parameter. De `dinsdag.html` wrapper zorgt dat WhatsApp de juiste og:title preview toont voor de dinsdag tinyurl.

### Tijdvelden

Start- en eindtijd worden opgeslagen als `"9:00"` / `"10:30"` (geen leading zero). Het formulier gebruikt `type="time"` inputs; `toTimeInput()` / `fromTimeInput()` converteren tussen beide formaten.
