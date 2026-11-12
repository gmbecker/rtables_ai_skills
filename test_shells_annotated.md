# Test Table Shells — Annotated Reasoning Path Analysis

This document describes the rationale behind each test shell in `test_shells_bare.md`. For each shell, it identifies:
- Which sub-skills and reasoning stages are exercised
- Which specific decision points, contracts, or anti-patterns are probed
- What known or hypothesized weaknesses the shell is designed to expose

---

## Coverage Matrix

| Shell | Col Struct | Row Struct | Select Afuns | Custom Afuns | Sort/Prune | Key Reasoning Traps |
|-------|-----------|-----------|-------------|-------------|-----------|-------------------|
| T01   | Simple    | Stacked   | tern + junco | —           | —         | Mixed continuous/categorical afun selection; no-split stacked archetype |
| T02   | Spanning+RD | Subgroup nested | a_freq_j dual conditionality | — | Yes | denom_by, riskdiff toggle per row level, Contract 3 |
| T03   | Simple    | Selective drill-down | a_two_tier | Possibly | — | Two-tier pattern; temptation to use split_rows_by for DCSREAS |
| T04   | Spanning+RD | Flat enumerated | a_maxlev workaround | Tier 3 | — | a_maxlev + riskdiff incompatibility; pre-computed factor pattern |
| T05   | Multivar+Diff | Param×Visit | a_summarize_aval_chg_diff_j | — | — | Contract 4; baseline blanking; multivars wiring |
| T06   | Spanning+RD | SOC→PT | a_eair100_j + a_patyrs_j | — | Yes | Mixed afun types in same layout; a_patyrs_j blanking in RD cols |
| T07   | Shift struct | Param→Arm→Level | a_freq_j + restr_columns | Tier 3 | — | shift_tbl_col_struct; N column restriction; cross-tab row structure |
| T08   | Simple    | page_by + Visit | tern::a_summary | — | — | page_by constraints; dynamic decimal precision; formats_var |
| T09   | Simple    | SOC→PT→Grade | a_freq_j + combo level | Possibly | Yes | No-op analyze anti-pattern trap; add_overall_level vs afun combo row |
| T10   | ARM+Total | 3-level hierarchy | a_freq_j | — | Yes | 3-level trim_levels_in_group; Overall column; multi-level sort |
| T11   | Subgrp cols | Stacked | tern::a_summary | — | — | grouped_cols_w_subgrps; column-agnostic tern afun is safe here |
| T12   | Non-MX subsets | SOC→PT | a_freq_subcol_j or some_v_all | — | Yes | some_v_all_col_struct; column-conditional event filtering |
| T13   | Stats-in-cols | Param→Arm→Visit | column_stats | — | — | stats_in_cols_setup; exclude_visits; formats_var; Contract 4 |
| T14   | Combo arms | Visit facets | a_summarize_ancova_j | — | — | add_combo_levels; method_combo = "contrasts"; ref_path blanking |
| T15   | Period cols | SOC→PT | a_freq_combos_j | — | — | combosdf; filter_var/flag_var; do_not_filter; double-counting prevention |
| T16   | Non-trt grouped_cols_w_diffs + asymmetric inner | Visit facets | tern::a_summary + a_diff_means | Tier 3 dispatch | — | junco on non-ARM var; blank span; asymmetric titer sub-cols |
| T17   | Non-trt split + Total | Param×Visit | tern::a_summary | — | — | junco over-engineering trap; Total inside span; non-ARM column var |
| T18   | Blank span detection | SOC→PT | a_freq_j | — | — | Whitespace spanning label; ref_path with " "; create_colspan_var |
| T19   | Total UNDER span | SOC→PT | a_freq_j | — | — | Centering disambiguation; add_overall_level vs add_overall_col |
| T20   | Total OUTSIDE span | SOC→PT | a_freq_j | — | — | Centering disambiguation; nested = FALSE; contrast with T19 |
| T21   | Simple | Stacked | a_freq_j (dual .stats) | Tier 3 stacking | — | Subject+event counts in same section; vertical stacking |
| T22   | Simple | 3 dissimilar sections | a_freq_j + a_maxlev | — | Yes | Dissimilar sections forced into single split; wrong afun per section |
| T23   | Simple | Stacked flags | a_freq_j × 6 | — | — | Factor fabrication trap; overlapping membership |
| T24   | Simple | at_sibling multi-section | ancova + a_summary | — | — | nested=FALSE escape; at_sibling requirement |
| T25   | Simple | Stacked flags | a_freq_j | — | — | Standalone marginal summary anti-pattern; no-op analyze |
| T26   | Spanning+RD | SOC→PT | a_freq_j | — | — | Contract 1 after filtering; stale factor levels |
| T27   | Simple | SOC→PT→Severity | a_maxlev | — | — | drop_split_levels inverted pitfall; ragged structure intentional |

---

## Shell T01: Demographics with Stacked Continuous + Categorical Sections

### Reasoning Paths Exercised

**Orchestrator Stage 1 (Decompose)**:
- Dataset identification: ADSL as both df and alt_counts_df (or no alt_counts_df needed). Tests whether the model correctly identifies that alt_counts_df is unnecessary when df IS adsl.
- Record granularity: one row per subject — trivial but must be verified.

**Row Structure (row_struct_layout.md)**:
- Archetype 1 (No-Split Stacked Analyses). This is the simplest row archetype but is frequently mishandled: the model may try to use `split_rows_by("SEX")` for the Sex section instead of `analyze("SEX", afun = tern::a_summary)`.
- Tests the `show_labels = "visible"` + `var_labels` pattern for section headers.
- Tests whether the model correctly uses multiple `analyze()` calls rather than trying to force everything into a single call.

**Select Afuns (select_afuns.md)**:
- Mixed continuous + categorical in the same table. The model must select `tern::a_summary` (or equivalent) and configure `.stats` differently for continuous (n, mean_sd, median, range) vs. categorical (count_fraction) variables.
- Tests Fundamental Question 1 (unit of analysis varies by section) and whether the model correctly identifies that `tern::a_summary` handles both continuous and categorical variables.

**Conditionality Assessment**:
- No risk difference columns, no comparison columns. Tests whether the model explicitly records "no conditionality" per the orchestrator's Stage 4 instruction, and correctly concludes that `tern::a_summary` (which lacks riskdiff support) is safe.

### Hypothesized Weaknesses Probed
- **Premature faceting**: Using `split_rows_by("SEX")` instead of `analyze("SEX")` for categorical sections.
- **Afun over-selection**: Choosing `a_freq_j` for categorical sections when `tern::a_summary` handles both continuous and categorical in a unified way.
- **Missing section headers**: Forgetting `var_labels` / `show_labels` on stacked `analyze()` calls.

---

## Shell T02: AE Table with Subgroup Row Split and Conditioned Denominators

### Structural Notes (from shell spec)
- The "Gender: Female" header row shows the subgroup N in arm columns and blanks in risk difference columns.

### Reasoning Paths Exercised

**Orchestrator Contract 3 (denom_by)**:
- This is the primary target. The model must wire `denom = "n_altdf"` and `denom_by = "SEX"` on all nested `summarize_row_groups()` and `analyze()` calls.
- The subgroup header row ("Gender: Female") must use `.stats = "n_altdf"` with `riskdiff = FALSE` (blank RD columns), while body rows use `.stats = "count_unique_denom_fraction"` with `riskdiff = TRUE`.

**Select Afuns — Dual Conditionality (Section 2.4C)**:
- The subgroup header vs. body rows require different `riskdiff` settings on the SAME function (`a_freq_j`). This is the canonical dual-conditionality pattern from select_afuns.md Recipe 2.
- Tests whether the model correctly uses `label_fstr = "Gender: %s"` on the subgroup header cfun.

**Column Structure (col_struct_layout.md)**:
- Spanning headers + risk difference columns via `grouped_cols_w_diffs()`.
- Tests Contract 2 (ref_path alignment) and the `ref_path` auto-injection rule (must NOT pass ref_path in extra_args when using grouped_cols_w_diffs).

**Introspection/Sort/Prune**:
- Sort SOC and PT by decreasing incidence within each sex subgroup.
- Tests multi-level sort with wildcard paths that must navigate through the SEX split.

### Hypothesized Weaknesses Probed
- **denom_by omission**: Forgetting `denom_by = "SEX"` and getting percentages out of the full arm N instead of the sex-specific N.
- **ref_path double-injection**: Passing `ref_path` in extra_args alongside `grouped_cols_w_diffs()`.
- **Uniform riskdiff**: Setting `riskdiff = TRUE` on the subgroup header row (producing nonsensical risk differences for sample sizes).
- **Sort path construction**: Getting the wildcard path wrong when SEX is an additional nesting level above AEBODSYS.

---

## Shell T03: Disposition Table with Selective Drill-Down

### Reasoning Paths Exercised

**Row Structure — Archetype 16 (Mixed Nesting / Selective Drill-Down)**:
- Only "Discontinued" expands into indented reason rows (DCSREAS). "Completed" and "Ongoing" remain single rows.
- The model must choose between `a_two_tier` (preferred) and the structural alternative (`split_rows_by("EOSSTT") |> summarize_row_groups() |> split_rows_by("DCSREAS")`).

**Orchestrator Contract 6 (Mixed Nesting Levels)**:
- Directly tests whether the model follows the preferred production pattern (single `analyze()` with `a_two_tier`) vs. the structural alternative.

**Select Afuns — a_two_tier**:
- Tests correct parameterization: `inner_var = "DCSREAS"`, `drill_down_levs = "DISCONTINUED"`.
- Tests whether the model correctly identifies that `a_two_tier` delegates to `a_freq_j` by default for both group and detail functions.

**Record Granularity**:
- ADSL has one row per subject — but the model must verify this and confirm that row counts = subject counts for disposition.

### Hypothesized Weaknesses Probed
- **Structural over-engineering**: Using `split_rows_by("EOSSTT") |> split_rows_by("DCSREAS")` which creates empty DCSREAS facets under "Completed" and "Ongoing" (requiring post-build pruning or a conditional split function).
- **No-op analyze anti-pattern**: If the structural approach is chosen, the model might use a no-op afun under non-DISCONTINUED levels.
- **a_two_tier parameterization**: Getting `drill_down_levs` wrong or forgetting `use_all_levels`.

---

## Shell T04: AE Table with Maximum Severity Grade and Risk Differences

### Structural Notes (from shell spec)
- "Subjects with >=1 TEAE" is the total of subjects with any grade.
- Grade rows show the distribution of subjects' worst grades.

### Reasoning Paths Exercised

**Select Afuns — a_maxlev Limitation**:
- This is the primary trap. `a_maxlev` is the natural choice for worst-grade counting, but it does NOT support `riskdiff`/`ref_path`. The model must recognize this limitation and use the pre-computed worst-grade factor + `a_freq_j` workaround.
- Directly tests the weakness documented in TSFAE01 skill feedback.

**Custom Afuns — Tier 1 Parameter Space Search**:
- If the model first tries `a_maxlev` and it fails in RD columns, the model should diagnose the failure and search for an alternative parameterization (pre-computed factor + `a_freq_j`) rather than jumping to a bespoke Tier 5 function.

**Data Prep**:
- Requires pre-computing a `WORST_GRADE` factor variable on ADAE (one row per subject at their worst grade) before passing to `build_table()`.
- Tests Building Block 7 (pre-computed derived variables) from custom_afuns.md.

**Column Structure**:
- Spanning headers + risk difference columns. Same as T02 but with a different afun challenge.

### Hypothesized Weaknesses Probed
- **a_maxlev in RD layout**: Using `a_maxlev` directly and getting `"'subset' must be logical"` errors in diff columns.
- **Premature escalation**: Jumping to a bespoke inline function instead of using pre-computed factor + `a_freq_j`.
- **Data prep omission**: Computing the worst grade inside the afun instead of in data prep (Building Block 7 violation).
- **Row structure confusion**: The "Subjects with >=1 TEAE" row is a flag-variable summary (val = "Y"), while grade rows are factor-level enumeration — two different `.var` choices in the same layout.

---

## Shell T05: Vital Signs with 3-Column Layout and Difference in Mean Change

### Structural Notes (from shell spec)
- Each arm has three sub-columns: n/N (%), Mean (95% CI), and Mean CFB (95% CI). A separate difference column compares mean change from baseline between arms.

### Reasoning Paths Exercised

**Orchestrator Contract 4 (Multi-Variable Column Splits)**:
- This is the canonical 3-column layout from Contract 4's table. Tests whether the model correctly wires `split_cols_by_multivar(c("AVAL", "AVAL", "CHG"))` with `a_summarize_aval_chg_diff_j`.

**Select Afuns — a_summarize_aval_chg_diff_j**:
- Tests correct parameterization: `multivars`, `comp_btw_group = TRUE`, `ref_path`, `ancova = FALSE`.
- Tests the dummy variable pattern: `analyze("STUDYID", afun = a_summarize_aval_chg_diff_j, ...)`.
- Tests baseline blanking: `df[df$ABLFL == "Y", "CHG"] <- NA`.

**Row Structure**:
- `split_rows_by("PARAM") |> split_rows_by("AVISIT", child_labels = "hidden")` — tests `child_labels = "hidden"` because the afun labels rows by visit name.
- Tests `label_pos = "topleft"` on both splits for the top-left corner hierarchy.

**Column Structure**:
- Complex: main branch with `split_cols_by_multivar` + non-nested diff branch.
- Tests whether the model correctly constructs the spanning header structure with the diff column as a sibling branch.

### Hypothesized Weaknesses Probed
- **Forgetting child_labels = "hidden"**: Producing duplicate visit labels (one from the split, one from the afun).
- **Wrong dummy variable**: Using `.var` directly instead of a dummy variable like "STUDYID".
- **Baseline blanking omission**: Not setting CHG to NA at baseline, producing non-blank CFB cells at baseline.
- **Contract 4 table mismatch**: Not following the exact wiring pattern from the orchestrator's Contract 4 table.

---

## Shell T06: Exposure-Adjusted Incidence Rate Table

### Structural Notes (from shell spec)
- Total patient-years row shows total exposure time; it should be blank in rate difference columns.

### Reasoning Paths Exercised

