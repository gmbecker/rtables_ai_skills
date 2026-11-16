# Expert Skill: All Other `junco` Analysis & Content Functions

Expert usage reference for `junco` `afun`s and `cfun`s not covered by a dedicated expert skill. For functions with their own expert skill, see:
- `a_freq_j`, `a_freq_j_with_exclude` → [`junco_a_freq_j_usage.md`](./junco_a_freq_j_usage.md)
- `a_eair100_j`, `a_eair_strat_j`, `a_patyrs_j` → [`junco_eair_family_usage.md`](./junco_eair_family_usage.md)
- `a_summarize_aval_chg_diff_j` → [`junco_a_summarize_aval_chg_diff_j_usage.md`](./junco_a_summarize_aval_chg_diff_j_usage.md)
- `a_cond_proportion_j`, `a_proportion_diff_mf`, `a_test_proportion_diff_mf` → [`junco_adaptive_proportion_diff_usage.md`](./junco_adaptive_proportion_diff_usage.md)

---

## 1. Frequency Variants

### 1.1 `a_freq_combos_j` — Cumulative / Cycle Period Columns

| Property | Value |
|---|---|
| **Type** | `afun` / `cfun` |
| **Diff Col** | Yes |
| **Purpose** | Frequency count and percentage in layouts with cumulative/period column designs controlled by `combosdf`. |

**Parameters** (beyond standard `a_freq_j` args):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `combosdf` | data.frame | — | **Required.** Mapping of column facets to cumulative count definitions. Each row defines which period facets contribute to a cumulative/total column |
| `filter_var` | string | — | Record-level filter variable to prevent double-counting subjects across individual period facets |
| `flag_var` | string | — | Flag variable identifying first-occurrence records within each period |
| `do_not_filter` | character | — | Column facet labels that retain cumulative (unfiltered) numerator counts (e.g., `"Total"`) |
| `val` | character | `NULL` | Keep specific factor levels only |
| `label` | string | `NULL` | Override row label (when `val` has length 1) |
| `label_fstr` | string | `NULL` | `sprintf`-style label formatting (cfun mode) |
| `label_map` | data.frame | `NULL` | Systematic label translation (same 4 modes as `a_freq_j`) |
| `.formats` | named list | `NULL` | Override default formats |
| `.labels_n` | named char | `NULL` | Override n-statistic labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |
| `na_str` | character | `rep("NA", 3)` | NA replacement strings |

**Available `.stats`**: `"count_unique_denom_fraction"` *(default)*, `"count"`, `"count_unique"`, `"count_unique_fraction"`, `"unique_fraction"`, `"n_df"`, `"n_altdf"`, `"n_rowdf"`, `"n_parentdf"`, `"denom"`.

**Denominator control**: Same as `a_freq_j` — `denom`, `denom_by`, `countsource` all supported.

**When to select**: Columns represent time intervals or treatment cycles paired with cumulative/total period columns. The `combosdf` defines which individual-period columns roll up into each cumulative column. `filter_var` and `flag_var` prevent double-counting subjects across individual period facets while `do_not_filter` preserves cumulative counts in overall/total facets.

---

### 1.2 `a_freq_subcol_j` — Conditional Sub-Column Event Filtering

| Property | Value |
|---|---|
| **Type** | `afun` / `cfun` |
| **Diff Col** | Yes |
| **Purpose** | Frequency count and percentage with conditional column-subsetting in selected sub-columns (e.g., serious/drug-related AEs nested under arms). |

**Parameters** (beyond standard `a_freq_j` args):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `subcol_split` | string | — | **Required.** Text to match against the column path to identify sub-columns requiring filtering |
| `subcol_var` | string | — | **Required.** Variable to inspect in matched sub-columns |
| `subcol_val` | scalar | — | **Required.** Value that `subcol_var` must equal in matched sub-columns; rows not matching are excluded from the count |
| `val` | character | `NULL` | Keep specific factor levels only |
| `label` | string | `NULL` | Override row label |
| `label_fstr` | string | `NULL` | `sprintf`-style label formatting (cfun mode) |
| `label_map` | data.frame | `NULL` | Systematic label translation |
| `.formats` | named list | `NULL` | Override default formats |
| `.labels_n` | named char | `NULL` | Override n-statistic labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |
| `na_str` | character | `rep("NA", 3)` | NA replacement strings |

**Available `.stats`**: Same as `a_freq_j`.

**Denominator control**: Same as `a_freq_j`.

**When to select**: Each treatment arm is subdivided into sub-columns where specific sub-columns represent restricted subsets of events (e.g., `"All AEs"`, `"Drug-Related AEs"`, `"Serious AEs"`). The function inspects the column path for `subcol_split` and filters `df` to `df[[subcol_var]] == subcol_val` only inside matching sub-columns; non-matching sub-columns get unfiltered counts.

---

### 1.3 `a_freq_resp_var_j` — Binary Response Flag Across Categories

