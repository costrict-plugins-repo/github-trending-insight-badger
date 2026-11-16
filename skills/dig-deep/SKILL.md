---
name: dig-deep
description: >
  Perform fully automated exploratory analysis on a survey-response file (CSV, XLSX or SPSS) with no prior guidance,
  and deliver a storytelling HTML report. Use when the user says "dig deep", "analyse this survey", "explore this
  survey", "what's interesting in this survey", "find insights", "what can you tell me about these responses", or
  drops a survey export without specific questions. Also triggers on "run dig-deep" or "/dig-deep".
argument-hint: "[survey file] [codebook file]"
metadata:
  version: "0.2.5"
---

# Dig Deep: autonomous survey analysis

The user has a survey export (Qualtrics, SurveyMonkey, Google Forms, Typeform,
Microsoft Forms, SPSS or custom) and no specific question. Find what matters and
deliver a report a non-technical manager can act on.

Read both reference files in this skill's folder before you start:
- [reference/methods.md](reference/methods.md): how to read the export and compute every number correctly.
- [reference/report.md](reference/report.md): the report design, writing rules and the HTML template.

Work in `insight-badger/<file name>/` next to the survey (on Claude.ai, in the
outputs folder the environment provides). Use Python with pandas, numpy, scipy
and scikit-learn; if a package is missing, ask the user before installing it.

## 1. Read and profile

Load the file following methods.md §1 (Qualtrics and SurveyMonkey have extra
header rows), then classify every column (§2–3): question types, rating batteries,
multi-select groups, KPI candidates, demographics, personal-data columns.

Tell the user the questionnaire structure in one or two sentences and which
question you will treat as the KPI. If the file is not a survey, say so and offer
a generic analysis instead.

## 2. Analyse

Write `analysis.py` that computes everything below and saves it to
`results.json` (methods.md §11). Run it, read the output, and fix anything that
looks wrong (a scale read as dates, percentages that don't add up, empty groups).

- Data quality (§4): finished, speeders and straight-liners removed, routing, blanks.
- Every question's headline metric (§5); the KPI with its likely range.
- Drivers of the KPI with zones (§7), leaving out other overall questions.
- The KPI by each demographic (up to 5), with Holm-adjusted verdicts, and an
  item-by-item comparison of the highest and lowest groups (§6).
- Mindsets (§8), if there are enough people and rating items.
- Trend and mix shift (§10), if there is a wave or period column.

## 3. Code the comments

For each open-text question, read the answers and code them yourself (§9):
4–8 themes, a tone per comment, counts computed in code from your coding, 1–2
cleaned quotes per theme. Add the results to `results.json`.

## 4. Write the report

Write `build_report.py` that reads `results.json` and writes the HTML report from
the template in report.md, with your words (title, answer, chapter titles, ledes,
"So what", persona names, actions, next questions) as variables in the script.

- **Title**: the single most important finding, as a sentence.
- **Answer**: 2–3 sentences, leading with the KPI and its biggest driver.
- **Chapters**: 3–5, ordered by importance. Typical: Drivers (driver matrix),
  Groups (groups chart and the two-group comparison for the biggest real gap),
  Mindsets (personas), Comments (themes and quotes), All ratings (diverging
  chart). Skip a chapter when the data does not support it.
- **Actions**: 2–4, tagged Act now / Plan next / Protect.
- **Next questions**: 2–3 follow-ups worth running with `/dig-for`.

Apply the user's `.insight-badger.json` branding if there is one (report.md §5).
Run the checks in report.md §7 before you hand it over.

## 5. Reply

2–3 short paragraphs: the KPI and the most actionable finding, the biggest gap or
segment, and what to do next. Give the report path and suggest opening it in a web
browser: app previews may show it as one long page. Offer a PDF, branded or
unbranded (report.md §6). No tables, no emoji.

## Guardrails

- Never report a gap marked chance or small group as a finding.
- Never quote personal details; remove emails, phone numbers and links from quotes.
- Never put raw data rows in the report.
- Never invent benchmarks. Compare to an industry figure only if the user supplied it.
- If the sample is under 100, or completion is low, say so in the answer.
