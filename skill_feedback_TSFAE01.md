# Skill Weakness Analysis: Generalizable Issues from TSFAE01 Session

Based on analysis of the TSFAE01 (CTC-V2) chat log against `rtables_junco_table_prog.md` and its subskills.

---

## 1. `a_maxlev` Does Not Support Difference Columns; Workarounds and Catalog Metadata Needed

**Problem:** `a_maxlev` silently fails in risk-difference column layouts (`grouped_cols_w_diffs`). The chat log shows it threw `"'subset' must be logical"` inside diff columns because it doesn't support `riskdiff`/`ref_path`. This is a general limitation — any table combining worst-grade-per-subject counting with comparative columns will hit this.

**Where it matters:** `select_afuns.md` Section 3 (Quick Selection Decision Matrix) lists `a_maxlev` without any indication that it doesn't work in difference column layouts. The function's entry in Section 4 (Master Catalog) also omits this. Section 5.1 item 5 (the detailed `a_maxlev` guide) gives no caveat. A user following the selection workflow would pick `a_maxlev` for worst-grade tables, then discover it breaks when diff columns are present. More broadly, the selection workflow has no systematic way to determine which afuns support difference columns and which don't — the user must discover this by trial and error.

**Suggested improvement:**

### A. Add a "Supports Diff Columns" indicator to the afun lookup surfaces

Add a **Diff Column Compatible** column (or similarly named, e.g., "Supports Difference Columns") to the Quick Selection Decision Matrix (Section 3) and the Master Catalog (Section 4). This column should contain a simple Yes/No indicator for each function. At minimum:
- `a_freq_j`: Yes
- `a_freq_j_with_exclude`: Yes
- `a_freq_combos_j`: Yes
- `a_freq_subcol_j`: Yes
- `a_freq_resp_var_j`: Yes
- `a_maxlev`: **No**
- `a_two_tier` / `a_three_tier`: Depends on `grp_fun`/`detail_fun`
- `a_eair100_j`: Yes
- `a_eair_strat_j`: Yes
- `a_patyrs_j`: Yes (blanks diff cols)
- `a_summarize_ex_j`: Yes (via `comp_btw_group`)
- `tern::a_summary`: No (no `riskdiff`/`ref_path` support)

This information should also be surfaced in:
- The detailed `a_maxlev` guide (Section 5.1 item 5) as a caveat.
- Diagnostic Checklist item #3 (Column Conditionality Match) — add `a_maxlev` and `tern::a_summary` as specific examples of functions that fail this check.
- Any other location where afun selection or recommendation occurs.

### B. Document two workaround patterns for worst-grade-per-subject in diff column layouts

**Workaround 1 (Pre-computed factor + `a_freq_j`):** Pre-compute the worst grade as a factor variable in data prep and use `a_freq_j` on the factor levels instead of `a_maxlev`. This is the simpler approach and works seamlessly with `riskdiff = TRUE`:
```r
# Data prep: compute worst grade per subject, then join back
worst <- adae |>
  group_by(USUBJID) |>
  summarize(WORST_GRADE = max(as.integer(AETOXGR), na.rm = TRUE)) |>
  mutate(WORST_GRADE_F = factor(WORST_GRADE, levels = seq_along(levels(adae$AETOXGR)),
                                 labels = levels(adae$AETOXGR)))
adsl <- left_join(adsl, worst, by = "USUBJID")
adsl$WORST_GRADE_F[is.na(adsl$WORST_GRADE_F)] <- levels(adsl$WORST_GRADE_F)[1]  # or "Missing"
# Then use a_freq_j on WORST_GRADE_F in the layout
```

