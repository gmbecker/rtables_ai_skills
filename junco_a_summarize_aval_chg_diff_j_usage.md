# junco `a_summarize_aval_chg_diff_j` — 3-Column Layout Analysis Function

Expert usage reference for the 3-column (N, Value, Change) analysis function
with optional between-group difference column.

## 1. Purpose

`a_summarize_aval_chg_diff_j` produces a single-row summary presented across a
3-column layout per treatment arm:

| Column 1 | Column 2 | Column 3 |
|---|---|---|
| n/N (%) | Mean (CI) or LS Mean (CI) | Mean CFB (CI) or LS Mean CFB (CI) |

Plus an optional difference column showing the between-group difference in
change from baseline (t-test or ANCOVA-based).

This function is designed to work with `split_cols_by_multivar()` and detects
which sub-column it is in via `.spl_context$cur_col_split_val`.

## 2. Parameters

### Core Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ancova` | logical | `FALSE` | `FALSE` = descriptive (arithmetic mean + t-test); `TRUE` = ANCOVA (LS means + model-based diff) |
| `comp_btw_group` | logical | `TRUE` | Enable between-group comparison in difference columns |
| `ref_path` | character | `NULL` | Column path to reference group. **Required when `comp_btw_group = TRUE`** |
| `denom` | string | `"N"` | Denominator for col1 proportion: `"N"` (unique subjects in split) or `".N_col"` (from `alt_counts_df`) |
| `d` | numeric or string | `0` | Decimal precision. Integer for fixed precision, or variable name for data-driven precision. One extra decimal is added for means |
| `id` | string | `"USUBJID"` | Subject identifier (used when `denom = "N"`) |
| `indatavar` | string or NULL | `NULL` | If not NULL, extra-subset df to non-missing values of this variable before col1 calculations |
| `multivars` | character(3) | `c("AVAL", "AVAL", "CHG")` | Variable names for the 3-column layout. Must match the `split_cols_by_multivar()` call |
| `conf_level` | numeric | `0.95` | Confidence level for CIs |
| `format_na_str` | string | `""` | String to display for NA/empty values |

### ANCOVA-Specific Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `variables` | named list | `list(arm = "TRT01A", covariates = NULL)` | `arm`: treatment variable; `covariates`: character vector of covariate names (can include interactions like `"X1 * X2"`) |
| `interaction_y` | character | `FALSE` | Level of interaction variable to report. `FALSE` = no interaction |
| `interaction_item` | character | `NULL` | Interaction variable name. Must be set together with `interaction_y` |
| `weights_emmeans` | string | `NULL` | Passed to `emmeans::emmeans()` weights argument (e.g. `"counterfactual"`) |
| `method_combo` | string | `"contrasts"` | Method for combined-group columns: `"contrasts"` (weighted contrasts from original model) or `"collapse"` (refit model with collapsed levels) |
| `weights_combo` | string | `NULL` | Weights for combined contrasts: `"equal"`, `"proportional"`, or `"proportional_marginal"` |

### Per-Column Statistics and Formats

Unlike most junco functions, `.stats`, `.formats`, and `.formats_fun` are **named lists**
with keys `col1`, `col23`, and `coldiff`:

| Key | Available Stats | Default Stat | Default Format |
|---|---|---|---|
| `col1` | `"count_denom_frac"`, `"count_frac"`, `"count"` | `"count_denom_frac"` | `jjcsformat_count_denom_fraction` |
| `col23` | `"mean_ci_3d"` (descriptive or LS mean) | `"mean_ci_3d"` | `"xx.dx (xx.dx, xx.dx)"` |
| `coldiff` | `"mean_diff_with_ci"` (t-test or LS mean diff) | `"mean_diff_with_ci"` | `"xx.dx (xx.dx, xx.dx)"` |

The `xx.dx` format uses dynamic decimal precision controlled by `d`.

## 3. Required Layout Structure

The function **requires** `split_cols_by_multivar()` in the layout and detects
which sub-column it is in. The canonical layout pattern is:

```r
lyt <- basic_table() |>
  # Treatment columns
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = map)) |>
  split_cols_by(trtvar) |>
  # Row structure
  split_rows_by("PARAMCD") |>
  split_rows_by("AVISIT", child_labels = "hidden") |>
  # 3-column sub-split for core columns
  split_cols_by_multivar(
    c("AVAL", "AVAL", "CHG"),
    varlabels = c("n/N (%)", "Mean (CI)", "CFB (CI)")
  ) |>
  # Difference columns (restart with nested = FALSE)
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by(trtvar, split_fun = remove_split_levels(ctrl_grp),
                labels_var = "rrisk_label") |>
  # Single sub-column for difference
  split_cols_by_multivar(c("CHG"), varlabels = c(" ")) |>
  # Analysis — .var is a DUMMY variable (e.g. "STUDYID")
  analyze("STUDYID", afun = a_summarize_aval_chg_diff_j,
          extra_args = list(...))
```

