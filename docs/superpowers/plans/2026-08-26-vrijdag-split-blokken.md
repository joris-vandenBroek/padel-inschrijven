# Vrijdag-lijst opsplitsen in 2 tijdblokken — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a new `sport: "split"` event-type mode to `inschrijflijst.html` so the vrijdagochtend-lijst shows 2 columns (9:00-11:30, max 20 / 11:30-12:00, max 12) with unlimited reserve per column, without changing any existing `"beide"` (dinsdag padel/tennis) or single-sport behavior.

**Architecture:** Everything lives in the single file `inschrijflijst.html` (no build step, no framework — GitHub Pages + Firebase Realtime Database). The new mode is implemented as fully separate functions (`renderSplit`, `doJoinSplitTransaction`, `doRemoveSplitTransaction`, …) added alongside the existing `beide`/single-sport functions, with one new `else if` branch added to each shared dispatch point (`render()`, `initialiseerSessie()`, `confirmClearList()`). No existing function bodies for `beide`/single-sport are modified.

**Tech Stack:** Vanilla HTML/CSS/JS, Firebase Realtime Database (compat SDK, already loaded via `<script>` tags), html2canvas (already loaded), no package manager, no test runner.

## Global Constraints

- Sport key for the new mode: `"split"`. Column suffixes are fixed as `_a` / `_b` (never derived from the label text, so labels can be edited later without a data migration).
- Both columns keep their own `mainSize`; reserve is uncapped for both columns (no `reserveSize` field for this sport).
- The `"beide"` sport (dinsdag, padel/tennis) must not be touched — no shared function bodies edited, only new `else if` branches added to dispatch points.
- Every new user-facing string needs both an `nl` and an `en` entry in `TRANSLATIONS` (see `t()` at `inschrijflijst.html:1332`) — a key present in only one language silently falls back to Dutch text in the other language, which is a real bug for this bilingual app.
- Reserve-empty placeholder text: "Nog geen reserves" (NL) / "No reserves yet" (EN), styled like the existing `.slot.empty .slot-name` (grey italic).
- This is a single HTML file with no test framework — every task's verification is a manual procedure in a real browser (via the Browser tool), driving the live Firebase project under an isolated test event-type id (`vrijdagochtend_split_test`) so production data (the real `vrijdagochtend` event type and any live session) is never touched until Task 9.
- To exercise admin-only UI during testing without real credentials, verification steps set `isAdmin = true; adminNaam = "Test"; adminEventTypes = "*";` directly via the browser JS console (a plain global variable in this app), then call `render()`/`openAdminPanel()` as needed. Never read or use real admin passwords from Firebase.
- Do not run `git push` or write to `instellingen/eventTypes/vrijdagochtend` (the real production event type) until Task 9/10, and only after explicit user go-ahead — those are live, user-facing changes (see Task 9/10 checkpoints).

---

### Task 1: Event type schema + admin "Evenement types" form for `sport: "split"`

**Files:**
- Modify: `inschrijflijst.html:883-889` (etSport select), `inschrijflijst.html:869-889` (capaciteit block — add wrapper id + new fields), `inschrijflijst.html:2428-2458` (`openEventTypeForm`), `inschrijflijst.html:2460-2492` (`slaEventTypeOp`)

**Interfaces:**
- Produces: event type objects with `sport: "split"`, `blokA: {label:{nl,en}, mainSize}`, `blokB: {label:{nl,en}, mainSize}` stored at `instellingen/eventTypes/<id>` — all later tasks read `eventConfig.blokA`/`eventConfig.blokB`.
- Produces: `toggleEtSportVelden()` global function.

- [ ] **Step 1: Add the `id="etCapaciteitGrid"` wrapper and split-only fields to the event type modal HTML**

In `inschrijflijst.html`, find (around line 870-889):

```html
      <div style="display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:8px;">
        <div>
          <div style="font-size:11px;color:var(--ink-light);margin-bottom:4px;">Max deelnemers</div>
          <input id="etMainSize" type="number" min="1" max="100" step="1"
            style="width:100%;box-sizing:border-box;padding:10px 10px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:15px;background:var(--bg);color:var(--ink);outline:none;" />
        </div>
        <div>
          <div style="font-size:11px;color:var(--ink-light);margin-bottom:4px;">Max reserve</div>
          <input id="etReserveSize" type="number" min="0" max="20" step="1"
            style="width:100%;box-sizing:border-box;padding:10px 10px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:15px;background:var(--bg);color:var(--ink);outline:none;" />
        </div>
      </div>
      <select id="etSport"
        style="width:100%;padding:10px 14px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:15px;background:var(--bg);color:var(--ink);outline:none;">
        <option value="padel">🎾 Padel</option>
        <option value="tennis">🎾 Tennis</option>
        <option value="beide">🎾 Padel &amp; Tennis</option>
      </select>
    </div>
```

Replace it with:

```html
      <div id="etCapaciteitGrid" style="display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:8px;">
        <div>
          <div style="font-size:11px;color:var(--ink-light);margin-bottom:4px;">Max deelnemers</div>
          <input id="etMainSize" type="number" min="1" max="100" step="1"
            style="width:100%;box-sizing:border-box;padding:10px 10px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:15px;background:var(--bg);color:var(--ink);outline:none;" />
        </div>
        <div>
          <div style="font-size:11px;color:var(--ink-light);margin-bottom:4px;">Max reserve</div>
          <input id="etReserveSize" type="number" min="0" max="20" step="1"
            style="width:100%;box-sizing:border-box;padding:10px 10px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:15px;background:var(--bg);color:var(--ink);outline:none;" />
        </div>
      </div>
      <select id="etSport" onchange="toggleEtSportVelden()"
        style="width:100%;padding:10px 14px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:15px;background:var(--bg);color:var(--ink);outline:none;">
        <option value="padel">🎾 Padel</option>
        <option value="tennis">🎾 Tennis</option>
        <option value="beide">🎾 Padel &amp; Tennis</option>
        <option value="split">🎾 Padel (gesplitst in 2 tijdvakken)</option>
      </select>
      <div id="etSplitVelden" style="display:none;flex-direction:column;gap:10px;margin-top:12px;">
        <div>
          <div style="font-size:11px;font-weight:600;color:var(--ink-light);margin-bottom:6px;">Blok A</div>
          <input id="etBlokANaamNl" type="text" maxlength="60" autocomplete="off" placeholder="Label NL, bv. 9:00 - 11:30 (20 plekken)"
            style="width:100%;box-sizing:border-box;padding:10px 14px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:14px;background:var(--bg);color:var(--ink);outline:none;margin-bottom:6px;" />
          <input id="etBlokANaamEn" type="text" maxlength="60" autocomplete="off" placeholder="Label EN, e.g. 9:00 - 11:30 (20 spots)"
            style="width:100%;box-sizing:border-box;padding:10px 14px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:14px;background:var(--bg);color:var(--ink);outline:none;margin-bottom:6px;" />
          <input id="etBlokAMainSize" type="number" min="1" max="100" step="1" placeholder="Max plekken blok A"
            style="width:100%;box-sizing:border-box;padding:10px 14px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:14px;background:var(--bg);color:var(--ink);outline:none;" />
        </div>
        <div>
          <div style="font-size:11px;font-weight:600;color:var(--ink-light);margin-bottom:6px;">Blok B</div>
          <input id="etBlokBNaamNl" type="text" maxlength="60" autocomplete="off" placeholder="Label NL, bv. 11:30 - 12:00 (12 plekken)"
            style="width:100%;box-sizing:border-box;padding:10px 14px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:14px;background:var(--bg);color:var(--ink);outline:none;margin-bottom:6px;" />
          <input id="etBlokBNaamEn" type="text" maxlength="60" autocomplete="off" placeholder="Label EN, e.g. 11:30 - 12:00 (12 spots)"
            style="width:100%;box-sizing:border-box;padding:10px 14px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:14px;background:var(--bg);color:var(--ink);outline:none;margin-bottom:6px;" />
          <input id="etBlokBMainSize" type="number" min="1" max="100" step="1" placeholder="Max plekken blok B"
            style="width:100%;box-sizing:border-box;padding:10px 14px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:14px;background:var(--bg);color:var(--ink);outline:none;" />
        </div>
      </div>
    </div>
```

- [ ] **Step 2: Add `toggleEtSportVelden()` and wire it into `openEventTypeForm()`**

Add this new function right before `function openEventTypeForm(id) {` (`inschrijflijst.html:2428`):

```js
function toggleEtSportVelden() {
  const split = document.getElementById("etSport").value === "split";
  document.getElementById("etCapaciteitGrid").style.display = split ? "none" : "grid";
  document.getElementById("etSplitVelden").style.display = split ? "flex" : "none";
}
```

In `openEventTypeForm()`, after the line `document.getElementById("etReserveSize").value = et ? et.reserveSize : 4;` (`inschrijflijst.html:2440`), add:

```js
  document.getElementById("etBlokANaamNl").value   = et?.blokA?.label?.nl || "";
  document.getElementById("etBlokANaamEn").value   = et?.blokA?.label?.en || "";
  document.getElementById("etBlokAMainSize").value = et?.blokA?.mainSize || 20;
  document.getElementById("etBlokBNaamNl").value   = et?.blokB?.label?.nl || "";
  document.getElementById("etBlokBNaamEn").value   = et?.blokB?.label?.en || "";
  document.getElementById("etBlokBMainSize").value = et?.blokB?.mainSize || 12;
```

At the end of `openEventTypeForm()`, right before `openModal("eventTypeModal");` (`inschrijflijst.html:2456`), add:

```js
  toggleEtSportVelden();
```

- [ ] **Step 3: Read/write `blokA`/`blokB` in `slaEventTypeOp()`**

In `slaEventTypeOp()`, after the line `const et = { id, label: { nl: naamNl, en: naamEn || naamNl }, dag, startTijd, eindTijd, verloopUur, mainSize, reserveSize, sport };` (`inschrijflijst.html:2481`), add:

```js
  if (sport === "split") {
    const blokANl = document.getElementById("etBlokANaamNl").value.trim();
    const blokAEn = document.getElementById("etBlokANaamEn").value.trim();
    const blokBNl = document.getElementById("etBlokBNaamNl").value.trim();
    const blokBEn = document.getElementById("etBlokBNaamEn").value.trim();
    et.blokA = { label: { nl: blokANl, en: blokAEn || blokANl }, mainSize: parseInt(document.getElementById("etBlokAMainSize").value) || 1 };
    et.blokB = { label: { nl: blokBNl, en: blokBEn || blokBNl }, mainSize: parseInt(document.getElementById("etBlokBMainSize").value) || 1 };
  }
```

- [ ] **Step 4: Manual verification in the browser**

Open the app in the Browser tool (any `?event=` works, admin panel is global). In the JS console, run:

```js
isAdmin = true; adminNaam = "Test"; adminEventTypes = "*";
document.getElementById("adminBanner").style.display = "flex";
openAdminPanel();
```

Click "🎯 Evenement types" → "+ Evenement type toevoegen". Confirm the modal opens with "Max deelnemers"/"Max reserve" visible and the split fields hidden. Select "🎾 Padel (gesplitst in 2 tijdvakken)" in the sport dropdown — confirm "Max deelnemers"/"Max reserve" disappear and the Blok A / Blok B fields appear. Fill:
- Naam NL: `Test Split`, Blok A label NL `9:00 - 11:30 (20 plekken)`, EN `9:00 - 11:30 (20 spots)`, max `20`
- Blok B label NL `11:30 - 12:00 (12 plekken)`, EN `11:30 - 12:00 (12 spots)`, max `12`

Click "Opslaan ✓". In the console, run `EVENT_TYPES['test_split']` and confirm the object has `sport: "split"`, `blokA: {label:{nl:"9:00 - 11:30 (20 plekken)", en:"9:00 - 11:30 (20 spots)"}, mainSize:20}`, and the matching `blokB`. Reopen the edit form for this type (click it in the "Evenement types" list) and confirm all 6 split fields repopulate correctly and the sport dropdown shows "split". Switch the dropdown to "padel" and confirm "Max deelnemers"/"Max reserve" reappear and the split fields hide (no save needed for this check).

- [ ] **Step 5: Commit**

```bash
git add inschrijflijst.html
git commit -m "Voeg split-sport event type toe aan Evenement types beheer"
```

---

### Task 2: Session initialization for split sport

**Files:**
- Modify: `inschrijflijst.html:1518-1554` (`initialiseerSessie`)

**Interfaces:**
- Consumes: `eventConfig.blokA.mainSize`, `eventConfig.blokB.mainSize` (Task 1)
- Produces: session documents at `sessies/<datum>` with `main_a`, `reserve_a: {}`, `main_b`, `reserve_b: {}` when `eventType`'s sport is `"split"` — later tasks (3-7) read/write these fields.

- [ ] **Step 1: Add the split branch to `initialiseerSessie()`**

In `inschrijflijst.html`, replace (`inschrijflijst.html:1518-1554`):

```js
async function initialiseerSessie(datum, title, time, eventType) {
  const s = eventConfig.mainSize, r = eventConfig.reserveSize;
  const leeg = (size) => { const o = {}; for (let i = 0; i < size; i++) o[i] = ""; return o; };
  const beide = eventConfig.sport === "beide";

  // Laad vaste deelnemers voor het juiste event type
  const et = eventType || DEFAULT_EVENT_TYPE;
  let vd = [], vdP = [], vdT = [];
  if (beide) {
    const [sp, st] = await Promise.all([
      db.ref(`instellingen/vasteDeelnemersPerType/${et}_padel`).once("value"),
      db.ref(`instellingen/vasteDeelnemersPerType/${et}_tennis`).once("value"),
    ]);
    vdP = Array.isArray(sp.val()) ? sp.val() : [];
    vdT = Array.isArray(st.val()) ? st.val() : [];
  } else {
    const snap = await db.ref(`instellingen/vasteDeelnemersPerType/${et}`).once("value");
    vd = Array.isArray(snap.val()) ? snap.val() : [];
  }

  let data;
  if (beide) {
    const main_padel = leeg(s), main_tennis = leeg(s);
    vdP.slice(0, s).forEach((naam, i) => { main_padel[i] = naam; });
    vdT.slice(0, s).forEach((naam, i) => { main_tennis[i] = naam; });
    data = { title, time, eventType: eventType || DEFAULT_EVENT_TYPE,
      main_padel, reserve_padel: leeg(r), main_tennis, reserve_tennis: leeg(r),
      aangemaaktOp: firebase.database.ServerValue.TIMESTAMP };
  } else {
    const main = leeg(s), reserve = leeg(r);
    vd.slice(0, s).forEach((naam, i) => { main[i] = naam; });
    data = { title, time, eventType: eventType || DEFAULT_EVENT_TYPE, main, reserve,
      aangemaaktOp: firebase.database.ServerValue.TIMESTAMP };
  }
  await db.ref(`sessies/${datum}`).set(data);
  return beide ? vdP.length + vdT.length : vd.length;
}
```

with:

```js
async function initialiseerSessie(datum, title, time, eventType) {
  const s = eventConfig.mainSize, r = eventConfig.reserveSize;
  const leeg = (size) => { const o = {}; for (let i = 0; i < size; i++) o[i] = ""; return o; };
  const beide = eventConfig.sport === "beide";
  const split = eventConfig.sport === "split";

  // Laad vaste deelnemers voor het juiste event type
  const et = eventType || DEFAULT_EVENT_TYPE;
  let vd = [], vdP = [], vdT = [], vdA = [], vdB = [];
  if (beide) {
    const [sp, st] = await Promise.all([
      db.ref(`instellingen/vasteDeelnemersPerType/${et}_padel`).once("value"),
      db.ref(`instellingen/vasteDeelnemersPerType/${et}_tennis`).once("value"),
    ]);
    vdP = Array.isArray(sp.val()) ? sp.val() : [];
    vdT = Array.isArray(st.val()) ? st.val() : [];
  } else if (split) {
    const [sa, sb] = await Promise.all([
      db.ref(`instellingen/vasteDeelnemersPerType/${et}_a`).once("value"),
      db.ref(`instellingen/vasteDeelnemersPerType/${et}_b`).once("value"),
    ]);
    vdA = Array.isArray(sa.val()) ? sa.val() : [];
    vdB = Array.isArray(sb.val()) ? sb.val() : [];
  } else {
    const snap = await db.ref(`instellingen/vasteDeelnemersPerType/${et}`).once("value");
    vd = Array.isArray(snap.val()) ? snap.val() : [];
  }

  let data;
  if (beide) {
    const main_padel = leeg(s), main_tennis = leeg(s);
    vdP.slice(0, s).forEach((naam, i) => { main_padel[i] = naam; });
    vdT.slice(0, s).forEach((naam, i) => { main_tennis[i] = naam; });
    data = { title, time, eventType: eventType || DEFAULT_EVENT_TYPE,
      main_padel, reserve_padel: leeg(r), main_tennis, reserve_tennis: leeg(r),
      aangemaaktOp: firebase.database.ServerValue.TIMESTAMP };
  } else if (split) {
    const main_a = leeg(eventConfig.blokA.mainSize), main_b = leeg(eventConfig.blokB.mainSize);
    vdA.slice(0, eventConfig.blokA.mainSize).forEach((naam, i) => { main_a[i] = naam; });
    vdB.slice(0, eventConfig.blokB.mainSize).forEach((naam, i) => { main_b[i] = naam; });
    data = { title, time, eventType: eventType || DEFAULT_EVENT_TYPE,
      main_a, reserve_a: {}, main_b, reserve_b: {},
      aangemaaktOp: firebase.database.ServerValue.TIMESTAMP };
  } else {
    const main = leeg(s), reserve = leeg(r);
    vd.slice(0, s).forEach((naam, i) => { main[i] = naam; });
    data = { title, time, eventType: eventType || DEFAULT_EVENT_TYPE, main, reserve,
      aangemaaktOp: firebase.database.ServerValue.TIMESTAMP };
  }
  await db.ref(`sessies/${datum}`).set(data);
  return beide ? vdP.length + vdT.length : split ? vdA.length + vdB.length : vd.length;
}
```

- [ ] **Step 2: Manual verification**

In the browser console (same test event type from Task 1, `test_split`):

```js
eventConfig = EVENT_TYPES['test_split'];
initialiseerSessie('2099-01-02', 'Test Split 2 jan', '9:00 – 12:00 uur', 'test_split').then(() => {
  db.ref('sessies/2099-01-02').once('value').then(s => console.log(JSON.stringify(s.val())));
});
```

