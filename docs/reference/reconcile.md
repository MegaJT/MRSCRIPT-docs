# 12 · Tab-Plan Reconciliation

**`mrscript reconcile`** validates an MRScript against a client tab-plan spreadsheet
and answers:

1. **Coverage** — which planned tables are present / missing / extra?
2. **Conformance** — do the matched tables have the right banner, stats, and base?

---

## Basic usage

```
mrscript reconcile script.mrs plan.xlsx
mrscript reconcile script.mrs plan.csv --fail-on error
```

The script is executed in full (data loaded, AUTOTAB expanded, cross-table ops run)
before matching — so all generated tables compose for free.

---

## Full option reference

```
mrscript reconcile script.mrs plan.xlsx [OPTIONS]

Options:
  --data FILE      Data file — overrides SOURCE in the script
  --sheet SHEET    Worksheet name to read from (xlsx only; defaults to first sheet)
  --json FILE      Write machine-readable JSON report to FILE
  --csv FILE       Write findings CSV to FILE
  --strict         Treat extra (produced-but-unplanned) tables as warnings
  --fail-on LEVEL  Exit 2 when findings at/above LEVEL exist (error|warn)
```

---

## Tab-plan format

The tab-plan is a **CSV or Excel (.xlsx) spreadsheet** — the reader is forgiving:

- Rows above the header (client logos, titles) are silently skipped.
- Blank rows and single-cell section dividers are skipped.
- Every column is optional; a plan with only a `Title` column works.

### Recognised column headers

| Column | Accepted header names |
|--------|-----------------------|
| `number` | Table, No, #, Ref, Table No., Table #, … |
| `title` | Title, Description, Heading, Question, Table Name, … |
| `name` | Handle, Key, Tag, Table Key, Table Tag |
| `stubs` | Stub, Stubs, Variable, Variables, Row, Rows, … |
| `banner` | Banner, Column, Crossbreak, Breaks, Col, … |
| `base` | Base, Universe, Filter, Sample, Base / Universe, … |
| `stats` | Stats, Statistics, Measures, Cells, Cell Content, … |

Stat values in the `stats` column are mapped to canonical tokens regardless of
phrasing (`%`, `col %`, `pct`, `percentage` → `col_pct`; `average`, `avg` → `mean`;
`count`, `freq` → `n`; etc.).

---

## Matching cascade

Tables are matched **greedily in plan order**, one-to-one:

1. **NAME** — plan `name` column matches the table's `NAME "handle"` (case-insensitive).
2. **Normalised title** — lowercased, punctuation-stripped title match.
3. **Stub variable** — a plan stub var matches a table stub variable (with or without `$`).
4. **Fuzzy title** — `difflib.SequenceMatcher` ratio ≥ 0.85 (emits an info finding to flag for manual review).

---

## Conformance checks

For each matched pair (only when the plan supplies the column):

| Check | Level | Condition |
|-------|-------|-----------|
| `stats_mismatch` | warn | Plan stats not present in the table |
| `banner_mismatch` | warn | Plan specifies a banner but table is Total-only (or vice versa with zero token overlap) |
| `base_mismatch` | warn | Plan base and table base descriptions clearly differ (synonym-aware) |
| `title_mismatch` | info | Matched by NAME or stub but titles differ |
| `probable_match` | info | Fuzzy match; emitted to prompt manual verification |

---

## Sample output

```
Reconciling: survey.mrs  ↔  plan.xlsx (12 planned table(s))

── COVERAGE ───────────────────────────────────────
  10 of 12 planned table(s) matched
  error  Table 3: Q3 Awareness    Planned table "Table 3: Q3 Awareness" not produced
  error  Table 7: Brand Image     Planned table "Table 7: Brand Image" not produced

── CONFORMANCE ────────────────────────────────────
  warn  [stats]   Table 1: Gender ↔ Table 1: Gender: plan wants mean but table has col_pct, n
  warn  [banner]  Table 5: Usage ↔ Table 5: Usage frequency: plan banner 'Gender, Age' …

10 matched · 2 missing · 3 extra  ·  2 error(s), 1 warning(s)
```

---

## Worked example

**`plan.csv`**:
```
Table,Title,Stubs,Banner,Base,Stats
1,Gender,$gender,@demographics,All respondents,%, n
2,Age,$age_group,@demographics,All respondents,%, n, mean
3,Q1 Satisfaction,$q1,@demographics,All respondents,%, n, mean
```

**Script snippet** (`survey.mrs`):
```
FORMAT
    BANNER @demographics
    STATS col_pct, n
END FORMAT

TABLE "Gender"        STUBS $gender      END TABLE
TABLE "Age"           STUBS $age_group   STATS col_pct, n, mean  END TABLE
TABLE "Q1 Satisfaction" STUBS $q1        STATS col_pct, n, mean  END TABLE
```

**Result**:
```
mrscript reconcile survey.mrs plan.csv
```
```
Reconciling: survey.mrs  ↔  plan.csv (3 planned table(s))

── COVERAGE ────────────────────────────────────────
  3 of 3 planned table(s) matched

3 matched
```

---

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Always (default) |
| `2` | `--fail-on error` and ≥1 error finding, or `--fail-on warn` and ≥1 warn/error finding |
| `1` | Invalid plan file or script error |

---

## Python API

```python
from mrscript.tabulation.job_runner import JobRunner
from mrscript.reconcile import ReconcileConfig

report = JobRunner().reconcile(
    script_text,
    plan_path="plan.xlsx",
    data_path=None,          # optional SOURCE override
    config=ReconcileConfig(
        fuzzy_threshold=0.85,
        extra_severity="info",   # "warn" for --strict mode
        check_banner=True,
        check_stats=True,
        check_base=True,
        sheet=None,              # xlsx sheet name
    ),
)
print(f"{report.matched}/{report.planned} matched")
for f in report.findings:
    print(f.level, f.kind, f.message)
```

### `ReconcileConfig` fields

| Field | Default | Description |
|-------|---------|-------------|
| `fuzzy_threshold` | `0.85` | Minimum `SequenceMatcher` ratio for a fuzzy title match |
| `extra_severity` | `"info"` | Severity for unplanned tables (`"warn"` with `--strict`) |
| `check_banner` | `True` | Run banner conformance check |
| `check_stats` | `True` | Run stats conformance check |
| `check_base` | `True` | Run base conformance check |
| `sheet` | `None` | xlsx worksheet name (None → first sheet) |
| `max_header_scan` | `15` | Max rows to scan when detecting the header |

### `ReconcileReport` properties

| Property | Type | Description |
|----------|------|-------------|
| `planned` | `int` | Number of plan rows parsed |
| `matched` | `int` | Number successfully matched |
| `missing` | `int` | Plan rows with no matching table (error level) |
| `extra` | `int` | Script tables not in the plan |
| `ok` | `bool` | `True` when `missing == 0` and no error/warn findings |
| `summary` | `dict` | `{"error": N, "warn": N, "info": N}` counts |
| `findings` | `list[Finding]` | All findings, ordered: coverage then conformance |
