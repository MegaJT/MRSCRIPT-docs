# Part 5 · Tables

## 17. TABLE {#table}

```mrs
TABLE ['title']
  clause*
END TABLE
```

One output table. `FORMAT` defaults ([§9](setup-blocks.md#format)) are inherited; any
clause here overrides that setting for this table only. The title is optional
(defaults to the first stub variable's label).

### Row axis — STUBS / DISTRIBUTION / ADD {#row-axis}

At least one row-axis clause is required.

#### STUBS

```mrs
STUBS item [, item ...]
```

Row variables, by their metadata stubs. Multiple items create stacked sections. An
item may nest with `BY` (inner → outer):

```mrs
STUBS $q1                     // one section
STUBS @ageband, $gender       // two sections
STUBS @region BY $gender      // region crossed within gender
```

A `HEADING` group may divide stub items:

```mrs
STUBS HEADING 'Demographics' @ageband, $gender ENDHEADING
```

#### DISTRIBUTION

```mrs
DISTRIBUTION var [, var ...]
```

Row axis built from the **unique codes present in the data** (sorted), rather than the
variable's metadata stubs. Nulls and declared `MISSING` codes are excluded. Each row
**leads with the raw code**: `"1: Male"` when the code has a value label, bare `"99"`
when it doesn't — keeping the data-driven audit view visually distinct from the
label-only `STUBS` table. Reveals codes that `STUBS` would silently drop, and works on
numeric and labelled variables (a derived multi-response var is exploded first).
Requesting a summary stat appends those rows computed over the raw values.

```mrs
DISTRIBUTION $age STATS n, col_pct, mean, std_dev
```

`STUBS` and `DISTRIBUTION` may both appear and compose in source order.

#### ADD — pool variables into one section

```mrs
ADD [DISTRIBUTION] var [, var ...] [SECTION LABEL 'text']
```

**Pool** several same-frame variables into **one section** — counts and base **add
up** (response-level): for each row and banner cell, `count = Σ count(var = code)` and
`base = Σ base(var)`. A respondent valid on two of the variables is counted twice
(the classic ADDTAB pooling — distinct from a `NET`, which is a de-duplicated union
with base N, and from a `GRID`, which lays variables side by side as columns).
`SECTION LABEL` names the pooled block (default `"@v1 + @v2"`).

- **Plain `ADD`** pools by stub code: rows are the variables' shared stub frame; NET
  rows and the score frame come from the variables (so the Mean is pooled too).
  Variables must be categorical (numeric / open_end rejected).

  ```mrs
  ADD @q1_brand, @q2_brand SECTION LABEL 'Any mention (Q1 + Q2)'
  ```

- **`ADD DISTRIBUTION`** pools by the data values present — rows are the sorted union
  of distinct codes across the columns (nulls and declared `MISSING` excluded). This
  accepts **numeric** columns too, so it sums a roster of numeric columns (e.g.
  children's ages held one per column) into one frequency table, with a pooled mean.
  Declare the "empty slot" sentinel as `MISSING` so unused columns don't inflate the
  base.

  ```mrs
  VARIABLE $child1_age MISSING 0 END VARIABLE   // ... repeat per column
  ADD DISTRIBUTION $child1_age, $child2_age, $child3_age
    SECTION LABEL 'All children'
  ```

### Column axis — BANNER

```mrs
BANNER item [, item ...]   // inline banner; each item is a variable or a BY-chain
BANNER Name                // reference a named banner (§18)
```

```mrs
BANNER @ageband                // single segment
BANNER @ageband, $gender       // two side-by-side segments
```

#### Nested (crossed) columns — `BANNER $a BY $b [BY $c …]`

Crosses variables into a column hierarchy with stacked header bands, to any depth. As
with `STUBS`, the variable **after** `BY` is the outer/spanning group; the first
(innermost) variable's categories are the leaf columns.

```mrs
BANNER $city BY $gender        // Gender spans; City nested beneath
```

```text
              Male              Female
       Mumbai   Delhi      Mumbai   Delhi
  ...     .       .          .       .
```

Each leaf cell counts the **AND** of its column conditions (e.g. gender=Male AND
city=Mumbai). Significance (`STATS sig`) is tested within the innermost group only.
Crossing operands must be plain categorical variables (no numeric, nets, or
headings).

Omitting `BANNER` gives a Total-only single column. A leading Total column is shown
by default when a banner is present — control it with `SHOW_TOTAL`. (Stacked header
bands render in text/CSV output; Excel is not supported.)

### GRID tables — `TYPE GRID` {#grid-tables}

A `GRID` table renders several variables' **own** single-variable distributions as
side-by-side **columns** — the transpose of a normal table (the rows are different
variables that share one code frame). Use it for brand-imagery grids (statements ×
brands) and rating batteries (scale × statements). It replaces `STUBS` / `BANNER` and
uses `COLUMN`:

```mrs
TABLE 'Brand imagery' TYPE GRID
  COLUMN @brand1 LABEL 'Brand A' [FILTER $aware1 = 1]
  COLUMN @brand2 LABEL 'Brand B'
  COLUMN @brand3 LABEL 'Brand C'
  [ANSWERED_BASE false]
  STATS col_pct, n, mean, sig
END TABLE
```

The rows are the columns' shared layout (leaves + NETs + headings), taken from the
first column and aligned by stub code; a column missing a code shows a blank cell.
**Everything** in a column comes from its variable — its NET rows and its Mean —
nothing is re-declared on the table.

| Clause | Meaning |
|--------|---------|
| `COLUMN var LABEL 'text' [FILTER cond]` | One column = one variable; `LABEL` is the header. An optional `FILTER` scopes that column's base. Variables must be categorical / list. (A `FILTER` right after `LABEL` binds to that column; a table-wide `FILTER` must precede the `COLUMN` list.) |
| `ANSWERED_BASE true \| false` | `true` (default) — a filter-less column bases on respondents who answered. `false` — bases on all in-scope respondents (the "Total" base). |

Significance compares the columns (the same respondents → paired), so the test is
approximate and a non-fatal advisory is emitted — drop the flags with
`CONFIG SUPPRESS_GRID_SIG true`. A `GRID` has no Total column and no `BANNER`.

### STATS

```mrs
STATS [+] stat [, stat ...]
```

Which statistic rows appear ([§22](reference-details.md#stats-values)). Three modes
([§26](reference-details.md#stats-modes)): inherit (no clause), override (`STATS …`),
merge (`STATS + …`).

### Other table clauses

| Clause | Meaning |
|--------|---------|
| `STATS_ONLY` | Render only the summary rows (mean, std dev, …); suppress the distribution rows (still computed internally). |
| `FILTER condition` | Row filter for this table only; AND-ed with any enclosing `SCOPE`. |
| `WEIGHT $weight_var` | Apply probability weights to this table (adds weighted_n / effective base). |
| `LEVEL @stack_name` | Tabulate against a stacked frame ([§12](data-preparation.md#stack)). Optional — inferred from the variable names when omitted; explicit `LEVEL` takes precedence. |
| `BASE respondents` | Unique-respondent base. On a `LEVEL` table it corrects for rotation; on a non-stacked table it de-duplicates by `%RESPID` (which must be declared). |
| `SHEET 'tab name'` | Target worksheet name in Excel output (no effect on text output). |
| `NAME 'handle'` | Register this table in the table store under `handle` instead of its title, so a cross-table op ([§20](#cross-table)) can address it. Names must be unique within the script. |
| `SHOW_TOTAL true\|false\|'Label'` | Show (default), hide, or relabel the leading Total column. The Total column is excluded from significance lettering. |

Other `FORMAT` directives may also appear at table level (overriding the `FORMAT`
default here): `BASE_LABEL`, `FOOTER`, `THOUSANDS_SEPARATOR`, `MIN_BASE`,
`CONFIDENTIAL`, `BLANK_SUPPRESS`, `SUPPRESS_EMPTY`, `AUTONUMBER`, `RANKING`,
`MAX_COL_WIDTH`, `DECIMALS`, `PCT_DECIMALS`, `COUNT_DECIMALS`, `MEAN_DECIMALS`,
`PCT_SIGN`.

```mrs
TABLE 'T7. Oral-care concerns by age'
  STUBS  @concerns
  BANNER @ageband
  STATS  n, col_pct, sig
  RANKING        descending
  AUTONUMBER     true
  BLANK_SUPPRESS row
  SHOW_TOTAL     'All adults'
  BASE_LABEL     'Total respondents'
END TABLE
```

### Total column {#total-column}

When a `BANNER` is present, a leading **Total** column is prepended by default.
`SHOW_TOTAL` accepts three forms (table-level or in the global `FORMAT` block; table
overrides global):

- `SHOW_TOTAL false` — hide it.
- `SHOW_TOTAL true` — show it (default), labelled "Total".
- `SHOW_TOTAL 'All Adults'` — show it with a custom label (implies show).

A banner-less table's single column is already the Total, so `SHOW_TOTAL` is a no-op
there. The Total column is **excluded from significance lettering/testing** — banner
columns are lettered A, B, C… starting at the first real column.

---

## 18. BANNER (named, reusable banners) {#named-banner}

```mrs
BANNER Name var [, var ...] END BANNER
```

Defines a reusable banner referenced by name in a `TABLE` or `FORMAT` clause
(`BANNER Name`). Keeps a common column structure in one place.

```mrs
BANNER Demographics @ageband, $gender, $region END BANNER

FORMAT  BANNER Demographics  END FORMAT     // default for all tables

TABLE 'Awareness' STUBS @aware BANNER Demographics END TABLE
```

---

## 19. SCOPE (pre-filtering tables) {#scope}

```mrs
SCOPE WHERE condition [LABEL 'text']
  statement*
ENDSCOPE
```

Pre-filters every `TABLE` inside the `SCOPE`. The `SCOPE` condition is AND-ed with
each table's own `FILTER`. `SCOPE`s may nest; conditions AND at each level.

```mrs
SCOPE WHERE $gender = 1 LABEL 'Males'
  TABLE 'Q1 — Males'    STUBS $q1   END TABLE
  TABLE 'Brands — Males' STUBS @brands END TABLE
ENDSCOPE

SCOPE WHERE $AGE_CAT IN (2, 3) LABEL 'Young adults'
  SCOPE WHERE $AGE_CAT = 2 LABEL '20–29 only'
    TABLE 'Frequency — youngest' STUBS $S12b END TABLE
  ENDSCOPE
ENDSCOPE
```

!!! note "Notes"

    - Data-preparation statements (`DERIVE`/`EDIT`/`RECODE`/…) inside a `SCOPE` still
      apply to the **whole dataset** — the `SCOPE` only filters which respondents
      count in its tables.
    - `SCOPE LABEL` is shown as the `Base: …` line in each enclosed table's text
      output (nested labels join outer→inner with `; `). Tables with no enclosing
      labelled `SCOPE` default to `Base: All respondents`.

---

## 20. Cross-table operations {#cross-table}

Cross-table operations post-process tables that have **already been tabulated** —
they read finished `TABLE` results out of the table store and never re-read the
data. Each references its sources by **name**: the `NAME 'handle'` clause on a
`TABLE` ([§17](#table)), or the table's title when no `NAME` is given. A source
must be declared **earlier** in the script.

Both produce ordinary tables that render exactly like any other (text or Excel),
appended after the regular tables and numbered sequentially. Significance flags
are **not** carried over — they were computed over the source bases; re-add
`STATS sig` only if you mean to test the combined result.

### ADDTAB — wave merge {#addtab}

```mrs
ADDTAB 'T1', 'T2' [, …] [TITLE 'text'] [NAME 'handle']
```

Adds the cell counts of several **same-layout** tables (same stubs × same banner)
into one combined result — the classic roll-up of monthly waves into a quarter.
Counts and weighted counts add; percentages are **recomputed** from the combined
counts and bases (not averaged); means/standard deviations re-derive correctly
from the summed sufficient statistics. Median / mode / quartile rows blank out on
merge (a pooled distribution can't be rebuilt from summary statistics) — re-run
those on the combined data if you need them.

- `TITLE` — title of the merged table (default: the source titles joined with `+`).
- `NAME` — store handle for the merged table (default: its title).
- All sources must share the same sections, stub order, and banner columns —
  otherwise the merge raises an error.

```mrs
TABLE 'Satisfaction — Jan' NAME 'Jan' STUBS $q1 BANNER $region STATS col_pct, n END TABLE
TABLE 'Satisfaction — Feb' NAME 'Feb' STUBS $q1 BANNER $region STATS col_pct, n END TABLE

ADDTAB 'Jan', 'Feb' TITLE 'Satisfaction — Q1 roll-up' NAME 'Q1'
```

### BANKED_TABLE — side-by-side tracker {#banked-table}

```mrs
BANKED_TABLE 'Stub' 'T1' [, 'T2' …] [TITLE 'text'] [NAME 'handle']
```

Places several tables (same stubs, different banners) next to each other as one
wide table — the "tracker" view of one question across periods or geographies.
The **first** name is the *stub source*: it supplies the row layout only. The
remaining names are the tables whose banner columns are concatenated, left to
right. To include the stub source's own columns, **list it again** among the
sources.

Duplicate column labels (e.g. "Total" in every wave) are disambiguated with the
source table's title, e.g. `Total (Satisfaction — Jan)`. All sources must share
the stub source's row layout.

```mrs
TABLE 'Satisfaction — Jan' NAME 'Jan' STUBS $q1 BANNER $region STATS col_pct, n END TABLE
TABLE 'Satisfaction — Feb' NAME 'Feb' STUBS $q1 BANNER $region STATS col_pct, n END TABLE

BANKED_TABLE 'Jan' 'Jan', 'Feb' TITLE 'Satisfaction tracker'
```

!!! tip "Simulating waves in one dataset"

    The engine tabulates one dataset per run, so the wave roll-up pattern works
    on data that already carries a wave indicator: `APPEND` the wave files (or add
    a `wave` column), then give each wave its own `TABLE … FILTER $wave = n NAME …`
    before the `ADDTAB`/`BANKED_TABLE`.
