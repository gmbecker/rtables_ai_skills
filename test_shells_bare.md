# Test Table Shells for Skill Reasoning Path Evaluation

Use the pharmaverseadamjnj R package data unless otherwise specified.

---

## Shell T01: Demographics and Baseline Characteristics

```
TSFDEM01: Demographics and Baseline Characteristics; Safety Analysis Set

————————————————————————————————————————————————————————————————————————
                               A: Drug X    B: Placebo    C: Combination
                                 N=XXX          N=XXX            N=XXX
————————————————————————————————————————————————————————————————————————
Age (years)
  N                               xx            xx              xx
  Mean (SD)                   xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)
  Median                         xx.x          xx.x            xx.x
  Min, Max                    xx.x, xx.x    xx.x, xx.x      xx.x, xx.x
Sex, n (%)
  Female                      xx (xx.x%)    xx (xx.x%)      xx (xx.x%)
  Male                        xx (xx.x%)    xx (xx.x%)      xx (xx.x%)
Race, n (%)
  Asian                       xx (xx.x%)    xx (xx.x%)      xx (xx.x%)
  Black or African American   xx (xx.x%)    xx (xx.x%)      xx (xx.x%)
  White                       xx (xx.x%)    xx (xx.x%)      xx (xx.x%)
  Other                       xx (xx.x%)    xx (xx.x%)      xx (xx.x%)
————————————————————————————————————————————————————————————————————————

Note: Percentages are based on the number of subjects in each treatment group.
```

Programming Notes:
1. Use ADSL as the analysis dataset.
2. Percentages for categorical variables are based on column N.

---

## Shell T02: Subjects With TEAEs by Sex

```
TSFAE-SUBGRP: Subjects With TEAEs by Sex; Safety Analysis Set

———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                  Active Study Agent                                          Risk Difference (%) (95% CI)
                              A: Drug X      C: Combination      B: Placebo      A: Drug X vs B: Placebo   C: Combination vs B: Placebo
Gender                          N=XXX              N=XXX              N=XXX                  N=XXX                         N=XXX
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Female                           xx                xx                xx
  Subjects with >=1 TEAE   xx / xx (xx.x%)   xx / xx (xx.x%)   xx / xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  SOC1                     xx / xx (xx.x%)   xx / xx (xx.x%)   xx / xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
    PT1                    xx / xx (xx.x%)   xx / xx (xx.x%)   xx / xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
Male                             xx                xx                xx
  Subjects with >=1 TEAE   xx / xx (xx.x%)   xx / xx (xx.x%)   xx / xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  SOC1                     xx / xx (xx.x%)   xx / xx (xx.x%)   xx / xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
    PT1                    xx / xx (xx.x%)   xx / xx (xx.x%)   xx / xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Percentages are based on the number of subjects of each sex in each treatment group.
```

Programming Notes:
1. Source dataset: ADAE with alt_counts_df = ADSL.
2. Percentages use sex-specific denominators within each arm (e.g., Female subjects in Arm A).
3. Risk differences use the Wald method.
4. Sort SOC and PT by decreasing incidence within each sex subgroup.

---

## Shell T03: Subject Disposition

```
TSFDS01: Subject Disposition; Safety Analysis Set

——————————————————————————————————————————————————————————————————
                          A: Drug X    B: Placebo   C: Combination
                             N=XXX         N=XXX           N=XXX
——————————————————————————————————————————————————————————————————
Completed                 xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Discontinued              xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  Adverse Event           xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  Lack of Efficacy        xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  Lost to Follow-up       xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  Physician Decision      xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  Protocol Violation      xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  Withdrawal by Subject   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Ongoing                   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
——————————————————————————————————————————————————————————————————

Note: Percentages are based on the number of subjects in each treatment group.
```

Programming Notes:
1. Source dataset: ADSL.
2. Percentages are based on column N.

---

## Shell T04: AE Table with Maximum Severity Grade and Risk Differences

```
TSFAE-CTC: Overall Summary of TEAEs by Maximum CTC Grade; Safety Analysis Set

——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                             Active Study Agent                                  Risk Difference (%) (95% CI)
                         A: Drug X    C: Combination   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo
                            N=XXX           N=XXX           N=XXX               N=XXX                         N=XXX
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Subjects with >=1 TEAE   xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  Grade 1                xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  Grade 2                xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  Grade 3                xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  Grade 4                xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  Grade 5                xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Each subject is counted once at their maximum CTC grade.
```

Programming Notes:
1. Source dataset: ADAE with alt_counts_df = ADSL.
2. Each subject is counted once at their maximum (worst) CTC grade (AETOXGR).
3. Risk differences use the Wald method.

---

## Shell T05: Vital Signs - Observed Values and Change from Baseline

```
TSFVS01: Vital Signs - Observed Values and Change from Baseline; Safety Analysis Set

————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                                    A: Drug X                                        B: Placebo
Parameter                                             N=XXX                                              N=XXX                         Diff in Mean Change
  Study Visit                      n/N (%)     Mean (95% CI)   Mean CFB (95% CI)     n/N (%)     Mean (95% CI)   Mean CFB (95% CI)         A vs B
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Systolic Blood Pressure (mmHg)
  Screening                      xx.x (xx.x)    xx.x (xx.x)       xx.x (xx.x)      xx.x (xx.x)    xx.x (xx.x)       xx.x (xx.x)          xx.x (xx.x)
  Baseline                       xx.x (xx.x)    xx.x (xx.x)       xx.x (xx.x)      xx.x (xx.x)    xx.x (xx.x)       xx.x (xx.x)          xx.x (xx.x)
  Week 2                         xx.x (xx.x)    xx.x (xx.x)       xx.x (xx.x)      xx.x (xx.x)    xx.x (xx.x)       xx.x (xx.x)          xx.x (xx.x)
  Week 4                         xx.x (xx.x)    xx.x (xx.x)       xx.x (xx.x)      xx.x (xx.x)    xx.x (xx.x)       xx.x (xx.x)          xx.x (xx.x)
————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Mean CFB and Diff columns are blank at Screening and Baseline.
```

