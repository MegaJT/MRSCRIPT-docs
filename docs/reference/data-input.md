# Part 2 · Data input

## 5. SOURCE {#source}

```mrs
SOURCE [name] "file" [AS fmt] [DATA "companion"] [CODEBOOK "book"] [SCHEMA "layout"]
```

Declares a data file. A script may declare one `SOURCE` (the common case) or several
named `SOURCE`s combined with `APPEND` / `JOIN` (see [§7](#append--join)).

### Clauses

| Clause | Meaning |
|--------|---------|
| `name` | Optional dataset name (an unquoted identifier). Required only when more than one `SOURCE` is declared, so `APPEND` / `JOIN` can refer to each file by name. |
| `AS fmt` | Force the loader format instead of detecting it from the file extension (see table below). |
| `DATA "file"` | Companion data file for split formats (Triple-S / MDD-DDF), where metadata and case data live in separate files. |
| `CODEBOOK "file"` | Sidecar metadata file (JSON) supplying labels, value labels, missing codes, types, and scores for code-only data. Applied first; inline `VARIABLE` blocks ([§10](setup-blocks.md#variable)) then override it (inline wins). |
| `SCHEMA "file"` | Fixed-width column layout file for the `ascii` format. |

### Formats

| `fmt` | Format | Status |
|-------|--------|--------|
| `spss` | SPSS `.sav` | **implemented** |
| `csv` | comma-separated | **implemented** |
| `triple_s` | Triple-S XML + data | staged |
| `ascii` | fixed-width + schema | staged |
| `mdd` | Dimensions `.mdd`/`.ddf` | staged |

> *Staged* formats are recognised but their readers are not yet built — using one
> raises a clear "not yet supported" error.

**Format detection** is by file extension (`.sav`→`spss`, `.csv`→`csv`,
`.sss`→`triple_s`, `.mdd`→`mdd`) unless `AS` overrides it.

### Examples

```mrs
SOURCE 'C:\Projects\Toothpaste\data.sav'         // SPSS, single file
SOURCE 'data/survey.csv'                          // CSV
SOURCE "raw/codes.csv" CODEBOOK "meta/book.json"  // coded CSV + codebook
SOURCE "data.txt" AS csv                          // force CSV on a .txt file
```

!!! note "Notes"

    - Source variable names come from the file's column names.
    - SPSS variable labels and value labels load automatically as defaults; all are
      overridable in `VARIABLE` blocks ([§10](setup-blocks.md#variable)) or a `CODEBOOK`.
    - SPSS missing-range metadata is **not** auto-applied — declare missing codes in
      the script ([§25](reference-details.md#missing-values)) so scripts behave
      identically across formats.
    - The CLI flag `--data FILE` overrides the `SOURCE` path for single-source
      scripts (see [§21](output.md#cli)).

### The codebook JSON format

Used by `CODEBOOK` and by the codebook written by `EXPORT DATA` ([§20](output.md#export-data)):

```json
{
  "variables": {
    "$gender": {"label": "Gender", "type": "single_punch",
                "values": {"1": "Male", "2": "Female"}},
    "$q1":     {"label": "Satisfaction",
                "values": {"1": "Low", "5": "High"},
                "missing": [99], "scores": {"1": 1, "5": 5}}
  }
}
```

---

## 6. %RESPID — canonical respondent key {#respid}

```mrs
%RESPID = $source_var
```

Binds the reserved constant `%RESPID` to the variable that uniquely identifies a
respondent. One declaration per script; the variable must exist and not be
open-ended text.

**Effects**

- Enables `BASE respondents` on a **non-stacked** table — counts de-duplicate by the
  real respondent key (useful when raw data has multiple rows per respondent).
  Without `%RESPID`, `BASE respondents` on a non-stacked table is a validation error.
- On a `STACK`ed (`LEVEL`) table, unique-respondent counts follow the true
  respondent across exposures, instead of a positional index.
- Is the key used by `JOIN … ON %RESPID` ([§7](#append--join)).

`%RESPID` is **optional**. A job that tabulates without it still runs, but emits a
non-fatal advisory.

```mrs
SOURCE 'survey.sav'
%RESPID = $respondent_id
```

---

## 7. APPEND / JOIN (combining multiple files) {#append--join}

Declare several **named** `SOURCE`s, then combine them with exactly one `APPEND` or
`JOIN` into a single working dataset. Everything downstream (transforms, tables,
export) is unchanged — it just sees one combined dataset.

### APPEND — stack rows (waves / more respondents)

```mrs
APPEND name1, name2 [, name3 ...]
```

Concatenates the rows of the named datasets, unioning columns (a column missing from
one file is null-filled there). Variable metadata is merged; the first file's
definition wins on shared columns.

```mrs
SOURCE w1 'wave1.sav'
SOURCE w2 'wave2.sav'
APPEND w1, w2
TABLE 'Gender (both waves)' STUBS $gender END TABLE
```

### JOIN — add columns (profiling / more variables)

```mrs
JOIN left WITH right ON %RESPID|$keyvar [TYPE left|inner]
```

Matches rows of the two datasets on a key column and adds the right file's non-key
columns. The key is `%RESPID` (requires a `%RESPID` declaration) or an explicit
`$variable` present in both files. `TYPE` is `left` (default — keep all left rows) or
`inner` (matched rows only).

A non-key column present in **both** files is an error: rename, drop, or recode one
side first (only the key may be shared).

```mrs
SOURCE main 'respondents.sav'
SOURCE prof 'demographics.sav'
%RESPID = $respondent_id
JOIN main WITH prof ON %RESPID
TABLE 'Income (from profile file)' STUBS $income END TABLE
```

!!! note "Rules"

    - When more than one `SOURCE` is declared, each must be named and exactly one
      `APPEND`/`JOIN` must reference them.
    - One combine operation per script (chained append-then-join is not yet
      supported).
