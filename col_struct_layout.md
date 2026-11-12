---
name: rtables-column-structure-programming
description: Design, implement, and customize column header structures and column splitting hierarchies in R using the `rtables` and `junco` packages. Activate this skill whenever the user wants to construct table column layouts (`split_cols_by*`, `add_overall_col`, `add_colcounts`), write custom column split functions (`make_split_fun`, `make_split_result`, `add_combo_levels`, `add_combo_facet`, `restrict_facets`, `trim_levels_in_group`, `trim_levels_to_map`), build non-nested or ragged/asymmetric banners, configure column counts (`alt_counts_df`, `col_counts`, `facet_colcount`, `facet_colcounts_visible`), or implement `junco` high-level and extension column structures (`grouped_cols_w_diffs`, `grouped_cols_w_subgrps`, `shift_tbl_col_struct`, `some_v_all_col_struct`, `create_colspan_var`, `create_colspan_map`, `make_multicomp_splfun`, `stats_in_cols_setup`).
---

# `rtables` & `junco` Column Structure Programming Skill

This skill provides a complete engineering guide and pattern catalog for programming column structures in `rtables` and `junco`, synthesized from 232 verified column layout patterns across clinical trials, biostatistics, survey research, finance, and product analytics.

> [!IMPORTANT]
> **Implementation Strategy: High-Level `junco` Primary with Manual `rtables` Fallback**
> For standard clinical reporting topologies (grouped treatment spanners with difference columns, treatment arms subdivided by subgroups, baseline shift matrices, and some vs. all subsetting), **use the high-level `junco` functions as the primary solution** when `junco` is installed (`library(junco)`). These functions encapsulate complex multi-step layout logic and automatically wire reference paths (`ref_path`) for downstream analysis functions. Retain the manual `rtables` approach as a **fallback-only secondary solution** when `junco` is unavailable.

---

## 1. Core Tabulation Model & Invariants

In `rtables`, a table layout (`PreDataTableLayouts`) is a declarative tree of splitting instructions evaluated against a dataset via `build_table(lyt, df, ...)`:

1. **Column Structure Exclusivity**:
   - Column structure is defined **exclusively** by zero or more calls to the `split_cols_by*` family of functions (`split_cols_by`, `split_cols_by_multivar`, `split_cols_by_cuts`, `split_cols_by_cutfun`, `split_cols_by_quartiles`, `split_cols_by_groups`), high-level layout generators (`grouped_cols_w_diffs`, `grouped_cols_w_subgrps`, `shift_tbl_col_struct`, `some_v_all_col_struct`), along with `add_overall_col`, `add_colcounts`, and `append_topleft`.
   - Row-splitting functions (`split_rows_by*`), group summaries (`summarize_row_groups`), and analysis functions (`analyze`, `analyze_colvars`) **never** alter column structure.
2. **Tree Nesting vs. Sibling Subtrees**:
   - By default, each subsequent `split_cols_by*` call **nests** under every leaf facet currently in the column layout tree (`nested = TRUE`).
   - Passing `nested = FALSE` (or calling `add_overall_col("Header")` before a subsequent `split_cols_by*`) places the new split as a **parallel sibling branch** at the top level of the column tree rather than nesting inside existing leaves.
3. **Two-Phase Evaluation**:
   - **Layout time**: `basic_table() |> split_cols_by(...)` constructs a pre-data column layout tree.
   - **Build time**: `build_table(lyt, df, alt_counts_df = ..., col_counts = ...)` recursively partitions `df` (and `alt_counts_df` when provided for header `(N=xx)` denominators) down the column split tree to materialize the `InstantiatedColumnInfo` object.
   - **Post-build time**: Column counts, visibility, and top-left labels can be inspected or mutated on the built `TableTree` via `col_counts(tbl)<-`, `facet_colcount(tbl, path)<-`, `facet_colcounts_visible(tbl, path)<-`, `colcount_na_str(tbl)<-`, and `top_left(tbl)<-`.

---

## 2. Decoding Visual Topology

Complete Visual Topology to Implementation Strategy Workflow:

1. **Initial Lookup In Decision Matrix (2.1)**: Attempt to match the full header topology against the lookup matrix; if successful, immediately follow the implementation guidance.
2. **Infer Column Structure From Visual And Semantic Clues (2.2)**: When a header does not directly match a single entry in 2.1, strip away top-left stub labels and use spanning alignment, repetition of sub-headers, and semantic relationships between adjacent columns to decompose the header into its underlying column tree or constituent sub-structures.
3. **Second-Pass Lookup In Decision Matrix (2.1)**: Match the normalized structure (or each decomposed sub-branch inferred in Step 2) against the API decision matrix; if successful, immediately follow and combine the implementation guidance.
4. **Directly Construct Column Structure From API (Sections 3–5)**: Use the core API reference and custom split function mechanics to build the implementation directly.

