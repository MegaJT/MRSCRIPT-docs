# 15 · Cell Provenance

**`mrscript provenance`** drills **one table cell** — a stub row × a banner column,
optionally a **NET** — back to the **respondent IDs (`%RESPID`)** that contribute to
it, with their weights and a CSV worklist. It answers the everyday DP question
*"who are the people in this cell?"* without dropping into SPSS to rebuild the
filter by hand.

```
$ mrscript provenance survey.mrs --table 1 --row 5 --col "$gender = 1" --data survey.sav

Table 1 - Overall Satisfaction
  Cell:  Strongly Agree (code 5)  x  Male (A)
  Base:  5 in column        Count: 2

  Contributing respondents (%RESPID):
    respondent_id    weight     row
    5                -          4
    10               -          9

  2 contributing row(s).
```

It is a **read over the same masks the engine already counted** — the cell's
respondents are exactly the rows where *that stub's evaluator*, *the column's
evaluator*, and *the table FILTER/SCOPE* all hold. Nothing is re-tabulated.

---

## Addressing a cell

A cell is `--table` + `--row` (+ `--section`) + `--col`:

| Option | Meaning |
|--------|---------|
| `--table T` | The displayed **`Table N`** number, or a table **`NAME`** / title. **Required.** |
| `--row R` | A stub **code** (`5`), a stub **label** (`"Strongly Agree"`), or a **NET label** (`"Top 2 Box"`). **Required.** |
| `--section S` | Which `STUBS` variable, when a table stacks several as sections (0-based; default `0`). |
| `--col C` | A banner column (below). Omit it for the **Total** column. |
| `--data FILE` | Override the script's `SOURCE` (a dataset is required — provenance reads real rows). |
| `--out FILE` | Write the **full** respondent worklist as CSV (atomic). |
| `--limit N` | Cap how many rows print to stdout (default 50; `--out` always writes all). |

### Ways to name the column (`--col`)

Resolved in this order — pick whichever is convenient:

| Form | Example | Notes |
|------|---------|-------|
| (omitted) / `TOTAL` | `--col TOTAL` | The leading **Total** column. |
| 0-based index | `--col 2` | Leaf position in the banner. |
| Column label | `--col Male` | Exact, case-insensitive. |
| Significance letter | `--col A` | The `A`/`B`/`C`… letter shown in the header. |
| Condition | `--col "$gender = 1"` | Matched by **evaluator identity** — works for crossed banners too: `--col "$city = 1 AND $gender = 2"` matches the crossed leaf regardless of operand order. |

The condition form reuses the same banner-column resolver as `SORT ON` / `INDEX ON`,
so `$a=1 AND $b=2` and `$b=2 AND $a=1` address the same crossed leaf.

The `--row` match is **order-independent**, so it is robust to `SORT` / `RANKING`
having reordered the rendered rows — `--row 5` always finds the code-5 stub.

---

## What you get

