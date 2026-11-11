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

## Updated Coverage Matrix (T21–T27)

| Shell | Col Struct | Row Struct | Select Afuns | Custom Afuns | Sort/Prune | Key Reasoning Traps |
|-------|-----------|-----------|-------------|-------------|-----------|-------------------|
| T21   | Simple    | Stacked   | a_freq_j (dual .stats) | Tier 3 stacking | — | Subject count + event count in same section; vertical stacking |
| T22   | Simple    | 3 dissimilar sections | a_freq_j + a_maxlev + a_freq_j | — | Yes | Dissimilar sections forced into single split; wrong afun per section |
| T23   | Simple    | Stacked flags | a_freq_j × 6 | — | — | Factor fabrication trap; overlapping membership; flag derivation |
| T24   | Simple    | at_sibling multi-section | a_summarize_ancova_j + a_summary | — | — | nested=FALSE escape; at_sibling requirement; mixed afuns |
| T25   | Simple    | Stacked flags | a_freq_j | — | — | Standalone marginal summary anti-pattern; no-op analyze trap |
| T26   | Spanning+RD | SOC→PT | a_freq_j | — | — | Contract 1 after filtering; stale factor levels; derived var scope |
| T27   | Simple    | SOC→PT→Severity | a_maxlev | — | — | drop_split_levels inverted pitfall; ragged structure intentional |

### Adversarial Design Principles

These shells target the following categories of reasoning failure:

1. **Anti-Pattern Attraction** (T22, T23, T25): The shell's visual structure strongly suggests a prohibited pattern (single split for dissimilar sections, factor fabrication for pre-specified rows, summarize_row_groups without analyze). The model must resist the visual suggestion and apply the structural rules.

2. **Counting Unit Confusion** (T21, T27): The shell requires two different counting mechanisms in the same table or section. The model must recognize that a single afun call cannot produce both.

3. **Nesting Escape** (T24): The shell requires multi-section structure inside an outer split. The model must use `at_sibling` instead of relying on default `nested = FALSE` behavior.

4. **Data Prep Completeness** (T23, T26): The shell requires derived variables that must exist in data prep, not inside afuns. The model must trace the data flow through filtering, joining, and factor level synchronization.

5. **Over-Correction from Warnings** (T27): The model has been warned about a pitfall, and the shell presents a case where the "pitfall" behavior is actually correct. The model must distinguish the two cases rather than blindly applying the warning.
