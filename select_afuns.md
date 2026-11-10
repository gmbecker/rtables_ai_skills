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
> - **After selecting a function**, escalate to its **expert usage skill** if one exists (see Section 3.1 below) for detailed parameterization, gotchas, and worked examples. If no expert skill exists, continue with the general reference in Sections 4–7 of this document.
> - **When no existing function satisfies the requirement**, escalate to `[custom_afuns.md](./custom_afuns.md)` for extending/combining existing functions (`make_afun()`, `c(rvs1, rvs2)`, `get_ref_info()`) or writing bespoke custom analysis functions (`function(x, df, .var, .spl_context, ...) { in_rows(...) }`). Always verify whether an existing `junco` or `tern` function (with appropriate `extra_args`) satisfies the requirement before escalating.

---

## 1. Architectural Paradigm: `rtables`, `junco`, and `tern`

1. **Role of `cfun` vs. `afun` in `rtables`**:
   - **`cfun` (Content Function)**: Passed to `summarize_row_groups(lyt, var, cfun = ..., extra_args = ...)`. Generates one or more summary rows at a **row-split group header level** (e.g., summarizing an entire System Organ Class `AEBODSYS`, subgroup `SEX`, or overall `TRTEMFL == "Y"` cohort before indenting into child rows). **`var` must be specified in calls to `summarize_row_groups` when the selected `cfun` accepts `.var` as an argument. The default row summary behavior does not require this.**
   - **`afun` (Analysis Function)**: Passed to `analyze(lyt, vars, afun = ..., extra_args = ...)`. Generates the **individual data rows** at the leaf (or current) level of the row tree (e.g., Preferred Terms `AEDECOD`, numeric summary rows for `AGE` or `AVAL`, model contrasts).
2. **Direct `afun`/`cfun` Paradigm (`junco`) vs. Layout Wrappers (`tern`)**:
   - In **`tern`**, tables are often built with high-level layout wrapper functions (e.g., `count_occurrences()`, `summarize_occurrences()`, `analyze_num_patients()`, `summarize_vars()`) that internally call `rtables::analyze()` or `rtables::summarize_row_groups()`, though underlying analysis functions like `tern::a_summary` or `tern::a_surv_time` can also be passed directly to `analyze()`.
   - In **`junco`**, statistical logic is implemented as standalone functions prefixed with `a_`, `c_`, or `ac_` that are passed **directly** as the `afun` argument to `rtables::analyze()` or as the `cfun` argument to `rtables::summarize_row_groups()`, configured via `extra_args = list(...)`.
   - **`tern` functions do not support junco-style difference column layouts**: No `tern` function (`a_summary`, `a_surv_time`, `coxph_pairwise`, etc.) supports `riskdiff`/`ref_path` or junco-style column conditionality out of the box. `junco` functions were designed with `grouped_cols_w_diffs()` column topologies in mind; `tern` functions were not. When a table requires difference columns and the statistical calculation is only available in `tern`, escalate to `[custom_afuns.md](./custom_afuns.md)` to wrap the `tern` function in a column-dispatch wrapper (Pattern 2A) or a `get_ref_info()` bridge (Tier 3).
   - **Dual-Purpose `a/cfun` Functions**: Many core `junco` functions (`a_freq_j`, `a_freq_j_with_exclude`, `a_freq_combos_j`, `a_freq_subcol_j`, `a_freq_resp_var_j`, `a_maxlev`, `a_eair100_j`, `a_eair_strat_j`, `a_patyrs_j`, `resp01_acfun`, `ac_blank_line`) work seamlessly as **both** an `afun` in `analyze()` and a `cfun` in `summarize_row_groups()`.

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
| **Unique Subjects (Patients)** | Number of distinct patients (`USUBJID`) experiencing $\ge 1$ event in that cell (`length(unique(x))`). Standard for AE incidence tables. | `a_freq_j` with `.stats = "count_unique"`, `"count_unique_fraction"`, or `"count_unique_denom_fraction"` *(default)* |
| **Total Events (Occurrences)** | Total number of event records in that cell (`length(x)`), where a single patient may contribute multiple events. | `a_freq_j` with `.stats = "count"`; or `a_eair100_j(..., count_multiple_events = TRUE)` |
| **Subject's Worst / Maximum Severity** | Each patient is counted **once** per category at their maximum ordered factor level (e.g., highest CTCAE toxicity grade `AETOXGR` or severity `AESEV`). | `a_maxlev` (`any_level = TRUE` in `cfun` for "Any Grade" header; `any_level = FALSE` in `afun` for grade breakdown) |
| **Person-Years / Time at Risk** | Cumulative follow-up duration across patients (either total exposure time or time censored at first event occurrence `occ_dy / 365.25`). | `a_patyrs_j` (total exposure) and `a_eair100_j` / `a_eair_strat_j` (events per 100 person-years) |
| **Cumulative Sums & Ratios of Sums** | Sum of a continuous measure (e.g., actual dose received) and its ratio to another summed measure (e.g., planned dose), across all records or unique subjects. | `a_sum_ratio_j` with `.stats = c("sum", "sum_unique", "ratio", "ratio_unique")` and `denom_by` |
| **Continuous Distribution / Model Estimate** | Arithmetic mean, SD, SE, median, IQR, min/max, geometric mean, or model-based LS means (ANCOVA, MMRM, RBMI) across observations in the facet. | `tern::a_summary`, `a_summary_subset`, `column_stats`, `a_summarize_aval_chg_diff_j`, `a_summarize_ancova_j`, `a_summarize_mmrm`, `a_lsmeans`, `a_rbmi_lsmeans` |

> [!WARNING]
> **Never Confuse Unique Patients with Total Events**
> In repeated-measures or event-level datasets (`ADAE`, `ADCM`, `ADMH`), a format shell showing `xx (xx.x%)` almost always requires **unique subjects** (`count_unique_fraction`), whereas a row labeled `"Total number of events"` requires **event counts** (`count`). Check the shell's row labels and footnotes carefully.

---

### 2.3 Fundamental Question 2: Denominators & Target (Sub)Populations

When a table shell displays percentages (`xx (xx.x%)` or `xx / xx (xx.x%)`), the percentage is meaningless unless you identify **which (sub)population is the denominator**.

In `rtables`, percentages can be calculated relative to five distinct structural scopes—and each scope can count either **subjects from `alt_counts_df` (`ADSL`)** or **records/subjects in the primary dataset (`df`)**:

| Denominator Scope | Meaning | When It Is Used | `a_freq_j` Parameterization | Other `junco` Functions |
| :--- | :--- | :--- | :--- | :--- |
| **1. Column Population (`.N_col` / `alt_counts_df`)** | Total subjects (or records) in the current column (e.g., Treatment Arm N). | Standard demographics, AE incidence, concomitant meds where `%` is out of all treated patients in that arm. | `denom = "N_col"` *(default)* or `denom = "n_altdf"` (requires `alt_counts_df = adsl` in `build_table()`) | `a_proportion_ci_logical`, `a_proportion_ci_factor`, `c_proportion_logical`, `resp01_acfun` |
| **2. Conditioned Subgroup within Column (`alt_counts_df` subset)** | Total subjects in `alt_counts_df` belonging to **both** the current column arm **and** an ancestor row subgroup (e.g., Female patients in Arm A). | Subgroup tables (e.g., AEs split by `SEX` or `AGEGR1` on rows) where `%` is out of all Females in Arm A, not all patients in Arm A. | `denom = "n_altdf"` combined with `denom_by = "SEX"` (or vector of row-split vars) | — |
| **3. Individual Cell Facet (`df` slice)** | Number of non-missing observations/subjects inside the current `(row, col)` cell itself. | Responder rates within a visit/parameter facet (`n/N (%)` where `N` is patients with an assessment at that visit), or conditional proportions. | `denom = "n_df"` | `a_cond_proportion_j(denom = "n")`, `response_by_var`, `a_summarize_aval_chg_diff_j(denom = "N")` |
| **4. Row Facet Across All Columns (`.df_row`)** | Total observations across **all** columns for the current row split level. | Cross-tabulations where horizontal row percentages sum to 100% across columns. | `denom = "n_rowdf"` | `a_cond_proportion_j(denom = "N_row")` |
| **5. Parent / Ancestor Row Facet** | Total observations in the parent row split (`n_parentdf`) or column group (`N_colgroup`). | Hierarchical breakdowns where child percentages are expressed as a share of the parent category count. | `denom = "n_parentdf"` or `denom = "N_colgroup"` | `cmp_cfun` (Received/Missing visits divided by Expected visits in that row) |

> [!IMPORTANT]
> **Why `alt_counts_df` Matters for Denominators**
> Not every patient in a clinical trial suffers an adverse event or takes a concomitant medication. Therefore, `ADAE` and `ADCM` only contain rows for patients who had an event.
> - If you compute denominators from `df` (`ADAE`), the denominator will only count patients who had at least one AE (or worse, total AE records).
> - Always pass `alt_counts_df = adsl` to `build_table(lyt, df, alt_counts_df = adsl)` and set `denom = "n_altdf"` (or `denom = "N_col"`) when percentages are relative to the full safety/ITT population, and add `denom_by = "<row_subgrp_var>"` when row splits partition the patient population into subgroups.

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

**Selection Rule**: Never use a column-agnostic `afun` in a heterogeneous column layout. **All `tern` functions are column-agnostic by default** and require wrapping (via Pattern 2A column dispatch or Tier 3 bridging from `[custom_afuns.md](./custom_afuns.md)`) before they can be used in layouts with difference columns. Select a `junco` function built for that exact column topology:
- Spanning arms + Risk/Rate/Mean Difference columns $\to$ `a_freq_j(..., riskdiff = TRUE, ref_path = ...)`, `a_eair100_j(..., riskdiff = TRUE, ref_path = ...)`, `a_diff_means(..., ref_path = ...)`, `a_summarize_ex_j(..., comp_btw_group = TRUE, ref_path = ...)`.
- 3-column `[n/N (%), Mean, Mean CFB]` + optional Diff column $\to$ `a_summarize_aval_chg_diff_j`.
- Stats-in-columns (`N`, `Mean`, `SD`, `SE`, `Med`, `Min`, `Max`) $\to$ `column_stats` (paired with `stats_in_cols_setup()`).
- Sub-column conditional event filtering $\to$ `a_freq_subcol_j`.
- Cumulative / period combo columns $\to$ `a_freq_combos_j`.
- Proportions + Risk Diff + p-value in separate columns on the same row $\to$ `a_combo_prop_diff_pval_mf`.
- `RESP01` arm proportions + Overall column RR/OR/p-value $\to$ `resp01_acfun`.
- Expected / Received / Missing compliance columns $\to$ `cmp_cfun`.
- Wide-format ANCOVA columns (Ref, Test, Variance, Comparison) $\to$ `lsmeans_wide_cfun`.