For each contributing respondent the worklist lists the **`%RESPID`** value, the
**weight** (when the table is weighted), and the 0-based **frame row** index. The
printed summary reports the unweighted **count** (which equals the cell's `n`) and,
when weighted, the **weighted count Σw** (which equals the cell's `weighted_n`).

A **NET** row drills to the de-duplicated union of its members — `--row "Top 2 Box"`
on a top-2-box net returns everyone coded 4 **or** 5, counted once.

---

## Weighting

On a weighted table each contributing row carries its weight, and the reported
**weighted count is Σw** over those rows:

```
$ mrscript provenance survey.mrs --table 2 --row 1 --col TOTAL --data survey.sav

Table 2 - Recommend
  Cell:  Yes (code 1)  x  Total
  Base:  4 in column        Count: 2   weighted sum(w) = 3

  Contributing respondents (%RESPID):
    respondent_id    weight     row
    1                2          0
    3                1          2
```

---

## Stacked (`LEVEL`) tables

On a stacked table the unit of analysis is an **exposure** (respondent × slot), so a
cell selects **exposure rows**. Provenance reports both the contributing exposure
rows — each tagged with the **real respondent** behind it (the `%RESPID` carried on
the stacked `__resp__` key) — and the number of **distinct respondents**:

```
Table 1 - Concepts
  Cell:  Concept A (code 1)  x  Total
  Base:  12 in column        Count: 4   (4 exposure rows, 4 respondents)
  ! stacked (LEVEL) table - 4 exposure row(s) from 4 respondent(s) (id = real %RESPID via __resp__)
```

With `BASE respondents` the reported count is the **distinct** respondents; with the
default row base it is the exposure rows.

---

## MIN_BASE, suppressed, and no-base cells

Provenance is an **internal audit tool** — it always reveals the contributing
respondents (that is the point) — but it **flags** when the rendered cell is masked
or blank:

- **No-base** — the column's base is 0, so the cell renders blank. The contributing
  set is empty; the result is flagged `no_base`.
- **MIN_BASE / CONFIDENTIAL masked** — the column base is below the table's
  `MIN_BASE` / `CONFIDENTIAL N`, so the cell renders `<<` / `**`. The IDs are **still
  listed**, with a note naming the threshold. Treat the worklist as confidential, like
  the data itself.

When **no `%RESPID`** is declared there is no respondent key, so the worklist reports
the **0-based DataFrame row position** as the id (and says so). A stable `row` index
is always included regardless.

---

## Scope (v1)

Provenance drills **plain banner tables** — `STUBS <var>` (leaves + **NET**s),
including nested `STUBS @a BY @b`. It returns a clear *"not drillable in v1"* error
for **GRID** / **SUMMARY** tables, **DISTRIBUTION** / **ADD** sections, numeric
(mean/std) sections, and cross-table outputs (**ADDTAB** / **MANIP** /
**BANKED_TABLE**) — those have no single stub-evaluator-per-row to read.

---

## Exit codes

| Situation | Exit |
|-----------|------|
| Cell drilled (including a masked / no-base cell, with notes) | 0 |
| Bad `--table`/`--row`/`--col`, an undrillable table kind, or a load/validation failure | 1 |

---

## Worked example, end to end

```mrs
SOURCE "survey.sav"
%RESPID = $respondent_id

DERIVE @t2b
    NET "Top 2 Box"
        STUB 4 WHERE $satisfaction = 4
        STUB 5 WHERE $satisfaction = 5
    ENDNET
    STUB 3 "Neutral" WHERE $satisfaction = 3
END DERIVE

TABLE "Satisfaction (grouped)" NAME "sat"
    STUBS @t2b
    BANNER $gender
END TABLE
```

Drill the **Top-2-Box × Total** cell and save the worklist:

```
$ mrscript provenance survey.mrs --table sat --row "Top 2 Box" --col TOTAL \
      --data survey.sav --out t2b_total.csv

Table 1 - Satisfaction (grouped)
  Cell:  Top 2 Box (NET)  x  Total
  Base:  9 in column        Count: 6

  Contributing respondents (%RESPID):
    respondent_id    weight     row
    1                -          0
    2                -          1
    4                -          3
    5                -          4
    7                -          6
    10               -          9

  6 contributing row(s).
Wrote t2b_total.csv  (6 rows)
```

`t2b_total.csv`:

```csv
respondent_id,weight,row_index
1,,0
2,,1
4,,3
5,,4
7,,6
10,,9
```

---

## Python API

```python
from mrscript.tabulation.job_runner import JobRunner
from mrscript.tabulation.provenance_report import to_csv

result = JobRunner().provenance(
    open("survey.mrs").read(),
    table=1, row="5", col="$gender = 1",
    data_path="survey.sav",
)
print(result.count, [c.respondent_id for c in result.contributions])
open("worklist.csv", "w").write(to_csv(result))
```

`provenance()` runs the same pipeline as `run()` and returns a `ProvenanceResult`
(count, weighted count, base, the contributing respondents with weights, and audit
notes).

---

See also: [Tables](tables.md), [Significance](reference-details.md), [Data
Health](health.md).
