# Curated Column Structure Programming Reference (`col_struct_prog_curated.md`)

This curated document collects canonical code-rendered table pairs specifically isolating the **column structure**, deduplicated and pruned from the full 605-pair dataset (`col_struct_prog.md`).

> [!IMPORTANT]
> **Tabulation Model Rules:**
> - **Column structure** is implemented via zero or more calls to the `split_cols_by*` family of functions; nothing else impacts column structure.
> - **Row faceting structure** (up to but not including individual rows) is implemented via zero or more calls to the `split_rows_by*` family.
> - **Individual rows** are implemented via one or more calls to the `analyze*` family of functions.
> - **Facet labels** are converted into group summary rows via a single call to `summarize_row_groups` directly after the relevant `split_rows_by*` function call.

### Curation Summary:
- **Total curated pairs**: 232 (157 standard `rtables`, 75 requiring `junco`), curated from the original 605 pairs plus 32 newly implemented diverse domain archetypes, advanced structural proposals, and practical clinical/cross-domain variant layouts.
- **Code & Domain Coverage**: Complete coverage across both core clinical trial structures and advanced cross-domain archetypes (measure-first/co-primary endpoint hierarchies, longitudinal visits with conditional change-from-baseline, interleaved within-arm contrasts, cross-variable conditional subgroup/ADA splits, two-way hierarchical marginals, 3-tier demographic banners crossed with treatment arms, multi-population predicate spans crossed with arms, periodic safety updates [DSUR/PBRER/PSUR] and multi-horizon reporting, observational/propensity-score adjustment stages with balance diagnostics, multi-arm shift matrices with complete scales, monotonically contracting disposition/attrition funnels, per-parent data-driven quantiles, continuous cumulative and sliding windows, predicate-defined multi-variable subpopulation columns, duplicated-control comparison blocks, variable-depth sibling branches, data-ordered top-k with residual other bucket, declared complete/empty level sets with missing columns, pooled subtree replicas for integrated summaries, horizontally paginated wide headers, repeated measures/visits, ragged/asymmetric trees, cumulative escalation, multi-reference comparisons, crossover studies, market research banner tables, dual-strata survey weighting, corporate financial P&L variance, SaaS cohort retention, ML benchmark leaderboards, A/B/n experimentation with lift, and high-order factorial assays).
- **Boilerplate Pruning**: High-frequency boilerplate patterns (such as identical `basic_table()` or `split_cols_by("ARM")` calls) were pruned to 2–3 canonical instances exhibiting distinct topological or layout variations.
- **Header Normalization / Deduplication**: Redundant pairs that differed only by sample size counts `(N=xx)` or row label dash widths were collapsed.
- **Artifact Removal**: Non-table data-frame introspection headers (e.g. `make_col_df` path listings) were removed.

---

# Section 1: Standard `rtables` Column Structures (No `junco` required)

These pairs use pure `rtables` column splitting functions (`split_cols_by`, `split_cols_by_multivar`, `split_cols_by_cuts`, `split_cols_by_cutfun`, `split_cols_by_groups`, `add_colcounts`, etc.).

### 1. Control splitting with provided function (limited customization)

`id: advanced_usage#2` | Source: `vignettes/advanced_usage.md`

**Setup for reproducibility:**

```r
library(rtables)

d1 <- subset(ex_adsl, AGE < 25)

d1$AGE <- as.factor(d1$AGE)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("AGE") |>
  build_table(d1)
```

```text
20   21   23   24
————————————————————————————————————
```


### 2. Control splitting with provided function (limited customization)

`id: advanced_usage#3` | Source: `vignettes/advanced_usage.md`

**Setup for reproducibility:**

```r
library(rtables)

sd_cutfun <- function(x) {
    cutpoints <- c(min(x), mean(x) - sd(x), mean(x) + sd(x), max(x))
    names(cutpoints) <- c("", "Low", "Medium", "High")
    cutpoints
}
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_cutfun("AGE", cutfun = sd_cutfun) |>
  build_table(ex_adsl)
```

```text
Low   Medium   High
——————————————————————————————————————
```


### 3. Control splitting with provided function (limited customization)

`id: advanced_usage#4` | Source: `vignettes/advanced_usage.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_cuts("AGE", cuts = c(0, 30, 60, 100), cutlabels = c("0-30 y.o.", "30-60 y.o.", 
      "60-100 y.o.")) |>
  build_table(ex_adsl)
```

```text
0-30 y.o.   30-60 y.o.   60-100 y.o.
———————————————————————————————————————————————————————
```


### 4. Adding an Overall Column Only When The Split Would Already Define 2+ Facets

`id: advanced_usage#6` | Source: `vignettes/advanced_usage.md`

**Setup for reproducibility:**

```r
library(rtables)

picky_splitter <- function(var) {
    function(df, spl, vals, labels, trim) {
        orig_vals <- vals
        if (is.null(vals)) {
            vec <- df[[var]]
            vals <- if (is.factor(vec)) 
                levels(vec)
            else unique(vec)
        }
        if (length(vals) == 1) {
            do_base_split(spl = spl, df = df, vals = vals, labels = labels, trim = trim)
        }
        else {
            add_overall_level("Overall", label = "All Obs", first = FALSE)(df = df, 
                spl = spl, vals = orig_vals, trim = trim)
        }
    }
}

d1 <- subset(ex_adsl, ARM == "A: Drug X")

d1$ARM <- factor(d1$ARM)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = picky_splitter("ARM")) |>
  build_table(d1)
```

```text
A: Drug X
————————————————
```


### 5. Adding an Overall Column Only When The Split Would Already Define 2+ Facets

`id: advanced_usage#7` | Source: `vignettes/advanced_usage.md`

**Setup for reproducibility:**

```r
library(rtables)

picky_splitter <- function(var) {
    function(df, spl, vals, labels, trim) {
        orig_vals <- vals
        if (is.null(vals)) {
            vec <- df[[var]]
            vals <- if (is.factor(vec)) 
                levels(vec)
            else unique(vec)
        }
        if (length(vals) == 1) {
            do_base_split(spl = spl, df = df, vals = vals, labels = labels, trim = trim)
        }
        else {
            add_overall_level("Overall", label = "All Obs", first = FALSE)(df = df, 
                spl = spl, vals = orig_vals, trim = trim)
        }
    }
}

d1 <- subset(ex_adsl, ARM == "A: Drug X")

d1$ARM <- factor(d1$ARM)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = picky_splitter("ARM")) |>
  build_table(ex_adsl)
```

```text
A: Drug X   B: Placebo   C: Combination   All Obs
————————————————————————————————————————————————————————
```


### 6. Different Formats For Different Values Within A Row-Split

`id: advanced_usage#8` | Source: `vignettes/advanced_usage.md`

**Setup for reproducibility:**

```r
library(rtables)

dta_test <- data.frame(USUBJID = rep(1:6, each = 3), PARAMCD = rep("lab", 6 * 3), 
    AVISIT = rep(paste0("V", 1:3), 6), ARM = rep(LETTERS[1:3], rep(6, 3)), AVAL = c(9:1, 
        rep(NA, 9)), CHG = c(1:9, rep(NA, 9)))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by_multivar(vars = c("AVAL", "CHG")) |>
  build_table(dta_test)
```

```text
A                         B                 C     
                AVAL           CHG         AVAL         CHG      AVAL   CHG
———————————————————————————————————————————————————————————————————————————
```


### 7. Creating an Example Table

`id: ard_how_to#2` | Source: `vignettes/ard_how_to.md`

**Setup for reproducibility:**

```r
library(rtables)

ADSL <- ex_adsl
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  build_table(ADSL)
```

```text
A: Drug X   B: Placebo   C: Combination
————————————————————————————————————————————————————————————
```


### 8. Introduction

`id: baseline#3` | Source: `vignettes/baseline.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", ref_group = "B: Placebo") |>
  build_table(DM)
```

```text
A: Drug X   B: Placebo   C: Combination
————————————————————————————————————————————————————————————————
```


### 9. Row Splitting

`id: baseline#6` | Source: `vignettes/baseline.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM", ref_group = "B: Placebo") |>
  build_table(subset(DM, SEX %in% c("M", "F")))
```

```text
A: Drug X   B: Placebo   C: Combination
                                (N=121)     (N=106)        (N=129)    
——————————————————————————————————————————————————————————————————————
```


### 10. Demographic Table

`id: clinical_trials#8` | Source: `vignettes/clinical_trials.md`

**Setup for reproducibility:**

```r
library(rtables)

ADSL <- ex_adsl
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(var = "ARM") |>
  build_table(ADSL)
```

```text
A: Drug X      B: Placebo     C: Combination
———————————————————————————————————————————————————————————————————
```


### 11. Demographic Table

`id: clinical_trials#11` | Source: `vignettes/clinical_trials.md`

**Setup for reproducibility:**

```r
library(rtables)

ADSL <- ex_adsl
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by(var = "ARMCD") |>
  build_table(ADSL)
```

```text
ARM A           ARM B           ARM C    
                        (N=134)         (N=134)         (N=132)   
——————————————————————————————————————————————————————————————————
```


### 12. Variations on the Demographic Table

`id: clinical_trials#12` | Source: `vignettes/clinical_trials.md`

**Setup for reproducibility:**

```r
library(rtables)

ADSL <- ex_adsl
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by(var = "ARM") |>
  build_table(ADSL)
```

```text
A: Drug X      B: Placebo     C: Combination
                 (N=134)         (N=134)         (N=132)    
————————————————————————————————————————————————————————————
```


### 13. Using Layouts

`id: clinical_trials#27` | Source: `vignettes/clinical_trials.md`

**Setup for reproducibility:**

```r
library(rtables)

ADSL <- ex_adsl
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  build_table(ADSL)
```

```text
A: Drug X      B: Placebo     C: Combination
                        (N=134)         (N=134)         (N=132)    
———————————————————————————————————————————————————————————————————
```


### 14. Adverse Events By ID

`id: clinical_trials#32` | Source: `vignettes/clinical_trials.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

lookup <- tribble(~AEDECOD, ~AEBODSYS, ~AETOXGR, "HEADACHE", "NERVOUS SYSTEM DISORDERS", 
    "5", "BACK PAIN", "MUSCULOSKELETAL AND CONNECTIVE TISSUE DISORDERS", "2", "GINGIVAL BLEEDING", 
    "GASTROINTESTINAL DISORDERS", "1", "HYPOTENSION", "VASCULAR DISORDERS", "3", 
    "FAECES SOFT", "GASTROINTESTINAL DISORDERS", "2", "ABDOMINAL DISCOMFORT", "GASTROINTESTINAL DISORDERS", 
    "1", "DIARRHEA", "GASTROINTESTINAL DISORDERS", "1", "ABDOMINAL FULLNESS DUE TO GAS", 
    "GASTROINTESTINAL DISORDERS", "1", "NAUSEA (INTERMITTENT)", "GASTROINTESTINAL DISORDERS", 
    "2", "WEAKNESS", "MUSCULOSKELETAL AND CONNECTIVE TISSUE DISORDERS", "3", "ORTHOSTATIC HYPOTENSION", 
    "VASCULAR DISORDERS", "4")

normalize <- function(x) x/sum(x)

weightsA <- normalize(c(0.1, dlnorm(seq(0, 5, length.out = 25), meanlog = 3)))

weightsB <- normalize(c(0.2, dlnorm(seq(0, 5, length.out = 25))))

N_pop <- 300

ADSL2 <- data.frame(USUBJID = seq(1, N_pop, by = 1), ARM = sample(c("ARM A", "ARM B"), 
    N_pop, TRUE), SEX = sample(c("F", "M"), N_pop, TRUE), AGE = 20 + rbinom(N_pop, 
    size = 40, prob = 0.7))

l.adae <- mapply(ADSL2$USUBJID, ADSL2$ARM, ADSL2$SEX, ADSL2$AGE, FUN = function(id, 
    arm, sex, age) {
    n_ae <- sample(0:25, 1, prob = if (arm == "ARM A") 
        weightsA
    else weightsB)
    i <- sample(seq_len(nrow(lookup)), size = n_ae, replace = TRUE, prob = c(6, rep(1, 
        10))/16)
    mutate(lookup[i, ], AESEQ = seq_len(n()), USUBJID = id, ARM = arm, SEX = sex, 
        AGE = age)
}, SIMPLIFY = FALSE)

ADAE2 <- do.call(rbind, l.adae)

ADAE2 <- select(mutate(ADAE2, ARM = factor(ARM, levels = c("ARM A", "ARM B")), AEDECOD = as.factor(AEDECOD), 
    AEBODSYS = as.factor(AEBODSYS), AETOXGR = factor(AETOXGR, levels = as.character(1:5))), 
    USUBJID, ARM, AGE, SEX, AESEQ, AEDECOD, AEBODSYS, AETOXGR)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  build_table(ADAE2)
```

```text
ARM A         ARM B    
                                                    (N=2060)       (N=1058)  
—————————————————————————————————————————————————————————————————————————————
```


### 15. Response Table

`id: clinical_trials#42` | Source: `vignettes/clinical_trials.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

ADRS_BESRSPI <- mutate(filter(ex_adrs, PARAMCD == "BESRSPI"), rsp = factor(AVALC %in% 
    c("CR", "PR"), levels = c(TRUE, FALSE), labels = c("Responders", "Non-Responders")), 
    is_rsp = (rsp == "Responders"))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARMCD", ref_group = "ARM A") |>
  build_table(ADRS_BESRSPI)
```

```text
ARM A            ARM B             ARM C     
                     (N=134)          (N=134)           (N=132)    
———————————————————————————————————————————————————————————————————
```


### 16. Time to Event Analysis Table

`id: clinical_trials#54` | Source: `vignettes/clinical_trials.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

ADSL <- ex_adsl
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM", ref_group = "A: Drug X") |>
  build_table(ADSL)
```

```text
A: Drug X      B: Placebo     C: Combination
                                             (N=134)         (N=134)         (N=132)    
————————————————————————————————————————————————————————————————————————————————————————
```


### 17. Setting column counts to visible at Layout time

`id: col_counts#1` | Source: `vignettes/col_counts.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  build_table(ex_adsl)
```

```text
A: Drug X        B: Placebo       C: Combination  
         F        M        F        M         F         M   
       (N=79)   (N=51)   (N=77)   (N=55)   (N=66)    (N=60) 
————————————————————————————————————————————————————————————
```


### 18. Hiding counts

`id: col_counts#3` | Source: `vignettes/col_counts.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
tbl <- basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  build_table(ex_adsl)
col_counts(tbl) <- c(17, 18, NA, 17, 18, 19)
tbl
```

```text
A: Drug X        B: Placebo      C: Combination  
         F        M        F       M         F         M   
       (N=17)   (N=18)           (N=17)   (N=18)    (N=19) 
———————————————————————————————————————————————————————————
```


### 19. Setting Column Counts to Visible at Layout Time

`id: col_counts#4` | Source: `vignettes/col_counts.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M")), show_colcounts = TRUE) |>
  build_table(ex_adsl)
```

```text
A: Drug X        B: Placebo       C: Combination  
         F        M        F        M         F         M   
       (N=79)   (N=51)   (N=77)   (N=55)   (N=66)    (N=60) 
————————————————————————————————————————————————————————————
```


### 20. Setting Column Counts to Visible at Layout Time

`id: col_counts#5` | Source: `vignettes/col_counts.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  build_table(ex_adsl)
```

```text
A: Drug X      B: Placebo      C: Combination  
          (N=134)         (N=134)           (N=132)     
         F       M       F       M        F         M   
————————————————————————————————————————————————————————
```


### 21. Setting Col Count Visibility

`id: col_counts#7` | Source: `vignettes/col_counts.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
tbl <- basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  build_table(ex_adsl)
facet_colcount(tbl, c("ARM", "C: Combination")) <- 75
facet_colcounts_visible(tbl, c("ARM", "A: Drug X", "SEX")) <- TRUE
tbl
```

```text
A: Drug X                                       
           (N=134)        B: Placebo      C: Combination  
         F        M         (N=134)           (N=75)      
       (N=79)   (N=51)     F       M        F         M   
——————————————————————————————————————————————————————————
```


### 22. Advanced Settings

`id: col_counts#9` | Source: `vignettes/col_counts.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
tbl <- basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  build_table(ex_adsl)
facet_colcount(tbl, c("ARM", "C: Combination")) <- 75
facet_colcounts_visible(tbl, c("ARM", "A: Drug X", "SEX")) <- TRUE
coldf <- make_col_df(tbl)
facet_colcount(tbl, coldf$path[[1]][c(1, 2)]) <- NA_integer_
tbl
```

```text
A: Drug X                                       
                          B: Placebo      C: Combination  
         F        M         (N=134)           (N=75)      
       (N=79)   (N=51)     F       M        F         M   
——————————————————————————————————————————————————————————
```


### 23. Rows and cell values alignments

`id: custom_appearance#3` | Source: `vignettes/custom_appearance.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  build_table(DM)
```

```text
all obs
————————————————
```


### 24. Top-left Materials

`id: custom_appearance#6` | Source: `vignettes/custom_appearance.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  build_table(DM)
```

```text
New top_left material here   A: Drug X   B: Placebo   C: Combination
————————————————————————————————————————————————————————————————————
```


### 25. Top-left Materials

`id: custom_appearance#7` | Source: `vignettes/custom_appearance.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  build_table(DM)
```

```text
STRATA1                                                     
  SEX                A: Drug X   B: Placebo   C: Combination
————————————————————————————————————————————————————————————
```


### 26. Table Inset

`id: custom_appearance#9` | Source: `vignettes/custom_appearance.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(inset = 5) |>
  build_table(DM)
```

```text
all obs
     ——————————————
```


### 27. Table Inset

`id: custom_appearance#12` | Source: `vignettes/custom_appearance.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "Title says Whaaaat", subtitles = "Oh, ok.", main_footer = "ha HA! Footer!", 
    prov_footer = "provenaaaaance") |>
  split_cols_by("ARM") |>
  build_table(ex_adsl)
```

```text
A: Drug X   B: Placebo   C: Combination
——————————————————————————————————————————————————
```


### 28. Section Dividers

`id: custom_appearance#17` | Source: `vignettes/custom_appearance.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("Species") |>
  build_table(iris)
```

```text
setosa      versicolor     virginica 
——————————————————————————————————————————————————————
```


### 29. Indent Modifier

`id: custom_appearance#20` | Source: `vignettes/custom_appearance.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "Study XXXXXXXX", subtitles = c("subtitle YYYYYYYYYY", "subtitle2 ZZZZZZZZZ"), 
    main_footer = "Analysis was done using cool methods that are correct", prov_footer = "file: /path/to/stuff/that/lives/there HASH:1ac41b242a") |>
  split_cols_by("ARM") |>
  build_table(DM)
```

```text
A: Drug X   B: Placebo   C: Combination
——————————————————————————————————————————————————————————
```


### 30. Variable Label Visibility

`id: custom_appearance#25` | Source: `vignettes/custom_appearance.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "Study XXXXXXXX", subtitles = c("subtitle YYYYYYYYYY", "subtitle2 ZZZZZZZZZ"), 
    main_footer = "Analysis was done using cool methods that are correct", prov_footer = "file: /path/to/stuff/that/lives/there HASH:1ac41b242a") |>
  split_cols_by("ARM") |>
  build_table(DM)
```

```text
SEX        A: Drug X   B: Placebo   C: Combination
——————————————————————————————————————————————————
```


### 31. Cell, Label, and Annotation Wrapping

`id: custom_appearance#26` | Source: `vignettes/custom_appearance.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

trimmed_data <- filter(filter(ex_adsl, SEX %in% c("M", "F")), RACE %in% levels(RACE)[1:2])

levels(trimmed_data$ARM)[1] <- "Incredibly long column name to be wrapped"

levels(trimmed_data$ARM)[2] <- "This_column_name_should_be_split_somewhere"
```

**Core layout instructions:**

```r
basic_table(title = "Title that is too long and also needs to be wrapped to a smaller width", 
    subtitles = "Subtitle that is also long and also needs to be wrapped to a smaller width", 
    main_footer = "Footnote that is wider than expected for this table.", prov_footer = "Provenance footer material that is also wider than expected for this table.") |>
  split_cols_by("ARM") |>
  build_table(trimmed_data)
```

```text
Incredibly long column name to be wrapped            This_column_name_should_be_split_somewhere                         C: Combination                  
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 32. Different Analysis Functions For Different Columns

`id: guided_advanced_afuns_rowsverticalsection#2` | Source: `vignettes/guided_advanced_afuns_rowsverticalsection.md`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` to directly construct the grouped treatment spanning headers and pairwise risk difference columns. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

adsl <- ex_adsl

adae <- ex_adae

adsl <- create_colspan_var(adsl, non_active_grp = "B: Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Treatment", colspan_var = "trt_span", trt_var = "ARM")

adae <- create_colspan_var(adae, non_active_grp = "B: Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Treatment", colspan_var = "trt_span", trt_var = "ARM")

