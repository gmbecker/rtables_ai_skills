# Expert Skill: `junco::a_freq_j` — Complete Usage Reference

## 1. What `a_freq_j` Is

`a_freq_j` is the primary frequency-counting analysis function in `junco`. It counts unique subjects (or events) across levels of a factor variable, computes percentages against a configurable denominator, and optionally computes risk differences with confidence intervals in comparison columns. It works as both an `afun` (in `rtables::analyze()`) and a `cfun` (in `rtables::summarize_row_groups()`).

### Dual Role

| Context | How to call | What `labelstr` does |
|---|---|---|
| `analyze("AEDECOD", afun = a_freq_j, extra_args = ...)` | `labelstr` is `NULL`; one row per factor level of `.var` | N/A |
| `summarize_row_groups("AEBODSYS", cfun = a_freq_j, extra_args = ...)` | `labelstr` is auto-populated by rtables with the current facet label | Collapses all data in the facet into a single summary row labeled by `labelstr` |

When used as a `cfun`, the `var` argument in `summarize_row_groups("VARNAME", cfun = a_freq_j)` is **required** — omitting it causes an error.

---

## 2. Available Statistics (`.stats`)

The `.stats` parameter selects which statistics appear as rows. Default: `"count_unique_denom_fraction"`.

| `.stats` value | Cell value | Default format | Typical display | Use case |
|---|---|---|---|---|
| `"count_unique_denom_fraction"` | `c(n_unique, denom, fraction)` | `xx / xx (xx.x%)` | `45 / 134 (33.6%)` | Subgroup tables where denominator varies by row |
| `"count_unique_fraction"` | `c(n_unique, fraction)` | `xx (xx.x%)` | `45 (33.6%)` | Standard AE/ConMed tables where N is in column header |
| `"count_unique"` | `c(n_unique)` | `xx` | `45` | Count only, no percentage |
| `"count"` | `c(n_events)` | `xx` | `67` | Total event records (not deduplicated by subject) |
| `"unique_fraction"` | `c(fraction)` | `(xx.x%)` | `(33.6%)` | Percentage only |
| `"n_altdf"` | `c(n)` | `xx` | `134` | Denominator N from `alt_counts_df` (subgroup header rows) |
| `"n_df"` | `c(n)` | `xx` | `45` | Record count in current cell |
| `"n_rowdf"` | `c(n)` | `xx` | `268` | Subject count across all columns for current row |
| `"n_parentdf"` | `c(n)` | `xx` | `134` | Subject count from parent row split |
| `"denom"` | `c(n)` | `xx` | `134` | The resolved denominator value |

Multiple statistics can be requested: `.stats = c("n_altdf", "count_unique_fraction")`. When an n-statistic and a count statistic are both requested, the n-statistic row is placed first (unindented) and count rows are indented by 1.

### In Risk Difference Columns

When the current column is a risk difference column (detected via `grepl("difference", colid)`):
- `count`, `count_unique`, and all n-statistics are **blanked out** (display as empty).
- `count_unique_fraction`, `count_unique_denom_fraction`, and `unique_fraction` are **replaced** by the risk difference estimate + CI, formatted as `xx.x (xx.x, xx.x)`.

---

## 3. Denominator Control (`denom` and `denom_by`)

### `denom` options

| `denom` | Source | Meaning | Requires |
|---|---|---|---|
| `"N_col"` *(default)* | `.N_col` from rtables | Column header count | Nothing extra |
| `"n_altdf"` | `.alt_df_full` | Unique subjects in `alt_counts_df` for current column | `alt_counts_df = adsl` in `build_table()` |
| `"n_df"` | `df` | Unique subjects in current `(row, col)` cell | Nothing extra |
| `"n_rowdf"` | `.df_row` | Unique subjects across all columns for current row split | Nothing extra |
| `"n_parentdf"` | parent split data | Unique subjects from a higher row split | `denom_by` must be specified |
| `"N_colgroup"` | `.alt_df_full` subset | Unique subjects from a column group ancestor | `colgroup` must be specified |

### `denom_by` — Conditioning the denominator on row splits

