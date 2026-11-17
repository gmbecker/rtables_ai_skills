# Prompt: Extract Column Structure from a Table Shell

You are parsing a clinical trial table shell (mock shell) rendered as text or extracted from a PDF. Your task is to determine the **exact hierarchical column tree**.

## Critical Concepts

### 1. Siblings are semantically related — use this as the PRIMARY grouping signal

Direct sibling columns (children of the same parent node) are always **semantically related**: they are meant to be understood in the context of each other. There are three distinct sibling relationship types:

- **Type A — Levels of a single categorical variable**: subpopulations or categories of the same kind (e.g., treatment arms: "Drug A", "Drug B", "Placebo"; or combination/total levels: "Drug A", "Drug B", "All Active"). A single subject belongs to exactly one sibling.
- **Type B — Multiple variables or measures for the same entity**: different measurements, statistics, or analysis variables that together describe the parent (e.g., "Level" and "Change From Baseline" under a treatment arm; or "n/N (%)", "Mean (95% CI)", "Mean CFB (95% CI)" under a treatment arm). A single subject contributes to *every* sibling.
- **Type C — Overlapping / non-mutually-exclusive subsets**: a total or "all" column alongside a filtered subset of the same analysis (e.g., "All Grades" and "High Toxicity" under a treatment arm; or "All AEs" and "Drug-Related AEs"). A single subject may appear in more than one but not necessarily all siblings.

**Distinguishing the three types**:
- **Type A**: Sibling labels name subpopulations — a subject is in one *or* the other (mutually exclusive).
- **Type B**: Sibling labels name different measurements — a subject has *all* of them (e.g., both a Level and a Change From Baseline).
- **Type C**: Sibling labels name overlapping scopes — a subject may appear in *more than one* sibling because one is a subset of the other (e.g., a Grade 4 AE appears in both "All Grades" and "High Toxicity").

**This is the most reliable tool for resolving column structure.** When a spanning header is present, ask: *"Does every leaf column beneath it semantically belong to the concept named by that header?"* If a column does not belong, it is NOT under that span — regardless of how the visual alignment appears.

**Example 1 — blank spanning label detected via semantic test:**

```
          Active Treatment
         Active 1   Active 2   Placebo
```

A naive positional reading might conclude `Active Treatment -> [Active 1, Active 2, Placebo]`. But Placebo is **not an active treatment**. It does not semantically belong under "Active Treatment". Therefore the correct structure must be:

```
Root
├── Active Treatment
│   ├── Active 1
│   └── Active 2
└── ""  (blank spanning label — structurally present, visually absent)
    └── Placebo
```

**Example 2 — multi-variable siblings (Type B) correctly grouped under a treatment arm:**

```
           A: Drug X                          B: Placebo
     Level   Change From Baseline       Level   Change From Baseline
```

"Level" and "Change From Baseline" are both measurements *of* Drug X subjects — they semantically belong under "A: Drug X". The repeated pattern under each arm confirms nesting. The correct structure is:

```
Root
├── A: Drug X
│   ├── Level          (variable: AVAL)
│   └── Change From Baseline  (variable: CHG)
└── B: Placebo
    ├── Level          (variable: AVAL)
    └── Change From Baseline  (variable: CHG)
```

**Example 3 — blank spanning label in a shift table:**

```
                       Baseline
      N   Grade 1   Grade 2   Grade 3   Total
```

"N" is a count — it is not a baseline grade. It does not semantically belong under "Baseline". The correct structure is:

```
Root
├── ""  (blank spanning label)
│   └── N
└── Baseline
    ├── Grade 1
    ├── Grade 2
    ├── Grade 3
    └── Total
```

### 2. Spanning headers are centered over the columns they span

A label that appears above and horizontally centered across two or more leaf columns is a **parent node** in the column hierarchy — it is NOT a leaf column itself. Use character alignment and horizontal extent as a secondary signal (after semantic analysis) to confirm which leaf columns fall under which spanning header.

### 3. Blank spanning labels are real structural nodes

A group of leaf columns may have a parent spanning label that is **visually blank** (empty string or whitespace). This is NOT the same as having no parent — it is a real structural level with an empty label.

