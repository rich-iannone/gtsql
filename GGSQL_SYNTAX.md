# ggsql Syntax Specification

**Version**: Alpha (subject to change)  
**Basis**: Grammar of Graphics + SQL  
**Reference**: https://ggsql.org/

---

## 1. Overview

ggsql extends SQL with visualization clauses, enabling data retrieval and visualization specification in a single, composable query. The fundamental principle is the **Grammar of Graphics** — a systematic approach to building plots from independent, reusable components.

### 1.1 Query Architecture

ggsql queries are split at the `VISUALISE` boundary:

```
┌─────────────────────────────────────┐
│  SQL PORTION (before VISUALISE)     │
│  - Standard SQL: SELECT, WITH, etc. │
│  - Executed by database backend     │
└────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  VISUALISE PORTION (after VISUALISE)│
│  - Visualization specification      │
│  - Parsed into typed AST            │
│  - Rendered via pluggable writers   │
└─────────────────────────────────────┘
```

### 1.2 Key Design Principles

- **Declarative**: Specify what to visualize, not how
- **Composable**: Stack independent layers and scales
- **Data-native**: Computation stays in database
- **Grammar-based**: Consistent mental model for all plot types
- **AI-friendly**: Structured syntax suitable for code generation

---

## 2. Query Structure

### 2.1 Complete Query Syntax

```
[SQL_PORTION]
VISUALISE_STATEMENT [VISUALISE_STATEMENT]*
```

### 2.2 Full Example

```sql
-- SQL PORTION: Data retrieval
SELECT date, revenue, region, quantity
FROM sales
WHERE year = 2024 AND region IN ('East', 'West')

-- VISUALISE PORTION: Visualization specification
VISUALISE date AS x, revenue AS y, region AS color
DRAW point
  MAPPING quantity AS size
  SETTING alpha => 0.7
DRAW smooth
SCALE CONTINUOUS y SETTING limits => [0, 100000]
SCALE ORDINAL color SETTING palette => 'Set2'
FACET region
LABEL 
  title => 'Sales Trends 2024',
  x => 'Date',
  y => 'Revenue ($)',
  color => 'Region'
PROJECT TO cartesian
```

---

## 3. VISUALISE Statement

### 3.1 Syntax

```
VISUALISE [global_mapping] [FROM data_source]
  [viz_clause]*
```

**Aliases**: `VISUALIZE` (American spelling supported; case-insensitive)

### 3.2 Global Mapping

Defines mappings inherited by all layers. Any layer-specific mapping overrides global.

#### 3.2.1 Mapping Forms

Three forms of mapping, mixable in same clause:

**Explicit Mapping**
```
VISUALISE column_name AS aesthetic
```
- Maps named column to specific aesthetic
- Example: `date AS x, revenue AS y, region AS color`

**Implicit Mapping**
```
VISUALISE x, y
```
- Column name matches aesthetic name
- Expands to: `x AS x, y AS y`
- Must be a valid aesthetic name (see Section 5.1)

**Wildcard Mapping**
```
VISUALISE *
```
- Auto-maps all columns to aesthetics with matching names
- Useful for exploratory queries
- Can be combined: `VISUALISE *, revenue AS y` (y takes precedence)

#### 3.2.2 Mapping Values

Mappings can reference:

**Columns** (by name)
```
VISUALISE sales_date AS x
```
- References column from data source
- Names are case-sensitive per database backend

**Constants** (literal values)
```
VISUALISE 'red' AS fill, 0.5 AS opacity
```
- String literals: `'value'`
- Numbers: `42`, `3.14`
- Booleans: `true`, `false`
- Constants repeated for every record

**null** (suppress inheritance)
```
VISUALISE null AS color
```
- Prevents inheritance of color from global mapping
- Valid only in explicit form
- Used when layer should not inherit specific aesthetic

#### 3.2.3 Aesthetic vs. Property

**Aesthetic**: Visual characteristic rendered on plot
- Position: x, y, angle, radius, etc.
- Color: fill, stroke, color, opacity
- Size/shape: size, shape, linetype, linewidth
- Text: label, fontsize, fontweight

