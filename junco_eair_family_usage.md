# junco EAIR Family: `a_patyrs_j`, `a_eair100_j`, `a_eair_strat_j`

Expert usage reference for the exposure-adjusted incidence rate (EAIR) analysis
function family in the junco package.

## 1. Function Overview

| Function | Purpose | Typical Layout Role |
|---|---|---|
| `a_patyrs_j` | Patient-years exposure summary row | `analyze("TRTDURY", afun = a_patyrs_j)` — produces a single "Patient years" row above the EAIR rows |
| `a_eair100_j` | Unstratified EAIR per N person-years, with optional risk difference | `analyze("AEDECOD", afun = a_eair100_j)` — one row per level of `.var` |
| `a_eair_strat_j` | Stratified (study-size-weighted) EAIR with pooled Wald CI | `analyze("AEDECOD", afun = a_eair_strat_j)` — same row structure, strata-weighted rates |

All three detect risk-difference columns via `grepl("difference", tolower(colid), fixed = TRUE)` on `.spl_context$cur_col_id`.

## 2. `a_patyrs_j` — Patient Years Row

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `.var` | string | — | Variable containing follow-up duration (e.g. `"TRTDURY"`) |
| `id` | string | `"USUBJID"` | Subject identifier |
| `source` | string | `"alt_df"` | `"alt_df"` → sum from `alt_counts_df`; `"df"` → sum from analysis df |
| `.stats` | character | `"patyrs"` | Only `"patyrs"` is supported |
| `.formats` | named char/list | `NULL` | Override default `jjcsformat_xx("xx.x")` |
| `.labels` | named char | `NULL` | Override default `"Patient years"` |

### Key Behaviors

- When `source = "alt_df"` (default), patient-years are summed from `alt_counts_df` after restricting to the current column via `.spl_context$cur_col_expr`. This gives the **full population** person-years.
- When `source = "df"`, patient-years are summed from the analysis dataset (only subjects with events in the current row split).
- Deduplicates on `(id, .var)` before summing.
- In risk-difference columns, returns `NULL` (blank cell).
- Requires `alt_counts_df` in `build_table()` when `source = "alt_df"`.

### Minimal Example

```r
analyze("TRTDURY", show_labels = "hidden", afun = a_patyrs_j,
        extra_args = list(.labels = c(patyrs = "Subject years")))
```

## 3. `a_eair100_j` — Unstratified EAIR

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `.var` | string | — | AE term variable (e.g. `"AEDECOD"`) — must be a factor |
| `fup_var` | string | — | **Required.** Follow-up duration variable on `alt_counts_df`, in **years** |
| `occ_var` | string or NULL | — | First-occurrence flag variable (`"Y"` = event). If `NULL`, all records count |
| `occ_dy` | string or NULL | `NULL` | Event onset day variable (days). Required when `occ_var` is not NULL |
| `num_p_year` | numeric | `100` | Scaling factor: rate per N person-years |
| `count_multiple_events` | logical | `FALSE` | When `occ_var = NULL`: `TRUE` = count total events; `FALSE` = count unique subjects |
| `id` | string | `"USUBJID"` | Subject identifier |
| `drop_levels` | logical | `FALSE` | Drop unobserved factor levels. Auto-set to `TRUE` when `.var` matches a prior `split_rows_by` |
| `riskdiff` | logical | `TRUE` | Enable risk-difference calculations |
| `ref_path` | character | `NULL` | Column path to reference group. **Required when `riskdiff = TRUE`** |
| `riskdiff_setup` | string | `"horizontal"` | `"horizontal"` = diff in separate columns; `"vertical"` = diff as extra rows |
| `conf_level` | numeric | `0.95` | Confidence level |
| `conf_type_eair` | string | `"normal"` | CI method for the rate itself: `"normal"`, `"normal_log"`, `"exact"`, `"byar"` |
| `conf_type_diff` | string | `"wald"` | CI method for rate difference: `"wald"`, `"waldcc"`, `"mn"`, `"scas"` |
| `.stats` | character | `"eair"` | Statistics to display (see §3.1) |
| `.formats` | named char/list | `NULL` | Override default formats |
| `.labels` | named char | `NULL` | Override default labels |
| `.indent_mods` | named integer | `NULL` | Indent modifiers |
| `na_str` | character | `rep("NA", 3)` | NA replacement strings |
| `row_labels_adj` | logical | `FALSE` | When `TRUE` and single stat requested, use level name as row label instead of stat label |
| `labelstr` | string | `NULL` | Present for cfun compatibility; not user-set |