**Workaround 2 (Column-dispatch wrapper around `a_maxlev`):** When `a_maxlev`'s exact counting semantics are preferred (or when the pre-computed factor approach doesn't fit), wrap `a_maxlev` in a column-conditional dispatch afun (Pattern 2A from `custom_afuns.md`) that calls `a_maxlev` in treatment columns and emits blank cells in difference columns:
```r
make_maxlev_diffsafe <- function(ref_path) {
  function(df, .var, .N_col, .spl_context, .alt_df_full, ...) {
    if (any(grepl("Risk Difference", .spl_context$cur_col_split_val[[1]], fixed = TRUE))) {
      # Emit blank cells matching a_maxlev's row count
      lvls <- levels(df[[.var]])
      in_rows(.list = setNames(replicate(length(lvls), list(NULL)), lvls))
    } else {
      a_maxlev(df, .var = .var, .N_col = .N_col,
               .spl_context = .spl_context, .alt_df_full = .alt_df_full, ...)
    }
  }
}
```
This template generalizes to any afun that doesn't support difference columns: call the real afun in treatment columns, emit the matching number of blank `rcell(NULL)` rows in difference columns.

### C. Add the caveat to the `a_maxlev` entry

In the Master Catalog (Section 4) and the detailed guide (Section 5.1 item 5), add a caveat stating that `a_maxlev` does not support `riskdiff`/`ref_path` and cannot be used directly in layouts with difference columns (`grouped_cols_w_diffs` or manual risk-difference column branches). Reference the two workaround patterns above.

### D. Add a blanket caveat that no `tern` function supports junco-style difference columns

`a_maxlev` is not the only function that lacks difference column support — **no `tern` function** (`a_summary`, `a_surv_time`, `coxph_pairwise`, etc.) supports `riskdiff`/`ref_path` or junco-style column conditionality out of the box. This is an architectural distinction: `junco` functions were designed with `grouped_cols_w_diffs()` column topologies in mind; `tern` functions were not. Any `tern` function used in a layout with difference columns will need a column-dispatch wrapper (Pattern 2A from `custom_afuns.md`) or a Tier 3 bridge via `get_ref_info()` to handle the difference columns.

This caveat should be added in three places in `select_afuns.md`:

1. **Section 1 (Architectural Paradigm)**, item 2 ("Direct `afun`/`cfun` Paradigm (`junco`) vs. Layout Wrappers (`tern`)") — add a note stating that `tern` functions do not support junco-style difference column layouts (`riskdiff`/`ref_path`). When a table requires difference columns and the statistical calculation is only available in `tern`, escalate to `custom_afuns.md` to wrap the `tern` function in a column-dispatch or `get_ref_info()` bridge pattern.

2. **Section 2.4A (Column Conditionality, Selection Rule)** — the existing rule says "Never use a column-agnostic `afun` in a heterogeneous column layout" and lists junco functions for each column topology. Add an explicit statement that all `tern` functions are column-agnostic by default and require wrapping (via Pattern 2A column dispatch or Tier 3 bridging from `custom_afuns.md`) before they can be used in layouts with difference columns.

3. **Diagnostic Checklist item #3 (Column Conditionality Match)** — add `tern` functions as a class that fails this check, alongside the specific `a_maxlev` example. State: "All `tern` functions and `junco::a_maxlev` lack built-in difference column support. If the layout includes difference columns, either select a `junco` function that supports `riskdiff`/`ref_path`, or wrap the selected function using the column-dispatch or `get_ref_info()` bridge patterns from `custom_afuns.md`."

The **Diff Column Compatible** column proposed in Section A above should mark all `tern` functions as "No" to make this immediately visible during lookup.

---

## 2. `ref_path` Must Never Be Passed in `extra_args` When Column Structure Auto-Injects It

**Problem:** `grouped_cols_w_diffs()` auto-injects `ref_path` into the `extra_args` of comparison columns. When a custom afun also receives `ref_path` via `extra_args` in the `analyze()` call, the comparison columns get `ref_path` twice, causing `"formal argument ref_path matched by multiple actual arguments"`. This affects any table using `grouped_cols_w_diffs()` (or any future column structure function that auto-injects `ref_path`) with any afun — built-in or custom.

