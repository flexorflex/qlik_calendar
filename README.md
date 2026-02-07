# Qlik Sense Script Library

Reusable QVS routines for Qlik Sense load scripts.

## Overview

| File | Description |
|------|------------|
| `calendar.qvs` | Calendar generation with multi-language support, fiscal year, TimeRanges |
| `logging.qvs` | Structured logging with severity levels and duration measurement |
| `incremental_load.qvs` | Incremental loading with QVD watermarking (Upsert / Append) |
| `variables.qvs` | Centralized variable/KPI management with Set Analysis generator |
| `data_quality.qvs` | Automated data quality checks (profiling, NULLs, FK, duplicates) |
| `time_dimension.qvs` | Time-of-day calendar with time slots, day periods, shifts, peak/off-peak |

---

## calendar.qvs — Calendar Routine

Generates a comprehensive master calendar with optional field prefix, fiscal year support, and TimeRange tables.

### Prerequisites

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/calendar.qvs);

LET vCalendarStart = num(MakeDate(year(Today()), 01, 01));
LET vCalendarEnd   = num(MakeDate(Year(Today()) + 1, 12, 31));
SET vDataModelLanguage = 'EN';   // or 'DE'
```

| Variable | Required | Description |
|----------|----------|------------|
| `vCalendarStart` | Yes | Start date as numeric value |
| `vCalendarEnd` | Yes | End date as numeric value |
| `vDataModelLanguage` | Yes | `'DE'` or `'EN'` — controls field names |
| `vCalendarFM` | No | First month of fiscal year (1-12, default: 1) |

### Subroutines

#### `startCalendarService`

Initializes the calendar service: validates `vCalendarFM`, builds multi-language field names.

```qlik
CALL startCalendarService;
```

#### `generateCalendar (vCalendarQualifier)`

Creates three tables:

- **`calendar`** (or `calendar<Qualifier>`) — Main calendar with all fields
- **`time_ranges`** — Lookup for time periods (YTM, YTD, LTM)
- **`calendar_time_range`** — Links dates to time periods

The optional `vCalendarQualifier` is used as a prefix on all field and table names. This allows multiple calendars to coexist.

```qlik
CALL generateCalendar;                    // without qualifier
CALL generateCalendar('Order');           // fields: OrderDatum, OrderJahr, ...
```

**Generated fields (example EN):**

| Category | Fields |
|----------|--------|
| Date | date, date_DE, date_EN, day, weekday |
| Week | week, week_current, week_previous, year_week |
| Month | month, month_no, month_current, month_previous, year_month, year_monthname |
| Quarter/Year | quarter, year |
| Workdays | workday, weekend |
| Time comparison | days_ago, weeks_ago, months_ago |
| Fiscal year | fiscal_year, fiscal_quarter, fiscal_month, fiscal_month_current, fiscal_month_previous |
| Time ranges | YTD, YTM, LTM |

#### `joinCalendar (targetTable, srcDateField, fieldPrefix)`

Joins reduced calendar fields (year, month, year_month, YTD) into an existing fact table via LEFT JOIN.

```qlik
CALL joinCalendar('FactSales', 'OrderDate', 'Order');
// Creates: Order_year, Order_month, Order_year_month, Order_YTD
```

| Parameter | Description |
|-----------|------------|
| `targetTable` | Name of the target table |
| `srcDateField` | Date field in the target table |
| `fieldPrefix` | Prefix for the generated fields |

#### `stopCalendarService`

Drops all temporary tables and cleans up variables. Always call last.

```qlik
CALL stopCalendarService;
```

### Full Example

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/calendar.qvs);

// Variables
LET vCalendarStart = num(MakeDate(year(Today()), 01, 01));
LET vCalendarEnd   = num(MakeDate(Year(Today()) + 1, 12, 31));
SET vDataModelLanguage = 'EN';
LET vCalendarFM = 4;   // fiscal year starts in April

// Logging
CALL logInit('My Dashboard');

// Generate calendar
CALL logStartTimer('Calendar');
CALL startCalendarService;
CALL generateCalendar;
CALL stopCalendarService;
CALL logStopTimer('Calendar');

CALL logRowCount('calendar');
CALL logFinalize;
```

### Fiscal Year

When `vCalendarFM` is set (e.g. `4` for April), the fiscal fields are shifted accordingly:

| Calendar Month | Fiscal Month (FM=4) | Fiscal Year |
|---------------|-------------------|------------|
| Jan 2024 | 10 | 2024 |
| Apr 2024 | 1 | 2025 |
| Dec 2024 | 9 | 2025 |
| Mar 2025 | 12 | 2025 |

