---
name: dig-deep
description: >
  Perform fully automated exploratory analysis on a survey-response CSV with no prior guidance.
  Use when the user says "dig deep", "analyse this survey", "explore this survey",
  "what's interesting in this survey", "find insights", "what can you tell me about these responses",
  or drops a survey CSV without specific questions. Also triggers on "run dig-deep" or "/dig-deep".
metadata:
  version: "0.2.0"
---

# Dig Deep — Autonomous Survey Analysis

The user has a survey-response CSV (any platform — Qualtrics, SurveyMonkey, Google Forms, Typeform, Microsoft Forms, custom) and no specific question. Produce a comprehensive, plain-language analysis a non-technical professional can act on.

**Hard assumption: the input is a survey.** Treat columns as questions, rows as respondents. Apply survey-aware methods first; fall back to generic methods only for columns that don't match any question type.

Use the reference file `references/analysis-techniques.md` for detailed method guidance throughout.

## Execution Flow

### 1. Ingest & Question-Type Detection

Read the CSV with Python (pandas). Then **classify each column as a question type** before anything else. See § 1 of the reference for full heuristics.

For every column produce a tag from:

- `likert-5` / `likert-7` (ordinal scale, with optional polarity: agreement / satisfaction / frequency)
- `nps` (0–10 recommend scale)
- `csat` (1–5 satisfaction)
- `ces` (Customer Effort Score, 1–7)
- `single-choice` (categorical, low cardinality, no ordering)
- `multi-select-binary` (a battery of 0/1 columns sharing a stem like `Q5_optA`, `Q5_optB`)
- `multi-select-string` (one column with semicolon/comma-separated options)
- `open-text` (free text, high cardinality, long strings)
- `numeric` (continuous numeric like age, income, tenure)
- `demographic` (age, gender, region, role, company size, segment)
- `datetime` (response timestamp or longitudinal wave)
- `metadata` (response ID, IP, duration, completion flag, language)
- `unknown` (fallback when no rule matches)

Use header patterns (`Q1`, `Q2_1`, `Q3a`), value ranges, value text matching (`"Strongly agree"`, `"Promoter"`), and cardinality thresholds.

### 2. Question Grouping

Reconstruct the questionnaire structure:

- **Matrix questions**: group columns sharing a stem (e.g. `Q3_1…Q3_5`, `sat_speed / sat_quality / sat_price`) when they share the same Likert scale. Treat each matrix as one item battery measuring potentially one construct.
- **Multi-select binary batteries**: group `Q5_optA…Q5_optN` as one multi-response question. Re-derive the option labels from suffix or header text.
- **Multi-select string**: split the cell on common delimiters (`;`, `,`, `|`) and re-pivot to a binary matrix internally.

### 3. Optional Codebook Ingestion

If the user provides a second file alongside the CSV (CSV, JSON, or Excel codebook mapping column → question text + response options), parse it and use it to override inferred labels. If not, infer question text from the column header (truncate long ones for charts; keep full text in the Info tab).

### 4. Survey Health Check

Always include this section. Compute and present:

- **Sample size** and completion rate (proportion of respondents who finished)
- **Skip-logic interpretation**: distinguish "wasn't asked" (legitimate) from "didn't answer" (potential issue) per column. Use co-missingness patterns and (if available) routing rules from the codebook.
- **Response quality flags**:
  - **Straight-liners**: respondents giving the same answer to every Likert item in a matrix
  - **Speeders**: respondents whose response duration is below the 5th percentile (if duration column exists)
  - **Inattentive responders**: failed attention checks (if detected)
- **Per-column traffic-light rating** (green / yellow / red):
  - Green: high completion, sensible distribution, no straight-lining concentration
  - Yellow: moderate missingness, some skewed answer patterns, possibly skip-logic-driven
  - Red: very high missingness without skip-logic explanation, single-value domination, suspect quality

Generic "outlier beyond 3σ" alarms are inappropriate for Likert and NPS — do not raise them.

### 5. Per-Question Reporting

For every detected question, run the right method (see § 4–9 of the reference):

- **Likert**: top-2-box, bottom-2-box, net-agree, full distribution. Mean score reported with the caveat that the scale is ordinal.
- **NPS**: promoter / passive / detractor split, NPS score, sample size, confidence interval.
- **CSAT / CES**: % satisfied (top-2-box), mean score, distribution.
- **Single-choice**: response distribution, modal answer, comparison to even split.
- **Multi-select**: option frequencies (% of respondents selecting each), mean number of options selected, common combinations.
- **Open-text**: theme extraction (TF-IDF + clustering, lightweight — see § 9), sentiment per theme, 2–3 exemplar quotes per theme.
- **Demographic**: distribution, share of total. No statistical analysis on its own — these are used as cross-tab dimensions.

Do **not** dump every question into the conversation summary. Save the full per-question deck for the **Questions** tab of the HTML report. In the conversation, surface only the most striking 5–7 findings.

### 6. KPI Detection & Driver Analysis

Identify a likely KPI question by name + type:

- NPS columns (`nps`, `recommend`, `would_recommend`)
- Overall satisfaction (`overall_sat`, `csat`, `satisfaction_overall`)
- Likelihood to renew / repurchase
- Single-choice "How would you rate us" with 5/7-point ordinal text

