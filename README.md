# Qlik Sense Script Library

Wiederverwendbare QVS-Routinen für Qlik Sense Ladescripte.

## Übersicht

| Datei | Beschreibung |
|-------|-------------|
| `calendar.qvs` | Kalender-Generierung mit Mehrsprachigkeit, Fiskaljahr, TimeRanges |
| `logging.qvs` | Strukturiertes Logging mit Severity-Levels und Zeitmessung |
| `incremental_load.qvs` | Inkrementelles Laden mit QVD-Watermarking (Upsert / Append) |
| `variables.qvs` | Zentrale Variablen-/KPI-Verwaltung mit Set-Analysis-Generator |
| `data_quality.qvs` | Automatische Datenqualitätsprüfung (Profiling, NULLs, FK, Duplikate) |
| `time_dimension.qvs` | Uhrzeitkalender mit Zeitfenstern, Tageszeit, Schichten, Peak/Off-Peak |

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

---

## data_quality.qvs — Datenqualitäts-Framework

Automatische Datenqualitätsprüfung nach dem Laden. Alle Ergebnisse werden in einer `_data_quality`-Tabelle gesammelt, die im Datenmodell für QA-Dashboards verbleibt.

### Verfügbare Checks

| Check | Subroutine | Prüft |
|-------|-----------|-------|
| **Profiling** | `profileTable` | NULL-Anteil und Distinct-Count pro Feld |
| **Pflichtfelder** | `validateNotNull` | Keine NULL/leeren Werte in definierten Feldern |
| **Primärschlüssel** | `validateUnique` | Eindeutigkeit eines Schlüsselfelds |
| **Fremdschlüssel** | `validateFK` | Referentielle Integrität zwischen Tabellen |
| **Datumsbereich** | `validateDateRange` | Datumswerte innerhalb eines erwarteten Bereichs |

### Severity-Levels

| Level | Bedeutung |
|-------|-----------|
| `OK` | Check bestanden |
| `WARN` | Auffälligkeit, aber kein Blocker (z.B. >50% NULLs) |
| `ERROR` | Datenqualitätsproblem (z.B. Pflichtfeld NULL, Orphan-FKs, Duplikate) |

### Subroutinen

#### `dqInit`

Initialisiert die `_data_quality`-Ergebnistabelle.

```qlik
CALL dqInit;
```

#### `profileTable (tableName)`

Erstellt ein Profil aller Felder einer Tabelle: Zeilenanzahl, NULL-Anteil (absolut + prozentual), Distinct-Count pro Feld.

```qlik
CALL profileTable('FactSales');
```

Severity-Regeln:
- `ERROR` — 100% NULL (alle Werte leer)
- `WARN` — >50% NULL
- `OK` — alles andere

#### `validateNotNull (tableName, fieldList)`

Prüft, dass angegebene Pflichtfelder keine NULL- oder Leer-Werte enthalten. `fieldList` ist kommagetrennt.

```qlik
CALL validateNotNull('FactSales', 'SalesID, CustomerID, Amount, OrderDate');
```

#### `validateUnique (tableName, keyField)`

Prüft, ob ein Feld nur eindeutige Werte enthält (Primärschlüssel-Check).

```qlik
CALL validateUnique('FactSales', 'SalesID');
// -> ERROR: 47 duplicate rows (99953 distinct of 100000 total)
// -> OK: All 100000 values are unique
```

#### `validateFK (srcTable, srcField, tgtTable, tgtField)`

Prüft referentielle Integrität: Existieren alle Werte aus `srcField` auch in `tgtField`?

```qlik
CALL validateFK('FactSales', 'CustomerID', 'DimCustomer', 'CustomerID');
// -> ERROR: 5 distinct orphan values not found in DimCustomer.CustomerID
// -> OK: All values exist in DimCustomer.CustomerID
```

| Parameter | Beschreibung |
|-----------|-------------|
| `srcTable` | Quell-Tabelle (mit Fremdschlüssel) |
| `srcField` | Fremdschlüsselfeld |
| `tgtTable` | Ziel-Tabelle (mit Primärschlüssel) |
| `tgtField` | Referenziertes Feld in der Zieltabelle |

#### `validateDateRange (tableName, dateField, minDate, maxDate)`

Prüft, ob alle Datumswerte innerhalb eines erwarteten Bereichs liegen.

```qlik
CALL validateDateRange('FactSales', 'OrderDate', '2020-01-01', '2026-12-31');
// -> WARN: 23 dates outside range [2020-01-01 - 2026-12-31]. Actual: [2019-11-03 - 2027-01-15]
// -> OK: All 100000 dates within [2020-01-01 - 2026-12-31]
```

