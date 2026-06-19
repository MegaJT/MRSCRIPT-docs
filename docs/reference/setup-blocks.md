# Part 3 · Setup blocks

## 8. CONFIG {#config}

```mrs
CONFIG
  clause*
END CONFIG
```

Script-wide execution settings. All clauses are optional.

| Clause | Values | Meaning |
|--------|--------|---------|
| `OUTPUT` | `text` \| `excel` | Default output format. `text` (default) = plain-text tables; `excel` = workbook. |
| `SIG_CONFIDENCE` | `<pct> [, <pct>]` | Confidence level(s) in **percent** (not an alpha). One value (default `95`) = a single level. Two values (e.g. `99, 95`) = **dual** testing: letters significant at the higher level are UPPERCASE, those significant only at the lower level are lowercase. Each value must be `50 < pct < 100`. |
| `SIG_CORRECTION` | `none` \| `bonferroni` | Multiple-comparison correction across the column pairs in a comparison group. Default `none` (each pair tested at the raw level, matching Quantum/MRDCL/SPSS). `bonferroni` divides alpha by the number of testable pairs. |
| `SIG_COMPARE` | `segment` \| `all` \| `total` | Which columns are compared. Default `segment`. |
| `MISSING_TREATMENT` | `exclude` \| `show` | Whether missing-coded respondents are excluded. Default `exclude`. |
| `SUPPRESS_STACKED_SIG` | `true` \| `false` | When `true`, drops significance flags from stacked (`LEVEL`) tables, where repeated exposures are not independent. Default `false` (a non-fatal advisory is emitted when sig is kept on a `LEVEL` table). |
| `SUPPRESS_GRID_SIG` | `true` \| `false` | When `true`, drops significance flags from `GRID` tables, whose columns are the same respondents (paired). Default `false` (sig runs; a non-fatal advisory notes the paired approximation). |
| `DEFAULT_STATS` | `stat [, stat …]` | Fallback stats if no `FORMAT STATS` exists. Prefer `FORMAT STATS` ([§9](#format)). |

**`SIG_COMPARE` values**

- `segment` — within each innermost banner group only.
- `all` — round-robin across all non-Total columns (cross-segment).
- `total` — each non-Total column against the Total column (Total is lettered and
  participates; needs a Total column shown).

```mrs
CONFIG
  OUTPUT          text
  SIG_CONFIDENCE  99, 95
  SIG_COMPARE     total
  SUPPRESS_STACKED_SIG  true
END CONFIG
```

---

## 9. FORMAT (script-wide display defaults) {#format}

```mrs
FORMAT
  clause*
END FORMAT
```

Default display settings inherited by every `TABLE`. Any clause may be overridden at
the table level ([§17](tables.md)). One `FORMAT` block per script.

| Clause | Meaning |
|--------|---------|
| `STATS stat [, …]` | Default stats for all tables ([§22](reference-details.md#stats-values)). |
| `BANNER var [, …]` | Default inline banner (column variables). |
| `BANNER Name` | Default named banner (defined by [§18](tables.md#named-banner)). |
| `WEIGHT $weight_var` | Default weight variable. |
| `BASE_LABEL 'text'` | Rename the base/respondents row. |
| `FOOTER 'text'` | Append a line below every table. |
| `THOUSANDS_SEPARATOR true\|false` | Comma-group large counts (e.g. `1,234`). |
| `MIN_BASE n [MASK 'text']` | Mask whole banner columns with base `< n` (mask: `<<`). |
| `CONFIDENTIAL n [MASK 'text']` | Mask individual cells with count `< n` (mask: `**`). |
| `BLANK_SUPPRESS row\|col\|both` | Hide all-zero rows / zero-base columns within a table. |
| `SUPPRESS_EMPTY true\|false` | Drop the **whole** table when it has no data — every column base is 0 in every section. Surviving tables renumber contiguously. Default `false`. |
| `AUTONUMBER true\|false` | Number leaf stub rows (`1. 2. 3. …`). |
| `RANKING ascending\|descending` | Sort stub rows by the **Total** column value — shorthand for `SORT col_pct ASC\|DESC ON TOTAL`. |
| `SORT [col_pct\|n\|row_pct] [ASC\|DESC] [ON $var=code\|TOTAL]` | Sort stub rows by any banner column and stat. Full syntax in [§17 Sorting rows](tables.md#sorting). |
| `SHOW_TOTAL true\|false\|'Label'` | Show/hide/label the leading Total column ([§17](tables.md#total-column)). |
| `PCT_SIGN true\|false` | Append the `%` sign to percentages (default `true`). `false` renders the bare number (`56` not `56%`). Text renderer only. |
| `MAX_COL_WIDTH n` | Text-renderer header-wrap cap per data column. |
| `DECIMALS n` | **Umbrella** decimal places — the fallback for percentages, counts **and** means when the specific directive below is not set. |
| `PCT_DECIMALS n` | Decimal places for percentages only. |
| `COUNT_DECIMALS n` | Decimal places for weighted/fractional counts and bases only. |
| `MEAN_DECIMALS n` | Decimal places for Mean / Std Dev / score-summary rows only. |

!!! info "Decimals resolution order"

    For each aspect: per-row `DECIMALS` ([§11b](data-preparation.md#stub-display-properties))
    wins, then the specific directive, then umbrella `DECIMALS`, then the default
    (pct **0**, count **0**, mean **2**). Integer (unweighted) counts always print
    whole.

```mrs
FORMAT
  STATS               n, col_pct
  BANNER              @ageband, $gender
  BASE_LABEL          'Respondents'
  THOUSANDS_SEPARATOR true
  MIN_BASE            20 MASK 'n<20'
  FOOTER              'Source: Natural Toothpaste Survey, Ipsos India 2025'
END FORMAT
```

---

## 10. VARIABLE (source variable metadata overrides) {#variable}

```mrs
VARIABLE $source_var
  clause*
END VARIABLE
```

Overrides metadata for a loaded source variable. Useful especially for CSV columns
that carry no labels. All clauses are optional. A sidecar `CODEBOOK`
([§5](data-input.md#source)) is applied first; inline `VARIABLE` wins on any conflict.

| Clause | Meaning |
|--------|---------|
| `LABEL 'text'` | Display label shown in table headers. |
| `TYPE type_value` | Measurement type (see below). |
| `VALUE code 'label'` | Add / override a value label for one code. Declaring value labels on a numeric column makes it categorical (distributes by stubs instead of the mean/std path). |
| `MISSING code [, …]` | Declare codes as missing (excluded from base; stubs removed). SPSS missing metadata is **not** auto-applied. |
| `SCORE code = value` | Numeric score for a code, used by mean / std summaries. |

**Type values**

| Type | Meaning |
|------|---------|
| `single_punch` | One response chosen from a list. |
| `multi_punch` | Multiple responses (stored; renderer awareness limited). |
| `numeric` | Continuous numeric (mean / std path). |
| `open_end` | Free text. |
| `multi_binary` | Binary 0/1 flag columns (auto-detected). |

```mrs
// Labelling a coded CSV column and declaring missing
VARIABLE $q1
  LABEL 'Overall satisfaction'
  VALUE 1 'Strongly disagree'
  VALUE 5 'Strongly agree'
  MISSING 99
END VARIABLE
```

```mrs
// Reverse scoring for a mean
VARIABLE $S12b
  LABEL 'Brushing frequency'
  SCORE 1 = 5   SCORE 2 = 4   SCORE 3 = 3   SCORE 4 = 2   SCORE 5 = 1
END VARIABLE
```
