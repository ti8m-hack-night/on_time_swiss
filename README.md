# on_time_swiss

Wie pünktlich ist der öffentliche Verkehr in der Schweiz? Abfahrtstafel, Verbindungen, Pünktlichkeits-Statistik und Umsteigerisiko.

> **UX-Prototyp:** _Link folgt (wird vor der Hack Night bereitgestellt)_
> **Zeitrahmen:** 1 Stunde. Priorisiert: zuerst alle **Muss**-Anforderungen, dann **Soll**, dann **Kann**.
> **Technologie, Git:** frei wählbar bzw. optional. Die Anwendung läuft **lokal**, die Daten werden aus dem **Internet** geladen.

## 1. Datenquelle (Pflicht)

[transport.opendata.ch](https://transport.opendata.ch/docs.html), kein API-Key nötig. Community-Dienst mit Rate-Limits: nicht im Sekundentakt abfragen, Antworten zwischenspeichern.

| Zweck | Aufruf |
|---|---|
| Stationssuche | `GET https://transport.opendata.ch/v1/locations?query=Zürich` |
| Abfahrtstafel | `GET https://transport.opendata.ch/v1/stationboard?station=Zürich&limit=20` |
| Verbindungen | `GET https://transport.opendata.ch/v1/connections?from=Bern&to=Zürich&limit=4` |

Verspätungen (`delay`) sind in Minuten angegeben und fehlen oder sind `null`, wenn keine Echtzeitdaten vorliegen. Ein fehlender Wert gilt als 0, wird aber nicht in die Verspätungsstatistik eingerechnet.

## 2. Anforderungen

### Daten laden
- **F-1 (Muss):** Alle Daten kommen live von der API. Es gibt keine fest einkompilierten Fahrplandaten.
- **F-2 (Muss):** Stationen werden über eine Suche mit Vorschlägen gewählt (`locations`).
- **F-3 (Muss):** Antworten werden lokal zwischengespeichert. Die Cache-Dauer ist konfigurierbar (Default 60 s). Bei Fehlern (Netz, Rate-Limit) erscheint eine verständliche Meldung, die App stürzt nicht ab.

### Ansicht 1: Abfahrtstafel
- **F-4 (Muss):** Für eine gewählte Station werden die nächsten 20 Abfahrten angezeigt: Linie (`category` plus `number`), Ziel, Gleis, Soll-Abfahrt, Verspätung in Minuten.
- **F-5 (Muss):** Verspätete Abfahrten sind optisch markiert (z. B. Farbe). Ausgefallene oder nicht verfügbare Echtzeitdaten werden als solche gezeigt.
- **F-6 (Soll):** Die Tafel aktualisiert sich automatisch (Intervall konfigurierbar, mindestens 60 s).
- **F-7 (Kann):** Filter nach Verkehrsmittel (Zug, Bus, Tram, Schiff).

### Ansicht 2: Verbindungen
- **F-8 (Muss):** Zwischen zwei gewählten Stationen werden mindestens 4 Verbindungen mit Abfahrt, Ankunft, Dauer, Anzahl Umstiege und Verspätung angezeigt.
- **F-9 (Muss):** Eine Verbindung lässt sich aufklappen und zeigt alle Teilstrecken (`sections`) mit Gleis, Linie und Zeiten.
- **F-10 (Soll):** Abfahrtszeit und Datum sind wählbar.

### Statistiken
- **F-11 (Muss):** Für die aktuell angezeigte Abfahrtstafel werden berechnet: Anzahl Abfahrten, **Anteil pünktlich** (Verspätung < 3 Minuten), **durchschnittliche Verspätung** (in Minuten, über Abfahrten mit Echtzeitdaten) und **grösste Verspätung**.
- **F-12 (Soll):** Statistik nach Verkehrsmittel aufgeschlüsselt.
- **F-13 (Kann):** Der Verlauf der Pünktlichkeit über mehrere Abfragen im Lauf der Hack Night (lokal gespeichert).

### Berechnung: Umsteigerisiko
- **F-14 (Muss):** Für jede Verbindung mit Umstieg wird pro Umstieg die **verbleibende Umsteigezeit** berechnet:
  `Umsteigezeit = Soll-Abfahrt Anschluss + Verspätung Anschluss − (Soll-Ankunft Zubringer + Verspätung Zubringer)`
- **F-15 (Muss):** Daraus ergibt sich eine Ampel:
  - **Grün**: Umsteigezeit ≥ 5 min
  - **Orange**: 2 bis 4 min
  - **Rot**: höchstens 1 min (inkl. negativ = Anschluss verpasst)
- **F-16 (Muss):** Die Gesamtampel einer Verbindung ist die schlechteste Ampel ihrer Umstiege. Verbindungen ohne Umstieg sind grün.
- **F-17 (Kann):** Alternativverbindung vorschlagen, wenn die Ampel rot ist.

## 3. Abnahmekriterien

Statistik (F-11), Beispiel mit fünf Abfahrten und Verspätungen `[0, 0, 2, 3, 10]`:

| Messgrösse | Erwartung |
|---|---|
| Anzahl Abfahrten | 5 |
| Anteil pünktlich (< 3 min) | 60 % (3 von 5) |
| Durchschnittliche Verspätung | 3.0 min |
| Grösste Verspätung | 10 min |

Umsteigerisiko (F-14 bis F-16), Anschluss Soll-Abfahrt 10:07 ohne Verspätung, Zubringer Soll-Ankunft 10:00:

| Verspätung Zubringer | Umsteigezeit | Ampel |
|---|---|---|
| 0 min | 7 min | Grün |
| 4 min | 3 min | Orange |
| 6 min | 1 min | Rot |
| 8 min | −1 min | Rot (verpasst) |

Weitere Prüfungen:

| # | Prüfung | Erwartung |
|---|---|---|
| A-1 | Station "Bern" suchen und wählen | Abfahrtstafel erscheint mit 20 Einträgen |
| A-2 | Verbindung Bern → Zürich | mindestens 4 Verbindungen mit Ampel |
| A-3 | Netzwerk getrennt, Abfrage bereits einmal geladen | App zeigt Daten aus dem Cache |
| A-4 | Abfahrt ohne `delay`-Wert | wird als "keine Echtzeitdaten" angezeigt und fliesst nicht in die Durchschnittsverspätung ein |

_Zu den Tabellen:_ Berechnungsfunktionen (Statistik, Umsteigezeit, Ampel) sollen ohne API-Zugriff mit festen Werten aufrufbar sein.

## 4. Abgrenzung (nicht gefordert)
Ticketkauf, Login, Karte mit Live-Fahrzeugpositionen, Datenbank, Deployment, CI.