---

## logging.qvs — Logging Framework

Structured logging for Qlik Sense load scripts with severity levels, duration measurement, and optional QVD export.

### Subroutines

#### `logInit (appName)`

Initializes the `_log` table and starts the overall timer.

```qlik
CALL logInit('Sales Dashboard');
```

#### `logInfo (message)` / `logWarn (message)` / `logError (message)`

Writes a log entry with the respective severity level. Each entry is also output via `TRACE` to the script console.

```qlik
CALL logInfo('Data loaded successfully');
CALL logWarn('12 records without postal code');
CALL logError('Database connection failed');
```

#### `logRowCount (tableName)`

Logs the row count of a table.

```qlik
CALL logRowCount('FactSales');
// -> [INFO] Table [FactSales] has 1234567 rows
```

#### `logStartTimer (stepName)` / `logStopTimer (stepName)`

Measures the duration of a load step. The duration is stored in seconds in the `log_duration_sec` column.

```qlik
CALL logStartTimer('Load Customers');
// ... load logic ...
CALL logStopTimer('Load Customers');
// -> [INFO] Timer stopped: Load Customers - duration: 00:01:23
```

#### `logExport (path)`

Persists the log table as a QVD for historical analysis across reload cycles.

```qlik
CALL logExport('lib://QVD/logs/load_log.qvd');
```

#### `logFinalize`

Logs the total duration and cleans up all logging variables. The `_log` table remains in the data model for QA dashboards.

```qlik
CALL logFinalize;
```

### Log Table `_log`

| Column | Type | Description |
|--------|------|------------|
| `log_seq` | Integer | Sequence number |
| `log_timestamp` | Timestamp | Time of the entry |
| `log_level` | String | `INFO`, `WARN` or `ERROR` |
| `log_message` | String | Log message |
| `log_app` | String | App name from `logInit` |
| `log_duration_sec` | Number | Duration in seconds (only for `logStopTimer`) |

### Full Example

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

## incremental_load.qvs — Incremental Loading

Reusable framework for delta loading with QVD watermarking. Supports two merge strategies: **Upsert** (insert/update by primary key) and **Append** (insert-only, no deduplication).

### Flow

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌─────────────┐
│  incrInit    │────>│  User LOAD   │────>│  incrMerge   │────>│  incrStore   │
│              │     │  (Delta/Full)│     │              │     │              │
│ Checks QVD  │     │ Uses         │     │ QVD + Delta  │     │ Stores       │
│ Sets Mode   │     │ vIncrWatermark│    │ merged       │     │ to QVD       │
│ + Watermark │     │ in WHERE     │     │              │     │              │
└─────────────┘     └──────────────┘     └─────────────┘     └─────────────┘
```

### Variables (set automatically)

| Variable | Description |
|----------|------------|
| `vIncrMode` | `'FULL'` or `'DELTA'` — set by `incrInit` |
| `vIncrWatermark` | Max value of the timestamp field from QVD (empty for FULL) |
| `vIncrForceReload` | Set to `1` **before** `incrInit` to force a full reload |

### Subroutines

#### `incrInit (qvdPath, timestampField)`

Checks if the QVD exists. If yes, reads the maximum timestamp value as watermark and sets `vIncrMode = 'DELTA'`. If not (or `vIncrForceReload = 1`), sets `vIncrMode = 'FULL'`.

```qlik
CALL incrInit('lib://QVD/FactSales.qvd', 'ModifiedDate');
// vIncrMode = 'DELTA', vIncrWatermark = '2024-06-15 14:30:00'
```

#### `incrMerge (tableName, qvdPath, primaryKey)`

Merges the loaded staging table with existing QVD data.

- **With primary key** (Upsert): Loads QVD with `WHERE NOT EXISTS(primaryKey)` — updated records are replaced by the delta version.
- **Without primary key** (Append): Loads the entire QVD and appends it to the delta table.
- **In FULL mode**: No merge needed, table already contains all data.

```qlik
CALL incrMerge('FactSales', 'lib://QVD/FactSales.qvd', 'SalesID');   // Upsert
CALL incrMerge('EventLog', 'lib://QVD/EventLog.qvd', '');             // Append
```

| Parameter | Description |
|-----------|------------|
| `tableName` | Name of the staging table (becomes the final table) |
| `qvdPath` | Path to the existing QVD |
| `primaryKey` | Field for deduplication (empty = append mode) |

#### `incrStore (tableName, qvdPath)`

Stores the table as QVD.

```qlik
CALL incrStore('FactSales', 'lib://QVD/FactSales.qvd');
```

#### `incrCleanup`

Cleans up all incremental load variables.

```qlik
CALL incrCleanup;
```

### Example 1: Upsert (Insert + Update)

For tables with a primary key where existing records may be updated.

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/incremental_load.qvs);

CALL logInit('Sales ETL');

// 1. Init: check QVD, set watermark
CALL incrInit('lib://QVD/FactSales.qvd', 'ModifiedDate');

// 2. Load delta (on FULL load vIncrWatermark is empty -> WHERE is ignored)
CALL logStartTimer('Load FactSales');
FactSales:
SQL SELECT SalesID, CustomerID, Amount, OrderDate, ModifiedDate
FROM dbo.Sales
WHERE ModifiedDate >= '$(vIncrWatermark)';
CALL logStopTimer('Load FactSales');

// 3. Merge: combine QVD + delta (upsert on SalesID)
CALL incrMerge('FactSales', 'lib://QVD/FactSales.qvd', 'SalesID');

// 4. Store
CALL incrStore('FactSales', 'lib://QVD/FactSales.qvd');
CALL incrCleanup;

CALL logFinalize;
```

