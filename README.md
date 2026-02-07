# Qlik Sense Script Library

Wiederverwendbare QVS-Routinen für Qlik Sense Ladescripte.

## Übersicht

| Datei | Beschreibung |
|-------|-------------|
| `calendar.qvs` | Kalender-Generierung mit Mehrsprachigkeit, Fiskaljahr, TimeRanges |
| `logging.qvs` | Strukturiertes Logging mit Severity-Levels und Zeitmessung |
| `incremental_load.qvs` | Inkrementelles Laden mit QVD-Watermarking (Upsert / Append) |
| `variables.qvs` | Zentrale Variablen-/KPI-Verwaltung mit Set-Analysis-Generator |

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

---

## incremental_load.qvs — Inkrementelles Laden

Wiederverwendbares Framework für Delta-Laden mit QVD-Watermarking. Unterstützt zwei Merge-Strategien: **Upsert** (Insert/Update per Primärschlüssel) und **Append** (nur neue Datensätze anhängen).

### Ablauf

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌─────────────┐
│  incrInit    │────>│  User LOAD   │────>│  incrMerge   │────>│  incrStore   │
│              │     │  (Delta/Full)│     │              │     │              │
│ Prüft QVD   │     │ Nutzt        │     │ QVD + Delta  │     │ Speichert    │
│ Setzt Mode  │     │ vIncrWatermark│    │ zusammen-    │     │ nach QVD     │
│ + Watermark │     │ in WHERE     │     │ führen       │     │              │
└─────────────┘     └──────────────┘     └─────────────┘     └─────────────┘
```

### Variablen (automatisch gesetzt)

| Variable | Beschreibung |
|----------|-------------|
| `vIncrMode` | `'FULL'` oder `'DELTA'` — wird von `incrInit` gesetzt |
| `vIncrWatermark` | Max-Wert des Timestamp-Felds aus dem QVD (leer bei FULL) |
| `vIncrForceReload` | Auf `1` setzen **vor** `incrInit` um Full Reload zu erzwingen |

### Subroutinen

#### `incrInit (qvdPath, timestampField)`

Prüft ob das QVD existiert. Wenn ja, wird der maximale Wert des Timestamp-Felds als Watermark gelesen und `vIncrMode = 'DELTA'` gesetzt. Wenn nein (oder `vIncrForceReload = 1`), wird `vIncrMode = 'FULL'` gesetzt.

```qlik
CALL incrInit('lib://QVD/FactSales.qvd', 'ModifiedDate');
// vIncrMode = 'DELTA', vIncrWatermark = '2024-06-15 14:30:00'
```

#### `incrMerge (tableName, qvdPath, primaryKey)`

Führt die geladene Staging-Tabelle mit den bestehenden QVD-Daten zusammen.

- **Mit Primärschlüssel** (Upsert): Lädt QVD mit `WHERE NOT EXISTS(primaryKey)` — aktualisierte Datensätze werden durch die Delta-Version ersetzt.
- **Ohne Primärschlüssel** (Append): Lädt das gesamte QVD und hängt es an die Delta-Tabelle an.
- **Bei FULL-Modus**: Kein Merge nötig, Tabelle enthält bereits alles.

```qlik
CALL incrMerge('FactSales', 'lib://QVD/FactSales.qvd', 'SalesID');   // Upsert
CALL incrMerge('EventLog', 'lib://QVD/EventLog.qvd', '');             // Append
```

| Parameter | Beschreibung |
|-----------|-------------|
| `tableName` | Name der Staging-Tabelle (wird zur finalen Tabelle) |
| `qvdPath` | Pfad zum bestehenden QVD |
| `primaryKey` | Feld für Deduplizierung (leer = Append-Modus) |

#### `incrStore (tableName, qvdPath)`

Speichert die Tabelle als QVD.

```qlik
CALL incrStore('FactSales', 'lib://QVD/FactSales.qvd');
```

#### `incrCleanup`

Räumt alle Incremental-Load-Variablen auf.

```qlik
CALL incrCleanup;
```

### Beispiel 1: Upsert (Insert + Update)

Für Tabellen mit Primärschlüssel, bei denen bestehende Datensätze aktualisiert werden können.

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/incremental_load.qvs);

CALL logInit('Sales ETL');

// 1. Init: prüft QVD, setzt Watermark
CALL incrInit('lib://QVD/FactSales.qvd', 'ModifiedDate');

// 2. Delta laden (bei FULL ist vIncrWatermark leer -> WHERE ignoriert)
CALL logStartTimer('Load FactSales');
FactSales:
SQL SELECT SalesID, CustomerID, Amount, OrderDate, ModifiedDate
FROM dbo.Sales
WHERE ModifiedDate >= '$(vIncrWatermark)';
CALL logStopTimer('Load FactSales');

// 3. Merge: QVD + Delta zusammenführen (Upsert auf SalesID)
CALL incrMerge('FactSales', 'lib://QVD/FactSales.qvd', 'SalesID');

// 4. Speichern
CALL incrStore('FactSales', 'lib://QVD/FactSales.qvd');
CALL incrCleanup;

CALL logFinalize;
```