**How to detect blank spanning labels**: When a visible spanning header is present but some leaf columns at the same depth do not semantically belong under it, those columns are under a **blank spanning label** — a sibling of the visible span, not a child of it. The blank span is structurally present but renders as empty space.

Blank spanning labels are common in clinical tables. Typical cases include:
- A control/placebo arm sitting beside active treatment arms grouped under an "Active Treatment" spanner
- A standalone "N" column beside graded categories grouped under a "Baseline" spanner
- Risk difference columns under their own spanner, parallel to treatment columns under a separate spanner

### 4. Distinguish blank spanning labels from variable-depth (parentless) leaves

A **blank spanning label** and a **variable-depth leaf** both result in a column with no visible text above it, but they are structurally different:

- **Blank spanning label**: The column is at the SAME tree depth as its siblings. There is a real parent node with an empty label. The column's leaf label occupies the same header row as other leaf labels.
- **Variable-depth leaf**: The column is at a SHALLOWER tree depth. It has no parent node at all — it is a top-level sibling branch. The column's label occupies a higher header row than the deepest leaf labels.

Example of a **variable-depth leaf** ("All Patients" is at depth 1; ARM×SEX children are at depth 2):

```
                    A: Drug X                B: Placebo
All Patients    F      M      U          F      M      U
```

"All Patients" appears on the leaf row with no spanning header row entry — it is a shallower sibling branch, not a blank-spanned node.

### 5. Repeated sub-headers imply nesting; distinct sub-headers imply sibling branches

- If the same set of lower-level headers appears under each parent header (e.g., "F", "M" repeated under each treatment arm), that is a **nested (crossed) split**.
- If different lower-level headers appear under different top-level spanners (e.g., treatment arm names under "Active Treatment" but comparison labels like "Drug X vs Placebo" under "Risk Differences"), those are **parallel sibling branches**.

### 6. Top-left material is NOT column structure

Any text in the top-left corner of the table — above the row labels and to the left of the first data column — is a table title, subtitle, or row-label header. It describes row structure or provides context. **Never interpret top-left material as a column spanning header.**

```
STRATA1   A: Drug X   B: Placebo   C: Combination
```

"STRATA1" is a top-left label (above where row labels will appear). The column structure is only `[A: Drug X, B: Placebo, C: Combination]`.

## Procedure

1. **Identify the data columns.** Locate the leftmost boundary of data columns. Everything to the left of this boundary in the header area is top-left material — ignore it for column structure.

2. **Identify the leaf (bottom) header row.** This is the lowest row of column headers. List all leaf column labels and their approximate horizontal positions.

3. **For each spanning label above the leaf row**, determine its candidate children by horizontal alignment, then **apply the semantic test**: does each candidate leaf column genuinely belong to the concept named by the spanning label? Exclude any column that fails the semantic test.

4. **For any leaf columns excluded by the semantic test** (or not covered by any visible spanning label) at a header level where sibling columns ARE covered: assign a blank spanning label (`""`) as their parent at that level. Do NOT leave them parentless. Do NOT attach them to a visible span they do not semantically belong to.

5. **Build the tree bottom-up.** Attach each spanning label (visible or blank) as a parent of the leaf columns it covers.

6. **Validate:**
   - Every leaf column appears exactly once.
   - **Every sibling group is semantically coherent.** If siblings look incoherent (e.g., a placebo arm grouped with active arms under an "Active Treatment" label, or a count column grouped with grade categories under a "Baseline" label), the tree is wrong — re-examine.
   - The number of tree levels is consistent: if any branch has depth N, all branches should have depth N (with blank labels filling in where visible labels are absent). The only exception is true variable-depth layouts (see Concept 4).

## Output Format

Return the column structure as a nested tree:

```
Root
├── [Spanning Label A]
│   ├── Leaf Column 1
│   └── Leaf Column 2
├── [""]  (blank spanning label)
│   └── Leaf Column 3
└── [Spanning Label B]
    ├── Leaf Column 4
    └── Leaf Column 5
```

Then separately state the nesting levels with their variable interpretation (e.g., "Level 1: Treatment grouping; Level 2: Treatment arm").

> **Note**: This extraction procedure aligns with the Mandatory Semantic Coherence Validation Gate in [col_struct_layout.md](./col_struct_layout.md) Section 2.3. The semantic test described here (Concept 1) is the same gate applied during column structure implementation.