If a KPI is found, run **driver analysis** (§ 11 of the reference): compute relative weights or dominance scores for each Likert item / matrix battery as predictors of the KPI. Present a horizontal bar chart of the top 8 drivers with plain-language captions ("Customers who feel onboarded well are the most likely to recommend us").

If no clear KPI exists, skip and say so.

### 7. Cross-Tabs by Demographics

For each detected demographic (max 5), break the KPI and the top 3 highest-variance Likert items by that demographic. Run the appropriate significance test (§ 14):

- **z-test for proportions** when comparing top-2-box across two groups
- **chi-squared** when comparing distributions across 3+ groups
- **t-test / ANOVA** for mean comparisons (with the ordinal caveat)

Flag only differences that are **both statistically significant (p < 0.05) and practically meaningful** (≥ 5 percentage points on top-2-box, or ≥ 0.3 effect size). Skip the rest.

### 8. Attitudinal Segmentation

If there are ≥ 100 respondents and ≥ 5 Likert items:

- Reverse-code negatively-worded items where possible (heuristic: items where high agreement signals dissatisfaction).
- If matrix batteries exist, check Cronbach's alpha (§ 12). When α ≥ 0.7, collapse the battery to a composite score for clustering.
- Run K-Means with k chosen by silhouette score (search k = 3–6). Cross-check with GMM if silhouette is borderline.
- **Profile each segment** by its Likert means and demographic composition. Give each a plain-language label (e.g. "Engaged advocates", "Frustrated power users", "Lukewarm passives").
- Show segment sizes, KPI per segment, and 2–3 distinguishing items per segment.

If the data is too small or too sparse, skip this section and explain why.

### 9. Open-Text Themes

For each open-text question with ≥ 30 non-empty responses:

- Lemmatise + remove stopwords + TF-IDF vectorise
- Cluster with K-Means (k chosen by silhouette, capped at 6)
- Label each cluster from its top terms (Claude can rewrite as a human-readable theme name)
- Compute **VADER sentiment** per response and aggregate per theme
- Surface 2–3 representative exemplar quotes per theme (closest to the cluster centroid)

If the volume is too low for clustering, fall back to: top 20 most frequent meaningful words, sentiment summary, a sample of 5 randomly-chosen quotes.

### 10. Synthesise & Recommend

End with a "Key Takeaways" section:

- 3–5 bullets a decision-maker can act on. Lead with the KPI score and its biggest driver.
- Suggest 2–3 follow-up questions worth running with `/dig-for` (e.g. "Why are enterprise customers significantly less satisfied with onboarding?").
- Note caveats (small sample, low completion rate, low Cronbach's alpha for a battery, etc.).

## Output Rules

1. **Two outputs**: a concise summary in the conversation AND a full HTML report saved to the user's output folder.

2. **HTML report**: Use the tabbed template from `references/analysis-techniques.md` § 16. Five tabs:
   - **Insights** (default): KPI summary card, headline findings, driver bar chart, top demographic gaps, segment overview, key takeaways.
   - **Questions**: collapsible list, one card per question, with the right visualisation per type. Group cards by section if the questionnaire has detected sections.
   - **Segments**: attitudinal segment profiles — sizes, KPI per segment, distinguishing Likert items, demographic composition. Skip the tab entirely if segmentation was skipped.
   - **Data**: health check table (traffic-light per column with skip-logic interpretation), response quality flags (straight-liners, speeders, completion), missingness map.
   - **Info**: report metadata, column reference (codebook view: column name → detected type → question text → response options), techniques applied, raw data sample (first 20 rows).

3. **Insights tab prioritisation**: the Insights tab must fit comfortably on screen — do **not** dump every question card here. Cap to: KPI card, top 5 driver findings, top 3 segment gaps, key takeaways. Full per-question detail belongs on the **Questions** tab.

4. **Branding**: brand colours `#3F8585` primary, `#FEFBF1` background, `#39403A` dark. Follow the chart palette in the template reference.

5. **Conversation summary**: 2–3 short paragraphs leading with the KPI score and the single most actionable finding. Then link the HTML report.

6. **Language**: write for someone who has never run a survey analysis before. Avoid "ordinal", "MAR", "eta-squared", "Cronbach's alpha" in user-facing text. If a technical term is necessary, define it in parentheses on first use ("statistically significant — meaning unlikely to be a coincidence"). Internally, always use the right method.

7. **Charts**: every chart needs a clear plain-language title and a one-sentence caption explaining what it shows and why it matters. No emoji anywhere — not in chart titles, not in takeaways, not in any output.

## Python Environment

Use pandas, numpy, scikit-learn, scipy, matplotlib/seaborn, vaderSentiment. All are available via pip. For the HTML report, generate Chart.js configs and embed them in the HTML.

## Hard Specialisation

This skill assumes the input is a survey. If the file genuinely doesn't look like a survey (no Likert / NPS / single-choice / open-text columns detectable, just a flat table of numeric measurements), tell the user explicitly: "This file doesn't look like a survey export. Insight Badger is specialised for survey responses — would you like me to run a generic exploratory analysis instead, or do you have a survey CSV to analyse?"