### Beispiel 2: Append-only (Log-Daten)

Für Tabellen ohne Updates — neue Datensätze werden nur angehängt.

```qlik
CALL incrInit('lib://QVD/EventLog.qvd', 'EventTimestamp');

EventLog:
SQL SELECT *
FROM dbo.EventLog
WHERE EventTimestamp > '$(vIncrWatermark)';
// Wichtig: > statt >= um Duplikate zu vermeiden (kein PK für Deduplizierung)

CALL incrMerge('EventLog', 'lib://QVD/EventLog.qvd', '');
CALL incrStore('EventLog', 'lib://QVD/EventLog.qvd');
CALL incrCleanup;
```

### Beispiel 3: Force Full Reload

```qlik
LET vIncrForceReload = 1;
CALL incrInit('lib://QVD/FactSales.qvd', 'ModifiedDate');
// vIncrMode ist jetzt 'FULL', unabhängig ob QVD existiert

FactSales:
SQL SELECT * FROM dbo.Sales;

// Kein Merge nötig bei FULL, aber Aufruf schadet nicht
CALL incrMerge('FactSales', 'lib://QVD/FactSales.qvd', 'SalesID');
CALL incrStore('FactSales', 'lib://QVD/FactSales.qvd');
CALL incrCleanup;
```

### Hinweis: WHERE-Klausel bei Delta

| Strategie | WHERE-Bedingung | Grund |
|-----------|----------------|-------|
| Upsert | `>= '$(vIncrWatermark)'` | Geänderte Datensätze zum gleichen Zeitpunkt werden per PK dedupliziert |
| Append | `> '$(vIncrWatermark)'` | Ohne PK muss der Watermark-Datensatz selbst ausgeschlossen werden |

---

## variables.qvs — Variable Management

Zentrale Verwaltung von KPI-Definitionen und Set-Analysis-Variablen. Variablen werden aus einer externen Datei (CSV/Excel) geladen und/oder automatisch für Zeitvergleiche generiert. Die `_variables`-Tabelle bleibt im Datenmodell als lebende Dokumentation.

### Ablauf

```
┌───────────┐     ┌─────────────────┐     ┌─────────────────────┐     ┌───────────┐
│  varInit   │────>│ varLoadFromFile │────>│ varCreateSetAnalysis │────>│ varApply   │
│            │     │ (aus CSV/Excel) │     │ (Zeitvergleiche)     │     │           │
│ Tabelle    │     │                 │     │                      │     │ Erstellt   │
│ anlegen    │     │ Optional        │     │ Optional             │     │ LET-Vars  │
└───────────┘     └─────────────────┘     └─────────────────────┘     └───────────┘
```

### Subroutinen

#### `varInit`

Initialisiert die interne `_variables`-Tabelle. Muss als erstes aufgerufen werden.

```qlik
CALL varInit;
```

#### `varLoadFromFile (filePath, nameField, defField, labelField, groupField)`

Lädt Variablen-Definitionen aus einer externen CSV-Datei (Semikolon-separiert).

```qlik
CALL varLoadFromFile('lib://Config/variables.csv', 'var_name', 'var_definition', 'var_label', 'var_group');
```

| Parameter | Beschreibung |
|-----------|-------------|
| `filePath` | Pfad zur Quelldatei (`lib://...`) |
| `nameField` | Spalte mit dem Variablennamen |
| `defField` | Spalte mit der Formel/Definition |
| `labelField` | Spalte mit dem Label (oder `''` zum Überspringen) |
| `groupField` | Spalte mit der Gruppe (oder `''` zum Überspringen) |

**CSV-Format (Semikolon-separiert, UTF-8):**

```csv
var_name;var_definition;var_label;var_group
vSales;Sum(Sales);Umsatz;KPI
vMargin;Sum(Margin)/Sum(Sales);Marge %;KPI
vOrderCount;Count(DISTINCT OrderID);Anzahl Aufträge;KPI
vAvgOrderValue;Sum(Sales)/Count(DISTINCT OrderID);Ø Auftragswert;KPI
```

