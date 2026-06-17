# Part 9 · Quick reference

## 27. All keywords at a glance

### Preprocessor (pre-parse)

```text
SET   DEFINE … END DEFINE   CALL          (text substitution + macro inlining)
```

### Authoring layer (.mrst → .mrs, transpiled before everything below)

```text
#for … #endfor   #if / #elif / #else … #endif   #set   #include   (## escapes #)
Loop sources   inline list · #set list · lo..hi range · SHEET("f","Tab") · CSV("f")
Interpolation  {name}  {row.column}  {loop.index|count|first|last}
```

### Top-level statements

```text
SOURCE  %RESPID  APPEND  JOIN  CONFIG  FORMAT  VARIABLE
DERIVE  STACK  EDIT  RECODE  KEEP ROWS  DROP ROWS  COMPUTE
BANNER (named)  TABLE  SCOPE  EXPORT DATA
```

### Block closers

```text
END CONFIG   END FORMAT   END VARIABLE   END DERIVE   END STACK
END TABLE    END BANNER   END DEFINE   ENDNET   ENDHEADING   ENDSCOPE
```

### Clauses by block

| Context | Clauses |
|---------|---------|
| `SOURCE` | `AS`  `DATA`  `CODEBOOK`  `SCHEMA`  (+ optional name) |
| `JOIN` | `WITH`  `ON`  `%RESPID`  `TYPE` (left \| inner) |
| `CONFIG` | `OUTPUT`  `SIG_CONFIDENCE`  `SIG_CORRECTION`  `SIG_COMPARE`  `MISSING_TREATMENT`  `DEFAULT_STATS`  `SUPPRESS_STACKED_SIG`  `SUPPRESS_GRID_SIG` |
| `FORMAT` / `TABLE` shared | `STATS`  `BANNER`  `WEIGHT`  `BASE_LABEL`  `FOOTER`  `THOUSANDS_SEPARATOR`  `MIN_BASE`  `CONFIDENTIAL`  `BLANK_SUPPRESS`  `SUPPRESS_EMPTY`  `AUTONUMBER`  `RANKING`  `SHOW_TOTAL`  `MAX_COL_WIDTH`  `DECIMALS`  `PCT_DECIMALS`  `COUNT_DECIMALS`  `MEAN_DECIMALS`  `PCT_SIGN` |
| `TABLE`-only | `STUBS`  `DISTRIBUTION`  `ADD`  `SECTION LABEL`  `LEVEL`  `BASE`  `FILTER`  `SHEET`  `STATS_ONLY` |
| `GRID` table | `TYPE GRID`  `COLUMN`  `LABEL`  `FILTER`  `ANSWERED_BASE` |
| `VARIABLE` | `LABEL`  `TYPE`  `VALUE`  `MISSING`  `SCORE` |
| `DERIVE` | `LABEL`  `TYPE`  `SCORE`  `STUB`  `NET`  `ENDNET`  `HEADING`  `ENDHEADING` |
| `STUB` display props | `DISPLAY` (pct_only \| count_only \| row_pct)  `SUPPRESS`  `CUMULATIVE`  `DECIMALS`  `KEEP_IF_ZERO` |
| `STACK` (single-axis) | `POSITIONS`  `MAP`  `FROM` |
| `STACK` (multi-axis) | `AXIS`  `AT`  `DROP EMPTY`  `NULL`  (also `MAP`  `FROM`) |
| `EDIT` / `RECODE` | `SET`  `WHERE`  `INTO`  `ELSE`  (`->`) |
| `COMPUTE` functions | `round`  `abs`  `sqrt`  `min`  `max` |
| `EXPORT DATA` | `AS`  `SOURCE_ONLY`  `CODEBOOK` |

### Values

| Group | Values |
|-------|--------|
| Condition keywords | `AND`  `OR`  `NOT`  `IN`  `BETWEEN`  `ALL` |
| STATS values | `n`  `col_pct`  `row_pct`  `weighted_n`  `sig`  `mean`  `std_dev`  `std_error`  `median`  `quartile(N)`  `mode`  `sum`  `sum_of_squares`  `error_variance` |
| Type values | `single_punch`  `multi_punch`  `numeric`  `open_end`  `multi_binary` |
| Variable prefixes | `$` (source)  `@` (derived)  `%RESPID` (respondent key) |

---

## 28. Common script patterns

The full, copy-pasteable pattern library lives on the **[How-to recipes](../guide/patterns.md)**
page — profile tables, significance crosstabs, T2B, computed indices, APPEND/JOIN,
concept tests, diaries, GRID batteries, ADD pooling, and reproducible-dataset export.

---

## Planned features (not yet implemented) {#planned-features}

Scripts using these will error until the feature ships.

| Area | Planned |
|------|---------|
| **Input readers** | `AS triple_s \| ascii \| mdd` are registered but staged (raise "not yet supported"). Only `spss` and `csv` are fully implemented today. |
| **What-if scenarios** | `SCENARIO` blocks + per-table `COMPARE` producing baseline vs scenario Δ / %Δ tables. |
| **Cross-table operations** | `DERIVED_TABLE` (cell arithmetic across stored results), `BANKED_TABLE` (side-by-side concat), `ADDTAB` / `CEPX` (multi-wave accumulation across stored results — in-table response pooling is already available via the `ADD` clause). |
| **Extra significance** | chi-square, t-test (pairs), Kolmogorov-Smirnov, Mann-Whitney; pluggable test strategies / FDR / `TAILS`. |
| **More COMPUTE** | `round(x, n)` decimals; `log` / `exp` / `floor` / `ceil` / `clip`; conditional `COMPUTE … WHERE`. |
| **Output formats** | Word (`.docx`), PDF, PowerPoint; Triple-S / XtabML export. |
| **Multi-file (more)** | Chained append-then-join; per-wave id tagging; `RENAME` / suffix to resolve `JOIN` column clashes; right / outer joins. |
