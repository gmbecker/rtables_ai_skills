---
name: rtables-custom-afuns-cfuns
description: Construct, extend, wrap, and combine custom analysis functions (`afun` in `analyze` / `analyze_colvars`) and group summary content functions (`cfun` in `summarize_row_groups`) in `rtables` and `junco`. Prioritizes reusing, parameterizing, wrapping (`make_afun`), stacking (`c()`), modifying (`RowsVerticalSection` accessors), and bridging (`get_ref_info`) existing `junco` and `tern` functions before writing bespoke statistical or formatting code from scratch. Activate this skill when existing `junco`/`tern` functions cannot directly satisfy a table shell's cell or row requirements via `extra_args` alone.
---

# Constructing Custom Analysis (`afun`) and Group Summary (`cfun`) Functions in `rtables`

> **Core Philosophy — Reuse & Extension Over Bespoke Code**:
> Writing a fully custom analysis or content function from scratch is a **last resort**. Production functions in `junco` and `tern` already handle edge cases (zero-row facets, `alt_counts_df` subject denominators, SAS rounding, dynamic decimal precision, reference group blanking, and referential footnotes). Always climb the **Customization Escalation Ladder** below and stop at the lowest tier that solves the problem.

---

## 1. The Customization Escalation Ladder (Decision Tree)

Before writing a custom function body, evaluate your requirement against these five tiers in order:

```
Need specific cell values, row labels, formats, or conditional behavior in analyze() / summarize_row_groups()?
 │
 ├─► Tier 0: Can an existing `junco` or `tern` function do this via `extra_args`?
 │     • Check `a_freq_j`, `a_summary_j`, `a_two_tier`, `a_summarize_aval_chg_diff_j`,
 │       `a_maxlev`, `a_eair100_j`, `c_summary_subset_label`, `column_stats`, etc.
 │     • Check `.stats`, `.formats`, `.labels`, `.indent_mods`, `denom`, `denom_by`,
 │       `ref_path`, `riskdiff`, `comp_btw_group`, `exclude_visits`, `drill_down_levs`.
 │     └──► YES: Do NOT write a custom function. Defer to `[select_afuns.md](./select_afuns.md)`.
 │
 ├─► Tier 1: Does an existing `s_*` or `a_*` function compute the right values,
 │           but you need fixed subsetting of `.stats`, custom `.formats`/`.labels`/`.indent_mods`,
 │           ungrouped list stats (`.ungroup_stats`), or curried default arguments?
 │     └──► YES: Wrap the existing function with `rtables::make_afun()` (or `list_wrap_x()` / `list_wrap_df()`).
 │               See Section 2.
 │
 ├─► Tier 2: Can the desired output be built by calling one or more EXISTING `afun`/`cfun`s
 │           and combining, selecting between, or tweaking their returned `RowsVerticalSection` objects?
 │     ├─► Different existing `afun`s in different columns (e.g., counts in Arm cols, custom stats in Diff cols, or counts in Arm cols, blank cells in Diff cols)?
 │     │     └──► Use Column-Conditional Dispatch Wrapper (Section 3.2).
 │     ├─► Multiple existing `afun`s stacked vertically in the same facet (e.g., Total Events + Unique Subjects)?
 │     │     └──► Use Vertical Stacking with `c(rvs1, rvs2)` (Section 3.3).
 │     ├─► Existing `afun` output needs post-hoc label, indent, format, or footnote tweaks?
 │     │     └──► Modify the returned `RowsVerticalSection` via accessors (`indent_mod<-`, `obj_format<-`) (Section 3.4).
 │     └─► Drill-down / secondary variable analysis under only specific levels or row facets
 │         (and `junco::a_two_tier` doesn't fit)?
 │           └──► Use Conditional Stacking with `lapply()` + `c()` + `indent_mod<-` (Section 3.5).
 │
 ├─► Tier 3: Does an existing `afun` expect `.ref_group` and `.in_ref_col`, but your table uses
 │           a nested/spanned column tree (`ref_path`) or an Overall column that breaks standard `ref_group`?
 │     └──► YES: Write a 3-line adapter using `junco::get_ref_info(ref_path, .spl_context, .var)`
 │               and delegate to the existing `afun` (Section 4).
 │
 └─► Tier 4: None of the above apply (genuinely novel statistical calculation or bespoke multi-ancestor logic).
       └──► Requires non-trivial novel statistical calculations beyond e.g., counts and percents?
       │    └──► YES: write a dummy function which warns the user and emits the correct number of rows. **DO NOT implement statistical calculations on your own**
       └──► Write a clean custom `afun`/`cfun` using `in_rows()`, `rcell()`, `.spl_context`,
            and reusable behavioral building blocks (Sections 5–7).
```

---

## 2. Tier 1: Wrapping and Customizing Existing Functions (`make_afun`)

When a statistical summary function `s_*` (returning a named `list`) or an analysis function `a_*` (returning a `RowsVerticalSection` via `in_rows()`, which inherits from `list`) already computes the required numbers, use `rtables::make_afun()` to create a customized `afun` without writing a new function body.

### 2.1 `make_afun()` Capabilities & Signature

```r
make_afun(
  fun,
  .stats = NULL,          # Character vector of element names to keep (and reorder) from fun's output
  .formats = NULL,        # Named vector/list of formats overriding fun's defaults
  .labels = NULL,         # Named character vector of row labels overriding fun's defaults
  .indent_mods = NULL,    # Named integer vector of row indent modifiers
  .ungroup_stats = NULL,  # Character vector of nested-list stat names to flatten into individual rows
  .format_na_strs = NULL, # Named vector/list of NA replacement strings
  ...,                    # New default values for arguments of `fun` (still overridable via `extra_args`)
  .null_ref_cells = ".in_ref_col" %in% names(formals(fun)) # Auto-blank reference column cells
)
```