### 2.1 Quick Lookup Visual Topology to API Decision Matrix

Use this lookup table to map a target visual header structure to the recommended mechanism:

| Target Header Topology | Recommended Primary Mechanism (contingent on `junco`) | Fallback Mechanism (pure `rtables`) |
| :--- | :--- | :--- |
| **Spanning Treatment Header + Multi-Comparator Risk Diffs** (`Active [Arms] \| Placebo \| Risk Diffs`) | **`grouped_cols_w_diffs(colspan_trt_map, diffs_label, ...)`** | Chained `split_cols_by("colspan_trt") \|> split_cols_by(trt) \|> split_cols_by("rrisk_header", nested = FALSE) \|> split_cols_by(trt, split_fun = make_multicomp_splfun(...))` |
| **Treatment Arms Subdivided by Subgroups (with within-arm Total)** (`Arm [Total \| Subgroups]`) | **`grouped_cols_w_subgrps(colspan_trt_map, subgrp_var, total_facet = TRUE)`** | `split_cols_by(trt) \|> split_cols_by(subgrp, split_fun = add_overall_level("Total", first = TRUE))` |
| **Treatment Arms Subdivided by Quartiles / Continuous Bins** | **`grouped_cols_w_subgrps(colspan_trt_map, subgrp_var, total_facet = FALSE)`** | `split_cols_by(trt) \|> split_cols_by_cuts(var, cuts, ...)` |
| **Clinical Laboratory / Toxicity Shift Table** (`N \| Baseline [Low \| Normal \| High] \| Total`) | **`shift_tbl_col_struct(var, span_lbl)`** | Two-level `make_split_fun` checking `.spl_context$value[[1]] == "N"` |
| **"Some vs All" / Non-Mutually Exclusive Cohort Subsets** (e.g. All AEs vs Related AEs, All Grades vs High Grade) | **`some_v_all_col_struct(colspan_trt_map, subgrp_var, subgrp_lvls, all_lbl, subgrp_lbl)`** | `make_split_fun(post = list(add_overall_facet, add_combo_facet, restrict_facets))` |
| **Single categorical header** (`Arm A \| Arm B \| Arm C`) | Standard column split: `split_cols_by("ARM")` | — |
| **Arms + Trailing/Leading Total** (`Arm A \| Arm B \| Total`) | `split_cols_by("ARM", split_fun = add_overall_level("Total", first = FALSE))` or `add_overall_col("Total")` | — |
| **Conditional Overall Column** (only add Total if $\ge 2$ facets exist) | Custom split function checking `length(vals) > 1` | — |
| **Multi-level Factorial Hierarchy** (`Arm -> Sex -> Smoke` or `Arm -> Visit`) | Chained nested `split_cols_by` calls | — |
| **Measure-First vs. Arm-First** (`Param 1 [Arm A \| Arm B]` vs `Arm A [Param 1 \| Param 2]`) | Order of `split_cols_by_multivar` vs `split_cols_by` | — |
| **Longitudinal Visits with Conditional Subcolumns** (`Baseline [Obs] \| Week 1 [Obs \| CFB]`) | `split_cols_by("AVISIT") \|> split_cols_by_multivar(c("AVAL", "CHG"), split_fun = make_split_fun(post = list(drop_cfb_at_baseline)))` | — |
| **Asymmetric / Ragged Split Trees** (Control unstratified; Active stratified by ADA + Subtotal) | Context-aware `.spl_context` in `make_split_fun(post = ...)` | — |
| **Parallel Non-Nested "Banner Table"** (`Total \| Sex [M \| F] \| Age [<35 \| >=35] \| Region`) | Repeat `add_overall_col("Header") \|> split_cols_by(...)` for each banner section | — |
| **Variable-Depth Sibling Branches** (Depth-1 `All Patients` next to Depth-2 `Arm -> Sex`) | `add_overall_col("All") \|> split_cols_by("ARM", nested = FALSE) \|> split_cols_by("SEX")` | — |
| **Pooled / Combined Cohorts** (`Low \| High \| Pooled Active`) | `add_combo_levels(combodf)` or `add_combo_facet` | — |
| **Per-Parent Data-Driven Quantiles** (Cutpoints recomputed within each Arm) | `make_split_fun(core_split = per_facet_qrt_core("AGE"))` | — |
| **Overlapping Multi-Variable Predicate Columns** (`Safety \| ITT \| Age >= 65 \| Biomarker High`) | Custom `core_split` evaluating a named list of quoted expressions via `eval(p, df)` | — |
| **Top-$k$ Frequent Levels + Pooled "Other" Column** | `make_split_fun(post = list(top_k_plus_other_post))` | — |

