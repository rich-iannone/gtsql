# gtsql Syntax Specification

**Version**: Alpha (subject to change)
**Basis**: Grammar of Tables (gt) + SQL
**Companion to**: ggsql (`GGSQL_SYNTAX.md`)
**Reference**: https://gt.rstudio.com/

---

## 0. High-Level Syntax Structure

A gtsql query is, at the highest level:

```
[ SQL_PORTION ]
TABULATE [ FROM data_source ] [ SETTING global_config ]
  [ tab_clause ]*
[ RENDER TO writer | EXPORT TO 'path' ]
```

### 0.1 Skeleton

```sql
<SQL: SELECT / WITH / joins / filters / aggregations>
TABULATE [FROM <source>]
  -- 1. Structure
  STUB        <col> [GROUP <col>]
  STUBHEAD    '<label>'
  SPANNER     '<label>' OVER (<cols>)
  COLUMN      <col> AS '<label>' [SETTING ...] [UNITS '<u>']
  HIDE        COLUMN <cols> | ROWS WHERE <cond>
  MOVE        COLUMN <cols> TO START | END | AFTER <c> | BEFORE <c>
  MERGE       <kind> (<cols>) [AS '<label>'] [SETTING ...]
  NANOPLOT    <new_col> FROM (<cols>) SETTING ...

  -- 2. Rows
  ROWS ADD    (<col> => <val>, ...) [AT <pos>]
  ROWS HIDE   WHERE <cond>
  ROWS ORDER BY <col> [ASC|DESC]
  SUMMARY     [ON <group_col>] SETTING fns => (...), columns => (...)
  GRAND SUMMARY SETTING fns => (...), columns => (...), label => '...'

  -- 3. Cell transformation (phase 1 → 2 → 3)
  FORMAT      <kind> ON (<cols>) [WHERE <cond>] [SETTING ...] [LOCALE '...']
  SUBSTITUTE  <kind> ON (<cols>) [WHERE <cond>] SETTING ...
  COLOR DATA  ON (<cols>) SETTING palette => '...', ...
  TEXT        REPLACE | CASE WHEN | CASE MATCH | TRANSFORM ON <loc> SETTING ...

  -- 4. Annotation
  HEADER      title => '...', subtitle => '...'
  FOOTNOTE    '<text>' [AT <location>]
  SOURCE      '<citation>'
  CAPTION     '<text>'

  -- 5. Styling
  STYLE       ON <location> [WHERE <cond>] SETTING <style_params>
  ALIGN       (<cols>) TO 'left'|'right'|'center' [DECIMAL]
  WIDTH       (<cols>) TO px(<n>) | pct(<n>)
  BORDER      ON <location> SETTING sides => ..., border_color => ...

  -- 6. Global
  OPTION      <gt_option> => <value>, ...
  THEME       <theme_kind> [SETTING ...]
  INTERACTIVE SETTING use_search => true, ...

  -- 7. Output
  SPLIT BY    <col>            -- produces a table group
  RENDER TO   html | latex | rtf | word | image | pdf | gtable
  EXPORT TO   '<file_path>' [SETTING ...]
```

### 0.2 Clause Categories

| Category | Clauses | Role |
|---|---|---|
| **Structure** | `STUB`, `STUBHEAD`, `SPANNER`, `COLUMN`, `HIDE`, `MOVE`, `MERGE`, `NANOPLOT` | Shape rows and columns |
| **Rows** | `ROWS ADD/HIDE/ORDER`, `SUMMARY`, `GRAND SUMMARY` | Add / reorder / aggregate rows |
| **Cells** | `FORMAT`, `SUBSTITUTE`, `TEXT`, `COLOR DATA` | Transform cell values (3-phase pipeline) |
| **Annotation** | `HEADER`, `FOOTNOTE`, `SOURCE`, `CAPTION` | Add titles, notes, citations |
| **Styling** | `STYLE`, `ALIGN`, `WIDTH`, `BORDER` | Visual appearance |
| **Global** | `OPTION`, `THEME`, `INTERACTIVE` | Table-wide configuration |
| **Output** | `SPLIT`, `RENDER TO`, `EXPORT TO` | Render / export |

### 0.3 Clause Composition Rules

- **Order**: After `TABULATE`, clauses may appear in any order; grouping by category is recommended for readability.
- **Last-call-wins**: `FORMAT` on the same cell — last clause wins.
- **Additive**: `FOOTNOTE`, `SOURCE`, `STYLE`, `SUMMARY` accumulate.
- **Single-value**: `HEADER`, `CAPTION`, `STUB`, `RENDER TO` — last clause replaces prior.
- **Targeting**: Cell-precise clauses use `ON (cols)` + optional `WHERE` or `ON <location_helper>`.
- **Settings**: All parameters use arrow notation `param => value`; lists use parentheses `(a, b, c)`.

### 0.4 The Three-Phase Cell Pipeline

```
raw value  ──FORMAT──▶  display string  ──SUBSTITUTE──▶  string  ──TEXT──▶  final string
  (phase 1)              (phase 2)                       (phase 3)
```

### 0.5 Quick Example

```sql
SELECT continent, country, gdp, population FROM economies WHERE year = 2024
TABULATE
  STUB country GROUP continent                                      -- structure
  HEADER title => '**World Economies, 2024**'                       -- annotation
  FORMAT currency ON (gdp)        SETTING currency => 'USD'         -- cell
  FORMAT integer  ON (population) SETTING sep_mark => ','
  SUBSTITUTE missing ON (gdp, population) SETTING with => '—'
  COLOR DATA ON (gdp) SETTING palette => 'viridis'                  -- cell
  SUMMARY ON continent SETTING fns => ('mean'), columns => (gdp)    -- rows
  SOURCE 'World Bank, 2025'                                         -- annotation
  THEME stylize_color SETTING color => 'blue'                       -- global
  RENDER TO html                                                    -- output
```

---

## 1. Overview

gtsql extends SQL with **display-table** specification clauses, enabling data retrieval and presentation-quality table construction in a single, composable query. Where ggsql adapts the *Grammar of Graphics*, gtsql adapts the **Grammar of Tables** — the systematic decomposition of a display table into a fixed set of **parts** (header, stub, column labels, body, footer) populated by independent, reusable operators (formatters, substitutions, stylers, summaries).

### 1.1 Query Architecture

gtsql queries are split at the `TABULATE` boundary:

```
┌──────────────────────────────────────┐
│  SQL PORTION (before TABULATE)       │
│  - Standard SQL: SELECT, WITH, etc.  │
│  - Executed by database backend      │
└──────────────────────────────────────┘
              ↓
┌──────────────────────────────────────┐
│  TABULATE PORTION (after TABULATE)   │
│  - Table specification               │
│  - Parsed into typed AST             │
│  - Rendered via pluggable writers    │
│    (HTML, LaTeX, RTF, Word, image)   │
└──────────────────────────────────────┘
```

### 1.2 Key Design Principles

- **Declarative**: Specify *what* the table should communicate, not *how* to draw it.
- **Composable**: Stack independent formatters, substitutions, stylers, and summaries; later clauses override earlier ones on the same target (last-call-wins for formatters; additive for footnotes / source notes / styles).
- **Part-oriented**: Every clause attaches to a named part of the table (`stub`, `column_labels`, `body`, `row_groups`, `footer`, etc.).
- **Location-targeted**: Cell-precise operations use **location helpers** (`cells_body`, `cells_column_labels`, …) analogous to ggsql's aesthetic scales.
- **Data-native**: Aggregation and filtering stay in the database whenever possible.
- **Writer-agnostic**: One spec → many output formats.
- **AI-friendly**: Structured syntax suitable for code generation.

### 1.3 Comparison to ggsql

| ggsql concept | gtsql analogue |
|---|---|
| `VISUALISE` (boundary) | `TABULATE` (boundary) |
| `DRAW geom` (layer) | `COLUMN`, `SUMMARY`, `ROWS` (content) |
| `MAPPING col AS aesthetic` | `COLUMN col AS label`, `STUB col` |
| `SCALE` (data → visual) | `FORMAT`, `SUBSTITUTE`, `COLOR DATA` |
| `FACET` (small multiples) | `GROUP BY` (row groups) / `SPLIT` |
| `PROJECT` (coordinate system) | `RENDER TO` (output writer) |
| `LABEL` (titles) | `HEADER`, `CAPTION`, `SOURCE` |
| Aesthetic settings (`fill => 'red'`) | `STYLE … SETTING fill => 'red'` |
| `SETTING param => value` | `SETTING param => value` (identical) |

---

## 2. Query Structure

### 2.1 Complete Query Syntax

```
[SQL_PORTION]
TABULATE_STATEMENT [TABULATE_STATEMENT]*
```

### 2.2 Full Example

```sql
-- SQL PORTION: Data retrieval
SELECT continent, country, gdp, population, year
FROM economies
WHERE year = 2024

-- TABULATE PORTION: Table specification
TABULATE
  STUB country GROUP continent
  HEADER
    title => '**World Economies, 2024**',
    subtitle => 'GDP and population by country'
  SPANNER 'Demographics' OVER (population)
  SPANNER 'Economy' OVER (gdp)
  COLUMN country  AS 'Country'
  COLUMN gdp      AS 'GDP (USD)'
  COLUMN population AS 'Population'
  FORMAT currency ON (gdp)        SETTING currency => 'USD', decimals => 0
  FORMAT integer  ON (population) SETTING sep_mark => ','
  SUBSTITUTE missing ON (gdp, population) SETTING with => '—'
  COLOR DATA ON (gdp) SETTING palette => 'viridis', direction => 1
  SUMMARY ON continent
    SETTING fns => ('mean', 'sum'),
    columns => (gdp, population)
  GRAND SUMMARY
    SETTING fns => ('sum'),
    columns => (gdp, population),
    label  => 'World Total'
  FOOTNOTE 'Estimated by IMF' AT cells_body(columns => gdp, rows => country = 'Argentina')
  SOURCE 'World Bank, 2025'
  CAPTION 'Table 1: National accounts'
  OPTION row_striping => true, table_width => pct(100)
  THEME stylize_color SETTING color => 'blue'
  RENDER TO html
```

---

## 3. TABULATE Statement

### 3.1 Syntax

```
TABULATE [global_config] [FROM data_source]
  [tab_clause]*
```

**Aliases**: `TABLE`, `RENDER_TABLE` (case-insensitive).
**British/American**: gtsql accepts both `COLOR` and `COLOUR`, `SUMMARISE` and `SUMMARIZE`.

### 3.2 Global Config

Optional configuration that applies to the whole table and is inherited by clauses. Equivalent to gt's `gt()` arguments.

```
TABULATE
  SETTING
    rowname_col   => country,
    groupname_col => continent,
    row_group_as_column => false,
    auto_align    => true,
    id            => 'tbl_econ'
```

Each of these can also be expressed with shorthand clauses:

- `STUB col` → `rowname_col => col`
- `STUB col GROUP gcol` → `rowname_col => col, groupname_col => gcol`
- `ID 'name'` → `id => 'name'`

### 3.3 FROM Clause

Specifies the global data source (mirrors ggsql §3.3).

#### 3.3.1 Table / CTE Reference

```
TABULATE FROM economies
TABULATE FROM my_cte
```

#### 3.3.2 File Path Reference

```
TABULATE FROM 'data/economies.parquet'
TABULATE FROM 'data/economies.csv'
```

#### 3.3.3 Interaction with SELECT

**Rule**: Cannot use both `SELECT … TABULATE` and `TABULATE FROM`.

Valid (SELECT provides data):
```sql
SELECT * FROM economies TABULATE COLUMN gdp
```

Valid (TABULATE FROM provides data):
```sql
TABULATE FROM economies COLUMN gdp
```

Invalid:
```sql
SELECT * FROM economies TABULATE FROM economies  -- ERROR
```

#### 3.3.4 FROM Shorthand Injection

```sql
TABULATE FROM economies COLUMN gdp
```
is automatically rewritten to:
```sql
SELECT * FROM economies
TABULATE FROM economies COLUMN gdp
```

---

## 4. Tabulation Clauses (tab_clause)

After `TABULATE`, **clause order is arbitrary** but a recommended logical grouping is:

1. Structure (`STUB`, `GROUP`, `SPANNER`, `COLUMN`, `HIDE`, `MOVE`, `MERGE`)
2. Content shaping (`ROWS ADD`, `ROWS HIDE`, `SUMMARY`, `GRAND SUMMARY`)
3. Cell transformation (`FORMAT`, `SUBSTITUTE`, `TEXT`, `COLOR DATA`)
4. Annotation (`HEADER`, `STUBHEAD`, `FOOTNOTE`, `SOURCE`, `CAPTION`)
5. Styling (`STYLE`, `ALIGN`, `WIDTH`, `BORDER`)
6. Global (`OPTION`, `THEME`, `INTERACTIVE`)
7. Output (`RENDER TO`, `EXPORT TO`)

