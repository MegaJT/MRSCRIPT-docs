# 14 · Recode Suggester

**`mrscript suggest`** reads a variable's **value labels** and proposes sensible
**NET / recode groupings**, emitting them as a **validated `DERIVE` block** you paste
into your script. It is the codebook-driven cousin of [`assist`](assist.md): where
`assist` takes a free-text request, `suggest` takes a *variable* and reads its
labels — you don't describe the grouping, it proposes one.

```
$ mrscript suggest $q1 --data survey.sav

DERIVE @q1_grp
    NET "Agree (Top 2 Box)"
        STUB 4 WHERE $q1 = 4
        STUB 5 WHERE $q1 = 5
    ENDNET
    STUB 3 "Neutral" WHERE $q1 = 3
    NET "Disagree (Bottom 2 Box)"
        STUB 1 WHERE $q1 = 1
        STUB 2 WHERE $q1 = 2
    ENDNET
END DERIVE

✓ validates against survey.sav  (detected: scale)
```

The block goes to **stdout** (clean for piping into your editor); the validation +
detected-kind line goes to **stderr**. With `--out FILE` it is also written to a file.

---

## Configuration — shared with `assist`

`suggest` calls an AI model through the **exact same provider layer as
[`assist`](assist.md#it-is-optional-and-configurable)** — there are **no new
environment variables**. Set `ANTHROPIC_API_KEY` (Anthropic, the default), or
`MRSCRIPT_ASSIST_URL` (+ `MRSCRIPT_ASSIST_MODEL`) for a **local / OpenAI-compatible**
model (Ollama, LM Studio — no key, no cost). With no provider configured it prints a
clear error and exits — the rest of `mrscript` is unaffected.

See the [NL Assistant configuration table](assist.md#it-is-optional-and-configurable)
for the full env-var reference.

---

## Scale vs nominal — what it produces

`suggest` looks at the variable's labels and decides whether it reads as an **ordered
rating scale** or a **nominal list**, then proposes the appropriate grouping. The
status line tells you which it picked (`detected: scale` / `detected: nominal`).

| Kind | Looks like | Proposes |
|------|-----------|----------|
| **scale** | An ordered rating battery item — contiguous codes (3–11 points), numeric or with ordered-scale labels (*agree/disagree*, *satisfied*, *likely* …) | **Top-box / middle / bottom-box NETs** — Top-2-Box / Neutral / Bottom-2-Box for a 5-point, Top-3 / middle / Bottom-3 for a 7-point. |
| **nominal** | An unordered list — brands, categories, regions | **Thematic NETs** grouping related labels ("Any premium", "Any value"), with leaf `STUB`s for items that don't group. |

Detection is **conservative**: when the scale signal is weak it falls back to
**nominal** (a thematic net on a mild scale is at worst suboptimal, but a top-box on a
genuine brand list is nonsense). Force the framing with **`--kind scale|nominal`** —
useful for an ordinal coded variable whose labels don't spell out the scale.

---

## Full option reference

```
mrscript suggest $var --data FILE [OPTIONS]

Arguments:
  $var             Source variable to recode (accepts $var or bare var)

Options:
  --data FILE      Dataset (.sav/.csv) — supplies the variable's value labels and
                   validates the generated DERIVE   (REQUIRED)
  --kind scale|nominal   Override the auto-detected variable kind
  --name @target   Name for the derived variable (default @<var>_grp)
  --model NAME     Model override (else MRSCRIPT_ASSIST_MODEL / the default)
  --out FILE       Also write the generated snippet to FILE
```

`--data` is **required** — without value labels there is nothing to group, and the
dataset is also what validates the output.

---

## What it can recode (v1 scope)

`suggest` recodes a **labelled categorical source (`$`) variable** —
`single_punch`, or a `numeric` variable carrying value labels — with **at least two
labels**. It always emits a **NET-grouping `DERIVE`**.

It will **refuse** (exit 1, with a message) an `open_end` variable, a weight, a
derived (`@`) variable, or a variable with fewer than two value labels. For **range
bands on an unlabelled numeric** (age → under 35 / 35–54 / 55+), use
[`assist`](assist.md) instead. Mutually-exclusive `RECODE … INTO` transforms and
scored recodes are out of scope for v1.

---

## Validation

Every generated DERIVE is run through the **real MRScript parser and validator**
against your dataset before you see it: `$var` must exist, stub codes must be unique,
and the NET structure must be well-formed. If it fails, the suggester feeds the
errors back to the model and tries once more (max 2 rounds). After that it shows the
best draft **with the errors** (and still exits 0), so you get something to fix
rather than nothing.

---

## Worked examples

### A rating scale → top/bottom-box

```
$ mrscript suggest $satisfaction --data survey.sav

DERIVE @satisfaction_grp
    NET "Satisfied (Top 2 Box)"
        STUB 4 WHERE $satisfaction = 4
        STUB 5 WHERE $satisfaction = 5
    ENDNET
    STUB 3 "Neutral" WHERE $satisfaction = 3
    NET "Dissatisfied (Bottom 2 Box)"
        STUB 1 WHERE $satisfaction = 1
        STUB 2 WHERE $satisfaction = 2
    ENDNET
END DERIVE

✓ validates against survey.sav  (detected: scale)
```

### A brand list → thematic nets

```
$ mrscript suggest $brand --data survey.sav --name @brand_tier --out brand_tier.mrs

DERIVE @brand_tier
    NET "Any premium"
        STUB 1 WHERE $brand = 1
        STUB 2 WHERE $brand = 2
    ENDNET
    NET "Any value"
        STUB 3 WHERE $brand = 3
        STUB 4 WHERE $brand = 4
    ENDNET
    STUB 5 "Other" WHERE $brand = 5
END DERIVE

✓ validates against survey.sav  (detected: nominal)
Wrote brand_tier.mrs
```

The member `STUB`s omit their labels — they are **inherited from the codebook** (the
`$brand = 1` row picks up "Lux"), so the table renders the real labels while the
DERIVE stays compact.

---

## Exit codes

| Situation | Exit |
|-----------|------|
| Snippet generated (valid **or** a draft that didn't validate) | 0 |
| No provider configured / data load failed / variable not recodable / backend error | 1 |

An invalid draft still exits 0 — `suggest` is an authoring aid, so it always hands
you something. Check the `⚠` line on stderr.

---

## Python API

```python
from mrscript.assist import AssistConfig, Suggester
from mrscript.variable_engine.registry import load_data

_df, store = load_data("survey.sav")
result = Suggester(AssistConfig.from_env()).suggest("$q1", store, kind=None, name=None)

print(result.mrscript)          # the DERIVE block (best attempt)
print(result.valid, result.kind)  # True, "scale"
```

`Suggester(config, backend=…)` accepts an injectable `LLMBackend` (tests pass a fake
one, so no network / key is touched).

---

See also: [NL Assistant](assist.md), [Tables](tables.md), [Data
Preparation](data-preparation.md).
