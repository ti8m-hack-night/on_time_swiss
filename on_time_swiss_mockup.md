# on_time_swiss: Mockup (Material 3)

Wie pünktlich ist der öffentliche Verkehr in der Schweiz? Abfahrtstafel, Verbindungen, Pünktlichkeits-Statistik und Umsteigerisiko.

Interaktives Mockup: https://claude.ai/artifact/TD4W91zTJKHmBeLaSzvb5r (privat, Beispieldaten, keine Live-API)

## Designsystem

- **Basis:** Material Design 3, Roboto und Roboto Mono, Material Symbols
- **Farben:** Seed Schweizer Rot (Primary `#B7131A`, Container `#FFDAD5`), Dark Mode via `prefers-color-scheme`
- **Semantik:** Grün (sicher), Orange (knapp), Rot (kritisch), getrennt vom Primary
- **Navigation:** Bottom Navigation Bar (Mobile), Navigation Rail ab 840 px
- **Komponenten:** Outlined Text Field mit Menü, Filter Chips, Filled/Tonal Button, Cards, Switch, Status-Badges

## Ansicht 1: Abfahrten

| Element | Anforderung |
|---|---|
| Stationssuche mit Vorschlagsmenü | F-2 |
| 20 Abfahrten: Linie, Ziel, Gleis, Soll-Zeit, Verspätung | F-4 |
| Statusbadge: pünktlich, +n min (Orange ab 3, Rot ab 6), Ausfall, Keine Echtzeitdaten | F-5 |
| Cache-Hinweis ("Cache 60 s · vor 12 s geladen"), Refresh-Button | F-3, F-6 |
| Filter Zug, Bus, Tram, Schiff | F-7 |
| Kennzahlen: Abfahrten, Pünktlich, Ø Verspätung, Grösste | F-11 |

## Ansicht 2: Verbindungen

| Element | Anforderung |
|---|---|
| Von, Nach, Datum, Zeit, Tauschen | F-8, F-10 |
| 5 Verbindungen: Abfahrt, Ankunft, Dauer, Umstiege, Verspätung | F-8 |
| Aufklappbar: Teilstrecken mit Gleis, Linie, Zeiten | F-9 |
| Pro Umstieg: Umsteigezeit und Ampel; Gesamtampel = schlechteste | F-14 bis F-16 |
| Alternative bei Rot | F-17 |

## Ansicht 3: Statistik und Einstellungen

- Kennzahlen der aktuellen Tafel (F-11)
- Balken pro Verkehrsmittel: Anteil pünktlich und Ø Verspätung (F-12)
- Prüftabellen mit den Abnahmebeispielen
- Einstellungen: Cache-Dauer, Auto-Aktualisierung, Schalter "Netz getrennt simulieren" (Fehler- und Cache-Meldung, F-3, A-3)

## Berechnungslogik

```js
const toMin = t => { const [h, m] = t.split(':').map(Number); return h * 60 + m; };

function stats(delays) {                       // delays: Minuten oder null
  const real = delays.filter(d => d !== null && d !== undefined);
  const onTime = real.filter(d => d < 3).length;
  return {
    count: delays.length,
    share: real.length ? onTime / real.length : null,
    avg:   real.length ? real.reduce((a, b) => a + b, 0) / real.length : null,
    max:   real.length ? Math.max(...real) : null
  };
}

const transferMinutes = (feederArr, feederDelay, connDep, connDelay) =>
  toMin(connDep) + (connDelay || 0) - (toMin(feederArr) + (feederDelay || 0));

const ampel = m => m >= 5 ? 'grün' : m >= 2 ? 'orange' : 'rot';   // ≤ 1 inkl. negativ = rot
const gesamtAmpel = list => list.includes('rot') ? 'rot' : list.includes('orange') ? 'orange' : 'grün';
```

### Abnahme (im Mockup verifiziert)

| Verspätungen `[0, 0, 2, 3, 10]` | Ergebnis |
|---|---|
| Anzahl | 5 |
| Anteil pünktlich | 60 % (3 von 5) |
| Ø Verspätung | 3,0 min |
| Grösste | 10 min |

| Verspätung Zubringer | Umsteigezeit | Ampel |
|---|---|---|
| 0 min | 7 min | Grün |
| 4 min | 3 min | Orange |
| 6 min | 1 min | Rot |
| 8 min | −1 min | Rot (verpasst) |

## Hinweis

Das Mockup enthält feste Beispieldaten. Für die Umsetzung werden `locations`, `stationboard` und `connections` von transport.opendata.ch mit lokalem Cache (Default 60 s) angebunden.