When `denom = "n_altdf"` and the table has row splits by a subgroup variable (e.g., `split_rows_by("SEX")`), set `denom_by = "SEX"` so the denominator reflects only subjects in that subgroup within each arm, not the full arm N.

```r
extra_args = list(
  denom = "n_altdf",
  denom_by = "SEX",
  .stats = "count_unique_denom_fraction"
)
```

**Critical rule**: Variables in `denom_by` must exist on both `df` and `alt_counts_df` with matching factor levels, and must correspond to actual `split_rows_by()` variables in the layout.

When `denom = "n_parentdf"`, `denom_by` identifies which ancestor row split provides the parent denominator.

### `colgroup` — Column group denominator

When `denom = "N_colgroup"`, the denominator is the number of unique subjects in `alt_counts_df` matching the column group ancestor identified by `colgroup`. This is used when columns have nested sub-splits (e.g., `ARM -> AETOXGR`) and the percentage should be relative to the arm total, not the grade sub-column total.

```r
extra_args = list(
  denom = "N_colgroup",
  colgroup = "ARM",
  .stats = "count_unique_fraction"
)
```

**Restriction**: `N_colgroup` cannot be used in layouts with risk difference columns.

---

## 4. Risk Difference Columns (`riskdiff`, `ref_path`, `method`)

### Enabling risk differences

```r
extra_args = list(
  riskdiff = TRUE,
  ref_path = c("colspan_trt", " ", "TRT01A", "Placebo"),
  method = "wald"
)
```

- `riskdiff = TRUE` *(default)*: Activates risk difference computation in comparison columns. Set to `FALSE` to blank out comparison columns (e.g., on subgroup header rows).
- `ref_path`: Character vector of alternating `c(split_var, split_val, ...)` identifying the reference/control column. **Required** when `riskdiff = TRUE`.
- When using `grouped_cols_w_diffs()`, `ref_path` is **automatically injected** into `extra_args` — do NOT also pass it explicitly or you get a "matched by multiple actual arguments" error.

### Available `method` values

All methods from `tern::estimate_proportion_diff` are supported:

| `method` | Description |
|---|---|
| `"wald"` *(default)* | Wald (asymptotic normal) |
| `"waldcc"` | Wald with continuity correction |
| `"cmh"` | Cochran-Mantel-Haenszel |
| `"ha"` | Hauck-Anderson |
| `"newcombe"` | Newcombe (score-based) |
| `"newcombecc"` | Newcombe with continuity correction |
| `"strat_newcombe"` | Stratified Newcombe |
| `"strat_newcombecc"` | Stratified Newcombe with continuity correction |
| `"cmh_sato"` | CMH with Sato variance |
| `"cmh_mn"` | CMH with Miettinen-Nurminen variance |
| `"uncond_exact_diff"` | Unconditional exact difference |

### Stratified risk differences

Pass `variables = list(strata = "STRATA1")` to use stratified methods (CMH, stratified Newcombe).

### `conf_level`

Default `0.95`. Controls the confidence level for the risk difference CI.

### `weights_method`

Default `"cmh"`. Passed to `tern::s_proportion_diff` for stratified methods.

---

## 5. Level Filtering and Manipulation

### `val` — Keep specific levels only

```r
# Single value: produces one row labeled by val (or by `label` if provided)
extra_args = list(val = "Y", label = "Subjects with >=1 AE")

# Multiple values: produces one row per value
extra_args = list(val = c("MILD", "MODERATE"))
```

When `val` is specified and no matching data exists, a "No data to report" row is generated.

### `drop_levels` — Empirical level dropping

```r
extra_args = list(drop_levels = TRUE)
```

Drops factor levels with zero observations in `.df_row` (the row-level data across all columns). **Cannot be combined with `val`.**

### `excl_levels` — Exclude specific levels

```r
extra_args = list(excl_levels = c("MISSING", "NOT APPLICABLE"))
```

Removes the specified levels. **Cannot be combined with `val`.**

### `new_levels` — Create combined/derived levels