Confirm the logged object has `main_a` with exactly 20 keys (`"0"`..`"19"`, all `""`), `reserve_a: {}` (or absent — Firebase drops empty objects, that's fine, `reserveArray`-style reads in later tasks treat a missing node the same as `{}`), `main_b` with 12 keys, `reserve_b: {}`, and `eventType: "test_split"`.

- [ ] **Step 3: Commit**

```bash
git add inschrijflijst.html
git commit -m "Initialiseer split-sessies met main_a/main_b en onbeperkte reserve"
```

---

### Task 3: Rendering — `renderSplit()`, `createSplitSlot()`, `render()` dispatch

**Files:**
- Modify: `inschrijflijst.html:1636-1665` (`isBeideSport`/`render`) — add `isSplitSport()` and dispatch branch
- Modify: `inschrijflijst.html:315-321` (`.beide-kolommen` CSS comment area — no change needed, class is reused as-is)

**Interfaces:**
- Consumes: `currentSession.main_a`/`reserve_a`/`main_b`/`reserve_b` (Task 2), `eventConfig.blokA`/`blokB` (Task 1)
- Produces: `isSplitSport()`, `renderSplit()`, `createSplitSlot(naam, i, listType, blokKey, verlopen)` — Tasks 4-5 wire button `onclick`s that these functions render, and reuse `createSplitSlot`'s DOM structure expectations (`.slot`, `.avatar`, `.slot-name`, `.slot-action` — identical to `createBeideSlot`).

- [ ] **Step 1: Add `isSplitSport()` and the `renderSplit()`/`createSplitSlot()` functions**

Add these two new functions right after `createBeideSlot` ends (after `inschrijflijst.html:1794`, i.e. right before `function renderList(...)`):

```js
function isSplitSport() { return eventConfig.sport === "split"; }

function reserveArray(obj) {
  return Object.keys(obj || {}).sort((a, b) => +a - +b).map(k => obj[k]);
}

function renderSplit() {
  const blokA = eventConfig.blokA, blokB = eventConfig.blokB;
  const mainA = slotArray(currentSession.main_a, blokA.mainSize);
  const mainB = slotArray(currentSession.main_b, blokB.mainSize);
  const resA  = reserveArray(currentSession.reserve_a);
  const resB  = reserveArray(currentSession.reserve_b);

  const verlopen = isLijstVerlopen();
  const container = document.getElementById("listContainer");
  const joinBtnHtml = (blokKey, id) => verlopen ? "" :
    `<button class="sport-join-btn padel" id="${id}" onclick="joinSplitFirstFree('${blokKey}')">${t("doeMeeBtn") || "Doe mee"}</button>`;

  container.innerHTML = `
    <div class="beide-kolommen" id="splitGrid">
      <div class="sport-header padel-header">
        <div class="sport-header-top">
          <div class="sport-header-label">
            <span class="sport-header-name">${blokA.label[lang] || blokA.label.nl}</span>
            <span class="sport-header-count" id="countBlokA">— / ${blokA.mainSize}</span>
          </div>
          ${joinBtnHtml("a","joinBlokABtn")}
        </div>
        <div class="sport-stripe padel"></div>
      </div>
      <div class="sport-header tennis-header">
        <div class="sport-header-top">
          <div class="sport-header-label">
            <span class="sport-header-name">${blokB.label[lang] || blokB.label.nl}</span>
            <span class="sport-header-count" id="countBlokB">— / ${blokB.mainSize}</span>
          </div>
          ${joinBtnHtml("b","joinBlokBBtn")}
        </div>
        <div class="sport-stripe padel"></div>
      </div>
    </div>`;

  const grid = document.getElementById("splitGrid");
  const leegVulling = () => { const d = document.createElement("div"); d.className = "beide-slot-padel"; return d; };
  const maxMain = Math.max(blokA.mainSize, blokB.mainSize);
  for (let i = 0; i < maxMain; i++) {
    if (i < blokA.mainSize) grid.appendChild(createSplitSlot(mainA[i], i, "main", "a", verlopen));
    else grid.appendChild(leegVulling());
    if (i < blokB.mainSize) grid.appendChild(createSplitSlot(mainB[i], i, "main", "b", verlopen));
    else grid.appendChild(document.createElement("div"));
  }

  const dA = document.createElement("div");
  dA.className = "divider-reserve beide-divider-padel";
  dA.innerHTML = `<span>${t("reserveLabelOnbeperkt")}</span>`;
  grid.appendChild(dA);
  const dB = document.createElement("div");
  dB.className = "divider-reserve beide-divider-tennis";
  dB.innerHTML = `<span>${t("reserveLabelOnbeperkt")}</span>`;
  grid.appendChild(dB);

  const maxRes = Math.max(resA.length, resB.length, 1);
  for (let i = 0; i < maxRes; i++) {
    if (i < resA.length) grid.appendChild(createSplitSlot(resA[i], i, "reserve", "a", verlopen));
    else if (i === 0) grid.appendChild(createSplitLegePlaceholder("a"));
    else grid.appendChild(leegVulling());
    if (i < resB.length) grid.appendChild(createSplitSlot(resB[i], i, "reserve", "b", verlopen));
    else if (i === 0) grid.appendChild(createSplitLegePlaceholder("b"));
    else grid.appendChild(document.createElement("div"));
  }

  const filledA = mainA.filter(n => n && n.trim()).length;
  const filledB = mainB.filter(n => n && n.trim()).length;
  document.getElementById("countBadge").textContent = `${filledA + filledB} / ${blokA.mainSize + blokB.mainSize}`;
  document.getElementById("countBlokA").textContent = `${filledA} / ${blokA.mainSize}`;
  document.getElementById("countBlokB").textContent = `${filledB} / ${blokB.mainSize}`;
}

// Hergebruikt de bestaande .beide-slot-padel/.beide-divider-padel CSS-klassen (rechter rand)
// puur voor de styling — heeft verder niets met de sport padel te maken.
function cssBlokKlasse(blokKey) { return blokKey === "a" ? "padel" : "tennis"; }

function createSplitLegePlaceholder(blokKey) {
  const div = document.createElement("div");
  div.className = "slot beide-slot-" + cssBlokKlasse(blokKey) + " empty";
  div.innerHTML = `<span class="slot-num"></span><div class="avatar empty">?</div><span class="slot-name">${t("nogGeenReserves")}</span><div class="slot-action"></div>`;
  return div;
}

function createSplitSlot(name, i, listType, blokKey, verlopen) {
  const filled = name && name.trim();
  const mine   = isMine(name);

  const div = document.createElement("div");
  div.className = ["slot", "beide-slot-" + cssBlokKlasse(blokKey),
    filled ? (mine ? "is-mine" : listType === "reserve" ? "reserve" : "filled") : "empty"
  ].join(" ");

  const num = document.createElement("span");
  num.className = "slot-num";
  num.textContent = i + 1;

  const av = document.createElement("div");
  av.className = `avatar ${mine ? "is-mine" : filled ? (listType === "reserve" ? "reserve" : "filled") : "empty"}`;
  av.textContent = filled ? initials(name) : "?";

  const nm = document.createElement("span");
  nm.className = "slot-name";
  nm.textContent = filled ? name : t("vrij");

  const action = document.createElement("div");
  action.className = "slot-action";
  action.style.cssText = "display:flex;align-items:center;gap:6px;flex-shrink:0;";

  if (isAdmin && filled) {
    if (mine) {
      const tag = document.createElement("span"); tag.className = "mine-tag"; tag.textContent = t("jijTag"); action.appendChild(tag);
    }
    const btn = document.createElement("button");
    btn.className = "btn-leave";
    btn.style.cssText = "border-color:var(--gold);color:var(--gold);padding:4px 8px;font-size:11px;";
    btn.textContent = "✕"; btn.title = name;
    btn.onclick = () => adminSplitRemove(listType, i, name, blokKey);
    action.appendChild(btn);
  } else if (mine) {
    const tag = document.createElement("span"); tag.className = "mine-tag"; tag.textContent = t("jijTag"); action.appendChild(tag);
    if (!verlopen) {
      const btn = document.createElement("button");
      btn.className = "btn-leave";
      btn.style.cssText = "padding:4px 8px;font-size:11px;";
      btn.textContent = "✕";
      btn.onclick = () => confirmSplitRemove(name, listType, i, blokKey);
      action.appendChild(btn);
    }
  }

  div.appendChild(num); div.appendChild(av); div.appendChild(nm); div.appendChild(action);
  return div;
}
```

- [ ] **Step 2: Add the `nogGeenReserves`/`reserveLabelOnbeperkt` translation keys**

In `TRANSLATIONS.nl` (right after the `reserveLabel: "Reserve (max 4)",` line, `inschrijflijst.html:989`), add:

```js
    reserveLabelOnbeperkt: "Reserve",
    nogGeenReserves: "Nog geen reserves",
```

In `TRANSLATIONS.en` (find the matching `reserveLabel` line in the `en` block and add right after it):

```js
    reserveLabelOnbeperkt: "Reserve",
    nogGeenReserves: "No reserves yet",
```

- [ ] **Step 3: Wire the dispatch branch into `render()`**

Replace (`inschrijflijst.html:1638-1664`):

```js
function render() {
  if (!currentSession) return;
  updateVerlopenState();
  updateViewBanner();

  const beide = isBeideSport();
  document.getElementById("doeMeeBtn").style.display       = beide ? "none" : "";
  document.getElementById("doeMeePadelBtn").style.display  = "none";
  document.getElementById("doeMeeTennisBtn").style.display = "none";
  document.getElementById("iemandAndersBtn").style.display = beide ? "none" : "";

  if (beide) {
    renderBeide();
  } else {
```

with:

```js
function render() {
  if (!currentSession) return;
  updateVerlopenState();
  updateViewBanner();

  const beide = isBeideSport();
  const split = isSplitSport();
  document.getElementById("doeMeeBtn").style.display       = (beide || split) ? "none" : "";
  document.getElementById("doeMeePadelBtn").style.display  = "none";
  document.getElementById("doeMeeTennisBtn").style.display = "none";
  document.getElementById("iemandAndersBtn").style.display = (beide || split) ? "none" : "";

  if (beide) {
    renderBeide();
  } else if (split) {
    renderSplit();
  } else {
```

(The final `}` that used to close the `if (beide) {...} else {...}` block still closes the new three-way `if/else if/else` — no other change needed on the closing lines.)

- [ ] **Step 4: Manual verification**

In the console, create and load a test split session (reuse `test_split` from Tasks 1-2), then simulate joining a few names into `main_a`/`main_b` and one into `reserve_a` directly via Firebase to check rendering without needing the join flow yet (Task 4 builds that):

```js
db.ref('sessies/2099-01-02').update({
  'main_a/0': 'Piet', 'main_a/1': 'Klaas',
  'main_b/0': 'Jan',
  'reserve_a/0': 'Marie'
});
```

Navigate to `?event=test_split` (or set `urlEventType`/`eventConfig` and call `switchToDatum('2099-01-02')` in console if easier), confirm:
- Two columns render side by side, headers show the exact blok labels and `— / 20` / `— / 12` counts updating to `2 / 20` and `1 / 12`.
- Column A shows 20 rows (Piet, Klaas, then 18 "vrij" rows); column B shows 12 rows (Jan, then 11 "vrij").
- Column A's reserve section shows "Marie" as row 1; column B's reserve section shows the grey italic "Nog geen reserves" placeholder (no numbered row).
- No console errors.

- [ ] **Step 5: Commit**

```bash
git add inschrijflijst.html
git commit -m "Render split-sport sessies in 2 kolommen met onbeperkte reserve"
```

---

### Task 4: Join flow — `joinSplitFirstFree()`, `doJoinSplitTransaction()`, admin-add branch

**Files:**
- Modify: `inschrijflijst.html:2110-2148` (`confirmAdminAdd`) — add split branch
- Add new functions near `joinFirstFree` (`inschrijflijst.html:1893`, insert before it)

**Interfaces:**
- Consumes: `renderSplit()`'s `onclick="joinSplitFirstFree('a'|'b')"` buttons (Task 3)
- Produces: `joinSplitFirstFree(blokKey)`, `doJoinSplitTransaction(blokKey)` — Task 5 reuses `logEntry`/`detectLijstPositie` the same way these do.

- [ ] **Step 1: Add `joinSplitFirstFree()` and `doJoinSplitTransaction()`**

Insert right before `function joinFirstFree(sport) {` (`inschrijflijst.html:1893`):

```js
function joinSplitFirstFree(blokKey) {
  if (isLijstVerlopen() && !isAdmin) return;
  if (!identity) { _pendingJoinSport = "split:" + blokKey; pendingJoin = true; openIdentityModal(); return; }
  if (!currentSession) return;

  if (isAdmin) {
    pendingAdminSlot = { listType: "auto", sport: blokKey, split: true };
    document.getElementById("adminAddDesc").textContent = t("adminAddTitle");
    document.getElementById("adminAddInput").value = "";
    openModal("adminAddModal");
    setTimeout(() => document.getElementById("adminAddInput").focus(), 100);
    return;
  }

  const mainKey = `main_${blokKey}`, reserveKey = `reserve_${blokKey}`;
  const alles = [
    ...Object.values(currentSession.main_a    || {}),
    ...Object.values(currentSession.reserve_a || {}),
    ...Object.values(currentSession.main_b    || {}),
    ...Object.values(currentSession.reserve_b || {}),
  ];
  if (alles.some(n => (n || "").trim().toLowerCase() === identity.trim().toLowerCase())) {
    showToast(t("alOpLijst")); return;
  }

  const mainSize = eventConfig[blokKey === "a" ? "blokA" : "blokB"].mainSize;
  const isReserve = !slotArray(currentSession[mainKey], mainSize).some(n => !n || !n.trim());
  document.getElementById("confirmTitle").textContent = t("inschrijvenVraag");
  document.getElementById("confirmMsg").textContent = isReserve
    ? t("inschrijvenMsgReserve", { naam: identity, title: currentSession.title, time: currentSession.time })
    : t("inschrijvenMsg",        { naam: identity, title: currentSession.title, time: currentSession.time });
  openModal("confirmModal");
  document.getElementById("confirmOkBtn").textContent = t("inschrijvenBevestig");
  document.getElementById("confirmOkBtn").onclick = () => {
    closeModal("confirmModal");
    doJoinSplitTransaction(blokKey);
  };
}

function doJoinSplitTransaction(blokKey) {
  const mainKey = `main_${blokKey}`, reserveKey = `reserve_${blokKey}`;
  const mainSize = eventConfig[blokKey === "a" ? "blokA" : "blokB"].mainSize;
  db.ref(`sessies/${currentDate}`).transaction(session => {
    if (!session) return;
    for (let i = 0; i < mainSize; i++) {
      if (!session[mainKey][i] || !session[mainKey][i].trim()) {
        session[mainKey][i] = identity; return session;
      }
    }
    const reserveLen = Object.keys(session[reserveKey] || {}).length;
    if (!session[reserveKey]) session[reserveKey] = {};
    session[reserveKey][reserveLen] = identity;
    return session;
  }, (error, committed, snapshot) => {
    if (error)      { showToast(t("foutInschrijven")); return; }
    if (!committed) { showToast(t("geenPlek")); return; }
    const data = snapshot.val();
    const pos = detectLijstPositie(data, identity, mainKey, reserveKey);
    logEntry("ingeschreven", identity, identity, pos);
    showToast(pos.lijstType === "reserve" ? t("opReservelijst") : t("ingeschreven"));
  });
}
```

- [ ] **Step 2: Handle the `"split:a"`/`"split:b"` pending-join marker in `confirmIdentity()`**

In `confirmIdentity()` (`inschrijflijst.html:1886`), replace:

```js
  if (pendingJoin) { pendingJoin = false; joinFirstFree(_pendingJoinSport); _pendingJoinSport = null; }
```

with:

```js
  if (pendingJoin) {
    pendingJoin = false;
    if (typeof _pendingJoinSport === "string" && _pendingJoinSport.startsWith("split:")) {
      joinSplitFirstFree(_pendingJoinSport.slice(6));
    } else {
      joinFirstFree(_pendingJoinSport);
    }
    _pendingJoinSport = null;
  }
```

- [ ] **Step 3: Add the split branch to `confirmAdminAdd()`**

In `confirmAdminAdd()` (`inschrijflijst.html:2110`), replace the whole function body with a split-aware version — insert this check right at the top, before the existing `const sport = pendingAdminSlot?.sport || null;` line:

```js
function confirmAdminAdd() {
  const naam = document.getElementById("adminAddInput").value.trim();
  if (!naam) { document.getElementById("adminAddInput").focus(); return; }

  if (pendingAdminSlot?.split) {
    const blokKey = pendingAdminSlot.sport;
    const mainKey = `main_${blokKey}`, reserveKey = `reserve_${blokKey}`;
    const mainSize = eventConfig[blokKey === "a" ? "blokA" : "blokB"].mainSize;
    const alles = [
      ...Object.values(currentSession?.main_a    || {}),
      ...Object.values(currentSession?.reserve_a || {}),
      ...Object.values(currentSession?.main_b    || {}),
      ...Object.values(currentSession?.reserve_b || {}),
    ];
    if (alles.some(n => (n || "").trim().toLowerCase() === naam.toLowerCase())) {
      const inp = document.getElementById("adminAddInput");
      inp.style.borderColor = "var(--accent)";
      showToast(t("alOpLijstWaarschuwing", { naam }));
      setTimeout(() => { inp.style.borderColor = ""; }, 2000);
      inp.focus();
      return;
    }
    db.ref(`sessies/${currentDate}`).transaction(session => {
      if (!session) return;
      for (let i = 0; i < mainSize; i++) {
        if (!session[mainKey][i] || !session[mainKey][i].trim()) { session[mainKey][i] = naam; return session; }
      }
      const reserveLen = Object.keys(session[reserveKey] || {}).length;
      if (!session[reserveKey]) session[reserveKey] = {};
      session[reserveKey][reserveLen] = naam;
      return session;
    }, (error, committed, snapshot) => {
      if (!committed) { showToast(t("geenVrijePlekken")); return; }
      closeModal("adminAddModal");
      const pos = snapshot ? detectLijstPositie(snapshot.val(), naam, mainKey, reserveKey) : {};
      logEntry("ingeschreven", naam, adminNaam, pos);
      showToast(`✓ ${naam} ingeschreven`);
    });
    return;
  }

  const sport = pendingAdminSlot?.sport || null;
  const mainKey    = sport ? `main_${sport}`    : "main";
  const reserveKey = sport ? `reserve_${sport}` : "reserve";

  const alles = isBeideSport()
    ? [...Object.values(currentSession?.main_padel    || {}),
       ...Object.values(currentSession?.reserve_padel || {}),
       ...Object.values(currentSession?.main_tennis   || {}),
       ...Object.values(currentSession?.reserve_tennis|| {})]
    : [...Object.values(currentSession?.main    || {}),
       ...Object.values(currentSession?.reserve || {})];
  if (alles.some(n => (n || "").trim().toLowerCase() === naam.toLowerCase())) {
    const inp = document.getElementById("adminAddInput");
    inp.style.borderColor = "var(--accent)";
    showToast(t("alOpLijstWaarschuwing", { naam }));
    setTimeout(() => { inp.style.borderColor = ""; }, 2000);
    inp.focus();
    return;
  }

  db.ref(`sessies/${currentDate}`).transaction(session => {
    if (!session) return;
    for (let i = 0; i < eventConfig.mainSize; i++) {
      if (!session[mainKey][i] || !session[mainKey][i].trim()) { session[mainKey][i] = naam; return session; }
    }
    for (let i = 0; i < eventConfig.reserveSize; i++) {
      if (!session[reserveKey][i] || !session[reserveKey][i].trim()) { session[reserveKey][i] = naam; return session; }
    }
  }, (error, committed, snapshot) => {
    if (!committed) { showToast(t("geenVrijePlekken")); return; }
    closeModal("adminAddModal");
    const pos = snapshot ? detectLijstPositie(snapshot.val(), naam, mainKey, reserveKey) : {};
    logEntry("ingeschreven", naam, adminNaam, pos);
    showToast(`✓ ${naam} ingeschreven`);
  });
}
```

- [ ] **Step 4: Manual verification**

As a non-admin: in the console, `isAdmin = false; identity = "Piet";`, navigate to the `test_split` session from Task 3 (reset it first: `db.ref('sessies/2099-01-02').set({main_a:{},main_b:{},reserve_a:{},reserve_b:{}})` won't work since main needs full-size empty object — instead re-run `initialiseerSessie` from Task 2's verification, or just `db.ref('sessies/2099-01-02').update({main_a: {...Array(20).fill("")}, main_b: {...Array(12).fill("")}, reserve_a: {}, reserve_b: {}})`). Click blok A's "Doe mee" 20 times with 20 different names (set `identity` between clicks via console) until full, then a 21st time — confirm the 21st lands in `reserve_a` index 0 (check via `db.ref('sessies/2099-01-02/reserve_a').once('value').then(s=>console.log(s.val()))`), and the UI reserve section for column A now shows that name instead of "Nog geen reserves". Then switch to `isAdmin = true; adminNaam="Test"; adminEventTypes="*";`, click blok B's "Doe mee", confirm the admin-add modal opens, type a name, confirm it's added to `main_b`.

- [ ] **Step 5: Commit**

```bash
git add inschrijflijst.html
git commit -m "Voeg join-flow toe voor split-sport kolommen (zelf en als beheerder)"
```

---

### Task 5: Leave flow — `confirmSplitRemove()`, `adminSplitRemove()`, `doRemoveSplitTransaction()`, `shiftUpDynamic()`; split branch in `confirmClearList()`

**Files:**
- Add new functions near `confirmRemove`/`doRemoveTransaction` (`inschrijflijst.html:2021-2096`, insert after `shiftUp`)
- Modify: `inschrijflijst.html:2150-2166` (`confirmClearList`) — add split branch

**Interfaces:**
- Consumes: `createSplitSlot()`'s `onclick="confirmSplitRemove(...)"` / `onclick="adminSplitRemove(...)"` (Task 3)
- Produces: `confirmSplitRemove(naam, listType, idx, blokKey)`, `adminSplitRemove(listType, idx, naam, blokKey)`, `doRemoveSplitTransaction(naam, listType, idx, blokKey)`, `shiftUpDynamic(obj, fromIdx)`

- [ ] **Step 1: Add `shiftUpDynamic()`, `confirmSplitRemove()`, `adminSplitRemove()`, `doRemoveSplitTransaction()`**

Insert right after the `function shiftUp(obj, fromIdx, size) {...}` function ends (`inschrijflijst.html:2089-2096`):

```js
function shiftUpDynamic(obj, fromIdx) {
  const keys = Object.keys(obj || {}).sort((a, b) => +a - +b);
  const len = keys.length;
  const result = {};
  for (let i = 0; i < fromIdx; i++) result[i] = obj[i];
  for (let i = fromIdx; i < len - 1; i++) result[i] = obj[i + 1];
  return result;
}

function confirmSplitRemove(naam, listType, idx, blokKey) {
  const reserveKey = `reserve_${blokKey}`;
  const eersteRKey = Object.keys(currentSession[reserveKey] || {})
    .sort((a, b) => +a - +b)
    .find(k => currentSession[reserveKey][k] && currentSession[reserveKey][k].trim());
  const promotee = (listType === "main" && eersteRKey !== undefined)
    ? currentSession[reserveKey][eersteRKey] : null;

  document.getElementById("confirmTitle").textContent = t("uitschrijvenVraag", { naam });
  document.getElementById("confirmMsg").textContent = promotee
    ? t("doorschuiven", { promotee }) : t("weetJeHetZeker");
  openModal("confirmModal");
  document.getElementById("confirmOkBtn").textContent = t("uitschrijvenKnop");
  document.getElementById("confirmOkBtn").onclick = () => {
    closeModal("confirmModal");
    doRemoveSplitTransaction(naam, listType, idx, blokKey);
  };
}

function adminSplitRemove(listType, idx, naam, blokKey) {
  document.getElementById("confirmTitle").textContent = t("uitschrijvenVraag", { naam });
  document.getElementById("confirmMsg").textContent = t("uitschrijvenAlsBeheerder");
  openModal("confirmModal");
  document.getElementById("confirmOkBtn").textContent = t("uitschrijvenKnop");
  document.getElementById("confirmOkBtn").onclick = () => {
    closeModal("confirmModal");
    doRemoveSplitTransaction(naam, listType, idx, blokKey);
  };
}

function doRemoveSplitTransaction(naam, listType, fromIdx, blokKey) {
  const mainKey = `main_${blokKey}`, reserveKey = `reserve_${blokKey}`;
  const mainSize = eventConfig[blokKey === "a" ? "blokA" : "blokB"].mainSize;
  db.ref(`sessies/${currentDate}`).transaction(session => {
    if (!session) return;
    if (listType === "main") {
      session[mainKey] = shiftUp(session[mainKey], fromIdx, mainSize);
      const eersteRKey = Object.keys(session[reserveKey] || {})
        .sort((a, b) => +a - +b)
        .find(k => session[reserveKey][k] && session[reserveKey][k].trim());
      if (eersteRKey !== undefined) {
        const leegMKey = Object.keys(session[mainKey])
          .sort((a, b) => +a - +b)
          .find(k => !session[mainKey][k] || !session[mainKey][k].trim());
        if (leegMKey !== undefined) {
          session[mainKey][leegMKey] = session[reserveKey][eersteRKey];
          session[reserveKey] = shiftUpDynamic(session[reserveKey], +eersteRKey);
        }
      }
    } else {
      session[reserveKey] = shiftUpDynamic(session[reserveKey], fromIdx);
    }
    return session;
  }, (error, committed, snapshot) => {
    if (committed) {
      const door = isAdmin ? adminNaam : naam;
      logEntry("uitgeschreven", naam, door, { lijstType: listType, positie: fromIdx });
      if (listType === "main" && snapshot) {
        const data = snapshot.val();
        const nieuweMain = data?.[mainKey] || {};
        const origineleMain = Object.values(currentSession?.[mainKey] || {}).filter(n=>n&&n.trim());
        const nieuweMainArr = Object.values(nieuweMain).filter(n=>n&&n.trim());
        const gepromoveerd = nieuweMainArr.find(n => !origineleMain.includes(n) && n !== naam);
        if (gepromoveerd) logEntry("doorgeschoven van reserve naar deelnemers", gepromoveerd, `uitschrijving ${naam}`);
      }
      showToast(t("uitgeschreven", { naam }));
    }
  });
}
```

- [ ] **Step 2: Add the split branch to `confirmClearList()`**

Replace (`inschrijflijst.html:2156-2163`):

```js
  document.getElementById("confirmOkBtn").onclick = () => {
    closeModal("confirmModal");
    const leeg = (size) => { const o = {}; for (let i = 0; i < size; i++) o[i] = ""; return o; };
    const s = eventConfig.mainSize, r = eventConfig.reserveSize;
    const update = isBeideSport()
      ? { main_padel: leeg(s), reserve_padel: leeg(r), main_tennis: leeg(s), reserve_tennis: leeg(r) }
      : { main: leeg(s), reserve: leeg(r) };
    db.ref(`sessies/${currentDate}`).update(update);
    showToast(t("lijstLeeggemaakt"));
  };
```

with:

```js
  document.getElementById("confirmOkBtn").onclick = () => {
    closeModal("confirmModal");
    const leeg = (size) => { const o = {}; for (let i = 0; i < size; i++) o[i] = ""; return o; };
    let update;
    if (isBeideSport()) {
      const s = eventConfig.mainSize, r = eventConfig.reserveSize;
      update = { main_padel: leeg(s), reserve_padel: leeg(r), main_tennis: leeg(s), reserve_tennis: leeg(r) };
    } else if (isSplitSport()) {
      update = { main_a: leeg(eventConfig.blokA.mainSize), reserve_a: null, main_b: leeg(eventConfig.blokB.mainSize), reserve_b: null };
    } else {
      const s = eventConfig.mainSize, r = eventConfig.reserveSize;
      update = { main: leeg(s), reserve: leeg(r) };
    }
    db.ref(`sessies/${currentDate}`).update(update);
    showToast(t("lijstLeeggemaakt"));
  };
```

(`null` in a Firebase `.update()` call deletes that key, which is the correct way to reset an unbounded reserve object back to empty.)

- [ ] **Step 3: Manual verification**

Using the `test_split` session with column A full (20 names) and one person in `reserve_a` (from Task 4's verification): as the "mine" user (set `identity` to one of the main_a names), click that row's "✕" — confirm the reserve person is promoted into the freed main slot, and `reserve_a` shrinks (check `db.ref('sessies/2099-01-02/reserve_a').once('value').then(s=>console.log(s.val()))` — should be `null`/empty after promotion if there was only 1 reserve entry). Add 2 people to `reserve_a` again, remove the first reserve entry directly via a slot's "✕" (as admin) — confirm the second entry shifts to index 0 and the object no longer has a trailing index. Then remove the last remaining reserve entry — confirm the column goes back to showing "Nog geen reserves". Finally, as admin, trigger `confirmClearList()` (find the button that calls it in the admin panel — search the HTML for `onclick="confirmClearList()"` if not obvious from the modal you already have open) against the test split session and confirm `main_a`/`main_b` reset to all-empty and `reserve_a`/`reserve_b` are removed from the session node entirely.

- [ ] **Step 4: Commit**

```bash
git add inschrijflijst.html
git commit -m "Voeg uitschrijf-flow en lijst-leegmaken toe voor split-sport kolommen"
```

---

### Task 6: Vaste deelnemers admin UI for split — `#vdSplitSection`

**Files:**
- Modify: `inschrijflijst.html:694-714` (vaste-deelnemers HTML — add `#vdSplitSection` sibling to `#vdBeideSection`)
- Modify: `inschrijflijst.html:2679-2781` (`laadVasteDeelnemers`, `renderVasteDeelnemers`, `voegVasteDeelnemerToe`, `verwijderVasteDeelnemer`)

**Interfaces:**
- Consumes: `eventConfig.blokA.label`/`blokB.label` (Task 1), `eventConfig.blokA.mainSize`/`blokB.mainSize` (Task 1)
- Produces: fixed-participant lists persisted at `instellingen/vasteDeelnemersPerType/<id>_a` / `_b`, read by `initialiseerSessie()` (Task 2).

- [ ] **Step 1: Add the `#vdSplitSection` HTML**

In `inschrijflijst.html`, right after the closing `</div>` of `#vdBeideSection` (`inschrijflijst.html:714`, the `</div>` that closes the "Beide sport: padel + tennis apart" block, right before the two closing `</div>` for the outer wrapper), add a new sibling section:

```html
          <!-- Split-sport: 2 tijdblokken apart -->
          <div id="vdSplitSection" style="display:none;flex-direction:column;gap:14px;">
            <div>
              <div id="vdSplitLabelA" style="font-size:12px;font-weight:600;color:var(--ink-light);margin-bottom:6px;"></div>
              <div id="vdLijstBlokA" style="display:flex;flex-direction:column;gap:6px;"></div>
              <div style="display:flex;gap:8px;margin-top:6px;">
                <input type="text" id="vdInputBlokA" placeholder="Naam toevoegen…" maxlength="40" autocomplete="off" list="ledenList"
                  style="flex:1;padding:9px 12px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:16px;background:var(--bg);color:var(--ink);outline:none;" />
                <button class="btn-confirm" style="flex:0 0 auto;padding:9px 14px;font-size:14px;" onclick="voegVasteDeelnemerToe('a')">+</button>
              </div>
            </div>
            <div>
              <div id="vdSplitLabelB" style="font-size:12px;font-weight:600;color:var(--ink-light);margin-bottom:6px;"></div>
              <div id="vdLijstBlokB" style="display:flex;flex-direction:column;gap:6px;"></div>
              <div style="display:flex;gap:8px;margin-top:6px;">
                <input type="text" id="vdInputBlokB" placeholder="Naam toevoegen…" maxlength="40" autocomplete="off" list="ledenList"
                  style="flex:1;padding:9px 12px;border:2px solid var(--border);border-radius:10px;font-family:'DM Sans',sans-serif;font-size:16px;background:var(--bg);color:var(--ink);outline:none;" />
                <button class="btn-confirm" style="flex:0 0 auto;padding:9px 14px;font-size:14px;" onclick="voegVasteDeelnemerToe('b')">+</button>
              </div>
            </div>
          </div>
```

- [ ] **Step 2: Extend `laadVasteDeelnemers()` and `renderVasteDeelnemers()`**

Replace (`inschrijflijst.html:2679-2719`):

```js
async function laadVasteDeelnemers(eventType) {
  vdSelectedType = eventType || urlEventType;
  const et = EVENT_TYPES[vdSelectedType];
  const beide = et?.sport === "beide";

  if (beide) {
    const [sp, st] = await Promise.all([
      db.ref(`instellingen/vasteDeelnemersPerType/${vdSelectedType}_padel`).once("value"),
      db.ref(`instellingen/vasteDeelnemersPerType/${vdSelectedType}_tennis`).once("value"),
    ]);
    vdPadel  = Array.isArray(sp.val()) ? sp.val() : [];
    vdTennis = Array.isArray(st.val()) ? st.val() : [];
  } else {
    const snap = await db.ref(`instellingen/vasteDeelnemersPerType/${vdSelectedType}`).once("value");
    vdSingle = Array.isArray(snap.val()) ? snap.val() : [];
    // Sync globale var voor backwards compat (initialiseerSessie leest direct, maar listener update kan verouderd zijn)
    if (vdSelectedType === urlEventType) vasteDeelnemers = vdSingle;
  }
  renderVasteDeelnemers();
}

function renderVasteDeelnemers() {
  const et = EVENT_TYPES[vdSelectedType || urlEventType];
  const label = et?.label?.[lang] || et?.label?.nl || vdSelectedType || urlEventType;
  const beide = et?.sport === "beide";

  const desc = document.getElementById("vasteDeelnemersDesc");
  if (desc) desc.textContent = lang === "en"
    ? `Automatically added when creating a new list for "${label}".`
    : `Worden automatisch toegevoegd bij een nieuwe lijst voor "${label}".`;

  document.getElementById("vdSingleSection").style.display = beide ? "none" : "flex";
  document.getElementById("vdBeideSection").style.display  = beide ? "flex" : "none";

  if (beide) {
    renderVdLijst("vdLijstPadel",  vdPadel,  "padel");
    renderVdLijst("vdLijstTennis", vdTennis, "tennis");
  } else {
    renderVdLijst("vasteDeelnemerLijst", vdSingle, null);
  }
}
```

with:

```js
let vdA = [], vdB = [];

async function laadVasteDeelnemers(eventType) {
  vdSelectedType = eventType || urlEventType;
  const et = EVENT_TYPES[vdSelectedType];
  const beide = et?.sport === "beide";
  const split = et?.sport === "split";

  if (beide) {
    const [sp, st] = await Promise.all([
      db.ref(`instellingen/vasteDeelnemersPerType/${vdSelectedType}_padel`).once("value"),
      db.ref(`instellingen/vasteDeelnemersPerType/${vdSelectedType}_tennis`).once("value"),
    ]);
    vdPadel  = Array.isArray(sp.val()) ? sp.val() : [];
    vdTennis = Array.isArray(st.val()) ? st.val() : [];
  } else if (split) {
    const [sa, sb] = await Promise.all([
      db.ref(`instellingen/vasteDeelnemersPerType/${vdSelectedType}_a`).once("value"),
      db.ref(`instellingen/vasteDeelnemersPerType/${vdSelectedType}_b`).once("value"),
    ]);
    vdA = Array.isArray(sa.val()) ? sa.val() : [];
    vdB = Array.isArray(sb.val()) ? sb.val() : [];
  } else {
    const snap = await db.ref(`instellingen/vasteDeelnemersPerType/${vdSelectedType}`).once("value");
    vdSingle = Array.isArray(snap.val()) ? snap.val() : [];
    // Sync globale var voor backwards compat (initialiseerSessie leest direct, maar listener update kan verouderd zijn)
    if (vdSelectedType === urlEventType) vasteDeelnemers = vdSingle;
  }
  renderVasteDeelnemers();
}

function renderVasteDeelnemers() {
  const et = EVENT_TYPES[vdSelectedType || urlEventType];
  const label = et?.label?.[lang] || et?.label?.nl || vdSelectedType || urlEventType;
  const beide = et?.sport === "beide";
  const split = et?.sport === "split";

  const desc = document.getElementById("vasteDeelnemersDesc");
  if (desc) desc.textContent = lang === "en"
    ? `Automatically added when creating a new list for "${label}".`
    : `Worden automatisch toegevoegd bij een nieuwe lijst voor "${label}".`;

  document.getElementById("vdSingleSection").style.display = (beide || split) ? "none" : "flex";
  document.getElementById("vdBeideSection").style.display  = beide ? "flex" : "none";
  document.getElementById("vdSplitSection").style.display  = split ? "flex" : "none";

  if (beide) {
    renderVdLijst("vdLijstPadel",  vdPadel,  "padel");
    renderVdLijst("vdLijstTennis", vdTennis, "tennis");
  } else if (split) {
    document.getElementById("vdSplitLabelA").textContent = et.blokA.label[lang] || et.blokA.label.nl;
    document.getElementById("vdSplitLabelB").textContent = et.blokB.label[lang] || et.blokB.label.nl;
    renderVdLijst("vdLijstBlokA", vdA, "a");
    renderVdLijst("vdLijstBlokB", vdB, "b");
  } else {
    renderVdLijst("vasteDeelnemerLijst", vdSingle, null);
  }
}
```

- [ ] **Step 3: Extend `voegVasteDeelnemerToe()` and `verwijderVasteDeelnemer()`**

Replace (`inschrijflijst.html:2737-2781`):

```js
function voegVasteDeelnemerToe(sport) {
  const inputId = sport === "padel" ? "vdInputPadel" : sport === "tennis" ? "vdInputTennis" : "vasteDeelnemerInput";
  const inp = document.getElementById(inputId);
  const naam = inp.value.trim();
  if (!naam) { inp.focus(); return; }
  if (!ledenNamen.includes(naam)) {
    inp.style.borderColor = "var(--accent)";
    showToast(t("kiesUitLedenlijst"));
    setTimeout(() => { inp.style.borderColor = ""; }, 2000);
    return;
  }
  const et = EVENT_TYPES[vdSelectedType];
  const maxSize = et?.mainSize || eventConfig.mainSize;

  let arr, path;
  if (sport === "padel")  { arr = vdPadel;  path = `instellingen/vasteDeelnemersPerType/${vdSelectedType}_padel`; }
  else if (sport === "tennis") { arr = vdTennis; path = `instellingen/vasteDeelnemersPerType/${vdSelectedType}_tennis`; }
  else                    { arr = vdSingle; path = `instellingen/vasteDeelnemersPerType/${vdSelectedType}`; }

  if (arr.includes(naam)) { showToast(t("vasteAlInLijst", { naam })); return; }
  if (arr.length >= maxSize) { showToast(t("maximumBereikt", { max: maxSize })); return; }

  arr.push(naam);
  db.ref(path).set(arr).then(() => {
    inp.value = "";
    if (vdSelectedType === urlEventType && !sport) vasteDeelnemers = arr;
    renderVasteDeelnemers();
    showToast(t("vasteToegevoegd", { naam }));
  });
}

function verwijderVasteDeelnemer(index, sport) {
  let arr, path;
  if (sport === "padel")  { arr = vdPadel;  path = `instellingen/vasteDeelnemersPerType/${vdSelectedType}_padel`; }
  else if (sport === "tennis") { arr = vdTennis; path = `instellingen/vasteDeelnemersPerType/${vdSelectedType}_tennis`; }
  else                    { arr = vdSingle; path = `instellingen/vasteDeelnemersPerType/${vdSelectedType}`; }

  const naam = arr[index];
  arr.splice(index, 1);
  db.ref(path).set(arr).then(() => {
    if (vdSelectedType === urlEventType && !sport) vasteDeelnemers = arr;
    renderVasteDeelnemers();
    showToast(t("vasteVerwijderd", { naam }));
  });
}
```

with:

```js
function vdArrPath(sport) {
  const et = EVENT_TYPES[vdSelectedType];
  if (sport === "padel")  return { arr: vdPadel,  path: `instellingen/vasteDeelnemersPerType/${vdSelectedType}_padel`, max: et?.mainSize || eventConfig.mainSize };
  if (sport === "tennis") return { arr: vdTennis, path: `instellingen/vasteDeelnemersPerType/${vdSelectedType}_tennis`, max: et?.mainSize || eventConfig.mainSize };
  if (sport === "a")      return { arr: vdA, path: `instellingen/vasteDeelnemersPerType/${vdSelectedType}_a`, max: et?.blokA?.mainSize || 1 };
  if (sport === "b")      return { arr: vdB, path: `instellingen/vasteDeelnemersPerType/${vdSelectedType}_b`, max: et?.blokB?.mainSize || 1 };
  return { arr: vdSingle, path: `instellingen/vasteDeelnemersPerType/${vdSelectedType}`, max: et?.mainSize || eventConfig.mainSize };
}

function voegVasteDeelnemerToe(sport) {
  const inputId = sport === "padel" ? "vdInputPadel" : sport === "tennis" ? "vdInputTennis"
    : sport === "a" ? "vdInputBlokA" : sport === "b" ? "vdInputBlokB" : "vasteDeelnemerInput";
  const inp = document.getElementById(inputId);
  const naam = inp.value.trim();
  if (!naam) { inp.focus(); return; }
  if (!ledenNamen.includes(naam)) {
    inp.style.borderColor = "var(--accent)";
    showToast(t("kiesUitLedenlijst"));
    setTimeout(() => { inp.style.borderColor = ""; }, 2000);
    return;
  }
  const { arr, path, max: maxSize } = vdArrPath(sport);

  if (arr.includes(naam)) { showToast(t("vasteAlInLijst", { naam })); return; }
  if (arr.length >= maxSize) { showToast(t("maximumBereikt", { max: maxSize })); return; }

  arr.push(naam);
  db.ref(path).set(arr).then(() => {
    inp.value = "";
    if (vdSelectedType === urlEventType && !sport) vasteDeelnemers = arr;
    renderVasteDeelnemers();
    showToast(t("vasteToegevoegd", { naam }));
  });
}

function verwijderVasteDeelnemer(index, sport) {
  const { arr, path } = vdArrPath(sport);
  const naam = arr[index];
  arr.splice(index, 1);
  db.ref(path).set(arr).then(() => {
    if (vdSelectedType === urlEventType && !sport) vasteDeelnemers = arr;
    renderVasteDeelnemers();
    showToast(t("vasteVerwijderd", { naam }));
  });
}
```

- [ ] **Step 4: Manual verification**

As simulated admin (`isAdmin=true; adminEventTypes="*";`), open "👥 Vaste deelnemers beheren", select `Test Split` in the dropdown. Confirm the section shows two labeled blocks reading the exact `blokA`/`blokB` label text from Task 1's test event type, each with its own input + list. Add a name to block A's input (must be a real name from `ledenNamen` — check `ledenNamen[0]` in console if unsure which name to type), confirm it appears in the block A list, confirm via `db.ref('instellingen/vasteDeelnemersPerType/test_split_a').once('value').then(s=>console.log(s.val()))` that it persisted. Add 20 names to block A (its `mainSize`) and confirm the 21st attempt shows the "maximum bereikt" toast. Remove one name via its "✕" and confirm it's gone from both the UI and Firebase.

- [ ] **Step 5: Commit**

```bash
git add inschrijflijst.html
git commit -m "Voeg vaste-deelnemers beheer toe voor split-sport blokken"
```

---

### Task 7: Export/share functions — `downloadCsv()`, `maakScreenshot()`, `copyWhatsApp()` split branches

**Files:**
- Modify: `inschrijflijst.html:3013-3024` (`copyWhatsApp`), `inschrijflijst.html:3045-3119` (`maakScreenshot`), `inschrijflijst.html:3121-3136` (`downloadCsv`)

**Interfaces:**
- Consumes: `currentSession.main_a`/`main_b`/`reserve_a`/`reserve_b`, `eventConfig.blokA`/`blokB` (Tasks 1-2), `reserveArray()` (Task 3)

- [ ] **Step 1: Split-aware `copyWhatsApp()`**

Replace (`inschrijflijst.html:3013-3024`):

```js
function copyWhatsApp() {
  if (!currentSession) return;
  const main    = slotArray(currentSession.main,    eventConfig.mainSize);
  const reserve = slotArray(currentSession.reserve, eventConfig.reserveSize);
  let lines = [`${currentSession.title} · ${currentSession.time}:`, ""];
  main.forEach((n, i) => lines.push(`${i+1}. ${n || ""}`));
  lines.push("— reserve —");
  reserve.forEach((n, i) => lines.push(`${i+1}. ${n || ""}`));
  navigator.clipboard.writeText(lines.join("\n"))
    .then(() => showToast(t("gekopieerd")))
    .catch(() => showToast(t("kopieerenMislukt")));
}
```

with:

```js
function copyWhatsApp() {
  if (!currentSession) return;
  let lines = [`${currentSession.title} · ${currentSession.time}:`, ""];
  if (isSplitSport()) {
    const blokA = eventConfig.blokA, blokB = eventConfig.blokB;
    lines.push(`— ${blokA.label[lang] || blokA.label.nl} —`);
    slotArray(currentSession.main_a, blokA.mainSize).forEach((n, i) => lines.push(`${i+1}. ${n || ""}`));
    lines.push("— reserve —");
    reserveArray(currentSession.reserve_a).forEach((n, i) => lines.push(`${i+1}. ${n || ""}`));
    lines.push("", `— ${blokB.label[lang] || blokB.label.nl} —`);
    slotArray(currentSession.main_b, blokB.mainSize).forEach((n, i) => lines.push(`${i+1}. ${n || ""}`));
    lines.push("— reserve —");
    reserveArray(currentSession.reserve_b).forEach((n, i) => lines.push(`${i+1}. ${n || ""}`));
  } else {
    const main    = slotArray(currentSession.main,    eventConfig.mainSize);
    const reserve = slotArray(currentSession.reserve, eventConfig.reserveSize);
    main.forEach((n, i) => lines.push(`${i+1}. ${n || ""}`));
    lines.push("— reserve —");
    reserve.forEach((n, i) => lines.push(`${i+1}. ${n || ""}`));
  }
  navigator.clipboard.writeText(lines.join("\n"))
    .then(() => showToast(t("gekopieerd")))
    .catch(() => showToast(t("kopieerenMislukt")));
}
```

- [ ] **Step 2: Split-aware `downloadCsv()`**

Replace (`inschrijflijst.html:3121-3136`):

```js
function downloadCsv() {
  if (!currentSession) return;
  const main    = slotArray(currentSession.main,    eventConfig.mainSize).filter(n => n && n.trim());
  const reserve = slotArray(currentSession.reserve, eventConfig.reserveSize).filter(n => n && n.trim());
  const titel   = currentSession.title || currentDate;
  const sep     = ";"; // puntkomma voor Nederlandse Excel
  let rows = [["Nr", "Naam", "Lijst"]];
  main.forEach((n, i)    => rows.push([i + 1, n, lang === "en" ? "Main" : "Hoofdlijst"]));
  reserve.forEach((n, i) => rows.push([i + 1, n, lang === "en" ? "Reserve" : "Reserve"]));
  const csv = "﻿" + rows.map(r => r.map(v => `"${String(v).replace(/"/g, '""')}"`).join(sep)).join("\r\n");
  const a = document.createElement("a");
  a.href = URL.createObjectURL(new Blob([csv], { type: "text/csv;charset=utf-8;" }));
  a.download = `padel-${currentDate}.csv`;
  a.click();
  showToast(t("csvGedownload"));
}
```

with:

```js
function downloadCsv() {
  if (!currentSession) return;
  const sep = ";"; // puntkomma voor Nederlandse Excel
  let rows = [["Nr", "Naam", "Blok", "Lijst"]];
  if (isSplitSport()) {
    const blokA = eventConfig.blokA, blokB = eventConfig.blokB;
    const labelA = blokA.label[lang] || blokA.label.nl, labelB = blokB.label[lang] || blokB.label.nl;
    slotArray(currentSession.main_a, blokA.mainSize).filter(n => n && n.trim())
      .forEach((n, i) => rows.push([i + 1, n, labelA, lang === "en" ? "Main" : "Hoofdlijst"]));
    reserveArray(currentSession.reserve_a).filter(n => n && n.trim())
      .forEach((n, i) => rows.push([i + 1, n, labelA, "Reserve"]));
    slotArray(currentSession.main_b, blokB.mainSize).filter(n => n && n.trim())
      .forEach((n, i) => rows.push([i + 1, n, labelB, lang === "en" ? "Main" : "Hoofdlijst"]));
    reserveArray(currentSession.reserve_b).filter(n => n && n.trim())
      .forEach((n, i) => rows.push([i + 1, n, labelB, "Reserve"]));
  } else {
    const main    = slotArray(currentSession.main,    eventConfig.mainSize).filter(n => n && n.trim());
    const reserve = slotArray(currentSession.reserve, eventConfig.reserveSize).filter(n => n && n.trim());
    main.forEach((n, i)    => rows.push([i + 1, n, "", lang === "en" ? "Main" : "Hoofdlijst"]));
    reserve.forEach((n, i) => rows.push([i + 1, n, "", "Reserve"]));
  }
  const csv = "﻿" + rows.map(r => r.map(v => `"${String(v).replace(/"/g, '""')}"`).join(sep)).join("\r\n");
  const a = document.createElement("a");
  a.href = URL.createObjectURL(new Blob([csv], { type: "text/csv;charset=utf-8;" }));
  a.download = `padel-${currentDate}.csv`;
  a.click();
  showToast(t("csvGedownload"));
}
```

- [ ] **Step 3: Split-aware `maakScreenshot()`**

Replace the body between `const main    = slotArray(currentSession.main,    eventConfig.mainSize);` and `el.appendChild(listEl);` (`inschrijflijst.html:3061-3092`) with a branch. Full replacement of `inschrijflijst.html:3061-3092`:

```js
  const listEl = document.createElement("div");
  listEl.style.cssText = "background:#fffef9;border:1px solid #ddd8cc;border-top:none;border-radius:0 0 12px 12px;overflow:hidden;margin-bottom:12px;";

  const appendRow = (i, naam, kleurFilled, kleurLeeg) => {
    const row = document.createElement("div");
    const filled = naam && naam.trim();
    row.style.cssText = `display:flex;align-items:center;padding:10px 16px;gap:12px;border-bottom:1px solid #ddd8cc;`;
    row.innerHTML = `
      <span style="font-size:13px;color:#6b6555;width:20px;text-align:right;">${i + 1}</span>
      <span style="font-size:14px;font-weight:${filled ? 500 : 300};color:${filled ? kleurFilled : "#ddd8cc"};font-style:${filled ? "normal" : "italic"};">${filled ? naam : "Vrij"}</span>`;
    listEl.appendChild(row);
  };
  const appendDivider = (tekst) => {
    const divider = document.createElement("div");
    divider.style.cssText = "text-align:center;padding:6px;font-size:11px;color:#6b6555;background:#f5f0e8;border-bottom:1px solid #ddd8cc;";
    divider.textContent = tekst;
    listEl.appendChild(divider);
  };

  if (isSplitSport()) {
    const blokA = eventConfig.blokA, blokB = eventConfig.blokB;
    appendDivider(blokA.label[lang] || blokA.label.nl);
    slotArray(currentSession.main_a, blokA.mainSize).forEach((n, i) => appendRow(i, n, "#1a1810"));
    appendDivider("Reserve");
    const resA = reserveArray(currentSession.reserve_a);
    if (!resA.length) appendRow(0, "", "#6b6555");
    else resA.forEach((n, i) => appendRow(i, n, "#6b6555"));
    appendDivider(blokB.label[lang] || blokB.label.nl);
    slotArray(currentSession.main_b, blokB.mainSize).forEach((n, i) => appendRow(i, n, "#1a1810"));
    appendDivider("Reserve");
    const resB = reserveArray(currentSession.reserve_b);
    if (!resB.length) appendRow(0, "", "#6b6555");
    else resB.forEach((n, i) => appendRow(i, n, "#6b6555"));
  } else {
    const main    = slotArray(currentSession.main,    eventConfig.mainSize);
    const reserve = slotArray(currentSession.reserve, eventConfig.reserveSize);
    main.forEach((naam, i) => appendRow(i, naam, "#1a1810"));
    appendDivider("Reserve (max 4)");
    reserve.forEach((naam, i) => appendRow(i, naam, "#6b6555"));
  }

  el.appendChild(listEl);