### Example 2: Append-only (Log Data)

For tables without updates — new records are only appended.

```qlik
CALL incrInit('lib://QVD/EventLog.qvd', 'EventTimestamp');

EventLog:
SQL SELECT *
FROM dbo.EventLog
WHERE EventTimestamp > '$(vIncrWatermark)';
// Important: > instead of >= to avoid duplicates (no PK for deduplication)

CALL incrMerge('EventLog', 'lib://QVD/EventLog.qvd', '');
CALL incrStore('EventLog', 'lib://QVD/EventLog.qvd');
CALL incrCleanup;
```

### Example 3: Force Full Reload

```qlik
LET vIncrForceReload = 1;
CALL incrInit('lib://QVD/FactSales.qvd', 'ModifiedDate');
// vIncrMode is now 'FULL' regardless of QVD existence

FactSales:
SQL SELECT * FROM dbo.Sales;

// No merge needed for FULL, but calling it doesn't hurt
CALL incrMerge('FactSales', 'lib://QVD/FactSales.qvd', 'SalesID');
CALL incrStore('FactSales', 'lib://QVD/FactSales.qvd');
CALL incrCleanup;
```

### Note: WHERE Clause for Delta

| Strategy | WHERE Condition | Reason |
|----------|----------------|--------|
| Upsert | `>= '$(vIncrWatermark)'` | Modified records at the same timestamp are deduplicated by PK |
| Append | `> '$(vIncrWatermark)'` | Without PK, the watermark record itself must be excluded |

---

## variables.qvs — Variable Management

Centralized management of KPI definitions and Set Analysis variables. Variables are loaded from an external file (CSV/Excel) and/or auto-generated for time-based comparisons. The `_variables` table remains in the data model as living documentation.

### Flow

```
┌───────────┐     ┌─────────────────┐     ┌─────────────────────┐     ┌───────────┐
│  varInit   │────>│ varLoadFromFile │────>│ varCreateSetAnalysis │────>│ varApply   │
│            │     │ (from CSV/Excel)│     │ (time comparisons)   │     │           │
│ Create     │     │                 │     │                      │     │ Creates   │
│ table      │     │ Optional        │     │ Optional             │     │ LET vars  │
└───────────┘     └─────────────────┘     └─────────────────────┘     └───────────┘
```

### Subroutines

#### `varInit`

Initializes the internal `_variables` table. Must be called first.

```qlik
CALL varInit;
```

#### `varLoadFromFile (filePath, nameField, defField, labelField, groupField)`

Loads variable definitions from an external CSV file (semicolon-separated).

```qlik
CALL varLoadFromFile('lib://Config/variables.csv', 'var_name', 'var_definition', 'var_label', 'var_group');
```

| Parameter | Description |
|-----------|------------|
| `filePath` | Path to source file (`lib://...`) |
| `nameField` | Column with the variable name |
| `defField` | Column with the formula/definition |
| `labelField` | Column with the label (or `''` to skip) |
| `groupField` | Column with the group (or `''` to skip) |

**CSV format (semicolon-separated, UTF-8):**

