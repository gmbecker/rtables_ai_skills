---
name: rtables-select-afuns-cfuns
description: Evaluate table shell statistical requirements and select/configure existing group summary content functions (`cfun` in `summarize_row_groups`) and analysis functions (`afun` in `analyze`) using `junco`, `tern`, and `rtables`. Activate this skill whenever determining which analysis or content function to use for a table, choosing count types (unique subjects vs. events), configuring percentage denominators (`N_col`, `n_altdf`, `denom_by`, `n_df`, `n_rowdf`, `n_parentdf`), handling column-facet, row-facet, or dual conditionality (risk difference columns, baseline visit exclusions, multi-variable columns, tiered drill-downs), setting up reference group paths (`ref_path`), or applying dynamic decimal precision (`formats_d`, `formats_var`). Does not cover writing custom analysis functions from scratch.
---

# Evaluating and Selecting Group Summary (`cfun`) and Analysis (`afun`) Functions in `rtables` & `junco`

This skill provides a systematic framework and complete reference catalog for **evaluating statistical requirements** from a target table shell and **selecting + configuring pre-built analysis functions (`afun`) and content functions (`cfun`)** from `junco`, `tern`, and `rtables`.

> [!IMPORTANT]
> **Scope Boundary & Strategy**
> - **Use this skill** to diagnose what a table's cells represent and to select/parameterize existing production functions passed to `rtables::analyze(..., afun = ..., extra_args = ...)` and `rtables::summarize_row_groups(..., cfun = ..., extra_args = ...)`.
> - **Prefer `junco` `a_*` / `c_*` / `ac_*` functions** whenever `library(junco)` is available: they plug directly into standard `rtables` layout pipelines (`analyze()` and `summarize_row_groups()`), natively resolve complex spanning headers via `ref_path`, and automatically handle column/row conditionality.
> - **After selecting a function**, escalate to its **expert usage skill** via the Section 3.1 expert skill index for detailed parameterization, gotchas, and worked examples.
> - **When no existing function satisfies the requirement**, escalate to `[custom_afuns.md](./custom_afuns.md)` for extending/combining existing functions (`make_afun()`, `c(rvs1, rvs2)`, `get_ref_info()`) or writing bespoke custom analysis functions (`function(x, df, .var, .spl_context, ...) { in_rows(...) }`). Always verify whether an existing `junco` or `tern` function (with appropriate `extra_args`) satisfies the requirement before escalating.
> - **A missing section-header label is not a gap in the afun**: If an existing function produces the correct cells and only a leading label row above the block is missing, use the function directly with `var_labels = "...", show_labels = "visible"` on the `analyze()` call. Do not escalate to `[custom_afuns.md](./custom_afuns.md)` or write a custom wrapper solely to add a header row.

---

## 1. Key Selection Constraints

> **RULE: IF the layout includes difference columns (`grouped_cols_w_diffs()` or manual risk-difference branches), THEN select a `junco` function that supports `riskdiff`/`ref_path`.** No `tern` function supports junco-style column conditionality. When the required calculation is only available in `tern`, escalate to `[custom_afuns.md](./custom_afuns.md)` to wrap it in a column-dispatch wrapper (Pattern 2A) or `get_ref_info()` bridge (Tier 4).

- **`var` in `summarize_row_groups()`**: Must be specified when the selected `cfun` accepts `.var` as an argument. The default row summary behavior does not require it.
- **Dual-Purpose `afun`/`cfun` Functions**: These `junco` functions work as both `afun` in `analyze()` and `cfun` in `summarize_row_groups()`: `a_freq_j`, `a_freq_j_with_exclude`, `a_freq_combos_j`, `a_freq_subcol_j`, `a_freq_resp_var_j`, `a_maxlev`, `a_eair100_j`, `a_eair_strat_j`, `a_patyrs_j`, `resp01_acfun`, `ac_blank_line`.

---

## 2. Phase 1: Evaluating Requirements ("Who Are Your Numbers and What Do They Do?")

Before selecting an `afun` or `cfun`, decompose the desired table shell into **Fundamental Requirements** and **Specific Requirements**.

### 2.1 The Three Fundamental Questions

Every number in a table cell is governed by three fundamental properties that determine how the table is interpreted:

1. **What TYPE of observation is being counted or analyzed?** (The unit of analysis)
2. **What (SUB)POPULATION are we analyzing within or comparing against?** (The denominator / reference group)
3. **Is the behavior CONDITIONAL on the column facet, row facet, or both?** (Facet conditionality)

Only after answering these three fundamental questions should you configure **Specific Requirements** (choice of confidence interval or p-value method, handling of missing/censored values, level exclusions, dynamic decimal precision).

---

### 2.2 Fundamental Question 1: Type of Observation (Unit of Counting / Analysis)

Even a seemingly simple frequency count in an event dataset (such as `ADAE` or `ADCM`) can mean fundamentally different things:

| Unit of Analysis | Meaning in Clinical Data | `junco` / `tern` Mechanism |
| :--- | :--- | :--- |
| **Unique Subjects (Patients)** | Number of distinct subjects (`USUBJID`) experiencing >=1 event in that cell (`length(unique(x))`). Standard for AE incidence tables. | `a_freq_j` with `.stats = "count_unique"`, `"count_unique_fraction"`, or `"count_unique_denom_fraction"` *(default)* |
| **Total Events (Occurrences)** | Total number of event records in that cell (`length(x)`), where a single subject may contribute multiple events. | `a_freq_j` with `.stats = "count"`; or `a_eair100_j(..., count_multiple_events = TRUE)` |
| **Subject's Worst / Maximum Severity** | Each subject is counted **once** per category at their maximum ordered factor level (e.g., highest CTCAE toxicity grade `AETOXGR` or severity `AESEV`). | `a_maxlev` (`any_level = TRUE` in `cfun` for "Any Grade" header; `any_level = FALSE` in `afun` for grade breakdown) |
| **Person-Years / Time at Risk** | Cumulative follow-up duration across subjects (either total exposure time or time censored at first event occurrence `occ_dy / 365.25`). | `a_patyrs_j` (total exposure) and `a_eair100_j` / `a_eair_strat_j` (events per 100 person-years) |
| **Cumulative Sums & Ratios of Sums** | Sum of a continuous measure (e.g., actual dose received) and its ratio to another summed measure (e.g., planned dose), across all records or unique subjects. | `a_sum_ratio_j` with `.stats = c("sum", "sum_unique", "ratio", "ratio_unique")` and `denom_by` |
| **Continuous Distribution / Model Estimate** | Arithmetic mean, SD, SE, median, IQR, min/max, geometric mean, or model-based LS means (ANCOVA, MMRM, RBMI) across observations in the facet. | `tern::a_summary`, `a_summary_subset`, `column_stats`, `a_summarize_aval_chg_diff_j`, `a_summarize_ancova_j`, `a_summarize_mmrm`, `a_lsmeans`, `a_rbmi_lsmeans` |

