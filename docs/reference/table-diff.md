# 16 · Table-Run Diff

**`mrscript table-diff A B`** compares **two sets of produced tables** — this
week's run vs last week's, a script before and after an edit — and reports what
changed: tables added or removed, structural changes (banner columns, stub rows,
stats, base description), and per-cell value shifts (n / col_pct / mean) that
exceed configurable thresholds.

This is **not** a data diff (`mrscript diff` — Feature #10 — compares two
*datasets*). `table-diff` compares two *output runs*: the `TableResult`s that a
script produces.

```
$ mrscript table-diff last_week/ this_week/

Comparing: last_week  (4 tables) → this_week  (5 tables)

── TABLES ─────────────────────────────────────────────────────
  error  Table 4 'Age 18-34' present in 'last_week' but not in 'this_week'
  error  Table 5 'Awareness' present in 'this_week' but not in 'last_week'

── STRUCTURE ──────────────────────────────────────────────────
  warn   Table 1 'Overall' — Banner column 'Under-30s' removed
  warn   Table 1 'Overall' — Banner column 'Working Adults' added

── CELL SHIFTS ────────────────────────────────────────────────
  warn   Table 1 'Overall'  row 'Agree' / col 'Total'   col_pct: 34.0% → 41.0% (+7.0 pp)
  warn   Table 1 'Overall'  row 'Agree' / col 'Male'    col_pct: 55.0% → 62.0% (+7.0 pp)
  info   Table 1 'Overall'  row 'Agree' / col 'Female'  col_pct: 32.0% → 33.0% (+1.0 pp)

2 error(s), 3 warn(s), 1 info(s)
```

---

## Inputs: stored runs vs live scripts

`A` and `B` can be either:

| Input kind | Example |
|------------|---------|
| Saved `TableStore` directory (contains `manifest.json` + `cells.parquet`) | `last_week/` |
| A `.mrs` script (run on-the-fly, needs `--data-a` / `--data-b` if the source file moved) | `survey_v1.mrs` |

When a `.mrs` script is given, `mrscript table-diff` runs the script through the
engine and then diffs the resulting stores — equivalent to `mrscript run` followed
immediately by a diff. A saved store is loaded directly with no re-tabulation.

Persist a run for later comparison with **`mrscript export-store`** (or in-script
via `TABLE SAVE "handle"`), or by using the `--out-store` flag on `mrscript run`.

---

## Table matching cascade

Tables are matched greedy one-to-one in A's script order:

| Step | Match on | Notes |
|------|----------|-------|
| 1 | Store key (case-insensitive) | The `NAME "handle"` given in `TABLE … NAME`; or the title when no name given |
| 2 | `Table N` sequence number | Exact integer match |
| 3 | Normalised title | Lowercase + punctuation stripped (e.g. `"Overall:"` ↔ `"Overall"`) |

**No fuzzy matching** — a structural diff needs exact matches; a false-positive match
between different tables produces meaningless cell comparisons. Unmatched A-tables
are *removed*; unmatched B-tables are *added*.

---

## Report findings

### Table findings

| Kind | Level | When |
|------|-------|------|
| `added` | `error`* | A table exists in B but not in A |
| `removed` | `error`* | A table exists in A but not in B |
| `renumbered` | `info` | Matched by name or title; sequence numbers differ |

*Configurable via `--missing-severity` and `--extra-severity`.*

### Structure findings (within matched tables)

| Kind | Level |
|------|-------|
| `row_added` / `row_removed` | `warn` |
| `col_added` / `col_removed` | `warn` |
| `section_added` / `section_removed` | `warn` |
| `section_label_changed` | `warn` |
| `stats_changed` | `info` |
| `base_changed` | `info` |

### Cell shifts (within common rows × columns)

| Stat | Threshold | Severity |
|------|-----------|----------|
| `col_pct` | `--pct-threshold` (default 2.0 pp) | `warn` if > threshold; `info` if ≤ threshold but ≠ 0 |
| `n` | `--n-threshold` (default 0 = any change) | `warn` |
| `weighted_n` | `--weighted-n-threshold` (default 0.5) | `warn` |
| `mean` (summary rows) | `--mean-threshold` (default 0.1) | `warn` |