Zeilen, die mit `//` beginnen, werden als Kommentare ignoriert.

#### `varAdd (name, definition, label, group)`

Fügt eine einzelne Variable programmatisch hinzu (ohne externe Datei).

```qlik
CALL varAdd('vCustomerCount', 'Count(DISTINCT CustomerID)', 'Anzahl Kunden', 'KPI');
```

#### `varCreateSetAnalysis`

Generiert Standard-Variablen für Zeitvergleiche. Nutzt die Feldnamen aus `calendar.qvs` (falls geladen).

**Generierte Variablen:**

| Variable | Definition | Beschreibung |
|----------|-----------|-------------|
| `vCY` | `Year(Today())` | Aktuelles Jahr |
| `vPY` | `Year(Today())-1` | Vorjahr |
| `vCM` | `Month(Today())` | Aktueller Monat |
| `vPM` | `Month(AddMonths(Today(),-1))` | Vormonat |
| `vCQ` | `'Q' & Ceil(Month(Today())/3)` | Aktuelles Quartal |
| `vPQ` | `'Q' & Ceil(...)` | Vorquartal |
| `vCYM` | `Date(MonthStart(Today()),'YYYY-MM')` | Aktueller Jahr-Monat |
| `vPYM` | `Date(MonthStart(...),'YYYY-MM')` | Vorheriger Jahr-Monat |

**Set-Analysis-Selektionen (für `{< >}` Blöcke):**

| Variable | Ergebnis | Verwendung |
|----------|---------|-----------|
| `vSetCY` | `year={$(vCY)}` | `Sum({<$(vSetCY)>} Sales)` |
| `vSetPY` | `year={$(vPY)}` | `Sum({<$(vSetPY)>} Sales)` |
| `vSetCM` | `year_month={$(vCYM)}` | `Sum({<$(vSetCM)>} Sales)` |
| `vSetPM` | `year_month={$(vPYM)}` | `Sum({<$(vSetPM)>} Sales)` |
| `vSetYTD` | `YTD={1}` | `Sum({<$(vSetYTD)>} Sales)` |
| `vSetYTM` | `YTM={1}` | `Sum({<$(vSetYTM)>} Sales)` |
| `vSetLTM` | `LTM={1}` | `Sum({<$(vSetLTM)>} Sales)` |
| `vSetCY_YTD` | `year={$(vCY)},YTD={1}` | CY + Year-to-Date |
| `vSetPY_YTD` | `year={$(vPY)},YTD={1}` | PY + Year-to-Date |

#### `varApply`

Erstellt alle geladenen Variablen als Qlik-Variablen (`LET`). Muss nach `varLoadFromFile` und/oder `varCreateSetAnalysis` aufgerufen werden.

```qlik
CALL varApply;
// -> [INFO] Variable Management: 25 of 25 variables applied
```

#### `varCleanup`

Abschluss-Routine. Die `_variables`-Tabelle bleibt im Datenmodell erhalten — sie dient als Dokumentation aller definierten Variablen im Dashboard.

### Vollständiges Beispiel

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/calendar.qvs);
$(Include=lib://Scripts/variables.qvs);

CALL logInit('Sales Dashboard');

// 1. Kalender (setzt Feldnamen für Set Analysis)
CALL startCalendarService;
CALL generateCalendar;

// 2. Variablen
CALL varInit;
CALL varLoadFromFile('lib://Config/kpis.csv', 'var_name', 'var_definition', 'var_label', 'var_group');
CALL varCreateSetAnalysis;
CALL varApply;
CALL varCleanup;

CALL stopCalendarService;
CALL logFinalize;
```

### Verwendung in Qlik Expressions

```qlik
// KPIs aus CSV
= $(vSales)                                      // -> Sum(Sales)
= $(vMargin)                                     // -> Sum(Margin)/Sum(Sales)

// Zeitvergleiche mit Set Analysis
= Sum({<$(vSetCY)>} Sales)                       // Umsatz aktuelles Jahr
= Sum({<$(vSetPY)>} Sales)                       // Umsatz Vorjahr
= Sum({<$(vSetCY_YTD)>} Sales)                   // Umsatz CY Year-to-Date
= Sum({<$(vSetCY)>} Sales) - Sum({<$(vSetPY)>} Sales)   // Differenz CY vs PY

// Kombiniert: KPI + Zeitraum
= $(vSales) & ' (CY: ' & Sum({<$(vSetCY)>} Sales) & ')'
```