```csv
var_name;var_definition;var_label;var_group
vSales;Sum(Sales);Revenue;KPI
vMargin;Sum(Margin)/Sum(Sales);Margin %;KPI
vOrderCount;Count(DISTINCT OrderID);Order Count;KPI
vAvgOrderValue;Sum(Sales)/Count(DISTINCT OrderID);Avg Order Value;KPI
```

Lines starting with `//` are treated as comments and ignored.

#### `varAdd (name, definition, label, group)`

Adds a single variable definition programmatically (without external file).

```qlik
CALL varAdd('vCustomerCount', 'Count(DISTINCT CustomerID)', 'Customer Count', 'KPI');
```

#### `varCreateSetAnalysis`

Generates standard variables for time-based comparisons. Uses the field names from `calendar.qvs` (if loaded).

**Generated variables:**

| Variable | Definition | Description |
|----------|-----------|------------|
| `vCY` | `Year(Today())` | Current Year |
| `vPY` | `Year(Today())-1` | Previous Year |
| `vCM` | `Month(Today())` | Current Month |
| `vPM` | `Month(AddMonths(Today(),-1))` | Previous Month |
| `vCQ` | `'Q' & Ceil(Month(Today())/3)` | Current Quarter |
| `vPQ` | `'Q' & Ceil(...)` | Previous Quarter |
| `vCYM` | `Date(MonthStart(Today()),'YYYY-MM')` | Current Year-Month |
| `vPYM` | `Date(MonthStart(...),'YYYY-MM')` | Previous Year-Month |

**Set Analysis selections (for `{< >}` blocks):**

| Variable | Result | Usage |
|----------|--------|-------|
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

Creates all loaded variables as Qlik variables (`LET`). Must be called after `varLoadFromFile` and/or `varCreateSetAnalysis`.

```qlik
CALL varApply;
// -> [INFO] Variable Management: 25 of 25 variables applied
```

#### `varCleanup`

Cleanup routine. The `_variables` table remains in the data model for documentation purposes.