**Where it matters:** The skill files are **internally contradictory** on this point:
- `select_afuns.md` Section 4 (line ~441) correctly states: "do not pass it via `extra_args` to your `afun`/`cfun`" when using `grouped_cols_w_diffs()`.
- `junco_a_freq_j_usage.md` (lines ~104, ~461) correctly warns: "do NOT also pass it explicitly or you get a 'matched by multiple actual arguments' error."
- **But** `rtables_junco_table_prog.md` Contract 2 (line ~228) says: "passing `ref_path = ref_path` in `extra_args` is always safe" — this is **wrong**. Even for built-in junco afuns that happen to handle deduplication internally, relying on that deduplication is fragile and the double-specification is an anti-pattern.
- `rtables_junco_table_prog.md` Contract 2 (line ~229) tells users to "pass `ref_path` via `extra_args = list(ref_path = ref_path)`" for custom/bridged afuns — correct for manual `split_cols_by()` layouts, but wrong when `grouped_cols_w_diffs()` is used.
- `custom_afuns.md` Tier 3 (Section 4) shows `ref_path` as a formal argument of the bridged afun but doesn't explain how it arrives (auto-injection vs `extra_args` vs closure capture), leaving users to guess.

The root issue is that the guidance doesn't enforce a single strict rule: **when a column structure function auto-injects `ref_path`, it must never also be passed in `extra_args`**. Instead, the afun must receive `ref_path` solely through the auto-injection mechanism. For custom afuns that need `ref_path` in treatment columns (where auto-injection doesn't apply), the afun should capture `ref_path` from a closure scope rather than receiving it via `extra_args`.

**Suggested improvement:**
- **`rtables_junco_table_prog.md` Contract 2**: Remove the claim that "passing `ref_path = ref_path` in `extra_args` is always safe". Replace with a strict two-case rule:
  1. **When using `grouped_cols_w_diffs()` or any column structure function that auto-injects `ref_path`**: Never pass `ref_path` in `extra_args`. For built-in junco afuns, `ref_path` arrives automatically in comparison columns and is ignored in treatment columns. For custom afuns that need `ref_path` in treatment columns (e.g., to call `get_ref_info()` or `a_freq_j(..., riskdiff = TRUE)` internally), capture `ref_path` from a closure scope.
  2. **When building column layouts manually with `split_cols_by()`**: Pass `ref_path` explicitly in `extra_args = list(ref_path = ref_path)` on each `analyze()`/`summarize_row_groups()` call.
- **`rtables_junco_table_prog.md` Contract 2, Custom/Bridged afun escalation bullet**: Amend to state that the `extra_args` approach applies only to manual column layouts. When using `grouped_cols_w_diffs()`, use the closure-capture pattern instead.
- **`custom_afuns.md` Tier 3 (Section 4)**: Add a note explaining how `ref_path` reaches the bridged afun: via `extra_args` in manual layouts, or via closure capture under `grouped_cols_w_diffs()`. Show the closure-capture pattern explicitly:
  ```r
  make_bridged_afun <- function(ref_path) {
    function(x, .spl_context, .var, ...) {
      ref <- junco::get_ref_info(ref_path, .spl_context, .var)
      standard_diff_afun(x, .ref_group = ref$ref_group, .in_ref_col = ref$in_ref_col)
    }
  }
  ```
- **`col_struct_layout.md` Section 8.1**: Strengthen the auto-injection note to a strict prohibition: "Because `grouped_cols_w_diffs()` auto-injects `ref_path` into comparison column `extra_args`, `ref_path` must never be passed in `extra_args` on downstream `analyze()` or `summarize_row_groups()` calls. Doing so causes double-injection errors in comparison columns."
- **`col_struct_layout.md` Section 9 (Pitfalls)**: Add a new pitfall item for `ref_path` double-injection.
- **Verification Checklist** (`rtables_junco_table_prog.md` Section 7, item about `ref_path`): Amend the existing check "Does `ref_path` in `extra_args` match..." to first ask: "Is the column structure built with `grouped_cols_w_diffs()` or another auto-injecting function? If so, `ref_path` must NOT appear in `extra_args`."

---

## 3. Signature Union Rule Needs Promotion, Strictness, and Guidance on Similar-Named Special Arguments

**Problem:** Two distinct errors in the chat log stem from incorrect special argument handling in custom afun wrappers:
1. `"argument .df_row is missing, with no default"` — the custom afun didn't list `.df_row` in its formals, so `rtables` never populated it, but the inner `a_freq_j` call needed it.
2. `"unused argument (.alt_df = .alt_df)"` — the custom afun listed `.alt_df` in its signature (guessing it was needed), but `a_freq_j` actually expects `.alt_df_full`, not `.alt_df`. Including the wrong one and forwarding via `...` caused the error.

