# Junco Analysis Functions (`afun`) and Content Functions (`cfuns`) Reference Guide

## Executive Overview

The **`junco`** package provides statistical analysis functions (**`afun`**) and content functions (**`cfun`**) tailored for clinical trial tabulation in R. Developed by the Clinical & Statistical Programming (C&SP) methodology team at Johnson & Johnson, `junco` is built on top of the **`rtables`**, **`formatters`**, and **`tern`** ecosystem.

### Key Architectural Differences from `tern`

1. **Direct `afun` and `cfun` Paradigm**:
   - In `tern`, tables are traditionally constructed using high-level layout wrapper functions (e.g., `count_occurrences()`, `summarize_vars()`, `analyze_num_patients()`) that wrap `rtables::analyze()`.
   - In `junco`, statistical analysis logic is implemented as standalone functions prefixed with `a_` (or `c_` / `ac_`) that are passed directly as the `afun` argument to `rtables::analyze()` or as the `cfun` argument to `rtables::summarize_row_groups()`.
   - Several `junco` functions are dual-purpose **`a/cfun`** functions (e.g., `a_freq_j`, `a_eair100_j`, `a_maxlev`, `resp01_acfun`) that can seamlessly serve as an `afun` in table body analyses or as a `cfun` in row group header summaries.

2. **Structural Support for Spanning Columns & Risk Differences**:
   - `junco` introduces structural layout generators like `grouped_cols_w_diffs()` and `create_colspan_map()` to create active-vs-comparator spanning headers alongside dedicated difference columns.
   - `junco` analysis functions natively interface with `get_ref_info()` and `ref_path` to resolve comparator/control columns across complex, nested, or multivariable column splits.
   - In risk difference columns, fractional statistics are automatically substituted with risk/mean difference point estimates and confidence intervals, while baseline/count statistics are automatically blanked out.

3. **Varying Decimal Precision (`formats_d`)**:
   - Analysis functions like `column_stats`, `a_summarize_aval_chg_diff_j`, and `a_summary_subset` support dynamic decimal formatting (`formats_var` and `fmt_d`), allowing row-specific decimal precision across clinical laboratory, vital signs, and pharmacokinetic parameters.

---

## Master Summary Table