Programming Notes:
1. Source dataset: ADVS with alt_counts_df = ADSL.
2. Mean CFB and Diff columns are blank at Screening and Baseline (no change from baseline exists at those visits).
3. Use descriptive statistics (not model-based) for means and differences.

---

## Shell T06: Exposure-Adjusted Incidence Rate Table

```
TSFAE-EAIR: Exposure-Adjusted Incidence Rates of TEAEs; Safety Analysis Set

——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                       Active Study Agent                                              Rate Difference (95% CI)
                                  A: Drug X        C: Combination       B: Placebo      A: Drug X vs B: Placebo   C: Combination vs B: Placebo
                                     N=XXX               N=XXX               N=XXX                  N=XXX                         N=XXX
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Total patient-years                 xxx.x              xxx.x              xxx.x
Subjects with >=1 TEAE
  Subjects with event, n              xx                 xx                 xx
  EAIR per 100 PY (95% CI)     xx.x (xx.x - xx.x)   xx.x (xx.x - xx.x)   xx.x (xx.x - xx.x)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
SOC1
  Subjects with event, n              xx                 xx                 xx
  EAIR per 100 PY (95% CI)     xx.x (xx.x - xx.x)   xx.x (xx.x - xx.x)   xx.x (xx.x - xx.x)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  PT1
    Subjects with event, n            xx                 xx                 xx
    EAIR per 100 PY (95% CI)   xx.x (xx.x - xx.x)   xx.x (xx.x - xx.x)   xx.x (xx.x - xx.x)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: EAIR = number of subjects with event / total patient-years * 100.
```

Programming Notes:
1. Source dataset: ADAE with alt_counts_df = ADSL.
2. EAIR = (number of subjects with event / total patient-years) × 100.
3. Follow-up variable: TRTDURD (treatment duration in days, on ADSL). Subjects with the event contribute time to first event; subjects without contribute full follow-up.
4. First occurrence flag: derive from ADAE (first AE per subject per SOC/PT).
5. Sort SOC and PT by decreasing EAIR. Prune empty levels before sorting.

---

## Shell T07: Shift Table for Laboratory Values

```
TSFLAB-SHIFT: Shift in Laboratory Values From Baseline; Safety Analysis Set

————————————————————————————————————————————————————
Parameter                       Baseline ALB
  Treatment Group   N    LOW   NORMAL   HIGH   Total
————————————————————————————————————————————————————
ALB
  A: Drug X         xx
    LOW                  xx      xx      xx     xx
    NORMAL               xx      xx      xx     xx
    HIGH                 xx      xx      xx     xx
    Total                xx      xx      xx     xx
  B: Placebo        xx
    LOW                  xx      xx      xx     xx
    NORMAL               xx      xx      xx     xx
    HIGH                 xx      xx      xx     xx
    Total                xx      xx      xx     xx
```

Programming Notes:
1. Source dataset: ADLB.
2. The N column shows the number of subjects with both baseline and post-baseline values.
3. Cell values are counts of subjects shifting from baseline category (column) to post-baseline category (row).

---

## Shell T08: Laboratory Parameters by Visit

```
TSFLAB-PARAM: Laboratory Parameters by Visit; Full Analysis Set

——————————————————————————————————————————————————————————
                 A: Drug X    B: Placebo    C: Combination
                   N=XXX          N=XXX            N=XXX
——————————————————————————————————————————————————————————
ALT
  Baseline
    N               xx            xx              xx
    Mean (SD)   xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)
    Median         xx.x          xx.x            xx.x
    Min, Max    xx.x, xx.x    xx.x, xx.x      xx.x, xx.x
  Week 2
    N               xx            xx              xx
    Mean (SD)   xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)
    Median         xx.x          xx.x            xx.x
    Min, Max    xx.x, xx.x    xx.x, xx.x      xx.x, xx.x
CRP
  Baseline
    N               xx            xx              xx
    Mean (SD)   xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)
    Median         xx.x          xx.x            xx.x
    Min, Max    xx.x, xx.x    xx.x, xx.x      xx.x, xx.x
  Week 2
    N               xx            xx              xx
    Mean (SD)   xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)
    Median         xx.x          xx.x            xx.x
    Min, Max    xx.x, xx.x    xx.x, xx.x      xx.x, xx.x
```

Programming Notes:
1. Source dataset: ADLB with alt_counts_df = ADSL.
2. Parameter-specific decimal precision: ALT = 1 decimal, CRP = 2 decimals.

---

## Shell T09: TEAEs by SOC, PT, and Maximum CTC Grade