When no suitable match is found, escalate to semantic decoding of the specific desired topology.

### 2.2 Inferring Column Structure Via Semantic Decoding of Visual Topology

Use the following four rules to infer column structure directly from rendered tables or mock shells.

#### 1. Use Placement of Higher-Level Spanning Labels to Infer Column Structure

**Spanning labels appear centered above the full horizontal width of the child facets nested beneath them.** (In mock shells, this extent may also be marked by underline bars `---`, bracket groups, or parent-level `(N=xx)` counts).

* **Case A — `Total` facet is a direct sibling of treatment level facets (`A: Drug X`, `B: Placebo`, `C: Combination`) *under* the `Treatment` spanning label** (notice `Treatment` is centered across the full width of all 4 columns, columns 1–4):

```text
                       Treatment                   
   Total   A: Drug X   B: Placebo   C: Combination
——————————————————————————————————————————————————
```

Implementation Guidance: Use a split function that adds a `Total` facet *inside* the treatment (`ARM`) split under the `"span"` parent split (along with any other required behaviors):

```r
adsl <- ex_adsl
adsl$span <- "Treatment"
basic_table() |>
  split_cols_by("span") |>
  split_cols_by("ARM", split_fun = add_overall_level("Total"))
```

* **Case B — `Total` is *outside* the spanning label (and thus outside the `ARM` split)** (notice `Treatment` is shifted right so it is centered only over the 3 arm columns `A: Drug X` through `C: Combination`, columns 2–4, leaving `Total` unspanned on the left):

```text
                           Treatment               
   Total   A: Drug X   B: Placebo   C: Combination
——————————————————————————————————————————————————
```

Implementation Guidance: Use `add_overall_col("Total")` followed by `nested = FALSE` on the spanning split to separate parallel column subhierarchies:

```r
adsl <- ex_adsl
adsl$span <- "Treatment"
basic_table() |>
  add_overall_col("Total") |>
  split_cols_by("span", nested = FALSE) |>
  split_cols_by("ARM")
```

**Key Takeaway**: Check which leaf columns a higher-level spanning label is centered across (or underlines) to determine whether a `Total` or comparison column is nested inside that split (`add_overall_level` / `add_combo_levels`) or sits in a sibling subtree (`nested = FALSE` / `add_overall_col`).

#### 2. Repeated Sub-Headers Imply Nesting (`nested = TRUE`); Distinct Sub-Headers Imply Sibling Branches (`nested = FALSE`)

* **Repeated lower-level headers under each parent header** indicate a **nested (crossed) split** (`nested = TRUE`, the default):

```text
           A: Drug X            B: Placebo     
        Observed    CFB      Observed    CFB   
———————————————————————————————————————————————
```

Implementation Guidance: Chain `split_cols_by("ARM") |> split_cols_by_multivar(c("AVAL", "CHG"), varlabels = c("Observed", "CFB"))` (or `split_cols_by("OUTER") |> split_cols_by("INNER")`).

* **Non-repeating lower-level headers under distinct top-level spanners** indicate **sibling subtrees** (`nested = FALSE` or `add_overall_col("Banner") |> split_cols_by(...)`):

```text
       Active Treatment                  Risk Differences        
  A: Drug X   C: Combination   Drug X vs Pbo   Combination vs Pbo
—————————————————————————————————————————————————————————————————
```

Implementation Guidance: Because the child columns under `Risk Differences` do not repeat inside `Active Treatment`, `Risk Differences` is a parallel top-level sibling branch (`split_cols_by("rrisk_header", nested = FALSE)` or `grouped_cols_w_diffs()`).

#### 3. Direct Siblings Will Be Semantically Related

**Direct siblings are intended to be understood in the context of each other**, either as:
- groups/cohorts to be compared (as when splitting on treatment arm), or
- complementary measures to be considered together (as with a `Mean`, `SD` pair of columns).

For example, `All Active` is semantically parallel to individual treatment arms (a cohort of patients analyzed alongside the individual arms):

```text
   A: Drug X   B: Placebo   C: Combination   All Active
———————————————————————————————————————————————————————
```

Implementation Guidance: Interpret `All Active` as a virtual combination arm within the `ARM` split:

```r
combodf <- data.frame(
  valname = "All Active",
  label = "All Active",
  levelcombo = I(list(c("A: Drug X", "C: Combination"))),
  exargs = I(list(list()))
)
basic_table() |>
  split_cols_by("ARM", split_fun = add_combo_levels(combodf))
```

