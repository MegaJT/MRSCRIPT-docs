# Tutorial: Comparing Two Data Deliveries

When a fieldwork agency sends a second delivery of the same survey, you need to know exactly what changed. This tutorial walks through `mrscript diff` on a realistic wave-vs-wave scenario.

---

## Setup

You have two SPSS files:

- `wave1.sav` — the first delivery, 500 respondents
- `wave2.sav` — the updated delivery, 512 respondents (12 new, some edits)

Your MRScript script declares `%RESPID`:

```mrs
SOURCE "wave1.sav"
%RESPID = $respondent_id
```

---

## Step 1 — Run the diff

```bash
mrscript diff wave1.sav wave2.sav --script survey.mrs
```

Or if you don't have a script yet:

```bash
mrscript diff wave1.sav wave2.sav --key $respondent_id
```

You'll see a grouped text summary on stdout.

---

## Step 2 — Save machine-readable outputs

```bash
mrscript diff wave1.sav wave2.sav \
    --key $respondent_id \
    --json diff_report.json \
    --csv changed_cells.csv
```

`diff_report.json` contains counts and sample IDs for each finding.
`changed_cells.csv` is a flat worklist — one row per changed respondent × variable:

```
respid,var,old_value,new_value
1042,$region,1.0,4.0
1055,$q1,,3.0
```

---

## Step 3 — Understand the output

### SCHEMA section

```
── SCHEMA ─────────────────────────────────────────
  warn  $q7    type changed: single_punch → numeric
  info  $q12   new variable added
```

- `warn` findings need attention before running tables — a type change can break STUBS definitions.
- `info` findings are informational — a new variable won't break existing scripts.

### ROSTER section

```
── ROSTER ─────────────────────────────────────────
  warn  12 respondent(s) dropped  sample IDs: 1042, 1055, …
  info  24 respondent(s) added
```

Dropped respondents are `warn` — their data was removed. Check the full list in the JSON report (`roster.dropped_sample`) or ask the agency for a reconciliation.

Added respondents are `info` — extra completes. Your existing tables will pick them up automatically on the next run.

### VALUES section

```
── VALUES ─────────────────────────────────────────
  warn  $region       8 respondent(s) changed value
  warn  $q1           3 respondent(s) changed value  (null→value: 2, value→null: 1)
```

Value-level changes are always `warn`. Use `changed_cells.csv` to review each case and decide whether it is a legitimate edit or an error.

The `null→value` / `value→null` breakdown tells you whether the change is a new answer or a retraction.

### DISTRIBUTION SHIFT section

```
── DISTRIBUTION SHIFT ─────────────────────────────
  warn  $region  code 4 (West):  12% → 18%  (+6 pp)
  info  $q1      code 5 (Very satisfied):  21% → 24%  (+3 pp)
```

A +6 pp shift in one region code over 12 new respondents deserves a check: is the regional profile of the new completes expected?

`warn` = max shift ≥ 5 pp (the `--shift` threshold); `info` = smaller.

---

## Step 4 — Integrate into CI

Add a diff check to your pipeline so the team is alerted when an unexpected delivery lands:

```bash
mrscript diff wave1.sav wave2.sav \
    --key $respondent_id \
    --json diff_$(date +%Y%m%d).json \
    --fail-on warn
```

Exit code 2 triggers the pipeline failure; the JSON report is archived as an artefact.

---

## Restricting the diff

Focus on a subset of variables:

```bash
mrscript diff old.sav new.sav --key $id --vars region,q1,q2
```

Adjust the distribution shift threshold (default 5 pp):

```bash
mrscript diff old.sav new.sav --key $id --shift 10
```

---

## Next steps

- [Data Health check](health-check.md) — run QC on a single file
- [Data Diff reference](../reference/diff.md) — full option list and API docs
