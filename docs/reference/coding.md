# 18 · Open-End Coding

**`mrscript code`** codes free-text **verbatim** answers ("Why did you choose that
brand?") into a frame of **themes**, then writes the result back as a **binary-coded
variable group** the rest of the pipeline can tabulate. You supply a *coding frame*
(a flat `code,label` list) and an `open_end` source variable; it produces a wide 0/1
matrix — one column per theme, keyed by respondent — that re-imports as a `CODEBOOK`
companion.

```
$ mrscript code survey.mrs $why_brand --frame frame.csv --out coded.csv --ai

Coded 480 respondent(s) -> coded.csv (412 assigned >=1 theme)
frame: 6 theme(s); mode: AI-assisted
```

The summary goes to **stdout**; the status line to **stderr**.

---

## The pieces

| Input | What it is |
|-------|-----------|
| `survey.mrs` | Your script. Its `SOURCE` (+ any `KEEP/DROP/EDIT` cleaning) defines the **universe** that gets coded — a respondent dropped from the tables is not coded. Can be a full tab script or a minimal `SOURCE` + `%RESPID` stub. |
| `$why_brand` | The **open-end** source variable (the free-text column). Must be `open_end`. |
| `--frame frame.csv` | The analyst's **theme frame** — a two-column CSV (see below). |
| `--out coded.csv` | The wide **0/1 dichotomy** output, `{var}_{code}` columns, keyed by respondent. |

### The coding frame CSV

A header row plus one row per theme. Columns may be in either order; the header is
case-insensitive; extra columns are ignored.

```csv
code,label
1,Price / value for money
2,Taste / flavour
3,Health / natural ingredients
4,Brand trust / reputation
99,Other / miscellaneous
```

Codes must be **integers, unique**, with **non-empty labels** (a bad frame fails fast,
before any data is loaded). Frame order is preserved — output columns follow it.

### The output

One `Int32` 0/1 column per frame code, named `{var}_{code}` (the same convention
[multi-response export](data-preparation.md) uses), keyed by the respondent id
(`%RESPID`, or a 0-based row index when none is declared):

```csv
respondent_id,why_brand_1,why_brand_2,why_brand_3,why_brand_4,why_brand_99
1,1,1,0,0,0
2,1,0,0,0,0
3,0,0,1,0,0
...
```

A respondent who expresses no theme is a valid all-zero row.

---

## Manual vs AI

| Mode | What happens |
|------|-------------|
| **manual** (default) | Every verbatim is assigned **no codes** — an all-zero **shell**. Use it to scaffold the output file + codebook, then fill the codes in yourself (or, in a future release, in the desktop coding panel). |
| **`--ai`** | Each distinct, non-empty verbatim is sent to an LLM with the frame; the model returns the codes it expresses. Hallucinated codes (not in the frame) are dropped; identical verbatims are coded once. |

`--ai` is a **starting point**, not a final answer — review the assignments before
delivery. `--max-codes N` caps how many themes one verbatim can receive.

### Configuration — shared with `assist` / `suggest`

The `--ai` path uses the **exact same provider layer** as
[`assist`](assist.md#it-is-optional-and-configurable) — **no new environment
variables**. Set `ANTHROPIC_API_KEY` (Anthropic, the default), or
`MRSCRIPT_ASSIST_URL` (+ `MRSCRIPT_ASSIST_MODEL`) for a **local / OpenAI-compatible**
model (Ollama, LM Studio — no key, no cost). **Manual mode needs no provider** — it
works fully offline. With `--ai` and no provider configured, it prints a clear error
and exits 1.

---

## Round-trip: code once, tabulate forever

With `--codebook`, the output also gets a JSON codebook registering each theme column
as a labelled `multi_binary` variable (plus a `VariableGroup`). Re-`SOURCE` the pair
and the themes tabulate like any other multi-response set:

```
$ mrscript code survey.mrs $why_brand --frame frame.csv \
      --out coded.csv --codebook coded.json --ai
```

```
SOURCE "coded.csv" CODEBOOK "coded.json"

TABLE "Reasons for choice"
    STUBS $why_brand_1, $why_brand_2, $why_brand_3, $why_brand_4
    BANNER $gender
    STATS n, col_pct
END TABLE
```

The columns render with their **frame labels** ("Price / value for money", …) — the
coding loop is closed: load → code → write dichotomies + codebook → re-`SOURCE`.

---

## Full option reference

```
mrscript code SCRIPT $var --frame FILE --out FILE [OPTIONS]

Arguments:
  SCRIPT           Path to the .mrs (or .mrst) script (defines the universe)
  $var             Open-end source variable to code (accepts $var or bare var)

Options:
  --frame FILE     Coding frame CSV (code,label)                 (REQUIRED)
  --out FILE       Output 0/1 dichotomy CSV                      (REQUIRED)
  --data FILE      Data file — overrides SOURCE in the script
  --ai             Use the AI assist harness to suggest assignments
  --model NAME     Model override for --ai (else MRSCRIPT_ASSIST_MODEL / default)
  --codebook FILE  Also write a JSON codebook (multi_binary vars + group)
  --max-codes N    Cap codes assigned per verbatim (AI path)
```

---

## What it codes (v1 scope)

`code` codes a single **`open_end` source (`$`) variable**. It **refuses** (exit 1) a
categorical (`single_punch`/`numeric`/`multi_binary`), a weight, the `%RESPID` key, or
a derived (`@`) variable. To net a *categorical* into groups, use
[`suggest`](suggest.md); for range bands on a numeric, use [`assist`](assist.md).

A future release adds a **desktop coding panel** to review and correct AI assignments
one-by-one (writing the same CSV format) and **batched** LLM calls.

---

## Exit codes

| Situation | Exit |
|-----------|------|
| Coded successfully (manual or AI) | 0 |
| Bad frame file / unknown or non-open_end variable / `--ai` with no provider / load error | 1 |

---

## Python API

```python
from mrscript.coding import Coder, read_frame, write_coded
from mrscript.tabulation.job_runner import JobRunner

frame = read_frame("frame.csv")
runner = JobRunner()
results = runner.code(open("survey.mrs").read(), "$why_brand", frame, coder=Coder())
write_coded(results, frame, "$why_brand", runner.code_store, "coded.csv",
            key_name=runner.code_key_name, codebook_path="coded.json")
```

`Coder(backend=…)` accepts an injectable `LLMBackend` (tests pass a fake one, so no
network / key is touched); `Coder()` with no backend is the manual shell.

---

See also: [NL Assistant](assist.md), [Recode Suggester](suggest.md), [Data
Preparation](data-preparation.md), [Tables](tables.md).
