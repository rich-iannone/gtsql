# GTSQL: gt Integration in ggsql

## Overview

This document specifies the design for integrating **gt** (https://gt.rstudio.com) table-building functionality into **ggsql** (https://ggsql.org). The entry point is the `TABULATE` keyword, which parallels `VISUALISE` but produces structured data tables instead of visualizations.

Just as `VISUALISE` maps to the grammar of graphics, `TABULATE` maps to the **grammar of tables** embodied by gt. The design follows ggsql's existing syntactic patterns (SQL-native keywords, `SETTING` clauses with `=>` arrows, `FROM` for data sources, etc.) while exposing gt's rich table-construction API.

---

## Design Principles

1. **Syntactic consistency with ggsql**: Use the same clause structure (`KEYWORD`, `SETTING key => value`, `FROM`, etc.)
2. **Declarative, not procedural**: The user declares *what* the table should look like; the engine determines *how*
3. **Composable clauses**: Clauses can appear in any order after `TABULATE` (same as after `VISUALISE`)
4. **Familiar SQL patterns**: `GROUP BY`, `ORDER BY`, `FILTER`, column selection work as expected
5. **gt API coverage**: Support the most impactful gt functions; defer niche features to later phases
6. **Sensible defaults**: A minimal `TABULATE` query should produce a reasonable table with auto-formatting
7. **Shallow hierarchy**: Only 6 top-level clauses; everything else is a subclause

---

## Top-Level Clause Architecture

There are exactly **6 top-level clauses**. All other operations are subclauses nested within these:

| Top-Level Clause | Purpose | Subclauses |
|---|---|---|
| `TABULATE` | Entry point, data source, column selection | `FROM` |
| `FORMAT` | Per-column formatting, structure (span/stub) | `SETTING`, `RENAMING` |
| `FACET` | Summary aggregation | `SUMMARIZE` |
| `SCALE` | Continuous aesthetic mapping (color, size) | `SETTING` |
| `HIGHLIGHT` | Conditional cell styling | `FILTER`, `SETTING` |
| `LABEL` | Text labels (title, subtitle, caption, column labels) | — |

---

## Syntax Structure (High-Level)

```sql
-- Standard SQL preamble (optional)
SELECT columns FROM table WHERE conditions

-- Entry point (required)
TABULATE <column>, ... FROM <data-source>

-- Formatting operations (can appear multiple times)
FORMAT <SPAN|STUB> <column>, ... AS span1
  SETTING <param> => <value>, ...
  RENAMING <value> => <string>, ...

-- Summary aggregation (can appear *only once*)
FACET <column>, ...
  SUMMARIZE <fn>, ..., <param> => <value>, ...

-- Styling based on numerical data (can appear multiple times)
SCALE background FROM (0, 10) TO ('white', 'red') VIA log10
  SETTING target => col1

-- Arbitrary styling (can appear multiple times)
HIGHLIGHT <column>, ...
  FILTER col > 30
  SETTING face => 'bold', background => 'red'

-- Labels and annotations (can appear *only once*)
LABEL 
  title => 'string', 
  subtitle => 'string', 
  caption => 'string',
  <column> => 'string', ...
```

---

## Clause Specifications

---

### 1. `TABULATE` — Entry Point (maps to `gt()`)

The `TABULATE` clause marks the transition from SQL to table declaration, analogous to `VISUALISE`. It defines which data enters the table and which columns are displayed.

**Syntax:**
```sql
TABULATE <column>, ... FROM <data-source>
```

**Behavior:**
- Columns listed after `TABULATE` define which columns are shown in the table body
- `*` selects all columns from the data source
- If a preceding `SELECT` provides data, `FROM` is optional
- Column order in `TABULATE` determines display order
- `AS` renames a column for display (maps to `cols_label()`)

**Examples:**
```sql
-- Minimal: show all columns from a query result
SELECT * FROM penguins
TABULATE *

-- Explicit columns, rename on the fly
SELECT * FROM sales
TABULATE date, region, revenue AS 'Revenue ($)' FROM sales

-- Using a CTE
WITH monthly AS (
  SELECT month, SUM(revenue) as total FROM sales GROUP BY month
)
TABULATE * FROM monthly
```

**gt mapping:** `gt(data, ...)` with column selection/ordering handled by the engine.

---

### 2. `FORMAT` — Per-Column Formatting and Structure (maps to `cols_*()`, `fmt_*()`, `sub_*()`, `tab_spanner()`, `text_transform()`)

The `FORMAT` clause is the workhorse for everything that operates on individual columns: formatting values, setting display properties, defining spanners, and designating the stub. It uses ggsql's existing `SETTING` and `RENAMING` subclauses (from `DRAW`/`SCALE`) to express properties and value formatting. The optional `SPAN` or `STUB` keyword before the column list defines structural roles.

**Syntax:**
```sql
FORMAT [SPAN|STUB] <column>, ... [AS <name>]
  SETTING <param> => <value>, ...
  RENAMING <value> => <string>, ...
```

- `SPAN` — groups columns under a spanner label (maps to `tab_spanner()`); `AS <name>` provides the spanner label
- `STUB` — designates column(s) as the table stub / row labels (maps to `gt(rowname_col=...)`)
- Without `SPAN`/`STUB` — standard formatting; `AS <name>` provides the stubhead label when used with a stub column

All subclauses within `FORMAT` are optional. Multiple `FORMAT` clauses can appear to address different columns.

---

#### 2.1 `FORMAT ... SETTING` — Display Properties

The `SETTING` subclause on `FORMAT` controls display properties:

- `width => 'string'` — column width, e.g., `'120px'`, `'20%'` (maps to `cols_width()`)
- `align => 'string'` — alignment: `'left'`, `'center'`, `'right'`, `'auto'` (maps to `cols_align()`)
- `hide => true/false` — hide column from display (maps to `cols_hide()`)
- `units => 'string'` — units notation (maps to `cols_units()`)

**Example:**
```sql
-- Designate stub column
FORMAT STUB model

-- Create a spanner grouping
FORMAT SPAN pop_2016, pop_2021 AS 'Population'
FORMAT SPAN density_2016, density_2021 AS 'Density'

-- Set display properties
FORMAT population
  SETTING width => '150px', align => 'right'
FORMAT area
  SETTING align => 'right'
FORMAT internal_id
  SETTING hide => true
```

**gt mapping:** `gt(rowname_col = "model")`, `tab_spanner(label = "Population", columns = c(pop_2016, pop_2021))`, `cols_width(population ~ px(150))`, `cols_align(align = "right", columns = population)`, `cols_hide(columns = internal_id)`

---

#### 2.2 `FORMAT ... RENAMING` — Value Formatting and Substitution

The `RENAMING` subclause uses the same syntax as ggsql’s `SCALE ... RENAMING`. The left-hand side identifies which values to transform; the right-hand side is a string interpolation pattern.

**Syntax:**
```sql
FORMAT <column>, ...
  RENAMING <value> => <string>, ...
```

**Left-hand side (LHS):**
- `*` — applies to all values (wildcard; used for formatting)
- `null` — applies to NA/missing values (maps to `sub_missing()`)
- `0` — applies to zero values (maps to `sub_zero()`)
- A literal value — direct value-to-label mapping (maps to `text_transform()`)

**Right-hand side (RHS) — string interpolation with formatters (ggsql-native):**
- `'{}'` — insert value as-is
- `'{:Title}'` — title-case
- `'{:UPPER}'` — upper-case
- `'{:lower}'` — lower-case
- `'{:num <printf>}'` — numeric formatting (printf syntax: `%0.2f`, `%+.1f`, `%,d`, `%e`, etc.)
- `'{:time <strftime>}'` — date/time formatting (strftime syntax: `%B %Y`, `%H:%M`, etc.)

Text surrounding `{}` is preserved as a prefix/suffix pattern.

**Examples:**
```sql
-- Format revenue as currency (comma-separated, 2 decimals, $ prefix)
FORMAT revenue
  RENAMING * => '${:num ,.2f}'

-- Format as percentage with 1 decimal
FORMAT satisfaction
  RENAMING * => '{:num .1f}%'

-- Format as integer with comma separators
FORMAT units
  RENAMING * => '{:num ,d}'

-- Numeric formatting with forced sign
FORMAT growth_rate
  RENAMING * => '{:num +.1f}%'

-- Scientific notation
FORMAT concentration
  RENAMING * => '{:num .2e}'

-- Date formatting
FORMAT report_date
  RENAMING * => '{:time %B %d, %Y}'

-- Replace missing values
FORMAT *
  RENAMING null => '—'

-- Replace zeros
FORMAT count
  RENAMING null => '—', 0 => '—'

-- Direct value mapping (like text_transform)
FORMAT status
  RENAMING 'active' => '✅ Active', 'inactive' => '❌ Inactive'

-- Title-case a column
FORMAT species
  RENAMING * => '{:Title}'
```

**gt mapping:**
- `* => '${:num ,.2f}'` → `fmt_currency()` / `fmt_number()`
- `* => '{:num .1f}%'` → `fmt_percent()`
- `* => '{:num ,d}'` → `fmt_integer()`
- `* => '{:num .2e}'` → `fmt_scientific()`
- `* => '{:time ...}'` → `fmt_date()`, `fmt_time()`, `fmt_datetime()`
- `null => 'text'` → `sub_missing(missing_text = "text")`
- `0 => 'text'` → `sub_zero()`
- Direct value mappings → `text_transform()`

---

#### 2.3 `FORMAT` — Consolidated Examples

Below are representative examples showing the range of `FORMAT` usage patterns.

**Example 1: Designate a stub and label it**
```sql
-- The 'country' column becomes the row label; AS provides stubhead text
FORMAT STUB country AS 'Country'
```

**Example 2: Group columns under a spanner**
```sql
-- Two spanners grouping related columns
FORMAT SPAN pop_2016, pop_2021 AS 'Population'
FORMAT SPAN density_2016, density_2021 AS 'Density'
```

**Example 3: Numeric formatting with display properties**
```sql
-- Format as currency, right-align, and set width
FORMAT revenue
  SETTING width => '140px', align => 'right'
  RENAMING * => '${:num ,.2f}'
```

**Example 4: Date formatting with column rename**
```sql
-- Render dates in long form; AS renames the column header
FORMAT order_date AS 'Order Date'
  RENAMING * => '{:time %B %d, %Y}'
```

**Example 5: Handle missing and zero values**
```sql
-- Replace nulls with em-dash globally, zeros with em-dash in 'count'
FORMAT *
  RENAMING null => '—'
FORMAT count
  RENAMING 0 => '—'
```

**Example 6: Direct value mapping (text transform)**
```sql
-- Map coded values to display labels
FORMAT status
  RENAMING 'A' => '✅ Active', 'I' => '❌ Inactive', 'P' => '⏸️ Paused'
```

**Example 7: Hide a column used only for SCALE targeting**
```sql
-- Keep 'score_raw' in the dataset for SCALE but hide from display
FORMAT score_raw
  SETTING hide => true
```

---

### 3. `FACET` — Summary Aggregation (maps to `summary_rows()`)

The `FACET` clause adds summary/aggregation rows to the table, paralleling ggsql's existing `FACET` clause in `VISUALISE`. In the table context, it targets specific columns for aggregation.

**Syntax:**
```sql
FACET <column>, ...
  SUMMARIZE <fn>, ..., <param> => <value>, ...
```

The columns listed after `FACET` specify which columns to aggregate.

---

#### 3.1 `FACET ... SUMMARIZE` — Summary Rows

**Syntax:**
```sql
FACET <column>, ...
  SUMMARIZE <fn>, ..., <param> => <value>, ...
```

**Aggregation functions** (same strings recognized by gt):
`'min'`, `'max'`, `'mean'`, `'median'`, `'sd'`, `'sum'`

**Parameters:**
- `groups => [grp1, grp2]` — which row groups (default: all)
- `side => 'top'/'bottom'` — placement relative to group (default: `'bottom'`)
- `label => 'string'` or `['label1', 'label2', ...]` — label(s) for the summary row(s)
- `missing_text => 'string'` — text for non-aggregable cells

**Examples:**
```sql
-- Minimal: single summary function with default placement (bottom)
TABULATE product, revenue, units FROM quarterly_sales
FACET revenue, units
  SUMMARIZE 'sum'

-- Multiple functions with custom labels
TABULATE date, open, high, low, close FROM sp500
FORMAT STUB date
FACET open, high, low, close
  SUMMARIZE 'min', 'max', 'mean',
            side => 'bottom', label => ['Min', 'Max', 'Avg']

-- Summary at the top, targeting specific row groups
TABULATE region, quarter, revenue FROM sales
FORMAT STUB region
FACET revenue
  SUMMARIZE 'mean',
            side => 'top', groups => ['North', 'South'],
            label => 'Avg', missing_text => '—'
```

**gt mapping:** `summary_rows(columns = c(revenue, units), fns = list("sum"))`, `summary_rows(fns = list("min", "max", list(label = "Avg", fn = "mean")))`, `summary_rows(groups = c("North", "South"), side = "top", ...)`

---

### 4. `SCALE` — Continuous Aesthetic Mapping (maps to `data_color()`, `tab_style()`)

The `SCALE` clause maps a continuous data range to a visual property range, applying it per-cell. This parallels ggsql's `SCALE` clause in `VISUALISE` which controls aesthetic mappings.

**Syntax:**
```sql
SCALE <aesthetic> FROM (<min>, <max>) TO (<val1>, <val2>) [VIA <transform>]
  SETTING target => <column>
```

**Components:**
- `<aesthetic>` — the visual property to scale:
  - `background` — cell background color
  - `foreground` — text color
  - `size` — font size
- `FROM (<min>, <max>)` — the data domain (numeric range)
- `TO (<val1>, <val2>)` — the output range (colors for `background`/`foreground`; sizes for `size`)
- `VIA <transform>` — optional transform: `log10`, `sqrt`, `reverse`, etc.
- `SETTING target => <column>` — which column to apply the scale to

**Examples:**
```sql
-- Color scale on background
TABULATE country, population, gdp_per_capita FROM world_data
SCALE background FROM (0, 100000) TO ('white', 'darkgreen') VIA log10
  SETTING target => gdp_per_capita
SCALE background FROM (0, 1500000000) TO ('#f7fbff', '#08306b')
  SETTING target => population

-- Font size scale
SCALE size FROM (0, 1000000) TO ('12px', '28px')
  SETTING target => population
```

**gt mapping:**
- `background`/`foreground` → `data_color(columns = ..., palette = c(...), domain = c(...))`
- `size` → `tab_style(style = cell_text(size = px(...)), locations = cells_body(...))` applied per-cell based on interpolated value

---

### 5. `HIGHLIGHT` — Conditional Cell Styling (maps to `tab_style()`)

The `HIGHLIGHT` clause applies style properties to cells matching a condition. This is the conditional formatting mechanism — the table analogue to conditional aesthetics in visualization.

**Syntax:**
```sql
HIGHLIGHT <column>, ...
  FILTER <condition>
  SETTING <style_param> => <value>, ...
```

**Style parameters (SETTING):**
- `face => 'bold'/'italic'/'bold.italic'` — font face
- `color => 'string'` — text color
- `background => 'string'` — cell background color
- `size => 'string'` — font size (e.g., `'14px'`, `'small'`)
- `weight => 'string'` — font weight
- `transform => 'uppercase'/'lowercase'` — text transform
- `decoration => 'underline'/'line-through'/'overline'` — text decoration

The `FILTER` subclause uses SQL-like condition expressions.

**Example:**
```sql
TABULATE name, score, grade FROM students
HIGHLIGHT score
  FILTER score < 60
  SETTING face => 'bold', color => 'red'
HIGHLIGHT grade
  FILTER grade = 'A'
  SETTING background => '#e6ffe6'
```

**gt mapping:** `tab_style(style = cell_text(weight = "bold", color = "red"), locations = cells_body(columns = score, rows = score < 60))`

---

### 6. `LABEL` — Text Labels (maps to `tab_header()`, `tab_caption()`, `cols_label()`)

The `LABEL` clause handles textual annotations: the table header (title/subtitle), caption, and column display labels. This parallels ggsql's existing `LABEL` clause in `VISUALISE` which sets titles and axis/legend labels.

**Syntax:**
```sql
LABEL
  title => 'string',
  subtitle => 'string',
  caption => 'string',
  <column> => 'string', ...
```

**Settings:**
- `title => 'string'` — main table title
- `subtitle => 'string'` — secondary descriptive title
- `caption => 'string'` — caption for cross-referencing
- `<column> => 'string'` — column display label (maps to `cols_label()`)

Reserved keys (`title`, `subtitle`, `caption`) are table-level; any other key is treated as a column name.

**Example:**
```sql
TABULATE * FROM sales
LABEL
  title => 'Quarterly Sales Report',
  subtitle => 'FY 2024 Q1-Q4',
  revenue => 'Revenue ($)',
  units => 'Units Sold',
  satisfaction => 'CSAT Score'
```

**gt mapping:** `tab_header(title = ..., subtitle = ...)`, `tab_caption(caption = ...)`, `cols_label(revenue = "Revenue ($)", units = "Units Sold", ...)`

---

## Complete Example

```sql
-- Data retrieval (standard SQL)
WITH quarterly AS (
  SELECT 
    region,
    quarter,
    SUM(revenue) as revenue,
    SUM(units_sold) as units,
    AVG(satisfaction) as satisfaction
  FROM sales_data
  WHERE year = 2024
  GROUP BY region, quarter
)

-- Entry point
TABULATE region, quarter, revenue, units, satisfaction FROM quarterly

-- Formatting operations
FORMAT STUB region
FORMAT SPAN revenue, units AS 'Financial'
FORMAT revenue
  RENAMING * => '${:num ,.1f}'
FORMAT units
  RENAMING * => '{:num ,d}'
FORMAT satisfaction
  RENAMING * => '{:num .1f}%'

-- Summaries
FACET revenue, units
  SUMMARIZE 'sum', 'mean',
            label => ['Total', 'Average'],
            side => 'bottom'

-- Conditional styling
SCALE background FROM (0, 1) TO ('white', 'green')
  SETTING target => satisfaction

HIGHLIGHT satisfaction
  FILTER satisfaction < 0.7
  SETTING face => 'bold', color => 'red'

-- Labels and annotations
LABEL
  title => '2024 Sales Performance',
  subtitle => 'By Region and Quarter',
  revenue => 'Revenue',
  units => 'Units Sold',
  satisfaction => 'CSAT Score'
```

---

## Mapping Summary: gt Functions → ggsql Clauses

| gt Function | ggsql Clause | Parent |
|---|---|---|
| `gt()` | `TABULATE ... FROM` | — |
| `gt(rowname_col)` | `FORMAT STUB <column>` | `FORMAT` |
| `tab_header()` | `LABEL title =>, subtitle =>` | `LABEL` |
| `tab_caption()` | `LABEL caption =>` | `LABEL` |
| `tab_spanner()` | `FORMAT SPAN <columns> AS 'label'` | `FORMAT` |
| `tab_stubhead()` | `FORMAT STUB <column> AS 'label'` | `FORMAT` |
| `fmt_number()` | `FORMAT ... RENAMING * => '{:num ...}'` | `FORMAT` |
| `fmt_integer()` | `FORMAT ... RENAMING * => '{:num ,d}'` | `FORMAT` |
| `fmt_percent()` | `FORMAT ... RENAMING * => '{:num .1f}%'` | `FORMAT` |
| `fmt_currency()` | `FORMAT ... RENAMING * => '${:num ,.2f}'` | `FORMAT` |
| `fmt_scientific()` | `FORMAT ... RENAMING * => '{:num .2e}'` | `FORMAT` |
| `fmt_date()` | `FORMAT ... RENAMING * => '{:time ...}'` | `FORMAT` |
| `fmt_time()` | `FORMAT ... RENAMING * => '{:time ...}'` | `FORMAT` |
| `fmt_datetime()` | `FORMAT ... RENAMING * => '{:time ...}'` | `FORMAT` |
| `sub_missing()` | `FORMAT ... RENAMING null => 'text'` | `FORMAT` |
| `sub_zero()` | `FORMAT ... RENAMING 0 => 'text'` | `FORMAT` |
| `text_transform()` | `FORMAT ... RENAMING <value> => <string>` | `FORMAT` |
| `cols_label()` | `LABEL <column> => 'string'` | `LABEL` |
| `cols_width()` | `FORMAT ... SETTING width => '...'` | `FORMAT` |
| `cols_align()` | `FORMAT ... SETTING align => '...'` | `FORMAT` |
| `cols_hide()` | `FORMAT ... SETTING hide => true` | `FORMAT` |
| `cols_units()` | `FORMAT ... SETTING units => '...'` | `FORMAT` |
| `summary_rows()` | `FACET ... SUMMARIZE` | `FACET` |
| `data_color()` | `SCALE background/foreground FROM ... TO ...` | `SCALE` |
| `tab_style()` (size) | `SCALE size FROM ... TO ...` | `SCALE` |
| `tab_style()` (conditional) | `HIGHLIGHT ... FILTER ... SETTING` | `HIGHLIGHT` |

---

## Phase Plan

### Phase 1: Core Table (MVP)
- `TABULATE` entry point with column selection and `FROM`
- `FORMAT` with `SETTING` (align, width), `STUB`, `SPAN`, and `RENAMING` (numeric, date/time formatting)
- `LABEL` with title/subtitle/caption and column labels
- HTML output via gt rendering engine

### Phase 2: Enrichment
- `FACET` with `SUMMARIZE`
- `FORMAT ... RENAMING` additions: date/time formatters, `null =>`, `0 =>`, direct value mapping
- `SCALE` for continuous color mapping
- `HIGHLIGHT` for conditional cell styling

### Phase 3: Extended Formatting & Output
- `FORMAT ... RENAMING` remaining formatters (scientific, bytes, duration)
- Multiple output formats (HTML, LaTeX, RTF)

### Phase 4: Advanced Features
- Nanoplots in cells (`cols_nanoplot` equivalent)
- `text_transform` equivalent
- Table groups (`gt_group` equivalent)

---

## Open Questions

1. **Keyword choice**: `TABULATE` vs `TABLE` vs `PRESENT`? (`TABULATE` avoids conflict with SQL `TABLE` keyword and is more distinctive.)
2. **Interplay with VISUALISE**: Can a query have both `VISUALISE` and `TABULATE`? Likely no — they are mutually exclusive output modes.
3. **Column aliasing**: Should `AS` in `TABULATE` set both the source mapping and the display label, or only the label? (Proposed: sets the display label, i.e., maps to `cols_label()`.)
4. **FORMAT wildcard**: `FORMAT *` with `RENAMING` — does it apply to all columns, or only type-compatible columns? (Proposed: only compatible columns, matching gt's behavior.)
5. **Output writer**: gt currently outputs to HTML, LaTeX, RTF, Word. Which writers should ggsql support first? (Proposed: HTML first, consistent with Vega-Lite being the first viz writer.)
6. **Locale inheritance**: Should a global locale setting propagate to all `FORMAT ... RENAMING` clauses (matching gt's global locale)? (Proposed: yes.)
7. **SCALE palette**: Should `SCALE` support named palettes (e.g., `TO 'viridis'`) in addition to explicit color pairs? (Proposed: yes, in a later phase.)
8. **HIGHLIGHT multiple columns**: Can `HIGHLIGHT` target different columns with the same filter? (Proposed: yes — list columns after `HIGHLIGHT`.)
9. **FORMAT SPAN nesting**: Can spanners be nested (multi-level)? (Proposed: yes — multiple `FORMAT SPAN` clauses with `level => <n>` parameter in `SETTING`.)

---

## Grammar Comparison: VISUALISE vs TABULATE

| Aspect | VISUALISE (plots) | TABULATE (tables) |
|---|---|---|
| Entry point | `VISUALISE` | `TABULATE` |
| Data binding | Aesthetic mappings (`col AS x`) | Column selection (`col`, `col AS 'Label'`) |
| Layers | `DRAW <type>` | `FORMAT` (per-column formatting) |
| Formatting | `SCALE` + `RENAMING` | `FORMAT ... RENAMING` (same syntax!) |
| Color scales | `SCALE` | `SCALE` (same keyword!) |
| Structure | `FACET` | `FACET` (SUMMARIZE) |
| Conditional | (via aesthetics) | `HIGHLIGHT` (conditional cell styling) |
| Annotations | `PLACE`, `LABEL` | `LABEL` (title, subtitle, caption) |
| Labels | `LABEL title =>` | `LABEL title =>` (same keyword!) |
| Coordinates | `PROJECT` | (N/A) |
| Settings | `SETTING key => val` | `FORMAT ... SETTING` (same keyword!) |

---

## References

- ggsql syntax: https://ggsql.org/syntax/
- ggsql source: https://github.com/posit-dev/ggsql
- gt reference: https://gt.rstudio.com/reference/index.html
- gt source: https://github.com/rstudio/gt