```
TSFAE-GRADE: TEAEs by System Organ Class, Preferred Term, and CTC Grade; Safety Analysis Set

—————————————————————————————————————————————————————————————
System Organ Class   A: Drug X    B: Placebo   C: Combination
  Preferred Term        N=XXX         N=XXX           N=XXX
—————————————————————————————————————————————————————————————
SOC1                 xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  PT1
    Any Grade        xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Grade 1          xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Grade 2          xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Grade 3          xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
—————————————————————————————————————————————————————————————

Note: Each subject is counted once per SOC and PT at their maximum grade.
```

Programming Notes:
1. Source dataset: ADAE with alt_counts_df = ADSL.
2. Each subject is counted once at their maximum grade per PT.
3. Sort SOC and PT by decreasing incidence. Prune empty levels.

---

## Shell T10: Concomitant Medications by ATC Class and Preferred Name

```
TSFCM01: Concomitant Medications by ATC Class and Preferred Name; Safety Analysis Set

————————————————————————————————————————————————————————————————————————————————
                             Total      A: Drug X    B: Placebo   C: Combination
                              N=XXX         N=XXX         N=XXX           N=XXX
————————————————————————————————————————————————————————————————————————————————
Subjects with >=1 ConMed   xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
ATC2 Class 1               xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  ATC4 Subclass 1          xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Preferred Name 1       xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Preferred Name 2       xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  ATC4 Subclass 2          xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Preferred Name 1       xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Preferred Name 2       xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
————————————————————————————————————————————————————————————————————————————————

Note: Subjects are counted once per medication class/name.
```

Programming Notes:
1. Source dataset: ADCM with alt_counts_df = ADSL.
2. Sort all levels by decreasing incidence in the Total column.

---

## Shell T11: Baseline Characteristics by Treatment and Sex

```
TSFDEM-SUBGRP: Baseline Characteristics by Treatment and Sex; Full Analysis Set

———————————————————————————————————————————————————————————————————————————————————————————————
                             A: Drug X                                B: Placebo
                               N=XXX                                      N=XXX
                                SEX                                       SEX
                 Total        Female         Male          Total        Female         Male
                 N=XXX           N=XXX           N=XXX          N=XXX           N=XXX           N=XXX
———————————————————————————————————————————————————————————————————————————————————————————————
Age (years)
  N               xx            xx            xx            xx            xx            xx
  Mean (SD)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)
  Median         xx.x          xx.x          xx.x          xx.x          xx.x          xx.x
Race, n (%)
  Asian       xx (xx.x%)    xx (xx.x%)    xx (xx.x%)    xx (xx.x%)    xx (xx.x%)    xx (xx.x%)
  White       xx (xx.x%)    xx (xx.x%)    xx (xx.x%)    xx (xx.x%)    xx (xx.x%)    xx (xx.x%)
```

Programming Notes:
1. Source dataset: ADSL.

---

## Shell T12: TEAEs — All vs Drug-Related

```
TSFAE-RELATED: TEAEs - All vs Drug-Related; Safety Analysis Set

————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                         All AEs                                Drug-Related AEs
                         A: Drug X    B: Placebo   C: Combination   A: Drug X    B: Placebo   C: Combination
                            N=XXX         N=XXX           N=XXX           N=XXX         N=XXX           N=XXX
————————————————————————————————————————————————————————————————————————————————————————————————————————————
Subjects with >=1 TEAE   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)     xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
SOC1                     xx (xx.x%)   xx (xx.x%)     xx (xx.x%)     xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  PT1                    xx (xx.x%)   xx (xx.x%)     xx (xx.x%)     xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Drug-related AEs are those with investigator-assessed relationship of "Related".
```

Programming Notes:
1. Source dataset: ADAE with alt_counts_df = ADSL.
2. Drug-related AEs are those with AEREL == "RELATED" (or equivalent).
3. Sort SOC and PT by decreasing incidence in the "All AEs" columns.

---

## Shell T13: Vital Signs Summary by Visit

```
TSFVS-COLS: Vital Signs Summary Statistics in Columns; Safety Analysis Set

——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Parameter
  Treatment Group                              Visit Values                            Change from Baseline
  Visit                           N     Mean    SD    Median   Min    Max     N     Mean    SD    Median   Min    Max
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Systolic Blood Pressure (mmHg)
  A: Drug X
    Baseline                     xx.x   xx.x   xx.x    xx.x    xx.x   xx.x   xx.x   xx.x   xx.x    xx.x    xx.x   xx.x
    Week 2                       xx.x   xx.x   xx.x    xx.x    xx.x   xx.x   xx.x   xx.x   xx.x    xx.x    xx.x   xx.x
  B: Placebo
    Baseline                     xx.x   xx.x   xx.x    xx.x    xx.x   xx.x   xx.x   xx.x   xx.x    xx.x    xx.x   xx.x
    Week 2                       xx.x   xx.x   xx.x    xx.x    xx.x   xx.x   xx.x   xx.x   xx.x    xx.x    xx.x   xx.x
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Change from Baseline columns are blank at Baseline.
```

Programming Notes:
1. Source dataset: ADVS.
2. Change from Baseline columns are blank at Baseline.
3. Use parameter-specific decimal precision.

---

## Shell T14: Change from Baseline in Primary Endpoint by Visit