| Name | Type | Description | Statistics Supported | Denominators Supported | Other Customization |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`a_freq_j`** | `afun` / `cfun` | Primary frequency and percentage analysis/content function. Tabulates counts, unique subject counts, fractions, and risk differences across treatment arms and difference columns. | `count_unique_denom_fraction` *(default)*, `count`, `count_unique`, `count_unique_fraction`, `unique_fraction`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, `denom`. In risk diff columns: risk difference + CI. | `N_col` *(default)*, `n_df`, `n_altdf`, `N_colgroup`, `n_rowdf`, `n_parentdf`. Subgroup denominator conditioning via `denom_by`. Data source via `countsource` (`"df"`, `"altdf"`, `"altdf_subset"`). | `riskdiff` (toggles RD calculation), `ref_path` (reference group path), `method` (`"wald"`, `"waldcc"`, `"cmh"`, `"ha"`, `"newcombe"`, `"newcombecc"`, `"strat_newcombe"`, `"strat_newcombecc"`, `"cmh_sato"`, `"cmh_mn"`, `"uncond_exact_diff"`), `conf_level`, `weights_method`, `val` (filter levels), `drop_levels`, `excl_levels`, `new_levels`, `new_levels_after`, `addstr2levs`, `label`, `label_fstr`, `label_map` (conditional row-split label translation), `.formats`, `.indent_mods`, `na_str`, `restr_columns`, `extrablankline`. |
| **`a_freq_j_with_exclude`** | `afun` / `cfun` | Wrapper around `a_freq_j()` that conditionally excludes specified row split levels from producing analysis output. | Identical to `a_freq_j()`. | Identical to `a_freq_j()`. | `exclude_levels` (named list mapping split variables to vectors of levels to exclude from tabulation), plus all parameters of `a_freq_j()`. |
| **`a_freq_combos_j`** | `afun` / `cfun` | Frequency count and percentage in layouts with cumulative/period column designs controlled by `combosdf`. | `count_unique_denom_fraction` *(default)*, `count`, `count_unique`, `count_unique_fraction`, `unique_fraction`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, `denom`. | `N_col`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, with `denom_by` support. | `combosdf` (mapping of column facets to cumulative counts), `filter_var` (record filter to prevent overcounting), `flag_var`, `do_not_filter` (facets retaining cumulative numerator, e.g. Total), `val`, `label`, `label_fstr`, `label_map`, `.formats`, `.labels_n`, `.indent_mods`, `na_str`. |
| **`a_freq_subcol_j`** | `afun` / `cfun` | Frequency count and percentage with conditional column-subsetting in selected sub-columns (e.g. serious/drug-related AEs nested under arms). | `count_unique_denom_fraction` *(default)*, `count`, `count_unique`, `count_unique_fraction`, `unique_fraction`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, `denom`. | `N_col`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, with `denom_by` support. | `subcol_split` (column path matching text), `subcol_var` (variable to inspect in matched columns), `subcol_val` (required subset value), `val`, `label`, `label_fstr`, `label_map`, `.formats`, `.labels_n`, `.indent_mods`, `na_str`. |
| **`a_freq_resp_var_j`** | `afun` / `cfun` | Summarizes binary/response variables (with `"Y"`/`"N"` flags) across categories of an analysis variable, with optional risk difference. | Count, denominator, percentage (`xx / xx (xx.x%)`); in risk difference columns: difference + CI (`xx.x (xx.x, xx.x)`). | Derived from column count `.N_col`. | `resp_var` (name of response flag variable, e.g. `"TRTEMFL"`), `riskdiff` (toggles RD calculation), `ref_path`, `conf_level`, `method` (RD methods: `"wald"`, `"cmh"`, etc.), `weights_method`, `drop_levels`, `.formats`, `na_str`, `legacy`. |
| **`a_maxlev`** | `afun` / `cfun` | Calculates count and percentage of the maximum level of an ordered factor (e.g. maximum AE toxicity grade or severity) per unique subject (`id`). | Frequencies of maximum levels per subject (`count_fraction`); when `any_level = TRUE` (used as `cfun`), tabulates "any" occurrence across maximum levels. | Derived from `.alt_df_full` / `alt_counts_df` (patient population). | `any_level` (`TRUE` for `cfun` group row, `FALSE` for `afun` level rows), `any_level_exclude` (levels to exclude from "any" summary, default `"Missing"`), `id` (subject identifier, default `"USUBJID"`). |
| **`a_two_tier`** | `afun` | Creates a two-tier nested row hierarchy within a single `analyze()` call: displays a group summary row and conditionally generates indented detail rows under specified levels. | Determined by `grp_fun` (outer level) and `detail_fun` (inner detail level), typically `a_freq_j()`. | Inherited from `grp_fun` and `detail_fun` via `.alt_df_full` / `.N_col` / `denom`. | `inner_var` (variable for detail rows), `drill_down_levs` (levels of `.var` that expand into detail rows), `use_all_levels` (whether to show unobserved levels of `inner_var`), `grp_fun`, `detail_fun`, `...` (passed to both functions). |
| **`a_three_tier`** | `afun` | Extends `a_two_tier()` to three nested levels within a single `analyze()` call (outer group summary -> mid-level detail -> innermost detail). | Determined by `grp_fun` and `detail_fun`. | Inherited from `grp_fun` and `detail_fun`. | `inner_var1` (second-tier variable), `inner_var2` (third-tier variable), `drill_down_levs`, `use_all_levels`, `grp_fun`, `detail_fun`, `...`. |
| **`a_eair100_j`** | `afun` / `cfun` | Exposure-Adjusted Incidence Rate (EAIR) summary per 100 person-years (or custom multiplier), with between-group rate differences and CIs. | `eair` *(default)*, `n_event`, `person_years`, `n_eair`, `eair_ci`, `eair_est_ci`, `eair_n`, `eair_n_py`, `eair_diff_est_ci`, `eair_diff_est`, `eair_diff_ci`. | Person-years of follow-up exposure (`fup_var` in years, or `occ_dy / 365.25` for event subjects) derived from `df` and `.alt_df_full`. | `fup_var` (follow-up duration variable in years), `occ_var` (first-occurrence event flag `"Y"`), `occ_dy` (event day relative to exposure), `num_p_year` (rate multiplier, default `100`), `count_multiple_events`, `conf_type_eair` (`"exact"`, `"normal"`, `"normal_log"`, `"byar"`), `conf_type_diff` (`"wald"`, `"waldcc"`, `"mn"`, `"scas"`), `riskdiff`, `ref_path`, `conf_level`, `riskdiff_setup` (`"horizontal"`, `"vertical"`), `row_labels_adj`, `.stats`, `.formats`, `.labels`, `na_str`. |
| **`a_eair_strat_j`** | `afun` / `cfun` | Stratified Exposure-Adjusted Incidence Rate (EAIR) across clinical strata (e.g., study size-weighted EAIR), with stratified rate differences. | `eair_strat` *(default)*, `n_event_total`, `person_years_total`, `n_eair_strat`, `eair_strat_ci`, `eair_strat_est_ci`, `eair_strat_n`, `eair_strat_n_py`, `eair_strat_diff_est_ci`, `eair_strat_diff_est`, `eair_strat_diff_ci`. | Total stratified person-years across strata levels from `fup_var` and `occ_dy`. | `strata` (stratification variable name), `fup_var`, `occ_var`, `occ_dy`, `num_p_year` (default `100`), `count_multiple_events`, `riskdiff`, `ref_path`, `conf_level`, `riskdiff_setup`, `row_labels_adj`, `.stats`, `.formats`, `.labels`, `na_str`. |
| **`a_patyrs_j`** | `afun` / `cfun` | Tabulates total patient-years of exposure. Used independently or as an exposure summary row above EAIR rows. | `patyrs` (total patient-years of exposure). | Sum of `.var` across subjects in `source` (`alt_df` or `df`). | `source` (`"alt_df"` or `"df"`), `id` (subject ID), `ref_path` (blanked out in risk diff columns), `.stats`, `.formats`, `.labels`. |
| **`a_summarize_ex_j`** | `afun` | Tabulation for exposure tables (extent/duration of exposure in days or months, dose intensity), with descriptive summaries or ANCOVA/t-test comparisons. | `mean_sd`, `median`, `range`, `quantiles`, `total_subject_years` *(defaults)*; in difference columns: `diff_mean_est_ci`. | Continuous summary / sum of duration; converts units via `daysconv`. | `comp_btw_group` (enables difference column estimation), `ref_path`, `ancova` (if `TRUE`, uses ANCOVA model for between-group differences; if `FALSE`, uses 2-sample t-test), `interaction_y`, `interaction_item`, `conf_level`, `daysconv` (unit multiplier: `1` for days, `30.4375` for months), `variables`, `.stats`, `.formats`, `.labels`, `na_str`. |
| **`column_stats`** | `afun` | Statistics-in-columns analysis function. Presents summary statistics across column variables (e.g. AVAL, CHG, BASE in columns across visits in rows) with varying decimal precision. | `n`, `mean`, `sd`, `se`, `median`, `min`, `max`, `range`, `mean_sd`, `mean_se` (evaluated across specified `var_names`). | N/A (continuous variables summary). | `formats_var` (list column in dataset defining parameter-specific formats `fmt_d`), `exclude_visits` (visits excluded from BASE/CHG columns, default `"Baseline (DB)"`), `var_names` (variables analyzed, default `c("AVAL", "CHG", "BASE")`), `.formats`. |
| **`a_diff_means`** | `afun` | Computes point estimate and inferential statistics for the difference in population means between active treatment and reference group. | `diff_means_est_ci` *(default)*, `diff_means_n1`, `diff_means_n2`, `diff_means_est`, `diff_means_se`, `diff_means_est_se`, `diff_means_ci`. | N/A (continuous variables). | `ref_path` (reference group column path via `get_ref_info()`), `paired` (`TRUE`/`FALSE`), `paired_by` (subject ID variable for paired t-test), `conf_level` (via `...` to `safe_t_test()`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_summarize_aval_chg_diff_j`** | `afun` | 3-column analysis presentation (col 1 = n/N (%), col 2 = raw/baseline mean & CI, col 3 = change from baseline mean & CI, plus optional difference columns). Supports descriptive stats or ANCOVA. | Specified per column section: `col1`: `count_denom_frac`, `count_frac`, `count`; `col23`: `mean_ci_3d` (arithmetic or LS mean); `coldiff`: `mean_diff_with_ci` (t-test or ANCOVA LS mean diff). | `denom`: `"N"` (records in split) or `".N_col"` (column total count from `alt_counts_df`). | `ancova` (`TRUE` for ANCOVA LS means, `FALSE` for descriptive/t-test), `comp_btw_group`, `ref_path`, `multivars` (default `c("AVAL", "AVAL", "CHG")`), `method_combo` (`"contrasts"` or `"collapse"` for combined arms), `weights_combo`, `weights_emmeans`, `d` (decimal precision integer or column name for varying precision), `interaction_y`, `interaction_item`, `conf_level`, `variables`, `.stats`, `.formats`, `.formats_fun`, `format_na_str`. |
| **`a_summarize_ancova_j`** | `afun` | Comprehensive ANCOVA summary function supporting single and combined treatment groups via linear contrasts or model collapsing. | Descriptive: `n`, `sum`, `mean`, `sd`, `se`, `mean_sd`, `mean_se`, `mean_ci`, `mean_ci_3d`, `median`, `mad`, `median_ci`, `quantiles`, `range`, `geom_mean`, etc. ANCOVA: `n_fit`, `lsmean`, `lsmean_se`, `lsmean_ci`, `lsmean_diff`, `lsmean_diff_ci`, `lsmean_diff_with_ci`, `pval`. Defaults: `n`, `mean_sd`, `median`, `range`, `quantiles`, `lsmean_se`, `lsmean_ci`, `lsmean_diff_with_ci`, `pval`. | N/A (continuous variables). | `ref_path` (reference arm path), `variables` (list with `arm` and `covariates`), `method_combo` (`"contrasts"` for contrast derivation or `"collapse"` for pooled refit), `weights_combo` (contrast weights, e.g. `"equal"`), `weights_emmeans` (`"equal"`, `"proportional"`, `"counterfactual"`), `conf_level`, `interaction_y`, `interaction_item`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_summarize_mmrm`** | `afun` | Dynamic tabulation of Mixed Model for Repeated Measures (MMRM) results within tables split by treatment arms and visits. | `n`, `adj_mean_se`, `adj_mean_ci`, `adj_mean_est_ci`, `diff_mean_se`, `diff_mean_ci`, `diff_mean_est_ci`, `change`, `p_value`. | N/A (repeated measures model). | `variables` (list with `arm`, `visit`, `id`, `covariates`, `response`), `ref_levels` (named list with reference `visit` and reference `arm`), `cor_struct` (`"toeplitz"`, `"unstructured"`, `"ar1"`, etc.), `conf_level`, `alternative` (`"two.sided"`, `"less"`, `"greater"`), `show_relative` (`"reduction"`, `"increase"`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_summarize_mmrm_with_exclude`** | `afun` | Wrapper around `a_summarize_mmrm()` that excludes specified row split levels (e.g. baseline visit) from MMRM estimation. | Identical to `a_summarize_mmrm()`. | N/A. | `exclude_levels` (named list of row split levels to exclude), plus all parameters of `a_summarize_mmrm()`. |
| **`a_lsmeans`** | `afun` | Formatted analysis function for tabulating Least Squares Means and contrasts (from `fit_mmrm_j()` or `fit_ancova()`). | `n`, `adj_mean_se`, `adj_mean_ci`, `adj_mean_est_ci`, `diff_mean_se`, `diff_mean_ci`, `diff_mean_est_ci`, `change`, `p_value`. | N/A. | `ref_path` (reference arm path), `conf_level`, `alternative` (`"two.sided"`, `"less"`, `"greater"`), `show_relative`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_rbmi_lsmeans`** | `afun` | Tabulates pooled Least Squares Means, differences, and CIs from Reference-Based Multiple Imputation (RBMI) via Rubin's rules. | `adj_mean_se`, `adj_mean_ci`, `diff_mean_se`, `diff_mean_ci`, `change`, `p_value`, `additional_title_row`. | N/A (pooled multiple imputation results). | `ref_path` (reference arm column path), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_summary_j_with_exclude`** | `afun` | Dataframe-level wrapper around `tern::a_summary()` that excludes specified row split levels from producing numeric summaries. | All statistics from `tern::a_summary()`: `n`, `sum`, `mean`, `sd`, `se`, `mean_sd`, `mean_se`, `mean_ci`, `median`, `mad`, `median_ci`, `quantiles`, `iqr`, `range`, `min`, `max`, `median_range`, `cv`, `geom_mean`, `geom_sd`, `geom_mean_sd`, `geom_mean_ci`, `geom_cv`, `mean_diff_with_ci`. | N/A (continuous numeric variable summary). | `exclude_levels` (named list of split levels to exclude), `.var` (variable to extract and summarize), and all parameters of `tern::a_summary()` via `...`. |
| **`a_summary_subset`** | `afun` | Wrapper around `tern::a_summary()` that evaluates a logical filter expression (`filter_expr`) to subset `df` before summarization. Supports varying precision. | All statistics from `tern::a_summary()`. | N/A (continuous numeric variable summary). | `filter_expr` (single logical expression evaluated within `df`, e.g. `expression(AGE > 30)`), `formats_var` (varying decimal precision specification), and all `tern::a_summary()` parameters via `...`. |
| **`c_summary_subset_label`** | `cfun` | Content row function wrapper around `tern::a_summary()` with optional row subsetting (`filter_expr`) and custom label prepending. | All statistics from `tern::a_summary()`. | N/A. | `filter_expr`, `label` (prepended label string), `label_indent_mod`, and all `tern::a_summary()` parameters. |
| **`a_sum_ratio_j`** | `afun` | Computes sums and ratios of sums (e.g. cumulative dose received vs. planned, optionally de-duplicated by subject ID). | `sum` *(default)*, `sum_unique`, `ratio`, `ratio_unique`. | `denom_by`: name of column in `df` whose sum serves as denominator in `ratio` and `ratio_unique`. | `denom_by` (required for ratio stats), `id_var` (subject ID variable, required for `*_unique` stats), `.stats`, `.formats`, `.labels`, `.indent_mods`, `na_str`. |
| **`a_proportion_ci_logical`** | `afun` | Formatted analysis function computing proportion confidence intervals for a logical vector. | `prop_ci` (confidence interval in percent, returned as formatted `rcell`). | Derived from `nrow(.alt_df)`. | `conf_level` (default 0.95), `method` (all methods in `tern::s_proportion`: `"wald"`, `"clopper-pearson"`, `"wilson"`, `"wilsonc"`, `"strat_wilson"`, `"agresti-coull"`, `"jeffreys"`), `formats` (`list(prop_ci = ...)`). |
| **`a_proportion_ci_factor`** | `afun` | Formatted analysis function computing proportion confidence intervals for levels of a factor variable. | `prop_ci` (confidence interval in percent for each factor level). | Derived from `nrow(.alt_df)`. | Identical to `a_proportion_ci_logical()`. |
| **`a_cond_proportion_j`** | `afun` | Adaptive response proportion and CI estimation, automatically selecting between exact (Clopper-Pearson) and asymptotic (Wald) methods based on count thresholds. | `n_prop` (count of responders and proportion, `xx / xx (xx.x%)`), `prop_ci` (proportion CI with dynamic method label). | `denom`: `"n"` *(default, cell record count)* or `"N_row"` (total row split record count). | `method_scope` (`"cell"` for cell-level decision, `"row"` for row-level pooled decision), `num_limit` (numerator threshold triggering exact CI, default `0`), `denom_limit` (denominator threshold triggering exact CI, default `10`), `conf_level` (default 0.95), `long` (verbose method explanation in label), `na.rm`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_proportion_diff_j`** | `afun` | Estimates proportion difference (risk difference) and CI vs. reference group (`ref_path`) for a logical response variable. | `diff`, `diff_ci`, `diff_est_ci` (`"% Difference (conf_level CI)"`). | Responders out of total observations in group/cell. | `ref_path` (column path to reference group), `method` (`"wald"`, `"waldcc"`, `"cmh"`, `"ha"`, `"newcombe"`, `"newcombecc"`, `"strat_newcombe"`, `"strat_newcombecc"`, `"cmh_sato"`, `"cmh_mn"`, `"uncond_exact_diff"`), `variables = list(strata = ...)` (stratification factors), `conf_level`, `label`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_proportion_diff_mf`** | `afun` | Proportion difference with automated method selection based on the Mantel-Fleiss (MF) criterion: uses CMH with Sato variance when MF >= threshold, otherwise exact unconditional difference. | `diff`, `diff_ci`, `diff_est_ci`. | Responders out of total observations in group/cell. | `mf_method` (`"cmh"`, `"cmh_sato"`, `"cmh_mn"`), `mf_threshold` (threshold for asymptotic CMH vs. exact, default `5`), `exact_footnote` (`RefFootnote` added when exact method triggers), `val` (response level), `variables = list(strata = ...)`, `conf_level`, `na.rm`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_test_proportion_diff`** | `afun` | Hypothesis test for the difference between two proportions vs. reference group (`ref_path`). | `pval` (p-value of proportion difference test). | Responders out of total observations in comparison groups. | `ref_path` (reference arm path), `method` (`"chisq"`, `"cmh"`, `"cmh_wh"`, `"fisher"`, `"schouten"`), `variables = list(strata = ...)`, `alternative` (`"two.sided"`, `"less"`, `"greater"`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_test_proportion_diff_mf`** | `afun` | Test of proportion difference with method selection based on Mantel-Fleiss criterion: CMH test when MF >= threshold, otherwise Fisher's exact test. | `pval` (p-value). | Responders out of total observations in comparison groups. | `mf_threshold` (default `5`), `exact_footnote` (`RefFootnote` added for exact test), `val`, `variables = list(strata = ...)`, `alternative`, `na.rm`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_combo_prop_diff_pval_mf`** | `afun` | Combined analysis presenting proportions, treatment-vs-control risk differences, and p-values across distinct columns within the same row, with Mantel-Fleiss method selection. | Specified via named list: `prop`: `count_denom_fraction` *(default)*, `count_fraction`, `count`; `diff`: `diff_est_ci` *(default)*, `diff`, `diff_ci`; `pval`: `pval` *(default)*. | Responders out of total records in column/cell. | `colpaths` (named list mapping `"prop"`, `"diff"`, and `"pval"` to column paths), `trt_var`, `ctrl_group`, `val` (response level in factor `.var`), `mf_threshold` (default `5`), `exact_footnote`, `variables = list(strata = ...)`, `conf_level`, `alternative`, `.stats`, `.formats`. |
| **`a_odds_ratio_j`** | `afun` | Estimates Odds Ratio (OR), CI, and p-value vs. reference group. Supports unstratified logistic regression, conditional logistic regression, and CMH. | `or_ci` (OR estimate and CI: `est`, `lcl`, `ucl`), `pval` (p-value), `n_tot` (total sample size). | Total sample size across comparison groups (`n_tot`). | `method` (`"exact"`, `"approximate"`, `"efron"`, `"breslow"`, `"cmh"`), `ref_path`, `variables` (list with `arm` and `strata`), `conf_level`, `na_if_no_events`, `groups_list`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_relative_risk`** | `afun` | Estimates stratified Relative Risk (RR), CI, and p-value using the CMH method with Greenland & Robins (1985) variance estimation. | `rel_risk_ci` (RR estimate and CI: `est`, `lcl`, `ucl`), `pval` (p-value). | Responders out of total observations in comparison groups. | `ref_path`, `variables = list(strata = ...)` (stratification factors required), `conf_level`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_cmhrms_j`** | `afun` | Performs Cochran-Mantel-Haenszel (CMH) Row Mean Scores test comparing ordered categorical response levels between treatments and reference group. | `pval` (row mean scores test p-value). | N/A (hypothesis testing p-value). | `ref_path`, `variables` (list with `arm` and optional `strata`), `collapse_combo` (whether multiple arm levels in `df` are collapsed into `"Combined"`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_cmhrms_j_with_exclude`** | `afun` | Wrapper around `a_cmhrms_j()` that excludes specified row split levels from producing the CMH Row Mean Scores test. | `pval`. | N/A. | `exclude_levels` (named list of row split levels to exclude), plus all parameters of `a_cmhrms_j()`. |
| **`prop_table_afun`** | `afun` | Formatted analysis function for proportion tables with column splits prepared via `prop_split_fun()`. Shows counts, percentages, and cumulative percentages across sub-columns. | In `n` col: category counts (+ optional Total); in `percent` col: category percentages; in `cum_percent` col: cumulative percentages. | Total records in column slice (`length(x)`). | `add_total_level` (`TRUE`/`FALSE`), `formats` (named list for `n`, `percent`, `cum_percent`). |
| **`c_proportion_logical`** | `cfun` | Simple content function computing the count and proportion of `TRUE` values in a logical vector relative to column count. | `est_prop` (`c(num, num / denom)`). | `.N_col` (total column count). | `label_fstr` (`sprintf` string using `labelstr`), `format`. |
| **`a_event_free`** | `afun` | Time-point survival (event-free rate) analysis function. Extends `tern::s_surv_timepoint` with a combined 3D statistic (`event_free_ci`). | `event_free_ci` *(combined rate & CI)*, `pt_at_risk`, `event_free_rate`, `rate_se`, `rate_ci`. | N/A (Kaplan-Meier survival estimation at time point). | `time_point` (numeric milestone time), `time_unit` (e.g. `"month"`, `"week"`), `is_event` (event indicator variable), `percent` (`TRUE` for %, `FALSE` for 0-1), `control` (`tern::control_surv_timepoint()`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`tefos03_afun`** | `afun` *(internal)* | Internal analysis function used by `summarize_coxreg_multivar()` to extract multivariable Cox model coefficients, hazard ratios, CIs, and p-values. | Model fit col: `coef_se`, `pval`; Hazard ratio col: `hr_est`, `hr_ci`. | N/A (Cox proportional hazards regression). | `variables` (time, event, arm, covariates), `control` (`tern::control_coxreg()`), `formats`. |
| **`a_coxph_hr`** | `afun` *(deprecated)* | Deprecated Cox proportional hazards hazard ratio function. Replaced by `tern:::a_coxph_pairwise()` and `tern::coxph_pairwise()`. | `hr`, `hr_ci`, `pvalue`. | N/A. | Deprecated in Junco 0.1.7 (`lifecycle::deprecate_stop`). |
| **`a_kaplan_meier`** | `afun` *(deprecated)* | Deprecated Kaplan-Meier summary statistics function. Replaced by `tern::a_surv_time()`. | `quantiles_lower`, `median_ci_3d`, `quantiles_upper`, `range_with_cens_info`. | N/A. | Deprecated in Junco 0.1.7 (`lifecycle::deprecate_stop`). |
| **`response_by_var`** | `afun` | Derives `count_denom_fraction` (`xx / xx (xx.x%)`) for records with response flag `"Y"` in `resp_var` across levels of `.var`, with denominator based on non-missing records. | `count_denom_fraction` (`c(num, denom, num / denom)`). | Non-missing observations of `.var` in input `df`. | `resp_var` (response flag variable), `id` (subject ID, default `"USUBJID"`), `.format`. |
| **`resp01_acfun`** | `afun` / `cfun` | Dual analysis and content function for RESP01 response tables. Dispatches to counts/CIs in arm columns, and comparative stats (RR, OR, p-value) in Overall column. | In arm columns: `count_prop` (count & proportion), `prop_ci` (proportion CI); in Overall column: `comp_stat_ci` (RR or OR estimate and CI), `pval` (p-value). | Derived from `.alt_df` (`nrow(.alt_df)`). | `methods` (list specifying `comp_stat_ci`: `"rr"`, `"or_cmh"`, `"or_logistic"`; `pval`: `"fisher"`, `"chisq"`, or CMH; `prop_ci`: `"wald"`, `"clopper-pearson"`), `formats`, `include_comp` (factor levels included in comparisons), `arm`, `strata`, `conf_level`. |
| **`resp01_a_comp_stat_logical`** | `afun` | Computes comparative statistics (RR or OR) and p-values for logical response variables in RESP01 response table layouts. | `comp_stat_ci`, `pval`. | Total group sizes in `df`. | `methods`, `formats`, `arm`, `strata`, `conf_level`, `include` (`TRUE`/`FALSE`). |
| **`resp01_a_comp_stat_factor`** | `afun` | Computes comparative statistics (RR or OR) and p-values across factor levels in RESP01 response table layouts. | `comp_stat_ci`, `pval` per factor level in `include`. | Total group sizes in `df`. | `include` (character vector of factor levels), `methods`, `formats`, `arm`, `strata`, `conf_level`. |
| **`resp01_counts_cfun`** | `cfun` | Content function for subgroup sample size counts from `.alt_df` in `count_prop` columns for RESP01 response tables. | `count` (`nrow(.alt_df)`). | Subgroup sample count from `.alt_df`. | `label_fstr` (`sprintf` string using `labelstr`). |
| **`cmp_cfun`** | `cfun` | Content function for study compliance tables split across three columns: Expected visits (N), Received visits (n (%)), and Missing visits (n (%)). | Expected col: `n_exp`; Received col: `n_rec`, `rec_percent`; Missing col: `n_mis`, `mis_percent`. | Denominator for Received and Missing is derived dynamically from Expected visits count (`sum(df[[variables$expected]])`). | `variables` (list with column names for `expected`, `received`, `missing`), `formats` (list containing `count_percent` format). |
| **`lsmeans_wide_cfun`** | `cfun` | Content function for wide-format ANCOVA LS means table layouts, organizing estimates, MSE, df, contrasts, and p-values across multi-tiered columns. | Reference/Testing cols: `treatment`, `n`, `lsmean`, `se`; Variance col: `mse`, `df`; Comparison col: `lsmean` (difference in LS means), `se`, `ci`, `pval`. | N/A. | `ref_level`, `treatment_levels`, `pval_sided` (`"2"`, `"1"`, `"-1"`), `conf_level`, `formats`. |
| **`c_row_counts`** | `cfun` *(internal)* | Content function to count rows in current data split (`nrow(df)`). Wrapped by `summarize_row_counts(alt_counts = FALSE)`. | `count` (`nrow(df)`). | N/A. | `label_fstr` (`sprintf` string using `labelstr`, default `"%s"`). |
| **`c_row_counts_alt`** | `cfun` *(internal)* | Content function to count rows in alternative dataset (`nrow(.alt_df)`). Wrapped by `summarize_row_counts(alt_counts = TRUE)`. | `count` (`nrow(.alt_df)`). | N/A. | `label_fstr`. |
| **`ac_blank_line`** | `afun` / `cfun` | Formatted analysis and content function producing a blank row gap in table layouts with invisible NA string (`""`). | Single blank cell (`NA_real_` formatted as `""`). | N/A. | `labelstr` (row label string, default `""`). |

---

## Detailed Functional Breakdown

### 1. Frequency, Categorical Counts, and Multitiered Layouts

#### `a_freq_j` & `a_freq_j_with_exclude`
`a_freq_j` is the primary workhorse function in `junco` for adverse event (AE), medical history, concomitant medication, and demographic frequency tables.
- **Role**: Can be passed directly to `rtables::analyze()` (`afun`) or `rtables::summarize_row_groups()` (`cfun`).
- **Statistics**:
  - `count`: raw observation count.
  - `count_unique`: unique subject count (based on `id`, default `"USUBJID"`).
  - `count_unique_fraction`: unique count with percentage (`xx (xx.x%)`).
  - `count_unique_denom_fraction` *(default)*: unique count, denominator, and percentage (`xx / xx (xx.x%)`).
  - `unique_fraction`: unique percentage without count.
  - `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, `denom`: individual count / denominator components.
  - In risk difference columns (identified via column paths matching `ref_path` comparisons): frequency stats are blanked out, and replaced by the treatment difference point estimate and confidence interval.
- **Denominators**:
  - `"N_col"`: column-wide total (default).
  - `"n_df"`: number of records in the current cell data slice.
  - `"n_altdf"`: number of records/subjects in `.alt_df_full` (e.g. ADSL population).
  - `"N_colgroup"`: column group total (requires `colgroup`).
  - `"n_rowdf"`: total observations in the current row split.
  - `"n_parentdf"`: observations in parent row split.
  - `denom_by`: permits conditional subsetting of denominators (e.g. `denom_by = "SEX"` ensures female and male subgroup rows use sex-specific ADSL denominators).
- **Customization**:
  - `riskdiff = TRUE`: enables risk difference calculations.
  - `ref_path`: specifies the control group column hierarchy (e.g. `c("ARM", "Placebo")` or `c("colspan_trt", " ", "ARM", "Placebo")`).
  - `method`: risk difference method (`"wald"`, `"waldcc"`, `"cmh"`, `"ha"`, `"newcombe"`, `"newcombecc"`, `"strat_newcombe"`, `"strat_newcombecc"`, `"cmh_sato"`, `"cmh_mn"`, `"uncond_exact_diff"`).
  - `label_map`: mapping dataframe supporting 4 types of translation (single-variable, multi-variable, conditional on row split, or conditional multi-variable) to relabel factor levels cleanly.
  - `a_freq_j_with_exclude`: adds `exclude_levels` to selectively suppress table rows for specified split levels.

#### `a_freq_combos_j`
Designed for cycle-based or cumulative period table layouts (e.g., cumulative adverse events over time).
- Uses `combosdf` to establish cumulative column header counts (`.N_col`).
- Uses `filter_var` and `flag_var` to prevent overcounting across cumulative facets in cell numerators, while `do_not_filter` preserves cumulative counts in designated columns (like "All Cycles" or "Total").

#### `a_freq_subcol_j`
Designed for tables where certain sub-columns represent restricted subsets of events (e.g. "All AEs", "Drug-Related AEs", and "Serious AEs" within each treatment arm).
- Uses `subcol_split` to match column path tokens, and filters data to `subcol_var == subcol_val` only within matching sub-columns.

#### `a_freq_resp_var_j`
Summarizes binary `"Y"`/`"N"` response flags (e.g., `TRTEMFL`) stratified across categories of an analysis variable (e.g., `SEX` or `AGEGR1`), with optional risk differences against reference arm.

#### `a_maxlev`
Computes the maximum severity level of an ordered factor per patient (e.g., maximum CTCAE toxicity grade across all occurrences of an AE for each subject).
- In row group headers (`cfun = a_maxlev, any_level = TRUE`), it counts subjects experiencing "Any" grade (excluding `"Missing"` via `any_level_exclude`).
- In child rows (`afun = a_maxlev, any_level = FALSE`), it tabulates the distribution of each patient's worst severity.

#### `a_two_tier` & `a_three_tier`
Simulates hierarchical drill-down row nesting within a single `analyze()` statement without creating rigid structural table sub-splits.
- `a_two_tier()` creates a group summary row (using `grp_fun`, e.g. `a_freq_j`) and conditionally generates indented detail rows (using `detail_fun`) under specific levels defined in `drill_down_levs` (e.g., in patient disposition, showing overall discontinued count and drilling down into specific discontinuation reasons).
- `a_three_tier()` extends this nesting to three tiers: outer group -> mid-level (`inner_var1`) -> innermost detail (`inner_var2`).

---

### 2. Exposure, Patient-Years, and Incidence Rates

#### `a_eair100_j` & `a_eair_strat_j`
Calculates Exposure-Adjusted Incidence Rates (EAIR) per `num_p_year` person-years (default 100), along with rate differences and confidence intervals vs. control.
- **Formula**: `num_p_year * n_event / total_person_years`.
- **Time at Risk**: Subjects without events contribute `fup_var` (in years). Subjects experiencing events contribute time to first event (`occ_dy / 365.25`) based on `occ_var == "Y"`.
- **Statistics**: `eair`, `n_event`, `person_years`, `n_eair`, `eair_ci`, `eair_est_ci`, `eair_diff_est_ci`, `eair_diff_est`, `eair_diff_ci`.
- **Methods**:
  - `conf_type_eair`: `"exact"` (Poisson exact), `"normal"`, `"normal_log"`, `"byar"`.
  - `conf_type_diff`: `"wald"`, `"waldcc"`, `"mn"` (Miettinen-Nurminen score), `"scas"` (skewness-corrected asymptotic score).
- **Stratified Variant (`a_eair_strat_j`)**: Computes study-size weighted stratified EAIR across strata defined by `strata`.

#### `a_patyrs_j`
Summarizes total patient-years of exposure across subjects from `alt_df` or `df`. Often paired directly above `a_eair100_j` in AE summary tables.

#### `a_summarize_ex_j`
Tabulates extent and duration of study drug exposure (e.g. days or months on treatment, cumulative dose, dose intensity).
- Summarizes continuous distributions: `mean_sd`, `median`, `range`, `quantiles`, `total_subject_years`.
- Between-group comparison (`comp_btw_group = TRUE`) calculates difference in means (`diff_mean_est_ci`) via two-sample t-test (`ancova = FALSE`) or ANCOVA model (`ancova = TRUE`).
- Unit conversion: `daysconv = 1` for days, `30.4375` for months.

---

### 3. Continuous Variables, Difference in Means, ANCOVA, MMRM & Summaries

#### `column_stats`
Implements stats-in-columns layouts where summary statistics (Mean, SD, Median, Min, Max) form table columns while study visits or parameters form table rows.
- Paired with `stats_in_cols_setup()`.
- Supports dynamic decimal precision via `formats_var` pointing to dataset list-columns (`fmt_d`).

#### `a_diff_means`
Estimates difference in population means between active treatment and control (`ref_path`).
- Supports both independent two-sample t-tests and paired t-tests (`paired = TRUE`, matched on `paired_by`).
- Returns sample sizes (`diff_means_n1`, `diff_means_n2`), point estimate (`diff_means_est`), SE (`diff_means_se`), CI (`diff_means_ci`), and 3D combo (`diff_means_est_ci`).

#### `a_summarize_aval_chg_diff_j`
Creates standard 3-column clinical trial presentations:
1. Column 1: Baseline or visit sample size and fraction (`n/N (%)`).
2. Column 2: Raw/baseline mean and 95% CI.
3. Column 3: Change from baseline mean and 95% CI.
4. Difference Columns: Difference in mean change vs. reference arm with 95% CI.
- When `ancova = FALSE`, uses arithmetic means and two-sample t-tests.
- When `ancova = TRUE`, uses linear ANCOVA models with LS means and contrasts.
- Supports combination treatment arms via `method_combo = "contrasts"` or `"collapse"`.
- Supports varying decimal precision via parameter `d`.

#### `a_summarize_ancova_j`
Comprehensive ANCOVA layout function supporting single and pooled/combination treatment groups.
- Model: `CHG ~ TRT + BASE + Covariates`.
- Combined arms: Handled via `method_combo = "contrasts"` (deriving estimates via linear contrasts from the full model) or `"collapse"`.
- Weighting: `weights_combo` (`"equal"`, etc.) and `weights_emmeans` (`"counterfactual"`, `"equal"`, `"proportional"`).
- Produces descriptive statistics as well as model statistics: `n_fit`, `lsmean`, `lsmean_se`, `lsmean_ci`, `lsmean_diff`, `lsmean_diff_with_ci`, `pval`.

#### `a_summarize_mmrm` & `a_summarize_mmrm_with_exclude`
Dynamic tabulation of MMRM repeated measures models across visits and treatment arms.
- Automatically handles reference visits (baseline, producing empty/null cells) and reference arms (producing LS means without contrasts).
- Extracts LS means, SEs, CIs, differences, and p-values (`p_value`).
- `a_summarize_mmrm_with_exclude` suppresses output for excluded visits/levels.

#### `a_lsmeans` & `a_rbmi_lsmeans`
- `a_lsmeans`: Formatted analysis function for models fitted via `fit_mmrm_j()` or `fit_ancova()`, supporting flexible `ref_path` and one-sided or two-sided p-values (`alternative`).
- `a_rbmi_lsmeans`: Formatted analysis function for Reference-Based Multiple Imputation (RBMI) results, tabulating pooled LS means and contrasts via Rubin's rules.

#### `a_summary_j_with_exclude` & `a_summary_subset` & `c_summary_subset_label`
Wrappers around `tern::a_summary()`:
- `a_summary_j_with_exclude`: Suppresses output for specified row split levels via `exclude_levels`.
- `a_summary_subset`: Subsets data frame rows using a logical `filter_expr` before summarizing `.var`, with `formats_var` support for varying precision.
- `c_summary_subset_label`: Content row function that applies `filter_expr` and prepends a custom label cell.

#### `a_sum_ratio_j`
Computes sums and ratios of sums (e.g., cumulative dose taken divided by planned dose):
- `"sum"`, `"sum_unique"`, `"ratio"`, `"ratio_unique"`.
- Denominator column specified via `denom_by`; unique de-duplication per subject specified via `id_var`.

---

### 4. Proportions, Confidence Intervals, Risk Difference, Odds Ratio, Relative Risk & CMH

#### `a_proportion_ci_logical` & `a_proportion_ci_factor`
Formatted analysis functions returning proportion confidence intervals (`prop_ci`) as formatted `rcell`s.
- Denominator is `nrow(.alt_df)`.
- Supports Clopper-Pearson exact, Wald, Wilson, Agresti-Coull, and Jeffreys methods.

#### `a_cond_proportion_j`
Adaptive response rate analysis function. Automatically selects between exact (Clopper-Pearson) and asymptotic (Wald) confidence intervals:
- Triggers exact CI if responders <= `num_limit` (default 0), non-responders <= `num_limit`, or denominator < `denom_limit` (default 10).
- Decision scope can be evaluated per cell (`method_scope = "cell"`) or row-wide (`method_scope = "row"`).
- Denominator can be within-cell count (`denom = "n"`) or row total (`denom = "N_row"`).

#### `a_proportion_diff_j`
Estimates proportion differences between treatment arms and control (`ref_path`).
- Supports unstratified and stratified risk difference methods (Wald, CMH Sato, Newcombe, Haenszel-Anderson, unconditional exact).
- Formats 3D point estimate and CI with short junco label `"% Difference (conf_level CI)"`.

#### `a_proportion_diff_mf` & `a_test_proportion_diff_mf` & `a_combo_prop_diff_pval_mf`
Implements the Mantel-Fleiss (MF) criterion for risk differences and hypothesis tests:
- When MF criterion >= `mf_threshold` (default 5), uses asymptotic CMH with Sato variance.
- When MF criterion < `mf_threshold`, switches to exact methods (unconditional exact difference for estimation; Fisher's exact test for p-value) and attaches `exact_footnote` (`RefFootnote`).
- `a_combo_prop_diff_pval_mf` combines proportions, risk differences, and p-values across distinct column paths (`colpaths`) within the same table row.

#### `a_odds_ratio_j`
Estimates Odds Ratios (OR), CIs, and p-values vs. control (`ref_path`):
- Supports standard/conditional logistic regression (`glm`/`clogit`) and Cochran-Mantel-Haenszel (`method = "cmh"`).
- Returns `or_ci` (point estimate, LCL, UCL), `pval`, and total sample size `n_tot`.

#### `a_relative_risk`
Estimates stratified Relative Risk (RR), CI, and p-value using the CMH method with Greenland & Robins (1985) variance.

#### `a_cmhrms_j` & `a_cmhrms_j_with_exclude`
Calculates p-values from the Cochran-Mantel-Haenszel Row Mean Scores test for ordered categorical responses across treatment groups and strata.
- `collapse_combo = TRUE` pools combination arm levels into `"Combined"`.
- `a_cmhrms_j_with_exclude` suppresses calculation for excluded split levels.

#### `prop_table_afun` & `c_proportion_logical`
- `prop_table_afun`: Analysis function for proportion tables with column facets created by `prop_split_fun()`. Displays counts in column `n`, percentages in column `percent`, and cumulative percentages in column `cum_percent`.
- `c_proportion_logical`: Content function computing count and proportion of `TRUE` in a logical vector relative to column count `.N_col`.

---

### 5. Time-to-Event and Survival Analysis

#### `a_event_free`
Milestone survival (event-free rate) analysis function. Extends `tern::s_surv_timepoint` with the combined 3D statistic `event_free_ci` (`xx.xx (xx.xx, xx.xx)`).
- Evaluates survival at milestone `time_point` (with `time_unit`).
- Identifies events via `is_event` (`1`/`TRUE` = event, `0`/`FALSE` = censored).

#### `tefos03_afun`
Internal analysis function used by `summarize_coxreg_multivar()` for multivariate Cox proportional hazards models.
- Populates model fit columns with coefficients and standard errors (`coef_se`) and p-values (`pval`).
- Populates hazard ratio columns with point estimates (`hr_est`) and confidence intervals (`hr_ci`).

#### Deprecated Functions: `a_coxph_hr` & `a_kaplan_meier`
Both functions were deprecated in `junco` version 0.1.7 and throw informative deprecation errors directing users to:
- `a_coxph_hr` -> `tern:::a_coxph_pairwise()` and `tern::coxph_pairwise()`.
- `a_kaplan_meier` -> `tern::a_surv_time()`.

---

### 6. Specialized Layouts: Compliance, Response Tables, LS Means Wide, Row Counts & Blank Lines

#### `resp01_acfun` & `resp01_counts_cfun`
Dedicated functions for response tables following the RESP01 layout:
- `resp01_split_fun_fct` splits treatment arm columns into `count_prop` (n (%)) and `prop_ci` (CI for %), while the Overall column displays `comp_stat_ci` (Relative Risk or Odds Ratio) and `pval`.
- `resp01_acfun` dispatches computation appropriately depending on column facet and data type (logical vs. factor).
- `resp01_counts_cfun` outputs sample size counts in content rows.

#### `cmp_cfun`
Content function for clinical trial study compliance tables:
- Operates on a 3-column split: Expected visits, Received visits, and Missing visits.
- Denominator for Received and Missing visit percentages is derived dynamically from the Expected visit count.

#### `lsmeans_wide_cfun`
Content function for wide-format ANCOVA layouts:
- Organizes LS means across multi-level column splits: Reference Group, Testing Group, optional Variance (MSE, df), and Treatment Difference (estimate, SE, CI, p-value).

#### `c_row_counts` & `c_row_counts_alt`
Internal content functions wrapped by `summarize_row_counts()`:
- `c_row_counts`: Counts rows in the primary data split (`df`).
- `c_row_counts_alt`: Counts rows in the alternative dataset (`.alt_df`, e.g. ADSL).

#### `ac_blank_line`
Dual-purpose analysis and content function that inserts an empty blank row into the table layout with empty formatting string (`""`), providing clean vertical spacing between table sections.

---

## Vignette Cross-References and Best Practice Patterns

1. **`junco.Rmd` (Tabulation Overview)**:
   - Illustrates end-to-end AE tabulation using `a_freq_j` across multiple `analyze()` and `summarize_row_groups()` calls.
   - Contrasts `junco`'s unified `a_freq_j` approach with `tern`'s separate `summarize_occurrences()` and `count_occurrences()` functions.
   - Highlights subgroup table construction using `denom_by = "SEX"`.
   - Explains 3-column laboratory/vital signs tables using `a_summarize_aval_chg_diff_j`.

2. **`ancova_combined.Rmd` (ANCOVA with Combined Groups)**:
   - Demonstrates pooling active treatment arms into virtual combination columns using `split_cols_by(split_fun = add_combo_levels(combodf))`.
   - Shows how `a_summarize_ancova_j` derives estimates and contrasts for combined columns via linear contrasts (`method_combo = "contrasts"`).

3. **`get_ref_info.Rmd` (Reference Group Handling)**:
   - Explains the `ref_path` alternating split variable/value convention (e.g. `c("colspan_trt", " ", "ARM", "Placebo")`).
   - Details how `get_ref_info()` allows Junco afuns to reliably locate reference/control groups under complex spanning headers and multivariable column splits where standard `rtables` `.ref_group` injection fails.

4. **`standard_column_structures.Rmd`**:
   - Demonstrates `grouped_cols_w_diffs()` for pairing active-vs-comparator spanning headers with difference columns.
   - Highlights the clean separation of concerns: `grouped_cols_w_diffs()` establishes column layout, while the selected `afun` (`a_freq_j`, `a_diff_means`, `a_eair100_j`, etc.) populates the difference calculations.

5. **`formats_d.Rmd` & `formats_significant_figures.Rmd`**:
   - Details varying decimal precision using `column_stats` and `formats_var` mapped to parameter-specific format lists (`fmt_d`).