> [!WARNING]
> **Never Confuse Unique Subjects with Total Events**
> In repeated-measures or event-level datasets (`ADAE`, `ADCM`, `ADMH`), a format shell showing `xx (xx.x%)` almost always requires **unique subjects** (`count_unique_fraction`), whereas a row labeled `"Total number of events"` requires **event counts** (`count`). Check the shell's row labels and footnotes carefully.

> [!WARNING]
> **Never Assume Row Counts Equal Subject Counts**
> In any dataset — not just event-level datasets like `ADAE` — a format shell showing `N` or `xx` as a count may require **unique subjects** rather than **total records**. Before using a row-counting mechanism (`summarize_row_groups(format = "xx")`, `.stats = "n_df"`, `nrow(df)`) as a subject count, verify that the dataset has exactly one row per subject within each relevant facet. Repeated-measures datasets (`ADLB`, `ADVS`, `ADEX`) can have multiple rows per subject per visit; visit-level datasets may have multiple assessments per visit. When in doubt, use a subject-deduplicating mechanism.

---

### 2.3 Fundamental Question 2: Denominators & Target (Sub)Populations

When a table shell displays percentages (`xx (xx.x%)` or `xx / xx (xx.x%)`), the percentage is meaningless unless you identify **which (sub)population is the denominator**.

In `rtables`, percentages can be calculated relative to five distinct structural scopes—and each scope can count either **subjects from `alt_counts_df` (`ADSL`)** or **records/subjects in the primary dataset (`df`)**:

| Denominator Scope | Meaning | When It Is Used | `a_freq_j` Parameterization | Other `junco` Functions |
| :--- | :--- | :--- | :--- | :--- |
| **1. Column Population (`.N_col` / `alt_counts_df`)** | Total subjects (or records) in the current column (e.g., Treatment Arm N). | Standard demographics, AE incidence, concomitant meds where `%` is out of all treated subjects in that arm. | `denom = "N_col"` *(default)* or `denom = "n_altdf"` (requires `alt_counts_df` in `build_table()`) | `a_proportion_ci_logical`, `a_proportion_ci_factor`, `c_proportion_logical`, `resp01_acfun` |
| **2. Conditioned Subgroup within Column (`alt_counts_df` subset)** | Total subjects in `alt_counts_df` belonging to **both** the current column arm **and** an ancestor row subgroup (e.g., Female subjects in Arm A). | Subgroup tables (e.g., AEs split by `SEX` or `AGEGR1` on rows) where `%` is out of all Females in Arm A, not all subjects in Arm A. | `denom = "n_altdf"` combined with `denom_by = "SEX"` (or vector of row-split vars) | — |
| **3. Individual Cell Facet (`df` slice)** | Number of non-missing observations/subjects inside the current `(row, col)` cell itself. | Responder rates within a visit/parameter facet (`n/N (%)` where `N` is subjects with an assessment at that visit), or conditional proportions. | `denom = "n_df"` | `a_cond_proportion_j(denom = "n")`, `response_by_var`, `a_summarize_aval_chg_diff_j(denom = "N")` |
| **4. Row Facet Across All Columns (`.df_row`)** | Total observations across **all** columns for the current row split level. | Cross-tabulations where horizontal row percentages sum to 100% across columns. | `denom = "n_rowdf"` | `a_cond_proportion_j(denom = "N_row")` |
| **5. Parent / Ancestor Row Facet** | Total observations in the parent row split (`n_parentdf`) or column group (`N_colgroup`). | Hierarchical breakdowns where child percentages are expressed as a share of the parent category count. | `denom = "n_parentdf"` or `denom = "N_colgroup"` | `cmp_cfun` (Received/Missing visits divided by Expected visits in that row) |

> [!IMPORTANT]
> **RULE: IF `df` is not a one-row-per-subject dataset, THEN pass the subject-count dataset as `alt_counts_df` to `build_table()` and set `denom = "n_altdf"` (or `denom = "N_col"`) for subject-population-relative percentages. IF row splits additionally partition subjects into subgroups, THEN also set `denom_by = "<row_subgrp_var>"`.**
>
> **Why `alt_counts_df` Matters for Denominators**
> Not every subject in a clinical trial suffers an adverse event or takes a concomitant medication. Therefore, `ADAE` and `ADCM` only contain rows for subjects who had an event.
> - If you compute denominators from `df` (`ADAE`), the denominator will only count subjects who had at least one AE (or worse, total AE records).
> - Always pass the subject-count dataset as `alt_counts_df` to `build_table(lyt, df, alt_counts_df = ...)` and set `denom = "n_altdf"` (or `denom = "N_col"`) when percentages are relative to the full safety/ITT population, and add `denom_by = "<row_subgrp_var>"` when row splits partition the subject population into subgroups.

---

### 2.4 Fundamental Question 3: Facet Conditionality (Column, Row, and Dual)

In `rtables`, a single `analyze()` or `summarize_row_groups()` call is evaluated across every leaf column and within every active row facet. Many production tables require **conditional behavior** depending on which facet is currently being populated.