trt_map <- create_colspan_map(adsl, non_active_grp = "B: Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Treatment", colspan_var = "trt_span", trt_var = "ARM")
```

**Core layout instructions:**

```r
basic_table() |>
  grouped_cols_w_diffs(trt_map, diffs_label = "Risk Differences") |>
  build_table(adae, adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

adsl <- ex_adsl

adae <- ex_adae

adsl$trt_span <- ifelse(adsl$ARM == "B: Placebo", " ", "Active Treatment")

adae$trt_span <- ifelse(adae$ARM == "B: Placebo", " ", "Active Treatment")

adsl$rr_header <- "Risk Differences"

adae$rr_header <- "Risk Differences"

adsl$rr_label <- paste(adsl$ARM, "vs B: Placebo")

adae$rr_label <- paste(adae$ARM, "vs B: Placebo")
```

**Fallback core layout instructions:**

```r
basic_table() |>
  split_cols_by("trt_span", split_fun = trim_levels_in_group("ARM")) |>
  split_cols_by("ARM") |>
  split_cols_by("rr_header", nested = FALSE) |>
  split_cols_by("rr_label", split_fun = remove_split_levels("B: Placebo vs B: Placebo")) |>
  build_table(adae, adsl)
```

</details>

```text
Active Treatment                                        Risk Differences                   
         A: Drug X   C: Combination   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo
—————————————————————————————————————————————————————————————————————————————————————————————————————————
```

### 33. Placing Specific Facets First And/Or Last

`id: guided_advanced_split_funs_new_bbbs#8` | Source: `vignettes/guided_advanced_split_funs_new_bbbs.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("SEX") |>
  build_table(ex_adsl)
```

```text
F   M   U   UNDIFFERENTIATED
———————————————————————————————
```


### 34. Placing Specific Facets First And/Or Last

`id: guided_advanced_split_funs_new_bbbs#9` | Source: `vignettes/guided_advanced_split_funs_new_bbbs.md`

**Setup for reproducibility:**

```r
library(rtables)

put_facets_first_last <- function(first = NULL, last = NULL) {
    if (is.null(first) && is.null(last)) {
        stop("must speficify at least one facet to be placed first or last")
    }
    function(ret, spl, fulldf) {
        fac_names <- names(ret$values)
        all_speced <- c(first, last)
        if (!all(all_speced %in% fac_names)) {
            stop("Facet(s) []", paste(setdiff(all_speced, fac_names), collapse = ", "), 
                "] not found in incoming split result.")
        }
        tmpfun <- restrict_facets(c(first, setdiff(fac_names, all_speced), last), 
            op = "keep", reorder = TRUE)
        tmpfun(ret, spl, fulldf)
    }
}

fl_splfun <- make_split_fun(post = list(put_facets_first_last(first = "U", last = "UNDIFFERENTIATED")))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("SEX", split_fun = fl_splfun) |>
  build_table(ex_adsl)
```

```text
U   F   M   UNDIFFERENTIATED
———————————————————————————————
```


### 35. Pre-sorting Or Pre-pruning Facets Based On Data Sparsity

`id: guided_advanced_split_funs_new_bbbs#11` | Source: `vignettes/guided_advanced_split_funs_new_bbbs.md`

**Setup for reproducibility:**

```r
library(rtables)

presort_facets <- function(ret, spl, fulldf) {
    fac_names <- names(ret$values)
    fac_ns <- vapply(ret$datasplit, NROW, 1L)
    ord <- order(fac_ns, decreasing = TRUE)
    tmpfun <- restrict_facets(fac_names[ord], op = "keep", reorder = TRUE)
    tmpfun(ret, spl, fulldf)
}

presort_splfun <- make_split_fun(post = list(presort_facets))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("STRATA1", split_fun = presort_splfun) |>
  build_table(ex_adsl)
```

```text
C         B         A   
   (N=143)   (N=135)   (N=122)
——————————————————————————————
```


### 36. Pre-sorting Or Pre-pruning Facets Based On Data Sparsity

`id: guided_advanced_split_funs_new_bbbs#12` | Source: `vignettes/guided_advanced_split_funs_new_bbbs.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("SEX") |>
  build_table(ex_adsl)
```

```text
F         M        U     UNDIFFERENTIATED
   (N=222)   (N=166)   (N=9)        (N=3)      
———————————————————————————————————————————————
```


### 37. Pre-sorting Or Pre-pruning Facets Based On Data Sparsity

`id: guided_advanced_split_funs_new_bbbs#13` | Source: `vignettes/guided_advanced_split_funs_new_bbbs.md`

**Setup for reproducibility:**

```r
library(rtables)

drop_sparse_facets <- function(ncutoff = 5) {
    function(ret, spl, fulldf) {
        fac_names <- names(ret$values)
        fac_ns <- vapply(ret$datasplit, NROW, 1L)
        keep_inds <- which(fac_ns >= ncutoff)
        tmpfun <- restrict_facets(fac_names[keep_inds], op = "keep", reorder = FALSE)
        tmpfun(ret, spl, fulldf)
    }
}

preprune_splfun2 <- make_split_fun(post = list(drop_sparse_facets()))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("SEX", split_fun = preprune_splfun2) |>
  build_table(ex_adsl)
```

```text
F         M        U  
   (N=222)   (N=166)   (N=9)
————————————————————————————
```


### 38. Pre-sorting Or Pre-pruning Facets Based On Data Sparsity

`id: guided_advanced_split_funs_new_bbbs#14` | Source: `vignettes/guided_advanced_split_funs_new_bbbs.md`

**Setup for reproducibility:**

```r
library(rtables)

drop_sparse_facets <- function(ncutoff = 5) {
    function(ret, spl, fulldf) {
        fac_names <- names(ret$values)
        fac_ns <- vapply(ret$datasplit, NROW, 1L)
        keep_inds <- which(fac_ns >= ncutoff)
        tmpfun <- restrict_facets(fac_names[keep_inds], op = "keep", reorder = FALSE)
        tmpfun(ret, spl, fulldf)
    }
}

preprune_splfun3 <- make_split_fun(post = list(drop_sparse_facets(10)))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("SEX", split_fun = preprune_splfun3) |>
  build_table(ex_adsl)
```

```text
F         M   
   (N=222)   (N=166)
————————————————————
```


### 39. Creating Preprocessing BBBs

`id: guided_advanced_split_funs_new_bbbs#16` | Source: `vignettes/guided_advanced_split_funs_new_bbbs.md`

**Setup for reproducibility:**

```r
library(rtables)

trim_facets_to_map <- function(map = NULL) {
    function(df, spl, vals, labels, .spl_context) {
        if (is.null(map)) {
            return(df)
        }
        cur_outer_val <- tail(.spl_context$value, 1)
        inner_var <- names(map)[2]
        inner_vec <- df[[inner_var]]
        inner_keep <- map[map[[1]] == cur_outer_val, inner_var, drop = TRUE]
        df_out <- df[inner_vec %in% inner_keep, ]
        df_out[[inner_var]] <- factor(df_out[[inner_var]], levels = intersect(levels(inner_vec), 
            inner_keep))
        df_out
    }
}

map <- data.frame(ARM = c("A: Drug X", "B: Placebo"), STRATA1 = c("B", "A"))

map_splfun <- make_split_fun(pre = list(trim_facets_to_map(map)))

outer_splfun <- make_split_fun(post = list(restrict_facets("C: Combination", op = "exclude")))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = outer_splfun) |>
  split_cols_by("STRATA1", split_fun = map_splfun) |>
  build_table(ex_adsl)
```

```text
A: Drug X   B: Placebo
       B           A     
—————————————————————————
```


### 40. Creating Preprocessing BBBs

`id: guided_advanced_split_funs_new_bbbs#17` | Source: `vignettes/guided_advanced_split_funs_new_bbbs.md`

**Setup for reproducibility:**

```r
library(rtables)

map <- data.frame(ARM = c("A: Drug X", "B: Placebo"), STRATA1 = c("B", "A"))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = trim_levels_to_map(map)) |>
  split_cols_by("STRATA1") |>
  build_table(ex_adsl)
```

```text
A: Drug X   B: Placebo
       B           A     
—————————————————————————
```


### 41. Creating ‘Multi-Section’ Tables With `nested = FALSE`

`id: guided_intro_nesting#6` | Source: `vignettes/guided_intro_nesting.md`

**Setup for reproducibility:**

```r
library(rtables)

trim_adsl <- subset(ex_adsl, RACE %in% levels(ex_adsl$RACE)[1:3] & SEX %in% c("F", 
    "M"))

trim_adsl$RACE <- factor(trim_adsl$RACE)

trim_adsl$SEX <- factor(trim_adsl$SEX)
```

**Core layout instructions:**

```r
basic_table(top_level_section_div = "-") |>
  split_cols_by("ARM") |>
  build_table(trim_adsl)
```

```text
A: Drug X   B: Placebo   C: Combination
———————————————————————————————————————————————————————————————————
```


### 42. Introduction

`id: introspecting_tables#2` | Source: `vignettes/introspecting_tables.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARMCD", show_colcounts = TRUE, colcount_format = "N=xx") |>
  split_cols_by("STRATA2", show_colcounts = TRUE) |>
  add_overall_col("All") |>
  build_table(ex_adsl)
```

```text
ARM A                     ARM B                     ARM C                       
                 N=134                     N=134                     N=132                       
            S1           S2           S1           S2           S1           S2                  
          (N=73)       (N=61)       (N=67)       (N=67)       (N=56)       (N=76)         All    
—————————————————————————————————————————————————————————————————————————————————————————————————
```


### 43. Detailed Table Structure

`id: introspecting_tables#12` | Source: `vignettes/introspecting_tables.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARMCD", show_colcounts = TRUE, colcount_format = "N=xx") |>
  split_cols_by("STRATA2", show_colcounts = TRUE) |>
  add_overall_col("All") |>
  build_table(ex_adsl)
```

```text
rowname    node_class    path                   
————————————————————————————————————————————————
```


### 44. An Illustrative Example

`id: pathing#1` | Source: `vignettes/pathing.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = keep_split_levels(c("A: Drug X", "C: Combination"))) |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  add_overall_col("All") |>
  build_table(DM)
```

```text
A: Drug X              C: Combination                   
                 F            M            F            M            All    
————————————————————————————————————————————————————————————————————————————
```


### 45. Row Paths

`id: pathing#3` | Source: `vignettes/pathing.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = keep_split_levels(c("A: Drug X", "C: Combination"))) |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  add_overall_col("All") |>
  build_table(DM)
```

```text
rowname       node_class    path                                      
——————————————————————————————————————————————————————————————————————
```


### 46. Column Paths

`id: pathing#12` | Source: `vignettes/pathing.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = keep_split_levels(c("A: Drug X", "C: Combination"))) |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  add_overall_col("All") |>
  build_table(DM)
```

```text
A: Drug X       
                 F            M     
————————————————————————————————————
```


### 47. Uniqueness of Paths

`id: pathing#16` | Source: `vignettes/pathing.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  build_table(DM)
```

```text
rowname    node_class    path                          
———————————————————————————————————————————————————————
```


### 48. Adding Column Structure

`id: rtables#12` | Source: `vignettes/rtables.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

n <- 400

df <- mutate(tibble(arm = factor(sample(c("Arm A", "Arm B"), n, replace = TRUE), 
    levels = c("Arm A", "Arm B")), country = factor(sample(c("CAN", "USA"), n, replace = TRUE, 
    prob = c(0.55, 0.45)), levels = c("CAN", "USA")), gender = factor(sample(c("Female", 
    "Male"), n, replace = TRUE), levels = c("Female", "Male")), handed = factor(sample(c("Left", 
    "Right"), n, prob = c(0.6, 0.4), replace = TRUE), levels = c("Left", "Right")), 
    age = rchisq(n, 30) + 10), weight = 35 * rnorm(n, sd = 0.5) + ifelse(gender == 
    "Female", 140, 180))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("arm") |>
  build_table(df)
```

```text
Arm A   Arm B
————————————————————
```


### 49. Adding Column Structure

`id: rtables#13` | Source: `vignettes/rtables.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

n <- 400

df <- mutate(tibble(arm = factor(sample(c("Arm A", "Arm B"), n, replace = TRUE), 
    levels = c("Arm A", "Arm B")), country = factor(sample(c("CAN", "USA"), n, replace = TRUE, 
    prob = c(0.55, 0.45)), levels = c("CAN", "USA")), gender = factor(sample(c("Female", 
    "Male"), n, replace = TRUE), levels = c("Female", "Male")), handed = factor(sample(c("Left", 
    "Right"), n, prob = c(0.6, 0.4), replace = TRUE), levels = c("Left", "Right")), 
    age = rchisq(n, 30) + 10), weight = 35 * rnorm(n, sd = 0.5) + ifelse(gender == 
    "Female", 140, 180))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("arm") |>
  split_cols_by("gender") |>
  build_table(df)
```

```text
Arm A           Arm B    
       Female   Male   Female   Male
————————————————————————————————————
```


### 50. Adding Row Structure

`id: rtables#16` | Source: `vignettes/rtables.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

n <- 400

df <- mutate(tibble(arm = factor(sample(c("Arm A", "Arm B"), n, replace = TRUE), 
    levels = c("Arm A", "Arm B")), country = factor(sample(c("CAN", "USA"), n, replace = TRUE, 
    prob = c(0.55, 0.45)), levels = c("CAN", "USA")), gender = factor(sample(c("Female", 
    "Male"), n, replace = TRUE), levels = c("Female", "Male")), handed = factor(sample(c("Left", 
    "Right"), n, prob = c(0.6, 0.4), replace = TRUE), levels = c("Left", "Right")), 
    age = rchisq(n, 30) + 10), weight = 35 * rnorm(n, sd = 0.5) + ifelse(gender == 
    "Female", 140, 180))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("arm") |>
  split_cols_by("gender") |>
  build_table(df)
```

```text
country: CAN

————————————————————————————————————————
```


### 51. A Table In Need of Attention

`id: sorting_pruning#4` | Source: `vignettes/sorting_pruning.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX") |>
  build_table(DM)
```

```text
A: Drug X                                              B: Placebo                                           C: Combination                   
                                                F            M           U      UNDIFFERENTIATED       F            M           U      UNDIFFERENTIATED       F            M           U      UNDIFFERENTIATED
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 52. Trimming Columns

`id: sorting_pruning#6` | Source: `vignettes/sorting_pruning.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX") |>
  build_table(DM)
```

```text
A: Drug X                B: Placebo              C: Combination     
                                F            M            F            M            F            M     
———————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 53. Trimming Columns

`id: sorting_pruning#8` | Source: `vignettes/sorting_pruning.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX") |>
  build_table(DM)
```

```text
rowname                      node_class    path                                                                
———————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 54. Sorting Within an Analysis Subtable

`id: sorting_pruning#28` | Source: `vignettes/sorting_pruning.md`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = drop_and_remove_levels(c("U", "UNDIFFERENTIATED"))) |>
  build_table(DM)
```

```text
A: Drug X      B: Placebo      C: Combination  
           F      M       F       M        F         M   
—————————————————————————————————————————————————————————
```


### 55. Path Based Cell Value Accessing:

`id: subsetting_tables#26` | Source: `vignettes/subsetting_tables.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = drop_split_levels) |>
  build_table(filter(ex_adsl, SEX %in% c("M", "F") & RACE %in% (levels(RACE)[1:3])))
```

```text
A: Drug X                B: Placebo              C: Combination     
                                F            M            F            M            F            M     
———————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 56. Path Based Cell Value Accessing:

`id: subsetting_tables#28` | Source: `vignettes/subsetting_tables.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = drop_split_levels) |>
  build_table(filter(ex_adsl, SEX %in% c("M", "F") & RACE %in% (levels(RACE)[1:3])))
```

```text
rowname                      node_class    path                                                                
———————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 57. Path Based Cell Value Accessing:

`id: subsetting_tables#29` | Source: `vignettes/subsetting_tables.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = drop_split_levels) |>
  build_table(filter(ex_adsl, SEX %in% c("M", "F") & RACE %in% (levels(RACE)[1:3])))
```

```text
C: Combination     
                F            M     
———————————————————————————————————
```


### 58. Building A Table Row By Row

`id: tabulation_concepts#6` | Source: `vignettes/tabulation_concepts.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

add_subgroup <- function(x) paste0(tolower(x), sample(1:3, length(x), TRUE))

df <- select(mutate(tibble(x = rnorm(100), c1 = factor(sample(c("A", "B", "C"), 100, 
    replace = TRUE), levels = c("A", "B", "C")), r1 = factor(sample(c("U", "V", "W"), 
    100, replace = TRUE), levels = c("U", "V", "W"))), c2 = add_subgroup(c1), r2 = add_subgroup(r1), 
    y = as.numeric(2 * as.numeric(c1) - 3 * as.numeric(r1))), c1, c2, r1, r2, x, 
    y)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("c1") |>
  build_table(df)
```

```text
A       B       C  
——————————————————————————————————
```


### 59. Introduction

`id: tabulation_dplyr#3` | Source: `vignettes/tabulation_dplyr.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

n <- 400

df <- mutate(tibble(arm = factor(sample(c("Arm A", "Arm B"), n, replace = TRUE), 
    levels = c("Arm A", "Arm B")), country = factor(sample(c("CAN", "USA"), n, replace = TRUE, 
    prob = c(0.55, 0.45)), levels = c("CAN", "USA")), gender = factor(sample(c("Female", 
    "Male"), n, replace = TRUE), levels = c("Female", "Male")), handed = factor(sample(c("Left", 
    "Right"), n, prob = c(0.6, 0.4), replace = TRUE), levels = c("Left", "Right")), 
    age = rchisq(n, 30) + 10), weight = 35 * rnorm(n, sd = 0.5) + ifelse(gender == 
    "Female", 140, 180))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("arm") |>
  split_cols_by("gender") |>
  build_table(df)
```

```text
Arm A                     Arm B         
             Female        Male        Female        Male   
             (N=96)      (N=105)       (N=92)      (N=107)  
————————————————————————————————————————————————————————————
```


### 60. various accessors work at the layout/table level

`id: test-accessors:13` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-accessors.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by_multivar(c("AGE", "BMRKR1")) |>
  build_table(DM)
```

```text
A: Drug X                            B: Placebo                          C: Combination           
         AGE              BMRKR1              AGE               BMRKR1              AGE               BMRKR1     
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 61. top_level_section_div works

`id: test-accessors:623` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-accessors.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(top_level_section_div = "a") |>
  split_cols_by("ARM") |>
  build_table(DM)
```

```text
A: Drug X   B: Placebo   C: Combination
———————————————————————————————————————————————————————————————————
```


### 62. as_html Viewer with newline test

`id: test-as_html:31` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-as_html.R`

**Setup for reproducibility:**

```r
library(rtables)

varlabs <- c("Mean Age", "SD\nLine Break!!! \nAge")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_multivar(c("AGE", "AGE"), varlabels = varlabs) |>
  build_table(DM)
```

```text
SD      
              Line Break!!! 
   Mean Age        Age      
————————————————————————————
```


### 63. as_html expands or not newlines depending on expand_newlines

`id: test-as_html:140` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-as_html.R`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

iris_mod <- mutate(mutate(iris, Species2 = as.factor(paste0("General", "\n ", as.character(Species)))), 
    Species = as.factor(sample(paste0("Petal", "\n ", as.character(Species)))))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("Species") |>
  build_table(iris_mod)
```

```text
Petal         Petal         Petal   
                 setosa      versicolor    virginica 
—————————————————————————————————————————————————————
```


### 64. mixing NA and non-NA counts and avars is ok

`id: test-binding:49` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-binding.R`

**Setup for reproducibility:**

```r
library(rtables)

iris2 <- iris

iris2$origin <- sample(c("USA", "CH", "JP"), nrow(iris), replace = TRUE)

iris2$manmade <- sample(c("Y", "N"), nrow(iris), replace = TRUE)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("manmade") |>
  build_table(iris2)
```

```text
Y     N 
————————————————
```


### 65. c/rbind and top-left behave

`id: test-binding:140` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-binding.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  build_table(DM)
```

```text
Hi         all obs     
———————————————————————
```


### 66. count visibility syncing works when cbinding

`id: test-binding:189` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-binding.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  split_cols_by("STRATA1") |>
  build_table(ex_adsl)
```

```text
A: Drug X                  B: Placebo               C: Combination     
         A        B        C        A        B        C        A        B        C   
       (N=38)   (N=47)   (N=49)   (N=44)   (N=45)   (N=45)   (N=40)   (N=43)   (N=49)
—————————————————————————————————————————————————————————————————————————————————————
```


### 67. count visibility syncing works when cbinding

`id: test-binding:196` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-binding.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by("STRATA1") |>
  build_table(ex_adsl)
```

```text
A: Drug X              B: Placebo            C: Combination    
              (N=134)                 (N=134)                 (N=132)       
         A       B       C       A       B       C       A       B       C  
————————————————————————————————————————————————————————————————————————————
```


### 68. equivalent split funs withs differrent environments dont' block rbinding

`id: test-binding:229` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-binding.R`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

combodf <- tribble(~valname, ~label, ~levelcombo, ~exargs, "A_B", "Arms A+B", c("A: Drug X", 
    "B: Placebo"), list(), "A_C", "Arms A+C", c("A: Drug X", "C: Combination"), list())
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM", split_fun = add_combo_levels(combodf)) |>
  build_table(DM)
```

```text
A: Drug X   B: Placebo   C: Combination   Arms A+B   Arms A+C
        (N=121)     (N=106)        (N=129)       (N=227)    (N=250) 
————————————————————————————————————————————————————————————————————
```


### 69. rbinding 2 objects with different titles/footers removes them

`id: test-binding:260` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-binding.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "Title A", subtitles = "Subtitle A", main_footer = "Footer A", 
    prov_footer = "PF A") |>
  build_table(ex_adsl)
```

```text
all obs
——————————
```


### 70. rbinding 2 objects with different titles/footers removes them

`id: test-binding:262` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-binding.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "Title B", subtitles = "Subtitle B", main_footer = "Footer B", 
    prov_footer = "PF B") |>
  build_table(ex_adsl)
```

```text
all obs
——————————
```


### 71. rbinding objects with only titles/footers for first object keeps them

`id: test-binding:273` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-binding.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "Title", subtitles = c("S1", "S2"), main_footer = c("F1", "F2"), 
    prov_footer = c("PF1", "PF2")) |>
  build_table(ex_adsl)
```

```text
all obs
——————————
```


### 72. rbinding objects with identical titles/footers keeps them

`id: test-binding:288` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-binding.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "Title", subtitles = c("S1", "S2", "S3"), main_footer = "Footer", 
    prov_footer = c("PF1", "PF2")) |>
  build_table(ex_adsl)
```

```text
all obs
——————————
```


### 73. Custom functions in multivar splits work

`id: test-custom_split_funs:25` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-custom_split_funs.R`

**Setup for reproducibility:**

```r
library(rtables)

uneven_splfun <- function(df, spl, vals = NULL, labels = NULL, trim = FALSE) {
    ret <- do_base_split(spl, df, vals, labels, trim)
    if (NROW(df) == 0) {
        ret <- lapply(ret, function(x) x[1])
    }
    ret
}
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by_multivar(c("USUBJID", "AESEQ", "BMRKR1"), varlabels = c("N", "E", "BMR1"), 
      split_fun = uneven_splfun) |>
  build_table(subset(ex_adae, as.numeric(ARM) <= 2))
```

```text
A: Drug X                    B: Placebo            C: Combination
    N    E          BMR1          N    E          BMR1               N       
—————————————————————————————————————————————————————————————————————————————
```


### 74. add_overall_level works

`id: test-custom_split_funs:50` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-custom_split_funs.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = add_overall_level("All Patients", first = FALSE)) |>
  build_table(DM)
```

```text
A: Drug X   B: Placebo   C: Combination   All Patients
—————————————————————————————————————————————————————————————
```


### 75. add_overall_level works

`id: test-custom_split_funs:56` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-custom_split_funs.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = add_overall_level("All Patients", first = TRUE)) |>
  build_table(DM)
```

```text
All Patients   A: Drug X   B: Placebo   C: Combination
—————————————————————————————————————————————————————————————
```


### 76. make_split_fun works

`id: test-custom_split_funs:114` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-custom_split_funs.R`

**Setup for reproducibility:**

```r
library(rtables)

mysplitfun <- make_split_fun(pre = list(drop_facet_levels), post = list(add_overall_facet("ALL", 
    "All Arms")))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM", split_fun = mysplitfun) |>
  build_table(subset(DM, ARM %in% c("B: Placebo", "C: Combination")))
```

```text
B: Placebo   C: Combination   All Arms
        (N=106)        (N=129)       (N=235) 
—————————————————————————————————————————————
```


### 77. restrict_facets works

`id: test-custom_split_funs:266` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-custom_split_funs.R`

**Setup for reproducibility:**

```r
library(rtables)

mflevs <- c("M", "F")

keep_reord <- restrict_facets(mflevs)

splfun <- make_split_fun(post = list(keep_reord))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("SEX", split_fun = splfun) |>
  build_table(DM)
```

```text
M       F  
————————————————————
```


### 78. add_colcounts works

`id: test-deprecated:54` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-deprecated.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  add_colcounts() |>
  build_table(DM)
```

```text
all obs
       (N=356)
——————————————
```


### 79. add_colcounts works

`id: test-deprecated:59` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-deprecated.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  build_table(DM)
```

```text
all obs
       (N=356)
——————————————
```


### 80. add_colcounts format argument works

`id: test-deprecated:68` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-deprecated.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  add_colcounts(format = "xx (xx%)") |>
  split_cols_by("ARM") |>
  build_table(DM)
```

```text
A: Drug X    B: Placebo   C: Combination
   121 (100%)   106 (100%)     129 (100%)  
———————————————————————————————————————————
```


### 81. referential footnotes work

`id: test-header-footer:19` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-header-footer.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "Title says Whaaaat", subtitles = "Oh, ok.", main_footer = "ha HA! Footer!") |>
  split_cols_by("ARM") |>
  build_table(ex_adsl)
```

```text
A: Drug X   B: Placebo   C: Combination
——————————————————————————————————————————————————
```


### 82. existing table in layout works

`id: test-lyt-tabulation:49` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
rawdat <- makefakedat()
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", "Gender", labels_var = "gend_label") |>
  build_table(rawdat)
```

```text
ARM1             ARM2     
                     Male    Female   Male    Female
————————————————————————————————————————————————————
```


### 83. ref_group comparisons work

`id: test-lyt-tabulation:245` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
rawdat <- makefakedat()
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", ref_group = "ARM1") |>
  build_table(rawdat)
```

```text
ARM1    ARM2 
——————————————————————————————————————————
```


### 84. split under analyze

`id: test-lyt-tabulation:489` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
rawdat <- makefakedat()
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by(var = "ARM") |>
  build_table(rawdat)
```

```text
ARM1      ARM2  
         (N=504)   (N=496)
——————————————————————————
```


### 85. extra args works

`id: test-lyt-tabulation:578` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

makefakedat2 <- function(n = 1000) {
  set.seed(0)
  if (n %% 4 != 0) stop("n not multiple of 4")
  many2s <- rep(2, n / 2)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = rep(c("ARM1", "ARM2"), times = rep(n / 2, 2)),
    SEX = rep(sample(c("M", "F"), n / 2, replace = TRUE), many2s),
    RACE = rep(sample(c("WHITE", "BLACK"), n / 2, replace = TRUE), times = many2s),
    PATID = rep(seq(1, n / 2), many2s),
    VISIT = rep(c("BASELINE", "FOLLOWUP"))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  mu <- 5 + (as.integer(factor(datadf$RACE)) + as.integer(factor(datadf$ARM)) + as.integer(factor(datadf$SEX))) / 2
  datadf$VALUE <- ifelse(datadf$VISIT == "BASELINE", 5, 5 + rnorm(n, mu, 4))
  datadf$PCTDIFF <- NA_real_
  seconds <- seq(2, n, by = 2)
  datadf$PCTDIFF[seq(2, n, by = 2)] <- 100 * (datadf$VALUE[seconds] - datadf$VALUE[seconds - 1]) / datadf$VALUE[seconds - 1]
  datadf
}
makefakedat2 <- function(n = 1000) {
  set.seed(0)
  if (n %% 4 != 0) stop("n not multiple of 4")
  many2s <- rep(2, n / 2)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = rep(c("ARM1", "ARM2"), times = rep(n / 2, 2)),
    SEX = rep(sample(c("M", "F"), n / 2, replace = TRUE), many2s),
    RACE = rep(sample(c("WHITE", "BLACK"), n / 2, replace = TRUE), times = many2s),
    PATID = rep(seq(1, n / 2), many2s),
    VISIT = rep(c("BASELINE", "FOLLOWUP"))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  mu <- 5 + (as.integer(factor(datadf$RACE)) + as.integer(factor(datadf$ARM)) + as.integer(factor(datadf$SEX))) / 2
  datadf$VALUE <- ifelse(datadf$VISIT == "BASELINE", 5, 5 + rnorm(n, mu, 4))
  datadf$PCTDIFF <- NA_real_
  seconds <- seq(2, n, by = 2)
  datadf$PCTDIFF[seq(2, n, by = 2)] <- 100 * (datadf$VALUE[seconds] - datadf$VALUE[seconds - 1]) / datadf$VALUE[seconds - 1]
  datadf
}
rawdat2 <- makefakedat2()
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by_multivar(c("VALUE", "PCTDIFF")) |>
  build_table(rawdat2)
```

```text
ARM1              ARM2      
   VALUE   PCTDIFF   VALUE   PCTDIFF
————————————————————————————————————
```


### 86. analysis function arguments work with NA rows in data

`id: test-lyt-tabulation:844` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

df <- data.frame(a_var = factor(c("a", NA, "b", "b", "a", "a", "b", "c", "a", NA)), 
    b_var = factor(c(NA, NA, "x", "x", "y", "x", "x", "y", "x", NA)))
```

**Core layout instructions:**

```r
basic_table() |>
  add_overall_col("all pts") |>
  build_table(df)
```

```text
all pts
—————————————————————————————————————
```


### 87. analyze_colvars inclNAs works

`id: test-lyt-tabulation:877` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

test <- data.frame(a = c(1, 2), b = c(1, NA))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_multivar(c("a", "b")) |>
  build_table(test)
```

```text
a   b
——————————————
```


### 88. analyze_colvars works generally

`id: test-lyt-tabulation:905` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

test <- data.frame(a = 1, b = 2, c = 3, d = 4, e = 5)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_multivar(c("a", "b", "c", "d")) |>
  build_table(test)
```

```text
a   b   c   d
————————————————————————
```


### 89. analyze_colvars works generally

`id: test-lyt-tabulation:909` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

test <- data.frame(a = 1, b = 2, c = 3, d = 4, e = 5)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_multivar(c("a", "b", "c", "d", "e")) |>
  build_table(test)
```

```text
a   b   c   d   e
————————————————————————————
```


### 90. topleft label position works

`id: test-lyt-tabulation:1044` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
rawdat <- makefakedat()
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", "Gender", labels_var = "gend_label") |>
  build_table(rawdat)
```

```text
ARM1                        ARM2           
Ethnicity                 Male         Female         Male         Female   
  Factor2                (N=256)       (N=248)       (N=248)       (N=248)  
————————————————————————————————————————————————————————————————————————————
```


### 91. .spl_context works in content and analysis functions

`id: test-lyt-tabulation:1147` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("M", "F"))) |>
  build_table(DM)
```

```text
A: Drug X                B: Placebo              C: Combination     
                             M            F            M            F            M            F     
————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 92. cut functions work

`id: test-lyt-tabulation:1191` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

ctnames <- c("young", "medium", "old")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by_cuts("AGE", split_label = "Age", cuts = c(0, 25, 35, 1000), cutlabels = ctnames) |>
  build_table(ex_adsl)
```

```text
A: Drug X              B: Placebo           C: Combination   
counts     young   medium   old   young   medium   old   young   medium   old
—————————————————————————————————————————————————————————————————————————————
```


### 93. empty factor levels represented correctly when ref group is set

`id: test-lyt-tabulation:1368` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-lyt-tabulation.R`

**Setup for reproducibility:**

```r
library(rtables)

df <- data.frame(val = 1:10, grp = factor(rep("a", 10), levels = c("a", "b")))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("grp", ref_group = "a") |>
  build_table(df)
```

```text
a     b 
————————————————
```


### 94. deeply nested and uneven column layouts work

`id: test-nesting:24` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-nesting.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by(var = "ARM") |>
  split_cols_by("STRATA1") |>
  split_cols_by("STRATA2") |>
  add_overall_col("All Patients") |>
  build_table(ex_adsl)
```

```text
A: Drug X                                            B: Placebo                                          C: Combination                                  
              A                 B                 C                 A                 B                 C                 A                 B                 C                      
         S1       S2       S1       S2       S1       S2       S1       S2       S1       S2       S1       S2       S1       S2       S1       S2       S1       S2     All Patients
       (N=18)   (N=20)   (N=28)   (N=19)   (N=27)   (N=22)   (N=22)   (N=22)   (N=19)   (N=26)   (N=26)   (N=19)   (N=14)   (N=26)   (N=18)   (N=25)   (N=24)   (N=25)     (N=400)   
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 95. Page by splitting works

`id: test-pagination:9` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-pagination.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "big title") |>
  build_table(DM)
```

```text
all obs
——————————————————————————
```


### 96. vertical and horizontal pagination work

`id: test-pagination:128` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-pagination.R`

**Setup for reproducibility:**

```r
library(rtables)

spoof_df <- data.frame(arm = factor(c("a", "b", "c", "d", "e", "f")), var1 = 6)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("arm") |>
  build_table(spoof_df)
```

```text
a      b      c      d      e      f  
————————————————————————————————————————————————————
```


### 97. Pagination works with referential footnotes

`id: test-pagination:532` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-pagination.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "main title", subtitles = "subtitle", main_footer = "main footer", 
    prov_footer = "provenance footer") |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  build_table(ex_adsl)
```

```text
A: Drug X                  B: Placebo                C: Combination      
                     F             M             F             M             F             M     
—————————————————————————————————————————————————————————————————————————————————————————————————
```


### 98. toString method works correclty

`id: test-printing:4` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-printing.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("Species") |>
  build_table(iris, hsep = "=")
```

```text
setosa      versicolor     virginica 
                 (N=50)        (N=50)        (N=50)   
======================================================
```


### 99. newline in column names and possibly cell values work

`id: test-printing:143` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-printing.R`

**Setup for reproducibility:**

```r
library(rtables)

df <- data.frame(n = 1, median = 10)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_multivar(vars = c("n", "median"), varlabels = c("N", "Median\n(Days)")) |>
  build_table(df)
```

```text
Median
       N   (Days)
—————————————————
```


### 100. Inset works for table, ref_footnotes, and main footer

`id: test-printing:493` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-printing.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = paste0("Very ", paste0(rep("very", 10), collapse = " "), " long title"), 
    subtitles = paste0("Very ", paste0(rep("very", 15), collapse = " "), " long subtitle"), 
    main_footer = paste0("Very ", paste0(rep("very", 6), collapse = " "), " long footer"), 
    prov_footer = paste0("Very ", paste0(rep("very", 15), collapse = " "), " prov footer"), 
    show_colcounts = TRUE, inset = 2) |>
  build_table(DM)
```

```text
all obs
                     (N=356)
  ——————————————————————————
```


### 101. row label indentation is kept even if there are newline characters

`id: test-printing:622` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-printing.R`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

ANL <- filter(mutate(DM, value = rnorm(n()), pctdiff = runif(n())), ARM == "A: Drug X")

