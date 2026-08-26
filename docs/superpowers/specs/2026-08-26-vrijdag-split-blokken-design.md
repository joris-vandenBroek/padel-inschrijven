# Vrijdag inschrijflijst opsplitsen in 2 tijdblokken

## Doel

De vrijdagochtend-lijst wordt, net als de dinsdag-lijst, getoond als 2 kolommen naast
elkaar. In tegenstelling tot dinsdag (padel + tennis, gelijke grootte) zijn de twee
vrijdag-kolommen dezelfde sport (padel) maar met **verschillende tijden en
verschillende capaciteit**:

- Linkerkolom: 9:00 – 11:30 uur, max 20 deelnemers
- Rechterkolom: 11:30 – 12:00 uur, max 12 deelnemers
- Beide kolommen: onbeperkte reservelijst (geen maximum)

De dinsdag-lijst (padel/tennis, "beide"-sport) moet volledig ongewijzigd blijven
werken.

## Aanpak: nieuw, geïsoleerd sport-type "split"

De bestaande `"beide"`-sport (padel/tennis, gedeelde grootte, gedeelde
reserve-cap) wordt **niet aangepast**. In plaats daarvan komt er een volledig
nieuw sport-type `"split"` met eigen render-, join-, uitschrijf- en
admin-functies. Geen enkele bestaande `beide`/enkelvoudige functie wordt
gewijzigd — alleen op de centrale dispatch-punten (`render()`,
`initialiseerSessie()`, lijst-resetten, admin-toevoegen) komt één extra
`else if`-tak bij.

Reden: de huidige "beide"-implementatie gaat overal uit van **gedeelde**
`mainSize`/`reserveSize` voor beide kolommen (zie `renderBeide`,
`doJoinTransaction`, `confirmAdminAdd`, vaste-deelnemers-beheer). Vrijdag heeft
per kolom een andere grootte en een onbeperkte reserve — dat past niet in de
bestaande aannames. Een losse implementatie is veiliger dan de bestaande
`beide`-code generiek maken, en sluit uit dat dinsdag per ongeluk verandert.

## Datamodel

### Event type config (Firebase `instellingen/eventTypes/vrijdagochtend`)

```
sport: "split"
blokA: { label: {nl, en}, mainSize }   // bv. "9:00 - 11:30 (20 plekken)", 20
blokB: { label: {nl, en}, mainSize }   // bv. "11:30 - 12:00 (12 plekken)", 12
```

Bestaande velden (`dag`, `startTijd`, `eindTijd`, `verloopUur`, `label`,
`eyebrow`, `shareUrl`) blijven ongewijzigd van betekenis — `startTijd`/`eindTijd`
dekken de volledige sessie (9:00–12:00) voor het venstertekstveld
("headerTime") en de sluitingslogica (`verloopUur`).

Er is voor dit sport-type geen `mainSize`/`reserveSize` op het top-level meer
nodig; die blijven wel bestaan als *ongebruikte* velden voor `padel`/`tennis`/
`beide` types (geen migratie nodig, want alleen `vrijdagochtend` wisselt van
sport-type).

### Sessie-data (Firebase `sessies/<datum>`)

```
main_a:    {0..mainSizeA-1: naam}   // vaste grootte, zoals nu
reserve_a: {0..N-1: naam}           // groeit/krimpt, geen cap
main_b:    {0..mainSizeB-1: naam}
reserve_b: {0..N-1: naam}
```

Suffixes `_a`/`_b` zijn vast in de code (niet afgeleid van het label), zodat
labels later vrij aangepast kunnen worden zonder databasemigratie.

### Vaste deelnemers (Firebase `instellingen/vasteDeelnemersPerType`)

```
vrijdagochtend_a: ["Naam", …]
vrijdagochtend_b: ["Naam", …]
```

## Nieuwe functies (parallel aan de bestaande beide-functies)

- `isSplitSport()` → `eventConfig.sport === "split"`
- `renderSplit()` — bouwt dezelfde 2-koloms grid-CSS (`.beide-kolommen`) als
  `renderBeide()`, maar:
  - kolomkoppen tonen het label uit `blokA.label`/`blokB.label` (bv.
    "9:00-11:30 (20 plekken)") in plaats van sportnaam + gekleurde stip
  - elke kolom heeft zijn eigen `mainSize`
  - reservelijst per kolom toont alleen bestaande entries (geen opgevulde
    "vrij"-plekken tot een maximum); is een kolom se reserve leeg, dan toont
    die kolom één grijze, cursieve rij "Nog geen reserves" / "No reserves yet"
    (zelfde stijl als de huidige "vrij"-tekst)
  - "Doe mee"-knoppen per kolom worden — net als bij `renderBeide` — in de
    kolomkop zelf gegenereerd (niet de losse knoppen in de actiebalk)