### Full Example

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/calendar.qvs);
$(Include=lib://Scripts/variables.qvs);

CALL logInit('Sales Dashboard');

// 1. Calendar (sets field names for Set Analysis)
CALL startCalendarService;
CALL generateCalendar;

// 2. Variables
CALL varInit;
CALL varLoadFromFile('lib://Config/kpis.csv', 'var_name', 'var_definition', 'var_label', 'var_group');
CALL varCreateSetAnalysis;
CALL varApply;
CALL varCleanup;

CALL stopCalendarService;
CALL logFinalize;
```

### Usage in Qlik Expressions

```qlik
// KPIs from CSV
= $(vSales)                                      // -> Sum(Sales)
= $(vMargin)                                     // -> Sum(Margin)/Sum(Sales)

// Time comparisons with Set Analysis
= Sum({<$(vSetCY)>} Sales)                       // Revenue current year
= Sum({<$(vSetPY)>} Sales)                       // Revenue previous year
= Sum({<$(vSetCY_YTD)>} Sales)                   // Revenue CY year-to-date
= Sum({<$(vSetCY)>} Sales) - Sum({<$(vSetPY)>} Sales)   // Difference CY vs PY

// Combined: KPI + time period
= $(vSales) & ' (CY: ' & Sum({<$(vSetCY)>} Sales) & ')'
```

---

## data_quality.qvs — Data Quality Framework

Automated data quality checks after loading. All results are collected in a `_data_quality` table that remains in the data model for QA dashboards.

### Available Checks

| Check | Subroutine | Validates |
|-------|-----------|----------|
| **Profiling** | `profileTable` | NULL ratio and distinct count per field |
| **Mandatory fields** | `validateNotNull` | No NULL/empty values in specified fields |
| **Primary key** | `validateUnique` | Uniqueness of a key field |
| **Foreign key** | `validateFK` | Referential integrity between tables |
| **Date range** | `validateDateRange` | Date values within an expected range |

### Severity Levels

| Level | Meaning |
|-------|---------|
| `OK` | Check passed |
| `WARN` | Anomaly detected, but not a blocker (e.g. >50% NULLs) |
| `ERROR` | Data quality issue (e.g. mandatory field NULL, orphan FKs, duplicates) |

### Subroutines

#### `dqInit`

Initializes the `_data_quality` results table.

```qlik
CALL dqInit;
```

#### `profileTable (tableName)`

Profiles all fields in a table: row count, NULL ratio (absolute + percentage), distinct count per field.

```qlik
CALL profileTable('FactSales');
```

Severity rules:
- `ERROR` — 100% NULL (all values empty)
- `WARN` — >50% NULL
- `OK` — everything else

#### `validateNotNull (tableName, fieldList)`

Validates that specified mandatory fields contain no NULL or empty values. `fieldList` is comma-separated.

```qlik
CALL validateNotNull('FactSales', 'SalesID, CustomerID, Amount, OrderDate');
```

#### `validateUnique (tableName, keyField)`

Validates that a field contains only unique values (primary key check).

```qlik
CALL validateUnique('FactSales', 'SalesID');
// -> ERROR: 47 duplicate rows (99953 distinct of 100000 total)
// -> OK: All 100000 values are unique
```

#### `validateFK (srcTable, srcField, tgtTable, tgtField)`

Validates referential integrity: do all values in `srcField` exist in `tgtField`?

```qlik
CALL validateFK('FactSales', 'CustomerID', 'DimCustomer', 'CustomerID');
// -> ERROR: 5 distinct orphan values not found in DimCustomer.CustomerID
// -> OK: All values exist in DimCustomer.CustomerID
```

| Parameter | Description |
|-----------|------------|
| `srcTable` | Source table (with foreign key) |
| `srcField` | Foreign key field |
| `tgtTable` | Target table (with primary key) |
| `tgtField` | Referenced field in the target table |

#### `validateDateRange (tableName, dateField, minDate, maxDate)`

Validates that all date values fall within an expected range.

```qlik
CALL validateDateRange('FactSales', 'OrderDate', '2020-01-01', '2026-12-31');
// -> WARN: 23 dates outside range [2020-01-01 - 2026-12-31]. Actual: [2019-11-03 - 2027-01-15]
// -> OK: All 100000 dates within [2020-01-01 - 2026-12-31]
```

#### `dqFinalize`

Counts errors and warnings, logs a summary. On errors, a `logError` is raised.

```qlik
CALL dqFinalize;
// -> [INFO] Data Quality: finalized - 42 checks: 2 errors, 5 warnings
// -> [ERROR] Data Quality: 2 ERROR(s) detected - review _data_quality table
```

### Results Table `_data_quality`

| Column | Type | Description |
|--------|------|------------|
| `dq_seq` | Integer | Sequence number |
| `dq_timestamp` | Timestamp | Time of the check |
| `dq_check` | String | Check type: `PROFILE`, `NOT_NULL`, `UNIQUE`, `FK`, `DATE_RANGE` |
| `dq_table` | String | Checked table |
| `dq_field` | String | Checked field |
| `dq_severity` | String | `OK`, `WARN` or `ERROR` |
| `dq_message` | String | Detailed description |
| `dq_value` | Number | Metric (e.g. NULL count, duplicates, orphans) |

### Full Example

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/data_quality.qvs);

CALL logInit('Sales Dashboard');

// ... load tables ...

// Data Quality Checks
CALL dqInit;

// Profiling
CALL profileTable('FactSales');
CALL profileTable('DimCustomer');

// Mandatory fields
CALL validateNotNull('FactSales', 'SalesID, CustomerID, Amount, OrderDate');
CALL validateNotNull('DimCustomer', 'CustomerID, CustomerName');

// Primary keys
CALL validateUnique('FactSales', 'SalesID');
CALL validateUnique('DimCustomer', 'CustomerID');

// Foreign keys
CALL validateFK('FactSales', 'CustomerID', 'DimCustomer', 'CustomerID');
CALL validateFK('FactSales', 'ProductID', 'DimProduct', 'ProductID');

// Date ranges
CALL validateDateRange('FactSales', 'OrderDate', '2020-01-01', '2026-12-31');

CALL dqFinalize;
CALL logFinalize;
```

### QA Dashboard

The `_data_quality` table can be visualized directly in a Qlik sheet:

- **Table** with `dq_severity` as color indicator (OK=green, WARN=yellow, ERROR=red)
- **KPI** with `Count({<dq_severity={'ERROR'}>} dq_seq)` for error count
- **Filter** on `dq_check` and `dq_table` for targeted drilling

---

## time_dimension.qvs — Time-of-Day Calendar

Generates a time dimension with minute-level granularity (1440 rows) for time-of-day analysis. Supports configurable time slots, shift models, and peak/off-peak marking.

### Generated Fields

| Field (DE) | Field (EN) | Description | Example |
|-----------|-----------|------------|---------|
| Uhrzeit | time | Formatted time (hh:mm) | `14:30` |
| Stunde | hour | Hour (0-23) | `14` |
| Minute | minute | Minute (0-59) | `30` |
| Zeitfenster | time_slot | Time slot by interval | `14:00 - 14:29` |
| Tageszeit | day_period | Day period classification | `Afternoon` |
| Schicht | shift | Shift assignment | `Late Shift` |
| Hauptzeit | peak_hour | Peak/off-peak flag (1/0) | `1` |

### Configuration

All variables are optional — defaults are used when not configured.

#### Time Slots

| Variable | Default | Description |
|----------|---------|------------|
| `vTimeSlotMinutes` | `30` | Interval in minutes (e.g. 15, 30, 60) |

#### Shift Model

Default: 3-shift model (Early 06-14, Late 14-22, Night 22-06)

| Variable | Default | Description |
|----------|---------|------------|
| `vTimeShift1Start` | `6` | Shift 1 start (hour) |
| `vTimeShift1End` | `14` | Shift 1 end |
| `vTimeShift1Name_DE/EN` | `Frühschicht / Early Shift` | Name |
| `vTimeShift2Start` | `14` | Shift 2 start |
| `vTimeShift2End` | `22` | Shift 2 end |
| `vTimeShift2Name_DE/EN` | `Spätschicht / Late Shift` | Name |
| `vTimeShift3Start` | `22` | Shift 3 start |
| `vTimeShift3End` | `6` | Shift 3 end (next day) |
| `vTimeShift3Name_DE/EN` | `Nachtschicht / Night Shift` | Name |

#### Peak Hours

| Variable | Default | Description |
|----------|---------|------------|
| `vTimePeakStart` | `8` | Peak period start |
| `vTimePeakEnd` | `18` | Peak period end |

### Day Period Mapping

| Hours | Day Period (DE) | Day Period (EN) |
|-------|----------------|----------------|
| 06:00 - 11:59 | Morgen | Morning |
| 12:00 - 17:59 | Nachmittag | Afternoon |
| 18:00 - 21:59 | Abend | Evening |
| 22:00 - 05:59 | Nacht | Night |

### Subroutines

#### `generateTimeDimension`

Creates the `time_dimension` table with 1440 rows (one per minute).

```qlik
CALL generateTimeDimension;
```

#### `joinTimeDimension (targetTable, srcTimestampField, fieldPrefix)`

Extracts the time (hh:mm) from a timestamp field and joins the time dimension fields into the target table.

```qlik
CALL joinTimeDimension('FactOrders', 'OrderTimestamp', 'Order');
// Creates: Order_hour, Order_time_slot, Order_day_period, Order_shift, Order_peak_hour
```

| Parameter | Description |
|-----------|------------|
| `targetTable` | Name of the target table |
| `srcTimestampField` | Timestamp field in the target table |
| `fieldPrefix` | Prefix for the generated fields |

#### `cleanupTimeDimension`

Cleans up all configuration and internal variables. The `time_dimension` table remains in the data model.

```qlik
CALL cleanupTimeDimension;
```

### Example 1: Default (30-min slots, 3-shift)

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/time_dimension.qvs);