### 3.1 Available Statistics

View with `junco_get_stats("a_eair100_j")`:

| Statistic | Width | Description | Default Format |
|---|---|---|---|
| `n_event` | 1 | Number of subjects with event (or total events if `count_multiple_events`) | `xx` |
| `person_years` | 1 | Total person-years of follow-up | `xx.x` |
| `eair` | 1 | EAIR per `num_p_year` PY | `xx.x` |
| `n_eair` | 2 | `(n_event, eair)` | `xx (xx.x)` |
| `eair_n` | 2 | `(eair, n_event)` | `xx.x (xx)` |
| `eair_n_py` | 3 | `(eair, n_event, person_years)` | `xx.x (xx/xx.x)` |
| `eair_ci` | 2 | CI for rate (method per `conf_type_eair`) | `(xx.xx, xx.xx)` |
| `eair_est_ci` | 3 | `(eair, lcl, ucl)` | `xx.xx (xx.xx, xx.xx)` |
| `eair_diff_est` | 1 | Difference in EAIR (current − reference) | `xx.xx` |
| `eair_diff_ci` | 2 | CI for difference | `(xx.xx, xx.xx)` |
| `eair_diff_est_ci` | 3 | `(difference, lcl, ucl)` | `xx.xx (xx.xx, xx.xx)` |

### 3.2 Person-Years Calculation

The helper `h_get_eair_df` computes per-subject at-risk time:

- **When `occ_var` is not NULL:**
  - Subjects with event (`occ_var == "Y"`): at-risk time = `occ_dy / 365.25`
  - Subjects without event: at-risk time = `fup_var` (from `alt_counts_df`)
  - Input must have exactly one record per subject per `.var` level per `occ_var == "Y"`. Duplicates cause an error.

- **When `occ_var` is NULL:**
  - All subjects use `fup_var` from `alt_counts_df` as at-risk time
  - If `count_multiple_events = TRUE`: n_event = total event count across all records
  - If `count_multiple_events = FALSE`: n_event = number of unique subjects with ≥1 record

### 3.3 Risk Difference Layouts

#### Horizontal (default: `riskdiff_setup = "horizontal"`)

Risk differences appear in **separate columns** (the standard spanning-header pattern).
In risk-diff columns, the core-column stats (`eair`, `n_eair`, `eair_n`, `eair_n_py`) are
**automatically replaced** by `eair_diff_est_ci`. You do NOT request `eair_diff_*` stats
explicitly — they are injected.

If you explicitly include `eair_diff_est_ci`/`eair_diff_est`/`eair_diff_ci` in `.stats`,
they are **silently removed** from the core columns (they only appear in the diff columns).

#### Vertical (`riskdiff_setup = "vertical"`)

Risk differences appear as **extra rows** beneath each level's core stats. You must
explicitly include `eair_diff_est_ci` (or `eair_diff_est`, `eair_diff_ci`) in `.stats`.
The reference column shows blank cells for these rows.

### 3.4 `n_event` Label is Dynamic

The label for `n_event` changes based on configuration:
- `occ_var` is not NULL OR `count_multiple_events = FALSE` → `"Number of subjects with event"`
- `occ_var` is NULL AND `count_multiple_events = TRUE` → `"Number of events"`

This label propagates to composite stats (`n_eair`, `eair_n`, `eair_n_py`).

