# junco Adaptive Proportion Functions: `a_cond_proportion_j`, `a_proportion_diff_mf`, `a_test_proportion_diff_mf`

Expert usage reference for junco's adaptive proportion analysis functions that
automatically select statistical methods based on data characteristics.

## 1. Function Overview

| Function | Purpose | Method Selection Criterion |
|---|---|---|
| `a_cond_proportion_j` | Response proportion with CI | Exact (Clopper-Pearson) vs Wald, based on count thresholds |
| `a_proportion_diff_mf` | Difference in proportions with CI | CMH vs unconditional exact, based on Mantel-Fleiss criterion |
| `a_test_proportion_diff_mf` | Test of proportion difference (p-value) | CMH test vs Fisher's exact, based on Mantel-Fleiss criterion |

All three functions automatically add footnotes when the fallback (exact/Fisher)
method is used.

## 2. `a_cond_proportion_j` — Adaptive CI for Response Proportion

### Purpose

Estimates a response proportion with confidence interval, automatically choosing
between Clopper-Pearson (exact) and Wald methods based on observed counts.

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `df` | data.frame | — | Analysis data (or logical vector) |
| `.var` | string | — | Response variable name (logical or convertible to logical) |
| `conf_level` | numeric | `0.95` | Confidence level |
| `denom` | string | `"n"` | Denominator: `"n"` (cell count) or `"N_row"` (row count) |
| `num_limit` | integer | `0` | Numerator threshold for exact method |
| `denom_limit` | integer | `10` | Denominator threshold for exact method |
| `method_scope` | string | `"cell"` | `"cell"` = decide per cell; `"row"` = decide based on row-wide counts |
| `long` | logical | `FALSE` | If `TRUE`, label includes detailed method selection criteria |
| `na.rm` | logical | `FALSE` | Remove NAs before analysis. If `FALSE` and NAs present, errors |
| `.stats` | character | `NULL` | Statistics to select |
| `.formats` | named char/list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |

### Method Selection Logic

The **exact** (Clopper-Pearson) method is used when ANY of these are true:
1. Denominator < `denom_limit` (e.g., n < 10)
2. Number of responders ≤ `num_limit` (e.g., x = 0)
3. Number of responders ≥ denominator − `num_limit` (e.g., x = n)

Otherwise, the **Wald** method is used.

When `method_scope = "row"`, the decision is based on row-wide counts (across
all columns), ensuring the same method is used for all columns in a row.

### Available Statistics

| Statistic | Description | Default Format |
|---|---|---|
| `n_prop` | `(n_responders, proportion)` | `xx (xx.x%)` |
| `prop_ci` | CI for proportion (×100) | `(xx.x, xx.x)` |

### Label Behavior

The `prop_ci` label automatically reflects the method(s):
- `method_scope = "cell"`: `"95% CI (Wald / Clopper-Pearson)"` (both methods listed)
- `method_scope = "row"`: `"95% CI (Wald)"` or `"95% CI (Clopper-Pearson)"` (specific method)
- `long = TRUE`: includes the selection criteria, e.g.,
  `"95% CI for Response Rates (Wald if n >= 10, x > 0, x < n - 0; else Clopper-Pearson)"`

### Example

```r
lyt <- basic_table() |>
  split_cols_by(var = "grp") |>
  analyze(vars = "rsp", afun = a_cond_proportion_j,
    extra_args = list(conf_level = 0.90, num_limit = 0, denom_limit = 10))

build_table(lyt, df = dta)
```

## 3. `a_proportion_diff_mf` — Adaptive Proportion Difference

### Purpose

Estimates the difference in response proportions between treatment and reference
groups for stratified data, with the method selected based on the Mantel-Fleiss
(MF) criterion.

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `df` | data.frame | — | Non-reference group data |
| `.var` | string | — | Response variable |
| `.ref_group` | data.frame | — | Reference group data (auto-populated by rtables via `ref_group`) |
| `.in_ref_col` | logical | — | Whether in reference column (auto-populated) |
| `val` | scalar | `TRUE` | Value identifying response of interest. For character/factor `.var`, use e.g. `"Y"` |
| `variables` | list | `list(strata = NULL)` | Strata variables: `list(strata = c("STRAT1", "STRAT2"))`. All must be factors |
| `na.rm` | logical | `FALSE` | Remove incomplete rows before analysis |
| `conf_level` | numeric | `0.95` | Confidence level |
| `mf_method` | string | `"cmh"` | CMH method when MF criterion satisfied: `"cmh"`, `"cmh_sato"`, `"cmh_mn"` |
| `mf_threshold` | numeric or NULL | `NULL` | Custom MF threshold. `NULL` uses `tern::mantel_fleiss_crit()` default |
| `exact_footnote` | string or RefFootnote | `RefFootnote(...)` | Footnote when exact method is used |
| `.stats` | character | `NULL` | Statistics to select |
| `.formats` | named char/list | `NULL` | Override formats |
| `.labels` | named char | `NULL` | Override labels |

