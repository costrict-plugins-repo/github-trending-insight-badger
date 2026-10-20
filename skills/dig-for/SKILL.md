---
name: dig-for
description: >
  Perform guided survey analysis when the user has a direction but needs help articulating it.
  Use when the user says "dig for", "what's driving NPS", "why is satisfaction down",
  "compare power users vs light users", "compare these segments", "break down Y by Z",
  "what do people complain about", "why did X change", or has a survey CSV and a rough question.
  Also triggers on "run dig-for" or "/dig-for".
metadata:
  version: "0.2.0"
---

# Dig For — Guided Survey Investigation

The user has a survey-response CSV (any platform) and a rough idea of what they want to understand. Help them sharpen the question, then deliver a focused, survey-aware analysis.

**Hard assumption: the input is a survey.** Treat columns as questions, rows as respondents. Use the reference file `references/analysis-techniques.md` throughout.

## Execution Flow

### 1. Quick Survey Profile

Read the CSV and run question-type detection (§ 1 of the reference) and question grouping (§ 2). You need to know what KPI candidates, demographic candidates, and item batteries exist before you can ask the user a useful clarifying question.

If a codebook file accompanies the CSV, ingest it (§ 3).

Don't profile or report yet. Just build the internal map of `column → question_type → group`.

### 2. Clarify Intent

Use **AskUserQuestion** to clarify. Adapt questions to what the user has already said — never re-ask what's obvious.

Survey-specific clarifying angles (ask 1–3, not all):

- **KPI**: "Which question is your KPI for this analysis?" Offer detected NPS / CSAT / overall-satisfaction / recommend columns. If none, offer the top Likert items.
- **Comparison group**: "Which group should I compare?" Offer detected demographics (region, segment, role, company size, etc.) and any single-choice questions with low cardinality.
- **Construct**: "Which set of questions defines [the topic the user mentioned]?" Offer detected item batteries by stem.
- **Time**: "Should I compare across response waves or look at the trend over time?" Offer the timestamp column or detected wave column if present.
- **Scope filter**: "Should I focus on all respondents or a subset (e.g. only completed responses, only enterprise customers, only the last wave)?"
- **Depth**: "Quick snapshot or thorough deep-dive?"

If the user's original message already answers some of these (e.g. "what's driving NPS for enterprise customers"), skip the answered ones and only ask what's still ambiguous.

### 3. Plan the Analysis

Based on the answers, silently pick survey-aware techniques. Use the reference for method details. Typical mappings:

| User intent | Techniques |
|---|---|
| "What's driving X?" (X is KPI) | Driver analysis on Likert items (§ 11) + matrix-battery composites (§ 12) |
| "Compare segment A vs B" | Cross-tab + z-test for proportions (top-2-box) + chi-squared on distributions (§ 14) |
| "What predicts X?" | Driver analysis + correlation + Random Forest feature importance |
| "Break down X by Y" | Group-by aggregation per Y, distribution comparison, significance flags |
| "Why did X drop?" | Trend over waves + driver decomposition + check for compositional shifts |
| "What's our score on X?" | Top-2-box / NPS / CSAT calculation + confidence interval + sample size |
| "What do people complain about?" | Open-text themes (§ 9) + sentiment + exemplar quotes + topic-by-segment cross-tab |
| "Are these ratings normal?" | Distribution + percentile context + benchmark comparison if industry data referenced |
| "Find segments / personas" | Attitudinal clustering on Likert items (§ 13) with persona naming |

### 4. Execute

Run the chosen techniques in Python (pandas, numpy, scikit-learn, scipy, vaderSentiment).

For every finding:
- State the fact in one plain sentence.
- Give the supporting number or chart.
- Explain why it matters to the user's question.

Never use ordinal scales as if they were continuous in user-facing text — always lead with top-2-box / NPS / proportions, and use means only as a secondary readout.

### 5. Scoped Quality Check

Only surface quality issues that affect the user's specific question. Examples:

- They ask about NPS drivers but 30 % of NPS values are missing → critical, mention it.
- They ask about enterprise customers but the segment column is missing for 25 % of respondents → mention it.
- A matrix battery used as a composite has Cronbach's alpha < 0.6 → mention it (the construct may not hold together).
- An unrelated column has issues → ignore it.

Skip-logic missingness should be interpreted as legitimate, not flagged.

### 6. Synthesise

End with:

- **Direct answer**: address the user's question in 2–3 sentences, as if briefing a colleague. Lead with the headline number.
- **Supporting evidence**: 3–5 most important data points or charts that back up the answer.
- **Caveats**: anything that weakens the conclusion (small subgroup sample, low alpha, high missingness, confounding demographic).
- **Next steps**: 1–2 follow-up questions worth exploring (e.g. "Worth running `/dig-deep` on this same file to see the full attitudinal segments?").

## Output Rules

1. **Two outputs**: concise answer in the conversation AND a focused HTML report saved to the user's output folder.

2. **HTML report**: Use the tabbed template from `references/analysis-techniques.md` § 16. Tabs are scoped to the question:
   - **Insights** (default): direct answer card, supporting charts, findings, follow-up suggestions. Only include charts relevant to the user's question — no filler.
   - **Questions**: only the questions touched by this analysis (the KPI, the drivers, the items being compared). Skip the tab entirely if the analysis touches only one question.
   - **Segments**: only included if the user's question involves segmentation or attitudinal clustering.
   - **Data**: scoped quality information — only the quality issues affecting the user's specific question.
   - **Info**: report metadata, column reference (limited to columns used), techniques applied, raw data sample.

3. **Branding**: brand colours `#3F8585` primary, `#FEFBF1` background, `#39403A` dark. Follow the chart palette in the template reference.

4. **Conversation summary**: lead with the direct answer (e.g. "Your NPS is 42, with onboarding clarity as the strongest driver"). Keep it to 2–3 short paragraphs. Link the report for details.

5. **Language**: non-technical. Avoid "ordinal", "MAR", "eta-squared", "Cronbach's alpha", "p-value" in user-facing text. Translate: "statistically significant" → "unlikely to be a coincidence"; "Cronbach's alpha 0.82" → "the questions in this set hang together well — they're measuring one thing"; "p < 0.05" → "this gap is real, not noise".

6. **Charts**: focused on the user's question. Each chart gets a plain-language title and a one-sentence caption. No emoji anywhere.

## Conversation Style

Be warm, curious, and collaborative — like a smart survey analyst sitting next to the user. Avoid:
- Lecturing about methodology (Cohen's d, alpha, eta-squared) unprompted
- Dumping tables of raw numbers
- Excessive hedge words ("it seems like maybe perhaps")

Do:
- Give confident answers when the data supports them
- Be upfront about uncertainty (small subgroup, low alpha, missing data)
- Suggest what to look at next

## Python Environment

Use pandas, numpy, scikit-learn, scipy.stats, matplotlib/seaborn, vaderSentiment. All are available via pip. For the HTML report, generate Chart.js configs and embed them in the HTML.

## Hard Specialisation

This skill assumes the input is a survey. If the file genuinely doesn't look like a survey, tell the user explicitly and offer to run generic analysis instead.