```

- [ ] **Step 4: Manual verification**

Using the `test_split` session with a few names filled in Task 4-5's testing, as admin open "📤 Exporteren & Screenshot":
- Click "📥 Excel (CSV)", confirm a CSV downloads (check the download in the Browser tool's downloads, or read the blob content via console before `a.click()` if the tool can't retrieve downloaded files easily — alternative: temporarily comment out `a.click()` and instead log `csv` to console to inspect content, then revert). Confirm it has a "Blok" column with the correct block labels and both main+reserve entries from both columns.
- Click "📸 Screenshot", confirm the preview image shows both blocks stacked with their own dividers and reserve sections (open `screenshotImg`'s `src` via `document.getElementById('screenshotImg').src` and view it, or take a Browser screenshot of the admin panel after generation).
- Call `copyWhatsApp()` from console, then `navigator.clipboard.readText().then(t=>console.log(t))` to confirm the copied text lists both blocks with headers.

- [ ] **Step 5: Commit**

```bash
git add inschrijflijst.html
git commit -m "Maak CSV-export, screenshot en WhatsApp-kopieertekst split-bewust"
```

---

### Task 8: WhatsApp preview text + `shareLink()` message

**Files:**
- Modify: `inschrijflijst.html:9-10` (`og:title`/`og:description`)
- Modify: `inschrijflijst.html:3026-3040` (`shareLink`)

**Interfaces:**
- Consumes: `eventConfig.blokA`/`blokB` (Task 1)

- [ ] **Step 1: Update the static WhatsApp preview meta tags**

Replace (`inschrijflijst.html:9-10`):

```html
<meta property="og:title"       content="🎾 Vrijdagochtend padel – Schrijf je in!" />
<meta property="og:description" content="De inschrijving is geopend! Tik op de link, kies je naam en doe mee. Vol = vol!" />
```

with:

```html
<meta property="og:title"       content="🎾 Vrijdagochtend padel – Schrijf je in!" />
<meta property="og:description" content="De inschrijving is geopend! Kies 9:00-11:30 of 11:30-12:00 en doe mee. Vol = vol!" />
```

- [ ] **Step 2: Make `shareLink()` mention both blocks for split sessions**

Replace (`inschrijflijst.html:3026-3029`):

```js
function shareLink() {
  const titel  = currentSession?.title || "Padel";
  const tijd   = currentSession?.time  || "";
  const tekst  = `🎾 ${titel} · ${tijd}\nDe inschrijving is geopend – schrijf je snel in, vol = vol!\nSign-up is open – register quickly, spots fill up fast!`;
```

with (the block labels already contain both the time and the capacity, e.g. "9:00 - 11:30 (20 plekken)", so the message lists each label once):

```js
function shareLink() {
  const titel  = currentSession?.title || "Padel";
  const tijd   = currentSession?.time  || "";
  let tekst;
  if (isSplitSport()) {
    const blokA = eventConfig.blokA, blokB = eventConfig.blokB;
    tekst = `🎾 ${titel}\n${blokA.label.nl}\n${blokB.label.nl}\nDe inschrijving is geopend – schrijf je snel in, vol = vol!\nSign-up is open – register quickly, spots fill up fast!`;
  } else {
    tekst = `🎾 ${titel} · ${tijd}\nDe inschrijving is geopend – schrijf je snel in, vol = vol!\nSign-up is open – register quickly, spots fill up fast!`;
  }
```

- [ ] **Step 3: Manual verification**

View the page source (`document.title` and `document.querySelector('meta[property="og:description"]').content` in console) and confirm the new description text. On the `test_split` session, call `shareLink()` from console (with `navigator.share` unavailable in the automated browser, it falls back to clipboard) then `navigator.clipboard.readText().then(t=>console.log(t))` and confirm the text lists both block labels on separate lines.

- [ ] **Step 4: Commit**

```bash
git add inschrijflijst.html
git commit -m "Werk WhatsApp-tekst bij voor 2 tijdblokken op vrijdag"
```

---

### Task 9: Cutover — apply the real vrijdagochtend config in production Firebase

**⚠️ Checkpoint — do not run this task's Firebase writes without the user explicitly confirming timing.** This changes the live `vrijdagochtend` event type that real participants see. It should happen right before creating the next week's list (not mid-week while a session is active in the old shape), and must be followed immediately by creating + activating a fresh split-shaped session for the upcoming Friday — an already-active old-shaped session (`main`/`reserve` keys) will not render correctly once the event type's `sport` flips to `"split"`.

**Files:** none (Firebase data only, via the app's own admin UI)

- [ ] **Step 1: Confirm timing with the user**

Ask: "Is now a good time to switch vrijdagochtend to the split layout? This should happen right after this week's list is done, and I'll create + activate next Friday's list in the new format immediately after." Wait for explicit go-ahead before Step 2.

- [ ] **Step 2: Edit the real `vrijdagochtend` event type**

As the real superadmin (not the console `isAdmin` hack — use the actual admin login), open "⚙️ Beheer → Evenement types", edit "Vrijdagochtend hussel" (or whatever its current label is), set sport to "🎾 Padel (gesplitst in 2 tijdvakken)", and fill:
- Blok A: label NL `9:00 - 11:30 (20 plekken)`, EN `9:00 - 11:30 (20 spots)`, max `20`
- Blok B: label NL `11:30 - 12:00 (12 plekken)`, EN `11:30 - 12:00 (12 spots)`, max `12`

Save.

- [ ] **Step 3: Migrate existing vaste deelnemers (optional, ask the user)**

Ask whether the current `vasteDeelnemersPerType/vrijdagochtend` fixed-participant list should carry over to Blok A. If yes, in "👥 Vaste deelnemers beheren" for vrijdagochtend, read the existing single list and re-add each name under Blok A (the UI's "+"; there's no automatic migration function, so do it by hand through the form, or via a one-off console script reading the old path and writing to `_a`).

- [ ] **Step 4: Create and activate next Friday's split-shaped list**

Use "📅 Nieuwe lijst aanmaken" for vrijdagochtend, pick the upcoming Friday date, click "Aanmaken ✓", then "✓ Maak lijst actief". Confirm the live list now renders as 2 columns.

---

### Task 10: Deploy

**⚠️ Checkpoint — confirm with the user before pushing.** This publishes the code change to the live GitHub Pages site (origin/main), visible to everyone using the app.

- [ ] **Step 1: Review the full diff**

```bash
git log --oneline main..HEAD
git diff origin/main..HEAD -- inschrijflijst.html
```

- [ ] **Step 2: Push, only after explicit user go-ahead**

```bash
git push origin main
```

---

### Task 11: Clean up test artifacts

**Files:** none (Firebase data only)

- [ ] **Step 1: Remove the test event type and test sessions from Firebase**

In the browser console, against the live app:

```js
db.ref('instellingen/eventTypes/test_split').remove();
db.ref('instellingen/vasteDeelnemersPerType/test_split_a').remove();
db.ref('instellingen/vasteDeelnemersPerType/test_split_b').remove();
db.ref('sessies/2099-01-02').remove();
```

- [ ] **Step 2: Confirm cleanup**

```js
db.ref('instellingen/eventTypes/test_split').once('value').then(s => console.log(s.val()));
```

Expect `null`.
