# Install & first run

Tablix is a normal Windows desktop application. **You do not need Python or any
technical setup** — everything it needs is bundled inside the installer. Download,
install, open, and you're ready to tabulate.

!!! info "What you need"

    - **Windows 10 or 11.**
    - The **Tablix installer** (`Tablix-Setup.exe`) — see *Get the installer* below.
    - A survey data file to work with (`.sav` or `.csv`).

    You do **not** need Python, Excel, SPSS, or any other software installed.

---

## Get the installer

Tablix is distributed as a **free 14-day trial**. Request it from the product site —
the download link is emailed to you:

[:material-download: Get Tablix (start free trial){ .md-button .md-button--primary }](https://mrtabulate.com)

<!-- TODO: confirm the exact download URL / trial flow and update this link. -->

You'll receive a single file: **`Tablix-<version>-Setup.exe`** (where `<version>` is
the release number, e.g. `Tablix-1.0.0-Setup.exe`).

---

## Install it

1. **Double-click** the downloaded `Tablix-<version>-Setup.exe`.
2. Windows may show a blue **"Windows protected your PC"** screen. This appears
   because the installer isn't yet code-signed — it is safe to continue:
    - Click **More info**
    - Then click **Run anyway**
3. Follow the installer (Next → Next → Install). It installs in a minute or two.
4. When it finishes, launch **Tablix** from the **Start Menu** (or the desktop
   shortcut).

!!! note "Why the security warning?"

    The current build is unsigned, so Windows SmartScreen flags the publisher as
    unknown. That's expected for now — "More info → Run anyway" is the correct
    response. (A signed build will remove this warning in a future release.)

---

## The Tablix window at a glance

When Tablix opens you'll see:

| Area | What it's for |
|------|---------------|
| **Toolbar** | `Run ▶` · `Stop ■` · `Open Script` · `Open Data` · `Open Output Folder` |
| **Script tab** | Where you write or open your MRScript (`.mrs`) |
| **Data tab** | A grid showing the loaded data file |
| **Output tab** | The tables your script produces |
| **Variable Explorer** | Lists the variables your script uses (source `$` in blue, derived `@` in green) |
| **Problems panel** (bottom) | Any errors or warnings, updated live as you type |

---

## Your first run, inside the app

1. **Open your data:** click **Open Data** on the toolbar and pick your `.sav` or
   `.csv` file. The **Data tab** fills with the rows so you can see it loaded.
2. **Write a script:** click the **Script tab** and type a small script (or click
   **Open Script** to load an existing `.mrs`):

    ```mrs
    SOURCE 'survey.sav'              // ← the file you just opened

    TABLE 'My first table'
      STUBS  $q1                     // ← any single variable in your data
      STATS  n, col_pct
    END TABLE
    ```

3. **Run it:** click **Run ▶** on the toolbar.
4. **See the result:** the **Output tab** shows your table — counts and column
   percentages for `$q1`.
5. **Find the saved output:** click **Open Output Folder** to open the folder where
   Tablix wrote the output file.

!!! tip "Don't know your variable names?"

    Look at the **Variable Explorer** (it lists the variables in your script), or
    browse the loaded file in the **Data tab**. Expanding a variable shows its
    categories and a quick frequency.

That's a complete tabulation. The [tutorial](../tutorial/first-crosstab.md) builds
from here to a full banner crosstab with derived variables, filters, weighting, and
significance testing.

---

## Where Tablix stores things

| Item | Location |
|------|----------|
| Output files | the folder shown by **Open Output Folder** |
| Log file (for troubleshooting) | `%LOCALAPPDATA%\Tablix\logs\tablix.log` |

If something goes wrong, the **Problems panel** at the bottom of the window explains
it in plain language; the log file has the technical detail if you need to send it to
support (`hello@mrtabulate.com`).

---

## Advanced: command-line & developer install

!!! warning "You only need this section if you are a developer or want to automate runs"

    Everyday DP users can ignore everything below — the desktop app does it all.

Tablix also ships an engine with a command-line tool, `mrscript`, useful for batch
jobs, automation, and CI. This path **does** require Python 3.11+.

```powershell
# install the engine + CLI (from the mrscript-core/ source folder)
pip install -e ".[dev,output]"
```

Then drive it from the command line — the format is chosen by the output file's
extension:

```powershell
mrscript run         script.mrs                   # print text tables to the screen
mrscript run         script.mrs --data other.sav  # run, but use a different data file
mrscript export      script.mrs report.csv        # tagged CSV (for external formatting)
mrscript export      script.mrs report.xlsx       # styled Excel workbook
mrscript export-data script.mrs clean.sav         # write the transformed DATA back out
mrscript project run project.yml                  # batch-run many scripts (YAML)
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

The [**tutorial**](../tutorial/first-crosstab.md) takes you from a one-variable table
to a proper banner crosstab with derived variables, filters, and significance
testing — one step at a time.