```r
extra_args = list(
  new_levels = list(
    c("Total"),                              # names of new levels
    list(c("LOW", "NORMAL", "HIGH"))         # values composing each new level
  ),
  new_levels_after = TRUE  # place after existing levels (FALSE = before)
)
```

Creates virtual combined levels by duplicating records. The new level "Total" will contain all records from LOW, NORMAL, and HIGH.

### `addstr2levs` — Append text to level labels

```r
extra_args = list(addstr2levs = ", n (%)")
# Produces labels like "Male, n (%)", "Female, n (%)"
```

---

## 6. Row Label Control

### `label` — Override label for single-value rows

When `val` has length 1, `label` replaces the factor level as the row label:

```r
extra_args = list(val = "Y", label = "Subjects with >=1 TEAE")
```

### `label_fstr` — sprintf-style label formatting (cfun mode)

When used as a `cfun` in `summarize_row_groups()`, `label_fstr` formats the `labelstr` (the facet label):

```r
extra_args = list(label_fstr = "Gender: %s")
# Produces "Gender: F", "Gender: M"
```

### `label_map` — Systematic label translation

A data.frame mapping factor levels to display labels. Four modes:

**Mode 1: Simple mapping** — columns `value`, `label`:
```r
label_map = data.frame(
  value = c("M", "F"),
  label = c("Male", "Female")
)
```

**Mode 2: Multi-variable mapping** — columns `var`, `value`, `label`:
```r
label_map = data.frame(
  var   = c("DTH30FL", "DTHA30FL"),
  value = c("Y", "Y"),
  label = c("Death <=30D of Last Trt", "Death >30D from Last Trt")
)
```

**Mode 3: Conditional on row split** — columns `<split_var>`, `value`, `label`:
```r
label_map = data.frame(
  SEX   = c("M", "F"),
  value = c("Y", "Y"),
  label = c("Male subjects with >=1 AE", "Female subjects with >=1 AE")
)
```

**Mode 4: Conditional multi-variable** — columns `<split_var>`, `var`, `value`, `label`:
```r
label_map = data.frame(
  PARAMCD = c("DIABP", "DIABP", "SYSBP", "SYSBP"),
  var     = c("CRIT1FL", "CRIT2FL", "CRIT1FL", "CRIT2FL"),
  value   = c("Y", "Y", "Y", "Y"),
  label   = c("<50 mmHg ...", ">105 mmHg ...", "<90 mmHg ...", ">180 mmHg ...")
)
```

**Important**: The analysis variable should be a `factor` when using `label_map`, otherwise you may get "doesn't provide labels for all values" errors.

---

## 7. Column Restriction (`restr_columns`)

Blanks out columns whose leaf label does not match any entry in `restr_columns`. Matching is case-insensitive on `tail(.spl_context$cur_col_split_val, 1)`.

```r
# Only populate the "N" column in a shift table layout
extra_args = list(restr_columns = c("N"), .stats = "n_altdf")
```

This is a **KEEP list** — columns not listed are blanked out.

---

## 8. Count Source (`countsource`)

Controls which dataset provides the numerator counts:

| `countsource` | Behavior |
|---|---|
| `"df"` *(default)* | Counts from the primary analysis dataset |
| `"altdf"` | Counts from `alt_counts_df` (the denominator dataset) |
| `"altdf_subset"` | Counts from `alt_counts_df`, restricted to levels present in the current row split |

`"altdf_subset"` is useful for subgroup processing where you want to count subjects from ADSL who belong to a specific subgroup, rather than counting from the event dataset.

---

## 9. Formatting and Display

### `.formats` — Override default formats

Named list or character vector keyed by statistic name:

```r
extra_args = list(
  .stats = "count_unique_fraction",
  .formats = list(count_unique_fraction = "xx (xx.xx%)")
)
```

### `.indent_mods` — Adjust indentation

Named integer vector. When n-statistics and count statistics are both requested, n-statistics default to indent 0 and count statistics to indent 1. Override with:

```r
extra_args = list(.indent_mods = c(n_altdf = 0L, count_unique_fraction = 0L))
```

### `.labels_n` — Override n-statistic labels

When requesting multiple n-statistics, provide custom labels:

```r
extra_args = list(
  .stats = c("n_altdf", "n_df"),
  .labels_n = c(n_altdf = "N (safety pop)", n_df = "n (with events)")
)
```

### `na_str` — NA replacement string

Default `rep("NA", 3)`. Controls how NA values are displayed. The junco formatting system replaces NA with "NE" by default. Set `na_str = NULL` to disable.

### `extrablankline` / `extrablanklineafter`

- `extrablankline = TRUE`: Adds a blank row after the last row produced by this call.
- `extrablanklineafter = "DISCONTINUED"`: Adds a blank row only after the row whose label matches the string.

**Note**: Prefer `section_div = " "` on `split_rows_by()` over `extrablankline` in production code.

---

## 10. `a_freq_j_with_exclude` — Conditional Row Suppression

Wraps `a_freq_j` and suppresses output entirely for specified row split levels:

```r
analyze("AEDECOD",
  afun = a_freq_j_with_exclude,
  extra_args = list(
    exclude_levels = list(AVISIT = "SCREENING"),
    .stats = "count_unique_fraction"
  )
)
```

The `exclude_levels` argument is a named list where names are split variable names and values are character vectors of levels to suppress. When the current row split matches an excluded level, the function returns `NULL` (no rows).

---

## 11. Internal Mechanics

### How `a_freq_j` detects risk difference columns

The function inspects `.spl_context$cur_col_id` for the substring `"difference"` (case-insensitive). If found, it calls `s_risk_diff_val_j()` instead of `s_freq_j()`, which:
1. Extracts the treatment variable and control group from `ref_path`.
2. Identifies the current treatment group via `h_get_cur_trt_grp()`.
3. Constructs response vectors for each factor level.
4. Delegates to `tern::s_proportion_diff()` for the actual CI computation.

### How denominators are resolved

1. `h_a_freq_dataprep()` calls `h_create_altdf()` which derives the denominator dataset from `.alt_df_full`, conditioned by `denom_by` row splits.
2. In non-risk-diff columns, the denominator dataset is further subset by the column expression (`col_expr`).
3. In risk-diff columns, the denominator dataset is NOT column-subset (it needs both arms for the comparison).

### How `labelstr` / cfun mode works

When `labelstr` is non-NULL (cfun mode), `h_upd_dfrow()` collapses all values of `.var` into a single factor level named by `labelstr` (optionally formatted via `label_fstr`). This means `s_freq_j` sees a single-level factor and produces one row.

---

## 12. Common Patterns (Recipes)

### Standard AE incidence table (SOC → PT with risk differences)

```r
extra_args_ae <- list(
  denom = "n_altdf",
  riskdiff = TRUE,
  ref_path = ref_path,
  method = "wald",
  .stats = "count_unique_fraction"
)

lyt |>
  # Overall row
  analyze("TRTEMFL", afun = a_freq_j,
    extra_args = append(extra_args_ae, list(val = "Y", label = "Subjects with >=1 AE"))) |>
  # SOC group summary
  split_rows_by("AEBODSYS", split_fun = trim_levels_in_group("AEDECOD")) |>
  summarize_row_groups("AEBODSYS", cfun = a_freq_j, extra_args = extra_args_ae) |>
  # PT detail rows
  analyze("AEDECOD", afun = a_freq_j, extra_args = extra_args_ae)
```

### Subgroup table with conditioned denominators

```r
# Header row: show subgroup N, blank risk diff columns
extra_args_hdr <- list(
  denom = "n_altdf", denom_by = "SEX",
  riskdiff = FALSE,
  .stats = "count_unique",
  label_fstr = "Gender: %s"
)

# Body rows: show n/N (%) with subgroup denominator, populate risk diff
extra_args_body <- list(
  denom = "n_altdf", denom_by = "SEX",
  riskdiff = TRUE, ref_path = ref_path,
  method = "wald",
  .stats = "count_unique_denom_fraction"
)

lyt |>
  split_rows_by("SEX", split_fun = drop_split_levels) |>
  summarize_row_groups("SEX", cfun = a_freq_j, extra_args = extra_args_hdr) |>
  split_rows_by("AEBODSYS", split_fun = trim_levels_in_group("AEDECOD")) |>
  summarize_row_groups("AEBODSYS", cfun = a_freq_j, extra_args = extra_args_body) |>
  analyze("AEDECOD", afun = a_freq_j, extra_args = extra_args_body)
```