**Key Takeaway**: Consider whether labels appearing directly adjacent to each other are related semantically when determining whether they should be siblings in the same split (`add_combo_levels` / `add_overall_level`) vs. separate non-nested splits.

#### 4. Interpret Fully Left-Justified / Far-Left Labels as "Top-Left" Materials

**Top-left materials are semantic stub labels that do NOT imply column structure**: they appear above the *row labels* (in the top-left corner to the left of the first data column) in a rendered table or shell. They often describe *row* structure when present, though they can contain any text.

```text
STRATA1   A: Drug X   B: Placebo   C: Combination
—————————————————————————————————————————————————
A                                                
  Mean      33.08       35.11          34.23     
B                                                
  Mean      33.85       36.00          36.33     
C                                                
  Mean      34.22       35.18          35.63  
```

Implementation Guidance: Ignore the far-left `STRATA1` label in the header when inferring column structure; leave top-left population (`label_pos = "topleft"` on `split_rows_by()` or `append_topleft()`) to row structure or final visual tweaks:

```r
basic_table() |>
  split_cols_by("ARM") ## top-left population left for row structure / append_topleft()
```

---

## 3. Core Column Splitting API Reference

### 3.1 `basic_table()` & Table-Level Column Count Options
```r
basic_table(
  title = "",
  subtitles = character(),
  main_footer = character(),
  prov_footer = character(),
  show_colcounts = FALSE,        # Show (N=xx) under all leaf columns
  colcount_format = "(N=xx)",    # Format string for column counts, e.g. "N=xx" or "xx (xx.%)"
  top_level_section_div = NA_character_
)
```
- `add_colcounts(lyt, format = "(N=xx)")`: Turns on column counts at the current layout depth (set `format = "N=xx"` when the shell omits parentheses).
- `append_topleft(lyt, newlines)`: Appends text lines to the top-left corner area (to the left of the column headers and above the row labels).
- **Shell column-restraint rule**: Match the exact leaf columns in the shell header—never add any unrequested column not called for directly by the shell.

### 3.2 `split_cols_by()` (Categorical / Factor Splitting)
```r
split_cols_by(
  lyt,
  var,                           # Name of variable in df to split on
  labels_var = var,              # Optional variable in df providing display labels for each level of var
  split_label = var,             # Split label
  split_fun = NULL,              # Custom or built-in split function
  format = NULL,                 # Default cell format for subtable
  nested = TRUE,                 # TRUE = nest under current leaves; FALSE = add as top-level sibling
  child_labels = c("default", "visible", "hidden"),
  extra_args = list(),           # Extra arguments passed down to analysis functions per facet
  ref_group = NULL,              # Reference level (e.g. "B: Placebo") for comparison analysis functions
  show_colcounts = FALSE,        # Show (N=xx) at this specific split level (including parent spanning levels!)
  colcount_format = NULL
)
```

### 3.3 Continuous / Numeric Column Splits
- `split_cols_by_cuts(lyt, var, cuts, cutlabels = NULL, split_label = var, cumulative = FALSE, ...)`: Splits continuous `var` into intervals. Setting `cumulative = TRUE` produces cumulative intervals (`<= cut1`, `<= cut2`, ...).
- `split_cols_by_cutfun(lyt, var, cutfun, cutlabelfun = function(x) NULL, cumulative = FALSE, ...)`: Computes dynamic cutpoints from the full dataset at build time.
- `split_cols_by_quartiles(lyt, var, split_label = var, extra_args = list(), ...)`: Splits `var` into 4 quartile buckets across the dataset.
- `split_cols_by_groups(lyt, var, groups, ...)`: Splits categorical `var` according to a named list of character vectors `groups`.

### 3.4 Multi-Variable Column Splits (`split_cols_by_multivar`)
```r
split_cols_by_multivar(
  lyt,
  vars,                          # Character vector of variable names, e.g. c("AVAL", "CHG")
  split_label = NULL,
  varlabels = vars,              # Display headers for each variable, e.g. c("Observed", "CFB")
  varnames = NULL,               # Unique internal path names (required if vars contains duplicates!)
  format = NULL,
  nested = TRUE,
  extra_args = list(),
  split_fun = NULL,
  show_colcounts = FALSE
)
```

### 3.5 `add_overall_col()` (Top-Level Column & Parallel Banner Idiom)
- **Single trailing/leading Total column**: `basic_table() |> split_cols_by("ARM") |> add_overall_col("All Patients")`
- **Spanning Header for Parallel Non-Nested Subtrees (The Banner Pattern)**: Calling `split_cols_by(...)` immediately after `add_overall_col("Banner Label")` nests inside that overall column, making `"Banner Label"` a spanning banner header.