#### A. Conditionality Based on Column Facet (Heterogeneous Column Structures)
Whenever a table layout has **non-nested (sibling) column splits** or **multi-variable column splits**, different columns in the *same row* display fundamentally different statistics:
- **Main Arm Columns vs. Risk / Mean Difference Columns**: Main columns show `n (%)` or `Mean (SD)`, whereas sibling comparison columns show `Risk Difference (95% CI)` or `LS Mean Diff (95% CI)` (and blank out raw counts or baseline rows).
- **Multi-Variable Columns (`split_cols_by_multivar`)**: E.g., Column 1 shows `n/N (%)`, Column 2 shows `Mean (95% CI)` of `AVAL`, Column 3 shows `Mean Change (95% CI)` of `CHG`, and Column 4 shows `Difference in Mean Change (95% CI)`.
- **Stats-in-Columns**: Columns represent individual statistics (`N`, `Mean`, `SD`, `SE`, `Med`, `Min`, `Max`) nested under `AVAL` and `CHG`.
- **Sub-Column Event Filtering**: E.g., each arm is split into `"All AEs"`, `"Drug-Related AEs"`, and `"Serious AEs"`, requiring the `afun`/`cfun` to apply an extra filter (`AEREL == "RELATED"`, `AESER == "Y"`) only in specific sub-columns.
- **Cycle / Cumulative Period Columns**: Individual cycle columns filter to avoid overcounting while cumulative/total columns retain pooled counts.

> **RULE: IF the column layout has non-nested (sibling) column splits or multi-variable column splits, THEN the `afun`/`cfun` must be column-aware.** All `tern` functions are column-agnostic by default and require wrapping before use in such layouts.

**Selection Rule**: Never use a column-agnostic `afun` in a heterogeneous column layout. Select a `junco` function built for that exact column topology:
- Spanning arms + Risk/Rate/Mean Difference columns → `a_freq_j(..., riskdiff = TRUE, ref_path = ...)`, `a_eair100_j(..., riskdiff = TRUE, ref_path = ...)`, `a_diff_means(..., ref_path = ...)`, `a_summarize_ex_j(..., comp_btw_group = TRUE, ref_path = ...)`.
- 3-column `[n/N (%), Mean, Mean CFB]` + optional Diff column → `a_summarize_aval_chg_diff_j`.
- Stats-in-columns (`N`, `Mean`, `SD`, `SE`, `Med`, `Min`, `Max`) → `column_stats` (paired with `stats_in_cols_setup()`).
- Sub-column conditional event filtering → `a_freq_subcol_j`.
- Cumulative / period combo columns → `a_freq_combos_j`.
- Proportions + Risk Diff + p-value in separate columns on the same row → `a_combo_prop_diff_pval_mf`.
- `RESP01` arm proportions + Overall column RR/OR/p-value → `resp01_acfun`.
- Expected / Received / Missing compliance columns → `cmp_cfun`.
- Wide-format ANCOVA columns (Ref, Test, Variance, Comparison) → `lsmeans_wide_cfun`.

#### B. Conditionality Based on Row Facet
> **RULE: IF the `afun`/`cfun` must suppress output, change statistics, or vary precision depending on the current row split level, THEN select a `junco` row-conditional function (see list below).**

Purely row-conditional `afun`s produce different numbers of rows, different statistics, or suppress output entirely depending on the current row split level:
- **Excluding Baseline/Screening Visits from Change-from-Baseline or Modeling**: In longitudinal visit tables (`split_rows_by("AVISIT")`), `BASELINE` has no Change from Baseline or MMRM contrast.
- **Suppressing Specific Row Split Levels**: E.g., suppressing summary statistics or p-values for a `"Missing"` or `"Not Applicable"` category.
- **Selective Hierarchical Drill-Down**: Expanding detail rows (e.g., discontinuation reasons) *only* under specific parent levels (e.g., `"DISCONTINUED"`), while leaving other parent levels (e.g., `"COMPLETED"`, `"ONGOING"`) as single summary rows.
- **Parameter-Specific Decimal Precision**: Formatting `ALT` to 2 decimals, `CRP` to 3 decimals, and `IGA` to 1 decimal based on the current `PARAMCD` row facet.

**Selection Rule**: Use `junco`'s row-conditional mechanisms:
- Exclude specific row split levels from output → `a_freq_j_with_exclude`, `a_summary_j_with_exclude`, `a_summarize_mmrm_with_exclude`, `a_cmhrms_j_with_exclude` (via `exclude_levels = list(VAR = c("LEV1", ...))`).
- Exclude baseline visits from `BASE`/`CHG` columns → `column_stats(exclude_visits = c("BASELINE", "SCREENING"))` or `a_summarize_mmrm(ref_levels = list(visit = "BASELINE", arm = ...))`.
- Selective 2-tier or 3-tier drill-down under specific levels → `a_two_tier(inner_var = ..., drill_down_levs = ...)` or `a_three_tier(...)`.
- Vary decimal precision by row parameter → `formats_var = "fmt_d"` in `analyze()` / `column_stats` / `a_summary_subset`, or `d` in `a_summarize_aval_chg_diff_j`.

#### C. Dual Conditionality (Simultaneous Row × Column Conditionality)
In many clinical tables, column conditionality *itself* depends on the current row facet:
- **Example 1 (Longitudinal Visits + Difference Columns)**: At `SCREENING` and `BASELINE`, the `Mean Change From Baseline` column and the `Difference in Mean Change` columns must be completely blank, whereas at `WEEK 1` through `WEEK 5` they are populated.
  - *Handled automatically by*: `a_summarize_aval_chg_diff_j` (when `advs[advs$ABLFL == "Y", "CHG"] <- NA`), `column_stats(exclude_visits = ...)`, and `a_summarize_mmrm`.
- **Example 2 (Subgroup Header Rows vs. Event Detail Rows with Risk Difference Columns)**: In a subgroup AE table (`split_rows_by("SEX")`), the `Gender: F` group summary row displays the subgroup sample size `n_altdf` in the treatment arm columns and **blanks out** the Risk Difference columns (`riskdiff = FALSE`), while the nested AE rows display `n/N (%)` in the arm columns and **populate** the Risk Difference columns (`riskdiff = TRUE`).
  - *Handled by*: Calling `summarize_row_groups("SEX", cfun = a_freq_j, extra_args = list(.stats = "n_altdf", denom = "n_altdf", denom_by = "SEX", riskdiff = FALSE))` followed by `analyze("AEDECOD", afun = a_freq_j, extra_args = list(.stats = "count_unique_denom_fraction", denom = "n_altdf", denom_by = "SEX", riskdiff = TRUE, ref_path = ref_path))`.