```
TSFEFF-ANCOVA: Change from Baseline in Primary Endpoint by Visit; Full Analysis Set

———————————————————————————————————————————————————————————————————————————————————————
                             A: Drug X     C: Combination    B: Placebo     All Active
                                N=XXX            N=XXX            N=XXX           N=XXX
———————————————————————————————————————————————————————————————————————————————————————
Week 2
  N                              xx              xx              xx             xx
  Mean (SD)                 xx.x (xx.x)     xx.x (xx.x)     xx.x (xx.x)    xx.x (xx.x)
  LS Mean (SE)              xx.x (xx.x)     xx.x (xx.x)     xx.x (xx.x)    xx.x (xx.x)
  LS Mean Diff vs Placebo       xx.x            xx.x            xx.x           xx.x
    95% CI                  (xx.x, xx.x)    (xx.x, xx.x)    (xx.x, xx.x)   (xx.x, xx.x)
    p-value                    x.xxxx          x.xxxx          x.xxxx         x.xxxx
———————————————————————————————————————————————————————————————————————————————————————

Note: "All Active" combines A: Drug X and C: Combination arms.
LS Means from ANCOVA model with treatment and baseline value as covariates.
```

Programming Notes:
1. Source dataset: ADLB (or similar efficacy dataset) with alt_counts_df = ADSL.
2. "All Active" LS Mean and difference should be derived via linear contrasts from the full ANCOVA model (not by pooling data).
3. LS Means from ANCOVA model with treatment and baseline value as covariates.
4. LS Mean Diff, CI, and p-value are blank in the Placebo (reference) column.

---

## Shell T15: Cumulative/Period AE Columns

```
TSFAE-CUMUL: TEAEs by Treatment Period; Safety Analysis Set

————————————————————————————————————————————————————————————————————————————————————————————
                         Induction (Wk 0-12)   Maintenance (Wk 12-52)   Cumulative (Wk 0-52)
                                 N=XXX                    N=XXX                     N=XXX
————————————————————————————————————————————————————————————————————————————————————————————
Subjects with >=1 TEAE       xx (xx.x%)              xx (xx.x%)              xx (xx.x%)
SOC1                         xx (xx.x%)              xx (xx.x%)              xx (xx.x%)
  PT1                        xx (xx.x%)              xx (xx.x%)              xx (xx.x%)
————————————————————————————————————————————————————————————————————————————————————————————

Note: Subjects may be counted in both individual period columns but are counted once in Cumulative.
```

Programming Notes:
1. Source dataset: ADAE with alt_counts_df = ADSL.
2. Subjects may appear in both individual period columns but are counted only once in Cumulative.
3. Individual period columns must prevent double-counting subjects who have multiple events within the same period.

---

## Shell T16: Immunogenicity by ADA Status and Peak Titer with Pairwise Differences

```
TIRADA-DIFF: [Continuous Response] by ADA Status; Full Analysis Set

—————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                               Peak Titers for Subjects Positive for
                                                   Treatment-emergent Antibodies          Difference (95% CI)
              Negative for   Positive for
               Treatment-     Treatment-                                                       Positive vs
                emergent       emergent                                                        Negative
               Antibodies     Antibodies         <10         10 to <100       >=100
Time Point        N=XXX           N=XXX           N=XXX            N=XXX           N=XXX              N=XXX
—————————————————————————————————————————————————————————————————————————————————————————————————————————————
Week 4
  N                xx             xx             xx              xx             xx
  Mean (SD)   xx.x (xx.x)    xx.x (xx.x)     xx.x (xx.x)    xx.x (xx.x)    xx.x (xx.x)    xx.x (xx.x - xx.x)
  Median          xx.x           xx.x           xx.x            xx.x           xx.x
Week 12
  N                xx             xx             xx              xx             xx
  Mean (SD)   xx.x (xx.x)    xx.x (xx.x)     xx.x (xx.x)    xx.x (xx.x)    xx.x (xx.x)    xx.x (xx.x - xx.x)
  Median          xx.x           xx.x           xx.x            xx.x           xx.x
—————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Difference column shows mean difference (95% CI) of Positive vs Negative for Treatment-emergent Antibodies.
```

Programming Notes:
1. Source dataset: Immunogenicity analysis dataset with alt_counts_df = ADSL (or subject-level ADA dataset).
2. Difference column shows mean difference (95% CI) between Positive and Negative groups.

---

## Shell T17: Biomarker Response by Exposure Quartile with Subgroup Columns

```
TSFBMK01: Biomarker Response by Exposure Quartile; Pharmacokinetic Analysis Set

———————————————————————————————————————————————————————————————————————————————————————————————————————
                                                          Exposure Quartile (AUC)
                                       Total       Q1 (<=xx)    Q2 (xx-xx)    Q3 (xx-xx)     Q4 (>xx)
                                       N=XXX          N=XXX          N=XXX          N=XXX          N=XXX
———————————————————————————————————————————————————————————————————————————————————————————————————————
Biomarker A (pg/mL)
  Baseline
    N                                   xx            xx            xx            xx            xx
    Mean (SD)                       xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)
    Median                             xx.x          xx.x          xx.x          xx.x          xx.x
  Week 12
    N                                   xx            xx            xx            xx            xx
    Mean (SD)                       xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)
    Median                             xx.x          xx.x          xx.x          xx.x          xx.x
  Change from Baseline at Week 12
    N                                   xx            xx            xx            xx            xx
    Mean (SD)                       xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)   xx.x (xx.x)
    Median                             xx.x          xx.x          xx.x          xx.x          xx.x
———————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Exposure quartiles are based on individual AUC values.
```

Programming Notes:
1. Source dataset: Biomarker analysis dataset merged with PK exposure quartiles.
2. No difference or comparison columns.

---

## Shell T18: TEAEs by Treatment Group and Severity