### Single-flag summary row (e.g., "Subjects with >=1 AE")

```r
analyze("TRTEMFL", afun = a_freq_j, extra_args = list(
  val = "Y",
  label = "Subjects with >=1 AE",
  .stats = "count_unique_fraction"
))
```

### Shift table: restrict to N column only

```r
summarize_row_groups("TRT01A", cfun = a_freq_j, extra_args = list(
  .stats = "n_altdf",
  restr_columns = c("N"),
  riskdiff = FALSE
))
```

### Multi-variable analysis with label_map

```r
analyze(c("DTH30FL", "DTHA30FL", "DTHB30FL"),
  afun = a_freq_j,
  extra_args = list(
    val = "Y",
    .stats = "count_unique_denom_fraction",
    label_map = data.frame(
      var = c("DTH30FL", "DTHA30FL", "DTHB30FL"),
      value = "Y",
      label = c("Death <=30D of Last Trt", "Death >30D from Last Trt", "Death <=30D of First Trt")
    )
  ),
  show_labels = "hidden"
)
```

---

## 13. Gotchas and Pitfalls

1. **`.var` must be a factor**: `a_freq_j` requires the analysis variable to be a `factor`. Character variables are auto-converted only in limited circumstances (when `labelstr` is NULL and the variable is character). Always pre-convert to factor.

2. **`ref_path` double-injection with `grouped_cols_w_diffs()`**: When using `grouped_cols_w_diffs()`, it automatically injects `ref_path` into `extra_args` for comparison columns. If you also pass `ref_path` explicitly in your `extra_args`, you get "formal argument matched by multiple actual arguments". Solution: omit `ref_path` from `extra_args` when using `grouped_cols_w_diffs()`.

3. **Duplicate IDs in `alt_counts_df`**: If `alt_counts_df` has multiple rows per subject (e.g., from a cross-join with row-split variables), `a_freq_j` with `riskdiff = TRUE` will error with "Denominator has multiple records per id". Solution: use `denom_by` to condition the denominator so each subject appears once per subgroup.

4. **`val` and `drop_levels` are mutually exclusive**: Cannot use both. Same for `val` and `excl_levels`.

5. **`val`/`excl_levels`/`new_levels` cannot be used in cfun mode**: When `labelstr` is non-NULL (i.e., inside `summarize_row_groups()`), these level-manipulation arguments are forbidden.

6. **`N_colgroup` incompatible with risk diff columns**: Using `denom = "N_colgroup"` in a layout that has risk difference columns will error.

7. **Zero-denominator handling**: When the denominator is 0, the fraction is set to `NA` (displayed as "NE" or per `na_str`), not as `0/0 (NaN%)`.

8. **`riskdiff = TRUE` is the default**: Even without risk difference columns in the layout, `riskdiff = TRUE` is harmless — the function only computes risk differences when it detects it's in a risk difference column. But if `riskdiff = TRUE` and `ref_path = NULL` and the function lands in a risk difference column, it will error.

9. **Data scope constraint for derived variables**: `a_freq_j` internally references `.df_row` (for level enumeration and row-facet denominators) and `.alt_df_full` (for `denom = "n_altdf"` subject counts). Any variable passed as `.var` — or referenced by `denom_by`, `val`, `excl_levels`, or `new_levels` — must exist in the dataset(s) passed to `build_table()`, not just in a `df` slice mutated inside a custom afun wrapper. Variables created via `mutate()` inside an afun exist only in the `df` column-slice; they are invisible to `.df_row`, `.alt_df_full`, `.spl_context$full_parent_df`, and all other data.frame special arguments that `rtables` populates from the original datasets. Always pre-compute derived variables (flags, recoded factors, computed numerics, re-leveled factors) in data prep before `build_table()`, on both `df` and `alt_counts_df` when the variable is needed in both.