**Property**: Value used by statistical transformation
- Not rendered directly
- Example: `weight` in histogram for weighted bins

Mapped aesthetics go through scales; properties do not.

### 3.3 FROM Clause

Specifies global data source.

#### 3.3.1 Table/CTE Reference
```
VISUALISE FROM sales_data
VISUALISE FROM my_cte
```
- References existing table or CTE from pre-query
- Unquoted identifier

#### 3.3.2 File Path Reference
```
VISUALISE FROM 'data/sales.parquet'
VISUALISE FROM 'data/sales.csv'
```
- Single-quoted string
- Backend reads file directly
- Supported formats depend on backend

#### 3.3.3 Interaction with SELECT

**Rule**: Cannot use both `SELECT ... VISUALISE` and `VISUALISE FROM`

Valid (SELECT provides data):
```sql
SELECT * FROM sales
VISUALISE DRAW point
```

Valid (VISUALISE FROM provides data):
```sql
VISUALISE FROM sales
DRAW point
```

Invalid (ambiguous):
```sql
SELECT * FROM sales
VISUALISE FROM sales  -- ERROR
```

#### 3.3.4 FROM Shorthand Injection

Query without SELECT but with VISUALISE FROM:
```sql
VISUALISE FROM data
DRAW point MAPPING x AS x, y AS y
```

Automatically converted to:
```sql
SELECT * FROM data
VISUALISE FROM data
DRAW point MAPPING x AS x, y AS y
```

---

## 4. Visualization Clauses (viz_clause)

After VISUALISE, order of clauses is arbitrary (except VISUALISE first). Grouping for readability recommended.

---

## 5. DRAW Clause - Layer Definition

### 5.1 Syntax

```
DRAW geom_type
  [MAPPING mapping, ... [FROM source]]
  [REMAPPING mapping, ...]
  [SETTING param => value, ...]
  [FILTER condition]
  [PARTITION BY column, ...]
  [ORDER BY column, ...]
```

Only `geom_type` is required if global VISUALISE provides data and mapping.

### 5.2 Geom Types

Complete list of available geometric types:

#### 5.2.1 Point-based Geoms

**point**
- Scatterplot: individual points
- Key aesthetics: x, y, color, fill, size, shape, opacity
- Example: `DRAW point MAPPING x AS x, y AS y`

**text**
- Text labels at positions
- Key aesthetics: x, y, label, color, fontsize, fontweight, hjust, vjust, rotation
- Example: `DRAW text MAPPING x AS x, y AS y, label AS label`

#### 5.2.2 Line-based Geoms

**line**
- Connected line plot, sorted by x-axis
- Key aesthetics: x, y, color, linetype, linewidth, opacity
- Auto-sorts along x; use `path` for custom ordering
- Example: `DRAW line MAPPING x AS x, y AS y`

**path**
- Connected path in data order (no sorting)
- Key aesthetics: x, y, color, linetype, linewidth, opacity
- Respects data order; critical for complex paths
- Use with `ORDER BY` for reproducible ordering

**segment**
- Individual line segments between two points
- Key aesthetics: x, y, xend, yend, color, linetype, linewidth
- No automatic grouping
- Example: `DRAW segment MAPPING x AS x, y AS y, xend AS xend, yend AS yend`

**rule**
- Horizontal or vertical reference lines
- Key aesthetics: intercept (h or v), color, linetype, linewidth
- Fixed reference lines, not data-driven positions

#### 5.2.3 Area-based Geoms

**area**
- Area chart (filled polygon under curve)
- Key aesthetics: x, y, color, fill, opacity
- Useful for time series with fill coloring
- Example: `DRAW area MAPPING date AS x, revenue AS y`

**ribbon**
- Ribbon between upper and lower bounds
- Key aesthetics: x, ymin, ymax, color, fill, opacity
- Shows confidence intervals, ranges
- Example: `DRAW ribbon MAPPING x AS x, ymin AS ymin, ymax AS ymax`

#### 5.2.4 Bar/Tile Geoms

**bar**
- Bar chart with optional position adjustment
- Key aesthetics: x, y (optional if aggregate used), color, fill, opacity, width
- Can compute y from count if not mapped
- Default position: `stack`
- Example: `DRAW bar MAPPING category AS x, value AS y`