#### B. Conditionality Based on Row Facet
Purely row-conditional `afun`s produce different numbers of rows, different statistics, or suppress output entirely depending on the current row split level:
- **Excluding Baseline/Screening Visits from Change-from-Baseline or Modeling**: In longitudinal visit tables (`split_rows_by("AVISIT")`), `BASELINE` has no Change from Baseline or MMRM contrast.
- **Suppressing Specific Row Split Levels**: E.g., suppressing summary statistics or p-values for a `"Missing"` or `"Not Applicable"` category.
- **Selective Hierarchical Drill-Down**: Expanding detail rows (e.g., discontinuation reasons) *only* under specific parent levels (e.g., `"DISCONTINUED"`), while leaving other parent levels (e.g., `"COMPLETED"`, `"ONGOING"`) as single summary rows.
- **Parameter-Specific Decimal Precision**: Formatting `ALT` to 2 decimals, `CRP` to 3 decimals, and `IGA` to 1 decimal based on the current `PARAMCD` row facet.

**Selection Rule**: Use `junco`'s row-conditional mechanisms:
- Exclude specific row split levels from output $\to$ `a_freq_j_with_exclude`, `a_summary_j_with_exclude`, `a_summarize_mmrm_with_exclude`, `a_cmhrms_j_with_exclude` (via `exclude_levels = list(VAR = c("LEV1", ...))`).
- Exclude baseline visits from `BASE`/`CHG` columns $\to$ `column_stats(exclude_visits = c("BASELINE", "SCREENING"))` or `a_summarize_mmrm(ref_levels = list(visit = "BASELINE", arm = ...))`.
- Selective 2-tier or 3-tier drill-down under specific levels $\to$ `a_two_tier(inner_var = ..., drill_down_levs = ...)` or `a_three_tier(...)`.
- Vary decimal precision by row parameter $\to$ `formats_var = "fmt_d"` in `analyze()` / `column_stats` / `a_summary_subset`, or `d` in `a_summarize_aval_chg_diff_j`.

#### C. Dual Conditionality (Simultaneous Row $\times$ Column Conditionality)
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
| **Standard AE / Concomitant Med / Medical History / Demographic Counts & `%` (with optional Risk Diff)** | **`junco::a_freq_j`** | `afun` & `cfun` | Yes | `.stats = "count_unique_fraction"` (or `"count_unique_denom_fraction"`), `denom = "n_altdf"`, `riskdiff = TRUE/FALSE`, `ref_path`, `method`, `val`, `label_map` |
| **Frequency Counts & `%` Suppressing Specific Row Split Levels** | **`junco::a_freq_j_with_exclude`** | `afun` & `cfun` | Yes | `exclude_levels = list(VAR = "LevelToSkip")` + all `a_freq_j` args |
| **Subgroup Header Row Showing Sample Size Only (`Gender: F   79   66   77`)** | **`junco::a_freq_j`** | `cfun` | Yes | `.stats = "n_altdf"`, `denom = "n_altdf"`, `denom_by = "SEX"`, `riskdiff = FALSE`, `label_fstr = "Gender: %s"` |
| **Subgroup Frequency Rows (`n/N (%)` with Subgroup Denominator)** | **`junco::a_freq_j`** | `afun` & `cfun` | Yes | `.stats = "count_unique_denom_fraction"`, `denom = "n_altdf"`, `denom_by = "SEX"`, `riskdiff = TRUE`, `ref_path` |
| **Cumulative / Cycle Period Frequency Columns** | **`junco::a_freq_combos_j`** | `afun` & `cfun` | Yes | `combosdf`, `filter_var`, `flag_var`, `do_not_filter` |
| **Sub-Columns Filtering Events (e.g., Serious / Related AEs under Arms)** | **`junco::a_freq_subcol_j`** | `afun` & `cfun` | Yes | `subcol_split`, `subcol_var`, `subcol_val` |
| **Binary Flag (`"Y"`/`"N"`) Stratified by Categorical Variable + Risk Diff** | **`junco::a_freq_resp_var_j`** | `afun` & `cfun` | Yes | `resp_var = "TRTEMFL"`, `riskdiff = TRUE`, `ref_path`, `method` |
| **Maximum Severity / Toxicity Grade per Subject** | **`junco::a_maxlev`** | `afun` & `cfun` | **No** | In `cfun`: `any_level = TRUE`, `any_level_exclude = "Missing"`. In `afun`: `any_level = FALSE`, `id = "USUBJID"` |
| **2-Tier or 3-Tier Conditional Drill-Down within Single `analyze()`** | **`junco::a_two_tier`** / **`junco::a_three_tier`** | `afun` | Depends | `inner_var` (or `inner_var1`, `inner_var2`), `drill_down_levs`, `grp_fun = a_freq_j`, `detail_fun = a_freq_j` |
| **Exposure-Adjusted Incidence Rate (EAIR per 100 PY) + Rate Diff** | **`junco::a_eair100_j`** | `afun` & `cfun` | Yes | `fup_var`, `occ_var`, `occ_dy`, `num_p_year = 100`, `conf_type_eair`, `conf_type_diff`, `riskdiff`, `ref_path` |
| **Stratified EAIR (Study-Size Weighted) + Stratified Rate Diff** | **`junco::a_eair_strat_j`** | `afun` & `cfun` | Yes | `strata`, `fup_var`, `occ_var`, `occ_dy`, `num_p_year = 100`, `riskdiff`, `ref_path` |
| **Total Patient-Years of Exposure Row** | **`junco::a_patyrs_j`** | `afun` & `cfun` | Yes (blanks) | `source = "alt_df"` (or `"df"`), `id = "USUBJID"`, `ref_path` (blanks diff cols) |
| **Study Drug Exposure Duration / Dose Intensity + t-test/ANCOVA Diff** | **`junco::a_summarize_ex_j`** | `afun` | Yes | `comp_btw_group = TRUE`, `ancova = TRUE/FALSE`, `daysconv = 1` (days) or `30.4375` (months), `ref_path`, `variables` |
| **Standard Continuous / Categorical Descriptive Summary** | **`tern::a_summary`** | `afun` & `cfun` | No | `.stats = c("n", "mean_sd", "median", "range", "quantiles", "count_fraction")`, `.formats`, `formats_var` |
| **Continuous Summary Suppressing Specific Row Split Levels** | **`junco::a_summary_j_with_exclude`** | `afun` | No | `exclude_levels = list(...)`, `.var`, plus `tern::a_summary` args |
| **Continuous Summary on Filtered Data Subset (with Varying Precision)** | **`junco::a_summary_subset`** / **`junco::c_summary_subset_label`** | `afun` / `cfun` | No | `filter_expr = expression(AGE > 30)`, `formats_var = "fmt_d"`, `label` (for `cfun`) |
| **Statistics in Columns (`N`, `Mean`, `SD`, `SE`, `Med`, `Min`, `Max`)** | **`junco::column_stats`** | `afun` | No | Pair with `stats_in_cols_setup()`. Args: `var_names = c("AVAL", "CHG", "BASE")`, `exclude_visits`, `formats_var = "fmt_d"` |
| **Difference in Means (Independent or Paired t-test)** | **`junco::a_diff_means`** | `afun` | Yes | `ref_path`, `paired = TRUE/FALSE`, `paired_by = "USUBJID"`, `.stats = "diff_means_est_ci"` |
| **3-Column `[n/N (%), Mean (95% CI), Mean CFB (95% CI)]` + Diff Col** | **`junco::a_summarize_aval_chg_diff_j`** | `afun` | Yes | Pass dummy var (e.g. `"STUDYID"`) to `analyze()`; `multivars = c("AVAL", "AVAL", "CHG")`, `ancova = TRUE/FALSE`, `comp_btw_group = TRUE`, `ref_path`, `d` |
| **ANCOVA Summary (Single or Combined Arms via Contrasts/Collapse)** | **`junco::a_summarize_ancova_j`** | `afun` | Yes | `variables = list(arm = ..., covariates = ...)`, `ref_path`, `method_combo = "contrasts"` or `"collapse"`, `weights_combo`, `weights_emmeans` |
| **MMRM Repeated Measures Across Visits and Arms** | **`junco::a_summarize_mmrm`** / **`junco::a_summarize_mmrm_with_exclude`** | `afun` | Yes | `variables = list(arm, visit, id, covariates, response)`, `ref_levels = list(visit, arm)`, `cor_struct`, `exclude_levels` |
| **Pre-Fitted LS Means (`fit_mmrm_j` / `fit_ancova`) or RBMI Pooled LS Means** | **`junco::a_lsmeans`** / **`junco::a_rbmi_lsmeans`** | `afun` | Yes | `ref_path`, `conf_level`, `alternative`, `.stats` |
| **Sums and Ratios of Sums (e.g., Actual vs. Planned Cumulative Dose)** | **`junco::a_sum_ratio_j`** | `afun` | No | `.stats = c("sum", "sum_unique", "ratio", "ratio_unique")`, `denom_by = "PLANDOSE"`, `id_var = "USUBJID"` |
| **Proportion CI Only (Logical or Factor)** | **`junco::a_proportion_ci_logical`** / **`junco::a_proportion_ci_factor`** | `afun` | No | `conf_level = 0.95`, `method = "clopper-pearson"` (or `"wald"`, `"wilson"`, etc.) |
| **Adaptive Proportion & CI (Auto-Switch Exact Clopper-Pearson vs. Wald)** | **`junco::a_cond_proportion_j`** | `afun` | No | `method_scope = "cell"` or `"row"`, `num_limit = 0`, `denom_limit = 10`, `denom = "n"` or `"N_row"` |
| **Proportion Difference (Risk Diff) & CI vs. Reference Group** | **`junco::a_proportion_diff_j`** | `afun` | Yes | `ref_path`, `method`, `variables = list(strata = ...)`, `.stats = "diff_est_ci"` |
| **Mantel-Fleiss Automated Risk Diff / Test / Combined Row** | **`junco::a_proportion_diff_mf`** / **`a_test_proportion_diff_mf`** / **`a_combo_prop_diff_pval_mf`** | `afun` | Yes | `mf_threshold = 5`, `exact_footnote`, `variables = list(strata = ...)`, `colpaths` (for combo) |
| **Hypothesis Test for Difference Between Two Proportions** | **`junco::a_test_proportion_diff`** | `afun` | Yes | `ref_path`, `method = "chisq"`, `"cmh"`, `"cmh_wh"`, `"fisher"`, `"schouten"` |
| **Odds Ratio (Logistic / Conditional Logistic / CMH) + p-value** | **`junco::a_odds_ratio_j`** | `afun` | Yes | `ref_path`, `method = "exact"`, `"approximate"`, `"efron"`, `"breslow"`, `"cmh"`, `variables` |
| **Stratified Relative Risk (CMH Greenland-Robins) + p-value** | **`junco::a_relative_risk`** | `afun` | Yes | `ref_path`, `variables = list(strata = ...)` |
| **CMH Row Mean Scores Test for Ordered Categorical Response** | **`junco::a_cmhrms_j`** / **`junco::a_cmhrms_j_with_exclude`** | `afun` | Yes | `ref_path`, `variables = list(arm = ..., strata = ...)`, `collapse_combo`, `exclude_levels` |
| **Proportion Table (`n`, `percent`, `cum_percent` Columns)** | **`junco::prop_table_afun`** | `afun` | No | Pair with `prop_split_fun()`; `add_total_level = TRUE/FALSE` |
| **Milestone Kaplan-Meier Survival (Event-Free Rate + 3D CI)** | **`junco::a_event_free`** | `afun` | No | `time_point`, `time_unit`, `is_event`, `percent = TRUE`, `.stats = "event_free_ci"` |
| **Kaplan-Meier Median Survival / Quartiles / Cox Pairwise HR** | **`tern::a_surv_time`** / **`tern::coxph_pairwise`** | `afun` / layout | No | Replaces deprecated `junco::a_kaplan_meier` and `junco::a_coxph_hr` |
| **RESP01 Response Table (Arm Proportions/CIs + Overall RR/OR/p-val)** | **`junco::resp01_acfun`** & **`resp01_counts_cfun`** | `afun` & `cfun` | N/A | Pair with `resp01_split_fun_fct`; `methods`, `arm`, `strata`, `include_comp` |
| **Study Visit Compliance Table (`Expected`, `Received`, `Missing`)** | **`junco::cmp_cfun`** | `cfun` | N/A | `variables = list(expected = ..., received = ..., missing = ...)` |
| **Wide-Format ANCOVA LS Means Table** | **`junco::lsmeans_wide_cfun`** | `cfun` | N/A | `ref_level`, `treatment_levels`, `pval_sided`, `conf_level` |
| **Blank Spacer Row in Layout** | **`junco::ac_blank_line`** | `afun` & `cfun` | N/A | Inserts empty row formatted as `""`. **⚠️ Not a substitute for a real `afun`**: only use (1) as a visual spacer `analyze()` between two substantive analysis sections (or via `insert_blank_line()`), or (2) as a `cfun` in `summarize_row_groups(cfun = ac_blank_line)` paired with `split_rows_by(..., child_labels = "visible")` to insert a blank line between a visible facet label row and its child rows. Never use as the sole terminating `afun` of a `split_rows_by()` chain — see No-Op `analyze()` Anti-Pattern in [`row_struct_layout.md`](./row_struct_layout.md). |

