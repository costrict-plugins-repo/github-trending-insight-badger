---
name: dig-for
description: >
  Perform guided survey analysis when the user has a direction but needs help articulating it, and deliver a focused
  one-question HTML report. Use when the user says "dig for", "what's driving NPS", "why is satisfaction down",
  "compare power users vs light users", "compare these segments", "break down Y by Z", "what do people complain
  about", "why did X change", or has a survey export and a rough question. Also triggers on "run dig-for" or "/dig-for".
argument-hint: "[question] [survey file]"
metadata:
  version: "0.2.5"
---

# Dig For: guided survey investigation

The user has a survey export and a rough question. Sharpen the question, run a
focused analysis, and deliver a one-question brief: the answer first, then the
evidence, then how sure we are.

Read both reference files in this skill's folder before you start:
- [reference/methods.md](reference/methods.md): how to read the export and compute every number correctly.
- [reference/report.md](reference/report.md): the report design, writing rules and the HTML template.

Work in `insight-badger/<file name>/` next to the survey (on Claude.ai, in the
outputs folder the environment provides). Use Python with pandas, numpy, scipy
and scikit-learn; if a package is missing, ask the user before installing it.

## 1. Read and profile

Load the file following methods.md §1 and classify the columns (§2–3). Note the
KPI candidates, demographics, rating batteries and open-text questions.

## 2. Clarify (AskUserQuestion)

Ask only what the user's message leaves open, 1–3 questions, offering the columns
you found:

- **KPI**: which question measures success.
- **Comparison**: which groups to compare, and which two matter most.
- **Scope**: everyone, or a subset (one wave, one plan, completed responses only).
- **Time**: compare waves, if a wave or date column exists.

Skip what the message already answers.

## 3. Analyse

| Question | What to compute (methods.md) |
|---|---|
| What's driving X? | Drivers of X (§7) |
| Compare A vs B / break down X by Y | X by Y with verdicts, and every rating item for A vs B (§6) |
| Why did X change? | Trend, mix shift and drivers by wave (§10, §7) |
| How do <subgroup> feel about X? | Filter to the subgroup, then §5 |
| What do people complain about? | Comment themes (§9), by group |
| Find personas | Mindsets (§8) |

Write `analysis.py`, save the numbers to `results.json` (§11), run it and check
the output. Always run the data-quality checks (§4), but report only the issues
that affect the question.

## 4. Code the relevant comments

If comments help answer the question, code them (§9), focusing on the groups in
question, and add the counts and cleaned quotes to `results.json`.

## 5. Write the report

Write `build_report.py` that reads `results.json` and writes the HTML report from
the template in report.md, using the dig-for layout (§3): the question word for
word and 2–3 scope chips in the hero, the answer as the title, 3–4 key figures
that answer the question, 1–3 reason cards with their numbers, a "Not the cause"
note, 1–3 chapters (usually Evidence and In their words), and a "How sure we are"
tab with Solid / Watch caveats. Apply `.insight-badger.json` branding if present
and run the checks in report.md §7.

## 6. Reply

Lead with the direct answer and its number, like briefing a colleague. One short
paragraph of evidence, one line of caveats, then the report path, a suggestion to
open it in a web browser, and an offer of a PDF (branded or unbranded, report.md §6).
No emoji, no tables.

## Guardrails

- Never report a gap marked chance or small group as a finding.
- Never quote personal details; remove emails, phone numbers and links from quotes.
- Never put raw data rows in the report.
- Never invent benchmarks.