ANL$ARM <- factor(ANL$ARM)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by_multivar(c("value", "pctdiff"), varlabels = c("abc", "def")) |>
  build_table(ANL)
```

```text
Ethnicity                                  A: Drug X       
  Sex                                  abc          def    
———————————————————————————————————————————————————————————
```


### 102. showing higher-level ncols works

`id: test-printing:869` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-printing.R`

**Setup for reproducibility:**

```r
library(rtables)

mydat <- subset(ex_adsl, SEX %in% c("M", "F"))

mydat$SEX2 <- factor(ifelse(mydat$SEX == "M", "males", "super long sentence that involves females"))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by("SEX2", show_colcounts = TRUE) |>
  split_cols_by("STRATA1") |>
  build_table(mydat)
```

```text
A: Drug X                                                                B: Placebo                                                              C: Combination                             
                                       (N=130)                                                                   (N=132)                                                                   (N=126)                                
               males              super long sentence that involves females              males              super long sentence that involves females              males              super long sentence that involves females   
              (N=51)                               (N=79)                               (N=55)                               (N=77)                               (N=60)                               (N=66)                     
         A       B       C           A                B                C           A       B       C           A                B                C           A       B       C           A                B                C      
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 103. add_overall_col with no col splits works

`id: test-regressions:282` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-regressions.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  add_overall_col("whaaat") |>
  build_table(DM)
```

```text
whaaat     
———————————————————————
```


### 104. column labeling works correctly when value label var is a factor

`id: test-regressions:335` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-regressions.R`

**Setup for reproducibility:**

```r
library(rtables)

ex_adsl$ARMLAB <- factor(ex_adsl$ARM, labels = c("Drug X", "Placebo", "Combination"))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", labels_var = "ARMLAB") |>
  build_table(ex_adsl)
```

```text
Drug X   Placebo   Combination
—————————————————————————————————————————
```


### 105. pathing works

`id: test-regressions:354` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-regressions.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  add_overall_col("overall") |>
  build_table(ex_adsl)
```

```text
A: Drug X   B: Placebo   C: Combination   overall
                      (N=134)     (N=134)        (N=132)       (N=400)
——————————————————————————————————————————————————————————————————————
```


### 106. specifying function format with no cfun in summarize_row_groups works

`id: test-regressions:460` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-regressions.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  build_table(DM[1:15, ])
```

```text
Ethnicity       F           M    
—————————————————————————————————
```


### 107. child_label = hidden does not affect tree structure/pathing

`id: test-regressions:482` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-regressions.R`

**Setup for reproducibility:**

```r
library(rtables)

df <- expand.grid(ARM = factor(paste("ARM", c("A", "B"))), FCT = factor(c("f1", "f2")))

df <- cbind(df, val = seq_len(NROW(df)))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", ref_group = "ARM A") |>
  build_table(df)
```

```text
ARM A   ARM B
——————————————————
```


### 108. export_as_txt works when there are newlines in column labels (naturally or after wrapping

`id: test-regressions:581` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-regressions.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ACTARM") |>
  split_cols_by_multivar(vars = c("AVAL", "CHG"), varlabels = c("Analysis Value", "Change from\nBaseline")) |>
  build_table(formatters::ex_adlb)
```

```text
A: Drug X                               B: Placebo                            C: Combination            
                                                             Change from                              Change from                             Change from    
                                        Analysis Value         Baseline         Analysis Value         Baseline          Analysis Value         Baseline     
                                           (N=2814)            (N=2814)            (N=2814)            (N=2814)             (N=2772)            (N=2772)     
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 109. overridden colcounts via build_table are used during tabulation correctly

`id: test-regressions:616` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-regressions.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("STRATA1") |>
  build_table(ex_adsl, col_counts = 1:9)
```

```text
A: Drug X              B: Placebo            C: Combination    
         A       B       C       A       B       C       A       B       C  
       (N=1)   (N=2)   (N=3)   (N=4)   (N=5)   (N=6)   (N=7)   (N=8)   (N=9)
————————————————————————————————————————————————————————————————————————————
```


### 110. as_result_df keeps label rows

`id: test-result_data_frame:159` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-result_data_frame.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("STRATA2") |>
  build_table(ex_adsl)
```

```text
A: Drug X      B: Placebo      C: Combination  
              S1      S2      S1      S2       S1        S2   
——————————————————————————————————————————————————————————————
```


### 111. make_ard works with summarize_row_groups

`id: test-result_data_frame:425` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-result_data_frame.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("STRATA1") |>
  build_table(ex_adsl)
```

```text
A: Drug X                              B: Placebo                           C: Combination           
                             A            B            C            A            B            C            A            B            C     
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 112. make_ard works if there are no stat_names

`id: test-result_data_frame:503` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-result_data_frame.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE, colcount_format = "N=xx") |>
  split_cols_by("ARM", split_fun = keep_split_levels(c("A: Drug X", "B: Placebo"))) |>
  build_table(ex_adsl)
```

```text
A: Drug X    B: Placebo
              N=134        N=134   
———————————————————————————————————
```


### 113. make_ard works with split_cols_by_multivar

`id: test-result_data_frame:564` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-result_data_frame.R`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

n <- 400

df <- mutate(tibble(arm = factor(sample(c("Arm A", "Arm B"), n, replace = TRUE), 
    levels = c("Arm A", "Arm B")), country = factor(sample(c("CAN", "USA"), n, replace = TRUE, 
    prob = c(0.55, 0.45)), levels = c("CAN", "USA")), gender = factor(sample(c("Female", 
    "Male"), n, replace = TRUE), levels = c("Female", "Male")), handed = factor(sample(c("Left", 
    "Right"), n, prob = c(0.6, 0.4), replace = TRUE), levels = c("Left", "Right")), 
    age = rchisq(n, 30) + 10), weight = 35 * rnorm(n, sd = 0.5) + ifelse(gender == 
    "Female", 140, 180))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("arm") |>
  split_cols_by_multivar(c("age", "weight")) |>
  build_table(df)
```

```text
Arm A                       Arm B          
             age         weight          age         weight   
——————————————————————————————————————————————————————————————
```


### 114. Subsetting by integer(0) keeps decorations

`id: test-subset-access:116` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-subset-access.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "t", subtitles = "s", main_footer = "mf", prov_footer = "pf") |>
  build_table(DM)
```

```text
t
s

——————————————
```


### 115. make_row_df, make_col_df give paths which all work

`id: test-subset-access:174` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-subset-access.R`

**Setup for reproducibility:**

```r
library(rtables)

makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
makefakedat <- function(n = 1000) {
  set.seed(0)
  datadf <- data.frame(
    stringsAsFactors = FALSE,
    ARM = c("ARM1", sample(c("ARM1", "ARM2"), n - 1, replace = TRUE)),
    SEX = c("M", sample(c("M", "F"), n - 1, replace = TRUE)),
    FACTOR2 = c("A", sample(c("A", "B", "C"), n - 1, replace = TRUE)),
    RACE = c("WHITE", sample(c("WHITE", "BLACK"), n - 1, replace = TRUE)),
    AGE = runif(n, 40, 70),
    VAR3 = c("level1", sample(c("level1", "level2"), n - 1, replace = TRUE))
  )
  datadf$ethn_label <- c(WHITE = "Caucasian", BLACK = "African American")[datadf$RACE]
  datadf$fac2_label <- paste("Level", datadf$FACTOR2)
  datadf$gend_label <- c(M = "Male", F = "Female")[datadf$SEX]
  datadf
}
rawdat <- makefakedat()
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", ref_group = "F") |>
  build_table(rawdat)
```

```text
ARM1                                  ARM2                
                                      M                  F                  M                  F        
————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 116. Duplicate colvars path correctly

`id: test-subset-access:269` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-subset-access.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_multivar(c("AGE", "BMRKR1", "AGE"), varlabels = c("Age", "Biomarker 1", 
      "Second Age")) |>
  build_table(DM)
```

```text
Age            Biomarker 1        Second Age   
————————————————————————————————————————————————————————————
```


### 117. top_left, title, footers retention behaviors are correct across all scenarios

`id: test-subset-access:311` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-subset-access.R`

**Setup for reproducibility:**

```r
library(rtables)

ti <- "ti"

sti <- "sti"

mf <- "mf"

pf <- "pf"
```

**Core layout instructions:**

```r
basic_table(title = ti, subtitles = sti, main_footer = mf, prov_footer = pf) |>
  split_cols_by("ARM") |>
  build_table(DM)
```

```text
ti
sti

—————————————————————————————————————————————————————————————————————————
```


### 118. tt_at_path and cell_values work with values even if they differ in naming

`id: test-subset-access:616` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-subset-access.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(var = "ARM", split_label = "asdar") |>
  add_colcounts() |>
  build_table(df = DM)
```

```text
A: Drug X          B: Placebo       C: Combination 
              (N=121)            (N=106)            (N=129)     
————————————————————————————————————————————————————————————————
```


### 119. subset_cols works as intended

`id: test-subset-access:868` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-subset-access.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table(title = "Title", subtitles = c("Sub", "titles"), prov_footer = "prov footer", 
    main_footer = "main footer") |>
  split_cols_by("ARM") |>
  split_cols_by("SEX") |>
  add_overall_col("All Patients") |>
  build_table(DM)
```

```text
A: Drug X                                              B: Placebo                                           C: Combination                                  
             F            M           U      UNDIFFERENTIATED       F            M           U      UNDIFFERENTIATED       F            M           U      UNDIFFERENTIATED   All Patients
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 120. cell_values(colpath=) does not choke on regex chars in path

`id: test-subset-access:932` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-subset-access.R`

**Setup for reproducibility:**

```r
library(rtables)

bad_levs <- c("^][.$+", ")(\\|?<>")

data <- DM

data$scary_fac <- factor(sample(bad_levs, nrow(data), replace = TRUE), levels = bad_levs)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", split_fun = keep_split_levels(levels(DM$ARM)[1:2])) |>
  split_cols_by("scary_fac") |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  build_table(data)
```

```text
A: Drug X                      B: Placebo          
          ^][.$+          )(\|?<>         ^][.$+          )(\|?<>   
         F       M       F       M       F       M       F       M  
————————————————————————————————————————————————————————————————————
```


### 121. full alt_counts_df is accessible from afun/cfun via .alt_df_full

`id: test-tab_afun_cfun:373` | Source: `/Users/gbecker/gabe/checkedout/rtables/tests/testthat/test-tab_afun_cfun.R`

**Setup for reproducibility:**

```r
library(rtables)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("SEX") |>
  split_cols_by("STRATA1") |>
  build_table(ex_adae[1:3, ], alt_counts_df = ex_adsl)
```

```text
F              M              U            UNDIFFERENTIATED   
            A    B    C    A    B    C    A    B    C      A        B       C  
———————————————————————————————————————————————————————————————————————————————
```


### 122. a_event_free works with default arguments in a table layout

`id: test:test-event_free:68` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-event_free.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(tern)

library(dplyr)

adtte_f <- dplyr::mutate(dplyr::filter(tern_ex_adtte, PARAMCD == "OS"), AVAL = tern::day2month(AVAL), 
    is_event = CNSR == 0)
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by(var = "ARMCD") |>
  build_table(df = adtte_f)
```

```text
ARM A   ARM B   ARM C
————————————————————————
```


### 123. (file level)

`id: test:test-pruning_functions:516` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-pruning_functions.R`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

my_DM <- filter(formatters::DM, RACE == "THIS LEAVES EMPTY DF")
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("ARM") |>
  build_table(my_DM)
```

```text
A: Drug X   B: Placebo   C: Combination
——————————————————————————————————————————————
```


### 124. (file level)

`id: test:test-sorting_functions:13` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-sorting_functions.R`

**Setup for reproducibility:**

```r
library(rtables)

DM2 <- formatters::DM

DM2$spanhead <- factor(ifelse(DM2$ARM == "C: Combination", " ", "This is a Spanning Header"), 
    levels = c("This is a Spanning Header", " "))
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("spanhead", split_fun = trim_levels_in_group("ARM")) |>
  split_cols_by("ARM") |>
  build_table(DM2)
```

```text
This is a Spanning Header                   
          A: Drug X      B: Placebo    C: Combination
—————————————————————————————————————————————————————
```


### 125. (file level)

`id: test:test-sorting_functions:134` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-sorting_functions.R`

**Setup for reproducibility:**

```r
library(rtables)

DM2 <- formatters::DM

DM2$spanhead <- factor(ifelse(DM2$ARM == "C: Combination", " ", "This is a Spanning Header"), 
    levels = c("This is a Spanning Header", " "))
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("spanhead", split_fun = trim_levels_in_group("ARM")) |>
  split_cols_by("ARM") |>
  split_cols_by("RACE") |>
  build_table(DM2)
```

```text
This is a Spanning Header                                                                                                                                                                                                                                                                                                                        
                                                                                   A: Drug X                                                                                                                                                          B: Placebo                                                                                                                                                       C: Combination                                                                         
          ASIAN      BLACK OR AFRICAN AMERICAN     WHITE     AMERICAN INDIAN OR ALASKA NATIVE   MULTIPLE   NATIVE HAWAIIAN OR OTHER PACIFIC ISLANDER    OTHER    UNKNOWN     ASIAN      BLACK OR AFRICAN AMERICAN     WHITE     AMERICAN INDIAN OR ALASKA NATIVE   MULTIPLE   NATIVE HAWAIIAN OR OTHER PACIFIC ISLANDER    OTHER    UNKNOWN     ASIAN      BLACK OR AFRICAN AMERICAN     WHITE     AMERICAN INDIAN OR ALASKA NATIVE   MULTIPLE   NATIVE HAWAIIAN OR OTHER PACIFIC ISLANDER    OTHER    UNKNOWN
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 126. a_summarize_ancova_j (s_ancova_j) with a combined column and method_combo = collapse

`id: test:test-summarize_ancova:429` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-summarize_ancova.R`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

iris_plus <- mutate(iris, Color = factor(sample(c("red", "blue"), size = nrow(iris), 
    prob = c(0.8, 0.2), replace = TRUE)))

iris_plus2 <- ungroup(filter(mutate(group_by(iris_plus, Species), id = row_number()), 
    Species == "setosa" & id < 45 | Species == "versicolor" & id < 30 | Species == 
        "virginica" & id < 50))

iris_plus2_fix <- iris_plus2

iris_plus2_fix[["Species"]] <- factor(as.character(iris_plus2_fix[["Species"]]), 
    levels = c("setosa", "versicolor", "virginica"), labels = c("Combined: setosa + virginica", 
        "versicolor", "Combined: setosa + virginica"))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("Species", ref_group = "versicolor") |>
  add_colcounts() |>
  build_table(iris_plus2_fix)
```

```text
Combined: setosa + virginica   versicolor
                                                    (N=93)                (N=29)  
——————————————————————————————————————————————————————————————————————————————————
```


### 127. more top left than col headers works

`id: test:test-tt_to_tblfile:334` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-tt_to_tblfile.R`

**Setup for reproducibility:**

```r
library(rtables)

data <- data.frame(TRT01A = "Dummy Treatment A", SEX = "M", AGEGR1 = "<65 years", 
    RACE = "white", AGE = 65, empty_split = " ")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("empty_split") |>
  split_cols_by("TRT01A") |>
  build_table(data)
```

```text
SEX                            
  AGEGR1                       
    RACE      Dummy Treatment A
———————————————————————————————
```


### 128. Longitudinal Visit / Repeated Measures Columns

`id: missing_col_structures#1` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

keep_visits <- levels(ex_advs$AVISIT)[c(2, 5, 7)]
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by(
    "AVISIT",
    split_fun = keep_split_levels(keep_visits),
    show_colcounts = TRUE
  ) |>
  build_table(ex_advs)
```

```text
                  A: Drug X                                  B: Placebo                               C: Combination             
                   (N=5628)                                   (N=5628)                                   (N=5544)                
   BASELINE   WEEK 3 DAY 22   WEEK 5 DAY 36   BASELINE   WEEK 3 DAY 22   WEEK 5 DAY 36   BASELINE   WEEK 3 DAY 22   WEEK 5 DAY 36
   (N=804)       (N=804)         (N=804)      (N=804)       (N=804)         (N=804)      (N=792)       (N=792)         (N=792)   
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 129. Longitudinal Visits with Conditional Change-from-Baseline (Baseline Observed vs. Post-Baseline Observed and CFB)

`id: argon_proposal#A2` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

drop_cfb_at_baseline <- function(ret, spl, fulldf, .spl_context) {
    parent_visit <- tail(.spl_context$value, 1)
    if (identical(parent_visit, "BASELINE")) {
        ret <- restrict_facets("AVAL", op = "keep")(ret, spl, fulldf)
    }
    ret
}

dat <- subset(ex_advs, AVISIT %in% c("BASELINE", "WEEK 1 DAY 8", "WEEK 2 DAY 15") & 
    PARAMCD == "DIABP")

dat$AVISIT <- droplevels(dat$AVISIT)

dat$CHG <- dat$AVAL - 80
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("AVISIT", show_colcounts = TRUE) |>
  split_cols_by_multivar(
    vars = c("AVAL", "CHG"),
    varlabels = c("Observed", "CFB"),
    split_fun = make_split_fun(post = list(drop_cfb_at_baseline))
  ) |>
  build_table(dat)
```

```text
   BASELINE    WEEK 1 DAY 8      WEEK 2 DAY 15  
   (N=400)        (N=400)           (N=400)     
   Observed   Observed    CFB   Observed    CFB 
————————————————————————————————————————————————
```


### 130. Asymmetric / Ragged Split Trees (Conditional Nesting)

`id: missing_col_structures#2` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- ex_adsl

dat$span_lab <- factor(ifelse(dat$ARM == "B: Placebo", "Placebo", "Active Drug"), 
    levels = c("Placebo", "Active Drug"))

dat$arm_lab <- ifelse(dat$span_lab == "Placebo", "Total", as.character(dat$ARM))

make_ragged_post <- function(splret, spl, fulldf, .spl_context) {
    parent <- .spl_context$value[[1]]
    tot_nm <- ifelse(parent == "Placebo", "Total", "Active Total")
    if (parent != "Placebo") {
        ret <- add_to_split_result(splret, values = tot_nm, labels = tot_nm, datasplit = setNames(list(fulldf), 
            tot_nm), subset_exprs = list(quote(TRUE)))
    }
    else {
        ret <- make_split_result(values = tot_nm, labels = tot_nm, datasplit = list(Total = fulldf), 
            subset_exprs = list(quote(TRUE)))
    }
    ret
}

ragged_dose_split <- make_split_fun(post = list(make_ragged_post))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("span_lab", split_fun = trim_levels_in_group("ARM"), show_colcounts = TRUE) |>
  split_cols_by("ARM",
                labels_var = "arm_lab",
                split_fun = ragged_dose_split,
                show_colcounts = TRUE) |>
  build_table(dat)
```

```text
   Placebo                  Active Drug               
   (N=134)                    (N=266)                 
    Total    A: Drug X   C: Combination   Active Total
   (N=134)    (N=134)       (N=132)         (N=266)   
——————————————————————————————————————————————————————
```


### 131. Cross-Variable Conditional Subgroup Split (Control Unstratified vs. Active Arms Stratified by Biomarker/ADA + Subtotal)

`id: argon_proposal#A4` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

set.seed(42)

dat <- ex_adsl

dat$ARM <- factor(as.character(dat$ARM), levels = c("B: Placebo", "A: Drug X", "C: Combination"))

dat$ADA_STATUS <- factor(ifelse(dat$ARM == "B: Placebo", "ADA-", sample(c("ADA-", 
    "ADA+"), nrow(dat), replace = TRUE, prob = c(0.75, 0.25))), levels = c("ADA-", 
    "ADA+"))

cond_ada_split <- function(ctrl_arm = "B: Placebo") {
    make_split_fun(post = list(function(ret, spl, fulldf, .spl_context) {
        parent_arm <- tail(.spl_context$value, 1)
        if (identical(parent_arm, ctrl_arm)) {
            make_split_result(values = "all_ctrl", datasplit = list(all_ctrl = fulldf), 
                labels = c(all_ctrl = "All Control"), subset_exprs = list(quote(TRUE)))
        } else {
            add_overall_facet("all_active", "All")(ret, spl, .spl_context, fulldf)
        }
    }))
}
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by("ADA_STATUS", split_fun = cond_ada_split("B: Placebo"), show_colcounts = TRUE) |>
  build_table(dat)
```

```text
   B: Placebo            A: Drug X                  C: Combination      
     (N=134)              (N=134)                      (N=132)          
   All Control    ADA-      ADA+      All      ADA-      ADA+      All  
     (N=134)     (N=101)   (N=33)   (N=134)   (N=100)   (N=32)   (N=132)
————————————————————————————————————————————————————————————————————————
```


### 132. Cumulative Dose & Escalating Exposure Bands (Phase I Oncology)

`id: missing_col_structures#3` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

set.seed(1)
dat <- data.frame(USUBJID = seq(1, 1000), dose = sample(c(10, 25, 50), 1000, replace = TRUE))
dat$ARM <- factor(paste(dat$dose, "mg"), levels = paste(c(10, 25, 50), "mg"))

only_overall_spl <- function(nm, label = nm) {
    make_split_fun(post = list(add_overall_facet(nm, label = label), restrict_facets(nm)))
}

lvls <- levels(dat$ARM)

cumlvls <- paste(">=", lvls)
```

**Core layout instructions:**

```r
basic_table() |>
  add_overall_col("Discrete Cohorts") |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  add_overall_col("Cumulative Exposure Bands") |>
  split_cols_by("ARM",
                split_fun = add_combo_levels(data.frame(valname = cumlvls, label = cumlvls, levelcombo = I(list(lvls[1], lvls[1:2], lvls[1:3])), exargs = I(replicate(3, list()))), keep_levels = cumlvls),
                show_colcounts = TRUE) |>
  add_overall_col("Total") |>
  split_cols_by("ARM", split_fun = only_overall_spl("all_doses", "All Doses"), show_colcounts = TRUE) |>
  build_table(dat)
```

```text
        Discrete Cohorts            Cumulative Exposure Bands        Total  
    10 mg     25 mg     50 mg    >= 10 mg    >= 25 mg   >= 50 mg   All Doses
   (N=351)   (N=339)   (N=310)    (N=351)    (N=690)    (N=1000)   (N=1000) 
————————————————————————————————————————————————————————————————————————————
```


### 133. Monotonically Contracting Disposition / Attrition Funnel Across Treatment Arms

`id: argon_proposal#A10` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

set.seed(123)

dat <- subset(ex_adsl, ARM %in% c("A: Drug X", "B: Placebo"))

dat$ARM <- droplevels(dat$ARM)

dat$RANDFL <- "Y"

dat$TRTEDFL <- ifelse(runif(nrow(dat)) > 0.04, "Y", "N")

dat$WK12FL <- ifelse(dat$TRTEDFL == "Y" & runif(nrow(dat)) > 0.15, "Y", "N")

dat$COMPFL <- ifelse(dat$WK12FL == "Y" & runif(nrow(dat)) > 0.12, "Y", "N")

predicate_cols <- function(preds) {
    make_split_fun(core_split = function(spl, df, vals, labels, .spl_context) {
        make_split_result(values = names(preds), labels = names(preds), datasplit = lapply(preds, 
            function(p) df[eval(p, df), , drop = FALSE]), subset_exprs = unname(preds))
    })
}

funnel_stages <- list(Randomized = quote(RANDFL == "Y"), Treated = quote(RANDFL == 
    "Y" & TRTEDFL == "Y"), `Reached Wk 12` = quote(RANDFL == "Y" & TRTEDFL == "Y" & 
    WK12FL == "Y"), Completed = quote(RANDFL == "Y" & TRTEDFL == "Y" & WK12FL == 
    "Y" & COMPFL == "Y"))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by("STUDYID", split_fun = predicate_cols(funnel_stages), show_colcounts = TRUE) |>
  build_table(dat)
```

```text
                      A: Drug X                                          B: Placebo                   
                       (N=134)                                            (N=134)                     
   Randomized   Treated   Reached Wk 12   Completed   Randomized   Treated   Reached Wk 12   Completed
    (N=134)     (N=132)      (N=104)       (N=85)      (N=134)     (N=133)      (N=114)       (N=103) 
——————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 134. Crossover & Multi-Period Study Layouts

`id: missing_col_structures#5` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- data.frame(seq = factor(rep(c("Sequence AB", "Sequence BA"), times = c(4,4))),
                  period = factor(paste("Period", rep(1:2, times = 8)))) |>
  mutate(trt = factor(case_when(seq == "Sequence AB" & period == "Period 1" ~ "Trt A",
                                seq == "Sequence AB" & period == "Period 2" ~ "Trt B",
                                seq == "Sequence BA" & period == "Period 1" ~ "Trt B",
                                seq == "Sequence BA" & period == "Period 2" ~ "Trt A")))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("seq", show_colcounts = TRUE) |>
  split_cols_by("period", split_fun = trim_levels_in_group("trt")) |>
  split_cols_by("trt") |>
  build_table(dat)
```

```text
       Sequence AB           Sequence BA    
          (N=8)                 (N=8)       
   Period 1   Period 2   Period 1   Period 2
    Trt A      Trt B      Trt B      Trt A  
    (N=4)      (N=4)      (N=4)      (N=4)  
————————————————————————————————————————————
```


### 135. The Market Research "Banner Table" (Parallel Non-Nested Cross-Tabs)

`id: missing_col_structures#6` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- ex_adsl
cntry <- dat$COUNTRY
dat$REGION <- case_when(cntry %in% c("CHN", "JPN") ~ "Asia",
                        cntry %in% c("USA", "BRA", "CAN") ~ "Americas",
                        cntry %in% c("PAK", "RUS", "GBR", "NGA") ~ "Europe")
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  add_overall_col("Total") |>
  split_cols_by("STUDYID", split_fun = add_combo_levels(data.frame(valname = "Sample", label = "Sample", levelcombo = I(list(select_all_levels)), exargs = I(list(list()))),
                                                        keep_levels = "Sample"),
                show_colcounts = TRUE) |>
  add_overall_col("Gender") |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("M", "F")), show_colcounts = TRUE) |>
  add_overall_col("Age Bracket") |>
  split_cols_by_cuts("AGE", c(18, 35, 55, 100), cutlabels = c("18-34", "35-54", "55+"), show_colcounts = TRUE) |>
  add_overall_col("Region") |>
  split_cols_by("REGION", show_colcounts = TRUE) |>
  build_table(dat)
```

```text
    Total         Gender                Age Bracket                    Region           
   Sample       M         F       18-34     35-54     55+     Asia     Europe   Americas
   (N=400)   (N=166)   (N=222)   (N=225)   (N=169)   (N=6)   (N=237)   (N=85)    (N=78) 
————————————————————————————————————————————————————————————————————————————————————————
```


### 136. 3-Tier Parallel Demographic Subgroup Banner Crossed with Treatment Arms

`id: argon_proposal#A6a` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- subset(ex_adsl, ARM %in% c("A: Drug X", "B: Placebo") & SEX %in% c("M", "F"))

dat$ARM <- droplevels(dat$ARM)

dat$SEX <- droplevels(dat$SEX)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  add_overall_col("Sex Subgroup") |>
  split_cols_by("SEX") |>
  split_cols_by("ARM") |>
  add_overall_col("Age Subgroup") |>
  split_cols_by_cuts("AGE", cuts = c(0, 35, 100), cutlabels = c("< 35 yrs", ">= 35 yrs")) |>
  split_cols_by("ARM") |>
  build_table(dat)
```

```text
                    Sex Subgroup                                      Age Subgroup                  
             F                        M                     < 35 yrs                >= 35 yrs       
   A: Drug X   B: Placebo   A: Drug X   B: Placebo   A: Drug X   B: Placebo   A: Drug X   B: Placebo
    (N=79)       (N=77)      (N=51)       (N=55)      (N=76)       (N=68)      (N=54)       (N=64)  
————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 137. Dual-Strata Projections (Unweighted Counts vs. Weighted %)

`id: missing_col_structures#7` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- data.frame(region = 1:5,
                  strata = rep(c("Urban", "Rural"), times = c(3, 2)),
                  sample_n = c(100, 200, 150, 100, 75),
                  weighted_pct = c(.25, .3, .3, .1, .05))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("strata") |>
  split_cols_by_multivar(c("sample_n", "weighted_pct"), varlabels = c("Unweighted (n)", "Weighted (%)")) |>
  build_table(dat)