---

## 4. Built-in Split Functions (`split_fun`)

| Built-in Split Function | Purpose & Behavior |
| :--- | :--- |
| `drop_split_levels` | Drops unobserved factor levels (`N=0`) present in `df[[var]]` within the current facet. |
| `keep_split_levels(only)` | Keeps only the specified level names in `only`. |
| `remove_split_levels(excl)` | Excludes the specified level names in `excl` (e.g., `c("U", "UNDIFFERENTIATED")`). |
| `reorder_split_levels(neworder, newlabels = neworder, drlevels = TRUE)` | Explicitly reorders (and optionally relabels) facets to match `neworder`. |
| `add_overall_level(valname = "Overall", label = valname, first = TRUE, ...)` | Adds an overall/subtotal facet at the current split level. |
| `add_combo_levels(combodf, trim = FALSE, first = FALSE, keep_levels = NULL)` | Appends virtual combined facets defined by a `data.frame` or `tibble::tribble` with columns `valname`, `label`, `levelcombo`, and `exargs`. |
| `trim_levels_in_group(inner_var, drop_outlevs = TRUE)` | Applied on an outer split so each outer facet drops unobserved levels of `inner_var` before the inner split runs. |
| `trim_levels_to_map(map = df_map)` | Restricts valid `(outer_var, inner_var)` combinations to the rows explicitly listed in `map`. |

---

## 5. Custom Split Function Engineering (`make_split_fun`)

```r
custom_splfun <- make_split_fun(
  pre = list(...),        # Pre-processing hooks: function(df, spl, vals, labels, .spl_context) -> df
  core_split = NULL,      # Custom core split: function(spl, df, vals, labels, .spl_context) -> SplitResult
  post = list(...)        # Post-processing hooks: function(ret, spl, fulldf, .spl_context) -> SplitResult
)
```

Core rules for custom split functions:
- A `core_split` **must** return `make_split_result(values, datasplit, labels, extras = NULL, subset_exprs)`.
- Always supply `subset_exprs` (even if `list(quote(TRUE))`) so column counts and `alt_counts_df` path evaluations work properly.
- Built-in post-processing functions include `add_overall_facet()`, `add_combo_facet()`, `restrict_facets(facets, op = "keep", reorder = TRUE)`, and `add_to_split_result()`.

---

## 6. Canonical Advanced Column Structure Archetypes

> **Quick Index** — Archetype → key functions / concepts:
>
> | # | Archetype | Key Functions / Concepts |
> |---|---|---|
> | 1 | Grouped Treatments with Pairwise Difference Columns | `grouped_cols_w_diffs()`, `create_colspan_var()`, `create_colspan_map()`, combo arms, risk-difference columns |
> | 2 | Treatment Arms Subdivided by Subgroups (± Totals) | `grouped_cols_w_subgrps()`, `add_overall_level()`, within-arm Total facet |
> | 3 | Shift Table (Clinical Laboratory / Toxicity) | `shift_tbl_col_struct()`, cross-tabulation columns, N / baseline-level / Total structure |
> | 4 | "Some vs All" / Non-Mutually Exclusive Comparisons | `some_v_all_col_struct()`, `add_combo_facet()`, `restrict_facets()` |
> | 5 | Longitudinal Visits with Conditional Change-from-Baseline | `split_cols_by_multivar()`, `.spl_context` post-hook, conditional facet removal |
> | 6 | Per-Parent Data-Driven Quantile Bins | `make_split_fun(core_split = ...)`, per-facet `quantile()` + `cut()` |

### Archetype 1: Grouped Treatments with Pairwise Difference Columns

#### Primary Solution (using `junco`):
```r
library(rtables)
library(junco)

# 1. Prepare data and spanning map
adsl <- create_colspan_var(
  adsl,
  non_active_grp = "Placebo",
  non_active_grp_span_lbl = " ",
  active_grp_span_lbl = "Active Study Agent",
  colspan_var = "colspan_trt",
  trt_var = "TRT01A"
)
trt_map <- create_colspan_map(adsl, non_active_grp = "Placebo", non_active_grp_span_lbl = " ", 
                              active_grp_span_lbl = "Active Study Agent", colspan_var = "colspan_trt", trt_var = "TRT01A")

# Optional virtual combo arms:
combodf <- tibble::tribble(
  ~valname,     ~label,        ~levelcombo,                                 ~exargs,
  "All_Active", "All Active",  c("Xanomeline Low Dose", "Xanomeline High Dose"), list()
)

# 2. Declarative one-call layout:
basic_table(show_colcounts = TRUE, colcount_format = "N=xx") |>
  grouped_cols_w_diffs(trt_map, combo_map_df = combodf, diffs_label = "Risk Difference (%) (95% CI)") |>
  build_table(ae, alt_counts_df = adsl)
```