### Method Selection Logic

1. If **no strata** specified → warning + unconditional exact (always)
2. If strata specified and **MF criterion satisfied** → CMH method (per `mf_method`)
3. If strata specified and **MF criterion NOT satisfied** (or NA) → unconditional exact

The Mantel-Fleiss criterion is evaluated by `tern::mantel_fleiss_crit()` on the
stratified 2×2 tables.

### Available CMH Methods

| `mf_method` | Description | Variance Estimator |
|---|---|---|
| `"cmh"` | Standard CMH | Standard |
| `"cmh_sato"` | CMH with Sato variance | Sato |
| `"cmh_mn"` | CMH with Miettinen-Nurminen | Miettinen-Nurminen score |

### Available Statistics

| Statistic | Description |
|---|---|
| `diff` | Point estimate of difference (×100, as percentage) |
| `diff_ci` | CI for difference |
| `diff_est_ci` | Combined estimate + CI |

### Footnote Behavior

When the unconditional exact method is used (MF criterion not met), a footnote
is automatically attached to all cells. The default footnote text is
`"Unconditional exact"` with symbol `+`.

Custom footnotes can be passed:
```r
exact_footnote = "This was the exact method"
# or
exact_footnote = rtables::RefFootnote("Exact Inference", index = 1L, symbol = "+")
```

### Label Format

Labels always show both possible methods:
`"Difference in Response rate (%) (CMH method / Unconditional exact)"`

The specific CMH method name varies:
- `"cmh"` → `"CMH, without correction"`
- `"cmh_sato"` → `"CMH, Sato variance estimator"`
- `"cmh_mn"` → `"CMH, Miettinen and Nurminen"`

### Reference Column Behavior

In the reference column (`.in_ref_col = TRUE` or `NULL`), all statistics return
`numeric(0)` (blank cells). This is standard tern convention.

### Example

```r
lyt <- basic_table() |>
  split_cols_by(var = "grp", ref_group = "Placebo") |>
  analyze(vars = "rsp", afun = a_proportion_diff_mf,
    extra_args = list(
      val = "Y",
      variables = list(strata = c("f1", "f2")),
      conf_level = 0.90,
      mf_method = "cmh_mn",
      .stats = "diff_est_ci"
    ))

build_table(lyt, df = dta)
```

## 4. `a_test_proportion_diff_mf` — Adaptive Proportion Difference Test

### Purpose

Tests the difference in response proportions, selecting between CMH test and
Fisher's exact test based on the Mantel-Fleiss criterion.

### Parameters

Same as `a_proportion_diff_mf` with these differences:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `alternative` | string | `"two.sided"` | Alternative hypothesis: `"two.sided"`, `"less"`, `"greater"` |
| `mf_method` | string | `"cmh"` | CMH test method: `"cmh"`, `"cmh_sato"`, `"cmh_wh"` (note: `"cmh_wh"` replaces `"cmh_mn"`) |

### Method Selection Logic

Same as `a_proportion_diff_mf`:
1. No strata → warning + Fisher's exact
2. MF satisfied → CMH test (per `mf_method`)
3. MF not satisfied → Fisher's exact

### Available CMH Test Methods

| `mf_method` | Description |
|---|---|
| `"cmh"` | Standard CMH test |
| `"cmh_sato"` | CMH with Sato variance |
| `"cmh_wh"` | CMH with Wilson-Hilferty transformation |

Note: The test function uses `"cmh_wh"` while the estimate function uses `"cmh_mn"`.

### Available Statistics

| Statistic | Description |
|---|---|
| `pval` | p-value from the selected test |

### Label Format

`"(Fisher's Exact Test / Cochran-Mantel-Haenszel Test)"` (or variant based on
`mf_method` and `alternative`).

### Example