### 3.5 Row Label Behavior

When `.var` is used in a prior `split_rows_by` call (detected via `.spl_context`):
- `drop_levels` is auto-set to `TRUE`
- Row labels show the **statistic name** (e.g., "eair (per 100 person years)")

When `.var` is NOT in a prior split:
- If multiple stats: labels are prefixed with the level name (e.g., "dcd A.1.1.1.1 eair ...")
- If single stat + `row_labels_adj = TRUE`: labels show the **level name** only

## 4. `a_eair_strat_j` — Stratified EAIR

Extends `a_eair100_j` with study-size-weighted pooling across strata.

### Additional/Changed Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `strata` | string | — | **Required.** Stratification variable (e.g. `"STUDYID"`). Must exist on `alt_counts_df` |

All other parameters match `a_eair100_j`. The `.stats` names are prefixed with `_strat`:

### Available Statistics

View with `junco_get_stats("a_eair_strat_j")`:

| Statistic | Description |
|---|---|
| `n_event_total` | Total event count across strata |
| `person_years_total` | Total person-years across strata |
| `eair_strat` | Stratified pooled EAIR |
| `n_eair_strat` | `(n_event_total, eair_strat)` |
| `eair_strat_ci` | Wald CI for pooled rate |
| `eair_strat_est_ci` | `(eair_strat, lcl, ucl)` |
| `eair_strat_n` | `(eair_strat, n_event_total)` |
| `eair_strat_n_py` | `(eair_strat, n_event_total, person_years_total)` |
| `eair_strat_diff_est_ci` | Stratified rate difference with Wald CI |
| `eair_strat_diff_est` | Difference estimate only |
| `eair_strat_diff_ci` | CI for difference only |

### Weighting Formula

Strata weight: `w_i = T_0i + T_1i` (total person-years across ALL arms in stratum i).

Pooled rate: `r_hat = sum(w_i * r_i) / W` where `W = sum(w_i)`.

Wald CI: `r_hat ± z * (num_p_year / W) * sqrt(sum(w_i^2 * n_i / T_i^2))`.

## 5. Data Requirements

### `alt_counts_df` (Required for all three functions)

Must contain:
- `id` column (default `"USUBJID"`)
- `fup_var` column — follow-up duration in **years** (not days)
- Treatment variable used in column splits
- For `a_eair_strat_j`: the `strata` variable

### Analysis dataset (`df`)

Must contain:
- `id` column
- `.var` column (the AE term, must be a factor for `a_eair100_j`/`a_eair_strat_j`)
- `occ_var` column (if not NULL) — first-occurrence flag, encoded as `"Y"`
- `occ_dy` column (if `occ_var` not NULL) — event onset day in **days**
- Treatment variable and all column-split variables

### Common Data Prep Pattern

```r
adsl <- ex_adsl |>
  mutate(TRTDURY = as.numeric(difftime(TRTEDT, TRTSDTM, units = "days")) / 365.25)

adae <- ex_adae |>
  group_by(USUBJID, AEDECOD) |>
  arrange(ASTDY) |>
  slice(1) |>
  ungroup() |>
  mutate(AOCCPFL = "Y")

aefup <- left_join(adae, adsl, by = "USUBJID")
```

## 6. CI Method Reference

### Rate CI (`conf_type_eair`)

| Value | Method | Notes |
|---|---|---|
| `"normal"` | Normal approximation | Default for `a_eair100_j` |
| `"normal_log"` | Log-normal | |
| `"exact"` | Exact (Clopper-Pearson-like) | Default for `s_eair100_levii_j` |
| `"byar"` | Byar's method | |

### Difference CI (`conf_type_diff`)

| Value | Method | Notes |
|---|---|---|
| `"wald"` | Wald without continuity correction | Default |
| `"waldcc"` | Wald with continuity correction | Adds `0.5 * (1/n1 + 1/n2)` to half-width |
| `"mn"` | Miettinen-Nurminen score | Iterative score-based CI |
| `"scas"` | Skewness-corrected asymptotic score | Extension of MN with skewness correction |

