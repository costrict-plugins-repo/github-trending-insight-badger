# Insight Badger

Survey-specialised data insights for non-technical professionals. Drop a survey response CSV from any platform — Qualtrics, SurveyMonkey, Google Forms, Typeform, Microsoft Forms, custom — and get meaningful, actionable analysis. No spreadsheet formulas, no SPSS, no coding.

## What it does

Insight Badger reads your survey export, automatically classifies each column by question type (Likert, NPS, single-choice, multi-select, open-text, demographic), reconstructs matrix and multi-select questions, and applies the right method per type. Results are presented in plain language with a polished HTML report.

## Skills

### `/dig-deep` — Autonomous Survey Analysis

**No questions asked.** Hand over a survey CSV and Insight Badger explores it autonomously:

- Detects question types per column (Likert 5/7, NPS 0–10, CSAT, single-choice, multi-select, open-text, numeric, demographic, datetime)
- Reconstructs matrix questions (`Q3_1…Q3_n`) and multi-select batteries (`Q5_optA…optN`)
- Computes the right metric per type — top-2-box / bottom-2-box / net-agree for Likert, NPS score with promoter/passive/detractor split, response distribution for single-choice
- Identifies a likely KPI question (NPS, recommend, overall satisfaction) and runs driver analysis
- Builds attitudinal segments from Likert items and names them as personas
- Themes open-text responses with exemplar quotes per theme
- Cross-tabs the KPI by detected demographics and flags significant gaps
- Delivers 3–5 key takeaways and recommended follow-up questions

**Trigger phrases**: "dig deep", "analyse this survey", "what's interesting in this survey", "explore this survey", "find insights"

### `/dig-for` — Guided Survey Investigation

**You have a question — Insight Badger sharpens it and answers it.** When you want to investigate a specific aspect of the survey, this skill asks a few survey-aware clarifying questions and runs a focused analysis.

Common patterns it handles:

- "What's driving our NPS?" → driver analysis on Likert items predicting NPS
- "Compare power users vs. light users" → cross-tabs + significance tests
- "Why did satisfaction drop in Q3?" → trend + driver decomposition (longitudinal surveys)
- "How do our enterprise customers feel about pricing?" → scoped subgroup analysis
- "What do people complain about in open-text?" → themed open-text analysis with quotes

**Trigger phrases**: "dig for", "what's driving NPS", "compare segments", "why did X change", "break down Y by Z"

## Output

Both skills produce:

1. **Conversation summary** — a concise plain-language answer right in the chat, leading with the headline finding (NPS score, top driver, biggest segment gap).
2. **HTML report** — a polished, self-contained file with interactive charts and detailed findings, saved to your output folder.

The HTML report is organised into five tabs:

- **Insights** — KPI summary, headline findings, driver chart, top segment differences, key takeaways
- **Questions** — one card per question with the right visualisation per type (Likert bar with top-2-box callout, donut for single-choice, NPS gauge, themed quote cards for open-text)
- **Segments** — attitudinal segments with persona profiles and demographic composition
- **Data** — health check per column (traffic-light rating), skip-logic interpretation, response quality flags (straight-lining, speeders, completion rate)
- **Info** — report metadata, column reference (codebook), techniques applied, raw data sample

## Survey-Aware Methods

Insight Badger draws from a survey-research toolkit:

- **Question-type detection** — header patterns, value ranges, response text matching
- **Matrix and multi-select reconstruction** — group `Q3_1…Q3_n` as one item battery, `Q5_optA…optN` as one multi-response question
- **Codebook ingestion** — optional second file (CSV/JSON) mapping column → question text + response options; inferred from headers if absent
- **Likert metrics** — top-2-box, bottom-2-box, net-agree, mean score (with caveats), distribution shape
- **NPS / CSAT / CES** — promoter/passive/detractor split, NPS score, CSAT %, with confidence intervals
- **Construct reliability** — Cronbach's alpha for matrix question batteries
- **Driver analysis** — relative weights / dominance analysis predicting KPI from Likert composites
- **Attitudinal segmentation** — K-Means / GMM on Likert items with auto-named personas
- **Open-text NLP** — TF-IDF + clustering for themes, VADER sentiment, exemplar quotes per theme
- **Cross-tabs** — KPI by demographic with chi-squared / z-test for proportions
- **Skip-logic awareness** — distinguishes "not asked" from "didn't answer"
- **Response quality flags** — straight-lining detection, speeder/laggard flagging, completion rate
- **Significance testing** — chi-squared, z-test for proportions, t-test, ANOVA — with effect sizes

## Typical Workflow

**Option A — Explore freely:**
Drop a survey CSV and say "dig deep" or "analyse this survey". Insight Badger profiles the questionnaire, identifies the KPI, runs drivers and segments, themes the open-text, and delivers a full HTML report.

**Option B — Investigate a question:**
Say "dig for what's driving our NPS" or "compare enterprise vs. SMB on satisfaction" along with your CSV. Insight Badger asks one or two clarifying questions, then runs a focused analysis.

**Optional codebook:**
For sharper results, provide a codebook alongside your CSV — a JSON or CSV file mapping each column to its question text and response options. Insight Badger works without one (inferring from headers and values), but a codebook eliminates ambiguity.

## Requirements

- Python 3.x with: pandas, numpy, scikit-learn, scipy, matplotlib, seaborn, vaderSentiment (for open-text sentiment)
- Survey response CSV from any platform (Qualtrics, SurveyMonkey, Google Forms, Typeform, Microsoft Forms, custom exports)

## Setup

No configuration needed. Install the plugin and start using `/dig-deep` or `/dig-for`.