### 2.2 Chaining `make_afun()` and Flattening Grouped Stats (`.ungroup_stats`)

- `make_afun()` can wrap **either** a raw list-returning helper `s_foo` **or** an already-wrapped `a_foo` to further restrict `.stats`, relabel rows, or change `.formats`.
- If `fun` returns a nested list element (e.g., `letters = list(a = 1, b = 2)` for factor levels or multi-row components), setting `.ungroup_stats = "letters"` unpacks its sub-elements into top-level rows and applies the parent's `.indent_mods = c(letters = 1L)` and `.formats` to each unpacked row.
- **Chaining Rule with `.ungroup_stats`**: Setting `.ungroup_stats` alters the structure of the returned object. Any subsequent `make_afun()` call wrapping that result must reference the **unpacked** element names (`"a"`, `"b"`), not the original group name (`"letters"`).

```r
# Wrap a list-returning function and curry default `b = 3`, while flattening `letters`
a_grp <- make_afun(
  s_grp,
  b = 3,
  .labels = c(nrow_df = "Row count", .N_col = "Column count"),
  .formats = c(nrow_df = "xx.", .N_col = "xx."),
  .indent_mods = c(letters = 1L),
  .ungroup_stats = "letters"
)

# Further specialize `a_grp` for a specific table section
a_subset <- make_afun(
  a_grp,
  .stats = c("nrow_df", "b"),
  .formats = c(b = "xx.x")
)
```

### 2.3 Adapting Simple R Functions (`list_wrap_x` and `list_wrap_df`)

If you have an existing standard R function (like `base::summary` or a custom utility) that returns a named vector rather than a `list`, convert it into a list-returning function first with `list_wrap_x(f)` (takes vector `x`) or `list_wrap_df(f)` (takes `data.frame` `df`):

```r
# Turn base::summary(x) into a valid list-returning function for analyze() or make_afun()
a_base_summary <- make_afun(
  list_wrap_x(summary),
  .formats = "xx.xx"
)
```

---

## 3. Tier 2: Combining and Extending Existing `afun`s via `RowsVerticalSection`

Every modern `afun` and `cfun` in `rtables`, `tern`, and `junco` returns the result of `in_rows()`, which is a **`RowsVerticalSection`** object. Understanding what a `RowsVerticalSection` contains—and how to manipulate and combine it—lets you build complex composite analysis functions by orchestrating existing `afun`s.

> [!IMPORTANT]
> **Mandatory a/cfun Argument Invariant**:
> - When constructing a custom afun that wraps one or more existing afuns, the custom function must accept - and appropriately pass-through - the union of all rtables special arguments (see Section 5.2) accepted by the wrapped existing afuns. Use the afuns' documentation or call `formals()` to determine what arguments the existing afuns expect.

### 3.1 Anatomy & Accessors of a `RowsVerticalSection` Object

A `RowsVerticalSection` wraps a list of `CellValue` (`rcell`) objects alongside **row-level** metadata. Note that during table construction, **only the first (left-most) column's `RowsVerticalSection`** dictates row-level names, labels, indent modifiers, row formats, row NA strings, and row footnotes.

| Property | Level | Accessor / Setter | Description |
| :--- | :--- | :--- | :--- |
| **Row Names** | Row | `value_names(rvs)` | Internal programmatic names of the generated rows (used in paths and format matching). |
| **Row Labels** | Row | `value_labels(rvs)` | Visible stub labels of the generated rows. |
| **Indent Modifiers** | Row | `indent_mod(rvs)` / `indent_mod(rvs) <- val` | Integer indent adjustments (`0L`, `1L`, `-1L`) relative to the natural split depth. |
| **Row Formats** | Row | `obj_format(rvs)` / `obj_format(rvs) <- val` | Row-level format strings or functions. |
| **Row NA Strings** | Row | `obj_na_str(rvs)` / `obj_na_str(rvs) <- val` | Row-level `NA` replacement strings. |
| **Row Footnotes** | Row | `row_footnotes(rvs)` / `row_footnotes(rvs) <- val` | Row-level `RefFootnote` objects attached to stub labels. |
| **Cell Objects** | Cell | `row_cells(rvs)` / `cell_values(rvs)` / `rawvalues(rvs)` | Extract the underlying `rcell` (`CellValue`) objects or their raw values. Cell-level properties (`.aligns`, `.cell_footnotes`, `.stat_names`) live on individual cells. |

### 3.2 Pattern 2A: Column-Conditional Dispatch Between Existing `afun`s

When different column branches require completely different existing `afun`s (and the combination is not already built into a single `junco` function like `a_freq_j(..., riskdiff = TRUE)`), write a lightweight dispatcher that inspects `.spl_context` and calls the appropriate existing `afun`:

```r
# Helper: robustly check column position via cur_col_split_val (preferred over cur_col_id)
# NOTE: The pattern string (e.g., "Risk Differences") must exactly match the label used
# in the layout's column split; watch for subtle variations like "Risk Difference" vs.
# "Risk Differences" or custom labels passed to split_cols_by()/add_combo_levels().
in_diff_col <- function(spl_context) {
  any(grepl("Risk Differences", spl_context$cur_col_split_val[[1]], fixed = TRUE))
}

# Suppose main_behavior_afun needs .N_col and .spl_context,
# while alt_behavior_afun needs .ref_group, .in_ref_col, and .spl_context.
# The wrapper must accept the UNION of both sets (Mandatory a/cfun Argument Invariant)
# so that rtables populates all of them; each inner call receives only what it needs.
col_dispatch_afun <- function(df, .var, .N_col, .ref_group, .in_ref_col, .spl_context, ...) {
  if (in_diff_col(.spl_context)) {
    alt_behavior_afun(df = df, .var = .var,
                      .ref_group = .ref_group, .in_ref_col = .in_ref_col,
                      .spl_context = .spl_context, ...)
  } else {
    main_behavior_afun(df = df, .var = .var,
                       .N_col = .N_col, .spl_context = .spl_context, ...)
  }
}
```

