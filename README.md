# Padel Inschrijflijst

Realtime inschrijfformulier voor wekelijkse padel- en tennissessies. Deelnemers openen de link, kiezen hun naam en schrijven zich in. Alle wijzigingen zijn direct zichtbaar voor iedereen.

**Lokale locatie:** `\\MyCloudEX2Ultra\Transmission\ETV-Volley\padel-inschrijven`  
**Live (vrijdagochtend):** https://tinyurl.com/padel-inschrijven  
**Live (dinsdag Losse Pols):** https://tinyurl.com/LossePols

## Functionaliteit

- Configureerbaar aantal plekken (hoofdlijst + reserve) per event type
- Automatisch doorschuiven van reserve naar hoofdlijst bij uitschrijving
- Naam kiezen uit de ledenlijst — vrije invoer is niet toegestaan
- Naam wordt onthouden op het apparaat (localStorage) — eenmalig invoeren
- Ledenlijst wordt automatisch opgehaald uit de KNLTB-ledenregistratie
- Realtime sync via Firebase — alle gebruikers zien wijzigingen direct
- Meerdere event types (bijv. vrijdagochtend, dinsdagavond) met eigen URL
- Padel & tennis gecombineerd in één sessie (beide-sport modus)
- Meertalig: Nederlands en Engels

## Event types

Elke sessie is gekoppeld aan een event type. Het type bepaalt:

- Naam (NL + EN)
- Koptekst in de header (NL + EN, bijv. "Losse Pols" i.p.v. "Inschrijflijst")
- Dag van de week, start- en eindtijd
- Uur waarop de inschrijflijst sluit op de sessiedag
- Maximaal aantal deelnemers en reservisten
- Sport: Padel, Tennis of Padel & Tennis (twee kolommen)
- Deel-link (optioneel, bijv. tinyurl) — leeg = automatisch afgeleid

De standaard event type is `vrijdagochtend` — de URL zonder `?event=` parameter verwijst altijd hiernaar. Andere types zijn bereikbaar via `?event=<id>`, bijv. `?event=dinsdag_losse_pols`.

### WhatsApp-preview per event type

Elke event type met een eigen tinyurl heeft een bijbehorend HTML-wrapper bestand (bijv. `dinsdag.html`). Dit bestand bevat de juiste `og:title` meta-tag voor de WhatsApp-preview en verwijst meteen door naar de app. De tinyurl wijst naar dit wrapper-bestand.

## Beheermodus

Klik op **⚙️ Beheer** en voer het wachtwoord in. Meerdere beheerders kunnen elk hun eigen wachtwoord hebben. Als beheerder kun je:

- Deelnemers toevoegen of verwijderen
- Iemand anders inschrijven via de **"Iemand anders inschrijven"** knop — ook alleen uit de ledenlijst
- Een eerdere lijst bekijken via "Andere lijst openen" — zonder dat dit de actieve lijst wijzigt
- Een niet-actieve lijst verwijderen via de 🗑 knop in de blauwe balk
- Een nieuwe lijst aanmaken voor de volgende sessiedag, en daarna pas activeren
- **Vaste deelnemers beheren** — worden automatisch toegevoegd bij elke nieuwe lijst, per event type; bij beide-sport apart voor padel en tennis
- **Evenement types beheren** — naam, koptekst, dag/tijd, capaciteit, sport, deel-link instellen
- Beheerders toevoegen en hun event type toegang beheren (alleen superadmin)
- De ledenlijst verversen (wordt automatisch gecacht — verversen is alleen nodig als er nieuwe leden zijn bijgekomen)

### Logboek

In de gouden beheerbalk staat een inklapbaar **📋 Logboek**. Dit toont per sessie een chronologisch overzicht van alle activiteit: wie zich heeft ingeschreven of uitgeschreven, wie automatisch is doorgeschoven van reserve naar de hoofdlijst, en welke beheerder de lijst heeft geopend. Het logboek volgt altijd de lijst die je op dat moment bekijkt.

### Rol-gebaseerde toegang

Elke beheerder heeft toegang tot een of meerdere event types. De superadmin (Joris van den Broek) heeft toegang tot alles en beheert ook andere beheerders en event type configuratie.

- Dropdowns voor "Nieuwe lijst", "Andere lijst openen" en "Vaste deelnemers" tonen alleen de toegestane event types
- "Beheerders" en "Evenement types" secties zijn alleen zichtbaar voor de superadmin
- Een beheerder zonder toegang tot een event type ziet de lijst als normale deelnemer

### Vaste deelnemers per event type

Vaste deelnemers zijn gekoppeld aan een specifiek event type. In het beheerpaneel kies je via een dropdown welk type je beheert. Bij een beide-sport type zijn er aparte lijsten voor padel en tennis.

## Technisch

| Onderdeel | Keuze |
|---|---|
| Hosting | GitHub Pages |
| Database | Firebase Realtime Database (europe-west1) |
| Ledenlijst | `leden.json` uit [knltb-autoboek](https://github.com/joris-vandenBroek/knltb-autoboek) |
| Frontend | Vanilla HTML/CSS/JS, geen framework |

De app gebruikt Firebase-transacties voor gelijktijdige inschrijvingen, zodat twee mensen niet dezelfde plek kunnen claimen.

### Firebase datastructuur

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
    vrijdagochtend/
      id, label {nl, en}, eyebrow {nl, en}, dag, startTijd, eindTijd,
      verloopUur, mainSize, reserveSize, sport, shareUrl?

sessies/
  YYYY-MM-DD/
    title: "Vrijdag X juni"
    time: "9:00 – 10:30 uur"
    eventType: "vrijdagochtend"
    aangemaaktOp: timestamp

    # Enkelvoudig sport:
    main/    { 0..N: naam }
    reserve/ { 0..M: naam }

    # Beide-sport:
    main_padel/    { 0..N: naam }
    reserve_padel/ { 0..M: naam }
    main_tennis/   { 0..N: naam }
    reserve_tennis/{ 0..M: naam }
```

## Firebase instellen

1. Maak een project aan op [console.firebase.google.com](https://console.firebase.google.com)
2. Schakel **Realtime Database** in (regio: europe-west1)
3. Kopieer de web-app config en vul in bovenaan `inschrijflijst.html` bij `FIREBASE_CONFIG`
4. Stel database-regels in:
```json
{ "rules": { ".read": true, ".write": true } }
```

## Nieuwe week

1. Klik **⚙️ Beheer → Nieuwe lijst aanmaken**, kies het event type en de datum en bevestig. Vaste deelnemers worden automatisch ingevuld.
2. De nieuwe lijst is nog **niet actief** — deelnemers zien de oude lijst totdat jij de nieuwe activeert.
3. Open de nieuwe lijst via **Andere lijst openen**, controleer hem, en klik **✓ Maak lijst actief**.
4. Zodra de lijst actief is, verschijnt automatisch een groene balk met een **🔗 Deel link** knop. Gebruik die om de link te kopiëren en te delen via WhatsApp. De link blijft hetzelfde voor iedereen.

Een lijst vervalt automatisch op de dag van de sessie op het ingestelde sluitingsuur.

## Exporteren

Via **⚙️ Beheer → Exporteren & Screenshot** kun je:

- **📸 Screenshot** — slaat een afbeelding van de lijst op, geschikt om in WhatsApp te plaatsen.
- **📤 Deel via WhatsApp** — maakt een screenshot en deelt die direct via WhatsApp.
- **📥 Excel (CSV)** — downloadt een `.csv`-bestand met alle namen (hoofdlijst + reserve), geschikt om te openen in Excel.
