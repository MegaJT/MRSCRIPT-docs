# 13 · NL Assistant

**`mrscript assist`** turns a plain-English request into a **validated MRScript
snippet** — a `TABLE` or `DERIVE` block you paste into your script. It is a narrow
code generator for the two authoring tasks DP specialists repeat all day, **not** a
chatbot.

```
$ mrscript assist "tab q1 by gender and region with col%, n, sig and the mean" --data survey.sav

TABLE "Overall Satisfaction"
    STUBS $q1
    BANNER $gender, $region
    STATS col_pct, n, sig, mean
END TABLE

✓ validates against survey.sav
```

The snippet goes to **stdout** (clean for piping into your editor); the validation
line goes to **stderr**. With `--out FILE` it is also written to a file.

---

## It is optional — and configurable

`assist` is the one part of Tablix that calls an AI model, so it needs a provider.
**If none is configured it prints a clear error and exits — nothing else in Tablix
is affected.** Configure it with environment variables:

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Use **Anthropic's API** (the default provider). Default model `claude-haiku-4-5-20251001` — cheap and plenty for short snippets. |
| `MRSCRIPT_ASSIST_URL` | Use a **local / OpenAI-compatible** endpoint instead (Ollama, LM Studio, …). **No API key, no cost.** Requires `MRSCRIPT_ASSIST_MODEL`. |
| `MRSCRIPT_ASSIST_MODEL` | Model name. Overrides the Anthropic default; **required** with `MRSCRIPT_ASSIST_URL`. |
| `MRSCRIPT_ASSIST_KEY` | *(optional)* Bearer token for a hosted OpenAI-compatible gateway that needs auth. |

A set `MRSCRIPT_ASSIST_URL` always wins over an ambient `ANTHROPIC_API_KEY` (you
pointed at the local server on purpose).

### Anthropic (default)

```
export ANTHROPIC_API_KEY=sk-ant-...
mrscript assist "derive age groups under 35, 35-54, 55+" --type derive
```

### Local model (Ollama) — no key, no cost

```
export MRSCRIPT_ASSIST_URL=http://localhost:11434/v1
export MRSCRIPT_ASSIST_MODEL=qwen2.5-coder
mrscript assist "tab q1 by gender with sig"
```

### No provider configured

```
$ mrscript assist "tab q1 by gender"
error: No AI provider configured for `mrscript assist`. Either:
  * set ANTHROPIC_API_KEY to use Anthropic's API (default model claude-haiku-4-5-20251001), or
  * set MRSCRIPT_ASSIST_URL (plus MRSCRIPT_ASSIST_MODEL) to use a local or OpenAI-compatible model ...
The rest of mrscript works without this.
```

---

## Full option reference

```
mrscript assist "<request>" [OPTIONS]

Options:
  --data FILE      Dataset (.sav/.csv) — grounds the request on real variables
                   and enables semantic validation (strongly recommended)
  --type table|derive   Bias toward a TABLE or a DERIVE block (default: inferred)
  --model NAME     Model override (else MRSCRIPT_ASSIST_MODEL / the default)
  --out FILE       Also write the generated snippet to FILE
```

---

## What it generates (v1 scope)

| Task | Example request | Produces |
|------|-----------------|----------|
| **table** | "tab q1 by gender with sig", "crosstab awareness by region filtered to aware", "top-2-box of the agreement battery by gender" | a `TABLE … END TABLE` block (with an inline `BANNER`/`FORMAT` only if needed) |
| **derive** | "derive age groups under 35 / 35–54 / 55+", "net the top two brands", "collapse the 5-point scale into agree/neutral/disagree" | a `DERIVE … END DERIVE` block |

v1 generates **TABLE and DERIVE** blocks (plus the `BANNER`/`FORMAT` a table may
need). `STACK`, `RIM`, transforms, cross-table ops, and whole-script generation are
out of scope.

---

## How the request is grounded

The model is given three things so its output references *your* data and the *real*
grammar:

1. **A curated MRScript mini-reference** — the grammar for `TABLE`/`DERIVE`/
   `BANNER`/`FORMAT`, the `$`/`@` prefix convention, and the STATS tokens.
2. **Hand-written few-shot examples** of the most common patterns.
3. **Your dataset's variable catalog** — when you pass `--data`, the real variable
   names, types, and value labels, so it writes `$gender` not `$sex`.

---

## Validation, and the `--data` recommendation

Every snippet is run through the **real MRScript parser and validator** before you
see it. If it fails, the assistant feeds the errors back to the model and tries once
more (max 2 rounds). After that it shows the best draft **with the errors**, so you
get something to fix rather than nothing.

- **With `--data`** — full validation: syntax **and** variable references are
  checked against the dataset. The status line reads `✓ validates against <file>`.
- **Without `--data`** — **syntax only** (there is no dataset to check variables
  against). The status line reads `ℹ syntax OK (pass --data to check variables)`.

Pass `--data` whenever you can — it improves both the grounding *and* the
validation.

---

## Worked examples

### A table

```
$ export ANTHROPIC_API_KEY=sk-ant-...
$ mrscript assist "show q1 and q2 as separate sections by region, with sig" --data survey.sav

TABLE "Q1 and Q2 by region"
    STUBS $q1, $q2
    BANNER $region
    STATS col_pct, n, sig
END TABLE

✓ validates against survey.sav
```

### A derive

```
$ mrscript assist "derive age groups: under 35, 35 to 54, 55 and over" --data survey.sav --type derive --out age_group.mrs

DERIVE @age_group
    STUB 1 "Under 35" WHERE $age < 35
    STUB 2 "35 to 54" WHERE $age >= 35 AND $age <= 54
    STUB 3 "55+" WHERE $age >= 55
END DERIVE

✓ validates against survey.sav
Wrote age_group.mrs
```

---

## Exit codes

| Situation | Exit |
|-----------|------|
| Snippet generated (valid **or** a draft that didn't validate) | 0 |
| No provider configured / data load failed / backend error | 1 |

An invalid draft still exits 0 — `assist` is an authoring aid, so it always hands
you something. Check the `⚠` line on stderr.

---

See also: [Tables](tables.md), [Data Diff](diff.md), [Tab-Plan
Reconciliation](reconcile.md).