---

## 3.1 Post-Selection Escalation: Expert Usage Skills

Once you have identified the correct `afun`/`cfun` from the decision matrix above, check whether a **dedicated expert usage skill** exists for that function. Expert skills provide exhaustive parameter documentation, internal mechanics, worked examples, and gotcha catalogs that go far beyond what this selection guide covers.

**Escalation routing table:**

| Selected Function(s) | Expert Skill Document | Covers |
| :--- | :--- | :--- |
| `a_freq_j`, `a_freq_j_with_exclude` | [`junco_a_freq_j_usage.md`](./junco_a_freq_j_usage.md) | All `.stats`, denominator control (`denom`, `denom_by`, `colgroup`, `countsource`), risk difference methods, level filtering/manipulation (`val`, `drop_levels`, `excl_levels`, `new_levels`), `label_map` (4 modes), `restr_columns`, formatting, internal mechanics, 13 gotchas |
| `a_eair100_j`, `a_eair_strat_j`, `a_patyrs_j` | [`junco_eair_family_usage.md`](./junco_eair_family_usage.md) | Person-years calculation, 11 statistics each, horizontal/vertical risk diff layouts, 4 rate CI methods, 4 difference CI methods, `occ_var`/`occ_dy`/`fup_var` wiring, 5 worked examples, 8 gotchas |
| `a_summarize_aval_chg_diff_j` | [`junco_a_summarize_aval_chg_diff_j_usage.md`](./junco_a_summarize_aval_chg_diff_j_usage.md) | Per-column `.stats`/`.formats` as named lists (`col1`, `col23`, `coldiff`), `split_cols_by_multivar` integration, descriptive vs ANCOVA mode, sparse data handling, combined group support, 3 examples, 8 gotchas |
| `a_cond_proportion_j`, `a_proportion_diff_mf`, `a_test_proportion_diff_mf` | [`junco_adaptive_proportion_diff_usage.md`](./junco_adaptive_proportion_diff_usage.md) | Adaptive method selection, Mantel-Fleiss criterion, footnote injection, CMH method options, combined usage pattern, 10 gotchas |

**Escalation rule** (applies only when a suitable function WAS identified):
1. If the selected function appears in the table above → **activate the linked expert skill** for detailed parameterization, gotchas, and worked examples.
2. If the selected function does NOT appear in the table above → **continue with the general reference** in Sections 4–7 of this document, supplemented by the canonical layout recipes in Section 7.

> Note: This escalation is independent of the `[custom_afuns.md](./custom_afuns.md)` escalation, which applies only when **no** existing function satisfies the requirement (see Section 8, item 5).

---

## 4. Master Catalog of `junco` `afun`s and `cfun`s