Both errors trace to the same root cause: the **Signature Union Rule** — the wrapper must list the exact union of all `rtables` special arguments needed by its inner functions — is not applied with sufficient rigor. The rule exists in `custom_afuns.md` Section 3.3 (Pattern 2B, Vertical Stacking) as a boxed note ("Mandatory a/cfun Argument Invariant"), but it has two weaknesses:
- **Placement**: It's buried in a subsection about vertical stacking and scoped to Section 3. The rule applies universally to ALL custom afun patterns (column dispatch, closure wrappers, Tier 3 bridging, Tier 4 bespoke), not just stacking. It doesn't appear in the Tier 4 template (Section 5.1) or the Verification Checklist (Section 8).
- **Strictness**: The rule says to include the union of needed arguments, but doesn't warn that the union must be *exact* — including a similarly-named but wrong argument (e.g., `.alt_df` instead of `.alt_df_full`, or `.ref_group` when the inner function expects `ref_path`) can cause "unused argument" errors when forwarded via `...`, or silently receive wrong data. The special arguments reference table (Section 5.2) lists all available arguments but doesn't flag the confusingly similar name pairs or guide which junco functions expect which.

**Where it matters:** This affects every custom afun pattern. The confusingly similar special argument names in `rtables` make it easy to guess wrong:
- `.alt_df` vs `.alt_df_full` vs `.alt_df_row` — three alt-counts variants with different scoping (column+row subset, full unsplit, row-only subset)
- `.df_row` vs `df` — full row-facet data vs column-sliced data
- `.ref_group` vs `.ref_full` — row-faceted reference data vs full reference data
- `.N_col` vs `.N_row` vs `.N_total` — different marginal counts with different data sources (`alt_counts_df` vs primary `df`)

In practice, users guess which argument an inner function needs rather than checking `formals()`, and the similar names make wrong guesses likely.

**Suggested improvement:**
- Promote the Signature Union Rule to a top-level invariant in `custom_afuns.md` Section 5 (Core Mechanics), alongside the existing template. State it as a universal rule that applies to all tiers, not just Pattern 2B.
- Strengthen the rule to emphasize *exactness*: the wrapper must include precisely the arguments its inner functions accept — no more, no less. Including an argument the inner function doesn't expect causes "unused argument" errors if forwarded via `...`; omitting one the inner function needs causes "missing argument" errors.
- Add a "Confusingly Similar Special Arguments" note to Section 5.2 (the special arguments reference table) that explicitly groups the similar-named arguments and notes which junco functions commonly expect which. At minimum: `a_freq_j` and most junco frequency functions use `.alt_df_full` (not `.alt_df`); `.df_row` is commonly needed by junco functions and must be forwarded.
- Add `.df_row` to the template function signature in Section 5.1, or at minimum add a note that `.df_row` is commonly needed by junco functions.
- In the Verification Checklist (Section 8), add two explicit checks:
  1. "Does the wrapper's `formals()` include every special argument that any inner function expects? Check via `names(formals(inner_fun))` against the special arguments reference in Section 5.2."
  2. "Does the wrapper include any special argument that no inner function accepts? If so, remove it — forwarding it via `...` will cause 'unused argument' errors."

---

## 4. Factor Level Synchronization After Joins Is Under-Specified

**Problem:** Two separate errors in the chat log stemmed from factor level mismatches: (1) `TRT01A` became character after filtering, causing `create_colspan_map` to fail; (2) `adae$TRT01A` factor levels weren't synchronized with `adsl$TRT01A` after `left_join`, causing `grouped_cols_w_diffs` to error with "All facets matched removal criteria."

**Where it matters:** Contract 1 in `rtables_junco_table_prog.md` says variables must exist on both `df` and `alt_counts_df` "with identical factor levels in the same order" and gives a "Standard Idiom" of `left_join`. But it doesn't warn that:
- Filtering `adsl` (e.g., removing "Screen Failure") can convert factors to character or leave stale levels.
- `left_join` / `inner_join` can desynchronize factor levels between the two datasets.
- `create_colspan_var` / `create_colspan_map` require factor input, not character.

