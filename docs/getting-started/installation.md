# Install & first run

This page gets `mrscript` working on your machine and walks you through running your
very first script from the command line.

!!! info "What you need"

    - **Python 3.11 or newer** — check with `python --version`.
    - A survey data file to play with (`.sav` or `.csv`). If you don't have one yet,
      any small SPSS export will do.

---

## Install

Tablix ships as two Python packages:

- **`mrscript-core`** — the engine + the `mrscript` command-line tool (no GUI).
- **`mrscript-desktop`** — the PySide6 desktop app (depends on the core).

=== "Just the engine + CLI"

    ```powershell
    # from the mrscript-core/ folder
    pip install -e ".[dev,output]"
    ```

    The `output` extra pulls in the Excel writer (openpyxl); `dev` adds the test
    tooling. After this, the `mrscript` command is available.

=== "The desktop app"

    ```powershell
    # from the mrscript-desktop/ folder
    pip install -e ".[dev]"

    # launch it
    python -m app.main
    # ...or, via the installed entry point:
    tablix
    ```

Verify the CLI is installed:

```powershell
mrscript --help
```

---

## Write your first script

Create a file called `hello.mrs` next to your data file:

```mrs
SOURCE 'survey.sav'              // ← change to your file name

TABLE 'My first table'
  STUBS  $q1                     // ← any single variable in your data
  STATS  n, col_pct
END TABLE
```

!!! tip "Don't know your variable names?"

    Open the file in the **desktop app** and look at the **Variable Explorer** — it
    lists every variable referenced in your script. Or load the data and browse the
    **Data** tab.

---

## Run it

```powershell
mrscript run hello.mrs
```

You'll see a plain-text table printed to the screen, something like:

```text
Table 1
My first table
Base: All respondents

                 Total
Total              105
  Very satisfied    47
                   45%
  Satisfied         38
                   36%
  Dissatisfied      20
                   19%
```

That's a complete tabulation job — one variable, counts and column percentages.

---

## Produce other formats

The CLI picks the output format from the **file extension** you give it:

```powershell
mrscript run         hello.mrs                   # print text tables to the screen
mrscript run         hello.mrs --data other.sav  # run, but use a different data file
mrscript export      hello.mrs report.csv        # tagged CSV (for external formatting)
mrscript export      hello.mrs report.xlsx       # styled Excel workbook
mrscript export      hello.mrs report.xlsx --theme blue --layout single_sheet
mrscript export-data hello.mrs clean.sav         # write the transformed DATA back out
mrscript project run project.yml                 # batch-run many scripts (YAML)
```

| Command | What it produces |
|---------|------------------|
| `mrscript run` | Plain-text tables, straight to the screen |
| `mrscript export … .csv` | Tagged CSV — every row labelled so Excel/macros can style it |
| `mrscript export … .xlsx` | A styled Excel workbook with an index sheet |
| `mrscript export-data … .sav` | The cleaned/transformed **dataset** (not tables) |
| `mrscript project run` | Runs a whole batch of scripts defined in a YAML file |

See [Output formats & the CLI](../reference/output.md) for every flag.

---

## Next: build something real

You now have the tool running. The [**tutorial**](../tutorial/first-crosstab.md)
takes you from this one-variable table to a proper banner crosstab with derived
variables, filters, and significance testing — one step at a time.