**tile**
- Rectangular tiles (heatmap-like)
- Key aesthetics: x, y, fill, color, width, height
- Useful for 2D binning, heatmaps
- Example: `DRAW tile MAPPING category AS x, type AS y, value AS fill`

**polygon**
- Arbitrary polygon shapes
- Key aesthetics: x, y, color, fill, opacity
- Requires correct point ordering
- Example: `DRAW polygon MAPPING longitude AS x, latitude AS y`

#### 5.2.5 Statistical Geoms

**histogram**
- Binned distribution
- Key aesthetics: x, y (computed count or density), color, fill, opacity
- Properties: `weight` (for weighted histogram)
- Computed columns: count, density (via REMAPPING)
- Default remapping: `count AS y`
- Settings: `binwidth`, `breaks` (for specific bin edges)
- Example: `DRAW histogram MAPPING value AS x SETTING binwidth => 5`

**density**
- Kernel density estimate (univariate)
- Key aesthetics: x, y (computed density), color, fill, opacity
- Computed columns: density, count
- Smooth distribution curve
- Example: `DRAW density MAPPING value AS x`

**violin**
- Rotated kernel density estimate
- Key aesthetics: x (discrete), y (continuous), color, fill, opacity
- Mirrored density plots per group
- Example: `DRAW violin MAPPING category AS x, value AS y`

**boxplot**
- Five-number summary (min, Q1, median, Q3, max)
- Key aesthetics: x (discrete), y (continuous), color, fill, opacity
- Computed columns: min, lower, middle, upper, max (via REMAPPING)
- Default remapping: uses 5-number summary
- Example: `DRAW boxplot MAPPING category AS x, value AS y`

**smooth**
- Trendline following data shape
- Key aesthetics: x, y, color, fill (for confidence band), linetype, linewidth, opacity
- Statistical smoother (LOESS by default)
- Can show confidence intervals
- Example: `DRAW smooth MAPPING x AS x, y AS y`

#### 5.2.6 Geometric Geoms

**range**
- Line segment between two values on axis
- Key aesthetics: x (discrete), ymin, ymax (or y_min, y_max), color, linewidth
- Optional hinges at endpoints
- Example: `DRAW range MAPPING category AS x, lower AS ymin, upper AS ymax`

**spatial**
- Simple features geometry
- Key aesthetics: geometry (WKT/GeoJSON), color, fill, opacity
- Requires geometry column in data
- Example: `DRAW spatial MAPPING geom_col AS geometry, region AS fill`

### 5.3 MAPPING Clause

Defines how data columns map to aesthetics in this layer.

#### 5.3.1 Syntax

```
MAPPING mapping, ... [FROM source]
```

Multiple mappings separated by commas.

#### 5.3.2 Mapping Forms (same as global)

- **Explicit**: `column AS aesthetic`
- **Implicit**: `column` (becomes `column AS column`)
- **Wildcard**: `*` (expands to all matching columns)
- **Constant**: `'value' AS aesthetic`

#### 5.3.3 FROM Sub-clause

Specifies data source for this layer only:

```
DRAW point 
  MAPPING x AS x, y AS y FROM other_table
```

- References different table/CTE
- Unquoted identifier (table name)
- Single-quoted string (file path)
- Overrides global data source for this layer

#### 5.3.4 Inheritance and Override

Global and layer mappings merge:
- Layer mapping takes precedence over global
- Layers inherit global mappings for unmapped aesthetics
- `null` in layer suppresses inheritance

Example:
```sql
VISUALISE x, y, color => 'blue'
DRAW point
DRAW line MAPPING color AS region  -- Overrides blue with region
```

### 5.4 REMAPPING Clause

Maps computed/statistical columns produced by geom's transformation.

#### 5.4.1 Syntax

```
REMAPPING mapping, ...
```

#### 5.4.2 Available Computed Columns (per geom)

**histogram**: count, density, width, x (bin edges)

**density**: density, count, x, y

**boxplot**: min, lower, middle, upper, max

**smooth**: y (predicted), se (std error), x

