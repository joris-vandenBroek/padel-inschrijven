# Padel Inschrijflijst

Realtime inschrijfformulier voor wekelijkse padelsessies. Deelnemers openen de link, kiezen hun naam en schrijven zich in. Alle wijzigingen zijn direct zichtbaar voor iedereen.

**Live:** https://joris-vandenbroek.github.io/padel-inschrijven/inschrijflijst.html

## Functionaliteit

- 20 plekken hoofdlijst + 4 reserveplekken
- Automatisch doorschuiven van reserve naar hoofdlijst bij uitschrijving
- Naam wordt onthouden op het apparaat (localStorage) — eenmalig invoeren
- Ledenlijst wordt wekelijks automatisch opgehaald uit de KNLTB-ledenregistratie

## Beheermodus

Klik op **⚙️ Beheer** en voer het wachtwoord in. Als beheerder kun je:
- Deelnemers toevoegen of verwijderen
- Een nieuwe lege lijst aanmaken voor de volgende vrijdag
- Een eerder aangemaakte lijst openen
- De ledenlijst verversen

## Technisch

| Onderdeel | Keuze |
|---|---|
| Hosting | GitHub Pages |
| Database | Firebase Realtime Database |
| Ledenlijst | `leden.json` uit [knltb-autoboek](https://github.com/joris-vandenBroek/knltb-autoboek) |
| Frontend | Vanilla HTML/CSS/JS, geen framework |

De app gebruikt Firebase-transacties voor gelijktijdige inschrijvingen, zodat twee mensen niet dezelfde plek kunnen claimen.

## Firebase instellen

1. Maak een project aan op [console.firebase.google.com](https://console.firebase.google.com)
2. Schakel **Realtime Database** in (regio: europe-west1)
3. Kopieer de web-app config en vul deze in bovenaan `inschrijflijst.html` bij `FIREBASE_CONFIG`
4. Stel de database-regels in op lezen/schrijven voor iedereen:
```json
{ "rules": { ".read": true, ".write": true } }
```

## Nieuwe week

Als beheerder: klik **⚙️ Beheer → Nieuwe lege lijst aanmaken**, kies de volgende vrijdag en bevestig. De link blijft hetzelfde.
