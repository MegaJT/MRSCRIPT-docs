# Part 8 · Reference details

## 22. STATS values — complete list {#stats-values}

### Distribution stats (per stub row)

| Stat | Meaning |
|------|---------|
| `n` | Unweighted count in the cell. |
| `col_pct` | Column %: cell n as % of the column base. |
| `row_pct` | Row %: cell n as % of the row total. |
| `weighted_n` | Sum of weights (requires `WEIGHT`; adds weighted `col_pct`). |
| `sig` | Significance flags (see below). |

**`sig`** — distribution rows use a pooled two-proportion **Z-test** (overlap-adjusted
when banner columns share respondents); a Mean row uses an independent-samples
(**Welch**) t-test. Columns are lettered in the header as `Label (B)`; a cell flagged
`B` is significantly **higher** than column B at the `SIG_CONFIDENCE` level
(UPPERCASE = high CL, lowercase = low CL in dual mode). Which columns are compared
follows `SIG_COMPARE` — `segment` (within each banner group), `all` (every column),
`total` (each vs the Total), or `prior` (**wave-on-wave**: each column vs the one
immediately to its left, within each segment, treated as independent samples —
approximate for same-respondent panel data; see [setup-blocks §8](setup-blocks.md#config)).
Correction follows `SIG_CORRECTION` (default `none`). A banner
column below the table's `MIN_BASE` is excluded from testing. The letters render on
their own row beneath each stub's count and percentage rows; multiple letters are
separated (`B/C`).

### Summary stats (appended below the distribution; numeric/scored variables)

| Stat | Meaning |
|------|---------|
| `mean` | Arithmetic mean (of scored values, or raw for numeric). |
| `std_dev` | Standard deviation. |
| `std_error` | Standard error of the mean. |
| `median` | Median (50th percentile). |
| `quartile(N)` | Nth percentile, N an integer 0–100 (e.g. `quartile(25)`). |
| `mode` | Most frequent value. |
| `sum` | Σx. |
| `sum_of_squares` | Σx². |
| `error_variance` | Σx² / n (or weighted equivalent). |

!!! note "Notes"

    - Summary stats on a **categorical** variable use its `SCORE` clauses when
      declared; otherwise each **code is used as its own score** (code = score),
      matching `DISTRIBUTION` and numeric variables. A multi-response (list) variable
      needs an explicit `SCORE` — a code-mean over a list of codes is not meaningful.
    - Numeric variables use their raw values.
    - All summary stats honour `WEIGHT` (weighted sums / Kish effective base).

---

## 23. Condition syntax {#conditions}

Conditions appear after `WHERE` in `STUB`, `EDIT`, `KEEP`/`DROP ROWS`, `FILTER`, and
`SCOPE`.

| Form | Syntax | Examples |
|------|--------|----------|
| Simple comparison | `$var OP value` — `OP ∈ = != > >= < <=` | `$AGE_CAT = 2`, `$q1 >= 4`, `$score != 99` |
| Set membership | `$var IN (v1, v2, …)` | `$q1 IN (4, 5)`, `$region IN (1, 2, 3)` |
| Range (inclusive) | `$var BETWEEN lo AND hi` | `$age BETWEEN 25 AND 34` |
| Logical | `cond AND cond`, `cond OR cond`, `NOT cond`, `(cond)` | `$gender = 1 AND $age >= 30` |
| Catch-all | `ALL` — matches every row | `STUB 9 'Everyone' WHERE ALL` |

```mrs
$q1 IN (4, 5) OR $q2 IN (4, 5)
($gender = 1 OR $gender = 2) AND $q1 >= 3
```

Conditions may reference source (`$`) and earlier-defined `@variables`. (Arithmetic
expressions like `$a + $b` are only available inside `COMPUTE`
([§16](data-preparation.md#compute)) — not in conditions, which are comparison-based.)

---

## 24. Base computation {#base}

The base is the per-column denominator for column percentages:

```text
base_n[col] = count( banner_col_mask AND valid_mask )
```

`valid_mask` = is-not-null **AND** not in a declared `MISSING` code. System missing
(`.`) is always excluded.

**Weighted base** — when `WEIGHT` is active, three base rows appear:

| Row | Meaning |
|-----|---------|
| `n` | Unweighted count |
| `weighted_n` | Sum of weights |
| `effective_n` | Kish effective base: (Σw)² / Σ(w²) |

**Respondent base** — `BASE respondents` counts unique respondents (by `%RESPID`). On
a stacked (`LEVEL`) table it corrects for rotation; on a non-stacked table it
de-duplicates rows.

---

## 25. Missing values {#missing-values}

MRScript applies a strict **"the script is the authority"** principle — nothing in the
data file is automatically treated as missing.

| Kind | Behaviour |
|------|-----------|
| **System missing** (`.`) | Stored as null; always excluded; never a table row. |
| **User-defined missing** | Real numeric codes (99 = Don't know, 98 = Refused, −1 = N/A) preserved as values. Exclude them by declaring them: `VARIABLE $q1 MISSING 99 END VARIABLE`. Effect: the code's stub is removed and those respondents drop out of the base. |
| **SPSS `.sav` missing ranges** | Intentionally **not** read — declare missing codes in the script (or a `CODEBOOK`), so scripts behave identically across CSV / Triple-S / ASCII / MDD, which have no "missing range" concept. |

---

## 26. STATS modes — inherit / override / merge {#stats-modes}

| Mode | Trigger | Behaviour |
|------|---------|-----------|
| **Inherit** | no `STATS` clause | Table uses `FORMAT STATS` exactly (or `CONFIG DEFAULT_STATS`, else `col_pct, n`). |
| **Override** | `STATS …` (without `+`) | Table uses exactly the listed stats; `FORMAT STATS` is replaced. |
| **Merge** | `STATS + …` | `FORMAT STATS` first, then the listed extras appended (no duplicates). |

```mrs
// Override
STATS n, col_pct, sig

// Merge
FORMAT STATS n, col_pct END FORMAT
... STATS + mean, std_dev, median    // → n, col_pct, mean, std_dev, median
```

`STATS_ONLY` may combine with any mode: the stats are computed, but only the summary
rows render.
