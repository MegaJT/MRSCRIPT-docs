# 10 · Data Diff

**`mrscript diff`** compares two deliveries of the same dataset and answers three questions:

1. **Who changed?** — new respondents, dropped respondents.
2. **What changed?** — which variables, which cells, old value vs new value.
3. **How much shifted?** — frequency % by variable so a structural shift is visible without running a full crosstab.

---

## Basic usage

```
mrscript diff old.sav new.sav --key $respondent_id
mrscript diff old.sav new.sav --script survey.mrs
```

The `--key` / `--script` options both supply the respondent identifier:

| Option | How the key is resolved |
|--------|------------------------|
| `--key $var` | Uses the bare column name literally |
| `--script file.mrs` | Parses `%RESPID` from the script |

---

## Full option reference

```
mrscript diff old.sav new.sav --key $respondent_id [OPTIONS]

Options:
  --key VAR         Respondent key column (e.g. $respondent_id)
  --script FILE     .mrs script to read %RESPID from
  --json FILE       Write DiffReport as JSON to FILE
  --csv FILE        Write per-respondent value changes as CSV to FILE
  --tolerance N     Float comparison tolerance (default 1e-9)
  --shift N         Distribution shift threshold in pp (default 5)
  --vars V1,V2,...  Restrict diff to these columns (bare names, no $)
  --fail-on LEVEL   Exit 2 when findings at/above LEVEL exist (error|warn)
```

---

## What is compared

### Schema diff

Compared by variable name (prefixed, e.g. `$gender`).

| Finding | Severity |
|---------|----------|
| Variable present in old, absent in new | `warn` |
| Variable present in new, absent in old | `info` |
| Variable label changed | `info` |
| Value labels changed | `info` |
| Type changed (`single_punch` → `numeric`, etc.) | `warn` |
| Missing-code set changed | `warn` |

### Respondent roster diff

Matched by the key column.

| Finding | Severity |
|---------|----------|
| Respondent in old, absent in new (dropped) | `warn` |
| Respondent in new, absent in old (added) | `info` |

### Value diff

For respondents present in **both** files, every shared variable is compared cell by cell:

- `null → value` — variable filled in
- `value → null` — variable made null
- `value → value` — outright change

Each variable reports a count of changed respondents. Use `--csv` to get the full per-cell worklist.

Numeric columns use `--tolerance` (default `1e-9`) for float comparison.

### Distribution shift

For each shared variable the frequency % in old vs new is computed.
Variables with a max single-cell delta ≥ `--shift` threshold (default 5 pp) are flagged **warn**; smaller shifts are **info**.

- **Categorical** (`single_punch`, `multi_binary`): frequency % per code.
- **Numeric**: mean ± std comparison.
- `open_end` variables are skipped.

---

## Example output

```
Comparing: wave1.sav (n=500) → wave2.sav (n=512)
Key: $respondent_id

── SCHEMA ─────────────────────────────────────────
  warn  $q7    type changed: single_punch → numeric
  info  $q12   new variable added

── ROSTER ─────────────────────────────────────────
  warn  12 respondent(s) dropped  sample IDs: 1042, 1055, …
  info  24 respondent(s) added

── VALUES ─────────────────────────────────────────
  warn  $region       8 respondent(s) changed value
  warn  $q1           3 respondent(s) changed value  (null→value: 2, value→null: 1)

── DISTRIBUTION SHIFT ─────────────────────────────
  warn  $region  code 4 (West):  12% → 18%  (+6 pp)
  info  $q1      code 5 (Very satisfied):  21% → 24%  (+3 pp)

2 warn(s), 2 info(s)
```

---

## Output files

### JSON (`--json report.json`)

Machine-readable DiffReport with all sections, sample IDs, and per-code shift details.

```json
{
  "old_n": 500,
  "new_n": 512,
  "key_col": "respondent_id",
  "summary": {"warn": 2, "info": 2},
  "roster": {"dropped_count": 12, "added_count": 24, ...},
  "value_changes": [{"var": "$region", "changed_count": 8, ...}],
  "dist_shifts": [{"var": "$region", "level": "warn", "max_shift_pp": 6.0, ...}]
}
```

### CSV (`--csv changes.csv`)

Flat worklist of every changed cell — one row per (respondent × variable):

```
respid,var,old_value,new_value
1042,$region,1.0,4.0
1055,$q1,3.0,5.0
```

---

## Exit codes

| Exit code | Meaning |
|-----------|---------|
| 0 | Completed (default, regardless of findings) |
| 2 | Findings at/above `--fail-on` level were found |

Use `--fail-on warn` in CI to fail when any warn-level difference is detected.

---

## API usage

```python
from mrscript.diff import DataDiffer, DiffConfig, DiffReporter
from mrscript.variable_engine.registry import load_data

old_df, old_store = load_data("wave1.sav")
new_df, new_store = load_data("wave2.sav")

cfg = DiffConfig(shift_threshold_pp=5.0)
report = DataDiffer(cfg).compare(old_df, old_store, new_df, new_store, "respondent_id")

print(DiffReporter().text(report))
```

`DataDiffer` is standalone — it does not run a script or job. It accepts any two `(df, store)` pairs produced by `load_data`.
