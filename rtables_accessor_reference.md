# `rtables` Accessor Reference

Quick-lookup reference for all user-facing accessors on built `TableTree` objects and related structures. Organized by task category.

> This file is a **reference**, not a skill. For decision-making guidance on when and how to use introspection, sorting, and pruning, see [introspection_sort_prune.md](./introspection_sort_prune.md).

---

## 1. Read Structure

| Accessor | Operates On | Returns | Notes |
|---|---|---|---|
| `tree_children(tt)` | `TableTree`, `ElementaryTable` | List of direct children (subtables or rows) | Primary way to iterate over a node's children |
| `content_table(tt)` | `TableTree` | `ElementaryTable` containing content rows (from `summarize_row_groups()`), or `NULL` | Returns `NULL` for `ElementaryTable` or non-table objects |
| `collect_leaves(tt, incl.cont, add.labrows)` | `TableTree`, `ElementaryTable` | Flat list of all `TableRow` objects | `incl.cont = TRUE` includes content rows; `add.labrows = FALSE` excludes label rows by default |
| `obj_name(x)` | Any table node or split | Character: structural name (used in paths) | May differ from display label; use for path construction |
| `obj_label(x)` | Any table node, row, or split | Character: display label | Use `obj_label<-` to modify post-build |
| `obj_avar(x)` | `TableRow`, `ElementaryTable` | Character: the analysis variable name | The variable passed to `analyze()` |
| `nrow(tt)` | `TableTree`, `ElementaryTable` | Integer: number of visible rows | |
| `ncol(tt)` | `TableTree`, `ElementaryTable`, `TableRow` | Integer: number of leaf columns | |
| `dim(tt)` | `TableTree`, `ElementaryTable` | Integer vector: `c(nrow, ncol)` | |

---

## 2. Read Values

| Accessor | Operates On | Returns | Wildcard `*` Support | Notes |
|---|---|---|---|---|
| `cell_values(tt, rowpath, colpath, omit_labrows)` | `TableTree`, `ElementaryTable`, `TableRow` | List (single row) or list-of-lists (multiple rows) of cell values | `colpath` only | Primary value extractor; see [introspection_sort_prune.md §2.1](./introspection_sort_prune.md) |
| `row_values(row)` | `TableRow` | Named list of cell values (one per column) | N/A (no path args) | Operates on a row object, not a path |
| `value_at(tt, rowpath, colpath)` | `TableTree`, `ElementaryTable`, `TableRow` | Single cell value | No | Errors if path resolves to more than one cell |
| `row_cells(row)` | `TableRow` | List of `CellValue` objects (formatted, with attributes) | N/A | Lower-level than `row_values`; preserves format/footnote attributes |
| `value_formats(tt)` | `TableTree`, `ElementaryTable`, `TableRow` | Matrix (storage mode list) of effective formats per cell | N/A | Includes `NULL` for label rows |

---

## 3. Read Column Info

| Accessor | Operates On | Returns | Notes |
|---|---|---|---|
| `col_counts(tt, path)` | `TableTree`, `InstantiatedColumnInfo` | Integer vector of leaf column counts | `path = NULL` returns all leaf counts |
| `col_total(tt)` | `TableTree`, `InstantiatedColumnInfo` | Integer: total population count | The `N` from `alt_counts_df` or `df` |
| `facet_colcount(tt, path)` | `TableTree`, `LayoutColTree` | Integer: count for a specific column facet | Path need not go to a leaf; works at any column nesting depth |
| `colcount_visible(tt, path)` | `TableTree`, `LayoutColTree` | Logical: whether the column count is displayed | |
| `clayout(tt)` | `TableTree` | `LayoutColTree`: the column tree structure | Rarely needed directly; use `col_paths_summary()` instead |

---

## 4. Modify Values