```
TSFAE-3SPAN: TEAEs by Treatment Group and Severity; Safety Analysis Set

—————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                     Active Treatment                                   Risk Difference (%) (95% CI)
                A: Drug X    C: Combination   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo
                   N=XXX           N=XXX           N=XXX               N=XXX                         N=XXX
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Subjects with   xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  >=1 TEAE
SOC1            xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  PT1           xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
```

This is the standard TSFAE02 shell. The key column structure observation is that "B: Placebo" sits under a **blank (whitespace) spanning label** at the same tier as "Active Treatment". The blank span is NOT the absence of a span — it is a real span node with an empty-string label occupying the same hierarchical tier as "Active Treatment" and "Risk Difference (%) (95% CI)".

Visual alignment proof:
- "Active Treatment" is centered over columns 1–2 (A: Drug X, C: Combination).
- There is NO text above "B: Placebo" (column 3), but it occupies the same vertical tier.
- "Risk Difference (%) (95% CI)" is centered over columns 4–5.

Programming Notes:
1. B: Placebo is the reference group for risk difference calculations.
2. Risk differences use the Wald method.

---

## Shell T19: TEAEs Summary with Treatment Group Total

```
TSFAE-TOTUNDER: TEAEs Summary; Safety Analysis Set

—————————————————————————————————————————————————————————————————————
                                   Treatment Group
                  Total      A: Drug X    B: Placebo   C: Combination
                   N=XXX         N=XXX         N=XXX           N=XXX
—————————————————————————————————————————————————————————————————————
Subjects with   xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  >=1 TEAE
SOC1            xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  PT1           xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
```

Note the spanning label "Treatment Group" is centered across ALL four columns including Total. This means Total is a sibling of the individual arms INSIDE the spanning label, not outside it.

Programming Notes:
1. Percentages based on column N.

---

## Shell T20: TEAEs Summary with Overall Total

```
TSFAE-TOTOUT: TEAEs Summary; Safety Analysis Set

—————————————————————————————————————————————————————————————————————
                                         Treatment Group
                  Total      A: Drug X    B: Placebo   C: Combination
                   N=XXX         N=XXX         N=XXX           N=XXX
—————————————————————————————————————————————————————————————————————
Subjects with   xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  >=1 TEAE
SOC1            xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  PT1           xx (xx.x%)   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
```

Note the spanning label "Treatment Group" is centered over columns 2–4 (A, B, C) only. Total (column 1) is NOT under the span — it sits to the left, outside the spanning label's extent.

Programming Notes:
1. Percentages based on column N.

---

## Shell T21: Overall Adverse Event Summary

```
TSFAE-DUAL: Overall AE Summary; Safety Analysis Set

——————————————————————————————————————————————————————————————————————————————
                                      A: Drug X    B: Placebo   C: Combination
                                         N=XXX         N=XXX           N=XXX
——————————————————————————————————————————————————————————————————————————————
Number of subjects with >=1 TEAE      xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Total number of events                    xx           xx             xx
Number of subjects with >=1 SAE       xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Total number of SAEs                      xx           xx             xx
Number of subjects with >=1 TEAE      xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  leading to discontinuation
Number of subjects with >=1           xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  drug-related TEAE
Total number of drug-related events       xx           xx             xx
——————————————————————————————————————————————————————————————————————————————

Note: Subjects are counted once per category. Events are total occurrences.
```

Programming Notes:
1. Source dataset: ADAE with alt_counts_df = ADSL.
2. Percentages are based on column N from ADSL.

---

## Shell T22: AE Summary by Incidence, Maximum Severity, and SOC/PT

```
TSFAE-MIXED: AE Summary with Heterogeneous Sections; Safety Analysis Set

——————————————————————————————————————————————————————————————————————
                              A: Drug X    B: Placebo   C: Combination
                                 N=XXX         N=XXX           N=XXX
——————————————————————————————————————————————————————————————————————
Subjects with >=1 TEAE        xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
TEAEs by Maximum Severity
  Mild                        xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  Moderate                    xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  Severe                      xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
TEAEs by System Organ Class
  SOC1                        xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    PT1                       xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    PT2                       xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  SOC2                        xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    PT1                       xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    PT2                       xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
——————————————————————————————————————————————————————————————————————

Note: Severity section counts each subject once at their maximum severity.
SOC/PT section counts each subject once per SOC and PT.
```

Programming Notes:
1. Source dataset: ADAE with alt_counts_df = ADSL.
2. Sort SOC and PT in Section 3 by decreasing incidence.

---

## Shell T23: AE Summary by Category

```
TSFAE-FLAT: AE Summary by Category; Safety Analysis Set

————————————————————————————————————————————————————————————————————————
                                A: Drug X    B: Placebo   C: Combination
                                   N=XXX         N=XXX           N=XXX
————————————————————————————————————————————————————————————————————————
Any TEAE                        xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Any serious TEAE                xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Any TEAE leading to death       xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Any TEAE leading to             xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  discontinuation
Any drug-related TEAE           xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Any serious drug-related TEAE   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
————————————————————————————————————————————————————————————————————————

Note: Subjects are counted once per category.
```

Programming Notes:
3. Percentages based on subjects randomized to each arm.

---

## Shell T24: Efficacy Summary by Subgroup