| Property | Value |
|---|---|
| **Type** | `afun` / `cfun` |
| **Diff Col** | Yes |
| **Purpose** | Summarizes a binary `"Y"`/`"N"` response flag variable across categories of an analysis variable, with optional risk difference. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `resp_var` | string | — | **Required.** Name of the binary response flag variable (e.g., `"TRTEMFL"`) |
| `riskdiff` | logical | `TRUE` | Enable risk difference in comparison columns |
| `ref_path` | character | `NULL` | Column path to reference group (required when `riskdiff = TRUE`) |
| `conf_level` | numeric | `0.95` | Confidence level for risk difference CI |
| `method` | string | `"wald"` | Risk difference CI method (same options as `a_freq_j`) |
| `weights_method` | string | `"cmh"` | Weights for stratified methods |
| `drop_levels` | logical | `FALSE` | Drop unobserved factor levels |
| `.formats` | named list | `NULL` | Override default formats |
| `na_str` | character | `rep("NA", 3)` | NA replacement strings |
| `legacy` | logical | `FALSE` | Use legacy output format |

**Output**: `xx / xx (xx.x%)` in arm columns (count of `resp_var == "Y"`, denominator from `.N_col`, percentage); risk difference + CI in comparison columns.

**When to select**: Summarizing a binary `"Y"`/`"N"` flag variable (`resp_var`) across the levels of a categorical variable passed to `analyze()` (e.g., `SEX` or `AGEGR1`). Distinct from `a_freq_j` with `val = "Y"` because `a_freq_resp_var_j` counts responders *within each level of `.var`* using a separate response variable, whereas `a_freq_j` counts levels of `.var` itself.

---

## 2. Max Level & Tiered Drill-Down

### 2.1 `a_maxlev` — Per-Subject Maximum of Ordered Factor

| Property | Value |
|---|---|
| **Type** | `afun` / `cfun` |
| **Diff Col** | **No** |
| **Purpose** | Counts each subject once at their highest level of an ordered factor (e.g., maximum CTCAE toxicity grade or worst AE severity). |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `any_level` | logical | — | **Required.** `TRUE` for `cfun` group row (counts subjects with any non-missing level); `FALSE` for `afun` level rows (tabulates distribution of worst grade) |
| `any_level_exclude` | character | `"Missing"` | Levels to exclude from the "any" summary when `any_level = TRUE` |
| `id` | string | `"USUBJID"` | Subject identifier for deduplication |

**Available `.stats`**: `count_fraction` — frequencies of maximum levels per subject. When `any_level = TRUE` (cfun), tabulates "any" occurrence across maximum levels.

**Denominator**: Derived from `.alt_df_full` / `alt_counts_df` (subject population).

**⚠️ Does not support difference columns**: `a_maxlev` does not support `riskdiff`/`ref_path` and cannot be used directly in layouts with difference columns (`grouped_cols_w_diffs` or manual risk-difference column branches). It will throw `"'subset' must be logical"` inside diff columns. Two workaround patterns:
- **Workaround 1 (Pre-computed factor + `a_freq_j`)**: Pre-compute the worst grade as a factor variable in data prep and use `a_freq_j` on the factor levels. Works seamlessly with `riskdiff = TRUE`.
- **Workaround 2 (Column-dispatch wrapper)**: Wrap `a_maxlev` in a column-conditional dispatch afun (Pattern 2A from [`custom_afuns.md`](./custom_afuns.md)) that calls `a_maxlev` in treatment columns and emits blank cells in difference columns.

**Dual role**:
- As `cfun` in `summarize_row_groups()`: Pass `any_level = TRUE` to count unique subjects with any non-missing grade in that organ class/term.
- As `afun` in `analyze()`: Pass `any_level = FALSE` to tabulate the distribution of each subject's worst grade.

---

### 2.2 `a_two_tier` — Selective 2-Level Drill-Down

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Depends on `grp_fun` / `detail_fun` |
| **Purpose** | Creates a two-tier nested row hierarchy within a single `analyze()` call: group summary row + conditionally indented detail rows under specified levels. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `inner_var` | string | — | **Required.** Variable for detail rows (e.g., `"DCSREAS"` for discontinuation reasons) |
| `drill_down_levs` | character | — | **Required.** Levels of `.var` that expand into indented detail rows (e.g., `"DISCONTINUED"`) |
| `use_all_levels` | logical | `FALSE` | Whether to show unobserved levels of `inner_var` |
| `grp_fun` | function | `a_freq_j` | Analysis function for outer (group summary) level |
| `detail_fun` | function | `a_freq_j` | Analysis function for inner (detail) level |
| `...` | | | Additional arguments passed to both `grp_fun` and `detail_fun` |

**Statistics and denominators**: Determined by `grp_fun` and `detail_fun` (typically `a_freq_j`).

**When to select**: A shell displays categories of an outer variable (`.var`), and **only selected categories** (`drill_down_levs`) expand into indented detail breakdown rows (`inner_var`). Example: Disposition tables expanding `"DISCONTINUED"` into discontinuation reasons while `"COMPLETED"` remains a single line.