| Setter | Operates On | Value | Notes |
|---|---|---|---|
| `row_values(row) <- val` | `TableRow` | List of raw values | Wraps each value in `rcell()` |
| `row_cells(row) <- val` | `TableRow` | List of `CellValue` objects | Lower-level; preserves existing format/footnote attributes only if passed `CellValue`s |
| `col_counts(tt, path) <- val` | `TableTree`, `InstantiatedColumnInfo` | Integer vector | `path = NULL` sets all leaf counts; length must match |
| `col_total(tt) <- val` | `TableTree`, `InstantiatedColumnInfo` | Integer | |
| `facet_colcount(tt, path) <- val` | `TableTree`, `LayoutColTree` | Integer (or `NA` for blank display) | Does **not** auto-update parent facet counts |
| `colcount_visible(tt, path) <- val` | `TableTree`, `LayoutColTree` | Logical | Siblings must have matching visibility or printing errors |

---

## 5. Modify Display

| Setter | Operates On | Value | Notes |
|---|---|---|---|
| `obj_label(x) <- val` | Any table node, row, or split | Character | Changes display label (not structural name used in paths) |
| `obj_format(x) <- val` | `TableRow`, `CellValue`, `TableTree` | Format spec (string, function, or list) | Non-recursive; use `set_format_recursive()` for tree-wide changes |
| `indent_mod(x) <- val` | `TableRow`, `TableTree`, `CellValue`, `RowsVerticalSection` | Integer | Adjusts indentation relative to structural depth |
| `top_left(tt) <- val` | `TableTree` | Character vector | Each element is one line in the top-left corner area |
| `horizontal_sep(tt) <- val` | `TableTree` | Single character string | Applied recursively to all subtables |
| `section_div(tt) <- val` | `TableTree` | Character vector (length = `nrow(tt)` or shorter) | See `section_div_at_path<-` for targeted changes; `NA_character_` = no divider |
| `section_div_at_path(tt, path) <- val` | `TableTree` | Single character string | Supports `"*"` wildcards in path; preferred over `section_div<-` for layout-declared dividers |

---

## 6. Modify Titles & Footnotes

| Setter | Operates On | Value | Notes |
|---|---|---|---|
| `main_title(tt) <- val` | `TableTree` | Single string | |
| `subtitles(tt) <- val` | `TableTree` | Character vector | |
| `main_footer(tt) <- val` | `TableTree` | Character vector | |
| `prov_footer(tt) <- val` | `TableTree` | Character vector | Provenance/source footer |
| `fnotes_at_path(tt, rowpath, colpath) <- val` | `TableTree` | Character (footnote text) | `rowpath = NULL` targets column header; `colpath = NULL` targets row; both non-NULL targets a cell |
| `row_footnotes(row) <- val` | `TableRow` | List of `RefFootnote` objects or character | |
| `cell_footnotes(cell) <- val` | `CellValue` | List of `RefFootnote` objects or character | |

---

## 7. Layout Introspection

| Accessor | Operates On | Returns | Notes |
|---|---|---|---|
| `vars_in_layout(lyt)` | `PreDataTableLayouts` | Character vector of variable names used in the layout | Does not detect implicit dependencies inside `afun`/`cfun` bodies |
| `clayout(lyt)` | `PreDataTableLayouts` | `PreDataColLayout`: the pre-data column layout | |
| `rlayout(lyt)` | `PreDataTableLayouts` | `PreDataRowLayout`: the pre-data row layout | Internal but occasionally useful for debugging |

---

## 8. Miscellaneous

| Accessor | Operates On | Returns | Notes |
|---|---|---|---|
| `splv_extra(sv)` | `SplitValue` | Named list of child-specific extra args | Distinct from `split_exargs()` which applies to all children |
| `value_expr(sv)` | `ValueWrapper` | Expression used for subsetting, or `NULL` | |
| `no_colinfo(tt)` | `TableTree`, `TableRow` | Logical: `TRUE` if no column structure is attached | |
| `names(tt)` | `TableTree` | Top-level column split values repped across spanned columns | |
| `row.names(tt)` | `TableTree` | Character vector of row display labels | |