### Critical Layout Rules

1. The variable passed to `analyze()` is a **dummy** — the function extracts
   actual analysis variables from the `split_cols_by_multivar` calls via
   `.spl_context$cur_col_split_val`.

2. The treatment variable must be the **last `split_cols_by` before
   `split_cols_by_multivar`**. The function discovers it via
   `.spl_context$cur_col_split`.

3. The difference columns must use a **separate** `split_cols_by_multivar`
   with only the change variable (e.g., `c("CHG")`).

4. Difference column detection uses `grepl("difference", tolower(cur_col_id))`.

## 4. How Column Detection Works

The function inspects `.spl_context` to determine which sub-column it is in:

1. Finds the position of `"multivars"` in `cur_col_split`
2. The treatment variable is at position `multivars_pos - 1`
3. The current multivar value (`last_val`) determines the behavior:
   - `last_val == multivars[1]` AND not in diff col → **Column 1** (n/N %)
   - `last_val %in% multivars[2:3]` → **Column 2 or 3** (mean/CI or CFB/CI)
   - In diff col → **Difference column** (mean diff with CI)

When `multivars[1] == multivars[2]` (both `"AVAL"`), the second occurrence is
internally renamed to `"AVAL._[[2]]_."` to distinguish them.

## 5. Descriptive vs ANCOVA Mode

### `ancova = FALSE` (Default)

- **Columns 2 & 3**: Arithmetic mean with 95% CI from `tern::s_summary()`
- **Difference column**: Two-sample t-test difference in means with CI
  (via `safe_t_test`)

### `ancova = TRUE`

- **Columns 2 & 3**: LS means with CI from `s_ancova_j()` → `emmeans`
- **Difference column**: ANCOVA-based difference in LS means with CI and p-value
- Statistics are named `lsmean_ci` and `lsmean_diff_with_ci` internally, but
  the `.stats` keys remain `col23` and `coldiff`

### Sparse Data Handling (4 Cases)

When `ancova = TRUE`, the function handles sparse data gracefully:

1. **No data at all** (`nrow(.df_row) == 0`): returns `NULL` (blank cell)
2. **Current column empty** (`nrow(df) == 0`) or only 1 treatment level: all stats = `NA`
3. **Reference group empty**: LS mean estimated, but difference = `NA`
4. **Missing factor levels**: levels are dropped from the factor before model fitting

## 6. Combined Treatment Group Support

When a combined column (e.g., "Combined A + C") is created via `add_combo_facet`,
the function detects `length(sum_level) > 1` and uses either:

- `method_combo = "contrasts"`: Weighted contrasts from the original ANCOVA model.
  Requires `weights_combo` to be set (`"equal"`, `"proportional"`, or
  `"proportional_marginal"`).
- `method_combo = "collapse"`: Refits the model with combined levels collapsed
  into a single factor level.

## 7. Complete Worked Examples

### 7.1 Basic 3-Column Table (Descriptive, No Difference)

```r
library(junco); library(rtables); library(tern); library(dplyr)

advs <- ex_advs |>
  filter(PARAMCD %in% c("DIABP")) |>
  filter(AVISIT %in% c("BASELINE", "WEEK 1 DAY 8")) |>
  mutate(PARAMCD = droplevels(PARAMCD), AVISIT = droplevels(AVISIT))

lyt <- basic_table() |>
  split_cols_by("ARM") |>
  split_rows_by("PARAMCD") |>
  split_rows_by("AVISIT", child_labels = "hidden") |>
  split_cols_by_multivar(
    c("AVAL", "AVAL", "CHG"),
    varlabels = c("n/N (%)", "Mean (95% CI)", "CFB (95% CI)")
  ) |>
  analyze("STUDYID", afun = a_summarize_aval_chg_diff_j,
    extra_args = list(d = 1, comp_btw_group = FALSE,
                      multivars = c("AVAL", "AVAL", "CHG")))

build_table(lyt, advs, alt_counts_df = ex_adsl)
```

### 7.2 3-Column Table with Difference Column (Descriptive)

