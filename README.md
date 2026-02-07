# Qlik Sense Script Library

Wiederverwendbare QVS-Routinen für Qlik Sense Ladescripte.

## Übersicht

| Datei | Beschreibung |
|-------|-------------|
| `calendar.qvs` | Kalender-Generierung mit Mehrsprachigkeit, Fiskaljahr, TimeRanges |
| `logging.qvs` | Strukturiertes Logging mit Severity-Levels und Zeitmessung |

---

## calendar.qvs — Kalender-Routine

Generiert einen vollständigen Masterkalender mit optionalem Feldprefix, Fiskaljahr-Unterstützung und TimeRange-Tabellen.

### Voraussetzungen

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/calendar.qvs);

LET vCalendarStart = num(MakeDate(year(Today()), 01, 01));
LET vCalendarEnd   = num(MakeDate(Year(Today()) + 1, 12, 31));
SET vDataModelLanguage = 'EN';   // oder 'DE'
```

| Variable | Pflicht | Beschreibung |
|----------|---------|-------------|
| `vCalendarStart` | Ja | Startdatum als numerischer Wert |
| `vCalendarEnd` | Ja | Enddatum als numerischer Wert |
| `vDataModelLanguage` | Ja | `'DE'` oder `'EN'` — steuert Feldnamen |
| `vCalendarFM` | Nein | Erster Monat des Fiskaljahres (1-12, Default: 1) |

### Subroutinen

#### `startCalendarService`

Initialisiert den Kalenderservice: validiert `vCalendarFM`, baut die Mehrsprachen-Feldnamen auf.

```qlik
CALL startCalendarService;
```

#### `generateCalendar (vCalendarQualifier)`

Erzeugt drei Tabellen:

- **`calendar`** (bzw. `calendar<Qualifier>`) — Hauptkalender mit allen Feldern
- **`time_ranges`** — Lookup für Zeiträume (YTM, YTD, LTM)
- **`calendar_time_range`** — Verknüpfung Datum ↔ Zeitraum

Der optionale `vCalendarQualifier` wird als Prefix auf alle Feldnamen und Tabellennamen gesetzt. So können mehrere Kalender parallel existieren.

```qlik
CALL generateCalendar;                    // ohne Qualifier
CALL generateCalendar('Order');           // Felder: OrderDatum, OrderJahr, ...
```

**Generierte Felder (Beispiel DE):**

| Kategorie | Felder |
|-----------|--------|
| Datum | Datum, Datum_DE, Datum_EN, Tag, Wochentag |
| Woche | Woche, Woche_aktuell, Woche_letzte, Jahr_KW |
| Monat | Monat, Monat_Nr, Monat_aktuell, Monat_letzter, Jahr_Monat, Jahr_Monatsname |
| Quartal/Jahr | Quartal, Jahr |
| Arbeitstage | Arbeitstag, Wochenende |
| Zeitvergleich | Tage_vergangen, Wochen_vergangen, Monate_vergangen |
| Fiskaljahr | Fiskaljahr, Fiskalquartal, Fiskalmonat, Fiskalmonat_aktuell, Fiskalmonat_letzter |
| Zeiträume | YTD, YTM, LTM |

#### `joinCalendar (targetTable, srcDateField, fieldPrefix)`

Joint reduzierte Kalenderfelder (Jahr, Monat, JahrMonat, YTD) in eine bestehende Faktentabelle per LEFT JOIN.

```qlik
CALL joinCalendar('FactSales', 'OrderDate', 'Order');
// Erzeugt: Order_year, Order_month, Order_year_month, Order_YTD
```

| Parameter | Beschreibung |
|-----------|-------------|
| `targetTable` | Name der Zieltabelle |
| `srcDateField` | Datumsfeld in der Zieltabelle |
| `fieldPrefix` | Prefix für die generierten Felder |

#### `stopCalendarService`

Räumt alle temporären Tabellen und Variablen auf. Immer als letztes aufrufen.

```qlik
CALL stopCalendarService;
```

### Vollständiges Beispiel

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/calendar.qvs);

// Variablen
LET vCalendarStart = num(MakeDate(year(Today()), 01, 01));
LET vCalendarEnd   = num(MakeDate(Year(Today()) + 1, 12, 31));
SET vDataModelLanguage = 'DE';
LET vCalendarFM = 4;   // Fiskaljahr ab April

// Logging
CALL logInit('Mein Dashboard');

// Kalender erzeugen
CALL logStartTimer('Kalender');
CALL startCalendarService;
CALL generateCalendar;
CALL stopCalendarService;
CALL logStopTimer('Kalender');

CALL logRowCount('calendar');
CALL logFinalize;
```