```

```text
               Urban                           Rural            
   Unweighted (n)   Weighted (%)   Unweighted (n)   Weighted (%)
————————————————————————————————————————————————————————————————
```


### 138. Measure-First / Co-Primary Endpoint Hierarchy (Outer Multi-Variable by Inner Treatment Arms + Total)

`id: argon_proposal#A1` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- subset(ex_adsl, ARM %in% c("A: Drug X", "B: Placebo"))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_multivar(
    vars = c("AGE", "BMRKR1"),
    varlabels = c("Age (Years)", "Biomarker 1 (ng/mL)")
  ) |>
  split_cols_by(
    "ARM",
    split_fun = make_split_fun(
      pre = list(drop_facet_levels),
      post = list(add_overall_facet("Total", "Total"))
    ),
    show_colcounts = TRUE
  ) |>
  build_table(dat)
```

```text
             Age (Years)                    Biomarker 1 (ng/mL)       
   A: Drug X   B: Placebo    Total    A: Drug X   B: Placebo    Total 
    (N=134)     (N=134)     (N=268)    (N=134)     (N=134)     (N=268)
——————————————————————————————————————————————————————————————————————
```


### 139. Financial Variance & Scenario Reporting (Actual vs. Budget vs. Variance)

`id: missing_col_structures#8` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- data.frame(entry = c("gross_income", "Toys", "Seeds", "Roofing"), budget_amt = c(15000, -5000, -2000, -7500), budget_pct_rev = c(1, -.33, -.13, -.5),  actual_amt = c(14750, -4400, -2200, -7000), actual_pct_rev = c(1, 2500, 173, 40000), actual_rev_pct  = c(1, -.30, -.15, -.47))

make_all_subset_exprs <- function(n) {
    ret <- vector("list", n)
    for (i in seq_len(n)) ret[[i]] <- quote(TRUE)
    ret
}

make_multi_allcols <- function(cats, labs = cats) {
    bblock <- function(splres, spl, fulldf) {
        ncats <- length(cats)
        make_split_result(values = cats, datasplit = replicate(ncats, list(fulldf)), 
            labels = labs, subset_exprs = make_all_subset_exprs(ncats))
    }
    make_split_fun(post = list(bblock))
}
```

**Core layout instructions:**

```r
basic_table() |>
  add_overall_col("Q1 2024 Actual") |>
  split_cols_by("entry", split_fun = make_multi_allcols(c("actual_amt", "actual_rev_pct"),
                                                        c("Amount ($)", "% Revenue"))) |>
  add_overall_col("Q1 2024 Budget") |>
  split_cols_by("entry", split_fun = make_multi_allcols(c("budget_amt", "budget_rev_pct"),
                                                        c("Amount ($)", "% Revenue"))) |>
  add_overall_col("Variance") |>
  split_cols_by("entry", split_fun = make_multi_allcols(c("delta_amt", "variance"), 
                                                        c("Delta Amount ($)", "Variance %"))) |>
  build_table(dat)
```

```text
       Q1 2024 Actual           Q1 2024 Budget                 Variance           
   Amount ($)   % Revenue   Amount ($)   % Revenue   Delta Amount ($)   Variance %
——————————————————————————————————————————————————————————————————————————————————
```


### 140. Period-over-Period (PoP) & Cohort Retention Matrices

`id: missing_col_structures#9` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- data.frame(cohort = rep(c("Q1", "Q2"), times = c(6, 8)),
                  user = c(1, 1, 1, 2, 2, 3, 4, 4, 5, 5, 5, 5, 6, 7),
                  day_milestone = c( 0, 7, 30, 0, 7, 0, 0, 7, 0, 7, 30, 90, 0, 0))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("cohort") |>
  split_cols_by_cuts("day_milestone", cuts = c(-1, 6, 29, 89, 100000), cutlabels = paste("Day", c(0, 7, 30, 90))) |>
  build_table(dat)
```

```text
                 Q1                                Q2               
   Day 0   Day 7   Day 30   Day 90   Day 0   Day 7   Day 30   Day 90
   (N=3)   (N=2)   (N=1)    (N=0)    (N=4)   (N=2)   (N=1)    (N=1) 
————————————————————————————————————————————————————————————————————
```


### 141. Model Evaluation Leaderboard Across Partitions

`id: missing_col_structures#10` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- data.frame(dataset = rep(c("Validation", "Test", "Challenge"), times = c(3, 3, 3)),
                  model = rep(c("Baseline", "Model A", "Model B"), times = 3),
                  error = c(.1, .05, .03, .05, .02, .03, .14, .8, .1))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("dataset") |>
  split_cols_by("model") |>
  build_table(dat)
```

```text
            Validation                        Test                        Challenge          
   Baseline   Model A   Model B   Baseline   Model A   Model B   Baseline   Model A   Model B
    (N=1)      (N=1)     (N=1)     (N=1)      (N=1)     (N=1)     (N=1)      (N=1)     (N=1) 
—————————————————————————————————————————————————————————————————————————————————————————————
```


### 142. High-Order Factorial Designs (2^k)

`id: missing_col_structures#12` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- data.frame(GENOTYPE = rep(c("Wild-Type (WT)", "Knockout (KO)"), times = c(8, 8)),
                  TREATMENT = rep(rep(c("Vehicle", "Compound A"), c(4, 4)), 2),
                  STIMULATION = rep(c("Unstim", "Stim"), 8))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("GENOTYPE") |>
  split_cols_by("TREATMENT") |>
  split_cols_by("STIMULATION") |>
  build_table(dat)
```

```text
          Wild-Type (WT)                   Knockout (KO)        
      Vehicle       Compound A        Vehicle       Compound A  
   Unstim   Stim   Unstim   Stim   Unstim   Stim   Unstim   Stim
————————————————————————————————————————————————————————————————
```


### 143. Per-Parent Data-Driven Bins (Quantiles Computed Within Each Facet)

`id: claude_proposal#P1` | Source: `claude_col_proposals_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

per_facet_qrt_core <- function(var) {
    function(spl, df, vals, labels, .spl_context) {
        vec <- as.numeric(as.character(df[[var]]))
        qs <- unname(quantile(vec, c(0.25, 0.5, 0.75)))
        fqs <- format(qs, digits = 2)
        cats <- cut(vec, c(-Inf, qs, Inf), labels = c(paste0("<=", fqs[1]), paste0(fqs[1], 
            "-", fqs[2]), paste0(fqs[2], "-", fqs[3]), paste0(">", fqs[3])))
        nms <- c("ltq1", "q1toq2", "q2toq3", "gtq3")
        make_split_result(values = nms, datasplit = setNames(split(df, cats), nms), 
            labels = setNames(levels(cats), nms), subset_exprs = list(substitute(var <= 
                val, list(var = as.name(var), val = qs[1])), substitute(var <= val2 & 
                var > val1, list(var = as.name(var), val1 = qs[1], val2 = qs[2])), 
                substitute(var <= val2 & var > val1, list(var = as.name(var), val1 = qs[2], 
                  val2 = qs[3])), substitute(var > val, list(var = as.name(var), 
                  val = qs[3]))))
    }
}

splfun <- make_split_fun(core_split = per_facet_qrt_core("AGE"))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM", split_fun = keep_split_levels(c("A: Drug X", "B: Placebo"))) |>
  split_cols_by("STUDYID", split_fun = splfun) |>
  build_table(ex_adsl)
```

```text
               A: Drug X                          B: Placebo            
    <=29    29-34    34-40     >40      <=32    32-37    37-42     >42  
   (N=35)   (N=35)   (N=37)   (N=27)   (N=41)   (N=29)   (N=33)   (N=31)
————————————————————————————————————————————————————————————————————————
```


### 144. Overlapping Interval Columns on a Continuous Variable (Cumulative and Sliding Windows)

`id: claude_proposal#P2` | Source: `claude_col_proposals_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- ex_advs
dat$ADY_WK <- floor(dat$ADY/7)

sliding_windows <- function(var, starts, ends, labels) {
    make_split_fun(core_split = function(spl, df, vals, labels_in, trim, .spl_context) {
        make_split_result(values = labels, labels = labels, datasplit = Map(function(s, 
            e) df[df[[var]] >= s & df[[var]] <= e, ], starts, ends), subset_exprs = Map(function(s, 
            e) bquote(.(as.name(var)) >= .(s) & .(as.name(var)) <= .(e)), starts, 
            ends))
    })
}
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  add_overall_col("Cumulative follow-up") |>
  split_cols_by_cuts("ADY_WK", cuts = c(-1, 32, 64, 96, 10000),
                     cutlabels = c("<= 32 wks", "<= 64 wks", "<= 96 wks", "All"),
                     cumulative = TRUE) |>
  add_overall_col("Sliding windows") |>
  split_cols_by("STUDYID", split_fun = sliding_windows("ADY_WK", c(0, 26, 52, 78), c(52, 78, 104, 104),
                                                       c("Wk 0-52", "Wk 26-78", "Wk 52-104", "Wk 78-104"))) |>
  build_table(dat)
```

```text
               Cumulative follow-up                              Sliding windows               
   <= 32 wks   <= 64 wks   <= 96 wks      All       Wk 0-52    Wk 26-78   Wk 52-104   Wk 78-104
   (N=7145)    (N=11910)   (N=15885)   (N=16800)   (N=10248)   (N=7805)   (N=6728)    (N=3169) 
———————————————————————————————————————————————————————————————————————————————————————————————
```


### 145. Periodic Safety Update (DSUR / PBRER) Reporting Interval and Cumulative-to-Date Nested by Treatment Arm

`id: argon_proposal#A7` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- subset(ex_adae, ARM %in% c("A: Drug X", "B: Placebo"))

dat$ARM <- droplevels(dat$ARM)

dat$ADY_MO <- ceiling(dat$ASTDY/30)

horizon_split <- make_split_fun(core_split = function(spl, df, vals, labels, .spl_context) {
    make_split_result(values = c("curr_yr", "cum_all"), labels = c(curr_yr = "Reporting Interval", 
        cum_all = "Cumulative to Date"), datasplit = list(curr_yr = df[df$ADY_MO <= 
        12, , drop = FALSE], cum_all = df), subset_exprs = list(quote(ADY_MO <= 12), 
        quote(TRUE)))
})
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(
    "ARM",
    split_fun = add_overall_level("Total", label = "Total", first = FALSE),
    show_colcounts = TRUE
  ) |>
  split_cols_by("STUDYID", split_fun = horizon_split, show_colcounts = TRUE) |>
  build_table(dat)
```

```text
                  A: Drug X                                B: Placebo                                   Total                 
                   (N=609)                                   (N=622)                                  (N=1231)                
   Reporting Interval   Cumulative to Date   Reporting Interval   Cumulative to Date   Reporting Interval   Cumulative to Date
        (N=373)              (N=609)              (N=403)              (N=622)              (N=776)              (N=1231)     
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 146. Predicate-Defined Subpopulation Columns (Multi-Variable, Overlapping)

`id: claude_proposal#P3` | Source: `claude_col_proposals_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

predicate_cols <- function(preds) {
    make_split_fun(core_split = function(spl, df, vals, labels, trim, .spl_context) {
        make_split_result(values = names(preds), labels = names(preds), datasplit = lapply(preds, 
            function(p) df[eval(p, df), , drop = FALSE]), subset_exprs = unname(preds))
    })
}

pops <- list(Safety = quote(SAFFL == "Y"), ITT = quote(ITTFL == "Y"), `Age >= 35` = quote(AGE >= 
    35), `Biomarker 2 High` = quote(BMRKR2 == "HIGH"))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("STUDYID", split_fun = predicate_cols(pops)) |>
  build_table(ex_adsl)
```

```text
   Safety      ITT     Age >= 35   Biomarker 2 High
   (N=400)   (N=400)    (N=195)        (N=130)     
———————————————————————————————————————————————————
```


### 147. Overlapping Analysis Populations (Predicate Spans) Crossed with Treatment Arms

`id: argon_proposal#A6b` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

set.seed(10)

dat <- subset(ex_adsl, ARM %in% c("A: Drug X", "B: Placebo"))

dat$ARM <- droplevels(dat$ARM)

dat$ITTFL <- "Y"

dat$PPROTFL <- ifelse(runif(nrow(dat)) > 0.12, "Y", "N")

dat$BMFL <- ifelse(dat$BMRKR1 > median(dat$BMRKR1), "Y", "N")

predicate_cols <- function(preds) {
    make_split_fun(core_split = function(spl, df, vals, labels, .spl_context) {
        make_split_result(values = names(preds), labels = names(preds), datasplit = lapply(preds, 
            function(p) df[eval(p, df), , drop = FALSE]), subset_exprs = unname(preds))
    })
}

pops <- list(`ITT Population` = quote(ITTFL == "Y"), `Per-Protocol Population` = quote(PPROTFL == 
    "Y"), `Biomarker-High Subset` = quote(BMFL == "Y"))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("STUDYID", split_fun = predicate_cols(pops)) |>
  split_cols_by("ARM") |>
  build_table(dat)
```

```text
       ITT Population        Per-Protocol Population      Biomarker-High Subset  
   A: Drug X   B: Placebo    A: Drug X     B: Placebo    A: Drug X    B: Placebo 
    (N=134)     (N=134)       (N=123)       (N=113)        (N=71)       (N=63)   
—————————————————————————————————————————————————————————————————————————————————
```


### 148. Interleaved Comparison Columns and Duplicated-Control Comparison Blocks

`id: claude_proposal#P4` | Source: `claude_col_proposals_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- ex_adsl
dat$ARM <- factor(as.character(dat$ARM), levels = levels(dat$ARM)[c(1, 3, 2)])

parent_fun <- make_split_fun(post = list(add_combo_facet("AvsPlacebo", "A vs Placebo", 
    levels = c("A: Drug X", "B: Placebo")), add_combo_facet("CombovsPlacebo", "Combo vs Placebo", 
    levels = c("C: Combination", "B: Placebo")), restrict_facets(c("AvsPlacebo", 
    "CombovsPlacebo"), op = "keep"), function(ret, spl, fulldf) {
    ret$datasplit <- lapply(ret$datasplit, function(dat) {
        dat$ARM <- factor(dat$ARM)
        dat
    })
    ret
}))

comp_block_child <- make_split_fun(post = list(function(ret, spl, fulldf) {
    add_to_split_result(ret, values = "Difference", labels = "Difference", datasplit = list(Difference = fulldf), 
        subset_exprs = list(quote(TRUE)))
}))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM", split_fun = parent_fun) |>
  split_cols_by("ARM", split_fun = comp_block_child) |>
  build_table(dat)
```

```text
              A vs Placebo                           Combo vs Placebo            
   A: Drug X   B: Placebo   Difference   C: Combination   B: Placebo   Difference
    (N=134)     (N=134)      (N=268)        (N=132)        (N=134)      (N=266)  
—————————————————————————————————————————————————————————————————————————————————
```


### 149. Interleaved Within-Arm Comparison Columns (Control Summary vs. Active Arm Summary and Diff vs. Control)

`id: argon_proposal#A3` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

arm_interleaved_subcols <- function(ctrl_arm = "B: Placebo") {
    make_split_fun(core_split = function(spl, df, vals, labels, .spl_context) {
        parent_arm <- tail(.spl_context$value, 1)
        if (identical(parent_arm, ctrl_arm)) {
            make_split_result(values = "summary", datasplit = list(summary = df), 
                labels = c(summary = "Summary"), subset_exprs = list(quote(TRUE)))
        }
        else {
            make_split_result(values = c("summary", "diff"), datasplit = list(summary = df, 
                diff = df), labels = c(summary = "Summary", diff = "Diff vs. Placebo"), 
                subset_exprs = list(quote(TRUE), quote(TRUE)))
        }
    })
}

dat <- ex_adsl

dat$ARM <- factor(as.character(dat$ARM), levels = c("B: Placebo", "A: Drug X", "C: Combination"))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by("STUDYID", split_fun = arm_interleaved_subcols("B: Placebo")) |>
  build_table(dat)
```

```text
   B: Placebo           A: Drug X                  C: Combination      
    (N=134)              (N=134)                      (N=132)          
    Summary     Summary   Diff vs. Placebo   Summary   Diff vs. Placebo
———————————————————————————————————————————————————————————————————————
```


### 150. Observational / Propensity-Score Adjustment Stages with In-Stage Balance Diagnostics (Std. Diff)

`id: argon_proposal#A8` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

set.seed(99)

dat <- subset(ex_adsl, ARM %in% c("A: Drug X", "B: Placebo"))

dat$ARM <- factor(ifelse(dat$ARM == "A: Drug X", "Treated", "Control"), levels = c("Treated", 
    "Control"))

trt_idx <- which(dat$ARM == "Treated")[1:90]

ctrl_idx <- which(dat$ARM == "Control")[1:90]

dat$MATCHED <- FALSE

dat$MATCHED[c(trt_idx, ctrl_idx)] <- TRUE

cohort_stage_split <- make_split_fun(core_split = function(spl, df, vals, labels, 
    .spl_context) {
    make_split_result(values = c("unadj", "matched"), labels = c(unadj = "Before Matching (Unadjusted)", 
        matched = "After 1:1 PS Matching"), datasplit = list(unadj = df, matched = df[df$MATCHED, 
        , drop = FALSE]), subset_exprs = list(quote(TRUE), quote(MATCHED == TRUE)))
})

add_smd_col <- make_split_fun(post = list(function(ret, spl, fulldf) {
    add_to_split_result(ret, values = "smd", labels = c(smd = "Std. Diff"), datasplit = list(smd = fulldf), 
        subset_exprs = quote(TRUE))
}))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("STUDYID", split_fun = cohort_stage_split) |>
  split_cols_by("ARM", split_fun = add_smd_col, show_colcounts = TRUE) |>
  build_table(dat)
```

```text
      Before Matching (Unadjusted)          After 1:1 PS Matching    
    Treated     Control    Std. Diff    Treated   Control   Std. Diff
    (N=134)     (N=134)     (N=268)     (N=90)    (N=90)     (N=180) 
—————————————————————————————————————————————————————————————————————
```


### 151. Variable-Depth Sibling Branches

`id: claude_proposal#P5` | Source: `claude_col_proposals_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  add_overall_col("All Patients") |>
  split_cols_by("ARM", split_fun = keep_split_levels(c("A: Drug X", "B: Placebo")), nested = FALSE) |>
  split_cols_by("SEX", split_fun = keep_split_levels(c("F", "M"))) |>
  build_table(ex_adsl)
```

```text
                    A: Drug X        B: Placebo   
  All Patients     F        M        F        M   
    (N=400)      (N=79)   (N=51)   (N=77)   (N=55)
———————————————————————————————————————————————————
```


### 152. Top-k Columns Plus a Residual "Other" Bucket, Ordered by Frequency

`id: claude_proposal#P6` | Source: `claude_col_proposals_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

top_k_plus_other <- function(k, other_lbl = "Other") {
    make_split_fun(post = list(function(ret, spl, fulldf) {
        var <- spl_variable(spl)
        ns <- vapply(ret$datasplit, NROW, 1L)
        ord <- order(ns, decreasing = TRUE)
        keep <- ord[seq_len(min(k, length(ord)))]
        rest <- setdiff(ord, keep)
        out <- make_split_result(values = ret$values[keep], labels = ret$labels[keep], 
            datasplit = ret$datasplit[keep], subset_exprs = replicate(length(keep), 
                list(NULL)))
        if (length(rest)) {
            out <- add_combo_facet(other_lbl, other_lbl, names(ret$values)[rest])(out, 
                spl, fulldf)
        }
        out
    }))
}
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("COUNTRY", split_fun = top_k_plus_other(4)) |>
  add_overall_col("Total") |>
  build_table(ex_adsl)
```

```text
     CHN      USA      PAK      BRA     Other     Total 
   (N=219)   (N=40)   (N=31)   (N=30)   (N=80)   (N=400)
————————————————————————————————————————————————————————
```


### 153. Declared Complete Level Sets (Empty Columns and an Explicit Missing Column)

`id: claude_proposal#P7` | Source: `claude_col_proposals_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- ex_adae
dat$AETOXGR[1:2] <- NA
grade_lvls <- c(paste("Grade", 0:5), "Missing")
dat$ATOXGR_LBL <- factor(paste("Grade", dat$AETOXGR), levels = grade_lvls)
na_to_missing <- function(df) {
  v <- as.character(df$ATOXGR_LBL)
  v[is.na(v)] <- "Missing"
  df$ATOXGR_LBL <- factor(v, levels = grade_lvls)
  df
}
dat <- na_to_missing(dat)

grade_lvls <- c(paste("Grade", 0:5), "Missing")

na_to_missing <- function(df) {
    v <- as.character(df$ATOXGR_LBL)
    v[is.na(v)] <- "Missing"
    df$ATOXGR_LBL <- factor(v, levels = grade_lvls)
    df
}
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM", split_fun = keep_split_levels("A: Drug X")) |>
  split_cols_by("ATOXGR_LBL") |>
  build_table(dat)
```

```text
                                A: Drug X                             
   Grade 0   Grade 1   Grade 2   Grade 3   Grade 4   Grade 5   Missing
    (N=0)    (N=190)   (N=116)   (N=129)   (N=55)    (N=117)    (N=2) 
——————————————————————————————————————————————————————————————————————
```


### 154. Multi-Arm Laboratory / Toxicity Shift Matrix with Complete Scale Levels and Within-Arm Totals

`id: argon_proposal#A9` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

set.seed(7)

dat <- subset(ex_adsl, ARM %in% c("A: Drug X", "B: Placebo"))

dat$ARM <- droplevels(dat$ARM)

shift_lvls <- c("Low", "Normal", "High", "Missing")

dat$ANRIND_PB <- factor(ifelse(dat$ARM == "A: Drug X", sample(c("Normal", "High", 
    "Missing"), nrow(dat), replace = TRUE, prob = c(0.75, 0.2, 0.05)), sample(c("Low", 
    "Normal", "High", "Missing"), nrow(dat), replace = TRUE, prob = c(0.1, 0.7, 0.15, 
    0.05))), levels = shift_lvls)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by(
    "ANRIND_PB",
    split_fun = add_overall_level("Total", label = "Total", first = FALSE),
    show_colcounts = TRUE
  ) |>
  build_table(dat)
```

```text
                    A: Drug X                                      B: Placebo                 
                     (N=134)                                        (N=134)                   
    Low    Normal     High    Missing    Total     Low     Normal    High    Missing    Total 
   (N=0)   (N=101)   (N=27)    (N=6)    (N=134)   (N=16)   (N=95)   (N=17)    (N=6)    (N=134)
——————————————————————————————————————————————————————————————————————————————————————————————
```


### 155. Pooled Replica of a Subtree (Integrated Summaries)

`id: claude_proposal#P8` | Source: `claude_col_proposals_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat1 <- ex_adsl
dat2 <- ex_adsl
dat2$STUDYID <- "AB54321"
dat <- rbind(dat1, dat2)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("STUDYID", split_fun = add_overall_level("Pooled", label = "Pooled Studies", first = FALSE)) |>
  split_cols_by("ARM") |>
  build_table(dat)
```

```text
                   AB12345                                   AB54321                               Pooled Studies             
   A: Drug X   B: Placebo   C: Combination   A: Drug X   B: Placebo   C: Combination   A: Drug X   B: Placebo   C: Combination
    (N=134)     (N=134)        (N=132)        (N=134)     (N=134)        (N=132)        (N=268)     (N=268)        (N=264)    
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 156. Two-Way Hierarchical Marginals (Parent Subtotals, Child Subtotals, and Grand Total Marginals)

`id: argon_proposal#A5` | Source: `argon_col_proposals.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)

dat <- subset(ex_adsl, COUNTRY %in% c("CHN", "USA", "BRA") & ARM %in% c("A: Drug X", 
    "B: Placebo"))

dat$REGION <- factor(ifelse(dat$COUNTRY == "CHN", "Asia", "Americas"), levels = c("Asia", 
    "Americas"))

dat$ARM <- droplevels(dat$ARM)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("REGION", split_fun = add_overall_level("All_Regions", label = "All Regions", first = FALSE)) |>
  split_cols_by("ARM", split_fun = add_overall_level("Total", label = "Total", first = FALSE)) |>
  build_table(dat)
```

```text
                 Asia                            Americas                         All Regions           
   A: Drug X   B: Placebo    Total    A: Drug X   B: Placebo   Total    A: Drug X   B: Placebo    Total 
    (N=74)       (N=81)     (N=155)    (N=23)       (N=20)     (N=43)    (N=97)      (N=101)     (N=198)
————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 157. Very Wide Headers That Need Column Pagination

`id: claude_proposal#P9` | Source: `claude_col_proposals_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(dplyr)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by("AVISIT") |>
  build_table(ex_advs)
```

```text
Page 1:
                                                A: Drug X                                                B: Placebo
   SCREENING   BASELINE   WEEK 1 DAY 8   WEEK 2 DAY 15   WEEK 3 DAY 22   WEEK 4 DAY 29   WEEK 5 DAY 36   SCREENING 
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Page 2:
                                         B: Placebo                                             C: Combination   
   BASELINE   WEEK 1 DAY 8   WEEK 2 DAY 15   WEEK 3 DAY 22   WEEK 4 DAY 29   WEEK 5 DAY 36   SCREENING   BASELINE
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Page 3:
                                  C: Combination                               
   WEEK 1 DAY 8   WEEK 2 DAY 15   WEEK 3 DAY 22   WEEK 4 DAY 29   WEEK 5 DAY 36
———————————————————————————————————————————————————————————————————————————————
```


---

# Section 2: `junco` Column Structures (Requires `junco`)



### 1. Least-squares means approach

`id: vig:ancova_combined#4` | Source: `ancova_combined.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adchg <- tibble(USUBJID = sprintf("SUBJ-%03d", 1:300), TRT01A = factor(rep(c("Placebo", 
    "Low Dose", "High Dose"), length.out = 300)), REGION = factor(sample(c("EU", 
    "US"), 300, replace = TRUE, prob = c(0.6, 0.4))), BASE = rnorm(300, 50, 10), 
    CHG = rnorm(300, 0, 8))

combodf <- tibble::tribble(~valname, ~label, ~levelcombo, ~exargs, "ACTIVE", "Active", 
    c("Low Dose", "High Dose"), list())
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("TRT01A", ref_group = "Placebo", split_fun = add_combo_levels(combodf)) |>
  add_colcounts() |>
  build_table(adchg)
```

```text
High Dose             Low Dose               Placebo               Active       
                                              (N=100)               (N=100)               (N=100)               (N=200)      
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 2. Examples

`id: vig:auto_colwidths#1` | Source: `auto_colwidths.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

adsl2 <- ex_adsl

adsl2$ARM2 <- adsl2$ARM

levels(adsl2$ARM2) <- c("A", "B", "C")

adsl2$ARM3 <- adsl2$ARM

levels(adsl2$ARM3) <- c("Full Drug Name Of Drug X", "Current Best-Practice Standard Of Care", 
    "The Weird Other Arm")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  build_table(adsl2)
```

```text
A: Drug X    B: Placebo    C: Combination
————————————————————————————————————————————————————————————————————————————
```


### 3. Examples

`id: vig:auto_colwidths#2` | Source: `auto_colwidths.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

adsl2 <- ex_adsl

adsl2$ARM2 <- adsl2$ARM

levels(adsl2$ARM2) <- c("A", "B", "C")

adsl2$ARM3 <- adsl2$ARM

levels(adsl2$ARM3) <- c("Full Drug Name Of Drug X", "Current Best-Practice Standard Of Care", 
    "The Weird Other Arm")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", labels_var = "ARM2") |>
  build_table(adsl2)
```

```text
A             B             C     
—————————————————————————————————————————————————————————————————————————
```


### 4. Examples

`id: vig:auto_colwidths#3` | Source: `auto_colwidths.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

adsl2 <- ex_adsl

adsl2$ARM2 <- adsl2$ARM

levels(adsl2$ARM2) <- c("A", "B", "C")

adsl2$ARM3 <- adsl2$ARM

levels(adsl2$ARM3) <- c("Full Drug Name Of Drug X", "Current Best-Practice Standard Of Care", 
    "The Weird Other Arm")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM", labels_var = "ARM3") |>
  build_table(adsl2)
```