- **Example 3 (Total Patient-Years Row above EAIR Rows)**: The `Total patient-years` row (`a_patyrs_j`) populates the treatment arm columns and automatically blanks out the Risk Difference columns (`ref_path`), while the subsequent `a_eair100_j` rows populate both the arm columns (with EAIR) and the Risk Difference columns (with rate differences + CI).

---

## 3. Phase 2: Quick Selection Decision Matrix

Use this decision matrix to map your table shell's analytical requirement to the exact `junco` or `tern` function:

| Analytical Task / Table Shell Pattern | Recommended `afun` / `cfun` | Role | Diff Col | Key `extra_args` / Setup |
| :--- | :--- | :--- | :--- | :--- |
| **Single-Row Unique Subject Count (e.g., "Subjects with ≥1 AE", "Evaluable Subjects")** | **`junco::a_freq_j`** | `afun` & `cfun` | Yes | Analyze a binary flag variable (e.g., `"TRTEMFL"`) with `val = "Y"`, `label = "..."`. ⚠️ Never pass `USUBJID` or other high-cardinality variable as `.var` |
| **Unique Subject Counts & `%` by Categorical Variable Levels (e.g., AE, concomitant medication, medical history, demographic tables) with optional Risk Diff** | **`junco::a_freq_j`** | `afun` & `cfun` | Yes | `.stats = "count_unique_fraction"` (or `"count_unique_denom_fraction"`), `denom = "n_altdf"`, `riskdiff = TRUE/FALSE`, `ref_path`, `method`, `val`, `label_map` |
| **Frequency Counts & `%` Suppressing Specific Row Split Levels** | **`junco::a_freq_j_with_exclude`** | `afun` & `cfun` | Yes | `exclude_levels = list(VAR = "LevelToSkip")` + all `a_freq_j` args |
| **Subgroup Header Row Showing Sample Size Only (`Gender: F   79   66   77`)** | **`junco::a_freq_j`** | `cfun` | Yes | `.stats = "n_altdf"`, `denom = "n_altdf"`, `denom_by = "SEX"`, `riskdiff = FALSE`, `label_fstr = "Gender: %s"` |
| **Subgroup Frequency Rows (`n/N (%)` with Subgroup Denominator)** | **`junco::a_freq_j`** | `afun` & `cfun` | Yes | `.stats = "count_unique_denom_fraction"`, `denom = "n_altdf"`, `denom_by = "SEX"`, `riskdiff = TRUE`, `ref_path` |
| **Cumulative / Cycle Period Frequency Columns** | **`junco::a_freq_combos_j`** | `afun` & `cfun` | Yes | `combosdf`, `filter_var`, `flag_var`, `do_not_filter` |
| **Sub-Columns Filtering by Binary Flags within Each Arm (e.g., Serious / Related / All AEs, on-treatment / off-treatment)** | **`junco::a_freq_subcol_j`** | `afun` & `cfun` | Yes | `subcol_split`, `subcol_var`, `subcol_val` |
| **Binary Flag (`"Y"`/`"N"`) Stratified by Categorical Variable + Risk Diff** | **`junco::a_freq_resp_var_j`** | `afun` & `cfun` | Yes | `resp_var = "TRTEMFL"`, `riskdiff = TRUE`, `ref_path`, `method` |
| **Unique Subject Counts by Per-Subject Maximum of Ordinal Variable (e.g., worst severity grade, highest toxicity grade)** | **`junco::a_maxlev`** | `afun` & `cfun` | **No** | In `cfun`: `any_level = TRUE`, `any_level_exclude = "Missing"`. In `afun`: `any_level = FALSE`, `id = "USUBJID"` |
| **Per-Subject Maximum of Ordinal Variable with Risk Differences (e.g., worst severity grade)** | **`junco::a_freq_j`** | `afun` & `cfun` | Yes | Pre-compute worst grade as factor in data prep. `val = precomputed_grade_var`, `riskdiff = TRUE`, `ref_path` |
| **2-Tier or 3-Tier Conditional Drill-Down within Single `analyze()`** | **`junco::a_two_tier`** / **`junco::a_three_tier`** | `afun` | Depends | `inner_var` (or `inner_var1`, `inner_var2`), `drill_down_levs`, `grp_fun = a_freq_j`, `detail_fun = a_freq_j` |
| **Exposure-Adjusted Incidence Rate (EAIR per 100 PY) + Rate Diff** | **`junco::a_eair100_j`** | `afun` & `cfun` | Yes | `fup_var`, `occ_var`, `occ_dy`, `num_p_year = 100`, `conf_type_eair`, `conf_type_diff`, `riskdiff`, `ref_path` |
| **Stratified EAIR (Study-Size Weighted) + Stratified Rate Diff** | **`junco::a_eair_strat_j`** | `afun` & `cfun` | Yes | `strata`, `fup_var`, `occ_var`, `occ_dy`, `num_p_year = 100`, `riskdiff`, `ref_path` |
| **Total Patient-Years of Exposure Row** | **`junco::a_patyrs_j`** | `afun` & `cfun` | Yes (blanks) | `source = "alt_df"` (or `"df"`), `id = "USUBJID"`, `ref_path` (blanks diff cols) |
| **Combined Continuous Descriptive Statistics + Between-Group Difference (t-test/ANCOVA) in Single `analyze()` (e.g., exposure duration, dose intensity)** | **`junco::a_summarize_ex_j`** | `afun` | Yes | `comp_btw_group = TRUE`, `ancova = TRUE/FALSE`, `daysconv = 1` (days) or `30.4375` (months), `ref_path`, `variables` |
| **Standard Continuous / Categorical Descriptive Summary** | **`tern::a_summary`** | `afun` & `cfun` | No | `.stats = c("n", "mean_sd", "median", "range", "quantiles", "count_fraction")`, `.formats`, `formats_var` |
| **Continuous Summary Suppressing Specific Row Split Levels** | **`junco::a_summary_j_with_exclude`** | `afun` | No | `exclude_levels = list(...)`, `.var`, plus `tern::a_summary` args |
| **Continuous Summary on Filtered Data Subset (with Varying Precision)** | **`junco::a_summary_subset`** / **`junco::c_summary_subset_label`** | `afun` / `cfun` | No | `filter_expr = expression(AGE > 30)`, `formats_var = "fmt_d"`, `label` (for `cfun`) |
| **Statistics in Columns (`N`, `Mean`, `SD`, `SE`, `Med`, `Min`, `Max`)** | **`junco::column_stats`** | `afun` | No | Requires `split_cols_by_multivar()` + `stats_in_cols_setup()`. Args: `var_names = c("AVAL", "CHG", "BASE")`, `exclude_visits`, `formats_var = "fmt_d"` |
| **Difference in Means (Independent or Paired t-test)** | **`junco::a_diff_means`** | `afun` | Yes | `ref_path`, `paired = TRUE/FALSE`, `paired_by = "USUBJID"`, `.stats = "diff_means_est_ci"` |
| **3-Column `[n/N (%), Mean (95% CI), Mean CFB (95% CI)]` + Diff Col** | **`junco::a_summarize_aval_chg_diff_j`** | `afun` | Yes | Pass dummy var (e.g. `"STUDYID"`) to `analyze()`; set `child_labels = "hidden"` on `split_rows_by("AVISIT", ...)`; `multivars = c("AVAL", "AVAL", "CHG")`, `ancova = TRUE/FALSE`, `comp_btw_group = TRUE`, `ref_path`, `d` |
| **Sums and Ratios of Sums (e.g., Actual vs. Planned Cumulative Dose)** | **`junco::a_sum_ratio_j`** | `afun` | No | `.stats = c("sum", "sum_unique", "ratio", "ratio_unique")`, `denom_by = "PLANDOSE"`, `id_var = "USUBJID"` |
| **ANCOVA Summary (Single or Combined Arms via Contrasts/Collapse)** | **`junco::a_summarize_ancova_j`** | `afun` | Yes | `variables = list(arm = ..., covariates = ...)`, `ref_path`, `method_combo = "contrasts"` or `"collapse"`, `weights_combo`, `weights_emmeans` |
| **MMRM Repeated Measures Across Visits and Arms** | **`junco::a_summarize_mmrm`** / **`junco::a_summarize_mmrm_with_exclude`** | `afun` | Yes | `variables = list(arm, visit, id, covariates, response)`, `ref_levels = list(visit, arm)`, `cor_struct`, `exclude_levels` |
| **Pre-Fitted LS Means (`fit_mmrm_j` / `fit_ancova`) or RBMI Pooled LS Means** | **`junco::a_lsmeans`** / **`junco::a_rbmi_lsmeans`** | `afun` | Yes | `ref_path`, `conf_level`, `alternative`, `.stats` |
| **Proportion CI Only (Logical or Factor)** | **`junco::a_proportion_ci_logical`** / **`junco::a_proportion_ci_factor`** | `afun` | No | `conf_level = 0.95`, `method = "clopper-pearson"` (or `"wald"`, `"wilson"`, etc.) |
| **Adaptive Proportion & CI (Auto-Switch Exact Clopper-Pearson vs. Wald)** | **`junco::a_cond_proportion_j`** | `afun` | No | `method_scope = "cell"` or `"row"`, `num_limit = 0`, `denom_limit = 10`, `denom = "n"` or `"N_row"` |
| **Proportion Difference (Risk Diff) & CI vs. Reference Group** | **`junco::a_proportion_diff_j`** | `afun` | Yes | `ref_path`, `method`, `variables = list(strata = ...)`, `.stats = "diff_est_ci"` |
| **Mantel-Fleiss Automated Risk Diff / Test / Combined Row** | **`junco::a_proportion_diff_mf`** / **`a_test_proportion_diff_mf`** / **`a_combo_prop_diff_pval_mf`** | `afun` | Yes | `mf_threshold = 5`, `exact_footnote`, `variables = list(strata = ...)`, `colpaths` (for combo) |
| **Hypothesis Test for Difference Between Two Proportions** | **`junco::a_test_proportion_diff`** | `afun` | Yes | `ref_path`, `method = "chisq"`, `"cmh"`, `"cmh_wh"`, `"fisher"`, `"schouten"` |
| **Odds Ratio (Logistic / Conditional Logistic / CMH) + p-value** | **`junco::a_odds_ratio_j`** | `afun` | Yes | `ref_path`, `method = "exact"`, `"approximate"`, `"efron"`, `"breslow"`, `"cmh"`, `variables` |
| **Stratified Relative Risk (CMH Greenland-Robins) + p-value** | **`junco::a_relative_risk`** | `afun` | Yes | `ref_path`, `variables = list(strata = ...)` |
| **CMH Row Mean Scores Test for Ordered Categorical Response** | **`junco::a_cmhrms_j`** / **`junco::a_cmhrms_j_with_exclude`** | `afun` | Yes | `ref_path`, `variables = list(arm = ..., strata = ...)`, `collapse_combo`, `exclude_levels` |
| **Proportion Table (`n`, `percent`, `cum_percent` Columns)** | **`junco::prop_table_afun`** | `afun` | No | Pair with `prop_split_fun()`; `add_total_level = TRUE/FALSE` |
| **Simple Logical Proportion (`cfun`: count & `%` of `TRUE` relative to `.N_col`)** | **`junco::c_proportion_logical`** | `cfun` | No | Lightweight alternative to `a_proportion_ci_logical` when no CI is needed |
| **Response Rate by Category Levels (`xx / xx (xx.x%)` for `resp_var == "Y"` across `.var` levels)** | **`junco::response_by_var`** | `afun` | No | `resp_var = "TRTEMFL"`. Denominator = non-missing observations of `.var` in `df`. No risk diff support; use `a_freq_resp_var_j` when diff columns are needed |
| **Milestone Kaplan-Meier Survival (Event-Free Rate + 3D CI)** | **`junco::a_event_free`** | `afun` | No | `time_point`, `time_unit`, `is_event`, `percent = TRUE`, `.stats = "event_free_ci"` |
| **Kaplan-Meier Median Survival / Quartiles / Cox Pairwise HR / Multivariable Cox Regression** | **`tern::a_surv_time`** / **`tern::coxph_pairwise`** / **`junco::tefos03_afun`** | `afun` / layout | No | See S4 CAUTION for deprecation notes. `tefos03_afun`: `summarize_coxreg_multivar()` columns (`coef_se`, `pval`, `hr_est`, `hr_ci`) |
| **RESP01 Response Table (Arm Proportions/CIs + Overall RR/OR/p-val)** | **`junco::resp01_acfun`**, **`resp01_a_comp_stat_logical`** / **`resp01_a_comp_stat_factor`**, & **`resp01_counts_cfun`** | `afun` & `cfun` | N/A | Pair with `resp01_split_fun_fct`; `methods`, `arm`, `strata`, `include_comp` |
| **Study Visit Compliance Table (`Expected`, `Received`, `Missing`)** | **`junco::cmp_cfun`** | `cfun` | N/A | `variables = list(expected = ..., received = ..., missing = ...)` |
| **Wide-Format ANCOVA LS Means Table** | **`junco::lsmeans_wide_cfun`** | `cfun` | N/A | `ref_level`, `treatment_levels`, `pval_sided`, `conf_level` |
| **Blank Spacer Row in Layout** | **`junco::ac_blank_line`** | `afun` & `cfun` | N/A | Visual spacer only. See No-Op `analyze()` rule in [`row_struct_layout.md`](./row_struct_layout.md) |
| **Row Count in `summarize_row_groups()` (from `df` or `alt_counts_df`)** | **`junco::c_row_counts`** / **`junco::c_row_counts_alt`** | `cfun` | No | Wrapped by `summarize_row_counts()`. `c_row_counts` counts rows in `df`; `c_row_counts_alt` counts rows in `.alt_df` |

