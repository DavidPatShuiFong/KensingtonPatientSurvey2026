# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

`KensingtonPatientSurvey2026.qmd` is the analysis report: it loads the CSV export and the xlsx
form definition, recodes the P3CEQ items onto the official [0–3] scale (see below), computes the
P3CEQ total and PC/CC subscales, and produces charts/tables for the clinic-specific items,
long-term conditions, and respondent demographics, plus foldable free-text comment sections. It
renders standalone (`quarto render`, self-contained `embed-resources: true` HTML). When extending
it, reuse the existing `choice_labels()`/`item_label()` helpers and the `recode_*()` functions
rather than re-deriving label or scoring logic.

## Commands

- Render the report: `quarto render` (HTML, default) or `quarto render --to pdf` (or the Render
  button in RStudio/Positron). Format options for both are in the `.qmd`'s YAML front matter.
  PDF needs a LaTeX install (TinyTeX is present in this environment: `pdflatex`/`xelatex`/
  `lualatex`). The `pdf` format sets `echo: false` (code is hidden — there's no fold UI in print,
  unlike the HTML format's `code-fold: true`) and figures use `dev = "cairo_pdf"` when
  `knitr::is_latex_output()` so Unicode characters (en dashes, etc.) in chart text render
  correctly — do not add figure-generating code that assumes the base R `pdf()` device.
- There is no package manager lockfile (no `renv.lock`) and no test suite — this is a single-document
  Quarto analysis project, not an R package.
- Read the Excel form definition with `readxl::read_excel(path, sheet = "survey"|"choices"|"settings")`
  — the `readxl` package is available in this environment.

## Data files

Two of the files below (the response CSV and the scoring PDF) are gitignored — they contain real
patient responses and a copyrighted external instrument respectively, so they stay local-only and
are never committed. They're expected to exist on disk for the report to run; don't re-add them to
git, and if `quarto render` fails because one is missing, that means it needs to be fetched from
wherever it's normally sourced (SurveyCTO export / the P3CEQ authors), not regenerated from git.

- `Patient feedback questionnaire — cohealth Kensington GP clinic 2026_WIDE.csv` — the raw SurveyCTO
  export, wide format, one row per respondent (95 rows as of the last export, collected 5–10 Sep 2026).
  Select-multiple questions are exploded into `q{n}_{choice_value}` binary indicator columns (e.g.
  `q9_1`...`q9_13` for Q9, `q25_1`...`q25_8` for Q25, `q28_1`...`q28_4` for Q28) alongside an `_other`
  free-text column. Metadata columns (`SubmissionDate`, `starttime`, `endtime`, `duration`,
  `device_info`, `instanceID`, `KEY`, etc.) are SurveyCTO plumbing, not survey answers.
- `kensington_p3ceq_2026_surveycto.xlsx` — the SurveyCTO form definition (XLSForm), with sheets
  `settings`, `survey` (question list: `type`, `name`, `label`, skip logic in `relevance`) and
  `choices` (option lists referenced by `select_one`/`select_multiple` types, keyed by `list_name`).
  This is the authoritative source for what each `q*` column in the CSV means and how its integer
  codes map to labels — read it before writing any code that interprets raw response values.
- `P3C-PEQ with scoring.pdf` — the official P3CEQ (Person Centred Coordinated Care Experiences
  Questionnaire) instrument and scoring methodology (Sugavanam, Byng & Lloyd, Plymouth University).
  This is the validated instrument this survey is partly built on; see below for how it maps onto
  the CSV columns.

## Survey structure

The instrument mixes three blocks under one SurveyCTO form:

1. **Clinic-specific items (Q1–Q3, `q1a`–`q3d` + `_comment` free text)**: reception experience,
   privacy, and clinician coordination with other services. These are cohealth Kensington's own
   items, not part of P3CEQ. Scale is `rate7`: 1=Poor, 2=Fair, 3=Good, 4=Very good, 5=Excellent,
   6=N/A, 7=Don't know — codes 6 and 7 must be excluded (treated as missing) from any mean/summary
   score, not treated as a "0" or bottom-of-scale response.
2. **P3CEQ items (Q4–Q20)**: this section reproduces the validated P3CEQ questionnaire (see the
   scoring PDF), but the SurveyCTO version re-wrote several items into different response-option
   wordings/orderings than the original instrument. **Raw SurveyCTO codes are not the same as
   P3CEQ's [0]–[3] scoring codes** — they must be recoded before computing official P3CEQ scores.
   The implemented mapping (see the `recode_*()` functions and the `p3ceq` pipeline in
   `KensingtonPatientSurvey2026.qmd`, which is the source of truth — this table is a summary of it):

   | CSV col | P3CEQ Q | Choice list | Recode to P3CEQ [0–3] |
   |---|---|---|---|
   | `q4` | Q1 (discussed what's important to you) | `yesdef` (1=Yes definitely…5=Not sure) | 1→3, 2→2, 3→1, 4→0, 5→0 |
   | `q5` | Q2 (involved in decisions) | `yesdef` | same as above |
   | `q6` | Q3 (whole person, not just condition) | `yesdef` | same as above |
   | `q7` | Q4 (repeated info — reverse scored in original) | `yesdef` | same as above |
   | `q8` | Q5 (care joined up) | `yesdef` | same as above |
   | `q10` | Q6 (single coordinating professional) | `q10opts` (1=Yes,2=No,3=only one service,4=Not sure) | 1→3, 2→0, 3→3, 4→0 |
   | `q11` | Q6 follow-up (same professional) | `q11opts` | supplementary detail, not separately scored in P3CEQ |
   | `q12` | — | `q12opts` | who the coordinating professional is; supplementary, not scored |
   | `q13` | Q7a (has a care plan) | `q13opts` (1=Yes,2=No,3=Not sure) | 1→3, 2→0, 3→0; No/Not sure skip 7b–7d |
   | `q14` | Q7b (care plan available) | `q14opts` (same 3 options as `q13opts`) | 1→3, 2→0, 3→0 |
   | `q15` | Q7c (care plan useful) | `extent5` (1=Not at all…4=Completely,5=Not sure) | 1→0, 2→1, 3→2, 4→3, 5→0 |
   | `q16` | Q7d (professionals follow same plan) | `extent5` | same as above |
   | `q17` | Q8 (enough support) | `q17opts` (1=do not need support,2=no support,3=sometimes enough,4=often enough,5=always enough,6=Not relevant,7=Not sure) | 1→3, 2→0, 3→1, 4→2, 5→3, 6→0, 7→0 |
   | `q18` | Q9 (useful information) | `q18opts` (1=no info,2=sometimes enough,3=often enough,4=always enough,5=too much,6=Not relevant,7=Not sure) | 1→0, 2→1, 3→2, 4→3, 5→2, 6→0, 7→0 |
   | `q19` | Q10 (confidence managing own health) | `q19opts` (1=Very confident,2=Confident,3=Somewhat confident,4=Not confident at all,5=Does not apply) — **note: order is reversed relative to `q4`-style scales** | 1→3, 2→2, 3→1, 4→0, 5→0 |
   | `q20` | Q11b (family/carer involvement, optional) | `q20opts` | optional item, excluded from the total score per the PDF |

   The original P3CEQ scores every "Not relevant"/"Don't know"/catch-all option as **0**, not
   missing — the recoding above follows that convention throughout (blank/skipped cells, from
   SurveyCTO's own skip logic on `q11`, `q12`, `q14`–`q16`, remain `NA`, but an explicit "Not sure"
   answer scores 0). The `q10` value 3 ("I do not receive care from more than one service") and the
   `q17`/`q18`/`q19` catch-all options have no exact equivalent in the original instrument; those
   mappings are this project's judgment call (documented inline in the `.qmd`), not something the
   scoring PDF states directly — revisit if cohealth or the P3CEQ authors give more specific guidance.

   Q7's P3CEQ score is the **average** of q13/q14/q15/q16 (0 outright if q13/7a = 0, since 7b–7d
   are then structurally skipped). The **total P3CEQ score** is the sum of Q1–Q10 (i.e. recoded
   `q4,q5,q6,q7,q8,q10,mean(q13..q16),q17,q18,q19`), range 0–30, higher = better. Two optional
   subscales, computed from the *recoded* [0–3] values:
   - Person-centred (PC) = Q1+Q2+Q3+Q4+Q5+Q8+Q9+Q10 → `q4+q5+q6+q7+q8+q17+q18+q19`
   - Care coordination (CC) = Q5+Q6+Q7+Q8+Q9 → `q8+q10+mean(q13..q16)+q17+q18`
3. **Demographics and follow-up (Q21–Q31)**: free-text improvement suggestions (`q21`), long-term
   conditions checklist (`q9`, select-multiple against the `conditions` list), and standard
   demographics (gender, Aboriginal/Torres Strait Islander status, languages spoken, age band,
   tenure at the practice, concession cards, visit frequency, education, self/carer visit).

When writing analysis code, load the `survey`/`choices` sheets from the xlsx to programmatically
build label lookups rather than hardcoding option text, since the xlsx is the source of truth and
any future form revisions will change the export's column set.