```text
Full Drug Name Of Drug X   Current Best-Practice Standard Of Care   The Weird Other Arm
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 5. Working Example 1: Table with Multiple Variables Analyzed with Different Precision (e.g. Demography Table)

`id: vig:formats_d#8` | Source: `formats_d.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(tern)

library(dplyr)

f2n <- function(x) as.numeric(x) - 1

df <- ex_adsl

df <- mutate(df, newvar = rnorm(nrow(df), mean = 50 + 3 * f2n(ARMCD) + 4 * AGE, sd = 20))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARMCD") |>
  build_table(df)
```

```text
ARM A             ARM B             ARM C     
————————————————————————————————————————————————————————————————————————
```


### 6. Working Example 4: Table with Multiple Parameters Analyzed with Different Precision, with statistics presented in columns (e.g. Lab/Vital Signs Table)

`id: vig:formats_d#11` | Source: `formats_d.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(tern)

library(dplyr)

df_d <- tribble(~PARAMCD, ~d, "ALT", 2, "CRP", 3, "IGA", 1)

df_d <- fmt_spec_df_d(df_d, d_column = "d", fmt_column = "fmt_d", stats_in = NULL, 
    fmt_d_def = junco_def_d_all, fmt_d_in = NULL)

df <- filter(ex_adlb, AVISIT %in% c("BASELINE", "WEEK 1 DAY 8", "WEEK 2 DAY 15"))

df <- left_join(df, df_d)

mysplitfun <- make_split_fun(post = list(stats_in_cols_setup()))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_multivar(c("AVAL", "BASE", "CHG"), varlabels = c("Visit Values", " ", 
      "Change from Baseline")) |>
  split_cols_by("STUDYID", split_fun = mysplitfun) |>
  build_table(df, alt_counts_df = ex_adsl)
```

```text
Visit Values                                                                     Change from Baseline                      
                                        N     Mean       SD        SE        Med      Min      Max      Base Mean (SD)      N     Mean        SD        SE        Med       Min      Max  
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 7. Example as in Tern TLG Catalog using `analyze_vars_in_cols`

`id: vig:formats_significant_figures#5` | Source: `formats_significant_figures.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl <- filter(random.cdisc.data::cadsl, ACTARM == "A: Drug X")

adpc <- filter(random.cdisc.data::cadpc, ACTARM == "A: Drug X", PARAM %in% c("Plasma Drug X", 
    "Urine Drug X"))

adpc_1 <- select(mutate(adpc, NFRLT = as.factor(NFRLT), AVALCAT1 = as.factor(AVALCAT1), 
    NOMTPT = as.factor(paste(NFRLT, "/", PCTPT))), STUDYID, NOMTPT, ACTARM, VISIT, 
    AVAL, PARAM, AVALCAT1)

adpc_1$NOMTPT <- factor(adpc_1$NOMTPT, levels = levels(adpc_1$NOMTPT)[order(as.numeric(gsub(".*?([0-9\\.]+).*", 
    "\\1", levels(adpc_1$NOMTPT))))])
```

**Core layout instructions:**

```r
basic_table() |>
  build_table(df = adpc_1, alt_counts_df = adsl)
```

```text
Parameter                                                                                                                                                   
  Treatment Group                            Number                                                                                                         
    Visit                                      of                                                                                                           
      Nominal Time (hr) / Timepoint    n    LTRs/BLQs    Mean       SD     CV (%) Mean   Geometric Mean   CV % Geometric Mean   Median    Minimum    Maximum
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 8. Same Example with `junco` afun `column_stats`

`id: vig:formats_significant_figures#9` | Source: `formats_significant_figures.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adpc <- filter(random.cdisc.data::cadpc, ACTARM == "A: Drug X", PARAM %in% c("Plasma Drug X", 
    "Urine Drug X"))

adpc_1 <- select(mutate(adpc, NFRLT = as.factor(NFRLT), AVALCAT1 = as.factor(AVALCAT1), 
    NOMTPT = as.factor(paste(NFRLT, "/", PCTPT))), STUDYID, NOMTPT, ACTARM, VISIT, 
    AVAL, PARAM, AVALCAT1)

adpc_1$NOMTPT <- factor(adpc_1$NOMTPT, levels = levels(adpc_1$NOMTPT)[order(as.numeric(gsub(".*?([0-9\\.]+).*", 
    "\\1", levels(adpc_1$NOMTPT))))])

mysplitfun <- make_split_fun(post = list(stats_in_cols_setup()))

adpc_1 <- filter(adpc_1, VISIT == "Day 1", !(NOMTPT %in% "0 / Predose"))

adpc_fmt <- tribble(~PARAM, ~fmt_d, "Plasma Drug X", list(mean = format_sigfig_j(3), 
    sd = format_sigfig_j(3), se = format_sigfig_j(3), median = format_sigfig_j(3), 
    min = format_sigfig_j(3), max = format_sigfig_j(3)), "Urine Drug X", list(mean = format_sigfig_j(2), 
    sd = format_sigfig_j(2), se = format_sigfig_j(2), median = format_sigfig_j(2), 
    min = format_sigfig_j(2), max = format_sigfig_j(2)))

adpc_2 <- left_join(adpc_1, adpc_fmt)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by_multivar(c("AVAL"), varlabels = c("Visit Values")) |>
  split_cols_by("STUDYID", split_fun = mysplitfun) |>
  build_table(adpc_2, round_type = "sas")
```

```text
Visit Values                        
                       N     Mean      SD       SE       Med       Min      Max  
—————————————————————————————————————————————————————————————————————————————————
```


### 9. Working Example: AE Table with Risk Difference Columns

`id: vig:get_ref_info#9` | Source: `get_ref_info.md`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` to directly generate the grouped treatment arms and pairwise risk difference columns. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

trtvar <- "TRT01A"

ctrl_grp <- "Placebo"

adsl <- mutate(select(filter(pharmaverseadamjnj::adsl, SAFFL == "Y"), STUDYID, USUBJID, 
    all_of(trtvar), SAFFL), `:=`(!!trtvar, factor(.data[[trtvar]], levels = c("Xanomeline Low Dose", 
    "Xanomeline High Dose", "Placebo"))))

adae <- select(filter(pharmaverseadamjnj::adae, TRTEMFL == "Y"), USUBJID, TRTEMFL, 
    AEBODSYS, AEDECOD)