---

## 3.1 Post-Selection Escalation: Expert Skill Index

After selecting a function from the decision matrix above, look up its expert skill and section number below, then activate that skill for detailed parameterization, gotchas, and worked examples.

> Note: This escalation is independent of the `[custom_afuns.md](./custom_afuns.md)` escalation, which applies only when **no** existing function satisfies the requirement (see Section 6, item 5).

| Function | Expert Skill | § |
| :--- | :--- | :--- |
| `ac_blank_line` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 10.2 |
| `a_cmhrms_j` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 6.7 |
| `a_cmhrms_j_with_exclude` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 6.7 |
| `a_combo_prop_diff_pval_mf` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 6.4 |
| `a_cond_proportion_j` | [`junco_adaptive_proportion_diff_usage.md`](./junco_adaptive_proportion_diff_usage.md) | 2 |
| `a_diff_means` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 4.2 |
| `a_eair100_j` | [`junco_eair_family_usage.md`](./junco_eair_family_usage.md) | 3 |
| `a_eair_strat_j` | [`junco_eair_family_usage.md`](./junco_eair_family_usage.md) | 4 |
| `a_event_free` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 7.3 |
| `a_freq_combos_j` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 1.1 |
| `a_freq_j` | [`junco_a_freq_j_usage.md`](./junco_a_freq_j_usage.md) | — |
| `a_freq_j_with_exclude` | [`junco_a_freq_j_usage.md`](./junco_a_freq_j_usage.md) | 10 |
| `a_freq_resp_var_j` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 1.3 |
| `a_freq_subcol_j` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 1.2 |
| `a_lsmeans` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 5.3 |
| `a_maxlev` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 2.1 |
| `a_odds_ratio_j` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 6.5 |
| `a_patyrs_j` | [`junco_eair_family_usage.md`](./junco_eair_family_usage.md) | 2 |
| `a_proportion_ci_factor` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 6.1 |
| `a_proportion_ci_logical` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 6.1 |
| `a_proportion_diff_j` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 6.2 |
| `a_proportion_diff_mf` | [`junco_adaptive_proportion_diff_usage.md`](./junco_adaptive_proportion_diff_usage.md) | 3 |
| `a_rbmi_lsmeans` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 5.4 |
| `a_relative_risk` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 6.6 |
| `a_sum_ratio_j` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 4.5 |
| `a_summarize_ancova_j` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 5.1 |
| `a_summarize_aval_chg_diff_j` | [`junco_a_summarize_aval_chg_diff_j_usage.md`](./junco_a_summarize_aval_chg_diff_j_usage.md) | — |
| `a_summarize_ex_j` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 3.1 |
| `a_summarize_mmrm` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 5.2 |
| `a_summarize_mmrm_with_exclude` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 5.2 |
| `a_summary_j_with_exclude` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 4.3 |
| `a_summary_subset` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 4.4 |
| `a_test_proportion_diff` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 6.3 |
| `a_test_proportion_diff_mf` | [`junco_adaptive_proportion_diff_usage.md`](./junco_adaptive_proportion_diff_usage.md) | 4 |
| `a_three_tier` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 2.3 |
| `a_two_tier` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 2.2 |
| `c_proportion_logical` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 7.2 |
| `c_row_counts` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 10.1 |
| `c_row_counts_alt` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 10.1 |
| `c_summary_subset_label` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 4.4 |
| `cmp_cfun` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 9.1 |
| `column_stats` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 4.1 |
| `lsmeans_wide_cfun` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 9.2 |
| `prop_table_afun` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 7.1 |
| `resp01_a_comp_stat_factor` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 8.2 |
| `resp01_a_comp_stat_logical` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 8.2 |
| `resp01_acfun` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 8.1 |
| `resp01_counts_cfun` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 8.3 |
| `response_by_var` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 7.5 |
| `tefos03_afun` | [`junco_other_afuns_usage.md`](./junco_other_afuns_usage.md) | 7.4 |