The parser executes clauses in the **declared order** within each phase (last-write-wins for `FORMAT`; additive for `STYLE`, `FOOTNOTE`, `SOURCE`).

---

## 5. STUB Clause — Row Identity

### 5.1 Syntax

```
STUB column [GROUP groupname_column] [SETTING param => value, ...]
```

### 5.2 Purpose

Defines the **stub** (leftmost identity column) and optionally the **row group** column. Equivalent to gt's `rowname_col` and `groupname_col` arguments to `gt()`.

### 5.3 Settings

| Setting | Type | Description |
|---|---|---|
| `indent` | int | Indent stub labels by N pixels |
| `as_column` | bool | Render row group as its own column |
| `label` | string | Stubhead label (shorthand for `STUBHEAD`) |

### 5.4 Examples

```sql
TABULATE STUB country
TABULATE STUB country GROUP continent
TABULATE STUB country GROUP continent SETTING indent => 12, label => 'Country'
```

---

## 6. STUBHEAD Clause

### 6.1 Syntax

```
STUBHEAD 'label_text'
```

### 6.2 Purpose

Places a label in the stubhead cell (intersection of stub + column labels).

### 6.3 Example

```sql
TABULATE
  STUB landmass
  STUBHEAD 'Landmass'
```

---

## 7. HEADER Clause — Title & Subtitle

### 7.1 Syntax

```
HEADER param => value, ...
```

### 7.2 Supported Keys

| Key | Description |
|---|---|
| `title` | Main table title |
| `subtitle` | Secondary title shown below the title |
| `preheader` | Pre-header text (optional, e.g. dateline) |

Values may be wrapped with the markup helpers `md('…')`, `html('…')`, `latex('…')`.

### 7.3 Example

```sql
HEADER
  title    => md('**World Population**, 2024'),
  subtitle => 'Top-10 countries'
```

---

## 8. SPANNER Clause — Column Spanners

### 8.1 Syntax

```
SPANNER 'label' OVER (col, col, ...) [SETTING param => value, ...]
SPANNER DELIM 'delimiter' [SETTING param => value]   -- delimited-name shortcut
```

### 8.2 Purpose

Adds a **spanner column label** above one or more columns; equivalent to `tab_spanner()` / `tab_spanner_delim()`.

### 8.3 Settings

| Setting | Type | Description |
|---|---|---|
| `id` | string | Spanner id for later targeting |
| `level` | int | Stacking level (multi-tier spanners) |
| `gather` | bool | Move spanned columns adjacent to each other |
| `replace` | bool | Replace existing spanner with same id |

### 8.4 Nested Spanners

```sql
SPANNER 'Measurement' OVER (ozone, solar, wind, temp)
SPANNER 'Atmospheric' OVER (ozone, solar) SETTING level => 2
```

Level 1 is the innermost spanner (closest to column labels). Higher levels stack above.

### 8.5 Delimited Form

```sql
SPANNER DELIM '_'
```
splits column names like `temp_min` / `temp_max` into spanner `temp` over labels `min` / `max`.

---

## 9. COLUMN Clause — Column Declaration

### 9.1 Syntax

```
COLUMN column_ref [AS 'label']
  [SETTING param => value, ...]
  [UNITS 'unit_str']
```

### 9.2 Purpose

Declares (or relabels / configures) a body column. Wraps `cols_label()`, `cols_align()`, `cols_width()`, `cols_units()`.

### 9.3 Settings

| Setting | Type | Description |
|---|---|---|
| `align` | enum | `'left'`, `'right'`, `'center'`, `'auto'` |
| `align_decimal` | bool | Align numerics along the decimal mark |
| `width` | string | CSS width: `px(120)`, `pct(20)`, `'120px'` |
| `hidden` | bool | Hide the column (still queryable) |
| `na_value` | string | Per-column missing-value placeholder |

### 9.4 Multi-column Form

```
COLUMN (col, col, ...) SETTING align => 'right'
```

### 9.5 Examples

```sql
COLUMN gdp AS 'GDP (USD)' SETTING align => 'right', width => px(140)
COLUMN (population, density) SETTING align_decimal => true
COLUMN temperature AS 'Temp' UNITS 'deg*F'
```

---

## 10. HIDE / UNHIDE / MOVE — Column Layout

### 10.1 HIDE

```
HIDE COLUMN col, col, ...
HIDE ROWS WHERE condition
```

### 10.2 UNHIDE

```
UNHIDE COLUMN col, col, ...
UNHIDE ROWS WHERE condition
```

### 10.3 MOVE

```
MOVE COLUMN col, col, ... TO START
MOVE COLUMN col, col, ... TO END
MOVE COLUMN col AFTER other_col
MOVE COLUMN col BEFORE other_col
```

Wraps `cols_move_to_start()`, `cols_move_to_end()`, `cols_move()`.

### 10.4 Example

```sql
MOVE COLUMN (year, month, day) TO START
HIDE COLUMN internal_id
```

---

## 11. MERGE Clause — Combining Columns

### 11.1 Syntax

```
MERGE merge_kind (col1, col2, ...) [AS 'label']
  [SETTING param => value, ...]
```

### 11.2 Merge Kinds

| Kind | Wraps | Purpose |
|---|---|---|
| `values` | `cols_merge()` | Generic merge via pattern |
| `uncert` | `cols_merge_uncert()` | `value ± uncertainty` |
| `range`  | `cols_merge_range()` | `low – high` |
| `n_pct`  | `cols_merge_n_pct()` | `count (pct%)` |

### 11.3 Common Settings

| Setting | Description |
|---|---|
| `pattern` | Substitution pattern; placeholders `{1}`, `{2}`, `{3}` |
| `sep`     | Separator string (shorthand for simple patterns) |
| `autohide`| Hide secondary columns automatically (default `true`) |

### 11.4 Examples

```sql
MERGE values  (measure, unit)        SETTING pattern => '{1} {2}'   AS 'Reading'
MERGE uncert  (estimate, std_err)    AS 'Estimate'
MERGE range   (low, high)            SETTING sep => '–'              AS 'Range'
MERGE n_pct   (count, percent)       AS 'Count'
```

---

## 12. FORMAT Clause — Cell Formatting

### 12.1 Syntax

```
FORMAT fmt_kind
  ON (col, col, ...)
  [WHERE row_condition]
  [SETTING param => value, ...]
  [LOCALE 'locale_code']
```

### 12.2 Semantics