adsl <- create_colspan_var(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

ae <- right_join(adae, adsl, by = "USUBJID")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE, colcount_format = "N=xx") |>
  grouped_cols_w_diffs(colspan_trt_map, diffs_label = "Risk Difference (%) (95% CI)") |>
  build_table(ae, alt_counts_df = adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

library(dplyr)

trtvar <- "TRT01A"

ctrl_grp <- "Placebo"

adsl <- mutate(select(filter(pharmaverseadamjnj::adsl, SAFFL == "Y"), STUDYID, USUBJID, 
    all_of(trtvar), SAFFL), `:=`(!!trtvar, factor(.data[[trtvar]], levels = c("Xanomeline Low Dose", 
    "Xanomeline High Dose", "Placebo"))))

adae <- select(filter(pharmaverseadamjnj::adae, TRTEMFL == "Y"), USUBJID, TRTEMFL, 
    AEBODSYS, AEDECOD)

adsl$colspan_trt <- factor(ifelse(adsl[[trtvar]] == ctrl_grp, " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

adsl$rrisk_header <- "Risk Difference (%) (95% CI)"

adsl$rrisk_label <- paste(adsl[[trtvar]], "vs", ctrl_grp)

ae <- right_join(adae, adsl, by = "USUBJID")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)
```

**Fallback core layout instructions:**

```r
basic_table(show_colcounts = TRUE, colcount_format = "N=xx") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by(trtvar) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by(trtvar, labels_var = "rrisk_label", split_fun = remove_split_levels(ctrl_grp)) |>
  build_table(ae, alt_counts_df = adsl)
```

</details>

```text
Active Study Agent                                              Risk Difference (%) (95% CI)                  
System Organ Class                       Xanomeline Low Dose   Xanomeline High Dose    Placebo     Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Placebo
  Preferred Term, n (%)                         N=67                   N=51              N=64                   N=67                             N=51              
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```

### 10. Tabulation Examples using `a_freq_j`

`id: vig:junco#6` | Source: `junco.md`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` to directly generate grouped treatment arms and risk difference columns. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl <- ex_adsl

adae <- ex_adae

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

adsl <- create_colspan_var(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

adae[["TRTEMFL"]] <- "Y"

adae <- left_join(adae, adsl, by = c("USUBJID", trtvar))
```

**Core layout instructions:**

```r
basic_table(top_level_section_div = " ", colcount_format = "N=xx") |>
  grouped_cols_w_diffs(colspan_trt_map, diffs_label = "Risk Difference (%) (95% CI)") |>
  build_table(adae, alt_counts_df = adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl <- ex_adsl

adae <- ex_adae

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

adsl$colspan_trt <- factor(ifelse(adsl[[trtvar]] == ctrl_grp, " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

adsl$rrisk_header <- "Risk Difference (%) (95% CI)"

adsl$rrisk_label <- paste(adsl[[trtvar]], paste("vs", ctrl_grp))

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

adae[["TRTEMFL"]] <- "Y"

adae <- left_join(adae, adsl, by = c("USUBJID", trtvar))
```

**Fallback core layout instructions:**

```r
basic_table(top_level_section_div = " ", colcount_format = "N=xx") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by(trtvar) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by(trtvar, labels_var = "rrisk_label", split_fun = remove_split_levels(ctrl_grp)) |>
  build_table(adae, alt_counts_df = adsl)
```

</details>

```text
Active Study Agent                                                                            
                              A: Drug X    C: Combination   B: Placebo                 Risk Difference (%) (95% CI)             
System Organ Class              N=134          N=132           N=134      A: Drug X vs B: Placebo   C: Combination vs B: Placebo
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 11. Minor differences between current table and a similar table using `tern` analyze functions.

`id: vig:junco#7` | Source: `junco.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(tern)

library(dplyr)

adsl <- ex_adsl

adae <- ex_adae

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

adsl$colspan_trt <- factor(ifelse(adsl[[trtvar]] == ctrl_grp, " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

adsl$rrisk_header <- "Risk Difference (%) (95% CI)"

adsl$rrisk_label <- paste(adsl[[trtvar]], paste("vs", ctrl_grp))

adsl$rrisk_header_vs <- "Difference in Mean Change (95% CI)"

adae[["TRTEMFL"]] <- "Y"

adae <- adae %>% left_join(., adsl)
```

**Core layout instructions:**

```r
basic_table(top_level_section_div = " ", colcount_format = "N=xx") |>
  build_table(adae, alt_counts_df = adsl)
```

```text
Risk Difference (%) (95% CI)   Risk Difference (%) (95% CI) 
                                                            A: Drug X    B: Placebo    C: Combination     B: Placebo vs. A: Drug X     B: Placebo vs. C: Combination
System Organ Class                                            N=134         N=134          N=132                   N=268                           N=266            
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 12. Basic Usage

`id: vig:standard_column_structures#1` | Source: `standard_column_structures.md`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` to directly generate grouped treatment arms and pairwise risk difference columns from `create_colspan_map()`. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

adsl <- subset(pharmaverseadamjnj::adsl, !is.na(TRT01A))

adsl <- create_colspan_var(adsl)

trt_map <- create_colspan_map(adsl)
```

**Core layout instructions:**

```r
basic_table() |>
  grouped_cols_w_diffs(trt_map) |>
  build_table(adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

adsl <- subset(pharmaverseadamjnj::adsl, !is.na(TRT01A))

adsl$colspan_trt <- factor(ifelse(adsl$TRT01A == "Placebo", " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

trt_map <- create_colspan_map(adsl)

adsl$diffs_label <- "Risk Differences"

adsl$rrisk_label <- paste(adsl$TRT01A, "vs Placebo")
```

**Fallback core layout instructions:**

```r
basic_table() |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = trt_map)) |>
  split_cols_by("TRT01A") |>
  split_cols_by("diffs_label", nested = FALSE) |>
  split_cols_by("TRT01A", labels_var = "rrisk_label", split_fun = remove_split_levels("Placebo")) |>
  build_table(adsl)
```

</details>

```text
Active Study Agent                                                 Risk Differences                        
   Xanomeline Low Dose   Xanomeline High Dose   Placebo   Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Placebo
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 13. A Note About Types Of Differences

`id: vig:standard_column_structures#2` | Source: `standard_column_structures.md`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs(..., diffs_label = "Mean Differences")` to directly generate grouped treatment arms and difference columns with custom spanner labels. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

adsl <- subset(pharmaverseadamjnj::adsl, !is.na(TRT01A))

adsl <- create_colspan_var(adsl)

trt_map <- create_colspan_map(adsl)
```

**Core layout instructions:**

```r
basic_table() |>
  grouped_cols_w_diffs(trt_map, diffs_label = "Mean Differences") |>
  build_table(adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

adsl <- subset(pharmaverseadamjnj::adsl, !is.na(TRT01A))

adsl$colspan_trt <- factor(ifelse(adsl$TRT01A == "Placebo", " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

trt_map <- create_colspan_map(adsl)

adsl$diffs_label <- "Mean Differences"

adsl$rrisk_label <- paste(adsl$TRT01A, "vs Placebo")
```

**Fallback core layout instructions:**

```r
basic_table() |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = trt_map)) |>
  split_cols_by("TRT01A") |>
  split_cols_by("diffs_label", nested = FALSE) |>
  split_cols_by("TRT01A", labels_var = "rrisk_label", split_fun = remove_split_levels("Placebo")) |>
  build_table(adsl)
```

</details>

```text
Active Study Agent                                                 Mean Differences                        
   Xanomeline Low Dose   Xanomeline High Dose   Placebo   Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Placebo
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 14. a_combo_prop_diff_pval_mf works as expected

`id: test:test-a_combo_prop_diff_pval_mf:65` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_combo_prop_diff_pval_mf.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

adrs_f <- formatters::ex_adrs

adrs_f <- subset(adrs_f, ARM %in% c("A: Drug X", "B: Placebo") & PARAMCD == "BESRSPI")

adrs_f$ARM <- droplevels(adrs_f$ARM)

adrs_f$PARAMCD <- droplevels(adrs_f$PARAMCD)

adrs_f$rsp <- factor(ifelse(adrs_f$AVALC == "CR", "Y", "N"))

adrs_f$ARM_FOR_RISKCOLS <- adrs_f$ARM

riskcols_combodf <- data.frame(valname = c("diff", "pval"), label = c("Risk Difference", 
    "p-value"), levelcombo = c("A: Drug X", "B: Placebo"), exargs = I(list(list(), 
    list())))

col_split_fun <- add_combo_levels(riskcols_combodf, keep_levels = riskcols_combodf$valname)
```

**Core layout instructions:**

```r
basic_table(top_level_section_div = " ") |>
  split_cols_by("ARM") |>
  split_cols_by("ARM_FOR_RISKCOLS", split_fun = col_split_fun, nested = FALSE) |>
  build_table(adrs_f)
```

```text
A: Drug X        B: Placebo     Risk Difference    p-value  
———————————————————————————————————————————————————————————————————————————————————
```


### 15. a_eair_strat_j: rate difference column is produced with riskdiff = TRUE

`id: test:test-a_eair_strat_j:143` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_eair_strat_j.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl_strat <- select(mutate(ex_adsl, TRTDURY = as.numeric(sub("d", "", sub("-", "", 
    substring(USUBJID, nchar(USUBJID) - 2)))), STUDY = rep(c("Study1", "Study2", 
    "Study3"), length.out = n())), USUBJID, ARM, STUDY, TRTDURY)

adae_strat <- mutate(filter(ungroup(slice(group_by(arrange(mutate(select(ex_adae, 
    USUBJID, AEDECOD, ASTDY), AOCCPFL = "Y"), USUBJID, AEDECOD, ASTDY), USUBJID, 
    AEDECOD), 1)), AEDECOD %in% c("dcd A.1.1.1.1", "dcd A.1.1.1.2", "dcd B.1.1.1.1", 
    "dcd B.2.1.2.1")), AEDECOD = droplevels(AEDECOD))

aefup_strat <- left_join(adsl_strat, adae_strat, by = "USUBJID")
```

**Core layout instructions:**

```r
basic_table(show_colcounts = FALSE) |>
  split_cols_by("ARM") |>
  build_table(aefup_strat, alt_counts_df = adsl_strat)
```

```text
A: Drug X        B: Placebo     C: Combination   
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 16. a_eair_strat_j: matches Scosyrev & Pethe (2022) Table 2 study size-adjusted EAIR

`id: test:test-a_eair_strat_j:254` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_eair_strat_j.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

make_arm_data <- function(study, arm, n_event, total_py, n_total) {
    data.frame(USUBJID = paste(study, arm, seq_len(n_total), sep = "_"), ARM = arm, 
        STUDY = study, TRTDURY = total_py/n_total, AOCCPFL = c(rep("Y", n_event), 
            rep(NA_character_, n_total - n_event)), stringsAsFactors = FALSE)
}

ph2_trt <- make_arm_data("Phase2", "New Treatment", 30, 150, 150)

ph2_pbo <- make_arm_data("Phase2", "Placebo", 10, 50, 50)

ph3_trt <- make_arm_data("Phase3", "New Treatment", 133, 700, 700)

ph3_pbo <- make_arm_data("Phase3", "Placebo", 67, 350, 350)

ph3r_trt <- make_arm_data("Phase3R", "New Treatment", 200, 500, 500)

ph3r_pbo <- make_arm_data("Phase3R", "Placebo", 200, 500, 500)

adsl_paper <- rbind(ph2_trt, ph2_pbo, ph3_trt, ph3_pbo, ph3r_trt, ph3r_pbo)

adsl_paper$ARM <- factor(adsl_paper$ARM, levels = c("New Treatment", "Placebo"))

adae_paper <- adsl_paper[!is.na(adsl_paper$AOCCPFL), c("USUBJID", "STUDY", "ARM")]

adae_paper$AEDECOD <- "AE"
```

**Core layout instructions:**

```r
basic_table(show_colcounts = FALSE) |>
  split_cols_by("ARM") |>
  build_table(adae_paper, alt_counts_df = adsl_paper[, c("USUBJID", "ARM", "STUDY", "TRTDURY")])
```

```text
New Treatment   Placebo
—————————————————————————————————————————
```


### 17. a_eair_strat_j: matches Scosyrev & Pethe (2022) Table 3 OccR rate difference (Asthma)

`id: test:test-a_eair_strat_j:338` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_eair_strat_j.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

make_arm_t3 <- function(arm, n_total, total_pm, n_event, aedecod) {
    data.frame(USUBJID = paste(arm, seq_len(n_total), sep = "_"), ARM = arm, STUDY = "Trial1", 
        TRTDURY = total_pm/n_total, AOCCPFL = c(rep("Y", n_event), rep(NA_character_, 
            n_total - n_event)), AEDECOD = aedecod, stringsAsFactors = FALSE)
}

trt_asthma <- make_arm_t3("Treatment", 396, 1189, 25, "Asthma")

ctl_asthma <- make_arm_t3("Control", 399, 1186, 80, "Asthma")

adsl_t3 <- rbind(trt_asthma[, c("USUBJID", "ARM", "STUDY", "TRTDURY")], ctl_asthma[, 
    c("USUBJID", "ARM", "STUDY", "TRTDURY")])

adsl_t3$ARM <- factor(adsl_t3$ARM, levels = c("Treatment", "Control"))

aefup_t3 <- rbind(trt_asthma, ctl_asthma)

aefup_t3$ARM <- factor(aefup_t3$ARM, levels = c("Treatment", "Control"))

aefup_t3$AEDECOD <- factor(aefup_t3$AEDECOD)

aefup_t3 <- aefup_t3[!is.na(aefup_t3$AOCCPFL), ]
```

**Core layout instructions:**

```r
basic_table(show_colcounts = FALSE) |>
  split_cols_by("ARM") |>
  build_table(aefup_t3, alt_counts_df = adsl_t3)
```

```text
Treatment          Control 
——————————————————————————————————————————————————————————————————————————————————————
```


### 18. a_freq_j with label_map works in a table layout as expected

`id: test:test-a_freq_j:22` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_freq_j.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

dta <- data.frame(id = 1:100, rsp = factor(sample(c(TRUE, FALSE), 100, TRUE)), grp = factor(rep(c("A", 
    "B"), each = 50), levels = c("A", "B")))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("grp") |>
  build_table(dta)
```

```text
A               B      
———————————————————————————————————————————
```


### 19. a_freq_j in specific situation error for not passing alt_counts_df

`id: test:test-a_freq_j:119` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_freq_j.R`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` to directly construct the grouped treatment headers and pairwise risk difference columns. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

trtvar <- "TRT01A"

ctrl_grp <- "Placebo"

adsl <- mutate(select(filter(pharmaverseadamjnj::adsl, SAFFL == "Y"), STUDYID, USUBJID, 
    all_of(trtvar), SAFFL), `:=`(!!trtvar, factor(.data[[trtvar]], levels = c("Xanomeline Low Dose", 
    "Xanomeline High Dose", "Placebo"))))

adsl <- create_colspan_var(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  grouped_cols_w_diffs(colspan_trt_map, diffs_label = "Risk Difference (%) (95% CI)") |>
  build_table(adsl, alt_counts_df = adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

library(dplyr)

trtvar <- "TRT01A"

ctrl_grp <- "Placebo"

adsl <- mutate(select(filter(pharmaverseadamjnj::adsl, SAFFL == "Y"), STUDYID, USUBJID, 
    all_of(trtvar), SAFFL), `:=`(!!trtvar, factor(.data[[trtvar]], levels = c("Xanomeline Low Dose", 
    "Xanomeline High Dose", "Placebo"))))

adsl$colspan_trt <- factor(ifelse(adsl[[trtvar]] == ctrl_grp, " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

adsl$rrisk_header <- "Risk Difference (%) (95% CI)"

adsl$rrisk_label <- paste(adsl[[trtvar]], "vs", ctrl_grp)

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)
```

**Fallback core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by(trtvar) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by(trtvar, labels_var = "rrisk_label", split_fun = remove_split_levels(ctrl_grp)) |>
  build_table(adsl, alt_counts_df = adsl)
```

</details>

```text
Active Study Agent                                              Risk Difference (%) (95% CI)                  
   Xanomeline Low Dose   Xanomeline High Dose   Placebo   Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Placebo
         (N=67)                 (N=51)          (N=64)                (N=67)                           (N=51)             
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```

### 20. a_freq_j in layout with relative risk column for combined facet

`id: test:test-a_freq_j:193` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_freq_j.R`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` with `combo_map_df` to directly generate virtual combination arms and their corresponding pairwise risk differences. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

cols <- c("USUBJID", "STRATA1", "EOSSTT", trtvar)

adsl <- ex_adsl[, cols]

adsl <- create_colspan_var(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

colspan_trt_map <- create_colspan_map(df = adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

combodf <- tibble::tribble(
  ~valname, ~label, ~levelcombo, ~exargs,
  "Combined", "Combined", setdiff(levels(adsl[[trtvar]]), ctrl_grp), list()
)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  grouped_cols_w_diffs(colspan_trt_map, combo_map_df = combodf, diffs_label = "Risk Difference (%) (95% CI)") |>
  build_table(adsl, alt_counts_df = adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

cols <- c("USUBJID", "STRATA1", "EOSSTT", trtvar)

adsl <- ex_adsl[, cols]

adsl$colspan_trt <- factor(ifelse(adsl[[trtvar]] == ctrl_grp, " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

adsl$rrisk_header <- "Risk Difference (%) (95% CI)"

adsl$rrisk_label <- paste(adsl[[trtvar]], paste("vs", ctrl_grp))

colspan_trt_map <- create_colspan_map(df = adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

add_combo <- add_combo_facet("Combined", label = "Combined", levels = setdiff(unique(adsl[[trtvar]]), 
    ctrl_grp))

rm_combo_from_placebo <- cond_rm_facets(facets = "Combined", ancestor_pos = NA, value = " ", 
    split = "colspan_trt")

mysplit_comb <- make_split_fun(post = list(add_combo, rm_combo_from_placebo))

rm_combo_from_placebo2 <- cond_rm_facets(facets = ctrl_grp, ancestor_pos = NA, value = "Risk Difference (%) (95% CI)", 
    split = "rrisk_header")

add_combo2 <- add_combo_facet("Combined", label = paste0("Combined vs ", ctrl_grp), 
    levels = setdiff(unique(adsl[[trtvar]]), ctrl_grp))

mysplit_comb2 <- make_split_fun(post = list(add_combo2, rm_combo_from_placebo2))
```

**Fallback core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by(trtvar, split_fun = mysplit_comb) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by(trtvar, labels_var = "rrisk_label", split_fun = mysplit_comb2) |>
  build_table(adsl, alt_counts_df = adsl)
```

</details>

```text
Active Study Agent                                                                             Risk Difference (%) (95% CI)                          
            A: Drug X   C: Combination   Combined   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo   Combined vs B: Placebo
             (N=134)       (N=132)       (N=266)     (N=134)             (N=134)                     (N=132)                     (N=266)        
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```

### 21. a_freq_j in case of data as in prior bug for cmh sato

`id: test:test-a_freq_j:333` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_freq_j.R`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` to directly construct the grouped treatment spanning headers and pairwise risk difference columns. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

tbl <- array(c(14, 15, 13, 19, 4, 0, 0, 0, 45, 32, 53, 61, 0, 8, 0, 0), dim = c(2L, 
    2L, 4L), dimnames = list(grp = c("ref", "Not-ref"), rsp = c("TRUE", "FALSE"), 
    strata = c("S1", "S2", "S3", "S4")))

df_input <- as.data.frame.table(tbl, responseName = "n")

expanded_df <- df_input[rep(seq_len(nrow(df_input)), df_input$n), c("grp", "rsp", 
    "strata")]

expanded_df$USUBJID <- rownames(expanded_df)

trtvar <- "grp"

ctrl_grp <- "ref"

expanded_df <- create_colspan_var(expanded_df, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

colspan_trt_map <- create_colspan_map(expanded_df, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)
```

**Core layout instructions:**

```r
basic_table() |>
  grouped_cols_w_diffs(colspan_trt_map, diffs_label = "Risk Difference (%) (95% CI)") |>
  build_table(expanded_df)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

tbl <- array(c(14, 15, 13, 19, 4, 0, 0, 0, 45, 32, 53, 61, 0, 8, 0, 0), dim = c(2L, 
    2L, 4L), dimnames = list(grp = c("ref", "Not-ref"), rsp = c("TRUE", "FALSE"), 
    strata = c("S1", "S2", "S3", "S4")))

df_input <- as.data.frame.table(tbl, responseName = "n")

expanded_df <- df_input[rep(seq_len(nrow(df_input)), df_input$n), c("grp", "rsp", 
    "strata")]

expanded_df$USUBJID <- rownames(expanded_df)

trtvar <- "grp"

ctrl_grp <- "ref"

expanded_df$colspan_trt <- factor(ifelse(expanded_df[[trtvar]] == ctrl_grp, " ", 
    "Active Study Agent"), levels = c("Active Study Agent", " "))

expanded_df$rrisk_header <- "Risk Difference (%) (95% CI)"

expanded_df$rrisk_label <- paste(expanded_df[[trtvar]], paste("vs", ctrl_grp))
```

**Fallback core layout instructions:**

```r
basic_table() |>
  split_cols_by("colspan_trt", split_fun = drop_split_levels) |>
  split_cols_by(trtvar, split_fun = drop_split_levels) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by(trtvar, labels_var = "rrisk_label", split_fun = keep_split_levels("Not-ref")) |>
  build_table(expanded_df)
```

</details>

```text
Active Study Agent                    Risk Difference (%) (95% CI)
            Not-ref              ref                Not-ref vs ref       
—————————————————————————————————————————————————————————————————————————
```

### 22. a_freq_resp_var_j works as expected with basic usage

`id: test:test-a_freq_resp_var_j:31` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_freq_resp_var_j.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

adsl <- ex_adsl

adrs <- data.frame(USUBJID = rep(adsl$USUBJID[1:20], each = 2), AVISIT = rep(c("WEEK 1", 
    "WEEK 2"), times = 20), ARM = rep(adsl$ARM[1:20], each = 2), COUNTRY = rep(adsl$COUNTRY[1:20], 
    each = 2), SEX = rep(adsl$SEX[1:20], each = 2), RSP = sample(c("Y", "N"), size = 40, 
    replace = TRUE, prob = c(0.3, 0.7)))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE, round_type = "sas") |>
  split_cols_by("ARM") |>
  build_table(adrs)
```

```text
A: Drug X    B: Placebo    C: Combination
                    (N=10)        (N=12)          (N=18)    
————————————————————————————————————————————————————————————
```


### 23. a_freq_resp_var_j in layout with relative risk column for combined facet

`id: test:test-a_freq_resp_var_j:309` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_freq_resp_var_j.R`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` with `combo_map_df` to directly generate virtual combination arms and their corresponding pairwise risk differences. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

cols <- c("USUBJID", "STRATA1", "EOSSTT", trtvar)

adsl <- ex_adsl[, cols]

adsl <- create_colspan_var(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

colspan_trt_map <- create_colspan_map(df = adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

combodf <- tibble::tribble(
  ~valname, ~label, ~levelcombo, ~exargs,
  "Combined", "Combined", setdiff(levels(adsl[[trtvar]]), ctrl_grp), list()
)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  grouped_cols_w_diffs(colspan_trt_map, combo_map_df = combodf, diffs_label = "Risk Difference (%) (95% CI)") |>
  build_table(adsl, alt_counts_df = adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

cols <- c("USUBJID", "STRATA1", "EOSSTT", trtvar)

adsl <- ex_adsl[, cols]

adsl$colspan_trt <- factor(ifelse(adsl[[trtvar]] == ctrl_grp, " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

adsl$rrisk_header <- "Risk Difference (%) (95% CI)"

adsl$rrisk_label <- paste(adsl[[trtvar]], paste("vs", ctrl_grp))

colspan_trt_map <- create_colspan_map(df = adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

add_combo <- add_combo_facet("Combined", label = "Combined", levels = setdiff(unique(adsl[[trtvar]]), 
    ctrl_grp))

rm_combo_from_placebo <- cond_rm_facets(facets = "Combined", ancestor_pos = NA, value = " ", 
    split = "colspan_trt")

mysplit_comb <- make_split_fun(post = list(add_combo, rm_combo_from_placebo))

rm_combo_from_placebo2 <- cond_rm_facets(facets = ctrl_grp, ancestor_pos = NA, value = "Risk Difference (%) (95% CI)", 
    split = "rrisk_header")

add_combo2 <- add_combo_facet("Combined", label = paste0("Combined vs ", ctrl_grp), 
    levels = setdiff(unique(adsl[[trtvar]]), ctrl_grp))

mysplit_comb2 <- make_split_fun(post = list(add_combo2, rm_combo_from_placebo2))
```

**Fallback core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by(trtvar, split_fun = mysplit_comb) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by(trtvar, labels_var = "rrisk_label", split_fun = mysplit_comb2) |>
  build_table(adsl, alt_counts_df = adsl)
```

</details>

```text
Active Study Agent                                                                             Risk Difference (%) (95% CI)                          
            A: Drug X   C: Combination   Combined   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo   Combined vs B: Placebo
             (N=134)       (N=132)       (N=266)     (N=134)             (N=134)                     (N=132)                     (N=266)        
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```

### 24. a_maxlev produces correct numbers for single treatment per subject

`id: test:test-a_maxlev:29` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_maxlev.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

my_adsl <- ex_adsl[, c("USUBJID", "ARM", "ACTARM")]

my_adae <- ex_adae[, c("USUBJID", "ARM", "ACTARM", "AESEV")]

my_adae <- mutate(mutate(my_adae, `:=`(AESEV, string_to_title(AESEV))), `:=`(AESEV, 
    ordered(AESEV, levels = c("Missing", "Mild", "Moderate", "Severe"))))
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("ARM") |>
  add_overall_col("Total") |>
  build_table(my_adae, alt_counts_df = my_adsl)
```

```text
A: Drug X    B: Placebo    C: Combination      Total   
—————————————————————————————————————————————————————————————————————
```


### 25. a_maxlev produces correct numbers for sequence of treatments (missing values)

`id: test:test-a_maxlev:73` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_maxlev.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

ae_severities <- c("Missing", "Mild", "Moderate", "Severe")

ae_severities <- ordered(ae_severities, levels = ae_severities)

my_adae <- data.frame(ID = c(1, 1, 1, 2, 2, 3, 3, 3, 3, 4), TRT = factor(c("a", "b", 
    "b", "b", "c", "c", "a", "c", "b", "b")), AESEV = ae_severities[c(4L, 1L, 2L, 
    1L, 2L, 1L, 2L, 3L, 1L, 2L)])

my_adsl <- data.frame(ID = rep(1:5, each = 3), TRT = factor(rep(c("a", "b", "c"), 
    times = 5)))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("TRT") |>
  add_overall_col("Total") |>
  build_table(my_adae, alt_counts_df = my_adsl)
```

```text
a           b           c         Total  
——————————————————————————————————————————————————————————
```


### 26. a_sum_ratio_j produces all 4 stats in a layout

`id: test:test-a_sum_ratio_j:22` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_sum_ratio_j.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

df <- data.frame(USUBJID = c("S01", "S01", "S02", "S03"), ARM = factor(c("A", "A", 
    "A", "B")), EVENTS = c(1, 1, 0, 1), DAYS = c(10, 10, 20, 15))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  build_table(df)
```

```text
A          B    
————————————————————————————————————
```


### 27. a_sum_ratio_j returns NA fraction when denominator is zero

`id: test:test-a_sum_ratio_j:157` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_sum_ratio_j.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

df_zero <- data.frame(USUBJID = c("S01", "S02"), ARM = factor(c("A", "A")), EVENTS = c(5, 
    3), DAYS = c(0, 0))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  build_table(df_zero)
```

```text
A   
———————————————
```


### 28. a_summarize_aval_chg_diff_j comp_btw_group = FALSE works as expected

`id: test:test-a_summarize_aval_chg_diff:35` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_summarize_aval_chg_diff.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl <- ex_adsl

advs <- mutate(filter(filter(ex_advs, PARAMCD %in% c("DIABP", "PULSE")), AVISIT %in% 
    c("BASELINE", "WEEK 1 DAY 8", "WEEK 2 DAY 15")), PARAMCD = droplevels(PARAMCD), 
    AVISIT = droplevels(AVISIT))

multivars <- c("AVAL", "AVAL", "CHG")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  split_cols_by_multivar(multivars, varlabels = c("n/N (%)", "Mean (95% CI)", "Mean Change From Baseline (95% CI)")) |>
  build_table(advs, alt_counts_df = adsl)
```

```text
A: Drug X                                                                      B: Placebo                                                                   C: Combination                               
                      n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)       n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)       n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 29. a_summarize_aval_chg_diff_j t-test sparse data works as expected

`id: test:test-a_summarize_aval_chg_diff:146` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_summarize_aval_chg_diff.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

ctrl_grp <- "B: Placebo"

trtvar <- "ARM"

adsl <- mutate(select(ex_adsl, STUDYID, USUBJID, ARM), colspan_trt = factor(ifelse(ARM == 
    ctrl_grp, " ", "Active treatment"), levels = c(" ", "Active treatment")), rrisk_header = "Difference in Mean Change (95% CI)", 
    rrisk_label = paste0(ARM, " vs ", ctrl_grp))

advs <- mutate(filter(filter(ex_advs, PARAMCD %in% c("DIABP", "PULSE")), AVISIT %in% 
    c("BASELINE", "WEEK 1 DAY 8", "WEEK 2 DAY 15")), PARAMCD = droplevels(PARAMCD), 
    AVISIT = droplevels(AVISIT))

advs <- inner_join(advs, adsl, by = join_by(STUDYID, USUBJID, ARM))

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active treatment", colspan_var = "colspan_trt", trt_var = trtvar)

multivars <- c("AVAL", "AVAL", "CHG")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by("ARM", show_colcounts = TRUE) |>
  split_cols_by_multivar(multivars, varlabels = c("n/N (%)", "Mean (95% CI)", "Mean Change From Baseline (95% CI)")) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by("ARM", split_fun = remove_split_levels(ctrl_grp), labels_var = "rrisk_label", 
      show_colcounts = FALSE) |>
  split_cols_by_multivar(multivars[3], varlabels = c(" ")) |>
  build_table(advs, alt_counts_df = adsl)
```

```text
Active treatment                                                                                                                                                                                                              
                                                   A: Drug X                                                                    C: Combination                                                                   B: Placebo                                              Difference in Mean Change (95% CI)          
                                                    (N=134)                                                                        (N=132)                                                                        (N=134)                                      A: Drug X vs B: Placebo   C: Combination vs B: Placebo
                      n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)       n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)       n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)                                                         
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 30. a_summarize_aval_chg_diff_j works with ancova = TRUE

`id: test:test-a_summarize_aval_chg_diff:434` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_summarize_aval_chg_diff.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

ADEG <- data.frame(STUDYID = c("DUMMY", "DUMMY", "DUMMY", "DUMMY", "DUMMY", "DUMMY", 
    "DUMMY", "DUMMY", "DUMMY", "DUMMY"), USUBJID = c("XXXXX01", "XXXXX02", "XXXXX03", 
    "XXXXX04", "XXXXX05", "XXXXX06", "XXXXX07", "XXXXX08", "XXXXX09", "XXXXX10"), 
    TRT01A = c("ARMA", "ARMA", "ARMA", "ARMA", "ARMA", "Placebo", "Placebo", "Placebo", 
        "ARMA", "ARMA"), PARAM = c("BP", "BP", "BP", "BP", "BP", "BP", "BP", "BP", 
        "BP", "BP"), AVISIT = c("Visit 1", "Visit 1", "Visit 1", "Visit 1", "Visit 1", 
        "Visit 1", "Visit 1", "Visit 1", "Visit 1", "Visit 1"), AVAL = c(56, 78, 
        67, 87, 88, 93, 39, 87, 65, 55), CHG = c(2, 3, -1, 9, -2, 0, 6, -2, 5, 2))

ADEG <- mutate(ADEG, TRT01A = as.factor(TRT01A), STUDYID = as.factor(STUDYID))

ADEG$colspan_trt <- factor(ifelse(ADEG$TRT01A == "Placebo", " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

ADEG$rrisk_header <- "Risk Difference (%) (95% CI)"

ADEG$rrisk_label <- paste(ADEG$TRT01A, paste("vs", "Placebo"))

colspan_trt_map <- create_colspan_map(ADEG, non_active_grp = "Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = "TRT01A")
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by("TRT01A") |>
  split_cols_by_multivar(c("AVAL", "AVAL", "CHG"), varlabels = c("n/N (%)", "Mean (CI)", 
      "CFB (CI)")) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by("TRT01A", split_fun = remove_split_levels("Placebo"), labels_var = "rrisk_label") |>
  split_cols_by_multivar(c("CHG"), varlabels = c(" ")) |>
  build_table(ADEG)
```

```text
Active Study Agent                                                                        Risk Difference (%) (95% CI)
Blood Pressure                          ARMA                                               Placebo                               ARMA vs Placebo       
  Study Visit      n/N (%)          Mean (CI)          CFB (CI)         n/N (%)          Mean (CI)          CFB (CI)                                   
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 31. a_summarize_aval_chg_diff_j ancova in a combined column work as expected

`id: test:test-a_summarize_aval_chg_diff:544` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_summarize_aval_chg_diff.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

ctrl_grp <- "B: Placebo"

adsl <- ex_adsl

adsl$rrisk_header <- "Difference in Mean Change (95% CI)"

adsl$rrisk_label <- paste(adsl[["ARM"]], paste("vs", ctrl_grp))

adsl$colspan_trt_dummy <- " "

adsl <- select(adsl, USUBJID, ARM, rrisk_header, rrisk_label, colspan_trt_dummy)

advs <- left_join(mutate(filter(filter(ex_advs, PARAMCD %in% c("DIABP", "PULSE")), 
    AVISIT %in% c("BASELINE", "WEEK 1 DAY 8", "WEEK 2 DAY 15")), PARAMCD = droplevels(PARAMCD), 
    AVISIT = droplevels(AVISIT)), adsl, by = c("USUBJID", "ARM"))

multivars <- c("AVAL", "AVAL", "CHG")

add_combo <- add_combo_facet("Combined", label = "Combined A + C", levels = c("A: Drug X", 
    "C: Combination"))

split_fun1 <- make_split_fun(post = list(add_combo))

add_combo2 <- add_combo_facet("Combined", label = "Combined A + C vs B: Placebo", 
    levels = c("A: Drug X", "C: Combination"))

rm_placebo_from_rdiff <- cond_rm_facets(facets = "B: Placebo", ancestor_pos = -1, 
    value = "Difference in Mean Change (95% CI)", split = "rrisk_header", keep_matches = FALSE)

split_fun2 <- make_split_fun(post = list(add_combo2, rm_placebo_from_rdiff))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("colspan_trt_dummy") |>
  split_cols_by("ARM", split_fun = split_fun1) |>
  split_cols_by_multivar(multivars, varlabels = c("n/N (%)", "Mean (95% CI)", "Mean Change From Baseline (95% CI)")) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by("ARM", split_fun = split_fun2, labels_var = "rrisk_label", colcount_format = "N=xx") |>
  split_cols_by_multivar(multivars[3], varlabels = c(" ")) |>
  build_table(advs, alt_counts_df = adsl)
```

```text
Difference in Mean Change (95% CI)                          
                                                   A: Drug X                                                                      B: Placebo                                                                   C: Combination                                                                 Combined A + C                                  A: Drug X vs B: Placebo   C: Combination vs B: Placebo   Combined A + C vs B: Placebo
                      n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)       n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)       n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)       n/N (%)           Mean (95% CI)       Mean Change From Baseline (95% CI)                                                                                        
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 32. a_two_tier works in a table layout as expected

`id: test:test-a_two_tier:43` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-a_two_tier.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

n <- 36

data <- data.frame(trt = factor(sample(c("A", "B", "C"), n, replace = TRUE)), eostt = factor(sample(c("COMPLETED", 
    "DISCONTINUED", "ONGOING"), n, replace = TRUE)), dcsreas = factor(sample(c("ADVERSE EVENT", 
    "LACK OF EFFICACY", "PHYSICIAN DECISION"), n, replace = TRUE)), subdcsreas = factor(sample(c("MILD", 
    "MODERATE", "SEVERE"), n, replace = TRUE)))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("trt") |>
  build_table(data)
```

```text
A   B   C
————————————————————————————————
```


### 33. analyze_values correctly modifies layout with proper formats

`id: test:test-analyze_values:36` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-analyze_values.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

mock_data <- data.frame(ARM = rep(c("A", "B"), each = 5), AGE = c(25, 30, 35, 40, 
    45, 22, 28, 32, 38, 42), WEIGHT = c(70, 75, 80, 85, 90, 68, 72, 78, 82, 88))
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("ARM") |>
  build_table(mock_data)
```

```text
A             B     
—————————————————————————————————————————
```


### 34. insert_blank_line works as expected

`id: test:test-blank_line:26` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-blank_line.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

ADSL <- ex_adsl
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("ARM") |>
  build_table(ADSL)
```

```text
A: Drug X      B: Placebo     C: Combination
——————————————————————————————————————————————————————————————
```


### 35. cmp_split_fun works as expected

`id: test:test-cmp_functions:2` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-cmp_functions.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("ID", split_fun = cmp_split_fun) |>
  build_table(formatters::DM)
```

```text
Expected, N   Received, n (%)   Missing, n (%)
—————————————————————————————————————————————————
```


### 36. basic direct make_multicomp_splfun usage works

`id: test:test-colstruct:175` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-colstruct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl_jnj <- pharmaverseadamjnj::adsl

adae_jnj <- pharmaverseadamjnj::adae

fix_usubjid <- function(adsl, newlev = "Std Of Care") {
    rws <- which(adsl$TRT01P == newlev)
    usubj_char <- as.character(adsl$USUBJID)
    subjid <- as.integer(as.character(adsl$SUBJID))
    subjid[rws] <- subjid[rws] + 1000
    substr(usubj_char, 8, 11) <- as.character(subjid)
    adsl$USUBJID <- factor(usubj_char)
    adsl$SUBJID <- factor(as.character(subjid))
    adsl
}

make_fake_adsl <- function(adsl, newlev = "Std Of Care", oldlev = "Placebo") {
    fakeyfake <- filter(adsl, TRT01P == !!oldlev)
    fakeyfake$TRT01P <- newlev
    fakeyfake$AGE <- floor(runif(NROW(fakeyfake), 30, 90))
    adsl$TRT01P <- as.character(adsl$TRT01P)
    adsl <- rbind(adsl, fakeyfake)
    adsl$TRT01P <- factor(adsl$TRT01P)
    fix_usubjid(adsl, newlev = newlev)
}

borrow_aes <- function(adae, adsl, mult = 1, newlev = "Std Of Care", oldlev = "Placebo") {
    plac_count <- sum(adae$TRT01P == oldlev, na.rm = TRUE)
    new_count <- floor(plac_count * mult)
    soc_usubjids <- as.character(adsl$USUBJID)[!is.na(adsl$TRT01P) & adsl$TRT01P == 
        newlev]
    duprows <- sample(seq_len(NROW(adae)), new_count, replace = TRUE)
    newrws <- adae[duprows, ]
    newrws$USUBJID <- sample(soc_usubjids, NROW(newrws), replace = TRUE)
    rbind(adae, newrws)
}

trtvar <- "TRT01P"

adsl <- select(mutate(create_colspan_var(make_fake_adsl(adsl_jnj), non_active_grp = c("Placebo", 
    "Std Of Care"), non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active Study Agent", 
    colspan_var = "colspan_trt", trt_var = trtvar), diffs_label = "Risk Difference (%) (95% CI)", 
    rrisk_label = paste(!!rlang::sym(trtvar), "vs Placebo"), rrisk_label2 = paste(!!rlang::sym(trtvar), 
        "vs Std of Care")), USUBJID, !!rlang::sym(trtvar), colspan_trt, diffs_label, 
    rrisk_label, rrisk_label2)

adae <- mutate(mutate(ungroup(mutate(group_by(select(borrow_aes(filter(adae_jnj, 
    TRTEMFL == "Y"), adsl), USUBJID, AESER, AESDTH, AESLIFE, AESHOSP, AESDISAB, AESCONG, 
    AESMIE, AESEV), USUBJID), maxsev = max(as.character(AESEV), na.rm = TRUE))), 
    maxsev = ifelse(is.na(maxsev), "Missing", maxsev)), maxsev = factor(maxsev, levels = c("Mild", 
    "Moderate", "Severe", "Missing")))

adae <- inner_join(adae, adsl, by = c("USUBJID"), multiple = "all")

ctrl_grp <- c("Placebo", "Std Of Care")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

splfun <- make_multicomp_splfun(colspan_trt_map)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(trtvar, split_fun = splfun) |>
  build_table(adae, adsl)
```

```text
Xanomeline High Dose vs Placebo   Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Std Of Care   Xanomeline Low Dose vs Std Of Care
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 37. basic direct make_multicomp_splfun usage works

`id: test:test-colstruct:204` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-colstruct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl_jnj <- pharmaverseadamjnj::adsl

adae_jnj <- pharmaverseadamjnj::adae

fix_usubjid <- function(adsl, newlev = "Std Of Care") {
    rws <- which(adsl$TRT01P == newlev)
    usubj_char <- as.character(adsl$USUBJID)
    subjid <- as.integer(as.character(adsl$SUBJID))
    subjid[rws] <- subjid[rws] + 1000
    substr(usubj_char, 8, 11) <- as.character(subjid)
    adsl$USUBJID <- factor(usubj_char)
    adsl$SUBJID <- factor(as.character(subjid))
    adsl
}

make_fake_adsl <- function(adsl, newlev = "Std Of Care", oldlev = "Placebo") {
    fakeyfake <- filter(adsl, TRT01P == !!oldlev)
    fakeyfake$TRT01P <- newlev
    fakeyfake$AGE <- floor(runif(NROW(fakeyfake), 30, 90))
    adsl$TRT01P <- as.character(adsl$TRT01P)
    adsl <- rbind(adsl, fakeyfake)
    adsl$TRT01P <- factor(adsl$TRT01P)
    fix_usubjid(adsl, newlev = newlev)
}

borrow_aes <- function(adae, adsl, mult = 1, newlev = "Std Of Care", oldlev = "Placebo") {
    plac_count <- sum(adae$TRT01P == oldlev, na.rm = TRUE)
    new_count <- floor(plac_count * mult)
    soc_usubjids <- as.character(adsl$USUBJID)[!is.na(adsl$TRT01P) & adsl$TRT01P == 
        newlev]
    duprows <- sample(seq_len(NROW(adae)), new_count, replace = TRUE)
    newrws <- adae[duprows, ]
    newrws$USUBJID <- sample(soc_usubjids, NROW(newrws), replace = TRUE)
    rbind(adae, newrws)
}

trtvar <- "TRT01P"

adsl <- select(mutate(create_colspan_var(make_fake_adsl(adsl_jnj), non_active_grp = c("Placebo", 
    "Std Of Care"), non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active Study Agent", 
    colspan_var = "colspan_trt", trt_var = trtvar), diffs_label = "Risk Difference (%) (95% CI)", 
    rrisk_label = paste(!!rlang::sym(trtvar), "vs Placebo"), rrisk_label2 = paste(!!rlang::sym(trtvar), 
        "vs Std of Care")), USUBJID, !!rlang::sym(trtvar), colspan_trt, diffs_label, 
    rrisk_label, rrisk_label2)

adae <- mutate(mutate(ungroup(mutate(group_by(select(borrow_aes(filter(adae_jnj, 
    TRTEMFL == "Y"), adsl), USUBJID, AESER, AESDTH, AESLIFE, AESHOSP, AESDISAB, AESCONG, 
    AESMIE, AESEV), USUBJID), maxsev = max(as.character(AESEV), na.rm = TRUE))), 
    maxsev = ifelse(is.na(maxsev), "Missing", maxsev)), maxsev = factor(maxsev, levels = c("Mild", 
    "Moderate", "Severe", "Missing")))

adae <- inner_join(adae, adsl, by = c("USUBJID"), multiple = "all")

ctrl_grp <- c("Placebo", "Std Of Care")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

lvls <- levels(adsl[[trtvar]])

combodf <- tribble(~valname, ~label, ~levelcombo, ~exargs, "all_active", "All Active", 
    lvls[3:4], list(), "all_patients", "All Patients", select_all_levels, list())

comp_map <- make_dflt_comp_map(adsl, trtvar, ctrl_grp, combodf)

splfun2 <- make_multicomp_splfun(colspan_trt_map, comp_level_map = subset(comp_map, 
    !active_is_combo))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(trtvar, split_fun = splfun2) |>
  build_table(adae, adsl)
```

```text
Xanomeline High Dose vs Placebo   Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Std Of Care   Xanomeline Low Dose vs Std Of Care
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 38. basic direct make_multicomp_splfun usage works

`id: test:test-colstruct:214` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-colstruct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl_jnj <- pharmaverseadamjnj::adsl

adae_jnj <- pharmaverseadamjnj::adae

fix_usubjid <- function(adsl, newlev = "Std Of Care") {
    rws <- which(adsl$TRT01P == newlev)
    usubj_char <- as.character(adsl$USUBJID)
    subjid <- as.integer(as.character(adsl$SUBJID))
    subjid[rws] <- subjid[rws] + 1000
    substr(usubj_char, 8, 11) <- as.character(subjid)
    adsl$USUBJID <- factor(usubj_char)
    adsl$SUBJID <- factor(as.character(subjid))
    adsl
}

make_fake_adsl <- function(adsl, newlev = "Std Of Care", oldlev = "Placebo") {
    fakeyfake <- filter(adsl, TRT01P == !!oldlev)
    fakeyfake$TRT01P <- newlev
    fakeyfake$AGE <- floor(runif(NROW(fakeyfake), 30, 90))
    adsl$TRT01P <- as.character(adsl$TRT01P)
    adsl <- rbind(adsl, fakeyfake)
    adsl$TRT01P <- factor(adsl$TRT01P)
    fix_usubjid(adsl, newlev = newlev)
}

borrow_aes <- function(adae, adsl, mult = 1, newlev = "Std Of Care", oldlev = "Placebo") {
    plac_count <- sum(adae$TRT01P == oldlev, na.rm = TRUE)
    new_count <- floor(plac_count * mult)
    soc_usubjids <- as.character(adsl$USUBJID)[!is.na(adsl$TRT01P) & adsl$TRT01P == 
        newlev]
    duprows <- sample(seq_len(NROW(adae)), new_count, replace = TRUE)
    newrws <- adae[duprows, ]
    newrws$USUBJID <- sample(soc_usubjids, NROW(newrws), replace = TRUE)
    rbind(adae, newrws)
}

trtvar <- "TRT01P"

adsl <- select(mutate(create_colspan_var(make_fake_adsl(adsl_jnj), non_active_grp = c("Placebo", 
    "Std Of Care"), non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active Study Agent", 
    colspan_var = "colspan_trt", trt_var = trtvar), diffs_label = "Risk Difference (%) (95% CI)", 
    rrisk_label = paste(!!rlang::sym(trtvar), "vs Placebo"), rrisk_label2 = paste(!!rlang::sym(trtvar), 
        "vs Std of Care")), USUBJID, !!rlang::sym(trtvar), colspan_trt, diffs_label, 
    rrisk_label, rrisk_label2)

adae <- mutate(mutate(ungroup(mutate(group_by(select(borrow_aes(filter(adae_jnj, 
    TRTEMFL == "Y"), adsl), USUBJID, AESER, AESDTH, AESLIFE, AESHOSP, AESDISAB, AESCONG, 
    AESMIE, AESEV), USUBJID), maxsev = max(as.character(AESEV), na.rm = TRUE))), 
    maxsev = ifelse(is.na(maxsev), "Missing", maxsev)), maxsev = factor(maxsev, levels = c("Mild", 
    "Moderate", "Severe", "Missing")))

adae <- inner_join(adae, adsl, by = c("USUBJID"), multiple = "all")

ctrl_grp <- c("Placebo", "Std Of Care")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

lvls <- levels(adsl[[trtvar]])

combodf <- tribble(~valname, ~label, ~levelcombo, ~exargs, "all_active", "All Active", 
    lvls[3:4], list(), "all_patients", "All Patients", select_all_levels, list())

comp_map <- make_dflt_comp_map(adsl, trtvar, ctrl_grp, combodf)

splfun3 <- make_multicomp_splfun(colspan_trt_map, comp_level_map = comp_map[c(6, 
    2, 1), ])
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(trtvar, split_fun = splfun3) |>
  build_table(adae, adsl)
```

```text
Xanomeline Low Dose vs Std Of Care   Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Placebo
——————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 39. basic direct make_multicomp_splfun usage works

`id: test:test-colstruct:234` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-colstruct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl_jnj <- pharmaverseadamjnj::adsl

adae_jnj <- pharmaverseadamjnj::adae

fix_usubjid <- function(adsl, newlev = "Std Of Care") {
    rws <- which(adsl$TRT01P == newlev)
    usubj_char <- as.character(adsl$USUBJID)
    subjid <- as.integer(as.character(adsl$SUBJID))
    subjid[rws] <- subjid[rws] + 1000
    substr(usubj_char, 8, 11) <- as.character(subjid)
    adsl$USUBJID <- factor(usubj_char)
    adsl$SUBJID <- factor(as.character(subjid))
    adsl
}

make_fake_adsl <- function(adsl, newlev = "Std Of Care", oldlev = "Placebo") {
    fakeyfake <- filter(adsl, TRT01P == !!oldlev)
    fakeyfake$TRT01P <- newlev
    fakeyfake$AGE <- floor(runif(NROW(fakeyfake), 30, 90))
    adsl$TRT01P <- as.character(adsl$TRT01P)
    adsl <- rbind(adsl, fakeyfake)
    adsl$TRT01P <- factor(adsl$TRT01P)
    fix_usubjid(adsl, newlev = newlev)
}

borrow_aes <- function(adae, adsl, mult = 1, newlev = "Std Of Care", oldlev = "Placebo") {
    plac_count <- sum(adae$TRT01P == oldlev, na.rm = TRUE)
    new_count <- floor(plac_count * mult)
    soc_usubjids <- as.character(adsl$USUBJID)[!is.na(adsl$TRT01P) & adsl$TRT01P == 
        newlev]
    duprows <- sample(seq_len(NROW(adae)), new_count, replace = TRUE)
    newrws <- adae[duprows, ]
    newrws$USUBJID <- sample(soc_usubjids, NROW(newrws), replace = TRUE)
    rbind(adae, newrws)
}

trtvar <- "TRT01P"

adsl <- select(mutate(create_colspan_var(make_fake_adsl(adsl_jnj), non_active_grp = c("Placebo", 
    "Std Of Care"), non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active Study Agent", 
    colspan_var = "colspan_trt", trt_var = trtvar), diffs_label = "Risk Difference (%) (95% CI)", 
    rrisk_label = paste(!!rlang::sym(trtvar), "vs Placebo"), rrisk_label2 = paste(!!rlang::sym(trtvar), 
        "vs Std of Care")), USUBJID, !!rlang::sym(trtvar), colspan_trt, diffs_label, 
    rrisk_label, rrisk_label2)

adae <- mutate(mutate(ungroup(mutate(group_by(select(borrow_aes(filter(adae_jnj, 
    TRTEMFL == "Y"), adsl), USUBJID, AESER, AESDTH, AESLIFE, AESHOSP, AESDISAB, AESCONG, 
    AESMIE, AESEV), USUBJID), maxsev = max(as.character(AESEV), na.rm = TRUE))), 
    maxsev = ifelse(is.na(maxsev), "Missing", maxsev)), maxsev = factor(maxsev, levels = c("Mild", 
    "Moderate", "Severe", "Missing")))

adae <- inner_join(adae, adsl, by = c("USUBJID"), multiple = "all")

ctrl_grp <- c("Placebo", "Std Of Care")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

lvls <- levels(adsl[[trtvar]])

combodf <- tribble(~valname, ~label, ~levelcombo, ~exargs, "all_active", "All Active", 
    lvls[3:4], list(), "all_patients", "All Patients", select_all_levels, list())

comp_map <- make_dflt_comp_map(adsl, trtvar, ctrl_grp, combodf)

splfun4 <- make_multicomp_splfun(colspan_trt_map, comp_level_map = comp_map)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(trtvar, split_fun = splfun4) |>
  build_table(adae, adsl)
```

```text
Xanomeline High Dose vs Placebo   Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Std Of Care   Xanomeline Low Dose vs Std Of Care
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 40. make_multicomp_splfun works with active combo levels

`id: test:test-colstruct:244` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-colstruct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl_jnj <- pharmaverseadamjnj::adsl

adae_jnj <- pharmaverseadamjnj::adae

fix_usubjid <- function(adsl, newlev = "Std Of Care") {
    rws <- which(adsl$TRT01P == newlev)
    usubj_char <- as.character(adsl$USUBJID)
    subjid <- as.integer(as.character(adsl$SUBJID))
    subjid[rws] <- subjid[rws] + 1000
    substr(usubj_char, 8, 11) <- as.character(subjid)
    adsl$USUBJID <- factor(usubj_char)
    adsl$SUBJID <- factor(as.character(subjid))
    adsl
}

make_fake_adsl <- function(adsl, newlev = "Std Of Care", oldlev = "Placebo") {
    fakeyfake <- filter(adsl, TRT01P == !!oldlev)
    fakeyfake$TRT01P <- newlev
    fakeyfake$AGE <- floor(runif(NROW(fakeyfake), 30, 90))
    adsl$TRT01P <- as.character(adsl$TRT01P)
    adsl <- rbind(adsl, fakeyfake)
    adsl$TRT01P <- factor(adsl$TRT01P)
    fix_usubjid(adsl, newlev = newlev)
}

borrow_aes <- function(adae, adsl, mult = 1, newlev = "Std Of Care", oldlev = "Placebo") {
    plac_count <- sum(adae$TRT01P == oldlev, na.rm = TRUE)
    new_count <- floor(plac_count * mult)
    soc_usubjids <- as.character(adsl$USUBJID)[!is.na(adsl$TRT01P) & adsl$TRT01P == 
        newlev]
    duprows <- sample(seq_len(NROW(adae)), new_count, replace = TRUE)
    newrws <- adae[duprows, ]
    newrws$USUBJID <- sample(soc_usubjids, NROW(newrws), replace = TRUE)
    rbind(adae, newrws)
}

trtvar <- "TRT01P"

adsl <- select(mutate(create_colspan_var(make_fake_adsl(adsl_jnj), non_active_grp = c("Placebo", 
    "Std Of Care"), non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active Study Agent", 
    colspan_var = "colspan_trt", trt_var = trtvar), diffs_label = "Risk Difference (%) (95% CI)", 
    rrisk_label = paste(!!rlang::sym(trtvar), "vs Placebo"), rrisk_label2 = paste(!!rlang::sym(trtvar), 
        "vs Std of Care")), USUBJID, !!rlang::sym(trtvar), colspan_trt, diffs_label, 
    rrisk_label, rrisk_label2)

adae <- mutate(mutate(ungroup(mutate(group_by(select(borrow_aes(filter(adae_jnj, 
    TRTEMFL == "Y"), adsl), USUBJID, AESER, AESDTH, AESLIFE, AESHOSP, AESDISAB, AESCONG, 
    AESMIE, AESEV), USUBJID), maxsev = max(as.character(AESEV), na.rm = TRUE))), 
    maxsev = ifelse(is.na(maxsev), "Missing", maxsev)), maxsev = factor(maxsev, levels = c("Mild", 
    "Moderate", "Severe", "Missing")))

adae <- inner_join(adae, adsl, by = c("USUBJID"), multiple = "all")

ctrl_grp <- c("Placebo", "Std Of Care")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

lvls <- levels(adsl[[trtvar]])

combodf <- tribble(~valname, ~label, ~levelcombo, ~exargs, "all_active", "All Active", 
    lvls[3:4], list(), "all_patients", "All Patients", select_all_levels, list())

comp_map <- make_dflt_comp_map(adsl, trtvar, ctrl_grp, combodf)

splfun <- make_multicomp_splfun(colspan_trt_map, combodf, comp_map)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by(trtvar, split_fun = splfun) |>
  build_table(adae, adsl)
```

```text
Xanomeline High Dose vs Placebo   Xanomeline Low Dose vs Placebo      All Active vs Placebo         All Patients vs Placebo      Xanomeline High Dose vs Std Of Care   Xanomeline Low Dose vs Std Of Care      All Active vs Std Of Care         All Patients vs Std Of Care   
                     (N=84)                            (N=84)                         (N=168)                        (N=392)                            (N=84)                                (N=84)                             (N=168)                            (N=392)             
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 41. make_multicomp_splfun works with combo comparator levels (default comps)

`id: test:test-colstruct:271` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-colstruct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl_jnj <- pharmaverseadamjnj::adsl

adae_jnj <- pharmaverseadamjnj::adae

fix_usubjid <- function(adsl, newlev = "Std Of Care") {
    rws <- which(adsl$TRT01P == newlev)
    usubj_char <- as.character(adsl$USUBJID)
    subjid <- as.integer(as.character(adsl$SUBJID))
    subjid[rws] <- subjid[rws] + 1000
    substr(usubj_char, 8, 11) <- as.character(subjid)
    adsl$USUBJID <- factor(usubj_char)
    adsl$SUBJID <- factor(as.character(subjid))
    adsl
}

make_fake_adsl <- function(adsl, newlev = "Std Of Care", oldlev = "Placebo") {
    fakeyfake <- filter(adsl, TRT01P == !!oldlev)
    fakeyfake$TRT01P <- newlev
    fakeyfake$AGE <- floor(runif(NROW(fakeyfake), 30, 90))
    adsl$TRT01P <- as.character(adsl$TRT01P)
    adsl <- rbind(adsl, fakeyfake)
    adsl$TRT01P <- factor(adsl$TRT01P)
    fix_usubjid(adsl, newlev = newlev)
}

borrow_aes <- function(adae, adsl, mult = 1, newlev = "Std Of Care", oldlev = "Placebo") {
    plac_count <- sum(adae$TRT01P == oldlev, na.rm = TRUE)
    new_count <- floor(plac_count * mult)
    soc_usubjids <- as.character(adsl$USUBJID)[!is.na(adsl$TRT01P) & adsl$TRT01P == 
        newlev]
    duprows <- sample(seq_len(NROW(adae)), new_count, replace = TRUE)
    newrws <- adae[duprows, ]
    newrws$USUBJID <- sample(soc_usubjids, NROW(newrws), replace = TRUE)
    rbind(adae, newrws)
}

trtvar <- "TRT01P"

adsl <- select(mutate(create_colspan_var(make_fake_adsl(adsl_jnj), non_active_grp = c("Placebo", 
    "Std Of Care"), non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active Study Agent", 
    colspan_var = "colspan_trt", trt_var = trtvar), diffs_label = "Risk Difference (%) (95% CI)", 
    rrisk_label = paste(!!rlang::sym(trtvar), "vs Placebo"), rrisk_label2 = paste(!!rlang::sym(trtvar), 
        "vs Std of Care")), USUBJID, !!rlang::sym(trtvar), colspan_trt, diffs_label, 
    rrisk_label, rrisk_label2)

adae <- mutate(mutate(ungroup(mutate(group_by(select(borrow_aes(filter(adae_jnj, 
    TRTEMFL == "Y"), adsl), USUBJID, AESER, AESDTH, AESLIFE, AESHOSP, AESDISAB, AESCONG, 
    AESMIE, AESEV), USUBJID), maxsev = max(as.character(AESEV), na.rm = TRUE))), 
    maxsev = ifelse(is.na(maxsev), "Missing", maxsev)), maxsev = factor(maxsev, levels = c("Mild", 
    "Moderate", "Severe", "Missing")))

adae <- inner_join(adae, adsl, by = c("USUBJID"), multiple = "all")

ctrl_grp <- c("Placebo", "Std Of Care")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

lvls <- levels(adsl[[trtvar]])

combodf <- tribble(~valname, ~label, ~levelcombo, ~exargs, "all_active", "All Active", 
    lvls[3:4], list(), "all_patients", "All Patients", select_all_levels, list())

combodf2 <- combodf

combodf2$is_control <- c(FALSE, TRUE)

splfun <- make_multicomp_splfun(colspan_trt_map, combodf2)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by(trtvar, split_fun = splfun) |>
  build_table(adae, adsl)
```

```text
Xanomeline High Dose vs Placebo   Xanomeline Low Dose vs Placebo      All Active vs Placebo       Xanomeline High Dose vs Std Of Care   Xanomeline Low Dose vs Std Of Care      All Active vs Std Of Care       Xanomeline High Dose vs All Patients   Xanomeline Low Dose vs All Patients      All Active vs All Patients    
                     (N=84)                            (N=84)                         (N=168)                            (N=84)                                (N=84)                             (N=168)                               (N=84)                                (N=84)                               (N=168)             
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 42. make_multicomp_splfun works with active comparators (non-default comps)

`id: test:test-colstruct:292` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-colstruct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl_jnj <- pharmaverseadamjnj::adsl

adae_jnj <- pharmaverseadamjnj::adae

fix_usubjid <- function(adsl, newlev = "Std Of Care") {
    rws <- which(adsl$TRT01P == newlev)
    usubj_char <- as.character(adsl$USUBJID)
    subjid <- as.integer(as.character(adsl$SUBJID))
    subjid[rws] <- subjid[rws] + 1000
    substr(usubj_char, 8, 11) <- as.character(subjid)
    adsl$USUBJID <- factor(usubj_char)
    adsl$SUBJID <- factor(as.character(subjid))
    adsl
}

make_fake_adsl <- function(adsl, newlev = "Std Of Care", oldlev = "Placebo") {
    fakeyfake <- filter(adsl, TRT01P == !!oldlev)
    fakeyfake$TRT01P <- newlev
    fakeyfake$AGE <- floor(runif(NROW(fakeyfake), 30, 90))
    adsl$TRT01P <- as.character(adsl$TRT01P)
    adsl <- rbind(adsl, fakeyfake)
    adsl$TRT01P <- factor(adsl$TRT01P)
    fix_usubjid(adsl, newlev = newlev)
}

borrow_aes <- function(adae, adsl, mult = 1, newlev = "Std Of Care", oldlev = "Placebo") {
    plac_count <- sum(adae$TRT01P == oldlev, na.rm = TRUE)
    new_count <- floor(plac_count * mult)
    soc_usubjids <- as.character(adsl$USUBJID)[!is.na(adsl$TRT01P) & adsl$TRT01P == 
        newlev]
    duprows <- sample(seq_len(NROW(adae)), new_count, replace = TRUE)
    newrws <- adae[duprows, ]
    newrws$USUBJID <- sample(soc_usubjids, NROW(newrws), replace = TRUE)
    rbind(adae, newrws)
}

trtvar <- "TRT01P"

adsl <- select(mutate(create_colspan_var(make_fake_adsl(adsl_jnj), non_active_grp = c("Placebo", 
    "Std Of Care"), non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active Study Agent", 
    colspan_var = "colspan_trt", trt_var = trtvar), diffs_label = "Risk Difference (%) (95% CI)", 
    rrisk_label = paste(!!rlang::sym(trtvar), "vs Placebo"), rrisk_label2 = paste(!!rlang::sym(trtvar), 
        "vs Std of Care")), USUBJID, !!rlang::sym(trtvar), colspan_trt, diffs_label, 
    rrisk_label, rrisk_label2)

adae <- mutate(mutate(ungroup(mutate(group_by(select(borrow_aes(filter(adae_jnj, 
    TRTEMFL == "Y"), adsl), USUBJID, AESER, AESDTH, AESLIFE, AESHOSP, AESDISAB, AESCONG, 
    AESMIE, AESEV), USUBJID), maxsev = max(as.character(AESEV), na.rm = TRUE))), 
    maxsev = ifelse(is.na(maxsev), "Missing", maxsev)), maxsev = factor(maxsev, levels = c("Mild", 
    "Moderate", "Severe", "Missing")))

adae <- inner_join(adae, adsl, by = c("USUBJID"), multiple = "all")

ctrl_grp <- c("Placebo", "Std Of Care")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

splfun <- make_multicomp_splfun(colspan_trt_map, comp_level_map = data.frame(active = "Xanomeline High Dose", 
    comparator = "Xanomeline Low Dose"))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by(trtvar, split_fun = splfun) |>
  build_table(adae, adsl)
```

```text
Xanomeline High Dose vs Xanomeline Low Dose       
                                  (N=84)                          
——————————————————————————————————————————————————————————————————
```


### 43. make_multicomp_splfun works with active comparators (non-default comps)

`id: test:test-colstruct:311` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-colstruct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl_jnj <- pharmaverseadamjnj::adsl

adae_jnj <- pharmaverseadamjnj::adae

fix_usubjid <- function(adsl, newlev = "Std Of Care") {
    rws <- which(adsl$TRT01P == newlev)
    usubj_char <- as.character(adsl$USUBJID)
    subjid <- as.integer(as.character(adsl$SUBJID))
    subjid[rws] <- subjid[rws] + 1000
    substr(usubj_char, 8, 11) <- as.character(subjid)
    adsl$USUBJID <- factor(usubj_char)
    adsl$SUBJID <- factor(as.character(subjid))
    adsl
}

make_fake_adsl <- function(adsl, newlev = "Std Of Care", oldlev = "Placebo") {
    fakeyfake <- filter(adsl, TRT01P == !!oldlev)
    fakeyfake$TRT01P <- newlev
    fakeyfake$AGE <- floor(runif(NROW(fakeyfake), 30, 90))
    adsl$TRT01P <- as.character(adsl$TRT01P)
    adsl <- rbind(adsl, fakeyfake)
    adsl$TRT01P <- factor(adsl$TRT01P)
    fix_usubjid(adsl, newlev = newlev)
}

borrow_aes <- function(adae, adsl, mult = 1, newlev = "Std Of Care", oldlev = "Placebo") {
    plac_count <- sum(adae$TRT01P == oldlev, na.rm = TRUE)
    new_count <- floor(plac_count * mult)
    soc_usubjids <- as.character(adsl$USUBJID)[!is.na(adsl$TRT01P) & adsl$TRT01P == 
        newlev]
    duprows <- sample(seq_len(NROW(adae)), new_count, replace = TRUE)
    newrws <- adae[duprows, ]
    newrws$USUBJID <- sample(soc_usubjids, NROW(newrws), replace = TRUE)
    rbind(adae, newrws)
}

trtvar <- "TRT01P"

adsl <- select(mutate(create_colspan_var(make_fake_adsl(adsl_jnj), non_active_grp = c("Placebo", 
    "Std Of Care"), non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active Study Agent", 
    colspan_var = "colspan_trt", trt_var = trtvar), diffs_label = "Risk Difference (%) (95% CI)", 
    rrisk_label = paste(!!rlang::sym(trtvar), "vs Placebo"), rrisk_label2 = paste(!!rlang::sym(trtvar), 
        "vs Std of Care")), USUBJID, !!rlang::sym(trtvar), colspan_trt, diffs_label, 
    rrisk_label, rrisk_label2)

adae <- mutate(mutate(ungroup(mutate(group_by(select(borrow_aes(filter(adae_jnj, 
    TRTEMFL == "Y"), adsl), USUBJID, AESER, AESDTH, AESLIFE, AESHOSP, AESDISAB, AESCONG, 
    AESMIE, AESEV), USUBJID), maxsev = max(as.character(AESEV), na.rm = TRUE))), 
    maxsev = ifelse(is.na(maxsev), "Missing", maxsev)), maxsev = factor(maxsev, levels = c("Mild", 
    "Moderate", "Severe", "Missing")))

adae <- inner_join(adae, adsl, by = c("USUBJID"), multiple = "all")

ctrl_grp <- c("Placebo", "Std Of Care")

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

lvls <- levels(adsl[[trtvar]])

combodf <- tribble(~valname, ~label, ~levelcombo, ~exargs, "all_active", "All Active", 
    lvls[3:4], list(), "all_patients", "All Patients", select_all_levels, list())

splfun2 <- make_multicomp_splfun(colspan_trt_map, comp_level_map = data.frame(active = c("Xanomeline High Dose", 
    "Xanomeline Low Dose"), comparator = "all_patients"), combo_levels_map = combodf)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by(trtvar, split_fun = splfun2) |>
  build_table(adae, adsl)
```

```text
Xanomeline High Dose vs All Patients                 Xanomeline Low Dose vs All Patients        
                               (N=84)                                               (N=84)                      
————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 44. (file level)

`id: test:test-colwidths:59` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-colwidths.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

ADSL <- data.frame(USUBJID = c("XXXXX01", "XXXXX02", "XXXXX03", "XXXXX04", "XXXXX05", 
    "XXXXX06", "XXXXX07", "XXXXX08", "XXXXX09", "XXXXX10"), TRT01P = factor(c("ARMA", 
    "ARMB", "ARMA", "ARMB", "ARMB", "Placebo", "Placebo", "Placebo", "ARMA", "ARMB")), 
    FASFL = c("Y", "Y", "Y", "Y", "N", "Y", "Y", "Y", "Y", "Y"), SAFFL = c("N", "N", 
        "N", "N", "N", "N", "N", "N", "N", "N"), PKFL = c("N", "N", "N", "N", "N", 
        "N", "N", "N", "N", "N"))
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("TRT01P") |>
  add_overall_col("Total") |>
  build_table(ADSL)
```

```text
ARMA          ARMB         Placebo         Total    
————————————————————————————————————————————————————————————————————————
```


### 45. a_freq_j with val = NA and denom option

`id: test:test-count_pct_relrisk:106` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-count_pct_relrisk.R`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` to directly construct the grouped treatment spanning headers and pairwise risk difference columns. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

cols <- c("USUBJID", "STRATA1", "EOSSTT", trtvar)

adsl <- ex_adsl[, cols]

adsl <- create_colspan_var(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

colspan_trt_map <- create_colspan_map(df = adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE, round_type = "sas") |>
  grouped_cols_w_diffs(colspan_trt_map, diffs_label = "Risk Difference (%) (95% CI)") |>
  build_table(adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

cols <- c("USUBJID", "STRATA1", "EOSSTT", trtvar)

adsl <- ex_adsl[, cols]

adsl$colspan_trt <- factor(ifelse(adsl[[trtvar]] == ctrl_grp, " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

adsl$rrisk_header <- "Risk Difference (%) (95% CI)"

adsl$rrisk_label <- paste(adsl[[trtvar]], paste("vs", ctrl_grp))
```

**Fallback core layout instructions:**

```r
basic_table(show_colcounts = TRUE, round_type = "sas") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_in_group("ARM")) |>
  split_cols_by("ARM") |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by("ARM", labels_var = "rrisk_label", split_fun = remove_split_levels(ctrl_grp)) |>
  build_table(adsl)
```

</details>

```text
Active Study Agent                                              Risk Difference (%) (95% CI)                  
   A: Drug X   C: Combination   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo
    (N=134)       (N=132)        (N=134)             (N=134)                     (N=132)          
——————————————————————————————————————————————————————————————————————————————————————————————————
```

### 46. a_freq_j with TotCol_only

`id: test:test-count_pct:124` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-count_pct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

adsl <- ex_adsl
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE, round_type = "sas") |>
  split_cols_by("ARM") |>
  add_overall_col("Total") |>
  build_table(adsl)
```

```text
A: Drug X   B: Placebo   C: Combination    Total 
       (N=134)     (N=134)        (N=132)       (N=400)
———————————————————————————————————————————————————————
```


### 47. a_freq_j with N_trt as denom - special situation

`id: test:test-count_pct:438` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-count_pct.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adae <- ex_adae

adae$TRTEMFL <- "Y"

adsl_ <- select(ex_adsl, USUBJID, ARM)

adae_ <- select(ex_adae, USUBJID, ARM, AEBODSYS, AEDECOD, AESEV)

adae_$TRTEMFL <- "Y"

trtvar <- "ARM"

adsl_ <- mutate(adsl_, ASEV = factor("Total", levels = c("Total", levels(adae$AESEV))))

adsl_$spanheader <- factor(ifelse(adsl_$ASEV == "Total", " ", "Severity"), levels = c(" ", 
    "Severity"))

adaetot <- ungroup(slice(group_by(arrange(mutate(adae_, AESEV = "Total", AEBODSYSx = AEBODSYS), 
    USUBJID, AEBODSYS, AEDECOD), USUBJID, AEBODSYS, AEDECOD), 1))

adaemaxpt <- ungroup(slice(group_by(arrange(mutate(filter(adae_, toupper(AESEV) %in% 
    toupper(c("Mild", "Moderate", "Severe"))), AESEVN = case_when(toupper(AESEV) == 
    "MILD" ~ 3, toupper(AESEV) == "MODERATE" ~ 2, toupper(AESEV) == "SEVERE" ~ 1)), 
    USUBJID, AEBODSYS, AEDECOD, AESEVN), USUBJID, AEBODSYS, AEDECOD), 1))

adaemaxsoc <- select(ungroup(slice(group_by(arrange(mutate(filter(adae_, toupper(AESEV) %in% 
    toupper(c("Mild", "Moderate", "Severe"))), AESEVN = case_when(toupper(AESEV) == 
    "MILD" ~ 3, toupper(AESEV) == "MODERATE" ~ 2, toupper(AESEV) == "SEVERE" ~ 1), 
    AEBODSYSx = AEBODSYS), USUBJID, AEBODSYS, AESEVN), USUBJID, AEBODSYS), 1)), USUBJID, 
    AEBODSYS, AESEV, AEBODSYSx)

adaemax <- left_join(adaemaxpt, adaemaxsoc, by = c("USUBJID", "AEBODSYS", "AESEV"))

adaeall <- select(mutate(bind_rows(adaemax, adaetot), ASEV = factor(as.character(AESEV), 
    levels = c("Total", levels(ex_adae$AESEV)))), USUBJID, TRTEMFL, ASEV, AEBODSYS, 
    AEBODSYSx, AEDECOD)

adaeall <- inner_join(adaeall, select(adsl_, USUBJID, ARM), by = c("USUBJID"), relationship = "many-to-many")

adaeall$spanheader <- factor(ifelse(adaeall$ASEV == "Total", " ", "Severity"), levels = c(" ", 
    "Severity"))
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by(trtvar, show_colcounts = TRUE) |>
  split_cols_by("spanheader", split_fun = trim_levels_in_group("ASEV")) |>
  split_cols_by("ASEV", show_colcounts = TRUE) |>
  build_table(adaeall, alt_counts_df = adsl_)
```

```text
A: Drug X                                                          B: Placebo                                                        C: Combination                          
                                                  (N=134)                                                             (N=134)                                                             (N=132)                             
                                                          Severity                                                            Severity                                                            Severity                    
                         Total             MILD           MODERATE          SEVERE           Total             MILD           MODERATE          SEVERE           Total             MILD           MODERATE          SEVERE    
System Organ Class      (N=134)           (N=0)            (N=0)            (N=0)           (N=134)           (N=0)            (N=0)            (N=0)           (N=132)           (N=0)            (N=0)            (N=0)     
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 48. summarize_coxreg_multivar works as expected with custom options

`id: test:test-coxreg_multivar:119` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-coxreg_multivar.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(tern)

library(dplyr)

anl <- dplyr::mutate(tern::tern_ex_adtte, EVENT = 1 - CNSR)
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  build_table(anl)
```

```text
Model Fit             Hazard Ratio     
                                             Coeff. (SE)   p-value   Estimate     50% CI  
——————————————————————————————————————————————————————————————————————————————————————————
```


### 49. a_diff_means works in a table layout (stacked header) with ex_advs data

`id: test:test-diff_means:396` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-diff_means.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

advs_bp_blw12 <- local({
    res <- subset(ex_advs, PARAMCD %in% c("DIABP", "SYSBP") & AVISIT %in% c("BASELINE", 
        "WEEK 1 DAY 8", "WEEK 2 DAY 15"))
    res$PARAMCD <- droplevels(res$PARAMCD)
    res$AVISIT <- droplevels(res$AVISIT)
    res
})

advs_bp_blw12$colspan_trt <- factor(ifelse(advs_bp_blw12$ARM == "B: Placebo", " ", 
    "Active Study Agent"), levels = c("Active Study Agent", " "))

colsp_trt_map <- create_colspan_map(advs_bp_blw12, non_active_grp = "B: Placebo", 
    non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", 
    trt_var = "ARM")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colsp_trt_map)) |>
  split_cols_by("ARM") |>
  build_table(advs_bp_blw12)
```

```text
Active Study Agent                         
                                                     A: Drug X          C: Combination      B: Placebo
——————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 50. a_proportion_diff_j label param works correctly

`id: test:test-estimate_proportion_diff:23` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-estimate_proportion_diff.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

dta <- data.frame(rsp = sample(c(TRUE, FALSE), 100, TRUE), grp = factor(sample(c("A", 
    "B"), 100, TRUE)))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("grp", ref_group = "B") |>
  build_table(dta)
```

```text
A            B
——————————————————————————————————————————————
```


### 51. a_proportion_diff_j works as expected in a table layout

`id: test:test-estimate_proportion_diff:72` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-estimate_proportion_diff.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

nex <- 100

dta <- data.frame(rsp = sample(c(TRUE, FALSE), nex, TRUE), grp = sample(c("A", "B"), 
    nex, TRUE), f1 = sample(c("a1", "a2"), nex, TRUE), f2 = sample(c("x", "y", "z"), 
    nex, TRUE), stringsAsFactors = TRUE)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(var = "grp") |>
  build_table(df = dta)
```

```text
A            B
——————————————————————————————————————————————
```


### 52. a_proportion_diff_j works as expected in a table layout with uncond_exact_diff method

`id: test:test-estimate_proportion_diff:112` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-estimate_proportion_diff.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

dta <- data.frame(rsp = c(rep(TRUE, 5), rep(FALSE, 12), rep(TRUE, 40), rep(FALSE, 
    38)), grp = c(rep("B", 17), rep("A", 78)), stringsAsFactors = FALSE)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(var = "grp") |>
  build_table(df = dta)
```

```text
B             A          
—————————————————————————————————————————————————
```


### 53. get_ref_info works with a df analysis function

`id: test:test-get_ref_info:56` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-get_ref_info.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

dm <- formatters::DM

dm$colspan_trt <- factor(ifelse(dm$ARM == "B: Placebo", " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

colspan_trt_map <- create_colspan_map(dm, non_active_grp = "B: Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = "ARM")
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by("ARM") |>
  build_table(dm)
```

```text
Active Study Agent                 
                         A: Drug X   C: Combination   B: Placebo
————————————————————————————————————————————————————————————————
```


### 54. get_ref_info works with a df in the presence of the overall column

`id: test:test-get_ref_info:185` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-get_ref_info.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

dm <- formatters::DM

dm$colspan_trt <- factor(ifelse(dm$ARM == "B: Placebo", " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

colspan_trt_map <- create_colspan_map(dm, non_active_grp = "B: Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = "ARM")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(colspan_trt_map)) |>
  split_cols_by("ARM") |>
  add_overall_col("Total") |>
  build_table(dm)
```

```text
Active Study Agent                         
                                 A: Drug X   C: Combination   B: Placebo   Total
————————————————————————————————————————————————————————————————————————————————
```


### 55. get_ref_info returns reference information for matching column splits

`id: test:test-get_ref_info:241` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-get_ref_info.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

dm <- formatters::DM

dm$colspan_trt <- factor(ifelse(dm$ARM == "B: Placebo", " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

colspan_trt_map <- create_colspan_map(dm, non_active_grp = "B: Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = "ARM")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by("ARM") |>
  build_table(dm)
```

```text
Active Study Agent                 
                       A: Drug X   C: Combination   B: Placebo
——————————————————————————————————————————————————————————————
```


### 56. get_ref_info returns NULL reference information in risk-diff columns

`id: test:test-get_ref_info:288` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-get_ref_info.R`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` to directly construct the grouped treatment spanning headers and pairwise risk difference columns. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

dm <- ex_adsl

dm <- create_colspan_var(dm, non_active_grp = "B: Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = "ARM")

colspan_trt_map <- create_colspan_map(dm, non_active_grp = "B: Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = "ARM")
```

**Core layout instructions:**

```r
basic_table() |>
  grouped_cols_w_diffs(colspan_trt_map, diffs_label = "Risk Differences") |>
  build_table(dm)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

dm <- ex_adsl

dm$rrisk_header <- "Risk Differences"

dm$rrisk_label <- paste(dm$ARM, "vs", "B: Placebo")

dm$colspan_trt <- factor(ifelse(dm$ARM == "B: Placebo", " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

colspan_trt_map <- create_colspan_map(dm, non_active_grp = "B: Placebo", non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = "ARM")
```

**Fallback core layout instructions:**

```r
basic_table() |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by("ARM") |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by("ARM", labels_var = "rrisk_label", split_fun = remove_split_levels("B: Placebo")) |>
  build_table(dm)
```

</details>

```text
Active Study Agent                                        Risk Differences                   
   A: Drug X   C: Combination   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo
———————————————————————————————————————————————————————————————————————————————————————————————————
```

### 57. a_freq_j with alt_counts_df retains nested row structure

`id: test:test-h_freq_funs:308` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-h_freq_funs.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl <- ex_adsl

adsl <- pharmaverseadamjnj::adsl

adsl_ie <- tibble::tribble(~USUBJID, ~IECAT2, ~IETEST, ~TRT01A, ~REGION1, ~SITEID, 
    "01-701-1097", "Subjects who did not meet inclusion criteria", "Medication criteria", 
    "Xanomeline Low Dose", "NA", "701", "01-701-1203", "Subjects who did not meet inclusion criteria", 
    "Laboratory criteria", "Placebo", "NA", "701", "01-703-1086", "Subjects who did not meet inclusion criteria", 
    "Laboratory criteria", "Xanomeline Low Dose", "NA", "703")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("TRT01A") |>
  build_table(adsl_ie, alt_counts_df = adsl)
```

```text
Xanomeline Low Dose      Placebo   
——————————————————————————————————————————————————————————————————————————————————————
```


### 58. Check patient years numbers are giving expected result

`id: test:test-patyrs-eair100:129` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-patyrs-eair100.R`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` to directly generate grouped treatment arms and pairwise risk difference columns. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl <- select(mutate(mutate(mutate(mutate(mutate(ex_adsl, TRTDURY = substring(USUBJID, 
    nchar(USUBJID) - 3 + 1), "-", ""), TRTDURY = sub("-", "", TRTDURY)), TRTDURY = sub("d", 
    "", TRTDURY)), TRTDURY = as.numeric(TRTDURY)), TRTDURY2 = TRTDURY + 25), USUBJID, 
    ARM, COUNTRY, STRATA1, TRTDURY, TRTDURY2, SEX)

adae <- select(ex_adae, USUBJID, AEDECOD, AEBODSYS, ASTDY)

adae$TRTEMFL <- "Y"

adaefirst <- select(mutate(ungroup(slice(group_by(arrange(adae, USUBJID, AEBODSYS, 
    AEDECOD, ASTDY), USUBJID, AEBODSYS, AEDECOD), 1)), AOCCPFL = "Y"), USUBJID, AEBODSYS, 
    AEDECOD, ASTDY, AOCCPFL)

adae <- left_join(adae, adaefirst, by = c("USUBJID", "AEBODSYS", "AEDECOD", "ASTDY"))

ctrl_grp <- "B: Placebo"

trtvar <- "ARM"

adsl <- create_colspan_var(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

adae <- mutate(left_join(adsl, adae, by = "USUBJID"), ASTDY2 = ASTDY + 10)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = FALSE, round_type = "sas") |>
  grouped_cols_w_diffs(colspan_trt_map, diffs_label = "Risk Difference (95% CI)") |>
  build_table(adae, adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl <- select(mutate(mutate(mutate(mutate(mutate(ex_adsl, TRTDURY = substring(USUBJID, 
    nchar(USUBJID) - 3 + 1), "-", ""), TRTDURY = sub("-", "", TRTDURY)), TRTDURY = sub("d", 
    "", TRTDURY)), TRTDURY = as.numeric(TRTDURY)), TRTDURY2 = TRTDURY + 25), USUBJID, 
    ARM, COUNTRY, STRATA1, TRTDURY, TRTDURY2, SEX)

adae <- select(ex_adae, USUBJID, AEDECOD, AEBODSYS, ASTDY)

adae$TRTEMFL <- "Y"

adaefirst <- select(mutate(ungroup(slice(group_by(arrange(adae, USUBJID, AEBODSYS, 
    AEDECOD, ASTDY), USUBJID, AEBODSYS, AEDECOD), 1)), AOCCPFL = "Y"), USUBJID, AEBODSYS, 
    AEDECOD, ASTDY, AOCCPFL)

adae <- left_join(adae, adaefirst, by = c("USUBJID", "AEBODSYS", "AEDECOD", "ASTDY"))

ctrl_grp <- "B: Placebo"

trtvar <- "ARM"

adsl$colspan_trt <- factor(ifelse(adsl[["ARM"]] == ctrl_grp, " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

adsl$diffs_label <- "Risk Difference (95% CI)"

adsl$rrisk_label <- paste(adsl[["ARM"]], "vs", ctrl_grp)

adae <- mutate(left_join(adsl, adae, by = "USUBJID"), ASTDY2 = ASTDY + 10)
```

**Fallback core layout instructions:**

```r
basic_table(show_colcounts = FALSE, round_type = "sas") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by("ARM") |>
  split_cols_by("diffs_label", nested = FALSE) |>
  split_cols_by("ARM", labels_var = "rrisk_label", split_fun = remove_split_levels(ctrl_grp)) |>
  build_table(adae, adsl)
```

</details>

```text
Active Study Agent                                   Risk Difference (95% CI)               
   A: Drug X   C: Combination   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo
———————————————————————————————————————————————————————————————————————————————————————————————————
```


### 59. Check a_eair100_j numbers are giving expected result relative risk in combined facet is available

`id: test:test-patyrs-eair100:386` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-patyrs-eair100.R`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` with `combo_map_df` to directly generate virtual combination arms and their corresponding pairwise risk differences. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl <- select(mutate(mutate(mutate(mutate(mutate(ex_adsl, TRTDURY = substring(USUBJID, 
    nchar(USUBJID) - 3 + 1), "-", ""), TRTDURY = sub("-", "", TRTDURY)), TRTDURY = sub("d", 
    "", TRTDURY)), TRTDURY = as.numeric(TRTDURY)), TRTDURY2 = TRTDURY + 25), USUBJID, 
    ARM, COUNTRY, STRATA1, TRTDURY, TRTDURY2, SEX)

adae <- select(ex_adae, USUBJID, AEDECOD, AEBODSYS, ASTDY)

adae$TRTEMFL <- "Y"

adaefirst <- select(mutate(ungroup(slice(group_by(arrange(adae, USUBJID, AEBODSYS, 
    AEDECOD, ASTDY), USUBJID, AEBODSYS, AEDECOD), 1)), AOCCPFL = "Y"), USUBJID, AEBODSYS, 
    AEDECOD, ASTDY, AOCCPFL)

adae <- left_join(adae, adaefirst, by = c("USUBJID", "AEBODSYS", "AEDECOD", "ASTDY"))

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

adsl <- create_colspan_var(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

adae <- mutate(left_join(adsl, adae, by = "USUBJID"), ASTDY2 = ASTDY + 10)

combodf <- tibble::tribble(
  ~valname, ~label, ~levelcombo, ~exargs,
  "Combined", "Combined", setdiff(levels(adsl[[trtvar]]), ctrl_grp), list()
)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE, colcount_format = "N=xx", top_level_section_div = " ") |>
  grouped_cols_w_diffs(colspan_trt_map, combo_map_df = combodf, diffs_label = "Risk Difference (95% CI)") |>
  build_table(adae, alt_counts_df = adsl)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

library(dplyr)

adsl <- select(mutate(mutate(mutate(mutate(mutate(ex_adsl, TRTDURY = substring(USUBJID, 
    nchar(USUBJID) - 3 + 1), "-", ""), TRTDURY = sub("-", "", TRTDURY)), TRTDURY = sub("d", 
    "", TRTDURY)), TRTDURY = as.numeric(TRTDURY)), TRTDURY2 = TRTDURY + 25), USUBJID, 
    ARM, COUNTRY, STRATA1, TRTDURY, TRTDURY2, SEX)

adae <- select(ex_adae, USUBJID, AEDECOD, AEBODSYS, ASTDY)

adae$TRTEMFL <- "Y"

adaefirst <- select(mutate(ungroup(slice(group_by(arrange(adae, USUBJID, AEBODSYS, 
    AEDECOD, ASTDY), USUBJID, AEBODSYS, AEDECOD), 1)), AOCCPFL = "Y"), USUBJID, AEBODSYS, 
    AEDECOD, ASTDY, AOCCPFL)

adae <- left_join(adae, adaefirst, by = c("USUBJID", "AEBODSYS", "AEDECOD", "ASTDY"))

trtvar <- "ARM"

ctrl_grp <- "B: Placebo"

adsl$colspan_trt <- factor(ifelse(adsl[["ARM"]] == ctrl_grp, " ", "Active Study Agent"), 
    levels = c("Active Study Agent", " "))

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ", 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

adsl$rrisk_header <- "Risk Difference (95% CI)"

adsl$rrisk_label <- paste(adsl[["ARM"]], "vs", ctrl_grp)

adae <- mutate(left_join(adsl, adae, by = "USUBJID"), ASTDY2 = ASTDY + 10)

add_combo <- add_combo_facet("Combined", label = "Combined", levels = setdiff(unique(adsl[[trtvar]]), ctrl_grp))

rm_combo_from_placebo <- cond_rm_facets(facets = "Combined", ancestor_pos = NA, value = " ", split = "colspan_trt")

mysplit_comb <- make_split_fun(post = list(add_combo, rm_combo_from_placebo))

rm_combo_from_placebo2 <- cond_rm_facets(facets = ctrl_grp, ancestor_pos = NA, value = "Risk Difference (95% CI)", split = "rrisk_header")

add_combo2 <- add_combo_facet("Combined", label = paste0("Combined vs ", ctrl_grp), levels = setdiff(unique(adsl[[trtvar]]), ctrl_grp))

mysplit_comb2 <- make_split_fun(post = list(add_combo2, rm_combo_from_placebo2))
```

**Fallback core layout instructions:**

```r
basic_table(show_colcounts = TRUE, colcount_format = "N=xx", top_level_section_div = " ") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by(trtvar, split_fun = mysplit_comb) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by(trtvar, labels_var = "rrisk_label", split_fun = mysplit_comb2) |>
  build_table(adae, alt_counts_df = adsl)
```

</details>

```text
Active Study Agent                                                     Risk Difference (95% CI)                            
   A: Drug X   C: Combination   Combined   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo   Combined vs B: Placebo
     N=134         N=132         N=266       N=134               N=134                       N=132                       N=266         
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 60. a_proportion_diff_mf() works in full table build (large sample)

`id: test:test-proportion_diff_mf:490` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-proportion_diff_mf.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

n <- 100

data <- data.frame(rsp = sample(c(TRUE, FALSE), n, replace = TRUE), grp = sample(c("Placebo", 
    "X"), n, replace = TRUE), strata = factor(sample(LETTERS[1:4], n, replace = TRUE)))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by(var = "grp", ref_group = "Placebo") |>
  build_table(data)
```

```text
X           Placebo
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 61. prop_split_fun works as expected

`id: test:test-proportions:106` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-proportions.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("ID", split_fun = prop_split_fun) |>
  build_table(formatters::DM)
```

```text
n   %   Cum %
————————————————
```


### 62. a_relative_risk in table layout gives same results as with SAS

`id: test:test-relative_risk:357` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-relative_risk.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

dat <- data.frame(Patient_ID = c(6L, 8L, 14L, 16L, 22L, 24L, 30L, 32L, 38L, 40L, 
    46L, 48L, 3L, 4L, 11L, 12L, 19L, 20L, 27L, 28L, 35L, 36L, 43L, 44L, 50L, 5L, 
    7L, 13L, 15L, 21L, 23L, 29L, 31L, 37L, 39L, 45L, 47L, 1L, 2L, 9L, 10L, 17L, 18L, 
    25L, 26L, 33L, 34L, 41L, 42L, 49L), Response = c(FALSE, TRUE, TRUE, FALSE, FALSE, 
    TRUE, FALSE, TRUE, FALSE, TRUE, TRUE, FALSE, TRUE, FALSE, FALSE, TRUE, FALSE, 
    TRUE, FALSE, TRUE, FALSE, TRUE, FALSE, TRUE, FALSE, FALSE, TRUE, FALSE, FALSE, 
    FALSE, TRUE, FALSE, TRUE, TRUE, FALSE, FALSE, FALSE, FALSE, TRUE, FALSE, TRUE, 
    TRUE, FALSE, TRUE, FALSE, FALSE, TRUE, FALSE, TRUE, TRUE), Treatment = as.factor(c("A", 
    "A", "B", "B", "A", "A", "B", "B", "A", "A", "B", "B", "A", "A", "B", "B", "A", 
    "A", "B", "B", "A", "A", "B", "B", "A", "A", "A", "B", "B", "A", "A", "B", "B", 
    "A", "A", "B", "B", "A", "A", "B", "B", "A", "A", "B", "B", "A", "A", "B", "B", 
    "A")), Gender = as.factor(c("Female", "Female", "Female", "Female", "Female", 
    "Female", "Female", "Female", "Female", "Female", "Female", "Female", "Female", 
    "Female", "Female", "Female", "Female", "Female", "Female", "Female", "Female", 
    "Female", "Female", "Female", "Female", "Male", "Male", "Male", "Male", "Male", 
    "Male", "Male", "Male", "Male", "Male", "Male", "Male", "Male", "Male", "Male", 
    "Male", "Male", "Male", "Male", "Male", "Male", "Male", "Male", "Male", "Male")), 
    Age_Group = as.factor(c("Old", "Old", "Old", "Old", "Old", "Old", "Old", "Old", 
        "Old", "Old", "Old", "Old", "Young", "Young", "Young", "Young", "Young", 
        "Young", "Young", "Young", "Young", "Young", "Young", "Young", "Young", "Old", 
        "Old", "Old", "Old", "Old", "Old", "Old", "Old", "Old", "Old", "Old", "Old", 
        "Young", "Young", "Young", "Young", "Young", "Young", "Young", "Young", "Young", 
        "Young", "Young", "Young", "Young")))
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("Treatment") |>
  build_table(dat)
```

```text
A            B
—————————————————————————————————————————————————
```


### 63. resp01_split_fun_fct 1 works as expected

`id: test:test-resp01_functions:8` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-resp01_functions.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

split_fun <- resp01_split_fun_fct(method = "or_cmh", conf_level = 0.95)
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("ARM", split_fun = add_overall_level("Overall")) |>
  split_cols_by("ID", split_fun = split_fun) |>
  build_table(formatters::DM)
```

```text
Overall                             A: Drug X              B: Placebo           C: Combination   
   Odds Ratio (95% CI)~[super a]   p-value~[super b]   n (%)   95% CI for %   n (%)   95% CI for %   n (%)   95% CI for %
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 64. resp01_split_fun_fct 2 works as expected

`id: test:test-resp01_functions:20` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-resp01_functions.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

split_fun <- resp01_split_fun_fct(method = "rr", conf_level = 0.92)
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("ARM", split_fun = add_overall_level("Overall")) |>
  split_cols_by("ID", split_fun = split_fun) |>
  build_table(formatters::DM)
```

```text
Overall                               A: Drug X              B: Placebo           C: Combination   
   Relative Risk (92% CI)~[super a]   p-value~[super b]   n (%)   92% CI for %   n (%)   92% CI for %   n (%)   92% CI for %
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 65. a_summarize_ancova_j  works as expected in table layout

`id: test:test-summarize_ancova:141` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-summarize_ancova.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)
```

**Core layout instructions:**

```r
basic_table(round_type = "sas") |>
  split_cols_by("Species") |>
  add_colcounts() |>
  build_table(iris)
```

```text
setosa         versicolor           virginica    
                                                                    (N=50)           (N=50)              (N=50)      
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 66. a_summarize_ancova_j with sparse data

`id: test:test-summarize_ancova:513` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-summarize_ancova.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

iris_plus <- mutate(iris, Color = factor(sample(c("red", "blue"), size = nrow(iris), 
    prob = c(0.8, 0.2), replace = TRUE)))

iris_plus2 <- ungroup(filter(mutate(group_by(iris_plus, Species), id = row_number()), 
    Species == "setosa" & id < 45 | Species == "versicolor" & id < 30 | Species == 
        "virginica" & id < 50))

iris_sparse <- filter(iris_plus2, Species != "versicolor")
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("Species") |>
  add_colcounts() |>
  build_table(iris_sparse)
```

```text
setosa         versicolor       virginica    
                                                                      (N=44)           (N=0)           (N=49)      
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 67. summarize_row_counts works as expected without alt counts

`id: test:test-summarize_row_counts:11` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-summarize_row_counts.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARM") |>
  add_colcounts() |>
  build_table(formatters::DM)
```

```text
A: Drug X   B: Placebo   C: Combination
                                   (N=121)     (N=106)        (N=129)    
—————————————————————————————————————————————————————————————————————————
```


### 68. lsmeans_wide_first_split_fun_fct with variance works as expected

`id: test:test-tabulate_lsmeans_wide:7` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-tabulate_lsmeans_wide.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

split_fun <- lsmeans_wide_first_split_fun_fct(include_variance = TRUE)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ID", split_fun = split_fun) |>
  build_table(DM)
```

```text
Reference Group   Testing Group      Testing-Reference
—————————————————————————————————————————————————————————
```


### 69. lsmeans_wide_first_split_fun_fct without variance works as expected

`id: test:test-tabulate_lsmeans_wide:16` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-tabulate_lsmeans_wide.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

split_fun <- lsmeans_wide_first_split_fun_fct(include_variance = FALSE)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ID", split_fun = split_fun) |>
  build_table(DM)
```

```text
Reference Group   Testing Group   Testing-Reference
——————————————————————————————————————————————————————
```


### 70. lsmeans_wide_second_split_fun_fct works as expected

`id: test:test-tabulate_lsmeans_wide:30` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-tabulate_lsmeans_wide.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

split_fun <- lsmeans_wide_second_split_fun_fct(include_pval = TRUE, pval_sided = "2", 
    conf_level = 0.92)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ID", split_fun = lsmeans_wide_first_split_fun_fct(FALSE)) |>
  split_cols_by("ID", split_fun = split_fun) |>
  build_table(DM)
```

```text
Reference Group                 Testing Group                           Testing-Reference                
   Treatment   N   LS Mean   SE   Treatment   N   LS Mean   SE   LS Mean   SE   92% CI   2-sided p-value~[super a]
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 71. lsmeans_wide_second_split_fun_fct works with variance and without pval

`id: test:test-tabulate_lsmeans_wide:44` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-tabulate_lsmeans_wide.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

split_fun <- lsmeans_wide_second_split_fun_fct(include_pval = FALSE, pval_sided = "1", 
    conf_level = 0.78)
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ID", split_fun = lsmeans_wide_first_split_fun_fct(TRUE)) |>
  split_cols_by("ID", split_fun = split_fun) |>
  build_table(DM)
```

```text
Reference Group                 Testing Group                                       Testing-Reference   
   Treatment   N   LS Mean   SE   Treatment   N   LS Mean   SE   M. S. Error   Error DF   LS Mean    SE    78% CI
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```


### 72. a_lsmeans can show two- and one-sided p-values correctly

`id: test:test-tabulate_lsmeans:172` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-tabulate_lsmeans.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

mmrm_results <- fit_mmrm_j(vars = list(response = "FEV1", covariates = c("RACE", 
    "SEX"), id = "USUBJID", arm = "ARMCD", visit = "AVISIT"), data = mmrm::fev_data, 
    cor_struct = "unstructured", weights_emmeans = "equal", averages_emmeans = list(`VIS1+2` = c("VIS1", 
        "VIS2")))

df <- broom::tidy(mmrm_results)

dat_adsl <- unique(dplyr::select(mmrm::fev_data, USUBJID, ARMCD))
```

**Core layout instructions:**

```r
basic_table() |>
  split_cols_by("ARMCD") |>
  add_colcounts() |>
  build_table(df = df, alt_counts_df = dat_adsl)
```

```text
PBO                       TRT          
                                                    (N=105)                   (N=95)         
—————————————————————————————————————————————————————————————————————————————————————————————
```


### 73. tt_to_tlgrtf doesn't lose titles

`id: test:test-tt_to_tblfile:537` | Source: `/Users/gbecker/gabe/checkedout/junco/tests/testthat/test-tt_to_tblfile.R`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

popfl <- "SAFFL"

trtvar <- "ARM"

adsl <- mutate(ex_adsl, `:=`(!!rlang::sym(trtvar), factor(.data[[trtvar]], levels = c("Dummy A - Tec-DR", 
    "Dummy B - Tal-DR", "Dummy C - DRd"))), SEX = factor(case_when(SEX == "M" ~ "Male", 
    SEX == "F" ~ "Female", TRUE ~ SEX), levels = c("Male", "Female", "Untersex", 
    "Unknown")))

adae <- mutate(ex_adae, `:=`(!!rlang::sym(trtvar), factor(.data[[trtvar]], levels = c("Dummy A - Tec-DR\t", 
    "Dummy B - Tal-DR", "Dummy C - DRd"))), SEX = factor(case_when(SEX == "M" ~ "Male", 
    SEX == "F" ~ "Female", TRUE ~ SEX), levels = c("Male", "Female", "Untersex", 
    "Unknown")))

adsl <- select(create_colspan_var(filter(adsl, !!rlang::sym(popfl) == "Y"), non_active_grp = NULL, 
    non_active_grp_span_lbl = NULL, active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", 
    trt_var = trtvar), USUBJID, !!rlang::sym(popfl), !!rlang::sym(trtvar), colspan_trt)

adae <- group_by(select(mutate(filter(mutate(adae, TRTEMFL = "Y"), TRTEMFL == "Y"), 
    TEAEFL = "Y", RELGR1 = "Y"), USUBJID, TEAEFL, RELGR1), USUBJID)

adae <- inner_join(adae, adsl, by = c("USUBJID"))

colspan_trt_map <- create_colspan_map(adsl, non_active_grp = NULL, non_active_grp_span_lbl = NULL, 
    active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = trtvar)

adae_combined <- bind_rows(mutate(ungroup(adae), AETOXGR_MAX = NA_character_))
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE, colcount_format = "N=xx") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by(trtvar) |>
  add_overall_col("Total") |>
  build_table(adae_combined, alt_counts_df = adsl, round_type = "sas")
```

```text
Active Study Agent                               
                       Dummy A - Tec-DR   Dummy B - Tal-DR   Dummy C - DRd      Total   
Event, n (%)                 N=0                N=0               N=0           N=400   
————————————————————————————————————————————————————————————————————————————————————————
```


### 74. Multi-Reference Pairwise Comparisons (Active-Controlled Trials)

`id: missing_col_structures#4` | Source: `missing_col_structures_impl.md`

> [!NOTE]
> **Primary Solution (using `junco`)**: The primary solution below uses `junco::grouped_cols_w_diffs()` with `comp_map` to specify multi-reference pairwise comparisons directly. This solution is contingent on `junco` being installed (`library(junco)`). If `junco` is unavailable, the manual `rtables` approach using `make_multicomp_splfun()` is retained below as a fallback-only secondary solution.

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

dat <- ex_adsl

dat <- create_colspan_var(dat, non_active_grp = "B: Placebo", non_active_grp_span_lbl = "Control", 
    active_grp_span_lbl = "Active", colspan_var = "span_lbl", trt_var = "ARM")

trt_map <- create_colspan_map(dat, non_active_grp = "B: Placebo", non_active_grp_span_lbl = "Control", 
    active_grp_span_lbl = "Active", colspan_var = "span_lbl", trt_var = "ARM")

comp_map <- data.frame(
  active = c("A: Drug X", "C: Combination", "C: Combination"), 
  comparator = c("B: Placebo", "B: Placebo", "A: Drug X")
)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  grouped_cols_w_diffs(trt_map, comp_map = comp_map) |>
  build_table(dat)
```

<details>
<summary><b>Fallback-only secondary solution (pure <code>rtables</code>)</b></summary>

**Fallback setup:**

```r
library(rtables)

library(junco)

library(dplyr)

dat <- ex_adsl

dat$diff_lab <- "Risk Differences"

comp_map <- data.frame(active = c("A: Drug X", "C: Combination", "C: Combination"), 
    comparator = c("B: Placebo", "B: Placebo", "A: Drug X"))

trt_map <- data.frame(span_lbl = c("Control", "Active", "active"), ARM = c("B: Placebo", 
    "A: Drug X", "C: Combination"))

splfun <- make_multicomp_splfun(trt_map, comp_level_map = comp_map)
```

**Fallback core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  split_cols_by("diff_lab", nested = FALSE) |>
  split_cols_by("ARM", split_fun = splfun) |>
  build_table(dat)
```

</details>

```text
                                                                               Risk Differences                                  
   A: Drug X   B: Placebo   C: Combination   A: Drug X vs B: Placebo   C: Combination vs B: Placebo   C: Combination vs A: Drug X
    (N=134)     (N=134)        (N=132)               (N=134)                     (N=132)                        (N=132)          
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
```

### 75. A/B/n Product Experimentation with Uplift Columns

`id: missing_col_structures#11` | Source: `missing_col_structures_impl.md`

**Setup for reproducibility:**

```r
library(rtables)

library(junco)

library(dplyr)

dat <- data.frame(group = factor(c("Control", "Control", "Control", "Variant 1", "Variant 1", "Variant 1", "Variant 2", "Variant 2", "Variant 2")))
dat$lift <- dat$group
trt_colmap <- data.frame(span = c("Variant", "Variant", "Control"), group = c("Variant 1", "Variant 2", "Control"))

trt_colmap <- data.frame(span = c("Variant", "Variant", "Control"), group = c("Variant 1", 
    "Variant 2", "Control"))

compspl <- make_multicomp_splfun(trt_colmap)
```

**Core layout instructions:**

```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("group") |>
  split_cols_by("lift", split_fun = compspl, nested = FALSE) |>
  build_table(dat)
```

```text
   Control   Variant 1   Variant 2   Variant 1 vs Control   Variant 2 vs Control
    (N=3)      (N=3)       (N=3)            (N=3)                  (N=3)        
————————————————————————————————————————————————————————————————————————————————
```