> [!CAUTION]
> **Deprecated `junco` Functions — Do Not Select**
> - `junco::a_coxph_hr` was deprecated in `junco 0.1.7` (`lifecycle::deprecate_stop`). Use **`tern::coxph_pairwise()`** / `tern:::a_coxph_pairwise()` instead.
> - `junco::a_kaplan_meier` was deprecated in `junco 0.1.7` (`lifecycle::deprecate_stop`). Use **`tern::a_surv_time()`** instead.

## 4. Cross-Cutting Configuration Mechanisms in `junco`

### 4.1 Wiring Reference Groups via `ref_path` & `get_ref_info()`

Standard `rtables` `ref_group` only works within a single flat `split_cols_by()` call. In clinical tables with spanning headers (e.g., `"Active Study Agent"` vs `" "`) or non-nested Risk Difference columns, `junco` analysis functions locate the control arm using **`ref_path`**:

- **`ref_path` Format**: A character vector alternating `<split_variable_name>, <split_value_name>` down the column hierarchy to the reference leaf column:
  ```r
  # When using spanning header variable "colspan_trt" (where Placebo has span label " ") and arm variable "TRT01A":
  ref_path <- c("colspan_trt", " ", "TRT01A", "Placebo")

  # When using a flat single-level column split on "TRT01A":
  ref_path <- c("TRT01A", "Placebo")
  ```