```
TSFEFF-SUBGRP: Efficacy Summary by Subgroup; Full Analysis Set

———————————————————————————————————————————————————————————————————
                                A: Drug X            B: Placebo
                                   N=XXX                 N=XXX
———————————————————————————————————————————————————————————————————
Stratum A
  Overall
    N                               xx                   xx
    Mean (SD)                  xx.x (xx.x)          xx.x (xx.x)
    LS Mean Diff (95% CI)   xx.x (xx.x - xx.x)   xx.x (xx.x - xx.x)
    p-value                       x.xxxx               x.xxxx
  By Sex
    Female
      N                             xx                   xx
      Mean (SD)                xx.x (xx.x)          xx.x (xx.x)
    Male
      N                             xx                   xx
      Mean (SD)                xx.x (xx.x)          xx.x (xx.x)
  By Age Group
    <65
      N                             xx                   xx
      Mean (SD)                xx.x (xx.x)          xx.x (xx.x)
    >=65
      N                             xx                   xx
      Mean (SD)                xx.x (xx.x)          xx.x (xx.x)
Stratum B
  Overall
    N                               xx                   xx
    Mean (SD)                  xx.x (xx.x)          xx.x (xx.x)
    LS Mean Diff (95% CI)   xx.x (xx.x - xx.x)   xx.x (xx.x - xx.x)
    p-value                       x.xxxx               x.xxxx
  By Sex
    Female
      N                             xx                   xx
      Mean (SD)                xx.x (xx.x)          xx.x (xx.x)
    Male
      N                             xx                   xx
      Mean (SD)                xx.x (xx.x)          xx.x (xx.x)
  By Age Group
    <65
      N                             xx                   xx
      Mean (SD)                xx.x (xx.x)          xx.x (xx.x)
    >=65
      N                             xx                   xx
      Mean (SD)                xx.x (xx.x)          xx.x (xx.x)
```

Programming Notes:
1. Source dataset: Efficacy analysis dataset

---

## Shell T25: Analysis Population Summary

```
TSFPOP: Analysis Population Summary; Safety Analysis Set

———————————————————————————————————————————————————————
               A: Drug X    B: Placebo   C: Combination
                  N=XXX         N=XXX           N=XXX
———————————————————————————————————————————————————————
Randomized     xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Treated        xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Completed      xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
Discontinued   xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
———————————————————————————————————————————————————————

Note: Percentages are based on the number of randomized subjects.
```

Programming Notes:
1. Percentages are based on the number of randomized subjects.

---

## Shell T26: Drug-Related TEAEs by SOC and PT

```
TSFAE-RELFLAG: Drug-Related TEAEs by SOC and PT; Safety Analysis Set

——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                     Active Study Agent                                  Risk Difference (%) (95% CI)
                                 A: Drug X    C: Combination   B: Placebo   A: Drug X vs B: Placebo   C: Combination vs B: Placebo
                                    N=XXX           N=XXX           N=XXX               N=XXX                         N=XXX
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Subjects with >=1 drug-related   xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  TEAE
SOC1                             xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
  PT1                            xx (xx.x%)     xx (xx.x%)     xx (xx.x%)      xx.x (xx.x - xx.x)             xx.x (xx.x - xx.x)
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Only drug-related TEAEs (investigator-assessed) are included.
Percentages are based on the number of subjects in each treatment group (not the number with drug-related AEs).
```

Programming Notes:
1. Source dataset: ADAE filtered to drug-related AEs only (AEREL == "RELATED"), with alt_counts_df = ADSL (unfiltered, full safety population).
2. Percentages are out of total arm patient count.

---

## Shell T27: TEAEs by SOC, PT, and Severity

```
TSFAE-RAGGED: TEAEs by SOC, PT, and Severity; Safety Analysis Set

—————————————————————————————————————————————————————————————
System Organ Class
  Preferred Term     A: Drug X    B: Placebo   C: Combination
    Severity            N=XXX         N=XXX           N=XXX
—————————————————————————————————————————————————————————————
SOC1                 xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  PT1                xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Mild             xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  PT2                xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Moderate         xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
SOC2                 xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
  PT1                xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
    Mild             xx (xx.x%)   xx (xx.x%)     xx (xx.x%)
—————————————————————————————————————————————————————————————

Note: Severity levels shown only when observed for that PT.
Each subject counted once at maximum severity per PT.
```

Programming Notes:
1. Each subject counted once at their maximum severity per PT.

---

## Shell T28: TEAEs by SOC and PT

```
TSFAE-MULTICFUN: TEAEs by SOC (with Dual Subject/Event Summaries) and PT; Safety Analysis Set

———————————————————————————————————————————————————————————————————————————————————————————————————
System Organ Class                                                                                 
  SOC Summary                                Xanomeline Low Dose   Xanomeline High Dose    Placebo 
    Preferred Term                                  N=xx                   N=xx             N=xx   
———————————————————————————————————————————————————————————————————————————————————————————————————
SOC1                                                                                               
  Subjects with >=1 TEAE                         xx (xx.x%)             xx (xx.x%)       xx (xx.x%)
  Total number of TEAEs                              xx                     xx               xx    
      PT1                                        xx (xx.x%)             xx (xx.x%)       xx (xx.x%)
SOC2                                                                                               
  Subjects with >=1 TEAE                         xx (xx.x%)             xx (xx.x%)       xx (xx.x%)
  Total number of TEAEs                              xx                     xx               xx    
      PT1                                        xx (xx.x%)             xx (xx.x%)       xx (xx.x%)
———————————————————————————————————————————————————————————————————————————————————————————————————

Note: Each SOC displays unique subjects with >=1 TEAE and total TEAE occurrences, followed by preferred terms (unique subjects).
```