#### Fallback Solution (pure `rtables` without `junco`):
```r
# Requires manual dummy variables, split functions, and 4 chained splits:
adsl$colspan_trt <- factor(ifelse(adsl$TRT01A == "Placebo", " ", "Active Study Agent"), levels = c("Active Study Agent", " "))
adsl$rrisk_header <- "Risk Difference (%) (95% CI)"
adsl$rrisk_label <- paste(adsl$TRT01A, "vs Placebo")

basic_table(show_colcounts = TRUE, colcount_format = "N=xx") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = trt_map)) |>
  split_cols_by("TRT01A") |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by("TRT01A", labels_var = "rrisk_label", split_fun = remove_split_levels("Placebo")) |>
  build_table(ae, alt_counts_df = adsl)
```

---

### Archetype 2: Treatment Arms Subdivided by Subgroups (with or without Totals)

#### Primary Solution (using `junco`):
```r
library(rtables)
library(junco)

# Standard layout with within-arm Total column (total_facet = TRUE):
basic_table(show_colcounts = TRUE) |>
  grouped_cols_w_subgrps(trtvar = "ARM", subgrp_var = "SEX", total_facet = TRUE) |>
  build_table(dat)

# Quartiles / continuous bins without within-arm Total (total_facet = FALSE):
basic_table(show_colcounts = TRUE) |>
  grouped_cols_w_subgrps(trtvar = "ARM", subgrp_var = "AGEGR1", subgrp_lbl = "Age Quartiles", total_facet = FALSE) |>
  build_table(dat)
```

#### Fallback Solution (pure `rtables`):
```r
basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  split_cols_by("SEX", split_fun = add_overall_level("Total", first = TRUE)) |>
  build_table(dat)
```

---

### Archetype 3: Shift Table (Clinical Laboratory / Toxicity)

#### Primary Solution (using `junco`):
```r
library(rtables)
library(junco)

basic_table() |>
  shift_tbl_col_struct(var = "BNRIND", span_lbl = "Baseline ALB") |>
  build_table(adlb)
# Produces: N | Baseline ALB [HIGH | LOW | NORMAL] | Total
```

#### Fallback Solution (pure `rtables`):
```r
outer_splfun <- make_split_fun(post = list(
  add_overall_facet("N", label = " "),
  add_overall_facet("shift_table", label = "Baseline ALB"),
  restrict_facets(c("N", "shift_table"), op = "keep")
))
inner_splfun <- make_split_fun(post = list(
  add_overall_facet("Total", label = "Total"),
  function(ret, spl, fulldf, .spl_context) {
    if (.spl_context$value[[1]] == "N") {
      make_split_result("N", labels = c(N = "N"), list(N = fulldf), subset_exprs = list(N = quote(TRUE)))
    } else ret
  }
))

basic_table() |>
  split_cols_by("BNRIND", split_fun = outer_splfun) |>
  split_cols_by("BNRIND", split_fun = inner_splfun) |>
  build_table(adlb)
```

---

### Archetype 4: "Some vs All" / Non-Mutually Exclusive Comparisons

#### Primary Solution (using `junco`):
```r
library(rtables)
library(junco)

basic_table() |>
  some_v_all_col_struct(
    colspan_trt_map,
    subgrp_var  = "AETOXGR",
    subgrp_lvls = c("4", "5"),
    all_lbl     = "All Grades",
    subgrp_lbl  = "High Toxicity"
  ) |>
  build_table(adae)
```

#### Fallback Solution (pure `rtables`):
```r
some_v_all_splfun <- make_split_fun(post = list(
  add_overall_facet("All Grades", label = "All Grades"),
  add_combo_facet(name = "AETOXGR_subset", label = "High Toxicity", levels = c("4", "5")),
  restrict_facets(c("All Grades", "AETOXGR_subset"), op = "keep")
))

basic_table() |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by("ARM") |>
  split_cols_by("AETOXGR", split_fun = some_v_all_splfun) |>
  build_table(adae)
```

---

### Archetype 5: Longitudinal Visits with Conditional Change-from-Baseline
Inspect `tail(.spl_context$value, 1)` in a post-processing hook on `split_cols_by_multivar`:
```r
drop_cfb_at_baseline <- function(ret, spl, fulldf, .spl_context) {
  parent_visit <- tail(.spl_context$value, 1)
  if (identical(parent_visit, "BASELINE")) {
    ret <- restrict_facets("AVAL", op = "keep")(ret, spl, fulldf)
  }
  ret
}

basic_table() |>
  split_cols_by("AVISIT", show_colcounts = TRUE) |>
  split_cols_by_multivar(
    vars = c("AVAL", "CHG"),
    varlabels = c("Observed", "CFB"),
    split_fun = make_split_fun(post = list(drop_cfb_at_baseline))
  ) |>
  build_table(dat)
```