- **Automatic Injection with `grouped_cols_w_diffs()`**:
  > **RULE: IF columns are built with `grouped_cols_w_diffs()`, THEN do NOT pass `ref_path` via `extra_args` to your `afun`/`cfun`.** `ref_path` is automatically injected into the comparison columns' `extra_args`. When building the column layout manually with `split_cols_by()`, pass `ref_path = ref_path` explicitly inside `extra_args` of your `afun`/`cfun`.

---

### 4.2 Dynamic & Parameter-Specific Decimal Precision (`formats_d` & `format_sigfig_j`)

Clinical laboratory (`ADLB`), vital signs (`ADVS`), and pharmacokinetic (`ADPC`) tables often require different decimal precision (d) for each parameter (`PARAMCD`) or variable. `junco` provides three ways to configure dynamic precision without writing custom `afun`s:

#### Pattern A: Multiple Analyzed Variables with Different Precision (e.g., Demography Table)
Use `fmt_spec_var_d()` and pass the resulting list to the `format` argument of `analyze()`, with `.formats = "default"` inside `extra_args`:
```r
vars_all <- c("AGE", "SEX", "BMRKR1")
vars_d <- c(AGE = 0, SEX = 0, BMRKR1 = 2)
vars_fmt <- fmt_spec_var_d(vars_d, fmt_d_def = junco_def_d_all)

basic_table() |>
  split_cols_by("ARMCD") |>
  analyze(
    vars = vars_all,
    afun = tern::a_summary,
    extra_args = list(.stats = c("n", "mean_sd", "range", "count_fraction"), .formats = "default"),
    format = vars_fmt
  )
```

#### Pattern B: Row-Split Parameters (`PARAMCD`) with Different Precision (`formats_var`)
Use `fmt_spec_df_d()` to create a list-column `fmt_d` on a lookup table, `left_join()` it onto `df`, and pass `formats_var = "fmt_d"` to `analyze()` (for `tern::a_summary`) or inside `extra_args` (for `column_stats` and `a_summary_subset`):
```r
df_d <- tibble::tribble(
  ~PARAMCD, ~d,
  "ALT",     2,
  "CRP",     3,
  "IGA",     1
) |>
  fmt_spec_df_d(d_column = "d", fmt_column = "fmt_d", fmt_d_def = junco_def_d_all)

df <- dplyr::left_join(adlb, df_d, by = "PARAMCD")

# With tern::a_summary (pass formats_var to analyze()):
basic_table() |>
  split_cols_by("ARMCD") |>
  split_rows_by("PARAMCD", split_fun = drop_split_levels) |>
  analyze("AVAL", afun = tern::a_summary,
          extra_args = list(.stats = c("n", "mean_se", "range"), .formats = "default"),
          formats_var = "fmt_d")

# With junco::column_stats (pass formats_var in extra_args):
basic_table() |>
  split_cols_by_multivar(c("AVAL", "BASE", "CHG"), varlabels = c("Visit Values", " ", "Change from Baseline")) |>
  split_cols_by("STUDYID", split_fun = make_split_fun(post = list(stats_in_cols_setup()))) |>
  split_rows_by("PARAMCD", split_fun = drop_split_levels) |>
  analyze("AVISIT", afun = column_stats,
          extra_args = list(exclude_visits = c("BASELINE", "SCREENING"), .formats = "default", formats_var = "fmt_d"))
```

#### Pattern C: Significant Figures (`format_sigfig_j`)
For PK/PD tables requiring k significant figures instead of fixed decimals, pass `format_sigfig_j(k)` in `.formats` or inside `fmt_d_def` with `fmt_spec_df_d()`.

---

## 5. Canonical Layout Wiring Recipes

### Recipe 1: Adverse Event Incidence Table with Spanning Headers & Risk Differences (`a_freq_j` as `afun` and `cfun`)

```r
library(rtables)
library(junco)

ref_path <- c("colspan_trt", " ", "TRT01A", "Placebo")

extra_args_ae <- list(
  denom = "n_altdf",
  riskdiff = TRUE,
  ref_path = ref_path,
  method = "wald",
  .stats = "count_unique_fraction"
)

lyt <- basic_table(show_colcounts = TRUE, colcount_format = "N=xx") |>
  grouped_cols_w_diffs(colspan_trt_map, diffs_label = "Risk Difference (%) (95% CI)") |>
  # Top-level summary row for subjects with >= 1 AE:
  analyze(
    "TRTEMFL",
    afun = a_freq_j,
    extra_args = append(extra_args_ae, list(val = "Y", label = "Subjects with >=1 AE"))
  ) |>
  # Organ class split + group summary row + Preferred Term detail rows:
  split_rows_by(
    "AEBODSYS",
    split_label = "System Organ Class",
    split_fun = trim_levels_in_group("AEDECOD"),
    label_pos = "topleft",
    section_div = " ",
    nested = FALSE
  ) |>
  summarize_row_groups("AEBODSYS", cfun = a_freq_j, extra_args = extra_args_ae) |>
  analyze("AEDECOD", afun = a_freq_j, extra_args = extra_args_ae) |>
  append_topleft("  Preferred Term, n (%)")

tbl <- build_table(lyt, adae, alt_counts_df = adsl)
```

### Recipe 2: Subgroup AE Table with Conditioned Denominators (`denom_by`) and Dual Conditionality