The difference CI is computed on the **unscaled** rate (events/person-years), then multiplied by `num_p_year`.

## 7. Complete Worked Examples

### 7.1 Basic EAIR Table with Horizontal Risk Difference

```r
library(junco); library(rtables); library(tern); library(dplyr)

trtvar <- "ARM"; ctrl_grp <- "B: Placebo"

adsl <- ex_adsl |>
  create_colspan_var(
    non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ",
    active_grp_span_lbl = "Active Study Agent",
    colspan_var = "colspan_trt", trt_var = trtvar
  ) |>
  mutate(
    rrisk_header = "Risk Difference (95% CI)",
    rrisk_label = paste(!!rlang::sym(trtvar), "vs", ctrl_grp),
    TRTDURY = ifelse(is.na(EOSDY), 365, as.numeric(EOSDY))
  ) |>
  select(USUBJID, all_of(trtvar), colspan_trt, rrisk_header, rrisk_label, TRTDURY)

adae <- ex_adae |>
  group_by(USUBJID, AEDECOD) |> arrange(ASTDY) |> slice(1) |> ungroup() |>
  mutate(AOCCPFL = "Y") |>
  select(USUBJID, AEDECOD, ASTDY, AOCCPFL)

aefup <- left_join(adae, adsl, by = "USUBJID")

colspan_trt_map <- create_colspan_map(adsl,
  non_active_grp = ctrl_grp, non_active_grp_span_lbl = " ",
  active_grp_span_lbl = "Active Study Agent",
  colspan_var = "colspan_trt", trt_var = trtvar)

ref_path <- c("colspan_trt", " ", trtvar, ctrl_grp)

lyt <- basic_table(show_colcounts = TRUE, colcount_format = "N=xx") |>
  split_cols_by("colspan_trt",
    split_fun = trim_levels_to_map(map = colspan_trt_map)) |>
  split_cols_by(trtvar) |>
  split_cols_by("rrisk_header", nested = FALSE) |>
  split_cols_by(trtvar, labels_var = "rrisk_label",
    split_fun = remove_split_levels(ctrl_grp)) |>
  analyze("TRTDURY", show_labels = "hidden", afun = a_patyrs_j,
    extra_args = list(.labels = c(patyrs = "Subject years"))) |>
  analyze("AEDECOD", afun = a_eair100_j,
    extra_args = list(
      fup_var = "TRTDURY", occ_var = "AOCCPFL", occ_dy = "ASTDY",
      ref_path = ref_path, drop_levels = TRUE, row_labels_adj = TRUE
    ))

build_table(lyt, aefup, alt_counts_df = adsl)
```

### 7.2 EAIR with Multiple Stats and Row Splits

```r
lyt <- basic_table() |>
  split_cols_by("ARM") |>
  split_rows_by("AEDECOD", split_fun = drop_split_levels) |>
  analyze("AEDECOD", afun = a_eair100_j,
    extra_args = list(
      fup_var = "TRTDURY", occ_var = "AOCCPFL", occ_dy = "ASTDY",
      ref_path = c("ARM", ctrl_grp),
      riskdiff = FALSE,
      .stats = c("person_years", "n_eair", "eair_ci"),
      num_p_year = 1000,
      conf_type_eair = "exact"
    ))
```

### 7.3 EAIR with `occ_var = NULL` (All-Records Mode)

```r
# Count unique subjects with event (default)
analyze("AEDECOD", afun = a_eair100_j,
  extra_args = list(
    fup_var = "TRTDURY", occ_var = NULL, occ_dy = NULL,
    ref_path = ref_path, count_multiple_events = FALSE
  ))

# Count total events
analyze("AEDECOD", afun = a_eair100_j,
  extra_args = list(
    fup_var = "TRTDURY", occ_var = NULL, occ_dy = NULL,
    ref_path = ref_path, count_multiple_events = TRUE
  ))
```