- Applied **once per cell** (last `FORMAT` for that cell wins; mirrors gt's "last-call-wins" rule).
- `WHERE` is a SQL expression evaluated against body rows — equivalent to gt's `rows` argument.
- `LOCALE` shortcut sets the `locale` setting.

### 12.3 Format Kinds

#### 12.3.1 Numeric

| Kind | gt analogue | Notable settings |
|---|---|---|
| `number` | `fmt_number` | `decimals`, `sep_mark`, `dec_mark`, `force_sign`, `scale_by`, `suffixing` |
| `integer` | `fmt_integer` | `sep_mark`, `force_sign`, `scale_by` |
| `scientific` | `fmt_scientific` | `decimals`, `exp_style` |
| `engineering` | `fmt_engineering` | `decimals` |
| `number_si` | `fmt_number_si` | `decimals` |
| `percent` | `fmt_percent` | `decimals`, `scale_values`, `symbol` |
| `partsper` | `fmt_partsper` | `to_units`, `decimals` |
| `fraction` | `fmt_fraction` | `accuracy`, `simplify`, `layout` |
| `currency` | `fmt_currency` | `currency`, `decimals`, `placement`, `use_subunits` |
| `roman` | `fmt_roman` | `case` |
| `index` | `fmt_index` | `base` |
| `spelled_num` | `fmt_spelled_num` | `cap_initial` |
| `bytes` | `fmt_bytes` | `standard`, `decimals` |

#### 12.3.2 Temporal

| Kind | gt analogue | Settings |
|---|---|---|
| `date` | `fmt_date` | `style` (1–14 or string), `format` (custom) |
| `time` | `fmt_time` | `style`, `format` |
| `datetime` | `fmt_datetime` | `date_style`, `time_style`, `sep` |
| `duration` | `fmt_duration` | `input_units`, `output_units`, `duration_style` |

#### 12.3.3 Categorical / Textual

| Kind | gt analogue |
|---|---|
| `bins` | `fmt_bins` |
| `tf` | `fmt_tf` |
| `markdown` | `fmt_markdown` |
| `units` | `fmt_units` |
| `chem` | `fmt_chem` |
| `url` | `fmt_url` |
| `email` | `fmt_email` |
| `image` | `fmt_image` |
| `flag` | `fmt_flag` |
| `country` | `fmt_country` |
| `icon` | `fmt_icon` |
| `passthrough` | `fmt_passthrough` |
| `auto` | `fmt_auto` |

#### 12.3.4 Custom

```
FORMAT WITH 'js_or_lambda_expr' ON (col)
```
maps to `fmt()` with a user-provided formatter.

### 12.4 Examples

```sql
FORMAT number   ON (revenue) SETTING decimals => 2, sep_mark => ',', suffixing => true
FORMAT currency ON (price)   SETTING currency => 'EUR', placement => 'right'
FORMAT date     ON (signup)  SETTING style => 'iso'   LOCALE 'fr-FR'
FORMAT percent  ON (rate)    WHERE country = 'JPN' SETTING decimals => 1
FORMAT auto     ON (*)
```

---

## 13. SUBSTITUTE Clause — Value Replacement

### 13.1 Syntax

```
SUBSTITUTE sub_kind
  ON (col, col, ...)
  [WHERE row_condition]
  SETTING param => value, ...
```

### 13.2 Sub Kinds

| Kind | gt analogue | Common settings |
|---|---|---|
| `missing` | `sub_missing` | `with` (replacement string, default `'—'`) |
| `zero` | `sub_zero` | `with` |
| `small_vals` | `sub_small_vals` | `threshold`, `small_pattern`, `format` |
| `large_vals` | `sub_large_vals` | `threshold`, `large_pattern`, `format` |
| `values` | `sub_values` | `values` (list), `replacement` |

### 13.3 Examples

```sql
SUBSTITUTE missing ON (gdp, population) SETTING with => 'n/a'
SUBSTITUTE small_vals ON (probability) SETTING threshold => 0.001, small_pattern => '<0.001'
SUBSTITUTE values ON (status) SETTING values => ('OK', 'ERR'), replacement => ('✓', '✗')
```

---

## 14. TEXT Clause — String Transformations

### 14.1 Syntax

```
TEXT REPLACE   ON location SETTING pattern => 'regex', with => 'replacement'
TEXT CASE WHEN ON location SETTING cases   => (cond => value, ...)
TEXT CASE MATCH ON location SETTING values => ('a' => 'A', 'b' => 'B', default => 'other')
TEXT TRANSFORM ON location SETTING fn      => 'js_or_lambda'
```

### 14.2 Purpose

Phase-3 cell transformation (operates on **strings** after `FORMAT` and `SUBSTITUTE`). Wraps `text_replace()`, `text_case_when()`, `text_case_match()`, `text_transform()`.

### 14.3 Example

```sql
TEXT REPLACE ON cells_body(columns => name)
  SETTING pattern => '_', with => ' '
```

---

## 15. COLOR DATA Clause — Cell Colorization

### 15.1 Syntax

```
COLOR DATA ON (col, col, ...)
  [WHERE row_condition]
  SETTING param => value, ...
```

### 15.2 Settings

| Setting | Description |
|---|---|
| `palette` | Named palette (`'viridis'`, `'Set2'`, `'RdBu'`) or list of colors |
| `domain` | Explicit value range `[min, max]` |
| `bins` | Number of discrete bins (binned scale) |
| `na_color` | Color for missing values |
| `direction` | `1` or `-1` (palette direction) |
| `method` | `'numeric'`, `'bin'`, `'quantile'`, `'factor'` |
| `target` | `'fill'` (default), `'color'`, `'both'` |
| `autocolor_text` | Auto-pick text color for contrast (default `true`) |
| `apply_to` | `'fill'` (default) or `'text'` |

### 15.3 Example

```sql
COLOR DATA ON (gdp, population)
  SETTING palette => 'viridis', method => 'quantile', bins => 5
```

This is gtsql's analogue to ggsql's `SCALE` clause — it maps data values to a visual property of a cell.

---

## 16. ROWS Clause — Row-Level Operations

### 16.1 Syntax

```
ROWS ADD (col => value, ...) [AT position]
ROWS HIDE WHERE condition
ROWS UNHIDE WHERE condition
ROWS ORDER BY col [ASC|DESC], ...
ROWS GROUP ORDER BY group_expr
```

### 16.2 Purpose

Wraps `rows_add()`, `rows_hide()`, `rows_unhide()`, `row_order()`, `row_group_order()`.

### 16.3 ROWS ADD Positions

| Position | Meaning |
|---|---|
| `START` / `END` | Prepend / append |
| `AFTER n` / `BEFORE n` | Index-based |
| `IN GROUP 'g' AT END` | Within a row group |

### 16.4 Examples

```sql
ROWS ADD (country => 'Eurozone', gdp => null, population => null) AT END
ROWS HIDE WHERE population < 1000
ROWS ORDER BY gdp DESC
ROWS GROUP ORDER BY ('Asia', 'Europe', 'Africa', 'Americas')
```

---

## 17. SUMMARY / GRAND SUMMARY Clauses

### 17.1 SUMMARY Syntax

```
SUMMARY [ON group_col]
  SETTING
    fns     => (fn, fn, ...),
    columns => (col, col, ...),
    [labels => (label, label, ...)],
    [fmt    => fmt_spec],
    [position => 'top' | 'bottom']
```

### 17.2 GRAND SUMMARY Syntax

```
GRAND SUMMARY
  SETTING
    fns     => (fn, ...),
    columns => (col, ...),
    [label  => 'Total'],
    [position => 'top' | 'bottom']
```

### 17.3 Aggregation Functions

Reuses the ggsql aggregate vocabulary (§5.5.3) plus gt-native names:

- **Basic**: `count`, `sum`, `min`, `max`, `range`, `mid`
- **Central**: `mean`, `median`, `geomean`, `harmean`, `rms`
- **Spread**: `sdev`, `var`, `iqr`, `se`
- **Percentiles**: `p05`, `p10`, `p25`, `p50`, `p75`, `p90`, `p95`
- **Sequence**: `first`, `last`, `diff`

Custom SQL: `'sql:AVG(col) FILTER (WHERE flag)'`.

### 17.4 Targeted Aggregation (per-column)

```
SUMMARY SETTING fns => ('gdp:mean', 'population:sum')
```

### 17.5 Examples

```sql
-- Per-continent means and sums
SUMMARY ON continent
  SETTING fns => ('mean', 'sum'),
          columns => (gdp, population)

-- Grand total at the bottom
GRAND SUMMARY
  SETTING fns => ('sum'),
          columns => (gdp, population),
          label  => 'World Total',
          position => 'bottom'
```

---

## 18. NANOPLOT Clause — Inline Sparklines

### 18.1 Syntax

```
NANOPLOT new_col
  FROM (col, col, ...) | LONG (id_col, value_col)
  SETTING
    plot_type => 'line' | 'bar' | 'boxplot' | 'ref' | 'auto',
    [reference_line => 'mean' | num],
    [reference_area => ('q1', 'q3')],
    [missing_vals => 'gap' | 'zero' | 'remove'],
    [autoscale => true]
```

### 18.2 Purpose

Adds a generated column of inline sparkline plots; wraps `cols_nanoplot()`.

### 18.3 Example

```sql
NANOPLOT trend
  FROM (q1_sales, q2_sales, q3_sales, q4_sales)
  SETTING plot_type => 'line', reference_line => 'mean'
```

---

## 19. FOOTNOTE / SOURCE / CAPTION Clauses

### 19.1 FOOTNOTE

```
FOOTNOTE 'text' [AT location] [SETTING mark => 'a' | 1 | '*']
```

- Multiple `FOOTNOTE` clauses are **additive**.
- `AT` accepts any location helper (see §22). Omitting `AT` attaches an unanchored footnote.

### 19.2 SOURCE

```
SOURCE 'citation' [SETTING ref => 'arabic' | 'roman' | 'letter']
```

Each `SOURCE` clause adds another source-note line.

### 19.3 CAPTION

```
CAPTION 'text'
```

A single caption (last-call-wins).

### 19.4 Example

```sql
FOOTNOTE md('**Estimated** by IMF.') AT cells_body(columns => gdp, rows => year < 1990)
FOOTNOTE 'Provisional.' AT cells_column_labels(columns => population)
SOURCE 'World Bank, Open Data, 2025-04.'
SOURCE md('See *IMF World Economic Outlook* for methodology.')
CAPTION 'Table 1: National accounts overview'
```

---

## 20. STYLE Clause — Cell Styling

### 20.1 Syntax

```
STYLE ON location
  SETTING style_param => value, ...
```

### 20.2 Style Parameters

Mirrors gt's `cell_text()`, `cell_fill()`, `cell_borders()` helpers, flattened:

**Text** (`cell_text`):
- `color`, `font`, `size`, `weight` (`'normal'`, `'bold'`, int 100–900)
- `style` (`'normal'`, `'italic'`, `'oblique'`)
- `align`, `v_align`, `decorate` (`'underline'`, `'overline'`, `'line-through'`)
- `transform` (`'uppercase'`, `'lowercase'`, `'capitalize'`)
- `whitespace`, `indent`

**Fill** (`cell_fill`):
- `fill`, `alpha`

**Borders** (`cell_borders`):
- `sides` (`'all'`, `'top'`, `'bottom'`, `'left'`, `'right'`, or list)
- `border_color`, `border_style` (`'solid'`, `'dashed'`, `'dotted'`, `'double'`), `border_width`

### 20.3 Conditional Styling (per-row)

```
STYLE ON cells_body(columns => gdp)
  WHERE gdp > 1e13
  SETTING fill => 'gold', weight => 'bold'
```

### 20.4 Example

```sql
STYLE ON cells_column_labels()
  SETTING weight => 'bold', fill => '#eef2ff'

STYLE ON cells_body(columns => gdp) WHERE gdp = max(gdp)
  SETTING fill => 'yellow', weight => 'bold'

STYLE ON cells_body() SETTING border_color => '#ddd', sides => 'bottom'
```

---

## 21. ALIGN / WIDTH / BORDER — Shorthand Stylers

Convenience clauses that desugar to `STYLE` / `COLUMN SETTING`:

```
ALIGN  (col, col, ...) TO 'right' [DECIMAL]
WIDTH  (col, col, ...) TO px(120)
BORDER ON location SETTING sides => 'bottom', border_color => '#000'
```

```sql
ALIGN (gdp, population) TO 'right'
WIDTH (country) TO pct(30)
BORDER ON cells_row_groups() SETTING sides => 'top', border_width => px(2)
```

---

## 22. Location Helpers

Locations are gtsql's analogue to ggsql aesthetics — they identify *where* in the table an operation applies.

### 22.1 Available Helpers

| Helper | Targets |
|---|---|
| `cells_body([columns =>], [rows =>])` | Body data cells |
| `cells_column_labels([columns =>])` | Column labels |
| `cells_column_spanners([spanners =>])` | Spanner labels |
| `cells_stub([rows =>])` | Stub cells |
| `cells_stubhead()` | Stubhead |
| `cells_row_groups([groups =>])` | Row group label rows |
| `cells_summary([groups =>], [columns =>], [rows =>])` | Per-group summaries |
| `cells_grand_summary([columns =>], [rows =>])` | Grand summary |
| `cells_stub_summary([groups =>])` | Stub side of group summaries |
| `cells_stub_grand_summary()` | Stub side of grand summary |
| `cells_title([part =>])` | Title / subtitle |
| `cells_footnotes()` | Footnote area |
| `cells_source_notes()` | Source-note area |

### 22.2 Composing Multiple Locations

```
STYLE ON (cells_column_labels(), cells_row_groups()) SETTING weight => 'bold'
```

### 22.3 Row Selection in Locations

`rows =>` accepts:

- A vector of row labels: `rows => ('USA', 'CAN')`
- Integer indices: `rows => 1:5`
- A SQL boolean expression: `rows => gdp > 1e12 AND continent = 'Asia'`
- The keyword `EVERYTHING` or omission (all rows)

### 22.4 Column Selection

`columns =>` accepts a list, a single name, a glob (`'gdp_*'`), or selector keywords: `STARTS_WITH('q')`, `ENDS_WITH('_pct')`, `CONTAINS('rate')`, `MATCHES('^[A-Z]')`, `EVERYTHING`, `WHERE_NUMERIC`, `WHERE_CHARACTER`.

---

## 23. OPTION Clause — Global Options

### 23.1 Syntax

```
OPTION param => value, ...
```

Wraps `tab_options()`. Any gt option is settable. Common keys:

| Key | Description |
|---|---|
| `table_width` | Overall width (`pct(100)`, `px(800)`) |
| `table_background_color` | Background color |
| `row_striping` | Boolean — zebra striping in body |
| `row_group_as_column` | Boolean |
| `heading_align` | `'left'` / `'center'` / `'right'` |
| `column_labels_font_weight` | Weight |
| `data_row_padding` | Vertical padding |
| `quarto_disable_processing` | Boolean |

### 23.2 Example

```sql
OPTION
  table_width => pct(100),
  row_striping => true,
  heading_align => 'left',
  data_row_padding => px(4)
```

---

## 24. THEME Clause — Quick Styling

### 24.1 Syntax

```
THEME theme_kind [SETTING param => value, ...]
```

### 24.2 Theme Kinds

| Kind | Wraps |
|---|---|
| `stylize_color` | `opt_stylize()` |
| `striping` | `opt_row_striping()` |
| `all_caps` | `opt_all_caps()` |
| `align_header` | `opt_align_table_header()` |
| `vertical_padding` | `opt_vertical_padding()` |
| `horizontal_padding` | `opt_horizontal_padding()` |
| `table_lines` | `opt_table_lines()` |
| `table_outline` | `opt_table_outline()` |
| `font` | `opt_table_font()` |
| `interactive` | `opt_interactive()` |
| `footnote_marks` | `opt_footnote_marks()` |
| `css` | `opt_css()` |

### 24.3 Examples

```sql
THEME stylize_color SETTING color => 'blue', style => 1
THEME interactive   SETTING use_search => true, use_filters => true, page_size_default => 25
THEME font          SETTING font => google_font('Inter')
THEME css           SETTING css => '.gt_row { font-variant-numeric: tabular-nums; }'
```

---

## 25. RENDER / EXPORT — Output

### 25.1 Syntax

```
RENDER TO writer_name [SETTING param => value, ...]
EXPORT TO 'file_path' [SETTING param => value, ...]
```

### 25.2 Writer Names

| Writer | gt analogue |
|---|---|
| `html` | `as_raw_html()` / default |
| `latex` | `as_latex()` |
| `rtf` | `as_rtf()` |
| `word` | `as_word()` |
| `gtable` | `as_gtable()` |
| `image` | `gtsave()` to PNG / JPG |
| `pdf` | `gtsave()` to PDF |

### 25.3 Settings

Common settings: `path`, `dpi`, `expand`, `zoom`, `vwidth`, `vheight`.

### 25.4 Examples

```sql
RENDER TO html SETTING inline_css => true
EXPORT TO 'reports/q4_economies.png' SETTING dpi => 200, expand => 5
EXPORT TO 'reports/q4_economies.docx'
```

### 25.5 Inference

If neither `RENDER` nor `EXPORT` is specified, the writer is inferred from the host runtime (HTML in notebooks, the default device elsewhere).

---

## 26. SPLIT Clause — Table Groups

### 26.1 Syntax

```
SPLIT BY split_col [INTO n_chunks] [SETTING param => value, ...]
```

### 26.2 Purpose

Produces a **`gt_group`** of related tables (`gt_split()` / `gt_group()`). Each chunk inherits the parent's `TABULATE` spec.

### 26.3 Settings

| Setting | Description |
|---|---|
| `row_every_n` | Split every N rows |
| `row_slice_i` | Split at specific row indices |
| `page_size` | Equivalent paginated chunks |

### 26.4 Example

```sql
SELECT * FROM transactions
TABULATE
  STUB txn_id
  FORMAT currency ON (amount)
SPLIT BY month
RENDER TO html
```

---

## 27. INTERACTIVE Clause

### 27.1 Syntax

```
INTERACTIVE
  SETTING
    use_search    => true,
    use_filters   => true,
    use_sorting   => true,
    use_pagination => true,
    page_size_default => 10,
    use_resizers  => true,
    use_highlight => true,
    use_compact_mode => false
```

Equivalent to `opt_interactive()`. Only the HTML writer honors these settings.

---

## 28. WITH Clause — Reusable Table Variables

### 28.1 Syntax

```
WITH TABULATION name AS (TABULATE … ) [, ...]
TABULATE … USE name
```

### 28.2 Purpose

Allows pre-defining table fragments (e.g., a shared theme + format set) once and applying to multiple queries.

### 28.3 Example

```sql
WITH TABULATION econ_style AS (
  TABULATE
    FORMAT currency ON (gdp) SETTING currency => 'USD'
    THEME stylize_color SETTING color => 'blue'
    OPTION row_striping => true
)

SELECT * FROM economies_2023
TABULATE USE econ_style
  STUB country
  HEADER title => '2023'

SELECT * FROM economies_2024
TABULATE USE econ_style
  STUB country
  HEADER title => '2024'
```

---

## 29. Multiple Tables in One Query

Like ggsql's multi-`VISUALISE`, a single SQL session can emit multiple tables:

```sql
SELECT * FROM economies
TABULATE STUB country HEADER title => 'Overview'

SELECT continent, AVG(gdp) AS avg_gdp FROM economies GROUP BY continent
TABULATE STUB continent HEADER title => 'Continental averages'
```

Each `TABULATE` statement ends at the next `TABULATE` or end-of-input.

---

## 30. Data Flow and Execution

### 30.1 Pipeline

```
1. Parse query at TABULATE boundary
2. Execute SQL portion against backend
3. Materialize source frame
4. Resolve structure (STUB, GROUP, SPANNER, COLUMN, HIDE, MOVE, MERGE)
5. Add synthetic rows (ROWS ADD, SUMMARY, GRAND SUMMARY)
6. Apply value-level transformations:
     a. FORMAT (last-call-wins per cell)
     b. SUBSTITUTE
     c. COLOR DATA
     d. TEXT transformations
7. Apply STYLE / ALIGN / WIDTH / BORDER
8. Attach annotations (HEADER, STUBHEAD, FOOTNOTE, SOURCE, CAPTION)
9. Apply OPTION / THEME / INTERACTIVE
10. Render via the chosen writer (RENDER TO / EXPORT TO)
```

### 30.2 Cell-Transformation Phases

Like gt itself, gtsql defines **three transformation phases** on cell values:

| Phase | Clause | Operates on |
|---|---|---|
| 1. Formatting | `FORMAT` | Raw values → display strings |
| 2. Substitution | `SUBSTITUTE` | Special values (missing, zero, small) → replacement strings |
| 3. Text | `TEXT REPLACE` / `TEXT CASE …` / `TEXT TRANSFORM` | Final strings → transformed strings |

Within a phase, last clause for a given cell wins. Across phases, output flows 1 → 2 → 3.

### 30.3 Backend Integration

- All SQL (`SELECT`, `WITH`, joins, window functions) runs in the database.
- Summaries can be pushed down (`SUMMARY` translates to `GROUP BY` aggregations when possible).
- Only the rendered frame is materialized client-side.

---

## 31. Aesthetic Context and Internal Representation

### 31.1 Internal Part Identifiers

User-facing clause targets are normalized to internal part ids used by writers:

| User-facing | Internal id |
|---|---|
| Stub | `stub` |
| Stubhead | `stubhead` |
| Column Labels | `column_labels` |
| Spanners | `column_spanners` |
| Body | `body` |
| Row Groups | `row_groups` |
| Summaries | `summary` |
| Grand Summary | `grand_summary` |
| Header | `header` |
| Footnotes | `footnotes` |
| Source Notes | `source_notes` |

### 31.2 Location Resolution

Each location helper resolves to a `(part, columns, rows)` triple at parse time, then is broadcast to concrete `(part, cell)` pairs at build time.

---

## 32. Syntax Flexibility

### 32.1 Case Insensitivity

All keywords case-insensitive:
```
TABULATE = tabulate = TabulaTe
FORMAT   = format   = Format
```

### 32.2 British / American

- `COLOR` or `COLOUR`
- `SUMMARISE` or `SUMMARIZE`

### 32.3 Identifier Quoting

```
COLUMN "Revenue ($)" AS 'Revenue'
```

### 32.4 Helper Functions

| Helper | Meaning |
|---|---|
| `md('…')` | Interpret as Markdown |
| `html('…')` | Interpret as HTML |
| `latex('…')` | Interpret as LaTeX |
| `px(n)` | Pixel value |
| `pct(n)` | Percentage value |
| `from_column(col)` | Use per-row column values for a parameter |
| `currency(code)` | Custom currency symbol |
| `google_font(name)` | Google Fonts font stack |
| `system_fonts(name)` | Themed system font stack |

### 32.5 Line Breaking

Clauses can span multiple lines:
```sql
TABULATE
  STUB country
    GROUP continent
  COLUMN gdp
    AS 'GDP'
    SETTING align => 'right', width => px(120)
```

---

## 33. Error Handling

### 33.1 Parse-Time Errors

- Unknown clause keyword
- Unknown format/substitute kind
- Missing `ON` target where required

### 33.2 Validation Errors

- `STUB` column not present in source frame
- `SUMMARY ON` references non-existent group column
- `FORMAT` settings inconsistent with kind (e.g. `currency` on `date`)
- `RENDER TO` writer not registered

### 33.3 Execution Errors

- SQL portion error
- Custom formatter (`fn =>`) raises
- File path for `EXPORT TO` not writable

---

## 34. Best Practices

### 34.1 Query Organization

1. Write SQL first (joins, filters, aggregations).
2. Add `TABULATE` with `STUB` and `GROUP`.
3. Declare `SPANNER` / `COLUMN` structure.
4. Add `FORMAT` then `SUBSTITUTE`.
5. Layer `STYLE` and `COLOR DATA`.
6. Add `SUMMARY` / `GRAND SUMMARY`.
7. Add annotations (`HEADER`, `FOOTNOTE`, `SOURCE`).
8. Apply `OPTION` / `THEME`.
9. Choose output (`RENDER TO` / `EXPORT TO`).

### 34.2 Composition Tips

- Push aggregation into SQL when possible; reserve `SUMMARY` for groupwise summaries that need to live *inside* the displayed table.
- Use `FORMAT auto ON (*)` for exploratory tables.
- Prefer `THEME stylize_color` over many `STYLE` clauses for simple looks.
- Use `FOOTNOTE` for cell-anchored annotations and `SOURCE` for citations.

### 34.3 Performance

- `MERGE` and `NANOPLOT` are computed client-side; for very large tables, pre-aggregate in SQL.
- `INTERACTIVE` shifts pagination and filtering to the browser; consider `page_size_default` for big result sets.

---

## 35. Example Gallery

### 35.1 Minimal Table

```sql
SELECT name, size FROM islands ORDER BY size DESC LIMIT 10
TABULATE
  STUB name
  STUBHEAD 'Landmass'
  COLUMN size AS 'Size (km²)' SETTING align => 'right'
  FORMAT integer ON (size) SETTING sep_mark => ','
  HEADER title => 'Largest Landmasses', subtitle => 'Top 10'
  SOURCE 'World Almanac, 1975'
```

### 35.2 Financial Report with Spanners and Summaries

```sql
SELECT region, country, q1, q2, q3, q4 FROM revenue WHERE year = 2024
TABULATE
  STUB country GROUP region
  SPANNER 'Quarter' OVER (q1, q2, q3, q4)
  COLUMN (q1, q2, q3, q4) SETTING align => 'right'
  FORMAT currency ON (q1, q2, q3, q4) SETTING currency => 'USD', decimals => 0
  SUBSTITUTE missing ON (q1, q2, q3, q4) SETTING with => '—'
  COLOR DATA ON (q1, q2, q3, q4) SETTING palette => 'Blues', method => 'numeric'
  SUMMARY ON region SETTING fns => ('sum'), columns => (q1, q2, q3, q4)
  GRAND SUMMARY SETTING fns => ('sum'), columns => (q1, q2, q3, q4), label => 'Worldwide'
  HEADER title => md('**FY 2024 Revenue**'), subtitle => 'By quarter, USD'
  SOURCE 'Internal finance system, exported 2025-01-15'
  THEME stylize_color SETTING color => 'gray'
  OPTION row_striping => true
```

### 35.3 Scientific Table with Units and Uncertainty

```sql
SELECT compound, k_rate, k_se, temperature FROM kinetics
TABULATE
  STUB compound
  MERGE uncert (k_rate, k_se) AS 'k (cm³/molec·s)'
  COLUMN temperature AS 'T' UNITS 'K'
  FORMAT scientific ON (k_rate) SETTING decimals => 2
  FORMAT number     ON (temperature) SETTING decimals => 0
  FOOTNOTE 'Measured at 1 atm.' AT cells_column_labels(columns => temperature)
  SOURCE md('Data: *NIST Chemical Kinetics Database*')
  HEADER title => 'Rate Constants'
```

### 35.4 Clinical Trial Summary

```sql
SELECT arm, age, sex, bmi, outcome FROM rx_adsl
TABULATE
  STUB arm
  COLUMN (age, bmi) SETTING align => 'right'
  FORMAT number  ON (age, bmi) SETTING decimals => 1
  FORMAT percent ON (outcome)  SETTING scale_values => false, decimals => 1
  SUMMARY ON arm SETTING fns => ('mean', 'median', 'sdev'), columns => (age, bmi)
  COLOR DATA ON (outcome) SETTING palette => 'RdYlGn', domain => [0, 1]
  HEADER title => 'Trial Outcomes by Arm'
  SOURCE 'Protocol XYZ-001'
  THEME stylize_color SETTING color => 'green'
  RENDER TO word
```

### 35.5 Interactive HTML Dashboard Table

```sql
SELECT * FROM gtcars
TABULATE
  STUB model GROUP mfr
  HIDE COLUMN (drivetrain, bdy_style)
  FORMAT currency ON (msrp) SETTING currency => 'USD', decimals => 0
  FORMAT number   ON (mpg_c, mpg_h) SETTING decimals => 1
  COLOR DATA ON (msrp) SETTING palette => 'viridis'
  NANOPLOT trend FROM (mpg_c, mpg_h) SETTING plot_type => 'bar'
  HEADER title => 'GT Cars'
  THEME interactive SETTING use_search => true, use_filters => true, page_size_default => 15
  RENDER TO html
```

### 35.6 Faceted (Split) Monthly Reports

```sql
SELECT month, region, sales FROM monthly_sales WHERE year = 2024
TABULATE
  STUB region
  FORMAT currency ON (sales)
  SUMMARY ON region SETTING fns => ('sum'), columns => (sales)
SPLIT BY month
EXPORT TO 'out/monthly_{month}.html'
```

---

## 36. Grammar-of-Tables Mapping

gtsql maps Grammar-of-Tables concepts to SQL:

| GoT Concept | gtsql Implementation |
|---|---|
| **Data** | `SELECT …` or `TABULATE FROM` |
| **Stub** | `STUB` |
| **Row Groups** | `STUB GROUP` / `ROWS GROUP ORDER BY` |
| **Column Labels** | `COLUMN`, `cols_label` via `AS` |
| **Spanners** | `SPANNER` |
| **Body** | implicit (the data frame minus stub) |
| **Formatters** | `FORMAT`, `SUBSTITUTE`, `TEXT` |
| **Colorization** | `COLOR DATA` |
| **Summaries** | `SUMMARY`, `GRAND SUMMARY` |
| **Annotations** | `HEADER`, `FOOTNOTE`, `SOURCE`, `CAPTION`, `STUBHEAD` |
| **Style / Theme** | `STYLE`, `ALIGN`, `WIDTH`, `BORDER`, `THEME`, `OPTION` |
| **Output** | `RENDER TO`, `EXPORT TO` |
| **Table Groups** | `SPLIT`, `WITH TABULATION` |

---

## 37. Implementation Notes

### 37.1 Parser Architecture (recommended)

Same architecture as ggsql:

- **grammar.js**: tree-sitter rules for the `TABULATE` block.
- **builder.rs**: walks the parse tree, builds a typed AST of `Clause` variants (`Stub`, `Header`, `Format`, `Style`, …).
- **source_tree.rs**: parse-once wrapper with query API.

### 37.2 AST Transformation

During build:

1. `STUB col GROUP gcol` lifted to the global `Config` node.
2. Location helpers eagerly normalized to `(part, columns, rows)` triples.
3. `FORMAT` clauses with overlapping targets merged so the *last* clause's resolved per-cell formatter wins.
4. `SUMMARY` clauses pushed to the data layer when expressible in SQL.
5. Writer chosen by `RENDER TO` / `EXPORT TO` / runtime default.

### 37.3 Query Splitting

The SQL portion ends at the `TABULATE` keyword. The splitter recognizes the same set of SQL constructs ggsql does (full `SELECT`, CTEs, set ops, subqueries, joins).

### 37.4 Writers

A writer is a function `(TableAst, Frame) → Output`. Built-in writers:

- `html` (default) — emits gt-compatible HTML.
- `latex`, `rtf`, `word`, `image`, `pdf`, `gtable`.

Third-party writers register via a plugin interface symmetric to ggsql's.

---

## 38. References

- **gt Documentation**: https://gt.rstudio.com/
- **gt Reference Index**: https://gt.rstudio.com/reference/index.html
- **Intro to gt**: https://gt.rstudio.com/articles/gt.html
- **ggsql Syntax**: see `GGSQL_SYNTAX.md` (companion specification)
- **Grammar of Graphics**: Wilkinson, L. (2005)
- **Grammar of Tables**: Iannone, R. et al., *gt* package design notes.