Cells where one side had a value and the other has `None` are flagged
`appeared` / `disappeared` (level `warn`).

---

## CLI reference

```
mrscript table-diff A B [options]

Positional:
  A                  Run A: saved TableStore dir or .mrs script
  B                  Run B: saved TableStore dir or .mrs script

Data options:
  --data-a FILE      Dataset for run A when A is a script (overrides SOURCE)
  --data-b FILE      Dataset for run B when B is a script (overrides SOURCE)

Threshold options:
  --pct-threshold N  col_pct shift warn level in pp (default: 2.0)
  --n-threshold N    n diff warn level (default: 0 = any change)
  --mean-threshold N mean/summary shift warn level (default: 0.1)

Output options:
  --json FILE        Write full report to FILE as JSON
  --csv  FILE        Write cell shifts to FILE as CSV (always written when flag given)
  --fail-on error|warn  Exit 1 when findings at that severity or higher exist
```

---

## Python API

```python
from mrscript.table_diff import TableDiffConfig, TableDiffer, TableDiffReporter

cfg = TableDiffConfig(pct_threshold=5.0)
report = TableDiffer(cfg).compare(store_a, store_b, a_label="Week 1", b_label="Week 2")

print(TableDiffReporter().text(report))

if not report.ok:
    for f in report.table_findings + report.structure_findings:
        print(f.level, f.message if hasattr(f, "message") else f.detail)
    for c in report.cell_shifts:
        print(c.stat, c.row_label, c.col_label, c.delta)
```

`compare()` takes two `TableStore` objects (from `JobRunner.run(..., return_store=True)`
or `table_store_io.load(path)`).

`TableDiffReport` properties:

| Property | Type | Description |
|----------|------|-------------|
| `ok` | `bool` | True when no error or warn findings |
| `summary` | `dict[str, int]` | `{"error": N, "warn": N, "info": N}` |
| `table_findings` | `list[TableFinding]` | Added / removed / renumbered tables |
| `structure_findings` | `list[StructureFinding]` | Banner / row / stats / base changes |
| `cell_shifts` | `list[CellShift]` | Per-cell value changes |

---

## Worked example

**Script before edit (`v1.mrs`):**
```mrs
SOURCE "survey.sav"
TABLE "Overall Satisfaction"
    STUBS $q1
    BANNER $gender
    STATS col_pct, n
END TABLE
```

**Script after edit (`v2.mrs`)** — adds a banner column, changes a stub label:
```mrs
SOURCE "survey.sav"
TABLE "Overall Satisfaction"
    STUBS $q1
    BANNER $gender, $ageband
    STATS col_pct, n
END TABLE
```

```
$ mrscript table-diff v1.mrs v2.mrs --data survey.sav

Comparing: v1.mrs  (1 tables) → v2.mrs  (1 tables)

── STRUCTURE ──────────────────────────────────────────────────
  warn   Table 1 'Overall Satisfaction' — Banner column '18-34' added
  warn   Table 1 'Overall Satisfaction' — Banner column '35-54' added
  warn   Table 1 'Overall Satisfaction' — Banner column '55+' added

── CELL SHIFTS ────────────────────────────────────────────────
  (none — same data, same stubs, same existing columns)

3 warn(s)
```

Use `--fail-on warn` in CI to gate a release on no structural changes:

```bash
mrscript table-diff baseline/ new_run/ --fail-on warn
```

---

## Notes

- Cell alignment is **by label**, not by position — reordering banner columns or
  stub rows does not create spurious cell shifts.
- Summary rows (Mean, Std Dev, …) are compared against the `mean_threshold`.
- The diff reads already-produced `TableResult` objects; it does not re-run the
  tabulation engine, apply weighting, or recompute significance.
- Significance flags are not compared (they depend on base sizes that may naturally
  drift between waves).
- GRID / SUMMARY / DISTRIBUTION / cross-table outputs (ADDTAB / MANIP / BANKED) are
  matched by title/number but cell comparisons work only on standard banner tables
  with numeric `col_pct` / `n` cells.