When `occ_var = NULL`, person-years for ALL subjects come from `fup_var` in `alt_counts_df`
(no event-day substitution occurs).

### 7.4 Stratified EAIR

```r
lyt <- basic_table() |>
  split_cols_by("ARM") |>
  analyze("AEDECOD", afun = a_eair_strat_j,
    extra_args = list(
      strata = "STUDYID",
      fup_var = "TRTDURY", occ_var = "AOCCPFL", occ_dy = "ASTDY",
      .stats = c("eair_strat", "eair_strat_ci"),
      riskdiff = TRUE,
      ref_path = c("ARM", "Placebo"),
      riskdiff_setup = "vertical"
    ))
```

### 7.5 Vertical Risk Difference Layout

```r
lyt <- basic_table() |>
  split_cols_by("ARM") |>
  split_rows_by("AEDECOD", split_fun = drop_split_levels) |>
  analyze("AEDECOD", afun = a_eair100_j,
    extra_args = list(
      fup_var = "TRTDURY", occ_var = "AOCCPFL", occ_dy = "ASTDY",
      ref_path = c("ARM", ctrl_grp),
      riskdiff = TRUE,
      riskdiff_setup = "vertical",
      .stats = c("eair", "eair_diff_est_ci")
    ))
```

In vertical mode, the reference column shows blank cells for `eair_diff_*` rows.

## 8. Gotchas and Pitfalls

### 8.1 `fup_var` Must Be in Years

The function does NOT convert units. If your follow-up variable is in days, divide by 365.25
before passing. The `occ_dy` variable IS in days and is internally converted via `/ 365.25`.

### 8.2 `alt_counts_df` is Mandatory

All three functions require `alt_counts_df` in `build_table()`. `a_eair100_j` will error:
`"a_eair100_j: .alt_df_full cannot be NULL"`.

### 8.3 Duplicate Records with `occ_var`

When `occ_var` is not NULL, the input must have exactly one record per subject per `.var`
level where `occ_var == "Y"`. Duplicates trigger:
`"Input dataset must uniquely identify one record per subject/.var/occ_var."`

### 8.4 `ref_path` Required When `riskdiff = TRUE`

Unlike `a_freq_j` where `grouped_cols_w_diffs()` auto-injects `ref_path`, `a_eair100_j`
requires explicit `ref_path` in `extra_args` when using the manual column structure pattern.
When using `grouped_cols_w_diffs()`, `ref_path` IS auto-injected — do NOT also pass it
in `extra_args` or you get a "matched by multiple actual arguments" error.

### 8.5 `riskdiff = TRUE` is the Default

Unlike `a_freq_j`, `a_eair100_j` defaults to `riskdiff = TRUE`. If you have no risk-diff
columns in your layout, set `riskdiff = FALSE` explicitly, or you will get an error about
`ref_path` being NULL.

### 8.6 Horizontal Layout Auto-Replaces Stats in Diff Columns

In horizontal mode, you do NOT request `eair_diff_*` stats. The function automatically
replaces `eair`/`n_eair`/`eair_n`/`eair_n_py` with `eair_diff_est_ci` when it detects
it is inside a risk-difference column. Requesting diff stats explicitly in `.stats` with
horizontal mode causes them to be silently dropped from core columns.

### 8.7 `conf_type_eair` Default Differs Between `s_` and `a_` Functions

- `s_eair100_levii_j` defaults to `conf_type_eair = "exact"`
- `a_eair100_j` defaults to `conf_type_eair = "normal"`

This means the `a_` function uses normal approximation by default. Set `"exact"` explicitly
if exact CIs are required.

### 8.8 NA Values in `fup_var`

If `fup_var` contains NA values on `alt_counts_df`, the person-years sum will be NA,
causing downstream errors. Ensure `fup_var` has no missing values.