Programming Notes:
1. System Organ Classes of interest defined by study lead.
2. Percentages are out of safety-evaluated patients within each arm.
2. Both SOCs and Preferred Terms within each SOC are sorted in descending order of unique subject incidence in the first treatment column, with alphabetical tie-breaking.

---

## Shell T29: Laboratory Summary by Visit, Sex Subgroup, and Post-Baseline Worst Status

```
TSFLAB-ATSIB2D: Laboratory Summary by Visit, Sex Subgroup, and Post-Baseline Shift; Safety Analysis Set

——————————————————————————————————————————————————————————————————————————————————————————————
                                      Xanomeline Low Dose   Xanomeline High Dose     Placebo  
                                             N=xx                   N=xx              N=xx    
——————————————————————————————————————————————————————————————————————————————————————————————
PARAM1                                                                
  Baseline                                                                                    
    N                                         xx                     xx                xx     
    Mean (SD)                            xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
    By Sex                                                                                    
      Female                                                                                  
        Mean (SD)                        xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
      Male                                                                                    
        Mean (SD)                        xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
  Cycle 02                                                                                    
    N                                         xx                     xx                xx     
    Mean (SD)                            xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
    By Sex                                                                                    
      Female                                                                                  
        Mean (SD)                        xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
      Male                                                                                    
        Mean (SD)                        xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
  Post-Baseline Worst Status, n (%)                                                           
    LOW                                   xx (xx.x%)            xx (xx.x%)        xx (xx.x%)  
    NORMAL                                xx (xx.x%)            xx (xx.x%)        xx (xx.x%)  
    HIGH                                  xx (xx.x%)            xx (xx.x%)        xx (xx.x%)  
PARAM2                                                             
  Baseline                                                                                    
    N                                         xx                     xx                xx     
    Mean (SD)                            xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
    By Sex                                                                                    
      Female                                                                                  
        Mean (SD)                        xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
      Male                                                                                    
        Mean (SD)                        xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
  Cycle 02                                                                                    
    N                                         xx                     xx                xx     
    Mean (SD)                            xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
    By Sex                                                                                    
      Female                                                                                  
        Mean (SD)                        xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
      Male                                                                                    
        Mean (SD)                        xx.x (xx.x)            xx.x (xx.x)       xx.x (xx.x) 
  Post-Baseline Worst Status, n (%)                                                           
    LOW                                   xx (xx.x%)            xx (xx.x%)        xx (xx.x%)  
    NORMAL                                xx (xx.x%)            xx (xx.x%)        xx (xx.x%)  
    HIGH                                  xx (xx.x%)            xx (xx.x%)        xx (xx.x%)  
——————————————————————————————————————————————————————————————————————————————————————————————

Note: Each visit displays overall summary statistics followed by sex-specific Mean (SD).
Post-Baseline Worst Reference Range Status summarizes each subject once per parameter below all visits.
```

Programming Notes:
1. Patients evaluated for safety.
2. For Post-Baseline Worst Status, each subject is summarized once per parameter at their worst post-baseline assessment across all post-baseline records (ordered `LOW < NORMAL < HIGH`), with percentages based on the safety population count in each treatment arm.

---

## Shell T30: TEAEs by Treatment, Sex, and Age Group Banners

```
TSFAE-BANNER: TEAEs by Parallel Treatment, Sex, and Age Group Banners; Safety Analysis Set

———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                                                             Treatment Group                            Sex Subgroup                 Age Group (Years)         
                                          All Subjects   Xanomeline Low Dose   Xanomeline High Dose    Placebo       Female        Male         <65      65 to <75      >=75   
                                              N=xx              N=xx                   N=xx              N=xx         N=xx         N=xx         N=xx        N=xx         N=xx  
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Subjects with >=1 TEAE in Selected SOCs    xx (xx.x%)        xx (xx.x%)             xx (xx.x%)        xx (xx.x%)   xx (xx.x%)   xx (xx.x%)  xx (xx.x%)  xx (xx.x%)  xx (xx.x%)
SOC1                                       xx (xx.x%)        xx (xx.x%)             xx (xx.x%)        xx (xx.x%)   xx (xx.x%)   xx (xx.x%)  xx (xx.x%)  xx (xx.x%)  xx (xx.x%)
  PT1                                      xx (xx.x%)        xx (xx.x%)             xx (xx.x%)        xx (xx.x%)   xx (xx.x%)   xx (xx.x%)  xx (xx.x%)  xx (xx.x%)  xx (xx.x%)
SOC2                                       xx (xx.x%)        xx (xx.x%)             xx (xx.x%)        xx (xx.x%)   xx (xx.x%)   xx (xx.x%)  xx (xx.x%)  xx (xx.x%)  xx (xx.x%)
  PT1                                      xx (xx.x%)        xx (xx.x%)             xx (xx.x%)        xx (xx.x%)   xx (xx.x%)   xx (xx.x%)  xx (xx.x%)  xx (xx.x%)  xx (xx.x%)
———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Columns represent parallel non-nested banners over the same Safety Analysis Set.
Percentages are based on column N within each banner facet.
```

Programming Notes:
1. Percents are among all patients evaluated for safety.
2. Treatment emergent aes defined by the study lead.
3. SOCs and Preferred Terms are sorted in descending order of subject count in the `All Subjects` column.

---

## Shell T31: Vital Signs Across Longitudinal Visits