#### Special-case: Blank Cells in Select Columns
Emit cells with `NULL` or `""` values to create blank cells, e.g.:
```r
col_dispatch_afun2 <- function(df, .var, ..., .spl_context) {
  if (in_diff_col(.spl_context)) {
    ## Replace with "k" and "labs" calculations that will match those generated by main_behavior_afun in your use-case
    ## assume var is a factor and we are making one row per level.
    k <- length(levels(df[[.var]]))
    labs <- levels(df[[.var]])
    val_lst <- setNames(replicate(k, list(NULL)), labs)
    in_rows(.list = val_lst)
  } else {
    main_behavior_afun(df = df, .var = .var, ..., .spl_context = .spl_context)
  }
}
```

> [!IMPORTANT]
> **Mandatory Row-Count Invariant Across Columns**:
> For any given row facet, `col_dispatch_afun` is called once per column and **MUST return the exact same number of rows in every column**. If `main_behavior_afun` returns $k$ rows (e.g., `n` plus 4 factor levels = 5 rows), `alt_behavior_afun` must also return $k$ rows in the same order (using blank `rcell(NULL)` cells for rows where no comparison statistic applies, such as `n`).
> **Mandatory Orthogonal Afun Selection For Column-Conditional Behavior**:
> - When constructing the per-column behavior functions, independently re-apply the full afun selection-and-construction pipeline (select_afuns.md then this skill) from the beginning for each distinct behavior — preferring existing junco/tern functions before writing bespoke ones.

### 3.3 Pattern 2B: Vertical Stacking of Existing `afun`s (`c(rvs1, rvs2)`)

In `rtables` ($\ge 0.6.16$), `RowsVerticalSection` objects have a native `c()` method (`c.RowsVerticalSection`) that concatenates multiple `in_rows()` outputs while preserving all row names, labels, indent modifiers, formats, footnotes, and `NA` strings.

Use vertical stacking when a single facet needs rows from two or more distinct `afun`s (for example, stacking a total event count row above a unique-subject frequency row, or stacking two `junco`/`tern` summaries on different variables within the same facet):

```r
stacked_afun <- function(df, .var, .N_col, .spl_context, ...) {
  # 1. Call first existing afun (e.g., total event count)
  rvs_events <- afun_total_events(df = df, .var = .var, ...)
  
  # 2. Call second existing afun (e.g., unique patients with event)
  # If the inner afun expects vector `x` instead of `df`, pass `df[[.var]]`:
  rvs_pats <- afun_unique_pats(df = df, .var = .var, .N_col = .N_col, .spl_context = .spl_context, ...)
  
  # 3. Combine and return as a single RowsVerticalSection
  c(rvs_events, rvs_pats)
}
```

#### Special Case: Adding A Combination Level

Suppose we need an afun which tabulates a categorical variable with levels `None`, `Low` and `High` 
and we need a new combination level `Any Present` which combines `Low` and `High`; we can do this 
by calling the desired function for the standard levels and adding the row for our new combination
level by combining the result with one we construct directly.

```r
add_any_row <- function(df, .var, ...) {
  main_ret <- regular_afun(df, .var, ...) ## a different approach is needed if the afun does not accept the df, .var pairing
  tmpdf <- df[df[[.var]] != "None", ]
  tmpdf[[.var]] <- factor("Any Present") ## spoof our new level, this is ok because we are in an afun and our data isn't going anywhere
  addl_ret <- regular_afun(tmpdf, .var, ...) ## get the same behavior we got for main_ret but on our new combination level
  c(main_ret, addl_ret)
}
```

#### Rules When Stacking Existing `afun`s:
1. **First Argument Promotion (`df` vs. `x`)**: If *any* of the inner `afun`s takes `df` as its first argument, `stacked_afun` **must** take `df` as its first argument. You can easily extract `x <- df[[.var]]` inside `stacked_afun` to pass to any inner function that expects a vector `x`.
2. **Signature Union Rule**: `rtables` only passes special arguments (`.var`, `.N_col`, `.N_row`, `.df_row`, `.alt_df`, `.alt_df_full`, `.spl_context`, `.ref_group`, `.in_ref_col`) that appear explicitly in the **outer wrapper's `formals()`**. Therefore, `stacked_afun` must explicitly list the union of all special arguments needed by *any* of the inner functions it calls.
3. **Unique Row Names**: Ensure the stacked `RowsVerticalSection` objects do not have colliding `value_names()` within the same elementary table; if they do, disambiguate labels/names or use `make_afun()` on one of the inner functions before stacking.

### 3.4 Pattern 2C: Post-Call `RowsVerticalSection` Modification

Often an existing `afun` produces the exact cells you need, except you want to shift indentation dynamically, append a footnote conditionally, or adjust formats based on `.spl_context`. Instead of re-implementing the `afun`, call it and mutate the returned `RowsVerticalSection`:

```r
indented_existing_afun <- function(df, .var, .spl_context, ...) {
  rvs <- existing_afun(df = df, .var = .var, .spl_context = .spl_context, ...)
  
  # Dynamically indent rows by +1 if inside a specific parent row split
  if (tail(.spl_context$value, 1) == "SPECIAL_GROUP") {
    indent_mod(rvs) <- indent_mod(rvs) + 1L
  }
  rvs
}
```

### 3.5 Pattern 2D: Conditional / Two-Tier Stacking Across Levels or Row Facets

Sometimes a table displays a primary summary for every level of `.var` (e.g., end-of-study status `EOSSTT`: `"COMPLETED"`, `"DISCONTINUED"`, `"ONGOING"`), and **only under specific level(s)** (`"DISCONTINUED"`) indents a secondary breakdown of another variable (`DCSREAS`).