**Suggested improvement:**
- Expand Contract 1's "Standard Idiom" to include explicit factor re-leveling after filtering and joining. Something like:
  ```r
  adsl <- adsl |> filter(...) |> mutate(TRT01A = factor(TRT01A))
  # After join:
  adae <- adae |> mutate(TRT01A = factor(TRT01A, levels = levels(adsl$TRT01A)))
  ```
- Add a pitfall to `col_struct_layout.md` Section 9 (Pitfalls) item #3 (Factor Level Hygiene) specifically about post-filter and post-join factor level repair.
- Consider adding a "Data Prep Checklist" subsection to Stage 1 of the orchestrator that explicitly calls out factor hygiene steps.

---

## 5. Archetype 1 Under-Specifies Multi-Row Sections; Label-Only Header Row Pattern Undocumented

**Problem:** The TSFAE01 table has 4 pre-specified sections of fixed rows with no data-driven faceting. Each section is a single `analyze()` call whose custom afun emits multiple named rows. This is structurally Archetype 1 ("No-Split Stacked Analyses"), but Archetype 1's description and canonical pipeline (`analyze("VAR1") |> analyze("VAR2")`) suggest simple one-variable-per-call usage where each `analyze()` emits one row (or a small number of rows from a single variable's standard afun). It doesn't address the common case where each `analyze()` call emits a *section* of multiple heterogeneous rows — with internal structure such as header rows, indented sub-rows, and rows computed from different variables — via a custom afun.

**Where it matters:** Many clinical tables have pre-specified row sections where each section is a block of rows with internal hierarchy (e.g., "Overall Summary of AEs" tables, exposure summary tables, baseline characteristics with mixed variable types). The existing skills cover the individual pieces — Archetype 1 for stacked `analyze()` calls, Pattern 2B in `custom_afuns.md` for vertical stacking, Pattern 2C for `indent_mod()` modification — but don't connect them into a coherent pattern for multi-row-section tables. A user following the archetype catalog would see Archetype 1 and think it only applies to simple cases, then struggle to find guidance for the multi-row-section variant.

Additionally, the pattern for emitting a **label-only header row** (a row with visible text in the stub but blank/empty cells) from within an afun — `in_rows("label" = rcell(NULL, format = "xx"))` — is not documented anywhere in the skill files. This is distinct from `show_labels = "visible"` on `analyze()` (which adds a variable-label header row above the analysis rows) and from `summarize_row_groups()` (which adds a content row at a facet boundary). The afun-emitted label-only row is needed when a section header must appear *within* a block of rows generated by a single `analyze()` call.

**Suggested improvement:**
- Expand Archetype 1's description in `row_struct_layout.md` Section 6 to explicitly note that each `analyze()` call can emit multiple rows with internal structure (headers, indented sub-rows, rows from different variables) via a custom afun. Add a brief note that this is the standard approach for tables with pre-specified row sections that don't correspond to data-driven facets, and cross-reference `custom_afuns.md` Pattern 2B (vertical stacking) and Pattern 2C (`indent_mod()` modification) as the mechanisms for constructing multi-row sections.
- Document the label-only header row pattern in `custom_afuns.md` as a building block (Section 7) or in the vertical stacking pattern (Section 3.3): to emit a row with a visible stub label but blank cells across all columns, use `in_rows("label_text" = rcell(NULL, format = "xx"))`. This can be stacked with other rows via `c()` to create section headers within an afun's output.

---

## 6. Pre-Computed Derived Variables Pattern Not Documented

**Problem:** The chat log documents that `a_freq_j` references `.df_row` internally, so any variable passed as `.var` must exist in `.df_row` (the full row-facet data across all columns). Variables created dynamically inside a custom afun via `mutate()` won't be in `.df_row`, causing silent failures. The workaround is to pre-compute all derived variables in data prep.