#### `dqFinalize`

Zählt Errors und Warnings zusammen, loggt eine Zusammenfassung. Bei Errors wird ein `logError` erzeugt.

```qlik
CALL dqFinalize;
// -> [INFO] Data Quality: finalized - 42 checks: 2 errors, 5 warnings
// -> [ERROR] Data Quality: 2 ERROR(s) detected - review _data_quality table
```

### Ergebnis-Tabelle `_data_quality`

| Spalte | Typ | Beschreibung |
|--------|-----|-------------|
| `dq_seq` | Integer | Laufende Nummer |
| `dq_timestamp` | Timestamp | Zeitpunkt des Checks |
| `dq_check` | String | Check-Typ: `PROFILE`, `NOT_NULL`, `UNIQUE`, `FK`, `DATE_RANGE` |
| `dq_table` | String | Geprüfte Tabelle |
| `dq_field` | String | Geprüftes Feld |
| `dq_severity` | String | `OK`, `WARN` oder `ERROR` |
| `dq_message` | String | Detaillierte Beschreibung |
| `dq_value` | Number | Kennzahl (z.B. Anzahl NULLs, Duplikate, Orphans) |

### Vollständiges Beispiel

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/data_quality.qvs);

CALL logInit('Sales Dashboard');

// ... Tabellen laden ...

// Data Quality Checks
CALL dqInit;

// Profiling
CALL profileTable('FactSales');
CALL profileTable('DimCustomer');

// Pflichtfelder
CALL validateNotNull('FactSales', 'SalesID, CustomerID, Amount, OrderDate');
CALL validateNotNull('DimCustomer', 'CustomerID, CustomerName');

// Primärschlüssel
CALL validateUnique('FactSales', 'SalesID');
CALL validateUnique('DimCustomer', 'CustomerID');

// Fremdschlüssel
CALL validateFK('FactSales', 'CustomerID', 'DimCustomer', 'CustomerID');
CALL validateFK('FactSales', 'ProductID', 'DimProduct', 'ProductID');

// Datumsbereiche
CALL validateDateRange('FactSales', 'OrderDate', '2020-01-01', '2026-12-31');