**Select Afuns — EAIR Family**:
- Tests selection of `a_patyrs_j` for the total patient-years row and `a_eair100_j` for EAIR rows.
- Tests that `a_patyrs_j` automatically blanks RD columns when `ref_path` is provided.
- Tests correct wiring of `fup_var`, `occ_var`, `occ_dy` for time-at-risk calculation.

**Mixed Afun Types in Same Layout**:
- Three different afun configurations in one layout: `a_patyrs_j` (root), `a_eair100_j` as cfun (SOC summary), `a_eair100_j` as afun (PT detail).
- Tests whether the model correctly uses different `extra_args` for each.

**Row Structure**:
- Standard AE archetype (root summary + SOC→PT) but with EAIR-specific afuns instead of `a_freq_j`.

**Sort/Prune**:
- Sort by EAIR (not by count). Tests whether the score function correctly extracts the EAIR value rather than the count.

### Hypothesized Weaknesses Probed
- **Wrong sort metric**: Sorting by count instead of EAIR value.
- **fup_var/occ_var confusion**: Incorrect variable wiring for person-years calculation.
- **a_patyrs_j in wrong position**: Using it as a cfun on SOC groups instead of as a standalone root row.
- **Missing first-occurrence derivation**: Not deriving the first-occurrence flag from ADAE.

---

## Shell T07: Shift Table for Laboratory Values

### Structural Notes (from shell spec)
- Column structure: N | Baseline levels (LOW, NORMAL, HIGH) under "Baseline ALB" span | Total.
- Row structure: Parameter -> Treatment Group -> Post-baseline shift levels.

### Reasoning Paths Exercised

**Column Structure — Archetype 3 (Shift Table)**:
- Tests `shift_tbl_col_struct()` for the N | Baseline levels | Total column layout.
- This is a non-standard column topology that many models struggle with.

**Select Afuns — restr_columns**:
- The N column shows subject counts while baseline-level and Total columns show shift counts. Tests whether the model uses `restr_columns` to restrict `a_freq_j` output to specific columns.

**Row Structure**:
- Parameter → Treatment Group → Post-baseline shift levels. Tests 3-level nesting with the innermost level being the analysis variable.

**Custom Afuns — Tier 3 (Vertical Stacking)**:
- May require a custom afun that combines N-column behavior (subject count) with shift-column behavior (cross-tabulation counts).

### Hypothesized Weaknesses Probed
- **Column structure confusion**: Not recognizing the shift table pattern and trying to build it manually.
- **restr_columns misuse**: Not understanding that restr_columns is a KEEP list (not an exclude list).
- **Cross-tabulation logic**: Confusing the row variable (post-baseline level) with the column variable (baseline level).

---

## Shell T08: Multi-Section Stacked Table with page_by

### Structural Notes (from shell spec)
- One page per parameter (separate page for ALT, CRP, etc.).
- Standard continuous summary statistics (N, Mean/SD, Median, Min/Max).

### Reasoning Paths Exercised

**Row Structure — page_by Constraints**:
- Tests `split_rows_by("PARAMCD", page_by = TRUE)` as the outermost split.
- Tests the constraint that page_by splits must be outermost with no top-level siblings.
- Tests that page_by only takes effect during pagination (not at console).

**Select Afuns — Dynamic Decimal Precision**:
- Tests `fmt_spec_df_d` and `formats_var = "fmt_d"` for parameter-specific precision.
- Tests Pattern B from select_afuns.md Section 6.2.

**Orchestrator Stage 1 — Record Granularity**:
- ADLB has multiple rows per subject per visit per parameter. The model must verify this and ensure that N counts use subject deduplication if needed.

### Hypothesized Weaknesses Probed
- **page_by placement**: Putting page_by on an inner split or having top-level siblings.
- **formats_var wiring**: Forgetting to left_join the fmt_d column onto the analysis dataset.
- **Record granularity assumption**: Assuming ADLB has one row per subject and using nrow() as N.

---

## Shell T09: AE Table with "Any Grade" Combo Level + Grade Breakdown

### Structural Notes (from shell spec)
- SOC rows show unique subject counts.
- Under each PT, "Any Grade" is a combination level combining all grades, followed by individual grade rows.

### Reasoning Paths Exercised

**Row Structure — No-Op Analyze Anti-Pattern Trap**:
- The "Any Grade" row is a combination level. The model might try `split_rows_by("AETOXGR", split_fun = add_overall_level("Any Grade"))` followed by `summarize_row_groups()` and a no-op `analyze()`. This is the prohibited anti-pattern.
- The correct approach is either: (a) `a_two_tier` or `a_three_tier` with grade as inner_var, or (b) `split_rows_by("AETOXGR", split_fun = add_overall_level("Any Grade")) |> summarize_row_groups() |> analyze()` with a real afun.

**Row Structure — Section 0 (No-Op Redirection)**:
- Tests whether the model correctly identifies that the "Any Grade" + individual grades pattern requires analysis rows (not just facet labels with marginal summaries).

**Select Afuns — a_maxlev vs a_freq_j**:
- "Each subject counted once at maximum grade" — this is the a_maxlev use case, but without RD columns this time, so a_maxlev IS valid here.
- Tests whether the model correctly identifies that a_maxlev is safe when there are no difference columns.

**Sort/Prune**:
- Sort SOC and PT by decreasing incidence while preserving the "Any Grade" row position (it should always be first under each PT).

### Hypothesized Weaknesses Probed
- **No-op analyze**: Using `split_rows_by("AETOXGR")` with a no-op terminating analyze.
- **Combo level in wrong place**: Trying to add "Any Grade" via a custom afun instead of via split_fun.
- **a_maxlev when safe**: Unnecessarily avoiding a_maxlev when there are no RD columns.
- **Sort disrupting "Any Grade" position**: Sorting grades by incidence and moving "Any Grade" out of first position.

---

## Shell T10: Concomitant Medications with 3-Level Hierarchy and Overall Row

### Structural Notes (from shell spec)
- Column structure: ARM split + Overall "Total" column.
- Row structure: 3-level hierarchy ATC2 -> ATC4 -> CMDECOD with group summary counts at ATC2 and ATC4 levels. ATC4 subclasses are hierarchically dependent on ATC2 classes; preferred names are hierarchically dependent on ATC4 subclasses.
- Top-level "Subjects with >=1 ConMed" row counts unique subjects with any concomitant medication.

### Reasoning Paths Exercised

**Row Structure — 3-Level Nesting**:
- ATC2 → ATC4 → CMDECOD with `summarize_row_groups` at both ATC2 and ATC4 levels.
- Tests Archetype 7 (2-level nested with group summaries at both tiers) extended to 3 levels.

**Row Structure — trim_levels_in_group Chaining**:
- Hierarchically dependent variables at 3 levels. Tests whether the model correctly chains `trim_levels_in_group` on the outer splits.

**Column Structure — Overall Column**:
- Simple ARM split + "Total" overall column. Tests `add_overall_col("Total")` or `add_overall_level("Total")`.

**Sort/Prune — Multi-Level Sort**:
- Sort at 3 levels (ATC2, ATC4, CMDECOD) by decreasing incidence in the Total column.
- Tests `jj_complex_scorefun` with a specific column path targeting the Total column.

**Select Afuns — Flag Variable Pattern**:
- "Subjects with >=1 ConMed" requires a flag variable (e.g., CMFL = "Y") with `val = "Y"` and `label = "Subjects with >=1 ConMed"`.
- Tests the a_freq_j flag-variable pattern that was the root cause of Weakness 1 in the TIRADA review.

### Hypothesized Weaknesses Probed
- **USUBJID as .var**: Passing USUBJID to analyze() for the overall count row instead of a flag variable.
- **trim_levels_in_group at wrong level**: Applying it on the innermost split instead of the outermost.
- **3-level sort path**: Getting wildcard paths wrong for the 3-level hierarchy.
- **Overall column sort**: Not targeting the Total column specifically when sorting.

---

## Shell T11: Arms Subdivided by Subgroups with Within-Arm Totals

### Structural Notes (from shell spec)
- Column structure: Each arm is subdivided by Sex, with a within-arm Total column.
- Row structure: Stacked sections for continuous (Age) and categorical (Race) variables.

### Reasoning Paths Exercised

**Column Structure — Archetype 2 (grouped_cols_w_subgrps)**:
- Tests `grouped_cols_w_subgrps(trtvar = "ARM", subgrp_var = "SEX", total_facet = TRUE)`.
- Tests the note that these functions work on arbitrary factor variables, not just treatment arms.

**Select Afuns — tern::a_summary Safety**:
- No difference columns in this layout. Tests whether the model correctly identifies that `tern::a_summary` is safe (no riskdiff/ref_path needed) and explicitly records this assessment.

**Row Structure — Stacked Analyses**:
- Same as T01 but with a more complex column structure. Tests that row structure is truly orthogonal to column structure.

### Hypothesized Weaknesses Probed
- **Manual column construction**: Building the ARM × SEX column structure manually instead of using `grouped_cols_w_subgrps`.
- **Unnecessary conditionality concern**: Worrying about column conditionality when there are no difference columns.
- **Column count wiring**: Ensuring N=XXX counts are correct for each subgroup column (requires alt_counts_df or correct df).

---

## Shell T12: Non-Mutually-Exclusive AE Subsets (All AEs vs Related AEs)

### Structural Notes (from shell spec)
- Column structure: "All AEs" and "Drug-Related AEs" are non-mutually-exclusive subsets. A subject counted in "Drug-Related AEs" is always also counted in "All AEs".

### Reasoning Paths Exercised

**Column Structure — Archetype 4 (some_v_all_col_struct)**:
- Tests `some_v_all_col_struct` for non-mutually-exclusive column subsets.
- "All AEs" includes all records; "Drug-Related AEs" filters to AEREL == "RELATED".

**Select Afuns — Column-Conditional Filtering**:
- Tests whether the model selects `a_freq_subcol_j` (which filters within specific sub-columns) or `some_v_all_col_struct` (which creates combo column facets).
- Two valid approaches exist; tests whether the model evaluates both and selects appropriately.

**Sort — Column-Specific Sorting**:
- Sort by incidence in the "All AEs" columns specifically, not across all columns.
- Tests column-path-aware scoring.

### Hypothesized Weaknesses Probed
- **Mutually-exclusive assumption**: Treating "All AEs" and "Drug-Related AEs" as separate datasets instead of overlapping column subsets.
- **Wrong column structure function**: Using `grouped_cols_w_diffs` (which is for comparison columns) instead of `some_v_all_col_struct`.
- **Sort across wrong columns**: Sorting by total across all columns (including Drug-Related) instead of just "All AEs".

---

## Shell T13: Statistics-in-Columns Layout

### Structural Notes (from shell spec)
- Column structure: Summary statistics (N, Mean, SD, Median, Min, Max) are arranged as columns, grouped under "Visit Values" (for AVAL) and "Change from Baseline" (for CHG).
- Row structure: Parameter -> Treatment Group -> Visit.

### Reasoning Paths Exercised

**Column Structure + Select Afuns — Contract 4 (Stats-in-Columns)**:
- Tests the full `split_cols_by_multivar` + `stats_in_cols_setup()` + `column_stats` pipeline.
- This is the most complex column structure pattern in the skill set.

**Select Afuns — column_stats**:
- Tests `exclude_visits` for blanking baseline CHG columns.
- Tests `formats_var = "fmt_d"` for parameter-specific precision.

**Row Structure**:
- PARAM → ARM (with `labels_var = "ARM"`) → analyze("AVISIT", afun = column_stats).
- Tests the unusual pattern where AVISIT is the analyzed variable (not a split variable) because column_stats labels rows by visit.

### Hypothesized Weaknesses Probed
- **stats_in_cols_setup omission**: Forgetting the `split_cols_by("STUDYID", split_fun = make_split_fun(post = list(stats_in_cols_setup())))` step.
- **AVISIT as split vs analyze**: Putting AVISIT in `split_rows_by` instead of `analyze`.
- **exclude_visits case sensitivity**: Getting the visit level names wrong (e.g., "Baseline" vs "BASELINE").

---

## Shell T14: Pooled/Combined Treatment Arms with ANCOVA

### Structural Notes (from shell spec)
- Column structure: ARM split with a virtual combined "All Active" arm pooling Drug X and Combination.

### Reasoning Paths Exercised

**Column Structure — Combo Arms**:
- Tests `add_combo_levels(combodf)` for the "All Active" virtual combined arm.
- Tests that the combo arm's LS Mean is derived via linear contrasts, not by pooling data.

**Select Afuns — a_summarize_ancova_j**:
- Tests `method_combo = "contrasts"`, `weights_combo = "equal"`, `ref_path`.
- Tests that LS Mean Diff, CI, and p-value are blank in the Placebo (reference) column.

**Orchestrator Contract 2 (ref_path)**:
- With combo arms, the column path structure is more complex. Tests whether ref_path correctly points to Placebo through the combo-augmented column tree.

### Hypothesized Weaknesses Probed
- **Data pooling instead of contrasts**: Collapsing Drug X and Combination data into a single "All Active" dataset instead of using linear contrasts from the full model.
- **ref_path with combo columns**: Getting the column path wrong when combo levels alter the column tree structure.
- **Blank reference column**: Not blanking the LS Mean Diff row in the Placebo column.

---

## Shell T15: Cumulative/Period AE Columns

### Structural Notes (from shell spec)
- Column structure: Treatment period columns where individual periods filter by AESTDY ranges, but Cumulative does not filter (counts each subject once across all periods).

### Reasoning Paths Exercised

**Select Afuns — a_freq_combos_j**:
- Tests the most specialized column-conditional counting function.
- Tests `combosdf`, `filter_var`, `flag_var`, `do_not_filter` parameterization.

**Column Structure**:
- Period columns are not a standard column split pattern — they require custom column definitions where each column filters to a different date range.
- Tests whether the model correctly identifies this as a `combosdf`-driven layout.

**Double-Counting Prevention**:
- Individual period columns must not double-count subjects who have events in both periods.
- The Cumulative column counts each subject once regardless of how many periods they had events in.
- Tests the `filter_var` / `do_not_filter` mechanism.

### Hypothesized Weaknesses Probed
- **Standard split_cols_by**: Trying to use a simple factor split instead of the combosdf mechanism.
- **Double-counting**: Not using filter_var to prevent counting a subject in both Induction and Maintenance.
- **Cumulative column logic**: Applying the period filter to the Cumulative column (undercounting) or not filtering individual periods (overcounting).

---