**violin**: density, count (per group/y-value)

#### 5.4.3 Usage

Change default remapping:
```sql
DRAW histogram MAPPING value AS x
REMAPPING density AS y  -- Use density instead of count
```

Use computed value for other aesthetic:
```sql
DRAW histogram MAPPING value AS x
REMAPPING count AS size  -- Bar size by count
```

#### 5.4.4 Default Remappings

Each geom has sensible defaults; remapping is optional but can override.

### 5.5 SETTING Clause

Configure parameters and literal aesthetic values.

#### 5.5.1 Syntax

```
SETTING param => value, ...
```

Arrow notation: `=>` (not `=`)

#### 5.5.2 Setting Types

**Aesthetic Settings** (literal values)
```
SETTING fill => 'red', opacity => 0.7
```
- Must be literal (not column reference)
- Values not scaled
- Override any mapping for that aesthetic

**Parameter Settings** (geom-specific)
```
DRAW histogram SETTING binwidth => 5
DRAW smooth SETTING method => 'loess'
```
- Geom-specific configuration
- Controls statistical transformation

**Position Settings**
```
SETTING position => 'dodge'
```
- Special: controls how overlapping objects reposition
- Values: `dodge`, `stack`, `jitter`, `identity`
- Default varies per geom (usually `identity` or `stack` for bar)

**Aggregate Settings** (collapse groups)
```
SETTING aggregate => 'mean'
```
- Collapses each group to single row
- Groups defined by discrete mappings + PARTITION BY
- See Section 5.8

#### 5.5.3 Aggregate Parameter Details

**Single Function** (applies to all numeric mappings):
```
SETTING aggregate => 'count'
```

**Targeted Function** (applies to specific aesthetic):
```
SETTING aggregate => 'y:mean'  -- Mean of y only
```

**Multiple Targeted Functions**:
```
SETTING aggregate => ('y:min', 'y:max')  -- Exploded aggregation
```

**Aggregate Functions**:

- **Basic**: count, sum, prod, min, max, range (max-min), mid ((min+max)/2)
- **Central Tendency**: mean, median
- **Special Means**: geomean, harmean, rms
- **Spread**: sdev, var, iqr, se
- **Percentiles**: p05, p10, p25, p50, p75, p90, p95
- **Sequence**: first, last, diff (last-first)

**Band Functions** (offset ± multiplier×expansion):
```
SETTING aggregate => 'mean-1.96sdev'  -- 95% CI
```

Allowed offsets: mean, median, geomean, harmean, rms, sum, prod, min, max, mid, p05–p95

Allowed expansions: sdev, se, var, iqr, range

**Exploded Aggregation** (creates multiple rows per group):
```
SETTING aggregate => ('y:min', 'y:max')
```
- Produces one row per function
- Creates synthetic `aggregate` column tagging each row
- Use REMAPPING to drive aesthetics by function
- Example: `REMAPPING aggregate AS stroke` colors lines by min/max

#### 5.5.4 Common Settings by Geom

| Geom | Common Settings |
|------|-----------------|
| histogram | binwidth, breaks, position |
| density | kernel, bw_adjust |
| smooth | method, se (show SE band) |
| bar | position, width |
| point | size, alpha |

### 5.6 FILTER Clause

Row-level filtering for this layer only.

#### 5.6.1 Syntax

```
FILTER condition
```

#### 5.6.2 Conditions

Any WHERE-clause expression supported by backend:
```
FILTER region = 'East'
FILTER value > 0 AND category IN ('A', 'B')
FILTER date >= '2024-01-01'
```

Evaluated by database backend; only matching rows appear in layer.

### 5.7 PARTITION BY Clause

Groups records for within-group connections.

#### 5.7.1 Syntax

```
PARTITION BY column, ...
```

#### 5.7.2 Purpose

- **Line layer**: Determines which records connect within each partition
- **Path layer**: Defines grouping for path ordering
- **Other geoms**: May define group-wise calculations

#### 5.7.3 Example

```sql
SELECT date, revenue, region FROM sales
VISUALISE date AS x, revenue AS y
DRAW line PARTITION BY region  -- One line per region
```