```r
# Subgroup header row: shows subgroup N in arm columns, blanks Risk Diff columns
extra_args_subgrp_hdr <- list(
  denom = "n_altdf",
  denom_by = "SEX",
  riskdiff = FALSE,
  extrablankline = TRUE,
  .stats = "n_altdf",
  label_fstr = "Gender: %s"
)

# Subgroup body rows: shows n/N (%) using sex-specific denominator, populates Risk Diff columns
extra_args_subgrp_body <- list(
  denom = "n_altdf",
  denom_by = "SEX",
  riskdiff = TRUE,
  ref_path = ref_path,
  method = "wald",
  .stats = "count_unique_denom_fraction"
)

lyt <- basic_table(colcount_format = "N=xx") |>
  grouped_cols_w_diffs(colspan_trt_map, diffs_label = "Risk Difference (%) (95% CI)") |>
  split_rows_by("SEX", split_fun = drop_split_levels) |>
  summarize_row_groups("SEX", cfun = a_freq_j, extra_args = extra_args_subgrp_hdr) |>
  split_rows_by("AEBODSYS", split_fun = trim_levels_in_group("AEDECOD"), label_pos = "topleft") |>
  summarize_row_groups("AEBODSYS", cfun = a_freq_j, extra_args = extra_args_subgrp_body) |>
  analyze("AEDECOD", afun = a_freq_j, extra_args = extra_args_subgrp_body)

tbl <- build_table(lyt, adae, alt_counts_df = adsl)
```

### Recipe 3: 3-Column Vital Signs / Laboratory Table with Difference in Mean Change (`a_summarize_aval_chg_diff_j`)

```r
multivars <- c("AVAL", "AVAL", "CHG")

extra_args_3col <- list(
  format_na_str = rep(NA, 3),
  ref_path = c("colspan_trt", " ", "ARM", "B: Placebo"),
  ancova = FALSE,
  comp_btw_group = TRUE,
  multivars = multivars
)

lyt <- basic_table(show_colcounts = FALSE, colcount_format = "N=xx") |>
  split_cols_by("colspan_trt", split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by("ARM", show_colcounts = TRUE, colcount_format = "N=xx") |>
  split_cols_by_multivar(multivars, varlabels = c("n/N (%)", "Mean (95% CI)", "Mean Change From Baseline (95% CI)")) |>
  split_cols_by("rrisk_header_vs", nested = FALSE) |>
  split_cols_by("ARM", split_fun = remove_split_levels("B: Placebo"), labels_var = "rrisk_label", show_colcounts = TRUE, colcount_format = "N=xx") |>
  split_cols_by_multivar(multivars[3], varlabels = " ") |>
  split_rows_by("PARAM", label_pos = "topleft", split_label = "Parameter", section_div = " ", split_fun = drop_split_levels) |>
  split_rows_by("AVISIT", label_pos = "topleft", split_label = "Study Visit", split_fun = drop_split_levels, child_labels = "hidden") |>
  analyze("STUDYID", afun = a_summarize_aval_chg_diff_j, extra_args = extra_args_3col)

tbl <- build_table(lyt, advs, alt_counts_df = adsl)
```

---

## 6. Diagnostic Checklist & Escalation Path

> **RULE: IF a selected function produces unexpected output, THEN the failure is more likely a parameterization error than a capability gap. Re-examine `.var`, `.stats`, `val`/`label`, and `denom`/`denom_by` before escalating to `[custom_afuns.md](./custom_afuns.md)`.**

Before finalizing your `afun`/`cfun` selection, verify all items:

0. **If a selected function produced unexpected output**: This checklist also serves as a diagnostic tool. Before concluding the function cannot satisfy the requirement, re-examine the parameterization against items 1–5 below — the failure is more likely a parameterization error than a capability gap. The most common failure modes are: (a) wrong `.var` (e.g., high-cardinality identifier instead of binary flag — `a_freq_j` enumerates all levels of `.var`), (b) wrong `.stats` (e.g., `"count"` instead of `"count_unique_fraction"`), (c) missing `val`/`label` for single-row flag summaries, (d) wrong `denom`/`denom_by` producing incorrect percentages. Fix the parameterization and retry before escalating to `[custom_afuns.md](./custom_afuns.md)`.
1. **Counting Unit Match**: Does the shell count unique subjects (`count_unique_fraction`), total events (`count`), worst grade per subject (`a_maxlev`), or exposure time (`a_eair100_j` / `a_patyrs_j`)?
2. **Denominator Match**: Is the percentage denominator the column arm total (`denom = "n_altdf"`), a row-subgroup-conditioned arm total (`denom = "n_altdf", denom_by = ...`), the cell sample size (`denom = "n_df"`), or the row total (`denom = "n_rowdf"`)? Is the subject-count dataset passed as `alt_counts_df` to `build_table()`?
3. **Column Conditionality Match**: If the layout has non-nested comparison columns or multi-variable sub-columns, does the selected `afun`/`cfun` accept `ref_path` / `riskdiff` / `comp_btw_group` / `multivars` / `colpaths` to populate each column branch appropriately? **All `tern` functions and `junco::a_maxlev` lack built-in difference column support.** If the layout includes difference columns, either select a `junco` function that supports `riskdiff`/`ref_path` (check the "Diff Col" column in Sections 3–4), or wrap the selected function using the column-dispatch (Pattern 2A) or `get_ref_info()` bridge (Tier 4) patterns from `[custom_afuns.md](./custom_afuns.md)`.
4. **Row Conditionality Match**: Are baseline/screening visits or excluded factor levels properly suppressed via `exclude_visits`, `exclude_levels`, `a_*_with_exclude`, or `a_two_tier(drill_down_levs = ...)`?
5. **Post-Selection Escalation** (two independent decision points):
   - **A suitable function WAS identified** → Check the Section 3.1 expert skill index and activate the linked expert skill for detailed parameterization, gotchas, and worked examples.
   - **No suitable function was identified** → Escalate to `[custom_afuns.md](./custom_afuns.md)` (prioritizing `make_afun()`, `RowsVerticalSection` stacking/dispatch, and `get_ref_info()` over bespoke code) when the table shell requires:
     - A bespoke statistical calculation or custom cell structure not present in `junco` or `tern` (see Section 3.1 to verify coverage).
     - Arbitrary column-facet dispatch across heterogeneous sibling column branches not covered by `junco`'s built-in risk/mean difference, `a_summarize_aval_chg_diff_j`, `column_stats`, `a_combo_prop_diff_pval_mf`, `resp01_acfun`, `cmp_cfun`, or `lsmeans_wide_cfun` patterns.
     - Custom multi-ancestor denominator arithmetic inspecting `.spl_context$full_parent_df` beyond `n_parentdf` and `denom_by`.

