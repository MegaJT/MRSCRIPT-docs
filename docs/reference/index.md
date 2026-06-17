---
title: Language Reference
---

# MRScript language reference

The complete specification of MRScript — every statement, clause, and statistic.
This is the page you keep open while you script. If you are new, work through the
[Tutorial](../tutorial/first-crosstab.md) first; come here for the exact rules.

!!! info "Version"

    MRScript **2.2**. Every feature documented here is implemented unless explicitly
    marked **(staged)** or listed under [Planned features](quick-reference.md#planned-features).

---

## Contents

<div class="grid cards" markdown>

-   **[1 · Basics](basics.md)**

    Script structure, comments, the `$`/`@`/`%` naming convention, string quoting,
    the preprocessor (`SET` / `DEFINE` / `CALL`) and the `.mrst` authoring layer.

-   **[2 · Data input](data-input.md)**

    `SOURCE` (files, formats, codebooks), `%RESPID` (the respondent key), and
    `APPEND` / `JOIN` for combining files.

-   **[3 · Setup blocks](setup-blocks.md)**

    `CONFIG` (execution settings), `FORMAT` (display defaults), and `VARIABLE`
    (metadata overrides).

-   **[4 · Data preparation](data-preparation.md)**

    `DERIVE`, `STACK`, `EDIT`, `RECODE`, `KEEP/DROP ROWS`, `COMPUTE` — clean,
    recode, compute, and reshape, all in script order.

-   **[5 · Tables](tables.md)**

    `TABLE` and all its clauses (`STUBS`, `DISTRIBUTION`, `ADD`, `BANNER`, `GRID`,
    `STATS`…), named `BANNER`s, and `SCOPE`.

-   **[6 · Output](output.md)**

    `EXPORT DATA` (write the dataset back out) and the output formats + CLI.

-   **[7 · Data health](health.md)**

    `mrscript health` — pre-tabulation QC: respondent checks (duplicate IDs,
    out-of-range codes, dead records), quality signals (straightliners, speeders,
    flatline batteries), structural checks (broken filters), and `EXPECT`
    routing assertions. Text / JSON / CSV reports.

-   **[8 · Reference details](reference-details.md)**

    The complete `STATS` list, condition syntax, base computation, missing values,
    and `STATS` inherit/override/merge modes.

-   **[9 · Quick reference](quick-reference.md)**

    Every keyword at a glance, common patterns, and planned features.

-   **[10 · Data Diff](diff.md)**

    `mrscript diff` — compare two deliveries of the same dataset: schema changes,
    dropped/added respondents, per-cell value changes, and distribution shift
    summaries. Text / JSON / CSV reports with configurable severity thresholds.

-   **[11 · RIM Weighting](weighting.md)**

    `RIM @weight … END RIM` — iterative proportional fitting (raking) to align
    the sample to population targets. `DIMENSION` targets, convergence controls,
    `WEIGHT_CAP`, and `BASE_WEIGHT`. Produces a derived `@weight` column usable
    in any `TABLE WEIGHT`.

</div>

---

## The naming convention, in one table

You'll see these everywhere. Memorise them and most scripts read themselves.

| Prefix | Kind | Created by | Example |
|--------|------|-----------|---------|
| `$name` | **Source** variable | a column in the data file | `$gender`, `$q1` |
| `@name` | **Derived** variable | `DERIVE`, `STACK`, `COMPUTE`, `RECODE/EDIT … INTO` | `@age_group`, `@bmi` |
| `%NAME` | **Reserved** constant | built in (only `%RESPID` today) | `%RESPID` |

Names are case-sensitive; the `$`/`@` prefix is mandatory and part of the grammar.