## Gap Analysis: Reasoning Paths NOT Covered by Existing Refinement Shells

The following reasoning paths are exercised by the proposed test shells but were NOT covered by the 5 existing refinement shells (TSFLAB06, TSFAE04, TSFAE02, TSFAE01, TIRADA05b):

| Gap | Covered By |
|-----|-----------|
| Stacked continuous + categorical with tern::a_summary | T01 |
| denom_by subgroup conditioning (Contract 3) | T02 |
| Dual conditionality (riskdiff toggle per row level) | T02 |
| a_two_tier selective drill-down | T03 |
| a_maxlev + riskdiff workaround (pre-computed factor) | T04 |
| a_summarize_aval_chg_diff_j 3-column layout (Contract 4) | T05 |
| EAIR family (a_eair100_j + a_patyrs_j) | T06 |
| page_by pagination splits | T08 |
| Dynamic decimal precision (formats_var) | T08, T13 |
| "Any Grade" combo level + grade breakdown | T09 |
| 3-level row hierarchy with multi-level sort | T10 |
| Flag variable pattern (USUBJID trap) | T10 |
| grouped_cols_w_subgrps | T11 |
| some_v_all_col_struct / non-MX subsets | T12 |
| stats_in_cols_setup + column_stats | T13 |
| add_combo_levels + ANCOVA contrasts | T14 |
| a_freq_combos_j cumulative period columns | T15 |
| junco column functions on non-treatment-arm variables | T16, T17 |
| Blank/whitespace spanning label detection | T16, T18 |
| Asymmetric inner columns (titer buckets under one parent only) | T16 |
| Total column placement disambiguation (under vs outside span) | T19, T20 |

### Known Weakness Categories Targeted

1. **Tier 1 Parameter Space Search Failures** (T04, T10): The model tries one parameterization, it fails, and it jumps to bespoke code instead of searching alternative `.var` / `.stats` / `extra_args` combinations.

2. **Anti-Pattern Violations** (T03, T09): The model uses `split_rows_by` where `analyze` belongs, or creates no-op `analyze()` calls.

3. **Contract Violations** (T02, T05, T14): The model fails to synchronize datasets (Contract 1), wire ref_path correctly (Contract 2), condition denominators (Contract 3), or couple multivar columns with afuns (Contract 4).

4. **Column Structure Misidentification** (T07, T12, T13, T15): The model selects the wrong column topology pattern from the decision matrix.

5. **Post-Build Operation Errors** (T02, T06, T09, T10): Wrong sort paths, wrong score functions, sorting before pruning, or sort disrupting pinned category positions.


---

## Shell T16: Immunogenicity by ADA Status and Peak Titer with Pairwise Differences

### Structural Notes (from shell spec)
- Column structure uses ADA_STATUS as the primary grouping variable (NOT a treatment arm). "Negative" is the reference group; "Positive" is under the "Peak Titers..." spanning label with titer bucket sub-columns; a non-nested "Difference" sibling branch compares Positive vs Negative.
- The titer bucket columns (<10, 10 to <100, >=100) exist only under the "Positive" level, not under "Negative" (asymmetric inner structure).
- Row structure: Visit facets with continuous summary statistics (N, Mean/SD, Median).

### Reasoning Paths Exercised

**Column Structure — junco Functions on Non-Treatment Variables**:
- This is the primary target. The column topology is `grouped_cols_w_diffs` + `grouped_cols_w_subgrps` (or `trim_levels_in_group`), but the outer grouping variable is `ADA_STATUS` (Negative / Positive), NOT a treatment arm. The titer buckets (<10, 10 to <100, >=100) are sub-columns under the "Positive" level.
- Tests whether the model recognizes that `create_colspan_var`, `create_colspan_map`, and `grouped_cols_w_diffs` operate on **arbitrary factor variables** despite their argument names (`trt_var`, `colspan_trt_map`). The orchestrator explicitly notes this: "A `grouped_cols_w_subgrps(trtvar = "ADA_STATUS", subgrp_var = "TITER_BUCKET")` call is perfectly valid."
- The column structure is a hybrid: the outer level is a `grouped_cols_w_diffs` topology (Negative vs Positive with a Difference branch), while the inner level under "Positive" is an asymmetric hierarchy (titer buckets exist only under Positive, not under Negative). This requires either `trim_levels_in_group("TITER_BUCKET")` on the ADA_STATUS split, or a `trim_levels_to_map(map)` approach.

**Blank Spanning Label Detection**:
- "Negative" sits under a blank spanning label at the same tier as "Peak Titers for Subjects Positive for Treatment-emergent Antibodies". This is the same `non_active_grp_span_lbl = " "` pattern as in standard AE tables, but applied to ADA status.
- Tests whether the model correctly identifies the blank span and wires `ref_path = c("colspan_trt", " ", "ADA_STATUS", "Negative")`.

**Select Afuns — Mixed Column Behavior**:
- Main columns show continuous summaries (tern::a_summary); the Difference column shows mean difference + CI (a_diff_means or a custom wrapper).
- This is column-conditional behavior where the main columns use a column-agnostic afun and the difference column uses a comparative afun. Tests whether the model recognizes this as a Tier 3 column-dispatch pattern (custom_afuns.md Pattern 2A) or whether it can use a_diff_means with ref_path directly.

**Orchestrator Stage 2 — Non-Standard Variable Naming**:
- The model must map the shell's semantic labels ("Negative for Treatment-emergent Antibodies", "Positive for Treatment-emergent Antibodies") to data variable names (ADA_STATUS, TITER_BUCKET) that don't follow the standard ARM/TRT01A naming convention.