Records within same region connect; records across regions do not.

### 5.8 ORDER BY Clause

Enforces record order for this layer.

#### 5.8.1 Syntax

```
ORDER BY column [ASC | DESC], ...
```

#### 5.8.2 Purpose

- **Path layer**: Critical — plots records in specified order
- **Line layer**: May affect overplotting z-order (top/bottom)
- **Histogram**: No effect (binning order-independent)

#### 5.8.2 Example

```sql
SELECT * FROM events
VISUALISE event_time AS x, event_type AS y
DRAW path ORDER BY timestamp  -- Connect in time order
```

---

## 6. PLACE Clause - Annotations

### 6.1 Syntax

```
PLACE geom_type [SETTING param => value, ...]
```

### 6.2 Purpose

Add annotation layers with literal values only (no data mapping).

### 6.3 Example

```sql
VISUALISE x, y
DRAW point MAPPING x AS x, y AS y
PLACE rule SETTING intercept => 100, color => 'red'  -- Reference line at y=100
PLACE text SETTING x => 0.5, y => 0.95, label => 'Important Note'
```

### 6.4 Limitations

- No MAPPING clause (literal values in SETTING only)
- No data from upstream SELECT
- Useful for thresholds, annotations, titles overlaid on plot

---

## 7. SCALE Clause - Aesthetic Scaling

### 7.1 Syntax

```
SCALE scale_type aesthetic_name [SETTING param => value, ...]
```

### 7.2 Purpose

Controls how data values map to visual properties (colors, sizes, positions, etc.).

### 7.3 Scale Types

#### 7.3.1 Continuous Scale

Maps continuous input → continuous output

```
SCALE CONTINUOUS x
SCALE CONTINUOUS y SETTING limits => [0, 100]
```

**Settings**:
- `limits` - Min/max output range: [0, 100]
- `expand` - Expansion factors: [0, 0] for no padding
- `breaks` - Specific positions for axis ticks
- `labels` - Custom tick labels
- `transform` - Transformation: 'log10', 'sqrt', 'identity'

#### 7.3.2 Discrete Scale

Maps discrete input → discrete output (unordered)

```
SCALE DISCRETE color
SCALE DISCRETE color SETTING palette => 'Set2'
```

**Settings**:
- `palette` - Color palette name or manual colors: ['red', 'blue']
- `limits` - Specify order/subset of categories
- `na_value` - Color for missing values

#### 7.3.3 Binned Scale

Maps continuous input → ordered discrete output via binning

```
SCALE BINNED x SETTING breaks => 'weeks', n_breaks => 10
```

**Settings**:
- `breaks` - Bin width or special: 'weeks', 'months', 'years'
- `n_breaks` - Number of bins to create
- `limits` - Explicit bin edges: [0, 10, 20, 30]

#### 7.3.4 Ordinal Scale

Maps discrete input → ordered discrete output (enforces ordering)

```
SCALE ORDINAL region SETTING limits => ['East', 'West', 'South']
```

**Settings**:
- `limits` - Explicit ordering of categories
- `palette` - Color palette (for color/fill aesthetics)

#### 7.3.5 Identity Scale

Passes values through unchanged (no transformation)

```
SCALE IDENTITY color  -- Use color column values directly as CSS colors
```

**Use case**: Data already in color hex codes, radii in points, etc.

### 7.4 Aesthetics

Subclasses define which scales apply:

#### 7.4.1 Position Aesthetics

Cartesian: x, y, xmin, xmax, ymin, ymax, xend, yend

Polar: angle, radius, anglemin, anglemax, radiusmin, radiusmax, angleend, radiusend

#### 7.4.2 Color Aesthetics

- `color` / `colour` - Stroke color
- `fill` - Fill color
- `stroke` - Alias for color
- `opacity` - Transparency [0, 1]

#### 7.4.3 Size/Shape Aesthetics

- `size` - Point radius (in points)
- `shape` - Point marker shape (0-24 in ggplot2 convention)
- `linetype` - Stroke pattern (solid, dashed, dotted, etc.)
- `linewidth` - Stroke width (in points)
- `width` - Bar/tile width
- `height` - Bar/tile height