---

### Archetype 6: Per-Parent Data-Driven Quantile Bins
When numeric cutpoints must be computed separately inside each parent column facet:
```r
per_facet_qrt_core <- function(var) {
  function(spl, df, vals, labels, .spl_context) {
    vec <- as.numeric(as.character(df[[var]]))
    qs <- unname(quantile(vec, c(0.25, 0.5, 0.75), na.rm = TRUE))
    fqs <- format(qs, digits = 2)
    cats <- cut(vec, c(-Inf, qs, Inf), labels = c(paste0("<=", fqs[1]), paste0(fqs[1], "-", fqs[2]), paste0(fqs[2], "-", fqs[3]), paste0(">", fqs[3])))
    nms <- c("ltq1", "q1toq2", "q2toq3", "gtq3")
    make_split_result(
      values = nms,
      datasplit = setNames(split(df, cats), nms),
      labels = setNames(levels(cats), nms),
      subset_exprs = list(
        substitute(var <= val, list(var = as.name(var), val = qs[1])),
        substitute(var <= val2 & var > val1, list(var = as.name(var), val1 = qs[1], val2 = qs[2])),
        substitute(var <= val2 & var > val1, list(var = as.name(var), val1 = qs[2], val2 = qs[3])),
        substitute(var > val, list(var = as.name(var), val = qs[3]))
      )
    )
  }
}

basic_table(show_colcounts = TRUE) |>
  split_cols_by("ARM") |>
  split_cols_by("STUDYID", split_fun = make_split_fun(core_split = per_facet_qrt_core("AGE"))) |>
  build_table(ex_adsl)
```

---

## 7. Column Counts (`N=xx`) & Post-Build Header Manipulation

### 7.1 Controlling Column Count Denominators at Build Time
1. **Default (`df` counts)**: `build_table(lyt, df)` counts rows of `df` in each column facet.
2. **Subject-level population dataset (`alt_counts_df`)**: When `df` is an event-level or repeated-measures dataset (`ADAE`, `ADLB`, `ADVS`), pass `alt_counts_df = adsl` so header counts reflect subjects in `adsl`. Every column split variable must exist in `alt_counts_df`. Only use when counts calculated on the primary df would be incorrect; otherwise omit the `alt_counts_df` argument (rtables >= 0.6.17) or explicitly pass df to it again (rtables < 0.6.17).
3. **Manual vector override (`col_counts`)**: Pass `col_counts = c(100L, 105L, ...)` to `build_table(...)`.

### 7.2 Post-Build Mutation on `tbl`
```r
col_counts(tbl) <- c(134L, 134L, 75L)
facet_colcount(tbl, c("ARM", "C: Combination")) <- 75L
facet_colcounts_visible(tbl, c("ARM", "A: Drug X", "SEX")) <- TRUE
colcount_na_str(tbl) <- "N/A"
```

---

## 8. `junco` High-Level Column Structure Architecture

When `library(junco)` is available, use its high-level pipeline functions for CDISC and regulatory trial shells:

### 8.1 `grouped_cols_w_diffs()`
Constructs active vs. control treatment spanners and pairwise difference columns.
- Arguments: `colspan_trt_map`, `combo_map_df`, `comp_map`, `diff_cols = TRUE`, `diffs_label = "Risk Differences"`, `.main_pre`, `.main_post`, `.rr_pre`, `.rr_post`.
- Supports virtual combination arms (`combo_map_df`) and arbitrary comparison matrices (`comp_map`).
- Automatically injects `ref_path` into the `extra_args` of each comparison column. **Because of this auto-injection, `ref_path` must never be passed in `extra_args` on downstream `analyze()` or `summarize_row_groups()` calls. Doing so causes `"formal argument ref_path matched by multiple actual arguments"` errors in comparison columns.** For custom afuns that need `ref_path` in treatment columns (e.g., to call `get_ref_info()`), capture `ref_path` from a closure scope (see Tier 3 closure-capture pattern in `[custom_afuns.md](./custom_afuns.md)`).

### 8.2 `grouped_cols_w_subgrps()`
Splits treatment arms by a subgroup variable.
- Arguments: `colspan_trt_map`, `trtvar`, `subgrp_var`, `subgrp_lbl`, `total_facet = TRUE`, `total_lbl = "Total"`.
- Set `total_facet = FALSE` for binned continuous quartiles.