### Hypothesized Weaknesses Probed
- **Treatment-arm fixation**: Refusing to use `grouped_cols_w_diffs` because the grouping variable isn't a treatment arm, and instead attempting to build the column structure manually with chained `split_cols_by` calls.
- **Asymmetric inner columns**: Not recognizing that titer buckets only exist under "Positive" and trying to create them under "Negative" too (producing empty N=0 columns).
- **Difference column afun**: Using `tern::a_summary` in the difference column (which doesn't compute between-group differences) instead of `a_diff_means` or a column-dispatch wrapper.
- **ref_path through non-standard variable**: Getting the ref_path wrong because the variable is ADA_STATUS instead of ARM/TRT01A.

---

## Shell T17: Biomarker Response by Exposure Quartile with Subgroup Columns

### Structural Notes (from shell spec)
- Column structure: Exposure quartile (AUCQRTL) as the primary grouping variable with a leading "Total" column. AUCQRTL is NOT a treatment arm — it is a PK exposure quartile.
- "Total" is under the "Exposure Quartile (AUC)" spanning label (same tier as Q1–Q4).
- Row structure: PARAM -> AVISIT -> continuous summary statistics.

### Reasoning Paths Exercised

**Column Structure — junco Functions on Non-Treatment Variables (Simpler Case)**:
- The column structure is `split_cols_by("AUCQRTL", split_fun = add_overall_level("Total", first = TRUE))` — a single categorical split with a leading Total, where the splitting variable is an exposure quartile, not a treatment arm.
- Tests whether the model considers `grouped_cols_w_subgrps(trtvar = "AUCQRTL", ...)` and correctly determines it's overkill here (no subgroup subdivision within each quartile), falling back to the simpler `split_cols_by` + `add_overall_level` pattern.
- Alternatively, tests whether the model recognizes that `grouped_cols_w_subgrps` with `total_facet = TRUE` and no `subgrp_var` is equivalent to the simpler pattern.

**Total Column Placement**:
- "Total" is under the "Exposure Quartile (AUC)" spanning label (same tier as Q1–Q4). Tests whether the model correctly places Total inside the split (via `add_overall_level`) rather than outside it (via `add_overall_col`).
- This is a simpler version of the T19/T20 ambiguity — here the spanning label clearly covers all columns including Total.

**Select Afuns — Conditionality Assessment**:
- No difference columns, no comparison columns. Tests whether the model explicitly records "no conditionality" and correctly concludes that `tern::a_summary` is safe.
- This is the same conditionality assessment as T01 and T11, but in a non-treatment-arm context where the model might be tempted to add comparison logic.

**Row Structure**:
- PARAM → AVISIT → continuous summary. Standard nested structure with `label_pos = "topleft"` on outer splits.

### Hypothesized Weaknesses Probed
- **Over-engineering with junco**: Using `grouped_cols_w_subgrps` or `grouped_cols_w_diffs` when a simple `split_cols_by` + `add_overall_level` suffices.
- **Treatment-arm assumption**: Assuming the column variable must be a treatment arm and trying to create spanning headers or comparison columns that don't exist in the shell.
- **Total placement**: Using `add_overall_col("Total")` (outside the span) instead of `add_overall_level("Total")` (inside the split).

---

## Shell T18: AE Summary with Blank Spanning Label Between Two Named Spans

### Structural Notes (from shell spec)
- The column header has THREE spanning regions at the same tier: "Active Treatment" (over Drug X and Combination), a blank/whitespace region (over Placebo), and "Risk Difference (%) (95% CI)" (over the two comparison columns).
- The blank region above Placebo is a real spanning node with a whitespace label, not the absence of a span. It occupies the same hierarchical tier as the other two spans.

### Reasoning Paths Exercised

**Column Structure — Blank Spanning Label Detection (col_struct_layout.md Section 2.2, Rule 1)**:
- This is the primary target. The shell has THREE top-level spanning regions at the same tier:
  1. "Active Treatment" (over A: Drug X, C: Combination)
  2. A blank/whitespace span (over B: Placebo)
  3. "Risk Difference (%) (95% CI)" (over the two comparison columns)
- The blank span is the critical detection challenge. In the visual shell, there is simply NO text above "B: Placebo" — the model must infer that this absence of text represents a real spanning node with a whitespace label, not the absence of a spanning tier.

**Visual Alignment Reasoning**:
- The model must use the vertical alignment of column headers to determine spanning structure. "B: Placebo" is at the same vertical position as "A: Drug X" and "C: Combination", and all three have N=XXX counts at the same level. But "A: Drug X" and "C: Combination" have "Active Treatment" above them, while "B: Placebo" has nothing.
- The key inference: since "Active Treatment" and "Risk Difference (%)" are at the same vertical tier, and "B: Placebo" is at the same leaf tier as the other arms, there MUST be a spanning node above "B: Placebo" at the same depth — it just has a blank label.

**create_colspan_var / create_colspan_map Mechanics**:
- Tests whether the model correctly uses `non_active_grp_span_lbl = " "` (a single space, not empty string) to create the blank spanning label.
- Tests whether the model correctly constructs `ref_path = c("colspan_trt", " ", "ARM", "B: Placebo")` with the blank `" "` as the second element.

**Relationship to grouped_cols_w_diffs**:
- This is the standard `grouped_cols_w_diffs` pattern, but the test focuses specifically on whether the model correctly identifies and implements the blank spanning label — the step that precedes the `grouped_cols_w_diffs` call.

### Hypothesized Weaknesses Probed
- **Missing the blank span entirely**: Treating "B: Placebo" as a top-level sibling of "Active Treatment" (no spanning parent), which would produce a 2-level column tree under Active Treatment but a 1-level column for Placebo — breaking the consistent-depth requirement.
- **Empty string vs space**: Using `""` instead of `" "` for the blank spanning label, which may cause factor-level or display issues.
- **Flat column structure**: Ignoring the spanning structure entirely and using a single `split_cols_by("ARM")` — this would work functionally but lose the spanning headers and make `grouped_cols_w_diffs` impossible.
- **ref_path with blank span**: Getting the ref_path wrong by omitting the blank `" "` element or using `""` instead.

---

## Shell T19: Total Column Placement Ambiguity — Total UNDER the Spanning Label

### Structural Notes (from shell spec)
- "Treatment Group" spans all 4 leaf columns (Total, A, B, C). Total is UNDER the span.

### Reasoning Paths Exercised

**Column Structure — Total Column Placement (col_struct_layout.md Section 2.2, Rule 1, Case A)**:
- This is one half of a matched pair (T19 + T20) designed to test the model's ability to resolve the ambiguous placement of a Total column relative to a spanning label.
- In T19, "Treatment Group" is centered across ALL four columns (Total, A, B, C). The visual cue is that the spanning label's horizontal extent covers the Total column.
- The correct implementation places Total INSIDE the ARM split via `add_overall_level("Total", first = TRUE)`, nested under the "Treatment Group" spanning label.

**Visual Centering as the Sole Disambiguation Cue**:
- The ONLY difference between T19 and T20 is the horizontal centering of "Treatment Group". In T19 it's centered over 4 columns; in T20 it's centered over 3. The model must use this centering information to determine the column tree topology.
- This tests col_struct_layout.md Section 2.2 Rule 1: "Check which leaf columns a higher-level spanning label is centered across (or underlines) to determine whether a Total or comparison column is nested inside that split or sits in a sibling subtree."

**Implementation Difference**:
- T19: `split_cols_by("span") |> split_cols_by("ARM", split_fun = add_overall_level("Total", first = TRUE))`
- T20: `add_overall_col("Total") |> split_cols_by("span", nested = FALSE) |> split_cols_by("ARM")`
- The two implementations produce different column tree structures and different column paths, which affects downstream `ref_path` construction and score function column targeting.

### Hypothesized Weaknesses Probed
- **Default to add_overall_col**: Always using `add_overall_col("Total")` regardless of spanning label placement, because it's the more commonly seen pattern.
- **Ignoring centering cues**: Not examining the horizontal extent of the spanning label and guessing the topology.
- **Functional equivalence assumption**: Assuming T19 and T20 produce the same table because they have the same leaf columns — they don't, because the column tree structure (and thus paths, column counts visibility, and spanning headers) differs.

---

## Shell T20: Total Column Placement Ambiguity — Total OUTSIDE the Spanning Label

### Structural Notes (from shell spec)
- "Treatment Group" spans only columns 2–4 (A, B, C). Total is OUTSIDE the span.
- The visual difference from T19 is subtle: "Treatment Group" is shifted right (centered over 3 columns, not 4). This is the ONLY cue distinguishing the two topologies.

### Reasoning Paths Exercised

**Column Structure — Total Column Placement (col_struct_layout.md Section 2.2, Rule 1, Case B)**:
- The complement of T19. Here "Treatment Group" is centered over columns 2–4 only, leaving Total outside the span.
- The correct implementation places Total as a top-level sibling via `add_overall_col("Total")`, with the "Treatment Group" spanning label and ARM split in a separate non-nested branch.

**Contrast with T19**:
- When presented as a pair, T19 and T20 test whether the model can distinguish two visually similar shells that require fundamentally different column tree implementations. The visual difference is minimal (a few characters of horizontal shift in the spanning label), but the implementation difference is significant.

**nested = FALSE Mechanics**:
- T20 requires `split_cols_by("span", nested = FALSE)` after `add_overall_col("Total")` to prevent the spanning split from nesting inside the Total column. Tests understanding of the `nested` parameter and the banner pattern from col_struct_layout.md Section 3.5.

### Hypothesized Weaknesses Probed
- **Confusing T19 and T20**: Implementing T20 the same way as T19 (or vice versa) because the visual shells look nearly identical.
- **Missing nested = FALSE**: Forgetting `nested = FALSE` on the spanning split, causing "Treatment Group" to nest inside "Total" instead of being a sibling.
- **Column path differences**: Not realizing that the different tree structures produce different column paths, which would break any downstream `ref_path` or score function that hardcodes a path.

---

## Updated Coverage Matrix (T16–T20)

| Shell | Col Struct | Row Struct | Select Afuns | Custom Afuns | Sort/Prune | Key Reasoning Traps |
|-------|-----------|-----------|-------------|-------------|-----------|-------------------|
| T16   | Non-trt grouped_cols_w_diffs + asymmetric inner | Visit facets | tern::a_summary + a_diff_means | Tier 3 dispatch | — | junco on non-ARM var; blank span; asymmetric titer sub-cols; ref_path through non-standard var |
| T17   | Non-trt split + Total | Param×Visit | tern::a_summary | — | — | junco over-engineering trap; Total inside span; non-ARM column variable |
| T18   | Blank span detection | SOC→PT | a_freq_j | — | — | Whitespace spanning label; ref_path with " "; create_colspan_var mechanics |
| T19   | Total UNDER span | SOC→PT | a_freq_j | — | — | Centering-based disambiguation; add_overall_level vs add_overall_col |
| T20   | Total OUTSIDE span | SOC→PT | a_freq_j | — | — | Centering-based disambiguation; nested = FALSE; contrast with T19 |

### Additional Gap Coverage

| Gap | Covered By |
|-----|-----------|
| junco column functions on non-treatment-arm variables (with difference columns) | T16 |
| junco column functions on non-treatment-arm variables (without difference columns) | T17 |
| Asymmetric inner columns under only one level of outer split | T16 |
| Blank/whitespace spanning label detection and implementation | T16, T18 |
| ref_path construction through blank spanning label | T18 |
| Total column under spanning label (add_overall_level) | T19 |
| Total column outside spanning label (add_overall_col + nested = FALSE) | T20 |
| Disambiguation of nearly-identical shells by spanning label centering | T19 vs T20 (paired) |


---

## Adversarial Shells (T21–T27): Designed to Trip Specific Decision Points

These shells are designed to be realistic clinical tables that create plausible-looking traps at the most fragile reasoning junctures in the skill set. Each targets a specific decision point where the "obvious" approach is wrong.

---

## Shell T21: AE Summary with Event Counts AND Subject Counts in Same Section

### Structural Notes (from shell spec)
- Each paired section has a unique-subject count row (count_unique_fraction) AND a total-event count row (count). These are fundamentally different counting units in the same layout.
- Some sections have only a subject count (no event count).
- All rows are at the root level — no SOC/PT nesting.

### The Trap
The model sees paired rows like "Number of subjects with >=1 TEAE" and "Total number of events" and must recognize these require **two fundamentally different counting mechanisms** (`count_unique_fraction` vs `count`) within the same layout section. The natural instinct is to use a single `a_freq_j` call, but no single `.stats` value produces both rows.

### Reasoning Paths Exercised

**Custom Afuns — Tier 3 Vertical Stacking (Pattern 2B)**:
- Each paired section requires stacking two `a_freq_j` calls: one with `.stats = "count_unique_fraction"` (subject count) and one with `.stats = "count"` (event count). This is the canonical `c(rvs1, rvs2)` pattern.
- The model must recognize that this is NOT a single-afun problem and escalate to Tier 3, not Tier 5.

**Row Structure — Archetype 1 (Stacked Analyses)**:
- All rows are at the root level. The model must use multiple stacked `analyze()` calls, each with a different flag variable and a stacking afun.
- Tests whether the model correctly decomposes the shell into separate sections rather than trying to force everything into a single `analyze()`.

**Select Afuns — Flag Variable Pattern (Repeated)**:
- Each section requires a DIFFERENT flag variable: TRTEMFL for any TEAE, AESER for SAE, AEACN for discontinuation, AEREL for drug-related. Tests whether the model derives all flags in data prep.

### Hypothesized Weaknesses Probed
- **Single-afun assumption**: Trying to get both subject counts and event counts from one `a_freq_j` call.
- **Premature Tier 5 escalation**: Writing a bespoke inline function instead of using `c(rvs1, rvs2)` with two `a_freq_j` calls.
- **Missing flag derivation**: Not deriving the SAE/discontinuation/related flags in data prep.
- **Inconsistent row counts across sections**: Some sections have 2 rows (subject + event), others have 1 row (subject only). The model must handle this asymmetry.

---

## Shell T22: Multi-Section Table Where Sections Have Dissimilar Row Structures

### Structural Notes (from shell spec)
- Three top-level sections with DIFFERENT row structures and counting rules:
   - Section 1: Single unfaceted row counting subjects with any TEAE.
   - Section 2: Flat enumeration of severity levels. Each subject counted once at their maximum severity. "TEAEs by Maximum Severity" is a label-only header row.
   - Section 3: Nested SOC→PT hierarchy with group summary counts. "TEAEs by System Organ Class" is a label-only header row. Each subject counted once per SOC and PT.

### The Trap
The three sections look like they could be levels of a single outer variable, but they have **completely different internal structures**: Section 1 is a single row, Section 2 is a flat factor enumeration (worst severity), Section 3 is a nested SOC→PT hierarchy. The model might try to unify them under a single `split_rows_by("SECTION")` which is structurally impossible.

### Reasoning Paths Exercised

**Row Structure — "Differentiating Sections From Facets" (Section 2, Step 1.3)**:
- The three sections scope over **dissimilar substructures**. Per the row_struct_layout.md rule: "If they scope over dissimilar substructures, treat outer labels as indicating top-level sections."
- Tests the model's ability to recognize that semantically related labels (all are AE summaries) with dissimilar internal structures must be modeled as separate top-level sections.

**Select Afuns — Three Different Afuns in One Table**:
- Section 1: `a_freq_j` with flag variable.
- Section 2: `a_maxlev` (or pre-computed worst-severity + `a_freq_j`). No RD columns, so `a_maxlev` is safe.
- Section 3: `a_freq_j` as both cfun and afun in SOC→PT hierarchy.
- Tests whether the model independently selects the correct afun for each section.

**Row Structure — Section Headers via var_labels**:
- "TEAEs by Maximum Severity" and "TEAEs by System Organ Class" are label-only header rows. Tests whether the model uses `var_labels = "...", show_labels = "visible"` (Tier 0) rather than emitting `rcell(NULL)` from a custom afun.

### Hypothesized Weaknesses Probed
- **Forced unification**: Trying to model all three sections as facets of a single split.
- **Wrong afun for severity section**: Using `a_freq_j` for the severity section (which counts subjects per level, not per worst level).
- **rcell(NULL) for headers**: Writing a custom afun that emits a leading `rcell(NULL)` for the section header instead of using `var_labels`.

---

## Shell T23: Table Where Row Labels Look Like Factor Levels But Are Actually Pre-Specified Sections

### Structural Notes (from shell spec)
- Each row is a SEPARATE pre-specified category requiring a DIFFERENT filter condition on ADAE.
- These rows are NOT levels of a single factor variable. The categories are overlapping (a subject counted in "Any serious TEAE" is also counted in "Any TEAE").

### The Trap
The six rows look like levels of a categorical variable (they're all "Any [type] TEAE"), which strongly suggests `split_rows_by("AE_CATEGORY") |> summarize_row_groups()` or `analyze("AE_CATEGORY")`. But there is NO single variable in the data whose levels correspond to these rows — each row requires a **different filter condition** on ADAE. The model must recognize this as Archetype 1 (stacked pre-specified sections), not a data-driven faceting problem.

### Reasoning Paths Exercised

**Row Structure — Section 0 (No-Op Analyze Redirection)**:
- If the model creates a factor variable with these levels and uses `split_rows_by`, it will need a no-op `analyze()` to terminate the chain — triggering the anti-pattern.
- Alternatively, if it uses `analyze("AE_CATEGORY")`, `a_freq_j` will enumerate the factor levels, but the counts will be wrong because the factor was derived by row-binding filtered subsets (double-counting subjects who appear in multiple categories).

**Data Prep — Flag Variable Derivation**:
- Each row requires a separate binary flag: TRTEMFL, SAEFL, DTHFL, DSCFL, RELFL, SAERELFL. These must be derived in data prep on ADAE (and possibly ADSL for the overall count).
- Tests Building Block 7 (pre-computed derived variables).

**Select Afuns — Repeated a_freq_j with Different Flags**:
- Each `analyze()` call uses `a_freq_j` with a different flag variable and `val = "Y"`, `label = "Any [type] TEAE"`.
- Tests whether the model correctly uses the flag-variable pattern six times rather than trying to find a single-variable solution.

### Hypothesized Weaknesses Probed
- **Factor fabrication**: Creating a synthetic factor variable with these levels and using `split_rows_by` or `analyze` on it.
- **Row-binding filtered subsets**: Creating a long-format dataset by stacking filtered ADAE subsets, which double-counts subjects.
- **Single analyze() with multi-level factor**: Passing a factor to `a_freq_j` that has overlapping membership (a subject can be in both "Any TEAE" and "Any serious TEAE").

---

## Shell T24: Table with at_sibling Anchoring Under an Outer Subgroup Split

### Structural Notes (from shell spec)
- Outer split: STRATA1.
- Inside each stratum, three sections: Overall, By Sex, By Age Group. The entire 3-section structure repeats for each stratum.
- "Overall" section has ANCOVA statistics (LS Mean Diff, p-value) while subgroup sections have only descriptive statistics (N, Mean/SD).

### The Trap
The multi-section structure (Overall, By Sex, By Age Group) inside each stratum looks like it should use `nested = FALSE` after each `analyze()` to start a new section. But if the model uses `nested = FALSE`, the "By Sex" and "By Age Group" sections will escape the STRATA1 facet and appear at the root level — breaking the per-stratum repetition.

### Reasoning Paths Exercised

**Row Structure — Section 3.3 (at_sibling for Multi-Section Layouts)**:
- This is the canonical `at_sibling` use case from row_struct_layout.md Section 3.3. The "By Sex" and "By Age Group" sections must use `at_sibling = "CHG"` (or the first analyze's table_name) to stay inside the STRATA1 facet.
- Tests whether the model recognizes the `nested = FALSE` anti-pattern and uses `at_sibling` instead.

**Select Afuns — Different Afuns Per Section**:
- "Overall" section uses `a_summarize_ancova_j` (LS Mean Diff, p-value).
- Subgroup sections use `tern::a_summary` (descriptive only, no comparison).
- Tests whether the model correctly selects different afuns for different sections within the same outer split.

**Row Structure — Deepest-First Ordering**:
- If both "By Sex" and "By Age Group" anchor to the same sibling, the ordering rule (deepest anchor first) applies. Tests whether the model follows this rule.

### Hypothesized Weaknesses Probed
- **nested = FALSE escape**: Using default `nested = FALSE` after `analyze()`, causing sections to escape the stratum.
- **Missing at_sibling**: Not knowing about `at_sibling` and trying to work around the nesting problem with a custom split function or by restructuring the data.
- **Wrong anchor name**: Anchoring to the wrong sibling name (e.g., using the variable name instead of the table_name).

---

## Shell T25: Table Where summarize_row_groups Appears to Be the Only Content

### Structural Notes (from shell spec)
- These are leaf rows with cell values — there are no nested rows under any of them.

### The Trap
The four rows (Randomized, Treated, Completed, Discontinued) look like facet labels with summary counts — strongly suggesting `split_rows_by("STATUS") |> summarize_row_groups()`. But there are NO nested rows under these labels. If the model uses `split_rows_by` + `summarize_row_groups`, it must add a terminating `analyze()` that produces no meaningful content — the **Standalone Marginal Summary Anti-Pattern**.

### Reasoning Paths Exercised

**Row Structure — Standalone Marginal Summary Anti-Pattern**:
- This is the most direct test of the anti-pattern. The model must recognize that these are leaf rows with cell values, not facet headers with nested content.
- The correct approach is stacked `analyze()` calls with flag variables (Archetype 1), or a single `analyze()` on a multi-level factor where each level is a population flag.

**Row Structure — Section 0 (No-Op Analyze Redirection)**:
- If the model uses `split_rows_by("POPFL") |> summarize_row_groups()`, it needs a terminating `analyze()`. Any `analyze()` here would be a no-op — the anti-pattern.

**Select Afuns — Flag Variables**:
- Each row requires a different flag: RANDFL, SAFFL, EOSSTT == "COMPLETED", EOSSTT == "DISCONTINUED".
- Tests whether the model derives these flags and uses `a_freq_j` with `val = "Y"` and custom labels.

### Hypothesized Weaknesses Probed
- **split_rows_by + summarize_row_groups without analyze**: The most likely error — using the structural pattern for facet headers when the rows are actually leaf analysis rows.
- **No-op analyze**: Adding `analyze("STUDYID", afun = ac_blank_line)` or similar to satisfy the structural requirement.
- **Denominator confusion**: "Percentages based on randomized subjects" means `denom` should reference the randomized population, not the safety population. If ADSL is filtered to safety population, the denominator for "Randomized" is wrong.

---

## Shell T26: Table with Derived Variable Needed in Both df and alt_counts_df

### Structural Notes (from shell spec)
- Column structure: spanning headers with risk difference columns (same as standard TSFAE02).

### The Trap
The table filters ADAE to drug-related AEs only, but percentages use the FULL arm N from ADSL. The model must:
1. Filter ADAE to AEREL == "RELATED"
2. Derive TRTEMFL = "Y" on the filtered ADAE
3. Ensure the colspan_trt spanning variable exists on BOTH filtered ADAE and ADSL
4. Re-synchronize factor levels after filtering
5. Pass alt_counts_df = ADSL (unfiltered) to build_table

The trap is that after filtering ADAE, factor levels of ARM may have stale levels, and the colspan_trt variable created on ADSL must be joined onto the filtered ADAE with matching levels.

### Reasoning Paths Exercised

**Orchestrator Contract 1 (Dataset Synchronization)**:
- Every column split variable must exist on both df and alt_counts_df with identical factor levels. After filtering ADAE, levels may be stale.
- Tests the "Factor Level Hygiene After Filtering and Joining" guidance.
- Specific steps required: (a) filter ADAE to AEREL == "RELATED"; (b) derive a TRTEMFL = "Y" flag on the filtered ADAE; (c) create colspan_trt on ADSL via `create_colspan_var`; (d) join colspan_trt onto filtered ADAE; (e) re-synchronize ARM/TRT01A factor levels between filtered ADAE and ADSL via `factor(..., levels = levels(adsl$...))`. The TRTEMFL flag must also exist on ADSL for the overall count row to work with `a_freq_j`'s `.alt_df_full` lookup.

**Custom Afuns — Building Block 7 (Pre-Computed Derived Variables)**:
- The TRTEMFL flag must be derived on the filtered ADAE in data prep, not inside an afun. If derived inside an afun, it won't be visible to `.df_row` or `.alt_df_full`.

**Column Structure — Contract 1 with grouped_cols_w_diffs**:
- `create_colspan_var` must be called on ADSL first, then the colspan_trt column must be joined onto the filtered ADAE. Mismatched factor levels cause `create_colspan_map` errors.

### Hypothesized Weaknesses Probed
- **Stale factor levels**: Not re-applying `factor()` after filtering ADAE, leaving empty ARM levels.
- **Missing colspan_trt on filtered ADAE**: Creating colspan_trt on ADSL but forgetting to join it onto ADAE.
- **Deriving TRTEMFL inside afun**: Using `mutate()` inside a custom afun instead of data prep.
- **Wrong denominator**: Using `denom = "N_col"` without `alt_counts_df`, which would count only subjects with drug-related AEs.

---

## Shell T27: Table Where drop_split_levels on Inner Split Creates Ragged Structure

### Structural Notes (from shell spec)
- 3-level hierarchy: SOC → PT → Severity (AESEV).
- Severity levels are shown ONLY when observed for that PT — different PTs may show different severity levels.

### The Trap
The model has been trained (via row_struct_layout.md Section 5.2) to be WARY of `drop_split_levels` on inner splits because it creates ragged structures. But in this table, the ragged structure is **intentional and correct** — severity levels should only appear when observed for that PT. The model might over-correct and use `keep_split_levels(c("Mild", "Moderate", "Severe"))` on the inner split, which would show empty severity rows under PTs where those severities don't occur.

### Reasoning Paths Exercised

**Row Structure — Section 5.2 (drop_split_levels Pitfall — Inverted)**:
- This is the INVERSE of the documented pitfall. The pitfall warns against `drop_split_levels` on inner splits when a UNIFORM inner scale is desired. Here, a RAGGED inner scale IS desired because severity is hierarchically dependent on the PT (only observed severities should appear). Using `drop_split_levels` on the innermost AESEV split is CORRECT here.
- Tests whether the model can distinguish between the two cases: uniform inner scale (use `keep_split_levels`) vs. hierarchically dependent inner variable (use `drop_split_levels` or `trim_levels_in_group`).

**Select Afuns — a_maxlev in 3-Level Hierarchy**:
- Each subject counted at maximum severity per PT. No RD columns, so `a_maxlev` is safe.
- Tests whether the model correctly identifies that `a_maxlev` works here (no difference columns) despite the documented limitation.

**Row Structure — 3-Level Nesting**:
- SOC → PT → AESEV with `trim_levels_in_group` on the outer splits and `drop_split_levels` on the innermost split.
- Tests correct chaining of split functions across 3 levels.

### Hypothesized Weaknesses Probed
- **Over-correction from Section 5.2 warning**: Using `keep_split_levels` on the inner AESEV split, producing empty severity rows.
- **Confusing the two cases**: Not recognizing that the Section 5.2 warning applies to UNIFORM inner scales (like SEX under RACE), not to hierarchically dependent inner variables (like severity under PT).
- **Wrong counting mechanism**: Using standard `a_freq_j` instead of `a_maxlev` for the severity breakdown (counting subjects per severity level rather than per worst severity).

---

## Shell T28: Multi-Row-Generating `cfun` at the SOC Level (`child_labels = "visible"`, Doubly-Indented Leaf PTs, and Custom Content-Row Sorting)

### Structural Notes (from shell spec)
- 2-level hierarchy: `AEBODSYS` (SOC) → `AEDECOD` (PT) on `adae` (`SAFFL == "Y"`, `TRTEMFL == "Y"`, restricted to `Cardiac disorders` and `Eye disorders`) with `alt_counts_df = adsl`.
- Each SOC displays its own visible facet label (`Cardiac disorders`, `Eye disorders`), immediately followed by a **2-row group summary (`cfun`)**:
  1. `Subjects with >=1 TEAE` — unique subjects `xx (xx.x%)` (`count_unique_fraction`).
  2. `Total number of TEAEs` — total event count `xx` (`count`).
- Preferred terms are doubly indented (`indent_mod = 1L`) beneath the 2-row SOC summary.
- Top-left corner has 3 hierarchical lines (`System Organ Class`, `  SOC Summary`, `    Preferred Term`) via `append_topleft()`.
- Both SOCs and PTs within each SOC are sorted by descending unique subject incidence in the first column (`Xanomeline Low Dose`).

### The Trap
This shell compounds **four interacting `rtables` structural traps** triggered by a multi-row `cfun`:
1. **`child_labels = "default"` hides the SOC name when `summarize_row_groups()` is present**: In standard single-row `summarize_row_groups("AEBODSYS", cfun = a_freq_j)`, `child_labels = "default"` (or `"hidden"`) works because the single content row is automatically relabeled with the SOC name (`labelstr`). When a custom `cfun` returns **2 rows** (`Subjects with >=1 TEAE` and `Total number of TEAEs`) via `c(r_subj, r_evt)`, neither row is named after the SOC! If the programmer leaves `child_labels = "default"` or sets `child_labels = "hidden"` (out of habit from standard SOC→PT tables), **the SOC names (`Cardiac disorders`, `Eye disorders`) vanish completely from the table!** `child_labels = "visible"` is mandatory on `split_rows_by("AEBODSYS")`.
2. **Sibling `analyze()` vs. `summarize_row_groups(cfun = ...)` trap**: A model may try to avoid writing a 2-row `cfun` by using two `analyze()` calls before `AEDECOD` inside `AEBODSYS`. That creates `ElementaryTable` children instead of a `@content` table on the `AEBODSYS` node, breaking standard content-table semantics and indentation.
3. **Leaf PT indentation collision**: Because `child_labels = "visible"` indents the 2-row `cfun` by 1 level (2 spaces) under the SOC label, a default `analyze("AEDECOD", indent_mod = 0L)` places PTs at the **same** indentation level as `Subjects with >=1 TEAE` and `Total number of TEAEs`. Explicit `indent_mod = 1L` on `analyze("AEDECOD")` is required so PTs indent beneath the SOC summary rows.
4. **Post-processing sort crash with `jj_complex_scorefun()` on a multi-row `@content` table**: Standard `sort_at_path(path = c("AEBODSYS"), scorefun = jj_complex_scorefun(...))` assumes a 1-row content table (or specific structure) at `AEBODSYS`. With a 2-row content table, sorting `AEBODSYS` requires a custom `scorefun` that extracts row 1 (`tree_children(content_table(tt))[[1]]`) of the content table, while `jj_complex_scorefun` still works at `c("AEBODSYS", "*", "AEDECOD")`.

### Reasoning Paths Exercised
- **Row Structure (`row_struct_layout.md` Section 2.2)**: Interaction between `child_labels = "visible"`, `summarize_row_groups()`, and `indent_mod` when a `cfun` emits multiple labeled rows instead of a single facet-named content row.
- **Custom Afuns (`custom_afuns.md` Section 3 — Tier 3 Stacking)**: Combining two `a_freq_j` calls (`count_unique_fraction` on `SOC_SUBJ` and `count` on `SOC_EVT`) via `c(r_subj, r_evt)` inside a `cfun`, forwarding `.N_col`, `.alt_df_full`, `.df_row`, and `.spl_context`.
- **Sorting & Pruning (`introspection_sort_prune.md`)**: Writing a custom `scorefun` (`content_table(tt)` → first child row → `cell_values(ct, rowpath = ..., colpath = first_col)[[1]][1]`) when the target `TableTree` node has a multi-row content table.

### Hypothesized Weaknesses Probed
- **Missing SOC headers**: Using `child_labels = "hidden"` or `"default"` on `split_rows_by("AEBODSYS")` with a 2-row `cfun`.
- **Flat indentation**: Forgetting `indent_mod = 1L` on `analyze("AEDECOD")`.
- **Invalid `basic_table(topleft = ...)` argument**: Passing `topleft` to `basic_table()` instead of piping `|> append_topleft(...)`.
- **Sort failure on multi-row content table**: Blindly applying `cont_n_onecol` or `jj_complex_scorefun` to `path = c("AEBODSYS")` without testing against a 2-row content table.

### Reference `rtables` Output (Exact)
```
TSFAE-MULTICFUN: TEAEs by SOC (with Dual Subject/Event Summaries) and PT; Safety Analysis Set

———————————————————————————————————————————————————————————————————————————————————————————————————
System Organ Class                                                                                 
  SOC Summary                                Xanomeline Low Dose   Xanomeline High Dose    Placebo 
    Preferred Term                                  N=67                   N=51             N=64   
———————————————————————————————————————————————————————————————————————————————————————————————————
Cardiac disorders                                                                                  
  Subjects with >=1 TEAE                         10 (14.9%)             10 (19.6%)        9 (14.1%)
  Total number of TEAEs                              24                     21               21    
      Sinus bradycardia                           6 (9.0%)              6 (11.8%)         1 (1.6%) 
      Myocardial infarction                       2 (3.0%)               2 (3.9%)         4 (6.2%) 
      Atrial fibrillation                         1 (1.5%)               1 (2.0%)         1 (1.6%) 
      Atrial flutter                              1 (1.5%)               1 (2.0%)             0    
      Atrioventricular block first degree         1 (1.5%)                  0                 0    
      Bundle branch block right                   1 (1.5%)                  0             1 (1.6%) 
      Palpitations                                1 (1.5%)                  0                 0    
      Supraventricular extrasystoles              1 (1.5%)               1 (2.0%)         1 (1.6%) 
      Supraventricular tachycardia                1 (1.5%)                  0                 0    
      Ventricular extrasystoles                   1 (1.5%)               1 (2.0%)             0    
      Atrial hypertrophy                              0                     0             1 (1.6%) 
      Atrioventricular block second degree            0                     0             1 (1.6%) 
      Bradycardia                                     0                     0             1 (1.6%) 
      Bundle branch block left                        0                     0             1 (1.6%) 
      Cardiac failure congestive                      0                     0             1 (1.6%) 
      Tachycardia                                     0                     0             1 (1.6%) 
Eye disorders                                                                                      
  Subjects with >=1 TEAE                          2 (3.0%)               1 (2.0%)         1 (1.6%) 
  Total number of TEAEs                               2                     2                 3    
      Conjunctival haemorrhage                    1 (1.5%)                  0                 0    
      Vision blurred                              1 (1.5%)               1 (2.0%)             0    
      Eye allergy                                     0                     0             1 (1.6%) 
      Eye pruritus                                    0                     0             1 (1.6%) 
      Eye swelling                                    0                     0             1 (1.6%) 
———————————————————————————————————————————————————————————————————————————————————————————————————

Note: Each SOC displays unique subjects with >=1 TEAE and total TEAE occurrences, followed by preferred terms (unique subjects).
```

---

## Shell T29: Inside-Out Multi-Depth `at_sibling` Anchoring (`PARAM_LBL` → `AVISIT` + Sibling `SEX_LBL` Split + Outer Sibling Shift Analysis)

### Structural Notes (from shell spec)
- Outer row split: `PARAM_LBL` (`Alanine Aminotransferase (U/L)`, `Aspartate Aminotransferase (U/L)`).
- Inside each `PARAM_LBL`, there are **two sibling branches at Depth 1**:
  1. `split_rows_by("AVISIT", label_pos = "hidden")` (`Baseline`, `Cycle 02`).
  2. `analyze("WORST_ANRIND", var_labels = "Post-Baseline Worst Status, n (%)", show_labels = "visible", at_sibling = "AVISIT")` (`LOW`, `NORMAL`, `HIGH`).
- Inside each `AVISIT` (Depth 2), there are **two sibling branches**:
  1. `analyze("AVAL", afun = a_summarize_ex_j, table_names = "visit_overall")` (`N`, `Mean (SD)`).
  2. `split_rows_by("SEX_LBL", split_label = "By Sex", label_pos = "visible", at_sibling = "visit_overall")` → `analyze("AVAL", afun = a_summarize_ex_j)` (`Mean (SD)`).

### The Trap
This shell is an **"Inside-Out" Multi-Depth `at_sibling` maze** with three distinct structural pitfalls:
1. **Two `at_sibling` hops at different nesting depths**:
   - First, after `analyze("AVAL", table_names = "visit_overall")` at depth 3 (`PARAM_LBL → AVISIT → visit_overall`), the layout must step back 1 level to `AVISIT` using `split_rows_by("SEX_LBL", ..., at_sibling = "visit_overall")` so `By Sex` is a sibling of `visit_overall` inside each `AVISIT`.
   - Next, after `analyze("AVAL", table_names = "visit_sex")` at depth 4 (`PARAM_LBL → AVISIT → SEX_LBL → visit_sex`), the layout must step back **2 levels** to `PARAM_LBL` using `analyze("WORST_ANRIND", ..., at_sibling = "AVISIT")` so `Post-Baseline Worst Status, n (%)` is a sibling of `AVISIT` inside each `PARAM_LBL`!
   - Using `nested = FALSE` instead of `at_sibling = "AVISIT"` is fatal: it escapes all the way to the root of the table and places `Post-Baseline Worst Status, n (%)` once at the very bottom of the entire table instead of inside each `PARAM_LBL`!
2. **Undocumented `rtables` `at_sibling` auto-visibility side-effect on `AVISIT`**: When `analyze(..., at_sibling = "AVISIT")` references a `split_rows_by("AVISIT")` node whose `label_pos` was left at its default (`"hidden"` implicitly or `"default"`), `rtables` automatically flips the referenced split's `label_pos` to `"visible"` so the two sibling sections under `PARAM_LBL` have section headers — which prints a literal `"AVISIT"` row above `Baseline`! To prevent `"AVISIT"` from appearing as an unwanted row header, `split_rows_by("AVISIT", label_pos = "hidden")` **must** explicitly specify `label_pos = "hidden"`.
3. **Mixed Longitudinal Continuous + Subject-Level Shift on the Same Dataset**: `AVAL` is analyzed at `Baseline` and `Cycle 02`, whereas `WORST_ANRIND` is derived across **all** post-baseline records (`ABLFL != "Y"`) for that subject and parameter. If the programmer filters `adlb` to `AVISIT %in% c("Baseline", "Cycle 02")` *before* deriving `WORST_ANRIND`, or forgets to include subjects who only have post-baseline records outside `Baseline`/`Cycle 02` (or doesn't restrict `AVAL` inside the visit split), either the shift counts or the visit summaries are corrupted.

### Reasoning Paths Exercised
- **Row Structure (`row_struct_layout.md` Section 3 — Non-Linear Tree Insertion with `at_sibling`)**: Navigating across multiple tree depths using `at_sibling = "visit_overall"` (depth 2) and `at_sibling = "AVISIT"` (depth 1) without escaping `PARAM_LBL`.
- **Row Structure (`row_struct_layout.md` Section 2.2 — `label_pos = "hidden"`)**: Explicitly locking `label_pos = "hidden"` on a `split_rows_by()` node that will later be targeted by `at_sibling`.
- **Data Prep & Custom Afuns**: Combining visit-filtered continuous summaries (`a_summarize_ex_j`) with a parameter-level subject worst-shift categorical summary (`a_freq_j` with `denom = "n_altdf"`).

### Hypothesized Weaknesses Probed
- **Using `nested = FALSE` instead of `at_sibling = "AVISIT"`**: Escaping the `PARAM_LBL` split entirely.
- **Spurious `"AVISIT"` label row**: Omitting explicit `label_pos = "hidden"` on `split_rows_by("AVISIT")` when targeting it with `at_sibling = "AVISIT"`.
- **Wrong `at_sibling` target name**: Targeting `"visit_overall"` or `"SEX_LBL"` instead of `"AVISIT"` for the `WORST_ANRIND` block, which nests `Post-Baseline Worst Status` inside every visit instead of once per parameter.

### Reference `rtables` Output (Exact)
```
TSFLAB-ATSIB2D: Laboratory Summary by Visit, Sex Subgroup, and Post-Baseline Shift; Safety Analysis Set

——————————————————————————————————————————————————————————————————————————————————————————————
                                      Xanomeline Low Dose   Xanomeline High Dose     Placebo  
                                             N=67                   N=51              N=64    
——————————————————————————————————————————————————————————————————————————————————————————————
Alanine Aminotransferase (U/L)                                                                
  Baseline                                                                                    
    N                                         67                     51                64     
    Mean (SD)                             17.6 (9.2)            19.5 (10.5)        17.3 (7.4) 
    By Sex                                                                                    
      Female                                                                                  
        Mean (SD)                         16.8 (9.7)             16.5 (4.5)        17.4 (8.2) 
      Male                                                                                    
        Mean (SD)                         19.0 (8.4)            22.2 (13.3)        17.2 (5.9) 
  Cycle 02                                                                                    
    N                                         61                     49                61     
    Mean (SD)                             20.5 (11.6)            21.8 (8.7)        17.9 (13.2)
    By Sex                                                                                    
      Female                                                                                  
        Mean (SD)                         19.7 (13.1)            18.9 (5.7)        18.7 (16.0)
      Male                                                                                    
        Mean (SD)                         21.8 (8.9)            24.2 (10.0)        16.7 (6.3) 
  Post-Baseline Worst Status, n (%)                                                           
    LOW                                        0                     0                  0     
    NORMAL                                44 (65.7%)             43 (84.3%)        56 (87.5%) 
    HIGH                                   5 (7.5%)              8 (15.7%)          6 (9.4%)  
Aspartate Aminotransferase (U/L)                                                              
  Baseline                                                                                    
    N                                         67                     51                64     
    Mean (SD)                             22.5 (8.2)             23.5 (6.7)        23.2 (6.7) 
    By Sex                                                                                    
      Female                                                                                  
        Mean (SD)                         22.6 (8.9)             23.2 (5.8)        23.9 (7.1) 
      Male                                                                                    
        Mean (SD)                         22.2 (7.1)             23.6 (7.5)        21.9 (5.8) 
  Cycle 02                                                                                    
    N                                         61                     49                61     
    Mean (SD)                             24.0 (8.2)             23.8 (4.8)        23.7 (13.6)
    By Sex                                                                                    
      Female                                                                                  
        Mean (SD)                         23.8 (9.6)             24.4 (4.9)        24.9 (16.8)
      Male                                                                                    
        Mean (SD)                         24.5 (5.6)             23.3 (4.8)        21.7 (5.0) 
  Post-Baseline Worst Status, n (%)                                                           
    LOW                                        0                     0                  0     
    NORMAL                                43 (64.2%)             45 (88.2%)        56 (87.5%) 
    HIGH                                   6 (9.0%)              6 (11.8%)          6 (9.4%)  
——————————————————————————————————————————————————————————————————————————————————————————————

Note: Each visit displays overall summary statistics followed by sex-specific Mean (SD).
Post-Baseline Worst Reference Range Status summarizes each subject once per parameter below all visits.
```

---

## Shell T30: Parallel Non-Nested Column Banners (`All Subjects` + `Treatment Group` + `Sex Subgroup` + `Age Group (Years)`) via `add_overall_col()` Composition

### Structural Notes (from shell spec)
- 4 parallel, non-nested top-level column banners over the same 182 safety subjects:
  1. `All Subjects` (single overall column, `N=182`)
  2. `Treatment Group` spanning `Xanomeline Low Dose` (`N=67`), `Xanomeline High Dose` (`N=51`), `Placebo` (`N=64`)
  3. `Sex Subgroup` spanning `Female` (`N=106`), `Male` (`N=76`)
  4. `Age Group (Years)` spanning `<65` (`N=25`), `65 to <75` (`N=50`), `>=75` (`N=107`)
- Programming Note 1 explicitly forbids duplicating or row-binding (`bind_rows`) `ADSL`/`ADAE`.
- Row structure: Top-level `Subjects with >=1 TEAE in Selected SOCs` (`analyze("ANY_TEAE")`), followed by `AEBODSYS → AEDECOD` sorted by the `All Subjects` column.

### The Trap
How do you create **parallel, non-nested column banners** in `rtables` where each banner has its own spanning header (`Treatment Group`, `Sex Subgroup`, `Age Group (Years)`) without duplicating the dataset or Cartesian-nesting `TRT01A × SEX_LBL × AGE_GRP`?
1. **The `split_cols_by(..., nested = FALSE)` spanning-header trap**: If a programmer writes:
   ```r
   split_cols_by("TRT01A", nested = FALSE) |>
   split_cols_by("SEX_LBL", nested = FALSE) |>
   split_cols_by("AGE_GRP", nested = FALSE)
   ```
   the columns sit side-by-side at depth 1, **with NO spanning headers** (`Treatment Group`, `Sex Subgroup`, `Age Group (Years)` are completely missing!). `split_cols_by()` does not render `split_label` as a column spanning header.
2. **The `add_overall_col("Banner") |> split_cols_by("VAR")` idiom (`col_struct_layout.md` Section 5)**: Because `add_overall_col("Banner")` always inserts a new top-level overall column at the root (`nested = FALSE` by default) and `split_cols_by("VAR")` defaults to `nested = TRUE` (nesting underneath the most recently added column node!), chaining:
   ```r
   add_overall_col("All Subjects") |>
   add_overall_col("Treatment Group") |>
   split_cols_by("TRT01A") |>
   add_overall_col("Sex Subgroup") |>
   split_cols_by("SEX_LBL") |>
   add_overall_col("Age Group (Years)") |>
   split_cols_by("AGE_GRP")
   ```
   creates `All Subjects` as a standalone leaf column, and turns `"Treatment Group"`, `"Sex Subgroup"`, and `"Age Group (Years)"` into parallel spanning banners over their respective split variables — **with zero data duplication**!
3. **Contract 1 across all 3 split variables**: `TRT01A`, `SEX_LBL`, and `AGE_GRP` must all exist with identical factor levels on both `adsl` (`alt_counts_df`) and `adae` (`df`).

### Reasoning Paths Exercised
- **Column Structure (`col_struct_layout.md` Section 5 — Non-Nested Parallel Column Splits with Spanning Headers)**: Combining `add_overall_col("Banner Name")` (`nested = FALSE`) with `split_cols_by("VAR")` (`nested = TRUE`) to build parallel multi-variable banners over unmodified `ADSL`/`ADAE`.
- **Column Structure (Contract 1)**: Joining `SEX_LBL` and `AGE_GRP` from `adsl` onto `adae` with matching factor levels so all 9 leaf columns partition both `df` and `alt_counts_df` identically.
- **Sorting (`introspection_sort_prune.md`)**: Extracting `first_col <- col_paths(tbl)[[1]]` (`c("All Subjects", "All Subjects")`) and sorting SOCs and PTs via `jj_complex_scorefun(colpath = first_col)`.

### Hypothesized Weaknesses Probed
- **Cartesian column explosion**: Forgetting `add_overall_col()` or `nested = FALSE` and nesting `TRT01A → SEX_LBL → AGE_GRP` into 1 + 3×2×3 = 19 columns.
- **Missing spanning headers**: Using `split_cols_by("TRT01A", nested = FALSE)` directly, which omits `"Treatment Group"`, `"Sex Subgroup"`, and `"Age Group (Years)"` from the column header tree.
- **Violating Programming Note 1**: Row-binding `ADSL` and `ADAE` 4 times (`bind_rows`) to fake a spanning banner variable.

### Reference `rtables` Output (Exact)
```
TSFAE-BANNER: TEAEs by Parallel Treatment, Sex, and Age Group Banners; Safety Analysis Set

———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                                                             Treatment Group                            Sex Subgroup                 Age Group (Years)         
                                          All Subjects   Xanomeline Low Dose   Xanomeline High Dose    Placebo       Female        Male         <65      65 to <75      >=75   
                                             N=182              N=67                   N=51              N=64        N=106         N=76        N=25        N=50        N=107   
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Subjects with >=1 TEAE in Selected SOCs    32 (17.6%)        11 (16.4%)             11 (21.6%)        10 (15.6%)   16 (15.1%)   16 (21.1%)   3 (12.0%)   5 (10.0%)   24 (22.4%)
Cardiac disorders                          29 (15.9%)        10 (14.9%)             10 (19.6%)        9 (14.1%)    15 (14.2%)   14 (18.4%)   3 (12.0%)   4 (8.0%)    22 (20.6%)
  Sinus bradycardia                        13 (7.1%)          6 (9.0%)              6 (11.8%)          1 (1.6%)     5 (4.7%)    8 (10.5%)    1 (4.0%)    4 (8.0%)     8 (7.5%) 
  Myocardial infarction                     8 (4.4%)          2 (3.0%)               2 (3.9%)          4 (6.2%)     7 (6.6%)     1 (1.3%)        0       1 (2.0%)     7 (6.5%) 
  Atrial fibrillation                       3 (1.6%)          1 (1.5%)               1 (2.0%)          1 (1.6%)     1 (0.9%)     2 (2.6%)        0           0        3 (2.8%) 
  Supraventricular extrasystoles            3 (1.6%)          1 (1.5%)               1 (2.0%)          1 (1.6%)     3 (2.8%)        0            0           0        3 (2.8%) 
  Atrial flutter                            2 (1.1%)          1 (1.5%)               1 (2.0%)             0            0         2 (2.6%)        0           0        2 (1.9%) 
  Bundle branch block right                 2 (1.1%)          1 (1.5%)                  0              1 (1.6%)     1 (0.9%)     1 (1.3%)        0           0        2 (1.9%) 
  Ventricular extrasystoles                 2 (1.1%)          1 (1.5%)               1 (2.0%)             0         1 (0.9%)     1 (1.3%)        0           0        2 (1.9%) 
  Atrial hypertrophy                        1 (0.5%)              0                     0              1 (1.6%)        0         1 (1.3%)    1 (4.0%)        0           0     
  Atrioventricular block first degree       1 (0.5%)          1 (1.5%)                  0                 0            0         1 (1.3%)        0           0        1 (0.9%) 
  Atrioventricular block second degree      1 (0.5%)              0                     0              1 (1.6%)        0         1 (1.3%)    1 (4.0%)        0           0     
  Bradycardia                               1 (0.5%)              0                     0              1 (1.6%)     1 (0.9%)        0            0       1 (2.0%)        0     
  Bundle branch block left                  1 (0.5%)              0                     0              1 (1.6%)     1 (0.9%)        0            0           0        1 (0.9%) 
  Cardiac failure congestive                1 (0.5%)              0                     0              1 (1.6%)     1 (0.9%)        0            0           0        1 (0.9%) 
  Palpitations                              1 (0.5%)          1 (1.5%)                  0                 0            0         1 (1.3%)        0       1 (2.0%)        0     
  Supraventricular tachycardia              1 (0.5%)          1 (1.5%)                  0                 0            0         1 (1.3%)        0           0        1 (0.9%) 
  Tachycardia                               1 (0.5%)              0                     0              1 (1.6%)     1 (0.9%)        0            0           0        1 (0.9%) 
Eye disorders                               4 (2.2%)          2 (3.0%)               1 (2.0%)          1 (1.6%)     2 (1.9%)     2 (2.6%)        0       1 (2.0%)     3 (2.8%) 
  Vision blurred                            2 (1.1%)          1 (1.5%)               1 (2.0%)             0         1 (0.9%)     1 (1.3%)        0       1 (2.0%)     1 (0.9%) 
  Conjunctival haemorrhage                  1 (0.5%)          1 (1.5%)                  0                 0         1 (0.9%)        0            0           0        1 (0.9%) 
  Eye allergy                               1 (0.5%)              0                     0              1 (1.6%)        0         1 (1.3%)        0           0        1 (0.9%) 
  Eye pruritus                              1 (0.5%)              0                     0              1 (1.6%)        0         1 (1.3%)        0           0        1 (0.9%) 
  Eye swelling                              1 (0.5%)              0                     0              1 (1.6%)        0         1 (1.3%)        0           0        1 (0.9%) 
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Columns represent parallel non-nested banners over the same Safety Analysis Set.
Percentages are based on column N within each banner facet.
```

---

## Shell T31: Asymmetric Multi-Variable Visit Columns (`make_split_fun` + `restrict_facets` Pruning `CFB` at `Baseline`) + Multi-Row `cfun` Interacting with `split_cols_by_multivar`

### Structural Notes (from shell spec)
- Column hierarchy: `AVISIT` (`Baseline`, `Cycle 02`, `Cycle 04`) → `c("AVAL", "CHG")` (`Observed`, `CFB`), with **`CFB` structurally pruned under `Baseline`** so `Baseline` has 1 sub-column (`Observed`) and `Cycle 02` / `Cycle 04` have 2 sub-columns (`Observed`, `CFB`).
- Row hierarchy: `PARAM` (`Systolic Blood Pressure (mmHg)`, `Diastolic Blood Pressure (mmHg)`) → `TRT01A` (`Xanomeline Low Dose`, `Xanomeline High Dose`, `Placebo`).
- Each `TRT01A` facet has a visible header (`child_labels = "visible"`), followed by a **2-row group summary (`cfun`)** (`n` and `Min, Max`) and doubly-indented leaf rows (`Mean (SD)` and `Median`, `indent_mod = 1L`).

### The Trap
This shell combines **asymmetric multi-variable column splitting** with **multi-row `cfun` dispatch across `split_cols_by_multivar`**:
1. **Pruning `CFB` under `Baseline` in `split_cols_by_multivar()`**: `drop_split_levels` cannot drop `CHG` under `Baseline` in `split_cols_by_multivar(c("AVAL", "CHG"))` because `split_cols_by_multivar` does not split by factor levels — it creates one facet per variable regardless of `NA`s! The programmer must use a custom post-processing step via `make_split_fun(post = list(drop_cfb_at_baseline))` that checks `tail(.spl_context$value, 1) == "Baseline"` and applies `restrict_facets("AVAL", op = "keep")(ret, spl, fulldf)` (`col_struct_layout.md` Section 7.2).
2. **Active column variable lookup inside `summarize_row_groups(cfun = ...)` under `split_cols_by_multivar()`**: When `analyze_colvars(afun = ...)` is used at the leaf level, `rtables` passes each column's variable (`AVAL` or `CHG`) to `afun`. However, in `summarize_row_groups("TRT01A", cfun = cfun_trt_n_range)`, `.var` is `"TRT01A"` (the row-split variable), NOT `"AVAL"` or `"CHG"`! If `cfun_trt_n_range` inspects `df[[.var]]`, it gets the factor `TRT01A` and crashes on `min(x)` / `max(x)`! And if it hardcodes `df$AVAL`, then the `CFB` columns in `n` and `Min, Max` secretly report `Observed` (`AVAL`) values instead of `CHG` values! The `cfun` must inspect `.spl_context$cur_col_id` (e.g. checking whether `"AVAL"` or `"CHG"` is the active column facet) to select `df[["AVAL"]]` vs `df[["CHG"]]`!
3. **`child_labels = "visible"` + `indent_mod = 1L` on `analyze_colvars()`**: Just as in T28, because `cfun_trt_n_range` returns 2 rows (`n`, `Min, Max`), `split_rows_by("TRT01A")` must set `child_labels = "visible"` so the treatment arm names remain visible, and `analyze_colvars()` must set `indent_mod = 1L` so `Mean (SD)` and `Median` indent beneath `n` and `Min, Max`.

### Reasoning Paths Exercised
- **Column Structure (`col_struct_layout.md` Section 7.2 — Asymmetric Column Tree via `make_split_fun` + `restrict_facets`)**: Conditionally pruning the `CHG` (`CFB`) column facet when the parent `AVISIT` facet is `"Baseline"`.
- **Custom Afuns (`custom_afuns.md` — `.spl_context` introspection)**: Reading `tail(.spl_context$cur_col_id, 1)` inside a `cfun` to determine which column variable (`AVAL` vs `CHG`) is active in the current column cell.
- **Row Structure (`row_struct_layout.md`)**: Combining `child_labels = "visible"`, 2-row `summarize_row_groups()`, `analyze_colvars(..., indent_mod = 1L)`, and `append_topleft(c("Parameter", "  Treatment Group"))`.

### Hypothesized Weaknesses Probed
- **Using `drop_split_levels` on `split_cols_by_multivar`**: Leaves an empty or `NA` `Baseline -> CFB` column in the table instead of pruning it to 5 columns total.
- **Hardcoding `df$AVAL` or using `df[[.var]]` inside `cfun_trt_n_range`**: Either crashing on `min(df[["TRT01A"]])` or reporting `AVAL` range in the `CFB` columns.
- **Disappeared `TRT01A` headers**: Leaving `child_labels = "default"` on `split_rows_by("TRT01A")` with a 2-row `cfun`.

### Reference `rtables` Output (Exact)
```
TSFVS-LONGCB: Vital Signs Across Longitudinal Visit Columns with Conditional CFB; Safety Analysis Set

————————————————————————————————————————————————————————————————————————————————————————————————————————
Parameter                           Baseline              Cycle 02                     Cycle 04         
  Treatment Group                   Observed       Observed         CFB         Observed         CFB    
————————————————————————————————————————————————————————————————————————————————————————————————————————
Systolic Blood Pressure (mmHg)                                                                          
  Xanomeline Low Dose                                                                                   
    n                                  67             66            66             42            42     
    Min, Max                      103.3, 181.3   96.7, 173.3    -27.3, 26.0   101.3, 169.3   -28.3, 22.0
        Mean (SD)                 137.1 (16.3)   135.0 (16.9)   -1.8 (12.8)   134.6 (17.2)   -1.6 (11.7)
        Median                       136.7          132.7          -0.3          132.3           0.7    
  Xanomeline High Dose                                                                                  
    n                                  51             51            51             47            47     
    Min, Max                      100.0, 179.3   104.0, 178.3   -49.3, 31.7   97.3, 164.0    -48.7, 38.0
        Mean (SD)                 137.4 (17.3)   131.5 (15.0)   -6.0 (16.2)   130.3 (16.0)   -8.4 (15.8)
        Median                       138.0          132.0          -5.3          129.3          -8.3    
  Placebo                                                                                               
    n                                  63             62            61             55            54     
    Min, Max                      86.7, 181.3    92.7, 166.7    -32.0, 35.3   88.0, 203.3    -38.7, 48.0
        Mean (SD)                 136.8 (17.7)   133.6 (16.1)   -3.6 (13.1)   134.1 (19.1)   -2.0 (16.0)
        Median                       136.0          131.7          -2.3          131.3          -2.0    
Diastolic Blood Pressure (mmHg)                                                                         
  Xanomeline Low Dose                                                                                   
    n                                  67             66            66             42            42     
    Min, Max                       57.3, 99.3    59.3, 100.7    -22.0, 16.7    56.7, 98.7    -18.7, 18.7
        Mean (SD)                  76.2 (9.3)     75.8 (9.1)    -0.4 (7.3)     75.4 (8.7)    -0.3 (7.2) 
        Median                        76.0           75.8           0.0           75.3          -0.3    
  Xanomeline High Dose                                                                                  
    n                                  51             51            51             47            47     
    Min, Max                      53.0, 102.0     52.7, 92.3    -33.3, 16.3    53.0, 96.0    -24.7, 15.0
        Mean (SD)                 78.5 (10.2)     76.7 (9.0)    -1.7 (9.5)     76.2 (9.6)    -2.7 (9.3) 
        Median                        78.7           75.7          -1.3           77.3          -2.7    
  Placebo                                                                                               
    n                                  63             62            61             55            54     
    Min, Max                      47.3, 103.3     53.3, 96.7    -20.7, 21.7   51.3, 103.3    -20.0, 16.0
        Mean (SD)                 76.0 (10.8)     74.3 (9.4)    -1.6 (7.9)     74.4 (9.5)    -2.1 (8.1) 
        Median                        76.0           74.0          -1.3           74.7          -2.0    
————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Baseline displays Observed values only; post-baseline visits display both Observed and Change from Baseline (CFB).
Each treatment group header summarizes n and Min, Max via a multi-row group summary.
```

---

## Shell T32: Subgroup TEAEs with Mixed-RD Multi-Row `cfun` + Heterogeneous Sibling Subsections (`at_sibling`) + `denom_by`

### Structural Notes (from shell spec)
- Column structure: `grouped_cols_w_diffs()` with spanning header `"Active Study Agent"` over active arms vs `"Placebo"` and `"Risk Difference (%) (95% CI)"` comparison columns (Wald method).
- Outer row split: `SEX_LBL` (`Female`, `Male`) with `child_labels = "visible"`.
- Attached to `SEX_LBL` is a **2-row group summary (`cfun_sex_dual`)** where the two rows have **different Risk Difference column behavior**:
  1. `Subjects in subgroup, n` — displays the sex-specific `ADSL` denominator count `xx` (`count_unique` from `alt_counts_df`) in treatment columns and is **blank** in the Risk Difference columns (`riskdiff = FALSE`).
  2. `Subjects with >=1 TEAE, n (%)` — displays `xx/xx (xx.x%)` (`count_unique_denom_fraction` with `denom = "n_altdf"`, `denom_by = "SEX_LBL"`) in treatment columns and **Wald Risk Difference** `xx.x (xx.x, xx.x)` in the Risk Difference columns (`riskdiff = TRUE`, `method = "wald"`).
- Inside each `SEX_LBL` facet, two heterogeneous sibling analysis sections follow:
  1. `TEAEs by Maximum Severity` (`MAX_SEV`: `Mild`, `Moderate`, `Severe`; pre-computed per subject in data prep so `a_freq_j` with `riskdiff = TRUE` can be used instead of `a_maxlev`).
  2. `TEAEs Leading to Discontinuation by SOC` (`DISC_AEBODSYS`: `AEACN == "DRUG WITHDRAWN"`, `drop_levels = TRUE`, anchored via `at_sibling = "max_sev_sec"` so it stays inside `SEX_LBL`).

### The Trap
This shell combines **five interacting layout and `a_freq_j` traps**:
1. **Multi-row `cfun` with mixed `riskdiff = FALSE` (Row 1) and `riskdiff = TRUE` (Row 2) + `ref_path` forwarding**: Inside `cfun_sex_dual`, Row 1 calls `a_freq_j(..., denom = "n_altdf", denom_by = "SEX_LBL", .stats = "count_unique", riskdiff = FALSE)` on `.alt_df` (or `SUBGRP_N` derived on `adsl` and `adae`) so the RD columns are blank, while Row 2 calls `a_freq_j(..., denom = "n_altdf", denom_by = "SEX_LBL", .stats = "count_unique_denom_fraction", riskdiff = TRUE, method = "wald", ref_path = ref_path)`. Notice that `grouped_cols_w_diffs` injects `ref_path` via `extra_args` only when `extra_args` is attached or inherited; if `cfun_sex_dual` does not accept `ref_path = NULL` (or fall back to `c("colspan_trt", " ", "TRT01A", ctrl_grp)`), `a_freq_j(..., riskdiff = TRUE)` inside a custom wrapper fails!
2. **`child_labels = "visible"` required on `split_rows_by("SEX_LBL")`**: Because `cfun_sex_dual` returns 2 rows (`Subjects in subgroup, n` and `Subjects with >=1 TEAE, n (%)`), omitting `child_labels = "visible"` hides `"Female"` and `"Male"`.
3. **`a_maxlev` incompatibility with Risk Difference columns**: `a_maxlev` does not support `riskdiff = TRUE` columns (`select_afuns.md` Section 4). The programmer must pre-compute each subject's worst severity (`MAX_SEV` on their worst-severity TEAE row, `NA` on other rows) in data prep and analyze it with a non-`NA`-filtering `a_freq_j` wrapper (`riskdiff = TRUE`, `denom = "n_altdf"`, `denom_by = "SEX_LBL"`).
4. **Sibling section anchoring inside `SEX_LBL` (`at_sibling = "max_sev_sec"`, NOT `nested = FALSE`)**: The second subsection (`TEAEs Leading to Discontinuation by SOC`) must appear inside each `SEX_LBL` facet immediately after `TEAEs by Maximum Severity`. Using `nested = FALSE` escapes `SEX_LBL` to the table root; `at_sibling = "max_sev_sec"` keeps it inside `SEX_LBL`.
5. **Subgroup-specific denominators (`denom = "n_altdf"`, `denom_by = "SEX_LBL"`)**: Every percentage (`33/41`, `23/24`, `31/41` for `Female`; `23/26`, `25/27`, `18/23` for `Male`) uses the sex-by-treatment subgroup size from `ADSL`, not the full column `N` (`67`, `51`, `64`).

### Reasoning Paths Exercised
- **Custom Afuns (`custom_afuns.md` Section 3 — Tier 3 Stacking with Mixed RD Behavior)**: Stacking two `a_freq_j` calls inside a `cfun` where Row 1 suppresses RD (`riskdiff = FALSE`) and Row 2 computes Wald RD (`riskdiff = TRUE`, forwarding `ref_path`).
- **Junco `a_freq_j` Usage (`junco_a_freq_j_usage.md` — Subgroup Denominators)**: Combining `denom = "n_altdf"` with `denom_by = "SEX_LBL"` and `.stats = "count_unique_denom_fraction"` across both the `cfun` and the inner sibling `analyze()` blocks.
- **Row Structure (`row_struct_layout.md` Section 3)**: Using `at_sibling = "max_sev_sec"` inside an outer subgroup split (`SEX_LBL`) to stack two heterogeneous `analyze()` blocks without escaping the subgroup split.

### Hypothesized Weaknesses Probed
- **Populating RD columns on Row 1 (`Subjects in subgroup, n`)**: Computing risk differences on the subgroup sample size row instead of leaving RD cells blank (`riskdiff = FALSE`).
- **Using `a_maxlev` with RD columns**: Triggering errors or blank RD columns in `TEAEs by Maximum Severity`.
- **Using `nested = FALSE` for `DISC_AEBODSYS`**: Escaping `SEX_LBL` and placing `TEAEs Leading to Discontinuation by SOC` at the bottom of the table outside `Female`/`Male`.
- **Using full-arm `N` (`67`, `51`, `64`) instead of `denom_by = "SEX_LBL"`**: Dividing by `N_col` instead of the sex-specific `ADSL` counts (`41`, `24`, `41` and `26`, `27`, `23`).

### Reference `rtables` Output (Exact)
```
TSFAE-SUBGRP-MIXED: Subgroup TEAEs with Dual-Row Subgroup Header, Severity, and Discontinuation SOCs; Safety Analysis Set

—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                                                                        Active Study Agent                                                 Risk Difference (%) (95% CI)                  
                                                                            Xanomeline Low Dose   Xanomeline High Dose      Placebo      Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Placebo
                                                                                   N=67                   N=51               N=64                     N=67                             N=51              
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Female                                                                                                                                                                                                   
  Subjects in subgroup, n                                                           41                     24                 41                                                                         
  Subjects with >=1 TEAE, n (%)                                                33/41 (80.5%)         23/24 (95.8%)       31/41 (75.6%)         4.9 (-13.0, 22.8)                 20.2 (4.8, 35.6)        
    TEAEs by Maximum Severity                                                                                                                                                                            
      Mild                                                                     9/41 (22.0%)           4/24 (16.7%)       9/41 (22.0%)          0.0 (-17.9, 17.9)                -5.3 (-24.9, 14.3)       
      Moderate                                                                 15/41 (36.6%)         11/24 (45.8%)       13/41 (31.7%)         4.9 (-15.6, 25.4)                14.1 (-10.4, 38.6)       
      Severe                                                                   9/41 (22.0%)           8/24 (33.3%)       9/41 (22.0%)          0.0 (-17.9, 17.9)                11.4 (-11.3, 34.1)       
    TEAEs Leading to Discontinuation by SOC                                                                                                                                                              
      Cardiac disorders                                                         2/41 (4.9%)               0/24               0/41               4.9 (-1.7, 11.5)                  0.0 (0.0, 0.0)         
      Gastrointestinal disorders                                                2/41 (4.9%)           1/24 (4.2%)         3/41 (7.3%)          -2.4 (-12.8, 7.9)                 -3.2 (-14.4, 8.1)       
      General disorders and administration site conditions                      2/41 (4.9%)           6/24 (25.0%)        2/41 (4.9%)           0.0 (-9.3, 9.3)                  20.1 (1.6, 38.7)        
      Infections and infestations                                               2/41 (4.9%)               0/24            2/41 (4.9%)           0.0 (-9.3, 9.3)                  -4.9 (-11.5, 1.7)       
      Investigations                                                            1/41 (2.4%)               0/24               0/41               2.4 (-2.3, 7.2)                   0.0 (0.0, 0.0)         
      Musculoskeletal and connective tissue disorders                           1/41 (2.4%)               0/24               0/41               2.4 (-2.3, 7.2)                   0.0 (0.0, 0.0)         
      Neoplasms benign, malignant and unspecified (incl cysts and polyps)       1/41 (2.4%)               0/24               0/41               2.4 (-2.3, 7.2)                   0.0 (0.0, 0.0)         
      Nervous system disorders                                                  2/41 (4.9%)           3/24 (12.5%)           0/41               4.9 (-1.7, 11.5)                 12.5 (-0.7, 25.7)       
      Psychiatric disorders                                                        0/41                   0/24            1/41 (2.4%)           -2.4 (-7.2, 2.3)                 -2.4 (-7.2, 2.3)        
      Respiratory, thoracic and mediastinal disorders                           1/41 (2.4%)               0/24               0/41               2.4 (-2.3, 7.2)                   0.0 (0.0, 0.0)         
      Skin and subcutaneous tissue disorders                                    4/41 (9.8%)           4/24 (16.7%)        3/41 (7.3%)           2.4 (-9.6, 14.5)                 9.3 (-7.6, 26.3)        
      Surgical and medical procedures                                              0/41                   0/24            1/41 (2.4%)           -2.4 (-7.2, 2.3)                 -2.4 (-7.2, 2.3)        
      Vascular disorders                                                           0/41                   0/24            1/41 (2.4%)           -2.4 (-7.2, 2.3)                 -2.4 (-7.2, 2.3)        
Male                                                                                                                                                                                                     
  Subjects in subgroup, n                                                           26                     27                 23                                                                         
  Subjects with >=1 TEAE, n (%)                                                23/26 (88.5%)         25/27 (92.6%)       18/23 (78.3%)         10.2 (-10.7, 31.1)                14.3 (-5.2, 33.9)       
    TEAEs by Maximum Severity                                                                                                                                                                            
      Mild                                                                     8/26 (30.8%)           6/27 (22.2%)       11/23 (47.8%)        -17.1 (-44.1, 10.0)               -25.6 (-51.3, 0.1)       
      Moderate                                                                 11/26 (42.3%)         13/27 (48.1%)       4/23 (17.4%)           24.9 (0.4, 49.4)                 30.8 (6.4, 55.2)        
      Severe                                                                   4/26 (15.4%)           6/27 (22.2%)       3/23 (13.0%)          2.3 (-17.2, 21.9)                 9.2 (-11.7, 30.0)       
    TEAEs Leading to Discontinuation by SOC                                                                                                                                                              
      Cardiac disorders                                                            0/26                   0/27            1/23 (4.3%)          -4.3 (-12.7, 4.0)                 -4.3 (-12.7, 4.0)       
      Eye disorders                                                             1/26 (3.8%)               0/27               0/23               3.8 (-3.5, 11.2)                  0.0 (0.0, 0.0)         
      Gastrointestinal disorders                                                2/26 (7.7%)           2/27 (7.4%)         2/23 (8.7%)          -1.0 (-16.4, 14.4)               -1.3 (-16.5, 13.9)       
      General disorders and administration site conditions                     4/26 (15.4%)           2/27 (7.4%)        4/23 (17.4%)          -2.0 (-22.8, 18.8)               -10.0 (-28.4, 8.4)       
      Infections and infestations                                                  0/26               1/27 (3.7%)         1/23 (4.3%)          -4.3 (-12.7, 4.0)                -0.6 (-11.6, 10.3)       
      Injury, poisoning and procedural complications                               0/26               2/27 (7.4%)            0/23                0.0 (0.0, 0.0)                  7.4 (-2.5, 17.3)        
      Investigations                                                            1/26 (3.8%)               0/27            1/23 (4.3%)          -0.5 (-11.6, 10.6)                -4.3 (-12.7, 4.0)       
      Nervous system disorders                                                     0/26               4/27 (14.8%)           0/23                0.0 (0.0, 0.0)                  14.8 (1.4, 28.2)        
      Psychiatric disorders                                                        0/26               2/27 (7.4%)            0/23                0.0 (0.0, 0.0)                  7.4 (-2.5, 17.3)        
      Reproductive system and breast disorders                                     0/26                   0/27            1/23 (4.3%)          -4.3 (-12.7, 4.0)                 -4.3 (-12.7, 4.0)       
      Respiratory, thoracic and mediastinal disorders                              0/26               1/27 (3.7%)            0/23                0.0 (0.0, 0.0)                  3.7 (-3.4, 10.8)        
      Skin and subcutaneous tissue disorders                                   3/26 (11.5%)           4/27 (14.8%)        1/23 (4.3%)           7.2 (-7.7, 22.0)                 10.5 (-5.3, 26.2)       
      Vascular disorders                                                           0/26                   0/27            1/23 (4.3%)          -4.3 (-12.7, 4.0)                 -4.3 (-12.7, 4.0)       
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Subgroup header displays sex-specific N (blank in Risk Difference columns) and overall TEAE incidence.
Severity and Discontinuation SOC sections use sex-specific denominators within each treatment group.
```

---

## Updated Coverage Matrix (T21–T32)

| Shell | Col Struct | Row Struct | Select Afuns | Custom Afuns | Sort/Prune | Key Reasoning Traps |
|-------|-----------|-----------|-------------|-------------|-----------|-------------------|
| T21   | Simple    | Stacked   | a_freq_j (dual .stats) | Tier 3 stacking | — | Subject count + event count in same section; vertical stacking |
| T22   | Simple    | 3 dissimilar sections | a_freq_j + a_maxlev + a_freq_j | — | Yes | Dissimilar sections forced into single split; wrong afun per section |
| T23   | Simple    | Stacked flags | a_freq_j × 6 | — | — | Factor fabrication trap; overlapping membership; flag derivation |
| T24   | Simple    | at_sibling multi-section | a_summarize_ancova_j + a_summary | — | — | nested=FALSE escape; at_sibling requirement; mixed afuns |
| T25   | Simple    | Stacked flags | a_freq_j | — | — | Standalone marginal summary anti-pattern; no-op analyze trap |
| T26   | Spanning+RD | SOC→PT | a_freq_j | — | — | Contract 1 after filtering; stale factor levels; derived var scope |
| T27   | Simple    | SOC→PT→Severity | a_maxlev | — | — | drop_split_levels inverted pitfall; ragged structure intentional |
| T28   | Simple + 3-line topleft | SOC (`child_labels = "visible"`) → 2-row `cfun` → PT (`indent_mod = 1L`) | a_freq_j | 2-row `cfun` (`count_unique_fraction` + `count`) | Custom `scorefun` on row 1 of 2-row `@content` + `jj_complex_scorefun` | `child_labels = "default"` hides SOC label when `cfun` returns 2 stat rows; `indent_mod = 1L` on leaf PTs; custom `scorefun` for multi-row `@content` |
| T29   | Simple    | Inside-out 2-depth `at_sibling` (`PARAM_LBL` → `AVISIT` + `at_sibling = "visit_overall"` + `at_sibling = "AVISIT"`) | a_summarize_ex_j + a_freq_j | Non-NA wrapper for worst shift | — | Two `at_sibling` hops at different depths (`visit_overall` at depth 2, `AVISIT` at depth 1); `rtables` auto-flips `AVISIT` `label_pos` to `"visible"` unless `label_pos = "hidden"` is explicit |
| T30   | 4 Parallel Non-Nested Banners (`add_overall_col` + `split_cols_by`) | `ANY_TEAE` + SOC→PT | a_freq_j | — | Sort by `All Subjects` (`col_paths(tbl)[[1]]`) | `split_cols_by(..., nested = FALSE)` omits spanning headers; requires `add_overall_col("Banner") |> split_cols_by("VAR")` without row-binding data |
| T31   | `AVISIT` → `split_cols_by_multivar(c("AVAL", "CHG"))` with `make_split_fun` + `restrict_facets` pruning `CFB` at `Baseline` | `PARAM` → `TRT01A` (`child_labels = "visible"`) → 2-row `cfun` → `analyze_colvars(indent_mod = 1L)` | — | 2-row `cfun` + 2-row `afun` reading `.spl_context$cur_col_id` | — | `drop_split_levels` fails on `split_cols_by_multivar`; `cfun` receives `.var = "TRT01A"` and must inspect `.spl_context$cur_col_id` for `"AVAL"` vs `"CHG"` |
| T32   | `grouped_cols_w_diffs` (Spanning + Wald RD) | `SEX_LBL` (`child_labels = "visible"`) → 2-row mixed-RD `cfun` → 2 sibling sections (`at_sibling = "max_sev_sec"`) | a_freq_j (`denom_by = "SEX_LBL"`) | 2-row `cfun` (`riskdiff = FALSE` on row 1, `riskdiff = TRUE` on row 2) + pre-computed `MAX_SEV` | — | Mixed RD behavior across rows of a single `cfun`; `a_maxlev` fails with RD columns; `at_sibling` inside `SEX_LBL`; `denom_by = "SEX_LBL"` |

### Adversarial Design Principles

These shells target the following categories of reasoning failure:

1. **Anti-Pattern Attraction** (T22, T23, T25, T30): The shell's visual structure strongly suggests a prohibited pattern (single split for dissimilar sections, factor fabrication for pre-specified rows, `summarize_row_groups` without `analyze`, or duplicating/row-binding `ADSL` and `ADAE` to fake parallel column banners). The model must resist the visual suggestion and apply the structural rules.

2. **Counting Unit & Multi-Row Content Table Interactions** (T21, T27, T28, T31, T32): The shell requires two different counting mechanisms or a multi-row-generating `cfun` (`summarize_row_groups`). When a `cfun` emits 2+ labeled rows instead of a single facet-named row, `child_labels = "default"` hides the parent facet header (`T28`, `T31`, `T32`), leaf rows require `indent_mod = 1L` (`T28`, `T31`), sorting on the parent split requires extracting row 1 of the 2-row `@content` table (`T28`), and under `split_cols_by_multivar()` the `cfun` must inspect `.spl_context$cur_col_id` because `.var` is the row-split variable (`T31`).

3. **Non-Linear Tree Navigation & Multi-Depth `at_sibling`** (T24, T29, T32): The shell requires multi-section structure inside an outer split across one or two tree depths (`at_sibling = "visit_overall"` at depth 2 followed by `at_sibling = "AVISIT"` at depth 1 in `T29`; `at_sibling = "max_sev_sec"` inside `SEX_LBL` in `T32`), while avoiding `rtables`'s implicit flip of the targeted split's `label_pos` to `"visible"` by setting `label_pos = "hidden"` explicitly (`T29`).

4. **Data Prep Completeness** (T23, T26, T29, T32): The shell requires derived variables that must exist in data prep, not inside afuns (`WORST_ANRIND` across all post-baseline visits in `T29`; subject-level `MAX_SEV` to work around `a_maxlev`'s RD limitation in `T32`). The model must trace the data flow through filtering, joining, and factor level synchronization.

5. **Over-Correction from Warnings & Asymmetric Trees** (T27, T31): The model must distinguish when defaults or warnings invert (`drop_split_levels` on ragged inner splits in `T27`; `make_split_fun` + `restrict_facets` instead of `drop_split_levels` to prune `CFB` under `Baseline` in `split_cols_by_multivar` in `T31`).