```r
lyt <- basic_table() |>
  split_cols_by(var = "grp", ref_group = "Placebo") |>
  analyze(vars = "rsp", afun = a_test_proportion_diff_mf,
    extra_args = list(
      val = "Y",
      variables = list(strata = c("f1", "f2"))
    ))

build_table(lyt, df = dta)
```

## 5. Using All Three Together

A common pattern combines `a_cond_proportion_j` for per-arm response rates with
`a_proportion_diff_mf` for between-arm differences and `a_test_proportion_diff_mf`
for the p-value:

```r
lyt <- basic_table() |>
  split_cols_by("ARM", ref_group = "Placebo") |>
  # Response rate with adaptive CI
  analyze("RSP", afun = a_cond_proportion_j,
    extra_args = list(num_limit = 0, denom_limit = 10),
    show_labels = "hidden") |>
  # Difference in proportions with adaptive method
  analyze("RSP", afun = a_proportion_diff_mf,
    extra_args = list(
      variables = list(strata = c("STRAT1")),
      mf_method = "cmh_mn",
      .stats = "diff_est_ci"
    ),
    show_labels = "hidden") |>
  # P-value with adaptive test
  analyze("RSP", afun = a_test_proportion_diff_mf,
    extra_args = list(
      variables = list(strata = c("STRAT1"))
    ),
    show_labels = "hidden")
```

## 6. Column Structure

These functions use the standard tern `ref_group` pattern (NOT the junco
`ref_path` / `grouped_cols_w_diffs` pattern):

```r
split_cols_by("ARM", ref_group = "Placebo")
```

The reference group is identified via `.in_ref_col` and `.ref_group` parameters
that rtables auto-populates from the `ref_group` argument.

## 7. Gotchas and Pitfalls

### 7.1 Strata Variables Must Be Factors

All variables listed in `variables = list(strata = ...)` must be factors.
Non-factor strata will cause errors in `tern::h_prepare_rsp_table()`.

### 7.2 No Strata = Warning + Exact Fallback

If `variables = list(strata = NULL)` (or strata not specified), both
`a_proportion_diff_mf` and `a_test_proportion_diff_mf` emit a warning and
fall back to the exact/Fisher method. This is by design — the MF criterion
requires strata.

### 7.3 `val` Must Match `.var` Type

If `.var` is a character/factor column, `val` must be a character string (e.g.,
`val = "Y"`). If `.var` is logical, `val` defaults to `TRUE`. Mismatched types
cause an error.

### 7.4 `na.rm = FALSE` (Default) Errors on NAs

By default, any NA values in `.var` or strata columns cause an error. Set
`na.rm = TRUE` to remove incomplete rows (with a warning about how many rows
were removed).

### 7.5 Difference is Expressed as Percentage

The `diff`, `diff_ci`, and `diff_est_ci` statistics are multiplied by 100
(expressed as percentages), not as proportions.

### 7.6 `mf_method` Options Differ Between Estimate and Test Functions

- `a_proportion_diff_mf`: `"cmh"`, `"cmh_sato"`, `"cmh_mn"`
- `a_test_proportion_diff_mf`: `"cmh"`, `"cmh_sato"`, `"cmh_wh"`

The `"cmh_mn"` (Miettinen-Nurminen) is for CI estimation only.
The `"cmh_wh"` (Wilson-Hilferty) is for testing only.

### 7.7 Footnotes Appear Only When Exact Method is Used

The `{+}` footnote marker and footnote text only appear when the unconditional
exact (or Fisher's exact) method was actually selected. When the CMH method is
used, no footnote is added.

### 7.8 Empty Groups Return NA, Not Errors

When `df` or `.ref_group` is empty (0 rows), the functions return `NA` values
for all statistics with `executed_method = "uncond_exact_diff"` (or `"fisher"`).
This allows tables to build even with sparse data.

### 7.9 `executed_method` is Stripped Before Formatting

The `s_` functions return an `executed_method` field, but the `a_` functions
remove it before calling `format_stats()`. It is used internally to decide
whether to attach the exact footnote.

### 7.10 Multiple Strata Variables Create Cross-Combinations

When multiple strata variables are specified (e.g., `strata = c("f1", "f2")`),
`tern::h_prepare_rsp_table()` creates cross-combinations of their levels as the
analysis strata. Unused cross-combinations (empty strata) may affect the MF
criterion evaluation.
