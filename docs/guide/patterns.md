# How-to recipes

Task-oriented snippets for things DP users do all the time. Each is a complete,
copy-pasteable fragment — adapt the variable names to your data. For the full rules
behind any clause, follow the links into the [reference](../reference/index.md).

!!! tip "How to use this page"

    Find the task that matches what you need, copy the block, swap in your variable
    names. These are the same patterns you'll reach for again and again.

---

## Profile table (counts + %, masked small bases)

A clean sample-profile table with a low-base mask.

```mrs
FORMAT
  STATS      n, col_pct
  BASE_LABEL 'Respondents'
  MIN_BASE   20 MASK 'n<20'
END FORMAT

TABLE 'Sample profile'
  STUBS @age_group, $gender, $region
END TABLE
```

`STUBS a, b, c` stacks three sections in one table. See
[STUBS](../reference/tables.md#row-axis).

---

## Crosstab with significance

```mrs
TABLE 'Brushing frequency by age'
  STUBS  $q5
  BANNER @age_group
  STATS  n, col_pct, sig
END TABLE
```

Configure the test with [`CONFIG SIG_CONFIDENCE / SIG_COMPARE / SIG_CORRECTION`](../reference/setup-blocks.md#config).

---

## Clean, label and tabulate a coded CSV

CSV columns carry no labels, so label them yourself (or supply a `CODEBOOK`).

```mrs
SOURCE 'raw.csv' CODEBOOK 'book.json'
DROP ROWS WHERE $status != 1               // completes only
EDIT $age SET 999 WHERE $age > 120         // flag impossible ages
VARIABLE $age MISSING 999 END VARIABLE     // then exclude them

TABLE 'Age distribution'
  DISTRIBUTION $age
  STATS n, col_pct, mean
END TABLE
```

`DISTRIBUTION` builds rows from the **actual codes present** in the data — handy for
auditing. See [DISTRIBUTION](../reference/tables.md#row-axis).

---

## Top-2-Box via RECODE

When you just need a quick collapse (no NET sub-rows), `RECODE` is the fastest route.

```mrs
RECODE $q1 (1,2,3 -> 1 "Bottom 3") (4,5 -> 2 "Top 2") INTO @q1_t2b

TABLE 'Satisfaction (T2B)'
  STUBS  @q1_t2b
  BANNER @region
END TABLE
```

For a T2B that keeps its member rows visible, use a `NET` inside `DERIVE` (see the
[tutorial, Step 7](../tutorial/first-crosstab.md#step-7-add-net-rows-top-2-box)).

---

## Computed index

```mrs
COMPUTE @bmi = $weight / ($height * $height)

TABLE 'BMI'
  DISTRIBUTION @bmi
  STATS mean, std_dev
END TABLE
```

See [COMPUTE](../reference/data-preparation.md#compute).

---

## Combine waves, then tabulate (APPEND)

Stack the rows of two files into one dataset.

```mrs
SOURCE w1 'wave1.sav'
SOURCE w2 'wave2.sav'
APPEND w1, w2

TABLE 'Awareness (pooled)'
  STUBS  $aware
  BANNER $wave
END TABLE
```

---

## Add profiling variables by JOIN

Match a profile file onto your main file by respondent key.

```mrs
SOURCE main 'respondents.sav'
SOURCE prof 'profile.sav'
%RESPID = $respondent_id
JOIN main WITH prof ON %RESPID

TABLE 'Brand by income'
  STUBS  $brand
  BANNER $income
END TABLE
```

See [APPEND / JOIN](../reference/data-input.md#append--join).

---

## Statistics card (summary only)

Show just the mean/std/median rows, hiding the distribution.

```mrs
TABLE 'Q1 mean by segment'
  STUBS  $q1
  BANNER @age_group
  STATS + mean, std_dev, std_error, median
  STATS_ONLY
END TABLE
```

---

## Rotated concept test (STACK)

Reshape three rotated concept slots into one long frame, then tabulate at that level.

```mrs
STACK @trial
  POSITIONS 1..3
  MAP @product = $I_#_Product_Shown
  MAP @overall = $I_#_Q1
END STACK

TABLE 'Liking by product'
  LEVEL @trial  STUBS @overall  BANNER @product
END TABLE

TABLE 'Reach (unique respondents)'
  LEVEL @trial  STUBS @product  BASE respondents
END TABLE
```

See [STACK](../reference/data-preparation.md#stack).

---

## Clean and export a reproducible dataset

```mrs
SOURCE 'raw.sav'
DROP ROWS WHERE $speeder = 1
RECODE  $q1 (1,2 -> 1) (3,4,5 -> 2) INTO @q1_grp
COMPUTE @score = ($q1 + $q2 + $q3) / 3

EXPORT DATA 'clean.sav'                       // metadata-complete SPSS
EXPORT DATA 'clean.csv' CODEBOOK 'book.json'  // portable data + metadata
```

See [EXPORT DATA](../reference/output.md#export-data).

---

## Diary / nested-loop data (day × occasion)

```mrs
SOURCE 'diary.sav'
%RESPID = $RESPID

STACK @diary
  AXIS day      1..2
  AXIS occasion 1..2
  MAP @day      AT day      FROM $DAY_1, $DAY_2
  MAP @beverage AT occasion FROM $BEV_1_1, $BEV_1_2, $BEV_2_1, $BEV_2_2
  DROP EMPTY @beverage = NULL, 99            // remove unfilled occasions
END STACK

// @day_n, @occasion_n, @diary_slot auto-emitted

TABLE 'Beverages by day'
  STUBS @beverage  BANNER @day_n  LEVEL @diary
END TABLE

TABLE 'Beverage penetration (unique respondents)'
  STUBS @beverage  LEVEL @diary  BASE respondents
END TABLE
```

See [Multi-axis STACK](../reference/data-preparation.md#stack).

---

## Rating battery as a GRID

Several rating statements side-by-side as columns, each showing its own
distribution + NETs + mean.

```mrs
DERIVE @s1
  NET 'Top 2 Box'    STUB 5 'Strongly agree' WHERE $s1 = 5
                     STUB 4 'Agree'          WHERE $s1 = 4   ENDNET
  STUB 3 'Neutral'   WHERE $s1 = 3
  NET 'Bottom 2 Box' STUB 2 'Disagree'          WHERE $s1 = 2
                     STUB 1 'Strongly disagree' WHERE $s1 = 1  ENDNET
  SCORE 1=1 SCORE 2=2 SCORE 3=3 SCORE 4=4 SCORE 5=5
END DERIVE
// ... @s2, @s3 defined the same way ...

TABLE 'Agreement battery' TYPE GRID
  COLUMN @s1 LABEL 'Statement 1'
  COLUMN @s2 LABEL 'Statement 2'
  COLUMN @s3 LABEL 'Statement 3'
  STATS col_pct, mean
END TABLE
```

See [GRID tables](../reference/tables.md#grid-tables).

---

## Pool two questions into one section (ADD)

When two questions share a code frame and you want their responses to **add up**.

```mrs
TABLE 'Total brand awareness (Q1 + Q2)'
  ADD    @q1_brand, @q2_brand SECTION LABEL 'Any mention'
  BANNER $gender
  STATS  col_pct, n, sig
END TABLE
```

See [ADD](../reference/tables.md#row-axis).