| Name | Type | Diff Col | Description | Statistics Supported (`.stats`) | Denominators Supported (`denom`) | Other Customization (`extra_args`) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`a_freq_j`** | `afun` / `cfun` | Yes | Primary frequency and percentage analysis/content function. Tabulates counts, unique subject counts, fractions, and risk differences across treatment arms and difference columns. | `count_unique_denom_fraction` *(default)*, `count`, `count_unique`, `count_unique_fraction`, `unique_fraction`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, `denom`. In risk diff columns: risk difference + CI. | `N_col` *(default)*, `n_df`, `n_altdf`, `N_colgroup`, `n_rowdf`, `n_parentdf`. Subgroup denominator conditioning via `denom_by`. Data source via `countsource` (`"df"`, `"altdf"`, `"altdf_subset"`). | `riskdiff` (toggles RD calculation), `ref_path` (reference group path), `method` (`"wald"`, `"waldcc"`, `"cmh"`, `"ha"`, `"newcombe"`, `"newcombecc"`, `"strat_newcombe"`, `"strat_newcombecc"`, `"cmh_sato"`, `"cmh_mn"`, `"uncond_exact_diff"`), `conf_level`, `weights_method`, `val` (filter levels), `drop_levels`, `excl_levels`, `new_levels`, `new_levels_after`, `addstr2levs`, `label`, `label_fstr`, `label_map` (conditional row-split label translation), `.formats`, `.indent_mods`, `na_str`, `restr_columns`, `extrablankline`. |
| **`a_freq_j_with_exclude`** | `afun` / `cfun` | Yes | Wrapper around `a_freq_j()` that conditionally excludes specified row split levels from producing analysis output. | Identical to `a_freq_j()`. | Identical to `a_freq_j()`. | `exclude_levels` (named list mapping split variables to vectors of levels to exclude from tabulation), plus all parameters of `a_freq_j()`. |
| **`a_freq_combos_j`** | `afun` / `cfun` | Yes | Frequency count and percentage in layouts with cumulative/period column designs controlled by `combosdf`. | `count_unique_denom_fraction` *(default)*, `count`, `count_unique`, `count_unique_fraction`, `unique_fraction`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, `denom`. | `N_col`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, with `denom_by` support. | `combosdf` (mapping of column facets to cumulative counts), `filter_var` (record filter to prevent overcounting), `flag_var`, `do_not_filter` (facets retaining cumulative numerator, e.g. Total), `val`, `label`, `label_fstr`, `label_map`, `.formats`, `.labels_n`, `.indent_mods`, `na_str`. |
| **`a_freq_subcol_j`** | `afun` / `cfun` | Yes | Frequency count and percentage with conditional column-subsetting in selected sub-columns (e.g. serious/drug-related AEs nested under arms). | `count_unique_denom_fraction` *(default)*, `count`, `count_unique`, `count_unique_fraction`, `unique_fraction`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, `denom`. | `N_col`, `n_df`, `n_altdf`, `n_rowdf`, `n_parentdf`, with `denom_by` support. | `subcol_split` (column path matching text), `subcol_var` (variable to inspect in matched columns), `subcol_val` (required subset value), `val`, `label`, `label_fstr`, `label_map`, `.formats`, `.labels_n`, `.indent_mods`, `na_str`. |
| **`a_freq_resp_var_j`** | `afun` / `cfun` | Yes | Summarizes binary/response variables (with `"Y"`/`"N"` flags) across categories of an analysis variable, with optional risk difference. | Count, denominator, percentage (`xx / xx (xx.x%)`); in risk difference columns: difference + CI (`xx.x (xx.x, xx.x)`). | Derived from column count `.N_col`. | `resp_var` (name of response flag variable, e.g. `"TRTEMFL"`), `riskdiff` (toggles RD calculation), `ref_path`, `conf_level`, `method` (RD methods: `"wald"`, `"cmh"`, etc.), `weights_method`, `drop_levels`, `.formats`, `na_str`, `legacy`. |
| **`a_maxlev`** | `afun` / `cfun` | **No** | Calculates count and percentage of the maximum level of an ordered factor (e.g. maximum AE toxicity grade or severity) per unique subject (`id`). **⚠️ Does not support `riskdiff`/`ref_path`; cannot be used directly in layouts with difference columns (`grouped_cols_w_diffs` or manual risk-difference column branches). See workaround patterns in Section 5.1 item 5.** | Frequencies of maximum levels per subject (`count_fraction`); when `any_level = TRUE` (used as `cfun`), tabulates "any" occurrence across maximum levels. | Derived from `.alt_df_full` / `alt_counts_df` (patient population). | `any_level` (`TRUE` for `cfun` group row, `FALSE` for `afun` level rows), `any_level_exclude` (levels to exclude from "any" summary, default `"Missing"`), `id` (subject identifier, default `"USUBJID"`). |
| **`a_two_tier`** | `afun` | Depends | Creates a two-tier nested row hierarchy within a single `analyze()` call: displays a group summary row and conditionally generates indented detail rows under specified levels. | Determined by `grp_fun` (outer level) and `detail_fun` (inner detail level), typically `a_freq_j()`. | Inherited from `grp_fun` and `detail_fun` via `.alt_df_full` / `.N_col` / `denom`. | `inner_var` (variable for detail rows), `drill_down_levs` (levels of `.var` that expand into detail rows), `use_all_levels` (whether to show unobserved levels of `inner_var`), `grp_fun`, `detail_fun`, `...` (passed to both functions). |
| **`a_three_tier`** | `afun` | Depends | Extends `a_two_tier()` to three nested levels within a single `analyze()` call (outer group summary $\to$ mid-level detail $\to$ innermost detail). | Determined by `grp_fun` and `detail_fun`. | Inherited from `grp_fun` and `detail_fun`. | `inner_var1` (second-tier variable), `inner_var2` (third-tier variable), `drill_down_levs`, `use_all_levels`, `grp_fun`, `detail_fun`, `...`. |
| **`a_eair100_j`** | `afun` / `cfun` | Yes | Exposure-Adjusted Incidence Rate (EAIR) summary per 100 person-years (or custom multiplier), with between-group rate differences and CIs. | `eair` *(default)*, `n_event`, `person_years`, `n_eair`, `eair_ci`, `eair_est_ci`, `eair_n`, `eair_n_py`, `eair_diff_est_ci`, `eair_diff_est`, `eair_diff_ci`. | Person-years of follow-up exposure (`fup_var` in years, or `occ_dy / 365.25` for event subjects) derived from `df` and `.alt_df_full`. | `fup_var` (follow-up duration variable in years), `occ_var` (first-occurrence event flag `"Y"`), `occ_dy` (event day relative to exposure), `num_p_year` (rate multiplier, default `100`), `count_multiple_events`, `conf_type_eair` (`"exact"`, `"normal"`, `"normal_log"`, `"byar"`), `conf_type_diff` (`"wald"`, `"waldcc"`, `"mn"`, `"scas"`), `riskdiff`, `ref_path`, `conf_level`, `riskdiff_setup` (`"horizontal"`, `"vertical"`), `row_labels_adj`, `.stats`, `.formats`, `.labels`, `na_str`. |
| **`a_eair_strat_j`** | `afun` / `cfun` | Yes | Stratified Exposure-Adjusted Incidence Rate (EAIR) across clinical strata (e.g., study size-weighted EAIR), with stratified rate differences. | `eair_strat` *(default)*, `n_event_total`, `person_years_total`, `n_eair_strat`, `eair_strat_ci`, `eair_strat_est_ci`, `eair_strat_n`, `eair_strat_n_py`, `eair_strat_diff_est_ci`, `eair_strat_diff_est`, `eair_strat_diff_ci`. | Total stratified person-years across strata levels from `fup_var` and `occ_dy`. | `strata` (stratification variable name), `fup_var`, `occ_var`, `occ_dy`, `num_p_year` (default `100`), `count_multiple_events`, `riskdiff`, `ref_path`, `conf_level`, `riskdiff_setup`, `row_labels_adj`, `.stats`, `.formats`, `.labels`, `na_str`. |
| **`a_patyrs_j`** | `afun` / `cfun` | Yes (blanks) | Tabulates total patient-years of exposure. Used independently or as an exposure summary row above EAIR rows. | `patyrs` (total patient-years of exposure). | Sum of `.var` across subjects in `source` (`alt_df` or `df`). | `source` (`"alt_df"` or `"df"`), `id` (subject ID), `ref_path` (blanked out in risk diff columns), `.stats`, `.formats`, `.labels`. |
| **`a_summarize_ex_j`** | `afun` | Yes | Tabulation for exposure tables (extent/duration of exposure in days or months, dose intensity), with descriptive summaries or ANCOVA/t-test comparisons. | `mean_sd`, `median`, `range`, `quantiles`, `total_subject_years` *(defaults)*; in difference columns: `diff_mean_est_ci`. | Continuous summary / sum of duration; converts units via `daysconv`. | `comp_btw_group` (enables difference column estimation), `ref_path`, `ancova` (if `TRUE`, uses ANCOVA model for between-group differences; if `FALSE`, uses 2-sample t-test), `interaction_y`, `interaction_item`, `conf_level`, `daysconv` (unit multiplier: `1` for days, `30.4375` for months), `variables`, `.stats`, `.formats`, `.labels`, `na_str`. |
| **`column_stats`** | `afun` | No | Statistics-in-columns analysis function. Presents summary statistics across column variables (e.g. `AVAL`, `CHG`, `BASE` in columns across visits in rows) with varying decimal precision. | `n`, `mean`, `sd`, `se`, `median`, `min`, `max`, `range`, `mean_sd`, `mean_se` (evaluated across specified `var_names`). | N/A (continuous variables summary). | `formats_var` (list column in dataset defining parameter-specific formats `fmt_d`), `exclude_visits` (visits excluded from `BASE`/`CHG` columns, default `"Baseline (DB)"`), `var_names` (variables analyzed, default `c("AVAL", "CHG", "BASE")`), `.formats`. |
| **`a_diff_means`** | `afun` | Yes | Computes point estimate and inferential statistics for the difference in population means between active treatment and reference group. | `diff_means_est_ci` *(default)*, `diff_means_n1`, `diff_means_n2`, `diff_means_est`, `diff_means_se`, `diff_means_est_se`, `diff_means_ci`. | N/A (continuous variables). | `ref_path` (reference group column path via `get_ref_info()`), `paired` (`TRUE`/`FALSE`), `paired_by` (subject ID variable for paired t-test), `conf_level` (via `...` to `safe_t_test()`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_summarize_aval_chg_diff_j`** | `afun` | Yes | 3-column analysis presentation (col 1 = `n/N (%)`, col 2 = raw/baseline mean & CI, col 3 = change from baseline mean & CI, plus optional difference columns). Supports descriptive stats or ANCOVA. | Specified per column section: `col1`: `count_denom_frac`, `count_frac`, `count`; `col23`: `mean_ci_3d` (arithmetic or LS mean); `coldiff`: `mean_diff_with_ci` (t-test or ANCOVA LS mean diff). | `denom`: `"N"` (records in split) or `".N_col"` (column total count from `alt_counts_df`). | `ancova` (`TRUE` for ANCOVA LS means, `FALSE` for descriptive/t-test), `comp_btw_group`, `ref_path`, `multivars` (default `c("AVAL", "AVAL", "CHG")`), `method_combo` (`"contrasts"` or `"collapse"` for combined arms), `weights_combo`, `weights_emmeans`, `d` (decimal precision integer or column name for varying precision), `interaction_y`, `interaction_item`, `conf_level`, `variables`, `.stats`, `.formats`, `.formats_fun`, `format_na_str`. |
| **`a_summarize_ancova_j`** | `afun` | Yes | Comprehensive ANCOVA summary function supporting single and combined treatment groups via linear contrasts or model collapsing. | Descriptive: `n`, `sum`, `mean`, `sd`, `se`, `mean_sd`, `mean_se`, `mean_ci`, `mean_ci_3d`, `median`, `mad`, `median_ci`, `quantiles`, `range`, `geom_mean`, etc. ANCOVA: `n_fit`, `lsmean`, `lsmean_se`, `lsmean_ci`, `lsmean_diff`, `lsmean_diff_ci`, `lsmean_diff_with_ci`, `pval`. Defaults: `n`, `mean_sd`, `median`, `range`, `quantiles`, `lsmean_se`, `lsmean_ci`, `lsmean_diff_with_ci`, `pval`. | N/A (continuous variables). | `ref_path` (reference arm path), `variables` (list with `arm` and `covariates`), `method_combo` (`"contrasts"` for contrast derivation or `"collapse"` for pooled refit), `weights_combo` (contrast weights, e.g. `"equal"`), `weights_emmeans` (`"equal"`, `"proportional"`, `"counterfactual"`), `conf_level`, `interaction_y`, `interaction_item`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_summarize_mmrm`** | `afun` | Yes | Dynamic tabulation of Mixed Model for Repeated Measures (MMRM) results within tables split by treatment arms and visits. | `n`, `adj_mean_se`, `adj_mean_ci`, `adj_mean_est_ci`, `diff_mean_se`, `diff_mean_ci`, `diff_mean_est_ci`, `change`, `p_value`. | N/A (repeated measures model). | `variables` (list with `arm`, `visit`, `id`, `covariates`, `response`), `ref_levels` (named list with reference `visit` and reference `arm`), `cor_struct` (`"toeplitz"`, `"unstructured"`, `"ar1"`, etc.), `conf_level`, `alternative` (`"two.sided"`, `"less"`, `"greater"`), `show_relative` (`"reduction"`, `"increase"`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_summarize_mmrm_with_exclude`** | `afun` | Yes | Wrapper around `a_summarize_mmrm()` that excludes specified row split levels (e.g. baseline visit) from MMRM estimation. | Identical to `a_summarize_mmrm()`. | N/A. | `exclude_levels` (named list of row split levels to exclude), plus all parameters of `a_summarize_mmrm()`. |
| **`a_lsmeans`** | `afun` | Yes | Formatted analysis function for tabulating Least Squares Means and contrasts (from `fit_mmrm_j()` or `fit_ancova()`). | `n`, `adj_mean_se`, `adj_mean_ci`, `adj_mean_est_ci`, `diff_mean_se`, `diff_mean_ci`, `diff_mean_est_ci`, `change`, `p_value`. | N/A. | `ref_path` (reference arm path), `conf_level`, `alternative` (`"two.sided"`, `"less"`, `"greater"`), `show_relative`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_rbmi_lsmeans`** | `afun` | Yes | Tabulates pooled Least Squares Means, differences, and CIs from Reference-Based Multiple Imputation (RBMI) via Rubin's rules. | `adj_mean_se`, `adj_mean_ci`, `diff_mean_se`, `diff_mean_ci`, `change`, `p_value`, `additional_title_row`. | N/A (pooled multiple imputation results). | `ref_path` (reference arm column path), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_summary_j_with_exclude`** | `afun` | No | Dataframe-level wrapper around `tern::a_summary()` that excludes specified row split levels from producing numeric summaries. | All statistics from `tern::a_summary()`: `n`, `sum`, `mean`, `sd`, `se`, `mean_sd`, `mean_se`, `mean_ci`, `median`, `mad`, `median_ci`, `quantiles`, `iqr`, `range`, `min`, `max`, `median_range`, `cv`, `geom_mean`, `geom_sd`, `geom_mean_sd`, `geom_mean_ci`, `geom_cv`, `mean_diff_with_ci`. | N/A (continuous numeric variable summary). | `exclude_levels` (named list of split levels to exclude), `.var` (variable to extract and summarize), and all parameters of `tern::a_summary()` via `...`. |
| **`a_summary_subset`** | `afun` | No | Wrapper around `tern::a_summary()` that evaluates a logical filter expression (`filter_expr`) to subset `df` before summarization. Supports varying precision. | All statistics from `tern::a_summary()`. | N/A (continuous numeric variable summary). | `filter_expr` (single logical expression evaluated within `df`, e.g. `expression(AGE > 30)`), `formats_var` (varying decimal precision specification), and all `tern::a_summary()` parameters via `...`. |
| **`c_summary_subset_label`** | `cfun` | No | Content row function wrapper around `tern::a_summary()` with optional row subsetting (`filter_expr`) and custom label prepending. | All statistics from `tern::a_summary()`. | N/A. | `filter_expr`, `label` (prepended label string), `label_indent_mod`, and all `tern::a_summary()` parameters. |
| **`a_sum_ratio_j`** | `afun` | No | Computes sums and ratios of sums (e.g. cumulative dose received vs. planned, optionally de-duplicated by subject ID). | `sum` *(default)*, `sum_unique`, `ratio`, `ratio_unique`. | `denom_by`: name of column in `df` whose sum serves as denominator in `ratio` and `ratio_unique`. | `denom_by` (required for ratio stats), `id_var` (subject ID variable, required for `*_unique` stats), `.stats`, `.formats`, `.labels`, `.indent_mods`, `na_str`. |
| **`a_proportion_ci_logical`** | `afun` | No | Formatted analysis function computing proportion confidence intervals for a logical vector. | `prop_ci` (confidence interval in percent, returned as formatted `rcell`). | Derived from `nrow(.alt_df)`. | `conf_level` (default `0.95`), `method` (all methods in `tern::s_proportion`: `"wald"`, `"clopper-pearson"`, `"wilson"`, `"wilsonc"`, `"strat_wilson"`, `"agresti-coull"`, `"jeffreys"`), `formats` (`list(prop_ci = ...)`). |
| **`a_proportion_ci_factor`** | `afun` | No | Formatted analysis function computing proportion confidence intervals for levels of a factor variable. | `prop_ci` (confidence interval in percent for each factor level). | Derived from `nrow(.alt_df)`. | Identical to `a_proportion_ci_logical()`. |
| **`a_cond_proportion_j`** | `afun` | No | Adaptive response proportion and CI estimation, automatically selecting between exact (Clopper-Pearson) and asymptotic (Wald) methods based on count thresholds. | `n_prop` (count of responders and proportion, `xx / xx (xx.x%)`), `prop_ci` (proportion CI with dynamic method label). | `denom`: `"n"` *(default, cell record count)* or `"N_row"` (total row split record count). | `method_scope` (`"cell"` for cell-level decision, `"row"` for row-level pooled decision), `num_limit` (numerator threshold triggering exact CI, default `0`), `denom_limit` (denominator threshold triggering exact CI, default `10`), `conf_level` (default `0.95`), `long` (verbose method explanation in label), `na.rm`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_proportion_diff_j`** | `afun` | Yes | Estimates proportion differences (risk difference) and CI vs. reference group (`ref_path`) for a logical response variable. | `diff`, `diff_ci`, `diff_est_ci` (`"% Difference (conf_level CI)"`). | Responders out of total observations in group/cell. | `ref_path` (column path to reference group), `method` (`"wald"`, `"waldcc"`, `"cmh"`, `"ha"`, `"newcombe"`, `"newcombecc"`, `"strat_newcombe"`, `"strat_newcombecc"`, `"cmh_sato"`, `"cmh_mn"`, `"uncond_exact_diff"`), `variables = list(strata = ...)` (stratification factors), `conf_level`, `label`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_proportion_diff_mf`** | `afun` | Yes | Proportion difference with automated method selection based on the Mantel-Fleiss (MF) criterion: uses CMH with Sato variance when $\text{MF} \ge \text{threshold}$, otherwise exact unconditional difference. | `diff`, `diff_ci`, `diff_est_ci`. | Responders out of total observations in group/cell. | `mf_method` (`"cmh"`, `"cmh_sato"`, `"cmh_mn"`), `mf_threshold` (threshold for asymptotic CMH vs. exact, default `5`), `exact_footnote` (`RefFootnote` added when exact method triggers), `val` (response level), `variables = list(strata = ...)`, `conf_level`, `na.rm`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_test_proportion_diff`** | `afun` | Yes | Hypothesis test for the difference between two proportions vs. reference group (`ref_path`). | `pval` (p-value of proportion difference test). | Responders out of total observations in comparison groups. | `ref_path` (reference arm path), `method` (`"chisq"`, `"cmh"`, `"cmh_wh"`, `"fisher"`, `"schouten"`), `variables = list(strata = ...)`, `alternative` (`"two.sided"`, `"less"`, `"greater"`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_test_proportion_diff_mf`** | `afun` | Yes | Test of proportion difference with method selection based on Mantel-Fleiss criterion: CMH test when $\text{MF} \ge \text{threshold}$, otherwise Fisher's exact test. | `pval` (p-value). | Responders out of total observations in comparison groups. | `mf_threshold` (default `5`), `exact_footnote` (`RefFootnote` added for exact test), `val`, `variables = list(strata = ...)`, `alternative`, `na.rm`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_combo_prop_diff_pval_mf`** | `afun` | Yes | Combined analysis presenting proportions, treatment-vs-control risk differences, and p-values across distinct columns within the same row, with Mantel-Fleiss method selection. | Specified via named list: `prop`: `count_denom_fraction` *(default)*, `count_fraction`, `count`; `diff`: `diff_est_ci` *(default)*, `diff`, `diff_ci`; `pval`: `pval` *(default)*. | Responders out of total records in column/cell. | `colpaths` (named list mapping `"prop"`, `"diff"`, and `"pval"` to column paths), `trt_var`, `ctrl_group`, `val` (response level in factor `.var`), `mf_threshold` (default `5`), `exact_footnote`, `variables = list(strata = ...)`, `conf_level`, `alternative`, `.stats`, `.formats`. |
| **`a_odds_ratio_j`** | `afun` | Yes | Estimates Odds Ratio (OR), CI, and p-value vs. reference group. Supports unstratified logistic regression, conditional logistic regression, and CMH. | `or_ci` (OR estimate and CI: `est`, `lcl`, `ucl`), `pval` (p-value), `n_tot` (total sample size). | Total sample size across comparison groups (`n_tot`). | `method` (`"exact"`, `"approximate"`, `"efron"`, `"breslow"`, `"cmh"`), `ref_path`, `variables` (list with `arm` and `strata`), `conf_level`, `na_if_no_events`, `groups_list`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_relative_risk`** | `afun` | Yes | Estimates stratified Relative Risk (RR), CI, and p-value using the CMH method with Greenland & Robins (1985) variance estimation. | `rel_risk_ci` (RR estimate and CI: `est`, `lcl`, `ucl`), `pval` (p-value). | Responders out of total observations in comparison groups. | `ref_path`, `variables = list(strata = ...)` (stratification factors required), `conf_level`, `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_cmhrms_j`** | `afun` | Yes | Performs Cochran-Mantel-Haenszel (CMH) Row Mean Scores test comparing ordered categorical response levels between treatments and reference group. | `pval` (row mean scores test p-value). | N/A (hypothesis testing p-value). | `ref_path`, `variables` (list with `arm` and optional `strata`), `collapse_combo` (whether multiple arm levels in `df` are collapsed into `"Combined"`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`a_cmhrms_j_with_exclude`** | `afun` | Yes | Wrapper around `a_cmhrms_j()` that excludes specified row split levels from producing the CMH Row Mean Scores test. | `pval`. | N/A. | `exclude_levels` (named list of row split levels to exclude), plus all parameters of `a_cmhrms_j()`. |
| **`prop_table_afun`** | `afun` | No | Formatted analysis function for proportion tables with column splits prepared via `prop_split_fun()`. Shows counts, percentages, and cumulative percentages across sub-columns. | In `n` col: category counts (+ optional Total); in `percent` col: category percentages; in `cum_percent` col: cumulative percentages. | Total records in column slice (`length(x)`). | `add_total_level` (`TRUE`/`FALSE`), `formats` (named list for `n`, `percent`, `cum_percent`). |
| **`c_proportion_logical`** | `cfun` | No | Simple content function computing the count and proportion of `TRUE` values in a logical vector relative to column count. | `est_prop` (`c(num, num / denom)`). | `.N_col` (total column count). | `label_fstr` (`sprintf` string using `labelstr`), `format`. |
| **`a_event_free`** | `afun` | No | Time-point survival (event-free rate) analysis function. Extends `tern::s_surv_timepoint` with a combined 3D statistic (`event_free_ci`). | `event_free_ci` *(combined rate & CI)*, `pt_at_risk`, `event_free_rate`, `rate_se`, `rate_ci`. | N/A (Kaplan-Meier survival estimation at time point). | `time_point` (numeric milestone time), `time_unit` (e.g. `"month"`, `"week"`), `is_event` (event indicator variable), `percent` (`TRUE` for `%`, `FALSE` for 0–1), `control` (`tern::control_surv_timepoint()`), `.stats`, `.formats`, `.labels`, `.indent_mods`. |
| **`tefos03_afun`** | `afun` *(internal)* | No | Internal analysis function used by `summarize_coxreg_multivar()` to extract multivariable Cox model coefficients, hazard ratios, CIs, and p-values. | Model fit col: `coef_se`, `pval`; Hazard ratio col: `hr_est`, `hr_ci`. | N/A (Cox proportional hazards regression). | `variables` (time, event, arm, covariates), `control` (`tern::control_coxreg()`), `formats`. |
| **`response_by_var`** | `afun` | No | Derives `count_denom_fraction` (`xx / xx (xx.x%)`) for records with response flag `"Y"` in `resp_var` across levels of `.var`, with denominator based on non-missing records. | `count_denom_fraction` (`c(num, denom, num / denom)`). | Non-missing observations of `.var` in input `df`. | `resp_var` (response flag variable), `id` (subject ID, default `"USUBJID"`), `.format`. |
| **`resp01_acfun`** | `afun` / `cfun` | N/A | Dual analysis and content function for `RESP01` response tables. Dispatches to counts/CIs in arm columns, and comparative stats (RR, OR, p-value) in Overall column. | In arm columns: `count_prop` (count & proportion), `prop_ci` (proportion CI); in Overall column: `comp_stat_ci` (RR or OR estimate and CI), `pval` (p-value). | Derived from `.alt_df` (`nrow(.alt_df)`). | `methods` (list specifying `comp_stat_ci`: `"rr"`, `"or_cmh"`, `"or_logistic"`; `pval`: `"fisher"`, `"chisq"`, or CMH; `prop_ci`: `"wald"`, `"clopper-pearson"`), `formats`, `include_comp` (factor levels included in comparisons), `arm`, `strata`, `conf_level`. |
| **`resp01_a_comp_stat_logical`** | `afun` | N/A | Computes comparative statistics (RR or OR) and p-values for logical response variables in `RESP01` response table layouts. | `comp_stat_ci`, `pval`. | Total group sizes in `df`. | `methods`, `formats`, `arm`, `strata`, `conf_level`, `include` (`TRUE`/`FALSE`). |
| **`resp01_a_comp_stat_factor`** | `afun` | N/A | Computes comparative statistics (RR or OR) and p-values across factor levels in `RESP01` response table layouts. | `comp_stat_ci`, `pval` per factor level in `include`. | Total group sizes in `df`. | `include` (character vector of factor levels), `methods`, `formats`, `arm`, `strata`, `conf_level`. |
| **`resp01_counts_cfun`** | `cfun` | N/A | Content function for subgroup sample size counts from `.alt_df` in `count_prop` columns for `RESP01` response tables. | `count` (`nrow(.alt_df)`). | Subgroup sample count from `.alt_df`. | `label_fstr` (`sprintf` string using `labelstr`). |
| **`cmp_cfun`** | `cfun` | N/A | Content function for study compliance tables split across three columns: Expected visits (`N`), Received visits (`n (%)`), and Missing visits (`n (%)`). | Expected col: `n_exp`; Received col: `n_rec`, `rec_percent`; Missing col: `n_mis`, `mis_percent`. | Denominator for Received and Missing is derived dynamically from Expected visits count (`sum(df[[variables$expected]])`). | `variables` (list with column names for `expected`, `received`, `missing`), `formats` (list containing `count_percent` format). |
| **`lsmeans_wide_cfun`** | `cfun` | N/A | Content function for wide-format ANCOVA LS means table layouts, organizing estimates, MSE, df, contrasts, and p-values across multi-tiered columns. | Reference/Testing cols: `treatment`, `n`, `lsmean`, `se`; Variance col: `mse`, `df`; Comparison col: `lsmean` (difference in LS means), `se`, `ci`, `pval`. | N/A. | `ref_level`, `treatment_levels`, `pval_sided` (`"2"`, `"1"`, `"-1"`), `conf_level`, `formats`. |
| **`c_row_counts`** | `cfun` *(internal)* | N/A | Content function to count rows in current data split (`nrow(df)`). Wrapped by `summarize_row_counts(alt_counts = FALSE)`. | `count` (`nrow(df)`). | N/A. | `label_fstr` (`sprintf` string using `labelstr`, default `"%s"`). |
| **`c_row_counts_alt`** | `cfun` *(internal)* | N/A | Content function to count rows in alternative dataset (`nrow(.alt_df)`). Wrapped by `summarize_row_counts(alt_counts = TRUE)`. | `count` (`nrow(.alt_df)`). | N/A. | `label_fstr`. |
| **`ac_blank_line`** | `afun` / `cfun` | N/A | Formatted analysis and content function producing a blank row gap in table layouts with invisible `NA` string (`""`). **⚠️ Legitimate uses only**: (1) as a visual spacer `analyze()` between two substantive analysis sections (or via `insert_blank_line()`), or (2) as a `cfun` in `summarize_row_groups(cfun = ac_blank_line)` paired with `split_rows_by(..., child_labels = "visible")` to insert a blank line between a visible facet label row and its child rows. **Never use as the sole terminating `afun` of a `split_rows_by()` chain** — this is the No-Op `analyze()` Anti-Pattern (see [`row_struct_layout.md`](./row_struct_layout.md)). | Single blank cell (`NA_real_` formatted as `""`). | N/A. | `labelstr` (row label string, default `""`). |

> [!CAUTION]
> **Deprecated `junco` Functions — Do Not Select**
> - `junco::a_coxph_hr` was deprecated in `junco 0.1.7` (`lifecycle::deprecate_stop`). Use **`tern::coxph_pairwise()`** / `tern:::a_coxph_pairwise()` instead.
> - `junco::a_kaplan_meier` was deprecated in `junco 0.1.7` (`lifecycle::deprecate_stop`). Use **`tern::a_surv_time()`** instead.

---

## 5. Detailed Selection & Configuration Guide by Domain

### 5.1 Frequency, Categorical Counts, Subgroups & Multitiered Layouts

#### 1. `a_freq_j` and `a_freq_j_with_exclude`
`a_freq_j` is the primary workhorse in `junco` for adverse event (AE), concomitant medication, medical history, disposition, and categorical demographic tables.
- **Dual Role**: Use in `analyze(..., afun = a_freq_j)` for category/term rows AND in `summarize_row_groups(..., cfun = a_freq_j)` for group header rows (e.g., System Organ Class `AEBODSYS`, subgroup headers, or overall event flags).
- **Selecting `.stats`**:
  - `"count_unique_denom_fraction"` *(default)*: `xx / xx (xx.x%)` — unique subjects, denominator, and percentage. Standard for subgroup tables where the denominator varies by row group.
  - `"count_unique_fraction"`: `xx (xx.x%)` — unique subjects and percentage. Standard for main AE / ConMed tables where `(N=xx)` is in the column header.
  - `"count_unique"`: `xx` — unique subject count only.
  - `"count"`: `xx` — raw event/record count (not deduplicated by subject).
  - `"unique_fraction"`: `(xx.x%)` — percentage of unique subjects only.
  - `"n_df"`, `"n_altdf"`, `"n_rowdf"`, `"n_parentdf"`, `"denom"`: standalone count/denominator values (e.g., `.stats = "n_altdf"` on a subgroup header row to display subgroup sample sizes `N`).
- **Selecting `denom` and `denom_by`**:
  - `denom = "N_col"` *(default)* or `denom = "n_altdf"`: divides by the column subject count from `alt_counts_df`.
  - `denom = "n_altdf", denom_by = "SEX"`: divides by the number of subjects in `alt_counts_df` matching both the current column AND the current `SEX` row split level.
  - `denom = "n_df"`: divides by records in the current `(row, col)` cell slice.
  - `denom = "n_rowdf"` / `"n_parentdf"` / `"N_colgroup"`: divides by row total, parent row total, or column group total.
- **Configuring Risk Differences (`riskdiff = TRUE`)**:
  - Automatically detects risk difference columns via `ref_path` (e.g., `c("colspan_trt", " ", "TRT01A", "Placebo")`).
  - In main arm columns, displays the requested `.stats`; in risk difference columns, blanks out non-comparative stats and displays the percentage point difference and CI (`xx.x (xx.x, xx.x)`).
  - Supports 11 `method` options: `"wald"`, `"waldcc"`, `"cmh"`, `"ha"`, `"newcombe"`, `"newcombecc"`, `"strat_newcombe"`, `"strat_newcombecc"`, `"cmh_sato"`, `"cmh_mn"`, `"uncond_exact_diff"`.
- **Column Restriction (`restr_columns`)**:
  - A KEEP list of leaf column labels: columns whose leaf label (case-insensitive match on `tail(.spl_context$cur_col_split_val, 1)`) is not in `restr_columns` are blanked out. Use when a single `analyze()` or `summarize_row_groups()` call should only populate a subset of columns (e.g., `restr_columns = c("N")` to populate only the `N` column in a shift table layout while blanking baseline-level and Total columns).
  - Distinct from `riskdiff`: `riskdiff` toggles behavior between arm columns and risk-difference columns; `restr_columns` restricts output to an arbitrary subset of leaf columns by name.
- **Filtering, Level Manipulation & Relabeling**:
  - **⚠️ Data scope constraint for derived variables**: `a_freq_j` internally references `.df_row` (for level enumeration and row-facet denominators) and `.alt_df_full` (for `denom = "n_altdf"` subject counts). Any variable passed as `.var` — or referenced by `denom_by`, `val`, `excl_levels`, or `new_levels` — must exist in the dataset(s) passed to `build_table()`, not just in a `df` slice mutated inside a custom afun. Variables created via `mutate()` inside an afun exist only in the `df` column-slice; they are invisible to `.df_row`, `.alt_df_full`, `.spl_context$full_parent_df`, and all other data.frame special arguments. Always pre-compute derived variables (flags, recoded factors, computed numerics) in data prep on both `df` and `alt_counts_df` when needed. See Building Block 7 in `[custom_afuns.md](./custom_afuns.md)`.
  - Single-flag summary row: `analyze("TRTEMFL", afun = a_freq_j, extra_args = list(val = "Y", label = "Subjects with >=1 AE", ...))` tabulates only level `"Y"` and labels the row `"Subjects with >=1 AE"`.
  - Level control: `drop_levels`, `excl_levels`, `new_levels`, `new_levels_after`, `addstr2levs`.
  - `label_map`: dataframe supporting 4 translation modes (single-variable, multi-variable, conditional on row split, or conditional multi-variable).
- **Excluding Specific Row Facets (`a_freq_j_with_exclude`)**:
  - Pass `exclude_levels = list(SPLIT_VAR = c("Level1", "Level2"))` in `extra_args` to suppress rows for specific row split levels.

#### 2. `a_freq_combos_j` (Cumulative / Cycle Period Columns)
Select `a_freq_combos_j` when columns represent time intervals or treatment cycles paired with cumulative/total period columns created via `combosdf`:
- Uses `filter_var` and `flag_var` to prevent double-counting subjects across individual period facets while `do_not_filter` preserves cumulative counts in overall/total facets.

#### 3. `a_freq_subcol_j` (Conditional Sub-Column Event Filtering)
Select `a_freq_subcol_j` when each treatment arm is subdivided into sub-columns where specific sub-columns represent restricted subsets of events (e.g., `"All AEs"`, `"Drug-Related AEs"`, `"Serious AEs"`):
- Inspects the column path for `subcol_split` and filters `df` to `df[[subcol_var]] == subcol_val` only inside matching sub-columns.

#### 4. `a_freq_resp_var_j` (Binary `"Y"`/`"N"` Response Across Categories)
Select `a_freq_resp_var_j` when summarizing a binary `"Y"`/`"N"` flag variable (`resp_var`, e.g., `"TRTEMFL"`) across the levels of a categorical variable passed to `analyze()` (e.g., `SEX` or `AGEGR1`), displaying `xx / xx (xx.x%)` in arm columns and optional risk differences (`riskdiff = TRUE`) in difference columns.

#### 5. `a_maxlev` (Worst / Maximum Ordered Factor Level per Subject)
Select `a_maxlev` when each subject must be counted only once at their highest level of an ordered factor (e.g., maximum CTCAE toxicity grade `AETOXGR` or worst AE severity `AESEV`):
- **As a `cfun` in `summarize_row_groups()`**: Pass `any_level = TRUE` (and optionally `any_level_exclude = "Missing"`) to count unique subjects with any non-missing grade in that organ class/term.
- **As an `afun` in `analyze()`**: Pass `any_level = FALSE` to tabulate the distribution of each subject's worst grade.
- **⚠️ Does not support difference columns**: `a_maxlev` does not support `riskdiff`/`ref_path` and cannot be used directly in layouts with difference columns (`grouped_cols_w_diffs` or manual risk-difference column branches). It will throw `"'subset' must be logical"` inside diff columns. Two workaround patterns are available:
  - **Workaround 1 (Pre-computed factor + `a_freq_j`)**: Pre-compute the worst grade as a factor variable in data prep and use `a_freq_j` on the factor levels. This works seamlessly with `riskdiff = TRUE`.
  - **Workaround 2 (Column-dispatch wrapper)**: Wrap `a_maxlev` in a column-conditional dispatch afun (Pattern 2A from `[custom_afuns.md](./custom_afuns.md)`) that calls `a_maxlev` in treatment columns and emits blank cells in difference columns.

#### 6. `a_two_tier` and `a_three_tier` (Selective Drill-Down Without Structural Sub-Splits)
Select `a_two_tier` or `a_three_tier` when a shell displays categories of an outer variable (`.var`), and **only selected categories** (`drill_down_levs`) expand into indented detail breakdown rows (`inner_var`), such as Disposition tables expanding `"DISCONTINUED"` into discontinuation reasons (`DCSREAS`) while `"COMPLETED"` remains a single line:
- `a_two_tier`: Takes `inner_var`, `drill_down_levs`, `use_all_levels`, `grp_fun` (default `a_freq_j`), and `detail_fun` (default `a_freq_j`).
- `a_three_tier`: Extends to 3 nested levels (`inner_var1`, `inner_var2`).

---

### 5.2 Exposure, Patient-Years, and Incidence Rates (EAIR)

#### 1. `a_eair100_j` and `a_eair_strat_j`
Select `a_eair100_j` for unstratified Exposure-Adjusted Incidence Rates (or `a_eair_strat_j` for study-size weighted stratified EAIR across `strata`):
- **Rate Calculation**: $\text{EAIR} = \text{num\_p\_year} \times \frac{n_{\text{event}}}{\text{total person-years}}$ (default `num_p_year = 100`).
- **Time-at-Risk Logic**: Subjects without the event contribute `fup_var` (in years). Subjects with the event (`occ_var == "Y"`) contribute time to first event (`occ_dy / 365.25`).
- **Supported `.stats`**: `eair` *(default)*, `n_event`, `person_years`, `n_eair`, `eair_ci`, `eair_est_ci`, `eair_n`, `eair_n_py`, and in difference columns: `eair_diff_est_ci`, `eair_diff_est`, `eair_diff_ci` (or `*_strat_*` equivalents in `a_eair_strat_j`).
- **CI Methods**:
  - `conf_type_eair`: `"exact"` (Poisson exact), `"normal"`, `"normal_log"`, `"byar"`.
  - `conf_type_diff`: `"wald"`, `"waldcc"`, `"mn"` (Miettinen-Nurminen score), `"scas"` (skewness-corrected asymptotic score).

#### 2. `a_patyrs_j`
Select `a_patyrs_j` to display a `"Total patient-years"` summary row (summing `.var` across unique subjects `id` from `source = "alt_df"` or `"df"`). Passing `ref_path` automatically blanks out the cell in risk difference columns.

#### 3. `a_summarize_ex_j`
Select `a_summarize_ex_j` for Study Drug Exposure tables (duration of exposure in days/months, total dose, dose intensity):
- Default `.stats`: `mean_sd`, `median`, `range`, `quantiles`, `total_subject_years`; in difference columns (`comp_btw_group = TRUE`): `diff_mean_est_ci`.
- Supports unit conversion via `daysconv` (`1` for days, `30.4375` for months) and between-group comparisons via two-sample t-test (`ancova = FALSE`) or ANCOVA (`ancova = TRUE`).

---

### 5.3 Continuous Variables, Difference in Means, ANCOVA, MMRM & Summaries

#### 1. `tern::a_summary`, `a_summary_j_with_exclude`, `a_summary_subset`, and `c_summary_subset_label`
- **`tern::a_summary`**: Standard descriptive summary function for continuous (`n`, `sum`, `mean`, `sd`, `se`, `mean_sd`, `mean_se`, `mean_ci`, `median`, `mad`, `median_ci`, `quantiles`, `iqr`, `range`, `min`, `max`, `median_range`, `cv`, `geom_mean`, `geom_sd`, `geom_mean_sd`, `geom_mean_ci`, `geom_cv`, `mean_diff_with_ci`) and categorical (`count`, `count_fraction`, `n_blq`) variables.
- **`a_summary_j_with_exclude`**: Select when you need `tern::a_summary` but must suppress output for specific row split levels (`exclude_levels = list(AVISIT = "BASELINE")`).
- **`a_summary_subset` / `c_summary_subset_label`**: Select when the summary must be computed on a filtered subset of `df` within each cell via `filter_expr = expression(...)` (with optional `formats_var` for dynamic precision, or `label` prepending in `c_summary_subset_label`).

#### 2. `column_stats` (Statistics-in-Columns Layout)
Select `column_stats` when summary statistics (`N`, `Mean`, `SD`, `SE`, `Med`, `Min`, `Max`, `Base Mean (SD)`) are arranged **horizontally as columns** under multi-variable headers (`AVAL`, `BASE`, `CHG`) while visits (`AVISIT`) are arranged **vertically as rows**:
- Must be paired with `split_cols_by_multivar(...) |> split_cols_by("STUDYID", split_fun = make_split_fun(post = list(stats_in_cols_setup())))`.
- Automatically blanks out `BASE` and `CHG` statistics on baseline/screening visits specified in `exclude_visits = toupper(c("Baseline", "Screening"))`.
- Supports parameter-specific decimal precision or significant figures via `formats_var = "fmt_d"`.

#### 3. `a_diff_means`
Select `a_diff_means` to compute independent two-sample t-test (`paired = FALSE`) or paired t-test (`paired = TRUE, paired_by = "USUBJID"`) mean differences, SEs, and CIs against `ref_path`.

#### 4. `a_summarize_aval_chg_diff_j` (3-Column Lab / Vital Signs Layout)
Select `a_summarize_aval_chg_diff_j` when each treatment arm has 3 sub-columns created by `split_cols_by_multivar(c("AVAL", "AVAL", "CHG"))`:
1. Column 1 (`col1`): `n/N (%)` (`count_denom_frac`, `count_frac`, or `count`).
2. Column 2 (`col23` first): Raw/Visit Mean (95% CI) (`mean_ci_3d`).
3. Column 3 (`col23` second): Change from Baseline Mean (95% CI) (`mean_ci_3d`) — automatically blank when `CHG` is `NA` (e.g., at `SCREENING`/`BASELINE`).
4. Optional Difference Columns (`coldiff`, when `comp_btw_group = TRUE`): Difference in Mean Change (95% CI) (`mean_diff_with_ci`), blanked out at `SCREENING`/`BASELINE`.
- Pass `ancova = FALSE` for descriptive arithmetic means + t-test differences, or `ancova = TRUE` for ANCOVA LS means + contrasts.
- Set `child_labels = "hidden"` on `split_rows_by("AVISIT", ...)` and pass a dummy variable (e.g., `"STUDYID"`) to `analyze("STUDYID", afun = a_summarize_aval_chg_diff_j, extra_args = ...)` because `a_summarize_aval_chg_diff_j` pulls the analysis variables from `multivars` and labels rows by the visit name.

#### 5. `a_summarize_ancova_j`
Select `a_summarize_ancova_j` when a vertical summary table needs both descriptive statistics (`n`, `mean_sd`, `median`, `range`, `quantiles`) and ANCOVA model estimates (`lsmean_se`, `lsmean_ci`, `lsmean_diff_with_ci`, `pval`), especially when the layout includes **virtual combined treatment arms** (`add_combo_levels`):
- `method_combo = "contrasts"` derives combined-arm LS means and differences via linear contrasts from the full unpooled model (`weights_combo = "equal"`).
- `method_combo = "collapse"` refits the ANCOVA model after pooling the combined arms.

#### 6. `a_summarize_mmrm`, `a_summarize_mmrm_with_exclude`, `a_lsmeans`, and `a_rbmi_lsmeans`
- **`a_summarize_mmrm` / `a_summarize_mmrm_with_exclude`**: Select when fitting and tabulating an MMRM model dynamically across `AVISIT` and `ARM` facets (`variables = list(arm = ..., visit = ..., id = ..., covariates = ..., response = ...)`). Automatically handles reference visits and reference arms (`ref_levels`).
- **`a_lsmeans`**: Select when the MMRM or ANCOVA model has already been pre-fitted via `fit_mmrm_j()` or `fit_ancova()` and merged into the data frame.
- **`a_rbmi_lsmeans`**: Select when tabulating Reference-Based Multiple Imputation (RBMI) results pooled via Rubin's rules.

#### 7. `a_sum_ratio_j`
Select `a_sum_ratio_j` when tabulating cumulative sums (`sum`, `sum_unique`) and ratios of sums (`ratio`, `ratio_unique`), where the denominator is the sum of another column specified via `denom_by` (e.g., total dose administered divided by total dose planned).

---

### 5.4 Proportions, Risk Differences, Mantel-Fleiss, Odds Ratios, Relative Risk & CMH

1. **Confidence Intervals for Proportions (`a_proportion_ci_logical`, `a_proportion_ci_factor`, `a_cond_proportion_j`)**:
   - Use `a_proportion_ci_logical` or `a_proportion_ci_factor` when a row only needs the formatted proportion CI (`prop_ci`) using a fixed `method` (`"wald"`, `"clopper-pearson"`, `"wilson"`, `"wilsonc"`, `"strat_wilson"`, `"agresti-coull"`, `"jeffreys"`).
   - Use **`a_cond_proportion_j`** when the statistical analysis plan requires **adaptive CI method selection**: it automatically switches from asymptotic Wald to exact Clopper-Pearson whenever responders $\le \text{num\_limit}$ (default `0`), non-responders $\le \text{num\_limit}$, or denominator $< \text{denom\_limit}$ (default `10`), evaluated per cell (`method_scope = "cell"`) or across the entire row (`method_scope = "row"`).
2. **Risk Difference & Hypothesis Tests (`a_proportion_diff_j`, `a_test_proportion_diff`)**:
   - Use `a_proportion_diff_j` for standalone risk difference rows (`diff`, `diff_ci`, `diff_est_ci`) against `ref_path`.
   - Use `a_test_proportion_diff` for standalone p-value rows (`chisq`, `cmh`, `cmh_wh`, `fisher`, `schouten`) against `ref_path`.
3. **Mantel-Fleiss (MF) Criterion Suite (`a_proportion_diff_mf`, `a_test_proportion_diff_mf`, `a_combo_prop_diff_pval_mf`)**:
   - Select these functions when the SAP specifies switching between stratified CMH and exact methods based on the **Mantel-Fleiss criterion** ($\text{MF} \ge \text{mf\_threshold}$, default `5`):
     - When $\text{MF} \ge 5$: uses CMH with Sato variance (`a_proportion_diff_mf`) or CMH test (`a_test_proportion_diff_mf`).
     - When $\text{MF} < 5$: switches to unconditional exact difference (`a_proportion_diff_mf`) or Fisher's exact test (`a_test_proportion_diff_mf`) and attaches `exact_footnote` (`RefFootnote`).
   - Use **`a_combo_prop_diff_pval_mf`** when a single table row presents proportions (`prop`), risk differences (`diff`), and p-values (`pval`) across separate column paths mapped via `colpaths`.
4. **Odds Ratios, Relative Risk & Ordered Categorical CMH (`a_odds_ratio_j`, `a_relative_risk`, `a_cmhrms_j`)**:
   - **`a_odds_ratio_j`**: Estimates OR, CI, and p-value (`or_ci`, `pval`, `n_tot`) via logistic regression, conditional logistic regression (`"exact"`, `"approximate"`, `"efron"`, `"breslow"`), or `"cmh"`.
   - **`a_relative_risk`**: Estimates stratified Relative Risk (`rel_risk_ci`, `pval`) via CMH with Greenland & Robins variance (`variables = list(strata = ...)` required).
   - **`a_cmhrms_j` / `a_cmhrms_j_with_exclude`**: Computes the CMH Row Mean Scores test p-value (`pval`) for ordered categorical responses across arms and strata.
5. **Proportion Tables & Simple Logical Proportions (`prop_table_afun`, `c_proportion_logical`, `response_by_var`)**:
   - **`prop_table_afun`**: Paired with `prop_split_fun()` to populate `n`, `percent`, and `cum_percent` sub-columns.
   - **`c_proportion_logical`**: Lightweight `cfun` computing count and proportion of `TRUE` relative to `.N_col`.
   - **`response_by_var`**: Computes `count_denom_fraction` (`xx / xx (xx.x%)`) for `resp_var == "Y"` across levels of `.var` with denominator equal to non-missing observations of `.var` in `df`.

---

### 5.5 Time-to-Event & Survival Analysis

- **`a_event_free`**: Select for milestone survival / event-free rate rows at `time_point` (e.g., 6-month or 12-month survival), extending `tern::s_surv_timepoint` with the 3D combined statistic `event_free_ci` (`xx.xx (xx.xx, xx.xx)`).
- **`tefos03_afun`**: Internal `afun` used by `summarize_coxreg_multivar()` to populate multivariable Cox regression model fit (`coef_se`, `pval`) and hazard ratio (`hr_est`, `hr_ci`) columns.
- **Kaplan-Meier & Pairwise Cox HR**: Use `tern::a_surv_time` and `tern::coxph_pairwise` (do not use deprecated `a_kaplan_meier` or `a_coxph_hr`).

---

### 5.6 Specialized Layouts (`RESP01`, Study Compliance, Wide LS Means, Row Counts & Blank Lines)

1. **`resp01_acfun`, `resp01_a_comp_stat_logical`, `resp01_a_comp_stat_factor`, `resp01_counts_cfun`**:
   - Select for `RESP01` response tables where `resp01_split_fun_fct` splits treatment arm columns into `count_prop` and `prop_ci` while the `Overall` column displays `comp_stat_ci` (RR or OR) and `pval`.
2. **`cmp_cfun`**:
   - Select as `cfun` in `summarize_row_groups()` for 3-column Study Compliance layouts (`Expected`, `Received`, `Missing` visits), where Received and Missing percentages use Expected visits as the dynamic denominator.
3. **`lsmeans_wide_cfun`**:
   - Select as `cfun` for wide-format ANCOVA layouts displaying Reference Group (`n`, `lsmean`, `se`), Testing Group (`n`, `lsmean`, `se`), Variance (`mse`, `df`), and Comparison (`lsmean` diff, `se`, `ci`, `pval`) across columns.
4. **`c_row_counts`, `c_row_counts_alt`, `ac_blank_line`**:
   - `c_row_counts` / `c_row_counts_alt`: Count rows in `df` or `.alt_df` (wrapped by `summarize_row_counts()`).
   - `ac_blank_line`: Inserts a blank spacing row (`""`) as either a `cfun` or `afun`. **Only** for (1) visual spacers between substantive analysis sections, or (2) as `cfun` in `summarize_row_groups(cfun = ac_blank_line)` with `split_rows_by(..., child_labels = "visible")` to separate a visible facet label from its children. **Never** use as the sole terminating `afun` of a `split_rows_by()` chain (No-Op Anti-Pattern).

---

## 6. Cross-Cutting Configuration Mechanisms in `junco`

### 6.1 Wiring Reference Groups via `ref_path` & `get_ref_info()`

Standard `rtables` `ref_group` only works within a single flat `split_cols_by()` call. In clinical tables with spanning headers (e.g., `"Active Study Agent"` vs `" "`) or non-nested Risk Difference columns, `junco` analysis functions locate the control arm using **`ref_path`**:

- **`ref_path` Format**: A character vector alternating `<split_variable_name>, <split_value_name>` down the column hierarchy to the reference leaf column:
  ```r
  # When using spanning header variable "colspan_trt" (where Placebo has span label " ") and arm variable "TRT01A":
  ref_path <- c("colspan_trt", " ", "TRT01A", "Placebo")

  # When using a flat single-level column split on "TRT01A":
  ref_path <- c("TRT01A", "Placebo")
  ```
- **Automatic Injection with `grouped_cols_w_diffs()`**: When you construct columns with `junco::grouped_cols_w_diffs(trt_map, ...)`, `ref_path` is automatically injected into the `extra_args` of the comparison columns, **do not pass it via `extra_args` to your `afun`/`cfun` in this case**. When building the column layout manually with `split_cols_by()`, pass `ref_path = ref_path` explicitly inside `extra_args` of your `afun`/`cfun`.

---

### 6.2 Dynamic & Parameter-Specific Decimal Precision (`formats_d` & `format_sigfig_j`)

Clinical laboratory (`ADLB`), vital signs (`ADVS`), and pharmacokinetic (`ADPC`) tables often require different decimal precision ($d$) for each parameter (`PARAMCD`) or variable. `junco` provides three ways to configure dynamic precision without writing custom `afun`s:

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
For PK/PD tables requiring $k$ significant figures instead of fixed decimals, pass `format_sigfig_j(k)` in `.formats` or inside `fmt_d_def` with `fmt_spec_df_d()`.

---

## 7. Canonical Layout Wiring Recipes

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

## 8. Diagnostic Checklist & Escalation Path

Before finalizing your `afun`/`cfun` selection, verify all five items:

1. **Counting Unit Match**: Does the shell count unique patients (`count_unique_fraction`), total events (`count`), worst grade per patient (`a_maxlev`), or exposure time (`a_eair100_j` / `a_patyrs_j`)?
2. **Denominator Match**: Is the percentage denominator the column arm total (`denom = "n_altdf"`), a row-subgroup-conditioned arm total (`denom = "n_altdf", denom_by = ...`), the cell sample size (`denom = "n_df"`), or the row total (`denom = "n_rowdf"`)? Is `alt_counts_df = adsl` passed to `build_table()`?
3. **Column Conditionality Match**: If the layout has non-nested comparison columns or multi-variable sub-columns, does the selected `afun`/`cfun` accept `ref_path` / `riskdiff` / `comp_btw_group` / `multivars` / `colpaths` to populate each column branch appropriately? **All `tern` functions and `junco::a_maxlev` lack built-in difference column support.** If the layout includes difference columns, either select a `junco` function that supports `riskdiff`/`ref_path` (check the "Diff Col" column in Sections 3–4), or wrap the selected function using the column-dispatch (Pattern 2A) or `get_ref_info()` bridge (Tier 3) patterns from `[custom_afuns.md](./custom_afuns.md)`.
4. **Row Conditionality Match**: Are baseline/screening visits or excluded factor levels properly suppressed via `exclude_visits`, `exclude_levels`, `a_*_with_exclude`, or `a_two_tier(drill_down_levs = ...)`?
5. **Post-Selection Escalation** (two independent decision points):
   - **A suitable function WAS identified** → Check the Section 3.1 routing table. If an expert usage skill exists for that function, activate it for detailed parameterization, gotchas, and worked examples. Otherwise, continue with the general reference in Sections 4–7 of this document.
   - **No suitable function was identified** → Escalate to `[custom_afuns.md](./custom_afuns.md)` (prioritizing `make_afun()`, `RowsVerticalSection` stacking/dispatch, and `get_ref_info()` over bespoke code) when the table shell requires:
     - A bespoke statistical calculation or custom cell structure not present in `junco` or `tern` (see Section 4 Master Catalog).
     - Arbitrary column-facet dispatch across heterogeneous sibling column branches not covered by `junco`'s built-in risk/mean difference, `a_summarize_aval_chg_diff_j`, `column_stats`, `a_combo_prop_diff_pval_mf`, `resp01_acfun`, `cmp_cfun`, or `lsmeans_wide_cfun` patterns.
     - Custom multi-ancestor denominator arithmetic inspecting `.spl_context$full_parent_df` beyond `n_parentdf` and `denom_by`.

