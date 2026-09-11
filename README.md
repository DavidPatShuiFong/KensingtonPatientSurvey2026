# Kensington Patient Survey 2026

Repo: https://github.com/DavidPatShuiFong/KensingtonPatientSurvey2026

Analysis of the 2026 cohealth Kensington GP clinic patient feedback survey. The survey combines
clinic-specific questions (reception, privacy, care coordination), the validated **P3CEQ**
(Person Centred Coordinated Care Experiences Questionnaire), and respondent demographics.

## Report

`KensingtonPatientSurvey2026.qmd` is a self-contained Quarto report that:

- recodes the embedded P3CEQ items onto the instrument's official 0–3 scoring scale and computes
  the total score plus the person-centred (PC) and care-coordination (CC) subscales
- charts clinic-specific ratings, long-term conditions, and respondent demographics
- includes the free-text patient comments

Render it to HTML (default) or PDF:

```sh
quarto render                     # KensingtonPatientSurvey2026.html, self-contained
quarto render --to pdf            # KensingtonPatientSurvey2026.pdf
```

The HTML version has fold/show-code toggles on every code chunk; the PDF hides code by default
(there's no fold interaction in print) so it reads as a plain report. PDF rendering needs a LaTeX
install — [TinyTeX](https://quarto.org/docs/output-formats/pdf-engine.html) is the easiest way
(`quarto install tinytex`).

## Data

Two source files the report reads are **not** in this repo, since they contain real patient
responses and a copyrighted external instrument respectively:

- `Patient feedback questionnaire — cohealth Kensington GP clinic 2026_WIDE.csv` — the raw
  SurveyCTO response export
- `P3C-PEQ with scoring.pdf` — the official P3CEQ instrument and scoring guide (Sugavanam, Byng &
  Lloyd, Plymouth University)

Both need to be placed in the project root (alongside the `.qmd`) before rendering. The SurveyCTO
form definition (`kensington_p3ceq_2026_surveycto.xlsx`, question/choice structure only — no
patient data) is included in the repo.

See `CLAUDE.md` for the survey structure, the P3CEQ recode mapping, and other implementation notes.