> [!TIP]
> **Check `junco::a_two_tier` First!**
> For production tables requiring a two-tier drill-down (like `EOSSTT` $\rightarrow$ `DCSREAS` or `AEBODSYS` $\rightarrow$ `AEDECOD` inside a single `analyze()` call), always use `junco::a_two_tier` first (see `[select_afuns.md](./select_afuns.md)`). Write a custom conditional stacking `afun` only when `a_two_tier` cannot support your specific inner analysis function.

When you do need a custom two-tier or row-conditional stacking wrapper, iterate over levels (or check `.spl_context`), call existing `afun`s for both tiers, adjust `indent_mod(det_rvs) <- 1L`, and combine with `c()`:

```r
custom_two_tier_afun <- function(df, .var, .spl_context, detail_var, detail_level, ...) {
  values <- lapply(
    levels(df[[.var]]),
    function(lvl) {
      dat <- df[!is.na(df[[.var]]) & df[[.var]] == lvl, , drop = FALSE]
      # Primary tier row for `lvl`
      rvs_out <- primary_level_afun(dat, .var = .var, lbl = lvl, ...)
      
      # Conditional detail tier (indented by +1) only for `detail_level`
      if (lvl %in% detail_level) {
        det_rvs <- detail_var_afun(dat[[detail_var]], ...)
        indent_mod(det_rvs) <- 1L
        rvs_out <- c(rvs_out, det_rvs)
      }
      rvs_out
    }
  )
  do.call(c, values)
}
```

---

## 4. Tier 3: Bridging Existing Comparative `afun`s to Nested Column Trees (`get_ref_info`)

Many comparative analysis functions (including `tern` functions and functions built with `make_afun` using `.ref_group` and `.in_ref_col`) assume `rtables` automatically populates `.ref_group` and `.in_ref_col`. However, `rtables`' built-in `ref_group` argument on `split_cols_by()` only works within a single flat column split—it cannot resolve a reference arm across **nested spanning headers** (e.g., `"Active Study Agent"` vs `" "` $\rightarrow$ `"B: Placebo"`) or when an **Overall/Total column** is added via `add_overall_col()`.

Instead of rewriting an existing `.ref_group`-based `afun`, wrap it in a 3-line adapter using **`junco::get_ref_info()`**:

```r
# Existing afun that expects standard .ref_group and .in_ref_col
standard_diff_afun <- function(x, .ref_group, .in_ref_col) {
  in_rows(
    "Difference of Averages" = non_ref_rcell(
      mean(x, na.rm = TRUE) - mean(.ref_group, na.rm = TRUE),
      is_ref = .in_ref_col,
      format = "xx.xx"
    )
  )
}

# 3-line adapter bridging `ref_path` + `.spl_context` to `standard_diff_afun`
bridged_diff_afun <- function(x, ref_path, .spl_context, .var) {
  ref <- junco::get_ref_info(ref_path = ref_path, .spl_context = .spl_context, var = .var)
  standard_diff_afun(x, .ref_group = ref$ref_group, .in_ref_col = ref$in_ref_col)
}
```

### How `junco::get_ref_info(ref_path, .spl_context, .var = NULL)` Works:
- **When `.var = NULL` (for `df`-taking `afun`s)**: Returns `list(ref_group = <data.frame>, in_ref_col = <logical>)`, where `ref_group` is the current row facet's `full_parent_df` subsetted to the column identified by `ref_path` (e.g., `c("colspan_trt", " ", "ARM", "B: Placebo")`).
- **When `.var = .var` (for `x`-taking `afun`s)**: Returns `list(ref_group = <vector>, in_ref_col = <logical>)`, extracting column `.var` from the reference group's data frame.
- **In non-matching sibling column branches** (e.g., inside a separate `"Risk Difference"` column tree or an `"Overall"` column): `in_ref_col` is `NULL` (not `FALSE`). Use `isTRUE(ref$in_ref_col)` to test for the reference column and `is.null(ref$in_ref_col)` to detect difference columns.

### How `ref_path` Reaches the Bridged `afun`: Auto-Injection vs. Closure Capture
- **When using `grouped_cols_w_diffs()` or any column structure function that auto-injects `ref_path`**: `ref_path` must NOT be passed in `extra_args` (doing so causes double-injection errors in comparison columns). Instead, capture `ref_path` from a closure scope:
  ```r
  make_bridged_afun <- function(ref_path) {
    function(df, .var, .spl_context, ...) {
      ref <- junco::get_ref_info(ref_path, .spl_context, .var = NULL)
      standard_diff_afun(df, .ref_group = ref$ref_group, .in_ref_col = isTRUE(ref$in_ref_col))
    }
  }
  ```
- **When building column layouts manually with `split_cols_by()`**: Pass `ref_path` explicitly in `extra_args = list(ref_path = ref_path)` on each `analyze()`/`summarize_row_groups()` call, and accept `ref_path` as a formal argument of the bridged afun.

---

## 5. Tier 4: Core Mechanics for Writing Bespoke `afun`s and `cfun`s

When Tiers 0–3 cannot satisfy a requirement, write a custom function following `rtables`' formal contracts. A fundamental contract is that **an `afun`/`cfun` must never read precomputed cell values or analysis results from the enclosing or global environment**. All cell values must be computed within the function body from the data and special arguments `rtables` passes to it, or retrieved from an environment cache populated by a prior `afun` call during the same `build_table()` execution (see Building Block 3, Section 7). Note that function factories which capture *configuration* (thresholds, method names, format strings, variable names) from their enclosing scope are fine — the prohibition is specifically against using closures or globals to inject precomputed data or results that should be derived from the faceted data subsets `rtables` provides.