---

### 2.3 `a_three_tier` — Selective 3-Level Drill-Down

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Depends on `grp_fun` / `detail_fun` |
| **Purpose** | Extends `a_two_tier()` to three nested levels within a single `analyze()` call (outer group → mid-level detail → innermost detail). |

**Parameters** (extends `a_two_tier`):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `inner_var1` | string | — | **Required.** Second-tier variable |
| `inner_var2` | string | — | **Required.** Third-tier variable |
| `drill_down_levs` | character | — | **Required.** Levels of `.var` that expand |
| `use_all_levels` | logical | `FALSE` | Show unobserved levels |
| `grp_fun` | function | `a_freq_j` | Group summary function |
| `detail_fun` | function | `a_freq_j` | Detail function (applied at both inner tiers) |
| `...` | | | Passed to both functions |

---

## 3. Exposure

### 3.1 `a_summarize_ex_j` — Study Drug Exposure Summary

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Tabulation for exposure tables (extent/duration of exposure in days or months, dose intensity), with descriptive summaries or ANCOVA/t-test comparisons. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `comp_btw_group` | logical | `FALSE` | Enable between-group difference estimation in comparison columns |
| `ref_path` | character | `NULL` | Column path to reference group (required when `comp_btw_group = TRUE`) |
| `ancova` | logical | `FALSE` | `TRUE` = ANCOVA model for between-group differences; `FALSE` = two-sample t-test |
| `interaction_y` | character | `FALSE` | Level of interaction variable to report |
| `interaction_item` | character | `NULL` | Interaction variable name |
| `conf_level` | numeric | `0.95` | Confidence level |
| `daysconv` | numeric | `1` | Unit conversion multiplier: `1` for days, `30.4375` for months |
| `variables` | named list | `list(arm = "TRT01A", covariates = NULL)` | Treatment variable and ANCOVA covariates |
| `.stats` | character | `c("mean_sd", "median", "range", "quantiles", "total_subject_years")` | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `na_str` | character | `rep("NA", 3)` | NA replacement strings |

**Available `.stats`**: `mean_sd`, `median`, `range`, `quantiles`, `total_subject_years` *(defaults)*; in difference columns (`comp_btw_group = TRUE`): `diff_mean_est_ci`.

**When to select**: Study Drug Exposure tables showing duration of exposure, total dose, or dose intensity. Supports unit conversion via `daysconv` and between-group comparisons via two-sample t-test (`ancova = FALSE`) or ANCOVA (`ancova = TRUE`).

---

## 4. Continuous Variables, Statistics-in-Columns & Summaries

### 4.1 `column_stats` — Statistics-in-Columns Layout

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | No |
| **Purpose** | Presents summary statistics across column variables (e.g., `AVAL`, `CHG`, `BASE` in columns, visits in rows) with varying decimal precision. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `var_names` | character | `c("AVAL", "CHG", "BASE")` | Variables analyzed, must match `split_cols_by_multivar()` call |
| `exclude_visits` | character | `c("BASELINE")` | Visits excluded from `BASE`/`CHG` columns (auto-blanked) |
| `formats_var` | string | `NULL` | List-column name in dataset defining parameter-specific formats (e.g., `"fmt_d"`) |
| `.formats` | named list | `NULL` | Override default formats |

**Available `.stats`**: `n`, `mean`, `sd`, `se`, `median`, `min`, `max`, `range`, `mean_sd`, `mean_se`.

**Required layout structure**: Must be paired with `split_cols_by_multivar()` and `stats_in_cols_setup()`:
```r
split_cols_by_multivar(c("AVAL", "BASE", "CHG"), varlabels = c(...)) |>
split_cols_by("STUDYID", split_fun = make_split_fun(post = list(stats_in_cols_setup())))
```

**⚠️ Requires `split_cols_by_multivar`**: The function detects which sub-column it is in via `.spl_context$cur_col_split_val`. Without the multivar split + `stats_in_cols_setup()`, the function will not work correctly.

---