### 8.3 `shift_tbl_col_struct()`
Standard J&J shift table column layout.
- Arguments: `var`, `span_lbl = "Baseline"`, `.outer_spl_var = var`.
- Generates: `N` column, followed by `var` levels under `span_lbl`, followed by `Total`.

### 8.4 `some_v_all_col_struct()`
Non-mutually exclusive comparisons (e.g. All AEs vs Related AEs, All Grades vs High Toxicity).
- Arguments: `colspan_trt_map`, `trtvar`, `subgrp_var`, `subgrp_lvls`, `all_lbl`, `subgrp_lbl`.

### 8.5 Helper Functions: `create_colspan_var` & `create_colspan_map`
- `create_colspan_var(df, non_active_grp, non_active_grp_span_lbl, active_grp_span_lbl, colspan_var, trt_var)`: Appends the spanning column to `df`.
- `create_colspan_map(df, non_active_grp, non_active_grp_span_lbl, active_grp_span_lbl, colspan_var, trt_var, active_first = TRUE)`: Generates the 2-column map used in layout functions.

### 8.6 Multi-Comparator Split Function: `make_multicomp_splfun()`
Low-level split function engine powering difference columns when constructing custom layouts outside `grouped_cols_w_diffs`.

### 8.7 Summary Statistics in Columns: `stats_in_cols_setup()`
Post-processing split function factory for multi-variable layouts (`split_cols_by_multivar(c("AVAL", "BASE", "CHG"))`) that expands variables into summary statistic subcolumns (`N`, `Mean`, `SD`, `SE`, `Med`, `Min`, `Max`).

---

## 9. Common Pitfalls & Verification Checklist

1. **Missing or Unordered Spanning Variable (`" "` Sorting Before `"A"`)**:
   - `grouped_cols_w_diffs` requires the spanning variable (`colspan_trt`) to exist on **both** `df` and `alt_counts_df` as a **factor with identical ordered levels**.
   - **Never create a spanning variable with raw `ifelse(..., " ", "Active ...")` without wrapping it in `factor(..., levels = c(active_lbl, " "))`**: `create_colspan_map()` does not error on character input—instead, ASCII sorting puts `" "` (32) before `"A"` (65). Al`ways use `create_colspan_var()` on a single-observation-per-subject dataset (`alt_counts_df`, if it will differ from `df`) first, then join or synchronize onto `df` via `factor(..., levels = levels(...))` rather than running `droplevels()` independently on `df`.
2. **Unintended Deep Nesting Instead of Parallel Banners**:
   - Chaining `split_cols_by` calls creates nested crossed subtrees. Use `nested = FALSE` or precede splits with `add_overall_col("Header")` for parallel banners.
3. **Unrequested `Total` / Extra Column Splits & Wrong `colcount_format`**:
   - Do not add `add_overall_level("Total")` or an extra `split_cols_by("TRT01A")` unless present in the shell header.
   - `basic_table()` defaults to `colcount_format = "(N=xx)"`; explicitly set `colcount_format = "N=xx"` whenever the shell displays `N=XXX` without parentheses.
4. **Factor Level Hygiene**:
   - `split_cols_by` preserves all factor levels (producing `N=0` columns). Use `droplevels()` or `keep_split_levels()` when unobserved levels should be omitted.
   - **Post-filter and post-join repair**: Filtering `adsl` (e.g., removing `"Screen Failure"`) can leave stale factor levels or convert factors to character. After any filter, re-apply `factor()` to drop unused levels: `adsl <- adsl |> filter(...) |> mutate(TRT01A = factor(TRT01A))`. After joining `adsl` columns onto `df`, synchronize factor levels: `adae <- adae |> mutate(TRT01A = factor(TRT01A, levels = levels(adsl$TRT01A)))`. Mismatched levels between `df` and `alt_counts_df` cause `"All facets matched removal criteria"` or `create_colspan_map` errors.
4. **`ref_path` Double-Injection with `grouped_cols_w_diffs()`**:
   - `grouped_cols_w_diffs()` auto-injects `ref_path` into comparison column `extra_args`. If `ref_path` is also passed in `extra_args` on downstream `analyze()` or `summarize_row_groups()` calls, comparison columns receive `ref_path` twice, causing `"formal argument ref_path matched by multiple actual arguments"`. When using `grouped_cols_w_diffs()`, never pass `ref_path` in `extra_args`; for custom afuns that need it in treatment columns, use the closure-capture pattern from `[custom_afuns.md](./custom_afuns.md)` Tier 3.
5. **Sibling Header Visibility**:
   - When modifying column count visibility via `facet_colcounts_visible(tbl, path) <- TRUE`, pass the path pointing to the parent split level, not a single leaf.