- `createSplitSlot(naam, i, listType, blokKey, verlopen)` — kloon van
  `createBeideSlot`, met dezelfde admin-verwijderknop/mine-tag-logica
- `joinSplitFirstFree(blokKey)`, `doJoinSplitTransaction(blokKey)` — eigen
  inschrijf-transactie: loop tot `mainSize` van dát blok; geen plek? dan altijd
  naar reserve (geen capaciteitscheck, want onbeperkt) door de eerstvolgende
  vrije integer-index te pakken (`Object.keys(reserve).length`)
- `doRemoveSplitTransaction(naam, listType, idx, blokKey)`,
  `shiftUpDynamic(obj, fromIdx)` — voor reserve: schuift op én **verkleint**
  het object (verwijdert de laatste key) in plaats van op te vullen tot een
  vaste grootte, zodat de reservelijst precies zo lang is als het aantal
  ingeschreven personen

## Wijzigingen op bestaande dispatch-functies (additief)

- `render()` — extra tak: `if (isSplitSport()) renderSplit();`
  (`doeMeeBtn`/`iemandAndersBtn` verborgen, zoals nu al bij `beide`)
- `initialiseerSessie()` — extra tak die `main_a`/`reserve_a`/`main_b`/
  `reserve_b` aanmaakt en vaste deelnemers laadt uit `..._a`/`..._b`
- Admin "lijst leegmaken" — extra tak die de 4 split-velden reset
- `confirmAdminAdd()` — extra branch (vóór de bestaande logica, met `return`)
  wanneer het beheerder-toevoegen een split-kolom betreft: per-kolom
  `mainSize`, onbeperkte reserve, geen wijziging aan de bestaande
  enkel/beide-paden erna

`confirmAndereInschrijven()` (publieke "iemand anders inschrijven") ondersteunt
vandaag al geen `beide` — die knop is voor `beide`-sessies al verborgen. Voor
`split` wordt dezelfde keuze gemaakt: knop blijft verborgen, inschrijven van
anderen kan (net als bij dinsdag) alleen via de beheerder.

## Admin UI: Evenement types

`etSport`-dropdown krijgt een 3e optie ("🎾 Padel (gesplitst in 2 tijdvakken)").
Er komt voor het eerst een `onchange`-handler op dat veld die een nieuwe
sectie toont/verbergt met:

- Blok A: label NL, label EN, max aantal
- Blok B: label NL, label EN, max aantal

De bestaande "Max deelnemers"/"Max reserve"-velden worden verborgen zodra
`"split"` gekozen is (reserve is bij dit type altijd onbeperkt, dus dat veld
is niet van toepassing). `openEventTypeForm()`/`slaEventTypeOp()` lezen/
schrijven `blokA`/`blokB` wanneer sport `"split"` is.

## Admin UI: Vaste deelnemers

Nieuwe sectie `#vdSplitSection` (zelfde opzet als de bestaande padel/tennis
sectie), met de kop-labels dynamisch uit `blokA.label`/`blokB.label` in plaats
van hardcoded "Padel"/"Tennis". Opslag onder `vrijdagochtend_a`/`_b`.
`voegVasteDeelnemerToe`/`verwijderVasteDeelnemer` krijgen een branch voor
`sport === 'a' | 'b'` met per-kolom `mainSize` als cap.

## Export/deel-functies (CSV, screenshot, WhatsApp-kopieertekst)

Onderzoek wees uit dat deze drie functies vandaag al **niet** rekening houden
met `beide`-sessies (ze lezen alleen de enkelvoudige velden — voor dinsdag dus
al een bestaand gat, buiten scope om te herstellen). Omdat vrijdag deze
export/deel-flow wél actief gebruikt (wekelijks WhatsApp-screenshot), krijgen
alle drie een split-bewuste variant die beide kolommen correct toont/exporteert.

## WhatsApp-tekst

- `og:title`/`og:description` in de `<head>` (bepalen de linkpreview voor de
  basis-URL, dus voor vrijdag): omschrijving wordt aangepast om de 2 blokken te
  noemen, bv.:
  - NL: "De inschrijving is geopend! Kies 9:00-11:30 of 11:30-12:00 en doe mee.
    Vol = vol!"
  - EN: vergelijkbare Engelse tekst
- `shareLink()` (het bericht dat daadwerkelijk gedeeld wordt): voor
  `split`-sessies wordt de gegenereerde tekst uitgebreid zodat beide blokken en
  hun tijden expliciet genoemd worden, in plaats van alleen de generieke
  "De inschrijving is geopend"-tekst.

## Niet in scope

- Dinsdag (`beide`-sport, padel/tennis) — geen enkele wijziging.
- Het generiek maken van `beide` zelf.
- Het herstellen van het bestaande CSV/screenshot-gat voor `beide`-sessies.