**Where it matters:** This is a general constraint that applies to **any derived variable** — not just flag/indicator variables — whenever a custom afun calls a junco or tern function that references `.df_row`, `.alt_df_full`, or any other data argument populated by `rtables` from the original dataset. Examples include:
- Flag/indicator variables (e.g., `SAEFL`, `RELFL`) derived via conditional logic
- Recoded or collapsed factor variables (e.g., collapsing granular categories into broader groups)
- Computed numeric variables (e.g., change-from-baseline, BMI, derived scores)
- Re-leveled or re-ordered factors (e.g., `WORST_GRADE_F` from the TSFAE01 session)
- Any column produced by `mutate()`, `case_when()`, `if_else()`, or similar transformations

The core invariant is: **`rtables` populates `.df_row` (and `.alt_df_full`) from the dataset passed to `build_table()` at the current row-facet level. It does not — and cannot — reflect transformations applied inside an afun to the `df` column-slice.** Any variable that an inner function will look up in `.df_row` must already exist in the data before `build_table()` is called.

This is not documented anywhere in the skills. The `custom_afuns.md` Tier 2 patterns show calling existing afuns but don't warn about the `.df_row` data scope limitation.

**Suggested improvement:**
- Add a warning to `custom_afuns.md` Section 3.3 (Pattern 2B) or as a new Building Block in Section 7: when calling `a_freq_j` (or any function that uses `.df_row`) from inside a custom afun, the `.var` column must exist in the original dataset passed to `build_table()`, not just in the `df` slice. Always pre-compute derived variables in data prep — this applies to flags, recoded factors, computed numerics, and any other transformation.
- State the invariant explicitly: **the `df` argument to an afun is a column-slice of the row-faceted data, while `.df_row` is the full row-faceted data across all columns. Transformations applied to `df` inside the afun are invisible to `.df_row`.** Any inner function that cross-references `.df_row` (as `a_freq_j` does for denominator calculation) will fail or silently produce wrong results if the variable only exists in the mutated `df`.
- Add this to the `a_freq_j` entry in `select_afuns.md` Section 4 as a caveat.
- Consider adding it to the expert usage skill `junco_a_freq_j_usage.md` gotchas list.
- In the Verification Checklist (`custom_afuns.md` Section 8), add: "Does the afun reference any variable (via `.var`, subsetting, or inner function calls) that doesn't exist in the dataset passed to `build_table()`? If so, move the derivation to data prep."

---

## Summary of Cross-Cutting Themes

| Theme | Affected Skills | Core Issue |
|---|---|---|
| **Function compatibility with difference columns** | `select_afuns.md`, `custom_afuns.md`, `rtables_junco_table_prog.md` | No systematic indicator of which afuns support diff columns; `a_maxlev` and all `tern` functions fail silently; need "Diff Column Compatible" metadata in lookup tables, blanket tern caveat, and workaround patterns (pre-computed factor, column-dispatch wrapper) |
| **`ref_path` double-specification in `extra_args` under auto-injecting column structures** | `rtables_junco_table_prog.md`, `custom_afuns.md`, `col_struct_layout.md` | Contract 2 claims `ref_path` in `extra_args` is "always safe" — contradicts correct guidance in `select_afuns.md` and `junco_a_freq_j_usage.md`; must be a strict prohibition when column structure auto-injects it |
| **Special argument forwarding in custom wrappers** | `custom_afuns.md` | Signature Union Rule is under-emphasized, missing from key locations, and doesn't warn about confusingly similar argument names (`.alt_df` vs `.alt_df_full`, etc.) |
| **Factor level hygiene in data prep** | `rtables_junco_table_prog.md`, `col_struct_layout.md` | Contract 1 states the requirement but doesn't give defensive coding patterns |
| **`.df_row` scope limitation for derived variables** | `custom_afuns.md`, `select_afuns.md` | Undocumented constraint: any variable referenced by inner functions via `.df_row` must exist in the `build_table()` dataset, not just in the afun's `df` slice |
| **Multi-row sections in stacked-analyze tables** | `row_struct_layout.md`, `custom_afuns.md` | Archetype 1 under-specifies multi-row sections; label-only header row pattern undocumented |