CALL dqFinalize;
CALL logFinalize;
```

### QA-Dashboard

Die `_data_quality`-Tabelle kann direkt in einem Qlik Sheet visualisiert werden:

- **Tabelle** mit `dq_severity` als Farbindikator (OK=grün, WARN=gelb, ERROR=rot)
- **KPI** mit `Count({<dq_severity={'ERROR'}>} dq_seq)` für Error-Count
- **Filter** auf `dq_check` und `dq_table` für gezieltes Drilling

---

## time_dimension.qvs — Uhrzeitkalender

Generiert eine Zeitdimension mit Minuten-Granularität (1440 Zeilen) für Analysen auf Uhrzeitbasis. Unterstützt konfigurierbare Zeitfenster, Schichtmodelle und Peak-/Off-Peak-Markierung.

### Generierte Felder

| Feld (DE) | Feld (EN) | Beschreibung | Beispiel |
|-----------|-----------|-------------|---------|
| Uhrzeit | time | Formatierte Uhrzeit (hh:mm) | `14:30` |
| Stunde | hour | Stunde (0-23) | `14` |
| Minute | minute | Minute (0-59) | `30` |
| Zeitfenster | time_slot | Zeitslot nach Intervall | `14:00 - 14:29` |
| Tageszeit | day_period | Tageszeit-Klassifizierung | `Nachmittag` |
| Schicht | shift | Schichtzuordnung | `Spätschicht` |
| Hauptzeit | peak_hour | Peak-/Off-Peak-Flag (1/0) | `1` |

### Konfiguration

Alle Variablen sind optional — ohne Konfiguration werden Defaults verwendet.

#### Zeitfenster

| Variable | Default | Beschreibung |
|----------|---------|-------------|
| `vTimeSlotMinutes` | `30` | Intervall in Minuten (z.B. 15, 30, 60) |

#### Schichtmodell

Default: 3-Schicht-Modell (Früh 06-14, Spät 14-22, Nacht 22-06)

| Variable | Default | Beschreibung |
|----------|---------|-------------|
| `vTimeShift1Start` | `6` | Beginn Schicht 1 (Stunde) |
| `vTimeShift1End` | `14` | Ende Schicht 1 |
| `vTimeShift1Name_DE/EN` | `Frühschicht / Early Shift` | Name |
| `vTimeShift2Start` | `14` | Beginn Schicht 2 |
| `vTimeShift2End` | `22` | Ende Schicht 2 |
| `vTimeShift2Name_DE/EN` | `Spätschicht / Late Shift` | Name |
| `vTimeShift3Start` | `22` | Beginn Schicht 3 |
| `vTimeShift3End` | `6` | Ende Schicht 3 (nächster Tag) |
| `vTimeShift3Name_DE/EN` | `Nachtschicht / Night Shift` | Name |

#### Peak-Stunden

| Variable | Default | Beschreibung |
|----------|---------|-------------|
| `vTimePeakStart` | `8` | Beginn Peak-Zeitraum |
| `vTimePeakEnd` | `18` | Ende Peak-Zeitraum |

### Tageszeit-Zuordnung

| Stunden | Tageszeit (DE) | Tageszeit (EN) |
|---------|---------------|----------------|
| 06:00 - 11:59 | Morgen | Morning |
| 12:00 - 17:59 | Nachmittag | Afternoon |
| 18:00 - 21:59 | Abend | Evening |
| 22:00 - 05:59 | Nacht | Night |

### Subroutinen

#### `generateTimeDimension`

Erzeugt die `time_dimension`-Tabelle mit 1440 Zeilen (eine pro Minute).

```qlik
CALL generateTimeDimension;
```

#### `joinTimeDimension (targetTable, srcTimestampField, fieldPrefix)`

Extrahiert die Uhrzeit (hh:mm) aus einem Timestamp-Feld und joint die Zeitdimensions-Felder in die Zieltabelle.

```qlik
CALL joinTimeDimension('FactOrders', 'OrderTimestamp', 'Order');
// Erzeugt: Order_hour, Order_time_slot, Order_day_period, Order_shift, Order_peak_hour
```

| Parameter | Beschreibung |
|-----------|-------------|
| `targetTable` | Name der Zieltabelle |
| `srcTimestampField` | Timestamp-Feld in der Zieltabelle |
| `fieldPrefix` | Prefix für die generierten Felder |

#### `cleanupTimeDimension`

Räumt alle Konfigurations- und internen Variablen auf. Die `time_dimension`-Tabelle bleibt im Datenmodell.

```qlik
CALL cleanupTimeDimension;
```

### Beispiel 1: Standard (30-Min-Slots, 3-Schicht)

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/time_dimension.qvs);

SET vDataModelLanguage = 'DE';
CALL logInit('Mein Dashboard');

CALL generateTimeDimension;
CALL cleanupTimeDimension;

CALL logFinalize;
```

### Beispiel 2: Custom (15-Min-Slots, 2-Schicht, Peak 09-17)

```qlik
SET vDataModelLanguage = 'EN';
LET vTimeSlotMinutes = 15;

// 2-Shift model
LET vTimeShift1Start = 6;
LET vTimeShift1End = 18;
LET vTimeShift1Name_DE = 'Tagschicht';
LET vTimeShift1Name_EN = 'Day Shift';
LET vTimeShift2Start = 18;
LET vTimeShift2End = 6;
LET vTimeShift2Name_DE = 'Nachtschicht';
LET vTimeShift2Name_EN = 'Night Shift';
LET vTimeShift3Start = 0;
LET vTimeShift3End = 0;
LET vTimeShift3Name_DE = 'Nachtschicht';
LET vTimeShift3Name_EN = 'Night Shift';

LET vTimePeakStart = 9;
LET vTimePeakEnd = 17;

CALL generateTimeDimension;
CALL cleanupTimeDimension;
```

### Beispiel 3: Integration mit Kalender + Join auf Faktentabelle

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/calendar.qvs);
$(Include=lib://Scripts/time_dimension.qvs);

CALL logInit('Production Dashboard');

// Kalender
LET vCalendarStart = num(MakeDate(year(Today()), 01, 01));
LET vCalendarEnd   = num(MakeDate(Year(Today()) + 1, 12, 31));
SET vDataModelLanguage = 'DE';

CALL startCalendarService;
CALL generateCalendar;
CALL stopCalendarService;

// Zeitdimension
CALL generateTimeDimension;

// Faktentabelle laden
FactProduction:
SQL SELECT EventID, EventTimestamp, MachineID, Quantity FROM dbo.Production;

// Zeit-Felder zur Faktentabelle joinen
CALL joinTimeDimension('FactProduction', 'EventTimestamp', 'Event');

CALL cleanupTimeDimension;
CALL logFinalize;
```