SET vDataModelLanguage = 'EN';
CALL logInit('My Dashboard');

CALL generateTimeDimension;
CALL cleanupTimeDimension;

CALL logFinalize;
```

### Example 2: Custom (15-min slots, 2-shift, peak 09-17)

```qlik
SET vDataModelLanguage = 'EN';
LET vTimeSlotMinutes = 15;

// 2-shift model
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

### Example 3: Integration with Calendar + Join to Fact Table

```qlik
$(Include=lib://Scripts/logging.qvs);
$(Include=lib://Scripts/calendar.qvs);
$(Include=lib://Scripts/time_dimension.qvs);

CALL logInit('Production Dashboard');

// Calendar
LET vCalendarStart = num(MakeDate(year(Today()), 01, 01));
LET vCalendarEnd   = num(MakeDate(Year(Today()) + 1, 12, 31));
SET vDataModelLanguage = 'EN';

CALL startCalendarService;
CALL generateCalendar;
CALL stopCalendarService;

// Time dimension
CALL generateTimeDimension;

// Load fact table
FactProduction:
SQL SELECT EventID, EventTimestamp, MachineID, Quantity FROM dbo.Production;

// Join time fields to fact table
CALL joinTimeDimension('FactProduction', 'EventTimestamp', 'Event');

CALL cleanupTimeDimension;
CALL logFinalize;
```