#### 7.4.4 Text Aesthetics

- `label` - Text to display
- `fontsize` - Font size (points)
- `fontweight` - Weight: 'normal', 'bold'
- `typeface` - Font family
- `hjust` - Horizontal alignment [0, 1] (0=left, 0.5=center, 1=right)
- `vjust` - Vertical alignment [0, 1] (0=bottom, 0.5=center, 1=top)
- `rotation` - Angle in degrees

#### 7.4.5 Specialty Aesthetics

- `slope` - Slope aesthetic (for specific geoms)
- `geometry` - WKT/GeoJSON geometry column
- `offset`, `density`, `count`, `intensity` - Computed aesthetics

#### 7.4.6 Facet Aesthetics

- `panel` - Facet panel assignment (1D)
- `row` - Row in facet grid
- `column` - Column in facet grid

### 7.5 Scale Precedence

Multiple scales for same aesthetic merge with precedence:
1. **Explicit SCALE clause** (highest)
2. **Default scale** from geom
3. **Identity** (lowest)

---

## 8. FACET Clause - Small Multiples

### 8.1 Syntax

```
FACET aesthetic_name
FACET row_aesthetic ~ column_aesthetic
```

### 8.2 Purpose

Split data into small multiple panels, one per unique value.

### 8.3 Single Aesthetic

```sql
VISUALISE x, y
DRAW point
FACET region
```

Creates one panel per unique region value.

### 8.4 Two-Dimensional Faceting

```sql
VISUALISE x, y
DRAW point
FACET region ~ year
```

Creates grid: rows = region values, columns = year values

### 8.5 Scale Sharing

**Shared scales** (default): All panels use same x/y axis limits

**Free scales**: Panels have independent scales
```
FACET region SETTING free_x => true, free_y => true
```

### 8.6 Labeling

```
FACET region SETTING strip_position => 'top'  -- top, bottom, left, right
```

---

## 9. PROJECT Clause - Coordinate Systems

### 9.1 Syntax

```
PROJECT [aesthetic_names] TO coordinate_system
```

### 9.2 Purpose

Defines how abstract position aesthetics (x, y) or (angle, radius) map to screen.

### 9.3 Cartesian Coordinate System

```
PROJECT x, y TO cartesian
PROJECT TO cartesian  -- Inferred from aesthetics
```

**Characteristics**:
- Classic perpendicular x-y axes
- Default system
- Used for scatterplots, lines, bars, histograms, etc.

**Aesthetics**: x, y (with optional min/max variants)

### 9.4 Polar Coordinate System

```
PROJECT angle, radius TO polar
PROJECT TO polar
```

**Characteristics**:
- Circular coordinate system
- angle = angular position (degrees, radians)
- radius = distance from center
- Useful for: pie charts, circular heatmaps, compass plots

**Aesthetics**: angle, radius (with optional min/max variants)

### 9.5 Coordinate Inference

If PROJECT not specified, inferred from DRAW mappings:
- If x, y mapped → cartesian
- If angle, radius mapped → polar

---

## 10. LABEL Clause - Titles and Labels

### 10.1 Syntax

```
LABEL identifier => 'value', ...
```

### 10.2 Supported Labels

```
LABEL
  title => 'Main Title',
  subtitle => 'Subtitle',
  x => 'X Axis Label',
  y => 'Y Axis Label',
  color => 'Color Legend Title',
  fill => 'Fill Legend Title',
  size => 'Size Legend Title',
  shape => 'Shape Legend Title',
  caption => 'Caption/Source'
```

### 10.3 Example

```sql
VISUALISE date AS x, revenue AS y, region AS color
DRAW line
LABEL
  title => 'Revenue by Region',
  subtitle => 'Quarterly trends',
  x => 'Date',
  y => 'Revenue (USD)',
  color => 'Region',
  caption => 'Data: Company Sales Database'
```

---

## 11. Advanced Features

### 11.1 Multiple Layers

Stack independent geoms on same plot:

```sql
VISUALISE x, y
DRAW point
DRAW line
DRAW smooth
```

Renders in order: points first (bottom), then line, then smooth (top).