```r
ctrl_grp <- "B: Placebo"
adsl <- ex_adsl |>
  mutate(
    colspan_trt = factor(ifelse(ARM == ctrl_grp, " ", "Active"),
                         levels = c("Active", " ")),
    rrisk_header = "Difference in Mean Change (95% CI)",
    rrisk_label = paste(ARM, "vs", ctrl_grp)
  )

advs <- ex_advs |>
  filter(PARAMCD == "DIABP", AVISIT %in% c("BASELINE", "WEEK 1 DAY 8")) |>
  mutate(PARAMCD = droplevels(PARAMCD), AVISIT = droplevels(AVISIT)) |>
  inner_join(adsl |> select(USUBJID, ARM, colspan_trt, rrisk_header, rrisk_label),
             by = c("USUBJID", "ARM"))

map <- create_colspan_map(adsl, non_active_grp = ctrl_grp,
  non_active_grp_span_lbl = " ", active_grp_span_lbl = "Active",
  colspan_var = "colspan_trt", trt_var = "ARM")

ref_path <- c("colspan_trt", " ", "ARM", ctrl_grp)

lyt <- basic_table() |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = map)) |>
  split_cols_by("ARM") |>
  split_rows_by("PARAMCD") |>
  split_rows_by("AVISIT", child_labels = "hidden") |>
  split_cols_by_multivar(c("AVAL", "AVAL", "CHG"),
    varlabels = c("n/N (%)", "Mean (CI)", "CFB (CI)")) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by("ARM", split_fun = remove_split_levels(ctrl_grp),
                labels_var = "rrisk_label") |>
  split_cols_by_multivar(c("CHG"), varlabels = c(" ")) |>
  analyze("STUDYID", afun = a_summarize_aval_chg_diff_j,
    extra_args = list(d = 1, ref_path = ref_path,
                      multivars = c("AVAL", "AVAL", "CHG")))

build_table(lyt, advs, alt_counts_df = adsl)
```

### 7.3 ANCOVA with Combined Columns

```r
extra_args <- list(
  d = 1, ancova = TRUE, comp_btw_group = TRUE,
  ref_path = c("ARM", "B: Placebo"),
  multivars = c("AVAL", "AVAL", "CHG"),
  variables = list(arm = "ARM", covariates = c("BASE")),
  method_combo = "contrasts",
  weights_combo = "equal"
)
```

## 8. Gotchas and Pitfalls

### 8.1 The `analyze` Variable is a Dummy

The variable passed to `analyze()` (e.g., `"STUDYID"`) is NOT used for
calculations. The actual analysis variables come from `split_cols_by_multivar`.
Any variable present on the dataset works.

### 8.2 `.stats`/`.formats`/`.formats_fun` Must Be Named Lists

Unlike other junco functions where `.stats` is a character vector, here they
must be named lists with keys `"col1"`, `"col23"`, `"coldiff"`:

```r
.stats = list(col1 = "count_frac", col23 = "mean_ci_3d", coldiff = "mean_diff_with_ci")
```

Passing a character vector will error.

### 8.3 `multivars` Must Match `split_cols_by_multivar`

The `multivars` argument must exactly match the variables passed to
`split_cols_by_multivar()`. If they differ, the column detection logic breaks.

### 8.4 Treatment Variable Must Precede `split_cols_by_multivar`

The function discovers the treatment variable as the last `split_cols_by` before
`split_cols_by_multivar`. If there's no `split_cols_by` before it, the function
errors: `"Layout must at least contain a split_cols_by prior to split_cols_by_multivar call."`

### 8.5 Difference Column Requires Separate `split_cols_by_multivar`

The difference columns must use their own `split_cols_by_multivar(c("CHG"), ...)`
call, separate from the core 3-column `split_cols_by_multivar`. The function
uses `grepl("difference", ...)` on the column ID to detect it is in a diff column.

### 8.6 `comp_btw_group = FALSE` in Difference Columns Returns NULL

When `comp_btw_group = FALSE` and the function is called in a difference column,
it returns `rcell(NULL)` — a blank cell. This is by design.

### 8.7 Dynamic Decimal Precision

The `d` parameter controls decimal places. When `d` is a string (variable name),
the precision is read from the data. The `format_xxd()` helper converts
`"xx.dx"` patterns to the appropriate precision. One extra decimal is added
for mean-based statistics.

### 8.8 Baseline Rows Show `0.00 (0.00, 0.00)` for Change

At baseline, CHG is typically 0 for all subjects, producing `0.00 (0.00, 0.00)`.
The difference column at baseline shows `0.00 (NE, NE)` because the t-test
cannot compute a CI when all values are identical.