```
TSFVS-LONGCB: Vital Signs Across Longitudinal Visit Columns with Conditional CFB; Safety Analysis Set

————————————————————————————————————————————————————————————————————————————————————————————————————————
Parameter                           Baseline              Cycle 02                     Cycle 04         
  Treatment Group                   Observed       Observed         CFB         Observed         CFB    
————————————————————————————————————————————————————————————————————————————————————————————————————————
PARAM1                                                                         
  Xanomeline Low Dose                                                                                   
    n                                  xx             xx            xx             xx            xx     
    Min, Max                       xx.x, xx.x     xx.x, xx.x    xx.x, xx.x     xx.x, xx.x    xx.x, xx.x 
        Mean (SD)                 xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x) 
        Median                        xx.x           xx.x          xx.x           xx.x          xx.x    
  Xanomeline High Dose                                                                                  
    n                                  xx             xx            xx             xx            xx     
    Min, Max                       xx.x, xx.x     xx.x, xx.x    xx.x, xx.x     xx.x, xx.x    xx.x, xx.x 
        Mean (SD)                 xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x) 
        Median                        xx.x           xx.x          xx.x           xx.x          xx.x    
  Placebo                                                                                               
    n                                  xx             xx            xx             xx            xx     
    Min, Max                       xx.x, xx.x     xx.x, xx.x    xx.x, xx.x     xx.x, xx.x    xx.x, xx.x 
        Mean (SD)                 xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x) 
        Median                        xx.x           xx.x          xx.x           xx.x          xx.x    
PARAM2                                                                        
  Xanomeline Low Dose                                                                                   
    n                                  xx             xx            xx             xx            xx     
    Min, Max                       xx.x, xx.x     xx.x, xx.x    xx.x, xx.x     xx.x, xx.x    xx.x, xx.x 
        Mean (SD)                 xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x) 
        Median                        xx.x           xx.x          xx.x           xx.x          xx.x    
  Xanomeline High Dose                                                                                  
    n                                  xx             xx            xx             xx            xx     
    Min, Max                       xx.x, xx.x     xx.x, xx.x    xx.x, xx.x     xx.x, xx.x    xx.x, xx.x 
        Mean (SD)                 xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x) 
        Median                        xx.x           xx.x          xx.x           xx.x          xx.x    
  Placebo                                                                                               
    n                                  xx             xx            xx             xx            xx     
    Min, Max                       xx.x, xx.x     xx.x, xx.x    xx.x, xx.x     xx.x, xx.x    xx.x, xx.x 
        Mean (SD)                 xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x)    xx.x (xx.x)   xx.x (xx.x) 
        Median                        xx.x           xx.x          xx.x           xx.x          xx.x    
————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Baseline displays Observed values only; post-baseline visits display both Observed and Change from Baseline (CFB).
Each treatment group header summarizes n and Min, Max via a multi-row group summary.
```

Programming Notes:
1. Safety population
---

## Shell T32: Subgroup TEAEs with Severity and Discontinuation SOCs

```
TSFAE-SUBGRP-MIXED: Subgroup TEAEs with Dual-Row Subgroup Header, Severity, and Discontinuation SOCs; Safety Analysis Set

—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
                                                                                        Active Study Agent                                                 Risk Difference (%) (95% CI)                  
                                                                            Xanomeline Low Dose   Xanomeline High Dose      Placebo      Xanomeline Low Dose vs Placebo   Xanomeline High Dose vs Placebo
                                                                                   N=xx                   N=xx               N=xx                     N=xx                             N=xx              
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
Female                                                                                                                                                                                                   
  Subjects in subgroup, n                                                            xx                      xx                  xx                                                                        
  Subjects with >=1 TEAE, n (%)                                                xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
    TEAEs by Maximum Severity                                                                                                                                                                            
      Mild                                                                     xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
      Moderate                                                                 xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
      Severe                                                                   xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
    TEAEs Leading to Discontinuation by SOC                                                                                                                                                              
      SOC1                                                                     xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
Male                                                                                                                                                                                                     
  Subjects in subgroup, n                                                            xx                      xx                  xx                                                                        
  Subjects with >=1 TEAE, n (%)                                                xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
    TEAEs by Maximum Severity                                                                                                                                                                            
      Mild                                                                     xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
      Moderate                                                                 xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
      Severe                                                                   xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
    TEAEs Leading to Discontinuation by SOC                                                                                                                                                              
      SOC1                                                                     xx / xx (xx.x%)        xx / xx (xx.x%)      xx / xx (xx.x%)       xx.x (xx.x - xx.x)                xx.x (xx.x - xx.x)      
—————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

Note: Subgroup header displays sex-specific N (blank in Risk Difference columns) and overall TEAE incidence.
Severity and Discontinuation SOC sections use sex-specific denominators within each treatment group.
```

Programming Notes:
1. Percentages for TEAE incidence, maximum severity, and discontinuation by SOC use sex-specific safety population denominators within each treatment arm.
2. Treatment-emergent AEs and AEs leading to study drug discontinuation are defined by the study lead.
3. Risk differences and 95% confidence intervals use the Wald method.
4. For TEAEs by Maximum Severity, each subject is counted once at their maximum severity within each sex subgroup (`Mild < Moderate < Severe`), with all three severity levels displayed.
5. Observed SOCs under TEAEs Leading to Discontinuation by SOC are displayed in alphabetical order within each sex subgroup.
6. Count-percent cells with zero counts should be displayed as "0".

