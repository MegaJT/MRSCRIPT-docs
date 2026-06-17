# Tutorial: pre-flight health check

Before you tab a single table, run `mrscript health` to answer the #1 DP
question — *"did the data arrive clean?"* This takes one minute and can save
hours of rework if something is wrong.

This tutorial walks through a realistic data-arrival scenario: a fresh `.sav`
lands in your inbox, you check it, find two problems, and add routing assertions
so the same checks run automatically on every subsequent delivery.

---

## Step 1 — Run the health check

Point `mrscript health` at your existing script (or just a `SOURCE` declaration):

```powershell
mrscript health survey.mrs
```

That's it. Tablix loads the data, runs all built-in checks, and prints a
grouped summary:

```text
1 error(s), 2 warning(s)  [survey.sav]

── ERROR ──
  duplicate_respid  $respondent_id  n=3
    3 respondent ID value(s) appear more than once in the data.
    sample IDs: 1042, 1099, 1201

── WARN ──
  out_of_range  $region  n=7
    7 row(s) carry unlabelled code(s) for 'Region' ($region).
    codes: 9  sample IDs: 2031, 2045 ...

  high_item_nonresponse  n=2
    2 respondent(s) are non-responsive on more than 80% of analysed variables.
    sample IDs: 1055, 1072
```

Three findings: one error and two warnings. You'd investigate the duplicates
(almost certainly a data-delivery bug) and discuss the rogue `region = 9` with
the fieldwork team.

---

## Step 2 — Get a machine-readable report

For a record you can share or feed into a cleaning script, add `--json` and/or
`--csv`:

```powershell
mrscript health survey.mrs --json health_report.json --csv flagged.csv
```

`flagged.csv` lists one row per (respondent × finding) — ready to paste into an
SPSS `PROCESS IF` or a `pandas` drop statement to remove the bad records before
retabbing.

---

## Step 3 — Understand what is checked automatically

Tablix runs these checks on every `mrscript health` call with no extra
configuration:

| Check | What it catches |
|-------|----------------|
| `duplicate_respid` | Same `%RESPID` on two or more rows |
| `null_respid` | Blank / null respondent ID |
| `out_of_range` | A coded variable carries a value that isn't in its label set and isn't declared `MISSING` |
| `all_missing_respondent` | A respondent is blank on *every* analysed variable |
| `high_item_nonresponse` | A respondent is blank on more than 80 % of analysed variables |
| `base_zero_with_data` | A table's filter leaves zero respondents in scope |
| `count_exceeds_base` | A cell count exceeds its base (weighting / aggregation error) |
| `pct_not_summing` | Single-punch column %s don't add to ~100 |
| `missing_code_as_stub` | A `MISSING`-declared code still appears as a live stub |

The structural checks (`base_zero_with_data` etc.) need table results, so they
only fire when your script contains at least one `TABLE`.

---

## Step 4 — Check a rating battery for straightliners

If your survey has a rating grid (e.g. brand imagery on a 1–5 scale), Tablix
can flag respondents who gave the same answer across every column. This is the
**straightliner** check.

If your script already uses `STACK` to create a stacked concept frame, Tablix
picks up the battery automatically from the `MAP` source list — no extra work:

```mrs
STACK @concept
  MAP @rating FROM $rate_A, $rate_B, $rate_C, $rate_D
END STACK
```

Re-running health will now include:

```text
── WARN ──
  straightliner  @rating  n=4
    4 respondent(s) gave identical answers across the battery.
    sample IDs: 1001, 1088, 2031, 2200
```

---

## Step 5 — Check for speeders (optional)

If your data includes an interview-duration column, you can flag respondents
who completed too quickly. Declare the duration variable in a `HealthConfig`
when calling the Python API, or check the `duration_var` key in the
[Configuration reference](../reference/health.md#health-config).

In the CLI, battery and duration hints come from what's declared in the script;
arbitrary per-run config is a Python API feature.

---

## Step 6 — Write routing assertions with `EXPECT`

The most powerful health feature is `EXPECT` — you write the **routing logic
of your questionnaire** directly in the script, and Tablix verifies it against
the data.

The idea: *"if a respondent was routed into Q5 (because Q4 = 1), they must
have answered Q5; if they were not routed in, Q5 must be blank."*

```mrs
// Q5 must be answered for respondents who were routed in (Q4 = 1)
EXPECT $q5 ANSWERED WHERE $q4 = 1

// Q5 must be blank for respondents who were not routed in (Q4 ≠ 1)
EXPECT $q5 MISSING WHERE $q4 != 1
```

Run health again:

```text
── ERROR ──
  expect_violation  n=6
    EXPECT #1: 6 respondent(s) violate the assertion (in 34 scoped rows)
    sample IDs: 1031, 1045, 1099 ...

  expect_violation  n=2
    EXPECT #2: 2 respondent(s) violate the assertion (in 71 scoped rows)
    sample IDs: 2031, 2200
```

Eight routing errors across two assertions. You'd send the `flagged.csv` back
to the fieldwork agency with a request to re-check those records.

### What `MISSING` and `ANSWERED` mean

| Sugar | Meaning |
|-------|---------|
| `$var MISSING` | the column is null — the respondent did not answer |
| `$var ANSWERED` | the column is not null — the respondent gave a value |

You can also use any general condition — `EXPECT $consent = 1`,
`EXPECT $region IN (1, 2, 3, 4)` — and combine conditions with `AND` / `OR` /
`NOT`, exactly as in `FILTER` or `DERIVE`.

---

## Step 7 — Gate CI on health

Once you trust the assertions, add `--fail-on error` to your delivery pipeline
so a broken file can never slip through:

```powershell
mrscript health survey.mrs --json report.json --fail-on error
# exits with code 2 if any error-level findings exist
```

---

## The finished script (with health assertions)

```mrs
SOURCE 'survey.sav'
%RESPID = $respondent_id

// ─── Routing assertions (checked by mrscript health, ignored by run/export) ─
EXPECT $q5 ANSWERED WHERE $q4 = 1
EXPECT $q5 MISSING  WHERE $q4 != 1
EXPECT $region IN (1, 2, 3, 4)

// ─── Variable overrides ──────────────────────────────────────────────────────
VARIABLE $q1  MISSING 99  END VARIABLE

// ─── Tables ──────────────────────────────────────────────────────────────────
TABLE 'Overall satisfaction'
  STUBS $q1  BANNER $gender  STATS n, col_pct
END TABLE
```

`EXPECT` declarations are invisible to `mrscript run` and `mrscript export` —
they only fire during `mrscript health`. You can keep them in the script
permanently as living documentation of the questionnaire's skip logic.

---

## What you've learned

| Command / clause | What it does |
|-----------------|--------------|
| `mrscript health script.mrs` | Run all health checks, print grouped summary |
| `--json file` | Machine-readable report |
| `--csv file` | Flagged-respondent worklist for cleaning |
| `--fail-on error` | Non-zero exit for CI gating |
| `EXPECT cond WHERE cond` | Assert routing logic; violations are findings |
| `$var MISSING` | Sugar for "this column is null" |
| `$var ANSWERED` | Sugar for "this column is not null" |

---

## Where to go next

<div class="grid cards" markdown>

-   :material-format-list-checks: **[How-to recipes](../guide/patterns.md)** —
    ready-made patterns for concept tests, diaries, cleaning, and more.

-   :material-book-open-variant: **[Health reference](../reference/health.md)** —
    every check code, severity, and configuration option.

-   :material-table: **[First crosstab tutorial](first-crosstab.md)** —
    if you skipped straight here, the tabulation walkthrough is a good companion.

</div>