### 4.2 `a_diff_means` — Difference in Means (t-test)

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Point estimate and inferential statistics for the difference in population means between active treatment and reference group. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ref_path` | character | — | **Required.** Reference group column path (via `get_ref_info()`) |
| `paired` | logical | `FALSE` | `TRUE` = paired t-test; `FALSE` = independent two-sample t-test |
| `paired_by` | string | `"USUBJID"` | Subject ID variable for paired t-test matching |
| `conf_level` | numeric | `0.95` | Confidence level |
| `.stats` | character | `"diff_means_est_ci"` | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `diff_means_est_ci` *(default)*, `diff_means_n1`, `diff_means_n2`, `diff_means_est`, `diff_means_se`, `diff_means_est_se`, `diff_means_ci`.

---

### 4.3 `a_summary_j_with_exclude` — Continuous Summary with Row Exclusion

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | No |
| **Purpose** | Dataframe-level wrapper around `tern::a_summary()` that excludes specified row split levels from producing numeric summaries. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `exclude_levels` | named list | — | **Required.** Row split levels to exclude: `list(AVISIT = "BASELINE")` |
| `.var` | string | — | Variable to extract and summarize from `df` |
| `...` | | | All parameters of `tern::a_summary()` passed through |

**Available `.stats`**: All statistics from `tern::a_summary()`: `n`, `sum`, `mean`, `sd`, `se`, `mean_sd`, `mean_se`, `mean_ci`, `median`, `mad`, `median_ci`, `quantiles`, `iqr`, `range`, `min`, `max`, `median_range`, `cv`, `geom_mean`, `geom_sd`, `geom_mean_sd`, `geom_mean_ci`, `geom_cv`, `mean_diff_with_ci`.

---

### 4.4 `a_summary_subset` / `c_summary_subset_label` — Filtered Subset Summary

**`a_summary_subset`** (`afun`):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `filter_expr` | expression | `NULL` | Logical filter expression evaluated within `df`, e.g., `expression(AGE > 30)` |
| `formats_var` | string | `NULL` | Varying decimal precision specification (e.g., `"fmt_d"`) |
| `...` | | | All `tern::a_summary()` parameters |

**`c_summary_subset_label`** (`cfun`):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `filter_expr` | expression | `NULL` | Same as above |
| `label` | string | `NULL` | Prepended label string for the content row |
| `label_indent_mod` | integer | `NULL` | Indent modifier for the label |
| `...` | | | All `tern::a_summary()` parameters |

**Available `.stats`**: All statistics from `tern::a_summary()`.

**Diff Col**: No.

---

### 4.5 `a_sum_ratio_j` — Sums and Ratios of Sums

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | No |
| **Purpose** | Computes sums and ratios of sums (e.g., cumulative dose received vs. planned), optionally deduplicated by subject ID. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `denom_by` | string | — | **Required for ratio stats.** Name of column in `df` whose sum serves as denominator in `ratio` and `ratio_unique` |
| `id_var` | string | `"USUBJID"` | Subject ID variable (required for `*_unique` stats) |
| `.stats` | character | `"sum"` | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |
| `na_str` | character | `rep("NA", 3)` | NA replacement strings |

**Available `.stats`**: `sum` *(default)*, `sum_unique`, `ratio`, `ratio_unique`.

---

## 5. ANCOVA, MMRM & LS Means

### 5.1 `a_summarize_ancova_j` — Comprehensive ANCOVA Summary

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Comprehensive ANCOVA summary supporting single and combined treatment groups via linear contrasts or model collapsing. Produces both descriptive statistics and ANCOVA model estimates. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ref_path` | character | — | **Required.** Reference arm column path |
| `variables` | named list | `list(arm = "TRT01A", covariates = NULL)` | `arm`: treatment variable; `covariates`: character vector of covariate names (can include interactions like `"X1 * X2"`) |
| `method_combo` | string | `"contrasts"` | Method for combined-group columns: `"contrasts"` (weighted contrasts from original model) or `"collapse"` (refit model with collapsed levels) |
| `weights_combo` | string | `NULL` | Contrast weights: `"equal"`, `"proportional"`, or `"proportional_marginal"` |
| `weights_emmeans` | string | `NULL` | Passed to `emmeans::emmeans()` weights argument (e.g., `"counterfactual"`) |
| `conf_level` | numeric | `0.95` | Confidence level |
| `interaction_y` | character | `FALSE` | Level of interaction variable to report. `FALSE` = no interaction |
| `interaction_item` | character | `NULL` | Interaction variable name |
| `.stats` | character | see below | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**:
- Descriptive: `n`, `sum`, `mean`, `sd`, `se`, `mean_sd`, `mean_se`, `mean_ci`, `mean_ci_3d`, `median`, `mad`, `median_ci`, `quantiles`, `range`, `geom_mean`, `geom_sd`, `geom_mean_sd`, `geom_mean_ci`, `geom_cv`.
- ANCOVA: `n_fit`, `lsmean`, `lsmean_se`, `lsmean_ci`, `lsmean_diff`, `lsmean_diff_ci`, `lsmean_diff_with_ci`, `pval`.
- Defaults: `n`, `mean_sd`, `median`, `range`, `quantiles`, `lsmean_se`, `lsmean_ci`, `lsmean_diff_with_ci`, `pval`.

**Combined treatment group support**: When a combined column (e.g., "Combined A + C") is created via `add_combo_facet`:
- `method_combo = "contrasts"`: Weighted contrasts from the original ANCOVA model. Requires `weights_combo`.
- `method_combo = "collapse"`: Refits the model with combined levels collapsed into a single factor level.

---

### 5.2 `a_summarize_mmrm` / `a_summarize_mmrm_with_exclude` — MMRM

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Dynamic tabulation of Mixed Model for Repeated Measures (MMRM) results within tables split by treatment arms and visits. |