> [!IMPORTANT]
> **Signature Union Rule (Universal Invariant)**
> This rule applies to ALL custom afun/cfun patterns (Tiers 1–4), not just vertical stacking. When constructing a custom afun that wraps one or more existing afuns, the custom function must list in its `formals()` the **exact union** of all `rtables` special arguments (see Section 5.2) accepted by the wrapped existing afuns — **no more, no less**:
> - **Missing arguments**: `rtables` only populates special arguments that appear in `names(formals(fun))`. If the wrapper omits a special argument that an inner function needs (e.g., `.df_row`), `rtables` never supplies it, and the inner function fails with `"argument .df_row is missing, with no default"`.
> - **Extra arguments**: Including a special argument that no inner function accepts (e.g., `.alt_df` when the inner function expects `.alt_df_full`) causes `"unused argument"` errors when forwarded via `...`.
> - **Verification**: Check via `names(formals(inner_fun))` against the special arguments reference in Section 5.2. Pay special attention to the confusingly similar argument name pairs listed in Section 5.2.

### 5.1 Unified `afun` / `cfun` Function Template

The **only** structural difference between an analysis function (`afun` in `analyze()`) and a group summary content function (`cfun` in `summarize_row_groups()`) is that `cfun` must accept **`labelstr`** as its second argument (populated automatically with the current row-split facet's display label).

By giving `labelstr = NULL` a default value in the second position, a single function can serve as **both** an `afun` and a `cfun`:

```r
template_acfun <- function(df,               # Or `x` if only the single analyzed vector is needed
                           labelstr = NULL,  # Non-NULL when called as cfun; NULL when called as afun
                           .var,             # Optional special args populated by rtables
                           .N_col,
                           .df_row,          # Full row-facet data across all columns; commonly needed by junco functions
                           .spl_context,
                           label = NULL,     # Optional user extra_args
                           ...) {
  # 1. Resolve row label(s) for cfun vs afun usage
  lbl <- if (!is.null(label)) {
    label
  } else if (!is.null(labelstr)) {
    labelstr # Used as cfun in summarize_row_groups()
  } else {
    "Default Analysis Row Label" # Used as afun in analyze()
  }

  # 2. Compute cell value(s) — MUST handle 0-row df / all-NA safely!
  val <- if (nrow(df) == 0) NA_real_ else mean(df[[.var]], na.rm = TRUE)

  # 3. Return a RowsVerticalSection via in_rows()
  in_rows(
    .list = list(my_stat = rcell(val, format = "xx.x")),
    .labels = c(my_stat = lbl)
  )
}
```

### 5.2 Complete Reference of Special Arguments Populated by `rtables`

`rtables` inspects `names(formals(fun))` and automatically supplies any of the following special arguments when present. **Users cannot override these via `extra_args`**.

| Category | Special Argument | Type | What It Contains & Critical Caveats |
| :--- | :--- | :--- | :--- |
| **Primary Input** | `x` *(1st arg)* | `vector` | The facet subset vector `df[[.var]]` for the current row $\times$ column cell. Mutually exclusive with `df` as 1st arg. |
| **Primary Input** | `df` *(1st arg)* | `data.frame` | The facet subset `data.frame` (all columns) for the current row $\times$ column cell. Required if your function inspects multiple variables (e.g., `USUBJID` + `.var`). |
| **Group Label** | `labelstr` *(2nd arg)* | `character(1)` | Passed **only** to `cfun` in `summarize_row_groups()`; contains the current row-group split value label. Always set default `labelstr = NULL` if dual-use. |
| **Variable Name** | `.var` | `character(1)` | Name of the variable currently being analyzed in `analyze(vars = ...)`. |
| **Marginal Counts** | `.N_col` | `integer(1)` | Column-wise $N$ for the current column. **Uses `alt_counts_df` (e.g., `ADSL`) when provided to `build_table()`!** |
| **Marginal Counts** | `.all_col_counts` | `integer` vector | Named vector of column counts across *all* columns in the table (uses `alt_counts_df` when provided). |
| **Marginal Counts** | `.N_row` | `integer(1)` | Row-wise observation count across all columns for the current row facet. **WARNING**: `.N_row` is **ALWAYS calculated from primary `df`**, NEVER from `alt_counts_df`! In an `ADAE` table with `alt_counts_df = adsl`, `.N_col` counts *patients* while `.N_row` counts *adverse event records*. |
| **Marginal Counts** | `.N_total` | `integer(1)` | **DO NOT USE.** Naively sums `.all_col_counts`, which double-counts whenever a table has an Overall column, overlapping subgroups, or Risk Difference columns. Instead, use `NROW(.spl_context$full_parent_df[[1]])` (for primary `df`) or `NROW(.alt_df_full)` (for `alt_counts_df`). |
| **Row Facet Data** | `.df_row` | `data.frame` | All observations in the current **row** facet across *all* columns (i.e., before column subsetting). Identical to `.spl_context$full_parent_df[[NROW(.spl_context)]]`. |
| **Alt Counts Data** | `.alt_df` | `data.frame` | `alt_counts_df` subsetted by **both** current row splits and current column expression. *(Warning: Can be 0-row or fail if row-split variables like `AEBODSYS` do not exist in `alt_counts_df`!)* |
| **Alt Counts Data** | `.alt_df_row` | `data.frame` | `alt_counts_df` subsetted by current **row** splits only (across all columns). Requires row-split variables to exist in `alt_counts_df`. |
| **Alt Counts Data** | `.alt_df_full` | `data.frame` | The **entire** unsplit `alt_counts_df` passed to `build_table()`. **Safe when row-split variables are absent from `alt_counts_df`**; can be filtered by column using `eval(.spl_context$cur_col_expr[[1]], .alt_df_full)`. *(Falls back to full `df` if `alt_counts_df` was not supplied).* |
| **Column Exprs** | `.all_col_exprs` | `list` of `expression` | Named list of R subsetting expressions for every leaf column in the table. |
| **Reference Info** | `.ref_group` | `data.frame` or `vector` | Reference column data for the current row facet (only populated when `ref_group` is set in `split_cols_by`; otherwise use `junco::get_ref_info()`). |
| **Reference Info** | `.ref_full` | `data.frame` or `vector` | Full reference column data ignoring row splits. |
| **Reference Info** | `.in_ref_col` | `logical(1)` | `TRUE` when currently executing inside the designated reference column. Pair with `non_ref_rcell(val, is_ref = .in_ref_col, format = ...)`. |
| **Split Context** | `.spl_context` | `data.frame` | Complete ancestry of row splits and current column coordinates. See Section 6. |

> [!WARNING]
> **Confusingly Similar Special Argument Names**
> Several special arguments have similar names but different scoping. Including the wrong one causes "unused argument" errors (if forwarded via `...`) or silently receives wrong data. Common confusions:
> - **`.alt_df`** vs **`.alt_df_full`** vs **`.alt_df_row`**: `.alt_df` = `alt_counts_df` subsetted by both row and column; `.alt_df_full` = entire unsplit `alt_counts_df`; `.alt_df_row` = `alt_counts_df` subsetted by row splits only. Most `junco` frequency functions (e.g., `a_freq_j`) use **`.alt_df_full`**, not `.alt_df`.
> - **`.df_row`** vs **`df`**: `.df_row` = full row-facet data across all columns; `df` = column-sliced data for the current cell. `.df_row` is commonly needed by `junco` functions for denominator calculations.
> - **`.ref_group`** vs **`.ref_full`**: `.ref_group` = reference column data for current row facet; `.ref_full` = full reference column data ignoring row splits.
> - **`.N_col`** vs **`.N_row`** vs **`.N_total`**: `.N_col` uses `alt_counts_df` when provided; `.N_row` always uses primary `df`; `.N_total` is unreliable (double-counts with Overall/diff columns).

---

## 6. Mastering `.spl_context` for Structure-Conditional Behavior

The `.spl_context` `data.frame` gives an `afun` or `cfun` complete visibility into **where** in the table hierarchy it is currently executing and **what data** belongs to every ancestor row split and sibling column.

### 6.1 Schema of `.spl_context`

`.spl_context` has **one row per row-faceting step** from the root of the table (`row 1`, where `split == "root"`) down to the immediate parent row split of the current `analyze()` or `summarize_row_groups()` call (`row NROW(.spl_context)`).

1. **Columns that VARY across rows of `.spl_context` (Row-Split Ancestry)**:
   - `split` (`character`): Name of the row split at step $i$ (e.g., `"root"`, `"AEBODSYS"`, `"AEDECOD"`).
   - `value` (`character`): String value of the facet at step $i$ (e.g., `"root"`, `"Cardiac disorders"`, `"Atrial fibrillation"`).
   - `full_parent_df` (`list` of `data.frame`s): The primary dataset `df` subsetted down to row-split step $i$ (across **all** columns).
   - `all_cols_n` (`integer`): Observation count in `full_parent_df[[i]]`.
   - `<col_id>` (one `list` column per leaf column in the table, named by `names(col_exprs(tab))`): Contains a `logical` vector of length `NROW(full_parent_df[[i]])` indicating which rows of `full_parent_df[[i]]` belong to that column.

2. **Columns that are CONSTANT across all rows of `.spl_context` (Current Column Coordinates — always index with `[[1]]`)**:
   - `cur_col_id` (`character`): Internal column identifier string (formed by pasting column split values with `"."`).
   - `cur_col_split` (`list` of `character`): Vector of column split variable/split names along the path to the current column (e.g., `c("colspan_trt", "ARM")`).
   - `cur_col_split_val` (`list` of `character`): Vector of column split values along the path to the current column (e.g., `c("Active Study Agent", "A: Drug X")`).
   - `cur_col_subset` (`list` of `logical`): Logical subsetting vectors for the current column at each row-split step.
   - `cur_col_expr` (`list` of `expression`): Symbolic R expression defining the current column subset (can be evaluated against `.alt_df_full` via `subset(.alt_df_full, eval(.spl_context$cur_col_expr[[1]]))`).
   - `cur_col_n` (`integer`): True observation count in `df` for the current column at each row-split step.

### 6.2 Essential `.spl_context` Lookup Idioms

| What You Need Inside `afun` / `cfun` | Exact R Expression |
| :--- | :--- |
| **Full unsplit primary `df`** (passed to `build_table`) | `.spl_context$full_parent_df[[1]]` |
| **Current row-facet `df` across all columns** | `.spl_context$full_parent_df[[NROW(.spl_context)]]` *(or `.df_row`)* |
| **Parent row-facet `df` one level up** | `.spl_context$full_parent_df[[NROW(.spl_context) - 1]]` |
| **Current (innermost) row split name** | `tail(.spl_context$split, 1)` *(or `.spl_context$split[NROW(.spl_context)]`)* |
| **Current (innermost) row split value** | `tail(.spl_context$value, 1)` *(or `.spl_context$value[NROW(.spl_context)]`)* |
| **Parent row split value (one level up)** | `.spl_context$value[NROW(.spl_context) - 1]` |
| **Value of a specific ancestor row split `"AVISIT"`** | `.spl_context$value[match("AVISIT", .spl_context$split)]` |
| **Current column path (interleaved `c(split1, val1, ...)` )** | ` as.vector(rbind(.spl_context$cur_col_split[[1]], .spl_context$cur_col_split_val[[1]]))` |
| **Leaf column split value (e.g., current arm/stat name)** | `tail(.spl_context$cur_col_split_val[[1]], 1)` |
| **Check if current column is under a specific header** | `"Risk Differences" %in% .spl_context$cur_col_split_val[[1]]` |
| **Extract another column's data in current row facet** | `facet_df <- .spl_context$full_parent_df[[NROW(.spl_context)]]`; `ref_id <- paste(ref_path[seq(2, length(ref_path), by = 2)], collapse = ".")`; `facet_df[.spl_context[[ref_id]][[NROW(.spl_context)]], ]` *(or use `junco::get_ref_info(ref_path, .spl_context)`)* |

---

## 7. Reusable Behavioral Building Blocks for Custom `afun`s

When writing a custom `afun`/`cfun` (or a hybrid wrapper), combine these proven building blocks rather than inventing ad-hoc logic.

### Building Block 1: Row- or Column-Conditioned Blank Cells (Simulating Baseline / Reference Exclusions)

When a statistic (such as Change from Baseline `CHG` at baseline visit `V1`, or a p-value in a reference column) should be **blank** in specific cells while still outputting the row itself:
- Pass `NULL` (or use `non_ref_rcell(val, is_ref = !show_it)`) as the cell value when `show_it == FALSE`, and set its format to `"xx"` (or let `non_ref_rcell` handle it).
- **Never omit the row from `in_rows()` in only some columns!** Every column in a row facet must emit the exact same row names. Only if `show_it` depends **purely on the row facet** (e.g., `AVISIT == "V1"` when `CHG` is analyzed in rows, not columns) can you omit a row across all columns simultaneously.

```r
# Suppressing CHG cells within Visit "V1" when AVAL and CHG are side-by-side in columns:
baseline_aware_afun <- function(x, .var, ref_rowgroup = "V1", .spl_context) {
  n <- sum(!is.na(x))
  meanval <- mean(x, na.rm = TRUE)
  sdval <- sd(x, na.rm = TRUE)

  cur_visit <- tail(.spl_context$value, 1)
  show_it <- (cur_visit != ref_rowgroup) || (.var != "CHG")

  in_rows(
    n = if (show_it) n else NULL,
    "Mean (SD)" = if (show_it) c(meanval, sdval) else NULL,
    .formats = c(
      n = "xx",
      "Mean (SD)" = if (show_it) "xx.x (xx.x)" else "xx"
    )
  )
}
```

### Building Block 2: Parent-Data-Driven Consistent Row Sorting Inside an `afun`

If an `afun` generates multiple rows (e.g., one row per level of `.var`) and sorts them dynamically by frequency (with a category like `"OTHER"` forced last):
- **CRITICAL**: You **cannot** sort based on the single-column `df` passed to `afun`, because each column would sort the levels differently and corrupt the table row alignment!
- Instead, compute the sort order `o` using the **full row-facet parent dataset across all columns**: `pardf <- .spl_context$full_parent_df[[NROW(.spl_context)]]`.

```r
# Sort generated level rows `cls` consistently across all columns using full_parent_df:
if (sort_by_prevalence) {
  pardf <- .spl_context$full_parent_df[[NROW(.spl_context)]]
  spldf_parent <- split(pardf, pardf[[.var]])
  o <- order(vapply(spldf_parent, NROW, integer(1)), decreasing = TRUE)

  # Force `last_cat` (e.g., "OTHER") to the bottom regardless of count
  if (!is.na(last_cat) && nzchar(last_cat)) {
    lasti <- which(names(spldf_parent)[o] == last_cat)
    if (length(lasti) > 0 && lasti != tail(o, 1)) {
      o <- c(o[-lasti], o[lasti])
    }
  }
  cls <- cls[o]
}
```

### Building Block 3: Environment Caching for Model-Fitting `afun`s Across Statistic Columns

When a table displays different statistics from the same statistical model across columns (e.g., `n`, `Hazard Ratio`, `95% CI`, `p-value (effect)`, `p-value (interaction)` via `split_cols_by_multivar`), `rtables` calls your `afun`/`cfun` once per column. To avoid fitting the model 5 times per row facet:
1. Create an `environment` before building the layout (`my_env <- new.env()`) and pass it via `extra_args`.
2. Inside the `afun`, key the cache by the current row facet value (`cov <- tail(.spl_context$value, 1)`), fit the model on `.df_row` (or `df` when columns are dummy-split on `"STUDYID"`), and store/retrieve `cache_env[[cov]]`:

```r
get_or_fit_model <- function(df, cov_key, cache_env) {
  if (is.null(cache_env[[cov_key]])) {
    cache_env[[cov_key]] <- fit_my_model(df, cov_key)
  }
  cache_env[[cov_key]]
}
```

### Building Block 4: Per-Column Argument Injection via `split_cols_by_multivar(..., extra_args = ...)`

When using `split_cols_by_multivar()`, passing a named `list` to `extra_args` where each element is a vector/list of length equal to `length(vars)` automatically dispatches the $j$-th element to the $j$-th column in all downstream `cfun` and `afun` calls:

```r
my_stats <- list("n", "hr", c("lcl", "ucl"), "pval")
my_formats <- c(n = "xx", hr = "xx.xx", lcl = "(xx.xx, xx.xx)", pval = "x.xxxx | (<0.0001)")
shared_env <- new.env()

lyt <- basic_table() |>
  split_cols_by_multivar(
    vars = rep("STUDYID", length(my_stats)),
    varlabels = c("n", "Hazard Ratio", "95% CI", "p-value"),
    extra_args = list(
      stat = my_stats,
      format = my_formats,
      cache_env = replicate(length(my_stats), list(shared_env))
    )
  ) |>
  summarize_row_groups(cfun = my_model_acfun, extra_args = list(is_summary = TRUE)) |>
  analyze_colvars(afun = my_model_acfun)
```

### Building Block 5: Format Precedence & Allowing Layout-Level Format Overrides

Understanding `rtables` format precedence is essential when deciding whether to hardcode formats inside your `afun` or allow callers to override them in `analyze(..., format = ..., formats_var = ...)`:

1. **Highest Precedence**: `.formats` in `in_rows()` (overwrites cell formats set in `rcell()`).
2. **Second Precedence**: `format` in `rcell()`.
3. **Third Precedence**: `format` or `formats_var` in `analyze()` / `summarize_row_groups()`.
4. **Lowest Precedence**: Parent split format $\rightarrow$ default `"xx"`.

> [!IMPORTANT]
> **How to Support `analyze(..., format = ...)` and `formats_var` in Custom `afun`s**:
> Format instructions passed to `analyze(format = ...)` or `analyze(formats_var = ...)` **only apply to cells/rows whose format returned by `afun` is `NULL` or `"default"`**!
> - If you want your custom `afun` to have sensible default formats *while still allowing users to override them*, accept `.formats = c(stat1 = "xx.x", ...)` as a formal argument of your `afun` (so users can override via `extra_args = list(.formats = ...)` or `make_afun()`), or leave `.formats = NULL` when delegating to `analyze(formats_var = ...)`.

### Building Block 6: Label-Only Header Row (Blank Cells with Visible Stub Label)

To emit a row with a visible stub label but blank cells across all columns from within an `afun`, use `rcell(NULL, format = "xx")`. This can be stacked with other rows via `c()` to create section headers within an afun's output:

```r
section_afun <- function(df, .var, .N_col, ...) {
  header <- in_rows("Section Header" = rcell(NULL, format = "xx"))
  row1 <- in_rows("Detail Row 1" = rcell(nrow(df), format = "xx"))
  indent_mod(row1) <- 1L
  c(header, row1)
}
```

This is distinct from `show_labels = "visible"` on `analyze()` (which adds a variable-label header row above the analysis rows) and from `summarize_row_groups()` (which adds a content row at a facet boundary). The afun-emitted label-only row is needed when a section header must appear *within* a block of rows generated by a single `analyze()` call.

### Building Block 7: Pre-Computed Derived Variables (Data Scope Invariant)

`rtables` populates **all data.frame special arguments** — `.df_row`, `.alt_df`, `.alt_df_row`, `.alt_df_full`, `.ref_group`, `.ref_full` — and the `full_parent_df` entries in `.spl_context` from the datasets passed to `build_table()` (`df` and `alt_counts_df`). **None of these reflect transformations applied inside an afun to the `df` column-slice.** The `df` argument to an afun is a column-slice of the row-faceted data; every other data.frame special argument is derived independently from the original datasets at the appropriate scope (row-facet, column-facet, or unsplit). Transformations applied to `df` inside the afun are invisible to all of them.

This means any variable that an inner function looks up via `.df_row` (as `a_freq_j` does for denominator calculation), `.alt_df_full` (as `a_freq_j` does for `denom = "n_altdf"`), `.spl_context$full_parent_df` (as Building Block 2 does for consistent sort order), or any other data.frame special argument must already exist in the appropriate dataset before `build_table()` is called. This applies to:
- Flag/indicator variables (e.g., `SAEFL`, `RELFL`) derived via conditional logic
- Recoded or collapsed factor variables
- Computed numeric variables (e.g., change-from-baseline, BMI)
- Re-leveled or re-ordered factors
- Any column produced by `mutate()`, `case_when()`, `if_else()`, or similar transformations

**Always pre-compute derived variables in data prep** (on both `df` and `alt_counts_df` when the variable is needed in both), not inside the afun.

---

## 8. Verification Checklist for Custom `afun` / `cfun` Code

Before using a custom `afun` or `cfun` in `build_table()`, verify:

1. **Reuse Check**: Could this function be replaced by `make_afun()`, `c(rvs1, rvs2)`, `junco::get_ref_info()`, or an existing `junco` function from `[select_afuns.md](./select_afuns.md)`?
2. **Signature Union Rule Check**: Does the wrapper's `formals()` include every special argument that any inner function expects? Check via `names(formals(inner_fun))` against the special arguments reference in Section 5.2. Does the wrapper include any special argument that no inner function accepts? If so, remove it — forwarding it via `...` will cause "unused argument" errors. Pay special attention to confusingly similar names (`.alt_df` vs `.alt_df_full`, `.df_row` vs `df`).
3. **Zero-Row / All-`NA` Safety**: What happens when `nrow(df) == 0` or `all(is.na(x))` in a sparse subgroup cell?
   - Does `in_rows()` still return the **exact same number of rows and row names** as a non-empty cell? (Never return a 0-row `in_rows()` with non-empty `.labels` or `.formats`, and never return fewer rows in an empty column than in populated columns).
3. **Row-Count & Row-Name Consistency Across Columns**:
   - Does every column branch in a column-conditional `afun` emit the exact same number of rows and identical `value_names()`?
   - Remember that the **left-most column** determines the stub labels (`value_labels()`), row formats, and `indent_mod()` for the entire row section.
4. **Subject vs. Event Denominator Correctness**:
   - If `df` has multiple rows per subject (e.g., `ADAE`, `ADLB`, `ADCM`), are subject counts deduplicated via `length(unique(df[[id]]))` and is `.N_col` (with `alt_counts_df = adsl`) or `.alt_df_full` used rather than `.N_row` or `nrow(df)`?
5. **Dual `afun`/`cfun` Compatibility**:
   - If the function is used in `summarize_row_groups()`, does it accept `labelstr = NULL` as its second formal parameter and assign `labelstr` to the returned row label?
7. **No Precomputed Cell Values Via External State**:
   - Does the function compute all cell values from its inputs (`df`/`x`, `.spl_context`, `.N_col`, etc.) rather than reading precomputed analysis results from the enclosing or global environment? Function factories capturing configuration parameters and environment caching per Building Block 3 are permitted; smuggling in precomputed data or results is not.
8. **Pre-Computed Derived Variables Check**:
   - Does the afun reference any variable (via `.var`, subsetting, or inner function calls) that doesn't exist in the dataset(s) passed to `build_table()`? If so, move the derivation to data prep. Variables created via `mutate()` inside the afun exist only in the `df` column-slice, not in `.df_row`, `.alt_df_full`, `.spl_context$full_parent_df`, or any other data.frame special argument (see Building Block 7). If the variable is needed in both `df` and `alt_counts_df` (e.g., for denominator calculations), it must be pre-computed in both datasets.