### Fiskaljahr

Wenn `vCalendarFM` gesetzt wird (z.B. `4` für April), werden die Fiskalfelder entsprechend verschoben:

| Kalendermonat | Fiskalmonat (FM=4) | Fiskaljahr |
|---------------|-------------------|------------|
| Jan 2024 | 10 | 2024 |
| Apr 2024 | 1 | 2025 |
| Dez 2024 | 9 | 2025 |
| Mar 2025 | 12 | 2025 |

---

## logging.qvs — Logging-Framework

Strukturiertes Logging für Qlik Sense Ladescripte mit Severity-Levels, Zeitmessung und optionalem QVD-Export.

### Subroutinen

#### `logInit (appName)`

Initialisiert die Log-Tabelle `_log` und startet den Gesamttimer.

```qlik
CALL logInit('Sales Dashboard');
```

#### `logInfo (message)` / `logWarn (message)` / `logError (message)`

Schreibt einen Log-Eintrag mit dem jeweiligen Severity-Level. Jeder Eintrag wird auch per `TRACE` in die Script-Konsole ausgegeben.

```qlik
CALL logInfo('Daten geladen');
CALL logWarn('12 Datensätze ohne PLZ');
CALL logError('Verbindung zur Datenbank fehlgeschlagen');
```

#### `logRowCount (tableName)`

Loggt die Zeilenanzahl einer Tabelle.

```qlik
CALL logRowCount('FactSales');
// -> [INFO] Table [FactSales] has 1234567 rows
```

#### `logStartTimer (stepName)` / `logStopTimer (stepName)`

Misst die Dauer eines Ladeschritts. Die Dauer wird in Sekunden in der Spalte `log_duration_sec` gespeichert.

```qlik
CALL logStartTimer('Load Customers');
// ... Lade-Logik ...
CALL logStopTimer('Load Customers');
// -> [INFO] Timer stopped: Load Customers - duration: 00:01:23
```

#### `logExport (path)`

Speichert die Log-Tabelle als QVD für historische Analyse über mehrere Ladezyklen.

```qlik
CALL logExport('lib://QVD/logs/load_log.qvd');
```

#### `logFinalize`

Loggt die Gesamtdauer und räumt alle Logging-Variablen auf. Die `_log`-Tabelle bleibt im Datenmodell für QA-Dashboards.

```qlik
CALL logFinalize;
```

### Log-Tabelle `_log`

| Spalte | Typ | Beschreibung |
|--------|-----|-------------|
| `log_seq` | Integer | Laufende Nummer |
| `log_timestamp` | Timestamp | Zeitpunkt des Eintrags |
| `log_level` | String | `INFO`, `WARN` oder `ERROR` |
| `log_message` | String | Log-Nachricht |
| `log_app` | String | App-Name aus `logInit` |
| `log_duration_sec` | Number | Dauer in Sekunden (nur bei `logStopTimer`) |

### Vollständiges Beispiel

```qlik
$(Include=lib://Scripts/logging.qvs);

CALL logInit('Sales Dashboard');

CALL logStartTimer('Load Facts');
FactSales:
LOAD * FROM [lib://Data/sales.qvd] (qvd);
CALL logStopTimer('Load Facts');
CALL logRowCount('FactSales');

CALL logStartTimer('Load Customers');
DimCustomer:
LOAD * FROM [lib://Data/customers.qvd] (qvd);
CALL logStopTimer('Load Customers');
CALL logRowCount('DimCustomer');

CALL logExport('lib://QVD/logs/load_log.qvd');
CALL logFinalize;
```