**Parameters** (`a_summarize_mmrm`):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `variables` | named list | — | **Required.** `list(arm = ..., visit = ..., id = ..., covariates = ..., response = ...)` |
| `ref_levels` | named list | — | **Required.** Reference levels: `list(visit = "BASELINE", arm = "Placebo")` |
| `cor_struct` | string | `"unstructured"` | Correlation structure: `"toeplitz"`, `"unstructured"`, `"ar1"`, etc. |
| `conf_level` | numeric | `0.95` | Confidence level |
| `alternative` | string | `"two.sided"` | `"two.sided"`, `"less"`, `"greater"` |
| `show_relative` | string | `NULL` | `"reduction"` or `"increase"` for relative change display |
| `.stats` | character | see below | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `n`, `adj_mean_se`, `adj_mean_ci`, `adj_mean_est_ci`, `diff_mean_se`, `diff_mean_ci`, `diff_mean_est_ci`, `change`, `p_value`.

**`a_summarize_mmrm_with_exclude`**: Wrapper adding `exclude_levels` (named list of row split levels to exclude, e.g., `list(AVISIT = "BASELINE")`), plus all parameters of `a_summarize_mmrm`.

---

### 5.3 `a_lsmeans` — Pre-Fitted LS Means

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Formatted tabulation of Least Squares Means and contrasts from pre-fitted models (`fit_mmrm_j()` or `fit_ancova()`). |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ref_path` | character | — | **Required.** Reference arm column path |
| `conf_level` | numeric | `0.95` | Confidence level |
| `alternative` | string | `"two.sided"` | `"two.sided"`, `"less"`, `"greater"` |
| `show_relative` | string | `NULL` | `"reduction"` or `"increase"` |
| `.stats` | character | see below | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `n`, `adj_mean_se`, `adj_mean_ci`, `adj_mean_est_ci`, `diff_mean_se`, `diff_mean_ci`, `diff_mean_est_ci`, `change`, `p_value`.

**When to select**: The MMRM or ANCOVA model has already been pre-fitted via `fit_mmrm_j()` or `fit_ancova()` and results merged into the data frame. Use instead of `a_summarize_mmrm` when the model is fitted outside the table-building pipeline.

---

### 5.4 `a_rbmi_lsmeans` — RBMI Pooled LS Means

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Tabulates pooled Least Squares Means, differences, and CIs from Reference-Based Multiple Imputation (RBMI) via Rubin's rules. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ref_path` | character | — | **Required.** Reference arm column path |
| `.stats` | character | see below | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `adj_mean_se`, `adj_mean_ci`, `diff_mean_se`, `diff_mean_ci`, `change`, `p_value`, `additional_title_row`.

---

## 6. Proportions, Risk Differences, Hypothesis Tests, Odds Ratios, Relative Risk & CMH

### 6.1 `a_proportion_ci_logical` / `a_proportion_ci_factor` — Proportion CI

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | No |
| **Purpose** | Proportion confidence intervals for a logical vector (`_logical`) or for each level of a factor variable (`_factor`). |

**Parameters** (identical for both):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `conf_level` | numeric | `0.95` | Confidence level |
| `method` | string | `"clopper-pearson"` | CI method: `"wald"`, `"clopper-pearson"`, `"wilson"`, `"wilsonc"`, `"strat_wilson"`, `"agresti-coull"`, `"jeffreys"` |
| `formats` | named list | `NULL` | Override formats: `list(prop_ci = ...)` |

**Available `.stats`**: `prop_ci` — confidence interval in percent, returned as formatted `rcell`.

**Denominator**: Derived from `nrow(.alt_df)`.

**When to select**: A row only needs the formatted proportion CI using a fixed method. For adaptive method selection (switching between exact and asymptotic based on count thresholds), use `a_cond_proportion_j` instead (see [`junco_adaptive_proportion_diff_usage.md`](./junco_adaptive_proportion_diff_usage.md)).

---

### 6.2 `a_proportion_diff_j` — Risk Difference vs. Reference Group

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Estimates proportion differences (risk difference) and CI vs. reference group for a logical response variable. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ref_path` | character | — | **Required.** Column path to reference group |
| `method` | string | `"wald"` | CI method: `"wald"`, `"waldcc"`, `"cmh"`, `"ha"`, `"newcombe"`, `"newcombecc"`, `"strat_newcombe"`, `"strat_newcombecc"`, `"cmh_sato"`, `"cmh_mn"`, `"uncond_exact_diff"` |
| `variables` | named list | `list(strata = NULL)` | Stratification factors: `list(strata = "STRATA1")` |
| `conf_level` | numeric | `0.95` | Confidence level |
| `label` | string | `NULL` | Override row label |
| `.stats` | character | `NULL` | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `diff`, `diff_ci`, `diff_est_ci` (`"% Difference (conf_level CI)"`).

**When to select**: Standalone risk difference rows against a reference group using the junco `ref_path` pattern. For adaptive method selection based on the Mantel-Fleiss criterion, use `a_proportion_diff_mf` instead (see [`junco_adaptive_proportion_diff_usage.md`](./junco_adaptive_proportion_diff_usage.md)).

---

### 6.3 `a_test_proportion_diff` — Hypothesis Test for Proportion Difference

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Hypothesis test for the difference between two proportions vs. reference group. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ref_path` | character | — | **Required.** Reference arm column path |
| `method` | string | `"chisq"` | Test method: `"chisq"`, `"cmh"`, `"cmh_wh"`, `"fisher"`, `"schouten"` |
| `variables` | named list | `list(strata = NULL)` | Stratification factors |
| `alternative` | string | `"two.sided"` | `"two.sided"`, `"less"`, `"greater"` |
| `.stats` | character | `"pval"` | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `pval`.

