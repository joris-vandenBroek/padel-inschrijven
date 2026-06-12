# Padel Inschrijflijst

Realtime inschrijfformulier voor wekelijkse padelsessies. Deelnemers openen de link, kiezen hun naam en schrijven zich in. Alle wijzigingen zijn direct zichtbaar voor iedereen.

**Live:** https://tinyurl.com/padel-inschrijven

## Functionaliteit

- 20 plekken hoofdlijst + 4 reserveplekken
- Automatisch doorschuiven van reserve naar hoofdlijst bij uitschrijving
- Naam kiezen uit de ledenlijst — vrije invoer is niet toegestaan
- Iemand anders inschrijven via "Iemand anders inschrijven" knop (ook alleen uit ledenlijst)
- Naam wordt onthouden op het apparaat (localStorage) — eenmalig invoeren
- Ledenlijst wordt wekelijks automatisch opgehaald uit de KNLTB-ledenregistratie
- Realtime sync via Firebase — alle gebruikers zien wijzigingen direct

## Beheermodus

Klik op **⚙️ Beheer** en voer het wachtwoord in. Meerdere beheerders kunnen elk hun eigen wachtwoord hebben. Als beheerder kun je:

- Deelnemers toevoegen of verwijderen (beheerder mag ook namen invullen die niet in de ledenlijst staan)
- Een eerdere lijst bekijken via "Andere lijst openen" — zonder dat dit de actieve lijst wijzigt
- Een nieuwe lijst aanmaken voor de volgende vrijdag, en daarna pas activeren wanneer je klaar bent om te delen
- Vaste deelnemers beheren — worden automatisch toegevoegd bij elke nieuwe lijst
- Beheerders toevoegen en wachtwoorden wijzigen
- De ledenlijst verversen

## Technisch

| Onderdeel | Keuze |
|---|---|
| Hosting | GitHub Pages |
| Database | Firebase Realtime Database |
| Ledenlijst | `leden.json` uit [knltb-autoboek](https://github.com/joris-vandenBroek/knltb-autoboek) |
| Frontend | Vanilla HTML/CSS/JS, geen framework |

De app gebruikt Firebase-transacties voor gelijktijdige inschrijvingen, zodat twee mensen niet dezelfde plek kunnen claimen.

### Firebase datastructuur

```
instellingen/
  datumActieveLijst: "YYYY-MM-DD"  ← actieve sessie voor alle gebruikers
  admins/
    "Naam": "wachtwoord"         ← één entry per beheerder
  vasteDeelnemers: ["Naam", …]   ← automatisch toegevoegd bij nieuwe lijst

sessies/
  YYYY-MM-DD/
    title: "Vrijdag X juni"
    time: "9:00 – 10:30 uur"
    aangemaaktOp: timestamp
    main/   { 0..19: naam }      ← max 20 deelnemers
    reserve/ { 0..3: naam }      ← max 4 reservisten
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

1. Klik **⚙️ Beheer → Nieuwe lijst aanmaken**, kies de volgende vrijdag en bevestig. Vaste deelnemers worden automatisch ingevuld.
2. De nieuwe lijst is nog **niet actief** — deelnemers zien de oude lijst totdat jij de nieuwe activeert.
3. Open de nieuwe lijst via **Andere lijst openen**, controleer hem, en klik **✓ Maak lijst actief**.
4. Deel daarna de link via **🔗 Deel link** in WhatsApp. De link blijft hetzelfde voor iedereen.

Een lijst vervalt automatisch op de dag van de sessie om 09:00 uur. Daarna kunnen deelnemers zich niet meer in- of uitschrijven, totdat een nieuwe lijst actief is gemaakt.

## Exporteren

Via **⚙️ Beheer → Exporteren & Screenshot** kun je:

- **Screenshot maken** — slaat een afbeelding van de lijst op, geschikt om in WhatsApp te plaatsen.
- **Download CSV** — downloadt een `.csv`-bestand met alle namen (hoofdlijst + reserve). Open dit bestand in Excel; de kolommen staan klaar om direct te gebruiken.