### 11.2 Layer-specific Data Sources

Each DRAW can use different data:

```sql
VISUALISE x, y FROM sales
DRAW point
  MAPPING x AS x, y AS y
DRAW line
  MAPPING x AS x, y AS y FROM forecast
```

Points from sales, line from forecast table.

### 11.3 Global vs. Layer Mappings

Global inherited by layers, layer overrides:

```sql
VISUALISE x, y, color => 'blue'
DRAW point  -- Uses blue
DRAW line MAPPING color => 'red'  -- Overrides to red
```

### 11.4 Wildcard Expansion

```sql
VISUALISE *  -- Maps x, y, color if columns exist
DRAW point MAPPING z AS size  -- Adds size mapping
```

Wildcard auto-maps matching columns; explicit overrides.

### 11.5 Aggregation Strategies

**Reduction aggregation** (single value per group):
```
SETTING aggregate => 'mean'  -- One row per group
```

**Exploded aggregation** (multiple rows per function):
```
SETTING aggregate => ('y:min', 'y:max', 'y:mean')
REMAPPING aggregate AS stroke  -- Color by function
```

### 11.6 Multiple Visualizations

One query, multiple visualization specs:

```sql
SELECT x, y FROM data
VISUALISE x, y DRAW point
VISUALISE x DRAW histogram
```

Produces two plots.

---

## 12. Aesthetic Context and Coordinate Transformation

### 12.1 Aesthetic Context Initialization

ggsql initializes aesthetic context based on:
- Active coordinate system (cartesian vs. polar)
- Facet dimensions (panel, row, column)
- Layer-specific mappings

### 12.2 Internal Aesthetic Transformation

User-facing aesthetic names transformed to internal representation:

**Cartesian**:
- x → pos1 (primary axis)
- y → pos2 (secondary axis)

**Polar**:
- angle → pos1
- radius → pos2

This enables generic handling throughout rendering pipeline.

### 12.3 Position Aesthetics Variants

For ranges and error bars:
- xmin, xmax, ymin, ymax
- anglemin, anglemax, radiusmin, radiusmax
- xend, yend, angleend, radiusend

Scales applied symmetrically to range bounds.

---

## 13. Data Flow and Execution

### 13.1 Query Execution Pipeline

```
1. Parse query at VISUALISE boundary
2. Execute SQL portion against backend
3. Extract data for visualization layers
4. Apply statistical transformations (bin, density, etc.)
5. Apply scales (map data to aesthetics)
6. Apply coordinate transformation
7. Apply facet splits
8. Render via writer (ggplot2, Vega-Lite, PNG)
```

### 13.2 Backend Integration

- SQL executes entirely in database
- Only visualization data extracted to client
- Scales to billions of rows (server-side aggregation)
- Supports CTEs, subqueries, window functions

### 13.3 Statistical Computation Location

- **Database-side**: Binning (histogram), aggregation (sum, mean)
- **Client-side**: Kernel density, smoothing (if too expensive server-side)

---

## 14. Syntax Flexibility

### 14.1 Case Insensitivity

All keywords case-insensitive:
```
VISUALISE = visualise = VisualiSe
DRAW = draw = Draw
```

### 14.2 American/British Spelling

- VISUALISE (preferred) or VISUALIZE
- colour or color (both valid in aesthetics)

### 14.3 Identifier Quoting

Identifiers can be quoted if containing special characters:
```
VISUALISE "Order Date" AS x, "Revenue ($)" AS y
```

### 14.4 Line Breaking

Clauses can span multiple lines for readability:
```sql
VISUALISE
  date AS x,
  revenue AS y,
  region AS color
DRAW point
  MAPPING size AS size
  SETTING alpha => 0.7
```

---

## 15. Error Handling

### 15.1 Parse-Time Errors

- Invalid geom type
- Unknown aesthetic name
- Syntax errors in SCALE, FACET clauses

### 15.2 Validation Errors

- VISUALISE FROM + SELECT (ambiguous data source)
- Missing required aesthetic for geom
- Mismatch between aesthetic and scale type

### 15.3 Execution Errors

- Column not found in data
- SQL syntax error
- Incompatible data types

---