**When to select**: Standalone p-value rows for proportion difference tests. For adaptive method selection (CMH vs. Fisher's exact based on Mantel-Fleiss criterion), use `a_test_proportion_diff_mf` instead.

---

### 6.4 `a_combo_prop_diff_pval_mf` — Combined Proportions + Risk Diff + p-value

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Combined analysis presenting proportions, treatment-vs-control risk differences, and p-values across distinct columns within the same row, with Mantel-Fleiss method selection. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `colpaths` | named list | — | **Required.** Maps `"prop"`, `"diff"`, and `"pval"` to column paths |
| `trt_var` | string | — | **Required.** Treatment variable name |
| `ctrl_group` | string | — | **Required.** Control group level |
| `val` | scalar | `NULL` | Response level in factor `.var` |
| `mf_threshold` | numeric | `5` | Mantel-Fleiss criterion threshold |
| `exact_footnote` | string/RefFootnote | `RefFootnote(...)` | Footnote when exact method is used |
| `variables` | named list | `list(strata = NULL)` | Stratification factors |
| `conf_level` | numeric | `0.95` | Confidence level |
| `alternative` | string | `"two.sided"` | Alternative hypothesis |
| `.stats` | named list | see below | Per-column statistics |
| `.formats` | named list | `NULL` | Override formats |

**Available `.stats`** (named list):
- `prop`: `"count_denom_fraction"` *(default)*, `"count_fraction"`, `"count"`
- `diff`: `"diff_est_ci"` *(default)*, `"diff"`, `"diff_ci"`
- `pval`: `"pval"` *(default)*

**When to select**: A single table row presents proportions, risk differences, and p-values across separate column paths. The function dispatches to the appropriate calculation based on which column it is in (identified via `colpaths`).

---

### 6.5 `a_odds_ratio_j` — Odds Ratio

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Estimates Odds Ratio (OR), CI, and p-value vs. reference group. Supports unstratified logistic regression, conditional logistic regression, and CMH. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `method` | string | `"exact"` | `"exact"`, `"approximate"`, `"efron"`, `"breslow"`, `"cmh"` |
| `ref_path` | character | — | **Required.** Reference arm column path |
| `variables` | named list | `NULL` | `list(arm = ..., strata = ...)` |
| `conf_level` | numeric | `0.95` | Confidence level |
| `na_if_no_events` | logical | `FALSE` | Return NA instead of error when no events in a group |
| `groups_list` | named list | `NULL` | Custom group definitions |
| `.stats` | character | `NULL` | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `or_ci` (OR estimate and CI: `est`, `lcl`, `ucl`), `pval`, `n_tot` (total sample size).

---

### 6.6 `a_relative_risk` — Stratified Relative Risk (CMH)

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Estimates stratified Relative Risk (RR), CI, and p-value using the CMH method with Greenland & Robins (1985) variance estimation. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ref_path` | character | — | **Required.** Reference arm column path |
| `variables` | named list | — | **Required.** `list(strata = ...)` — stratification factors required |
| `conf_level` | numeric | `0.95` | Confidence level |
| `.stats` | character | `NULL` | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `rel_risk_ci` (RR estimate and CI: `est`, `lcl`, `ucl`), `pval`.

---

### 6.7 `a_cmhrms_j` / `a_cmhrms_j_with_exclude` — CMH Row Mean Scores Test

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | Yes |
| **Purpose** | Cochran-Mantel-Haenszel (CMH) Row Mean Scores test comparing ordered categorical response levels between treatments and reference group. |

**Parameters** (`a_cmhrms_j`):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ref_path` | character | — | **Required.** Reference arm column path |
| `variables` | named list | — | **Required.** `list(arm = ..., strata = ...)` (strata optional) |
| `collapse_combo` | logical | `FALSE` | Whether multiple arm levels in `df` are collapsed into `"Combined"` |
| `.stats` | character | `"pval"` | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `pval` (row mean scores test p-value).

**`a_cmhrms_j_with_exclude`**: Wrapper adding `exclude_levels` (named list of row split levels to exclude), plus all parameters of `a_cmhrms_j`.

---

## 7. Specialized Functions

### 7.1 `prop_table_afun` — Proportion Table

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | No |
| **Purpose** | Proportion tables with column splits prepared via `prop_split_fun()`. Shows counts, percentages, and cumulative percentages across sub-columns. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `add_total_level` | logical | `FALSE` | Add a "Total" row summing all categories |
| `formats` | named list | `NULL` | Override formats for `n`, `percent`, `cum_percent` sub-columns |

**Output**: In `n` col: category counts (+ optional Total); in `percent` col: category percentages; in `cum_percent` col: cumulative percentages.

**Denominator**: Total records in column slice (`length(x)`).

**Required pairing**: Must be used with `prop_split_fun()` to set up the 3-column layout.

---

### 7.2 `c_proportion_logical` — Simple Logical Proportion (cfun)

| Property | Value |
|---|---|
| **Type** | `cfun` |
| **Diff Col** | No |
| **Purpose** | Simple content function computing the count and proportion of `TRUE` values in a logical vector relative to column count. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `label_fstr` | string | `"%s"` | `sprintf` string using `labelstr` |
| `format` | string | `NULL` | Override format |

**Available `.stats`**: `est_prop` — `c(num, num / denom)` where denom is `.N_col`.

---

### 7.3 `a_event_free` — Milestone Kaplan-Meier Survival

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | No |
| **Purpose** | Time-point survival (event-free rate) analysis. Extends `tern::s_surv_timepoint` with a combined 3D statistic. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `time_point` | numeric | — | **Required.** Milestone time (e.g., `6` for 6-month survival) |
| `time_unit` | string | — | **Required.** Time unit (e.g., `"month"`, `"week"`) |
| `is_event` | string | — | **Required.** Event indicator variable name |
| `percent` | logical | `TRUE` | `TRUE` for percentage display, `FALSE` for 0–1 proportion |
| `control` | list | `tern::control_surv_timepoint()` | Control parameters for survival estimation |
| `.stats` | character | `"event_free_ci"` | Statistics to display |
| `.formats` | named list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |

**Available `.stats`**: `event_free_ci` *(combined rate & CI, `xx.xx (xx.xx, xx.xx)`)*, `pt_at_risk`, `event_free_rate`, `rate_se`, `rate_ci`.

---

### 7.4 `tefos03_afun` — Multivariable Cox Regression (Internal)

| Property | Value |
|---|---|
| **Type** | `afun` *(internal)* |
| **Diff Col** | No |
| **Purpose** | Internal analysis function used by `summarize_coxreg_multivar()` to extract multivariable Cox model coefficients, hazard ratios, CIs, and p-values. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `variables` | named list | — | `list(time = ..., event = ..., arm = ..., covariates = ...)` |
| `control` | list | `tern::control_coxreg()` | Cox regression control parameters |
| `formats` | named list | `NULL` | Override formats |

**Output**: Model fit col: `coef_se`, `pval`; Hazard ratio col: `hr_est`, `hr_ci`.

**Note**: Not typically called directly — used internally by `summarize_coxreg_multivar()`.

---

### 7.5 `response_by_var` — Response Flag Across Factor Levels

| Property | Value |
|---|---|
| **Type** | `afun` |
| **Diff Col** | No |
| **Purpose** | Derives `count_denom_fraction` (`xx / xx (xx.x%)`) for records with response flag `"Y"` in `resp_var` across levels of `.var`, with denominator based on non-missing records. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `resp_var` | string | — | **Required.** Response flag variable name |
| `id` | string | `"USUBJID"` | Subject ID for deduplication |
| `.format` | string | `NULL` | Override format |

**Available `.stats`**: `count_denom_fraction` — `c(num, denom, num / denom)`.

**Denominator**: Non-missing observations of `.var` in input `df`.

---

## 8. RESP01 Response Table Family

### 8.1 `resp01_acfun` — Dual Analysis/Content Function

| Property | Value |
|---|---|
| **Type** | `afun` / `cfun` |
| **Diff Col** | N/A (column-dispatched) |
| **Purpose** | Dual analysis and content function for `RESP01` response tables. Dispatches to counts/CIs in arm columns, and comparative stats (RR, OR, p-value) in Overall column. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `methods` | named list | — | **Required.** Specifies methods: `comp_stat_ci` (`"rr"`, `"or_cmh"`, `"or_logistic"`), `pval` (`"fisher"`, `"chisq"`, or CMH), `prop_ci` (`"wald"`, `"clopper-pearson"`) |
| `formats` | named list | `NULL` | Override formats |
| `include_comp` | factor levels | `NULL` | Factor levels included in comparisons |
| `arm` | string | — | Treatment variable name |
| `strata` | character | `NULL` | Stratification variables |
| `conf_level` | numeric | `0.95` | Confidence level |

**Output**:
- In arm columns: `count_prop` (count & proportion), `prop_ci` (proportion CI).
- In Overall column: `comp_stat_ci` (RR or OR estimate and CI), `pval` (p-value).

**Denominator**: Derived from `.alt_df` (`nrow(.alt_df)`).

**Required pairing**: Use with `resp01_split_fun_fct` to split treatment arm columns into `count_prop` and `prop_ci` sub-columns.

---

### 8.2 `resp01_a_comp_stat_logical` / `resp01_a_comp_stat_factor`

**`resp01_a_comp_stat_logical`** (`afun`): Computes comparative statistics (RR or OR) and p-values for logical response variables.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `methods` | named list | — | Same as `resp01_acfun` |
| `formats` | named list | `NULL` | Override formats |
| `arm` | string | — | Treatment variable |
| `strata` | character | `NULL` | Stratification variables |
| `conf_level` | numeric | `0.95` | Confidence level |
| `include` | logical | `TRUE` | Whether to include this response in comparisons |

**`resp01_a_comp_stat_factor`** (`afun`): Same but for factor variables.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `include` | character | — | Character vector of factor levels to include in comparisons |
| (other params) | | | Same as `resp01_a_comp_stat_logical` |

**Available `.stats`** (both): `comp_stat_ci`, `pval`.

---

### 8.3 `resp01_counts_cfun` — Subgroup Sample Size Counts

| Property | Value |
|---|---|
| **Type** | `cfun` |
| **Diff Col** | N/A |
| **Purpose** | Content function for subgroup sample size counts from `.alt_df` in `count_prop` columns for `RESP01` response tables. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `label_fstr` | string | `"%s"` | `sprintf` string using `labelstr` |

**Available `.stats`**: `count` — `nrow(.alt_df)`.

---

## 9. Compliance & Wide-Format LS Means

### 9.1 `cmp_cfun` — Study Visit Compliance

| Property | Value |
|---|---|
| **Type** | `cfun` |
| **Diff Col** | N/A |
| **Purpose** | Content function for study compliance tables split across three columns: Expected visits (`N`), Received visits (`n (%)`), and Missing visits (`n (%)`). |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `variables` | named list | — | **Required.** `list(expected = ..., received = ..., missing = ...)` — column names in `df` |
| `formats` | named list | `NULL` | Override formats: `list(count_percent = ...)` |

**Output**: Expected col: `n_exp`; Received col: `n_rec`, `rec_percent`; Missing col: `n_mis`, `mis_percent`.

**Denominator**: Received and Missing percentages use Expected visits count (`sum(df[[variables$expected]])`) as the dynamic denominator.

---

### 9.2 `lsmeans_wide_cfun` — Wide-Format ANCOVA LS Means

| Property | Value |
|---|---|
| **Type** | `cfun` |
| **Diff Col** | N/A |
| **Purpose** | Content function for wide-format ANCOVA LS means table layouts, organizing estimates, MSE, df, contrasts, and p-values across multi-tiered columns. |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ref_level` | string | — | **Required.** Reference treatment level |
| `treatment_levels` | character | — | **Required.** Treatment levels to display |
| `pval_sided` | string | `"2"` | `"2"` (two-sided), `"1"` (one-sided greater), `"-1"` (one-sided less) |
| `conf_level` | numeric | `0.95` | Confidence level |
| `formats` | named list | `NULL` | Override formats |

**Output**: Reference/Testing cols: `treatment`, `n`, `lsmean`, `se`; Variance col: `mse`, `df`; Comparison col: `lsmean` (difference), `se`, `ci`, `pval`.

---

## 10. Utility Functions

### 10.1 `c_row_counts` / `c_row_counts_alt` — Row Counts

**`c_row_counts`** (`cfun`, internal): Counts rows in current data split (`nrow(df)`). Wrapped by `summarize_row_counts(alt_counts = FALSE)`.

**`c_row_counts_alt`** (`cfun`, internal): Counts rows in alternative dataset (`nrow(.alt_df)`). Wrapped by `summarize_row_counts(alt_counts = TRUE)`.

**Parameters** (both):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `label_fstr` | string | `"%s"` | `sprintf` string using `labelstr` |

**Available `.stats`**: `count`.

**Note**: Not typically called directly — use `summarize_row_counts()` layout function instead.

---

### 10.2 `ac_blank_line` — Blank Spacer Row

| Property | Value |
|---|---|
| **Type** | `afun` / `cfun` |
| **Diff Col** | N/A |
| **Purpose** | Produces a blank row gap in table layouts with invisible `NA` string (`""`). |

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `labelstr` | string | `""` | Row label string |

**Output**: Single blank cell (`NA_real_` formatted as `""`).

**⚠️ Legitimate uses only**:
1. As a visual spacer `analyze()` between two substantive analysis sections (or via `insert_blank_line()`).
2. As a `cfun` in `summarize_row_groups(cfun = ac_blank_line)` paired with `split_rows_by(..., child_labels = "visible")` to insert a blank line between a visible facet label row and its child rows.

**RULE: IF you find yourself using `ac_blank_line` as the sole terminating `afun` of a `split_rows_by()` chain, THEN STOP — this is the No-Op `analyze()` Anti-Pattern. See [`row_struct_layout.md`](./row_struct_layout.md).**
