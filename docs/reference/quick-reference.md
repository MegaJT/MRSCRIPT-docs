# Part 9 · Quick reference

## 27. All keywords at a glance

### Preprocessor (pre-parse)

```text
SET   DEFINE … END DEFINE   CALL          (text substitution + macro inlining)
```

### Authoring layer (.mrst → .mrs, transpiled before everything below)

```text
#for … #endfor   #if / #elif / #else … #endif   #set   #include   (## escapes #)
#// comment                                                          (comment — stripped at transpile time, whole-line or inline)
Loop sources   inline list · #set list · lo..hi range · SHEET("f","Tab") · CSV("f")
Interpolation  {name}  {row.column}  {loop.index|count|first|last}
```

### Top-level statements

```text
SOURCE  %RESPID  APPEND  JOIN  CONFIG  FORMAT  VARIABLE
DERIVE  STACK  EDIT  RECODE  KEEP ROWS  DROP ROWS  COMPUTE  RIM
BANNER (named)  TABLE  SCOPE  EXPORT DATA
ADDTAB  BANKED_TABLE                              (cross-table ops — reference stored tables by NAME)
EXPECT                                            (routing assertions / data QC)
```

### Block closers

```text
END CONFIG   END FORMAT   END VARIABLE   END DERIVE   END STACK
END TABLE    END BANNER   END DEFINE   END RIM   ENDNET   ENDHEADING   ENDSCOPE
```

### Clauses by block

| Context | Clauses |
|---------|---------|
| `SOURCE` | `AS`  `DATA`  `CODEBOOK`  `SCHEMA`  (+ optional name) |
| `JOIN` | `WITH`  `ON`  `%RESPID`  `TYPE` (left \| inner) |
| `CONFIG` | `OUTPUT`  `SIG_CONFIDENCE`  `SIG_CORRECTION`  `SIG_COMPARE`  `MISSING_TREATMENT`  `DEFAULT_STATS`  `SUPPRESS_STACKED_SIG`  `SUPPRESS_GRID_SIG` |
| `FORMAT` / `TABLE` shared | `STATS`  `BANNER`  `WEIGHT`  `BASE_LABEL`  `FOOTER`  `THOUSANDS_SEPARATOR`  `MIN_BASE`  `CONFIDENTIAL`  `BLANK_SUPPRESS`  `SUPPRESS_EMPTY`  `AUTONUMBER`  `RANKING`  `SHOW_TOTAL`  `MAX_COL_WIDTH`  `DECIMALS`  `PCT_DECIMALS`  `COUNT_DECIMALS`  `MEAN_DECIMALS`  `PCT_SIGN` |
| `TABLE`-only | `STUBS`  `DISTRIBUTION`  `ADD`  `SECTION LABEL`  `LEVEL`  `BASE`  `FILTER`  `SHEET`  `STATS_ONLY`  `NAME` |
| `GRID` table | `TYPE GRID`  `COLUMN`  `LABEL`  `FILTER`  `ANSWERED_BASE` |
| `VARIABLE` | `LABEL`  `TYPE`  `VALUE`  `MISSING`  `SCORE` |
| `DERIVE` | `LABEL`  `TYPE`  `SCORE`  `STUB`  `NET`  `ENDNET`  `HEADING`  `ENDHEADING` |
| `STUB` display props | `DISPLAY` (pct_only \| count_only \| row_pct)  `SUPPRESS`  `CUMULATIVE`  `DECIMALS`  `KEEP_IF_ZERO` |
| `STACK` (single-axis) | `POSITIONS`  `MAP`  `FROM` |
| `STACK` (multi-axis) | `AXIS`  `AT`  `DROP EMPTY`  `NULL`  (also `MAP`  `FROM`) |
| `EDIT` / `RECODE` | `SET`  `WHERE`  `INTO`  `ELSE`  (`->`) |
| `COMPUTE` functions | `round`  `abs`  `sqrt`  `min`  `max` |
| `EXPORT DATA` | `AS`  `SOURCE_ONLY`  `CODEBOOK` |
| `RIM` | `DIMENSION`  `TARGETS`  `MAX_ITERATIONS`  `CONVERGENCE`  `WEIGHT_CAP`  `BASE_WEIGHT` |
| `ADDTAB` | `TITLE`  `NAME`  (source table names as quoted strings) |
| `BANKED_TABLE` | `TITLE`  `NAME`  (stub-source name + source table names as quoted strings) |
| `EXPECT` | `WHERE`  `MISSING`  `ANSWERED`  (condition or sugar form) |

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
| **Cross-table operations** | `DERIVED_TABLE` / `MANIP` (cell arithmetic across stored results, e.g. `T3 = T1 − T2`); `CEPX` re-emit (route a stored table to a second output destination). `ADDTAB` (wave merge) and `BANKED_TABLE` (side-by-side) are already implemented — see [§20](tables.md#cross-table). |
| **Extra significance** | chi-square, t-test (pairs), Kolmogorov-Smirnov, Mann-Whitney; pluggable test strategies / FDR / `TAILS`. |
| **More COMPUTE** | `round(x, n)` decimals; `log` / `exp` / `floor` / `ceil` / `clip`; conditional `COMPUTE … WHERE`. |
| **Output formats** | Word (`.docx`), PDF, PowerPoint; Triple-S / XtabML export. |
| **Multi-file (more)** | Chained append-then-join; per-wave id tagging; `RENAME` / suffix to resolve `JOIN` column clashes; right / outer joins. |