## 16. Best Practices

### 16.1 Query Organization

1. Write SQL first (SELECT, filters, joins, aggregations)
2. Add VISUALISE with global mapping
3. Add DRAW with geom type
4. Add SCALE, FACET, labels as needed
5. Group clauses logically (all SCALEs together, etc.)

### 16.2 Layer Composition

- Start with primary layer (point, line, bar)
- Add layers for context (smooth, reference lines)
- Use transparent colors for overlapping geoms

### 16.3 Data Efficiency

- Push computation to database (GROUP BY, aggregation)
- Use FILTER in DRAW to limit layer data
- Avoid extracting unnecessary columns

### 16.4 Aesthetic Mapping

- Reserve color/size for important dimensions
- Use implicit mappings when column = aesthetic name
- Use explicit mappings for clarity in complex plots

---

## 17. Example Gallery

### 17.1 Scatter Plot with Trend

```sql
SELECT x, y FROM dataset
VISUALISE x, y
DRAW point MAPPING category AS color
DRAW smooth SETTING se => false
LABEL title => 'Relationship with Trend'
```

### 17.2 Stacked Bar Chart

```sql
SELECT category, type, value FROM data
VISUALISE category AS x, value AS y
DRAW bar
  MAPPING type AS fill
  SETTING position => 'stack'
SCALE ORDINAL fill SETTING palette => 'viridis'
```

### 17.3 Faceted Histogram

```sql
SELECT value, group FROM data
VISUALISE value AS x
DRAW histogram
  SETTING binwidth => 0.5
FACET group
LABEL title => 'Distribution by Group'
```

### 17.4 Multi-layer Aggregated Plot

```sql
SELECT date, revenue, forecast_revenue, region FROM sales
WHERE year = 2024
VISUALISE date AS x, region AS color
DRAW point MAPPING revenue AS y
DRAW line MAPPING forecast_revenue AS y
LABEL 
  title => 'Actual vs. Forecast',
  y => 'Revenue ($)'
```

### 17.5 Boxplot with Points

```sql
SELECT category, value FROM data
VISUALISE category AS x, value AS y
DRAW boxplot
DRAW point SETTING position => 'jitter', alpha => 0.3
SCALE CONTINUOUS y SETTING limits => [0, 100]
```

### 17.6 Violin Plot

```sql
SELECT group, measurement FROM experiment
VISUALISE group AS x, measurement AS y
DRAW violin
DRAW point SETTING position => 'jitter', size => 1
```

---

## 18. Grammar of Graphics Mapping

ggsql maps Grammar of Graphics concepts to SQL:

| GoG Concept | ggsql Implementation |
|-------------|---------------------|
| **Data** | SELECT ... or VISUALISE FROM |
| **Mapping** | MAPPING / VISUALISE aesthetics |
| **Geom** | DRAW geom_type |
| **Stat** | DRAW geom_type (implicit) |
| **Scale** | SCALE clause |
| **Coord** | PROJECT clause |
| **Facet** | FACET clause |
| **Theme** | Implicit (writer-specific) |

---

## 19. Implementation Notes

### 19.1 Parser Architecture

ggsql uses **tree-sitter** for parsing:

- **grammar.js**: Defines syntax rules
- **builder.rs**: Walks parse tree, builds typed AST
- **source_tree.rs**: Parse-once wrapper with query API

### 19.2 AST Transformation

During parsing:
1. Wildcard mappings expanded to explicit
2. Implicit mappings converted to explicit
3. Aesthetic names transformed to internal (x→pos1, y→pos2)
4. Coordinate system inferred if not explicit

### 19.3 Query Splitting

SQL portion ends at VISUALISE keyword. ggsql recognizes:
- Complete SELECT statements
- CTEs (WITH clauses)
- Set operations (UNION, INTERSECT, EXCEPT)
- Subqueries and joins

---

## 20. References

- **Official Docs**: https://ggsql.org/
- **Grammar of Graphics**: Wilkinson, L. (2005). "The Grammar of Graphics"
- **ggplot2 Reference**: Wickham & Grolemund (R implementation)
- **Vega-Lite**: Interactive specification language (output format)

