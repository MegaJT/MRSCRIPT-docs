---
title: Tablix & MRScript
hide:
  - navigation
---

# Tablix & MRScript

**Tablix** turns survey data (SPSS `.sav`, CSV, and more) into professional
crosstabulations and reports. You drive it with **MRScript** — a short, readable
script language built for the way DP (data processing) specialists actually think:
declare your data, clean it, build your tables, done.

If you have ever used Quantum, MRDCL, or wrestled with SPSS syntax, MRScript will
feel familiar — but it reads in plain words and one script reproduces your whole job.

```mrs
SOURCE 'survey.sav'
%RESPID = $respondent_id

TABLE 'Overall satisfaction by age'
  STUBS  $q1
  BANNER $age_group
  STATS  n, col_pct, sig
END TABLE
```

That is a complete job: load the data, cross *satisfaction* by *age group*, show
counts, column percentages, and significance letters.

---

## New here? Start with these

<div class="grid cards" markdown>

-   :material-rocket-launch: **What is Tablix?**

    ---

    The five-minute picture: what the tool does, the pieces of a script, and the
    `$` / `@` / `%` naming convention you will see everywhere.

    [:octicons-arrow-right-24: Read the overview](getting-started/what-is-tablix.md)

-   :material-download: **Install & first run**

    ---

    Get `mrscript` on your machine, run your first script, and produce text and
    Excel output from the command line.

    [:octicons-arrow-right-24: Installation](getting-started/installation.md)

-   :material-school: **Your first crosstab**

    ---

    A guided, build-it-yourself walkthrough — from a raw data file to a
    significance-tested cross-tab, one clause at a time.

    [:octicons-arrow-right-24: Start the tutorial](tutorial/first-crosstab.md)

-   :material-book-open-variant: **Language reference**

    ---

    The complete, exhaustive specification of every statement, clause, and
    statistic — the page you keep open while you script.

    [:octicons-arrow-right-24: Browse the reference](reference/index.md)

</div>

---

## What can you do with it?

<div class="grid cards" markdown>

-   :material-table: **Crosstabs & banners**

    Cross any variable by any banner, nest columns (`$city BY $gender`), add a
    Total column, and letter significant differences automatically.

-   :material-broom: **Clean & transform data**

    `EDIT`, `RECODE`, `KEEP/DROP ROWS`, and `COMPUTE` — declarative, replayable
    edits. Re-running the script reproduces the clean dataset exactly.

-   :material-shuffle-variant: **Reshape for concept tests & diaries**

    `STACK` wide repeated-measures columns into a long frame for concept rotations,
    rating batteries, and day × daypart × occasion diaries.

-   :material-sigma: **Real statistics**

    Means, std dev, quartiles, NET (Top-2-Box) rows, weighted bases (Kish), and
    column-proportion / column-mean significance with confidence levels.

-   :material-file-export: **Multiple outputs**

    Plain text (Quantum/MRDCL style), tagged CSV for external formatting, and
    styled Excel workbooks — all from one script.

-   :material-database-arrow-right: **Export clean data**

    Write the transformed dataset back out to `.sav` / `.csv` / `.parquet` with a
    full codebook — closing the load → clean → reproduce loop.

</div>

---

!!! tip "How to read these docs"

    - **Just starting?** Go top-to-bottom: *Getting Started* → *Tutorial*.
    - **Need to do a specific thing?** Jump to **How-to Recipes**.
    - **Need the exact rule for a clause?** Use the **Language Reference** (or the
      :material-magnify: search box — it indexes every page).

    Throughout, code looks `like this`, and **complete, runnable script blocks**
    appear in coloured boxes with a copy button in the top-right corner.
