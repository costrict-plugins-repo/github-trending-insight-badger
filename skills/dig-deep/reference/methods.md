# Survey analysis methods

How to read a survey export and compute every number in the report. Write your
analysis in Python (pandas, numpy, scipy, scikit-learn) following these rules.
The snippets show the parts that are easy to get wrong; adapt them to the file.

Work in a folder next to the survey: `insight-badger/<file name>/`. Put your
analysis in `analysis.py`, write every number the report needs to
`results.json`, and build the report from that file, so no number is typed by
hand.

---

## 1. Reading the export

Read every cell as text first (`dtype=str, keep_default_na=False, header=None`),
then detect the layout. Try encodings `utf-8-sig`, `utf-16`, `cp1252`, `latin-1`
and sniff the delimiter (`, ; \t |`).

| Platform | Layout | What to do |
|---|---|---|
| **Qualtrics** | Row 1 codes (`Q1`, `Q3_2`), row 2 question wording, row 3 `{"ImportId": ...}` | Use row 1 as column names, row 2 as question labels, drop rows 2 and 3 |
| **SurveyMonkey** | Row 1 question text (blank for sub-columns), row 2 sub-options (`Response`, `Open-Ended Response`, option names) | Forward-fill row 1, name each column `question | sub-option` unless the sub-option is `Response` or blank, drop both rows |
| **Google Forms** | One header row of full question text; grids as `Question [Row]`; checkboxes as `A, B, C` in one cell | Use as is |
| Typeform, Microsoft Forms (XLSX), plain CSV | One header row | Use as is |
| SPSS `.sav` | Needs `pyreadstat` | `read_sav(path, apply_value_formats=True)`; variable labels are the question text |

Then turn blank strings into missing values, trim text, and convert a column to
numbers only if **every** non-missing value is numeric.

An optional codebook (CSV/JSON/XLSX with `column_name`, `question_text`,
optional `type`) overrides labels and types.

## 2. Question types

Classify every column before analysing anything. Check in this order:

1. **Metadata**: response/respondent ID, collector, recipient fields, IP address,
   location, distribution channel, language, start/end/recorded date, progress,
   finished, duration, timestamp, Qualtrics `Status` ("Response Type"). Match whole
   column names; "Employment status" or "Preferred language" are questions.
2. **Personal data** (never shown): email, names, phone, IP address,
   latitude/longitude, postcode, street address, recipient fields.
3. **Numbers** (all values numeric):
   - whole numbers 0–10 with a "recommend"/"NPS" hint, or with 0 present, max ≥ 9
     and 6+ distinct values → `nps`
   - 1–5 → `likert` (or `csat` with a "satisf" hint); 1–7 → `likert` (or `ces`
     with an "effort/easy" hint). **A 1–7 or 1–10 scale is not NPS**, and
     "likely to renew" on 1–5 is a rating, not NPS.
   - 0/1 only → `binary`; other 0–10 ranges → `rating`
   - demographic hint → `demographic`; otherwise `numeric`
4. **Text**:
   - every answer on a known scale (section 3) → `likert` / `csat` / `ces`
   - "10 - Extremely likely" style → read the leading number and apply rule 3
   - dates (`2026-05-04 08:00`, `04/05/2026`) → `datetime`. **Never call
     `pd.to_datetime` on numbers**: it reads 7 as a 1970 timestamp.
   - delimited multi-select: a few short options that recur across people,
     separated by `;`, `|` or `,`. **A comma followed by exactly three digits is a
     thousands separator** ("1,000+ employees" is one answer, not two).
   - open text: long answers (average > 30 characters) or mostly unique answers;
     or a comment-style question ("why", "comment", "anything else", "_TEXT")
   - demographic hint (age, gender, region, country, "where is your company based",
     role, department, industry, company size, "how many people/employees",
     tenure, "how long have you been", plan, tier, segment, education, income)
     with up to 40 values → `demographic`
   - up to 25 values → `single-choice`

**Groups of columns:**
- **Rating batteries (matrix questions)**: columns sharing a stem (`Q3_1…Q3_7`,
  `Question - item`, `Question | item`, `Question [item]`) on the same scale.
- **Multi-select batteries**: sibling columns that each hold a single value (1, the
  option text, Yes) or 0/1. **A blank means "not selected", not "missing".**

**KPI**: NPS first; otherwise an "overall satisfaction" question; then CSAT;
then likelihood to renew / continue. Other overall questions are outcomes, not
drivers (section 7).

Show the user the questionnaire structure in one or two sentences, not a table.

## 3. Answer scales and "don't know"

Map text answers to codes 1..k, lowest first. Normalise first: lower case, trim,
curly quotes to straight, drop numeric prefixes and suffixes ("5 - Strongly agree",
"Agree (4)").

| Scale | Codes 1 → k |
|---|---|
| Agreement (5) | strongly disagree · disagree / somewhat disagree · **neither agree nor disagree** / neutral · agree / somewhat agree · strongly agree |
| Agreement (7) | strongly disagree · disagree · somewhat disagree · neither agree nor disagree · somewhat agree · agree · strongly agree |
| Satisfaction (5) | very dissatisfied · dissatisfied · neither satisfied nor dissatisfied · satisfied · very satisfied |
| Frequency (5) | never · rarely · sometimes · often · always |
| Quality (5) | very poor · poor · fair / average · good · excellent / very good |
| Likelihood (5) | very unlikely · unlikely · neither likely nor unlikely · likely · very likely |
| Importance (5) | not at all · slightly · moderately · very · extremely important |
| Ease (5) | very difficult · difficult · neither easy nor difficult · easy · very easy |

Choose the scale with the fewest points that covers all answers **with different
answers on different codes**: data with both "disagree" and "somewhat disagree"
is a 7-point scale.

**Not applicable answers** ("Not applicable", "N/A", "Don't know", "Prefer not to
say", "No opinion", "Unsure") become missing, so they never count in the base of
a percentage. Encode every rating column once and use the codes everywhere:
drivers, segments, reliability and trends all need numbers, not text.

```python
def norm(v):                        # "5 - Strongly agree " -> "strongly agree"
    s = str(v).strip().lower().replace("’", "'")
    s = re.sub(r"^\(?\d{1,2}\)?\s*[-:=.)]\s*", "", s)
    return re.sub(r"\s*[(\[]\s*\d{1,2}\s*[)\]]\s*$", "", s).strip(" .")

def encode(series, order):          # order: {normalised answer: code}; N/A answers are absent
    return series.map(lambda v: order.get(norm(v), np.nan) if pd.notna(v) else np.nan)
```

## 4. Data quality

- **Finished**: Qualtrics `Finished` / `Progress = 100`; otherwise people who
  answered most questions.
- **Speeders**: finished in **under a third of the median time** of finishers.
  (A fixed percentile always flags the same share, even when nobody rushed.)
  Use a duration column, or end minus start time.
- **Straight-liners**: identical answers across every rating battery of 4+ items,
  with at least 8 rated items in total.
- Remove speeders and straight-liners from the analysis by default and say how
  many.
- **Missing answers**: explain before flagging.
  - *Routing (skip logic)*: missing for exactly the people who gave certain answers
    to an earlier question → "Asked only if Q5 was Yes". Not a problem.
  - *Drop-off*: missing only among people who did not finish. Not a problem.
  - *Optional comment boxes*: blanks are expected.
  - Otherwise: blank among finishers; Good < 10%, Check 10–25%, Weak > 25%.

## 5. Metrics

**Ratings**: headline = % in the top two boxes ("agree or strongly agree"; top
one box on a 3-point scale; 9–10 on a 0–10 rating). Also bottom two, net, the
distribution, and the average as a secondary number only.

**NPS** = % scoring 9–10 minus % scoring 0–6. Promoters and detractors come from
the same people, so their covariance belongs in the likely range:

```python
p, d, n = promoters_share, detractors_share, n_eff
se = np.sqrt((p + d - (p - d) ** 2) / n)          # not p(1-p)/n + d(1-d)/n
ci = (nps - 1.96 * se * 100, nps + 1.96 * se * 100)
```

**Proportions**: Wilson interval. **Weights**: weighted shares; use the effective
sample size `n_eff = sum(w)^2 / sum(w^2)` in every interval and test.

**Multi-select**: % of people choosing each option, with the base = people who
answered the question (for blank-coded exports: people who chose at least one
option). Also the average number chosen and the top combinations.

**Single choice**: % per answer, top 7 plus "Other".

## 6. Comparing groups

For each comparison column (demographics, or the user's chosen groups):

- Order bands naturally ("1-10", "11-50", "1,000+"; "Less than 6 months",
  "6-12 months", "1-2 years", "More than 2 years": convert months to years).
- KPI per group with its likely range and size.
- Test each group against everyone else: two-proportion z-test for top-box,
  `(nps_a - nps_b) / sqrt(var_a + var_b)` for NPS, Mann-Whitney / Kruskal-Wallis
  for averages, chi-squared for distributions.
- **Adjust the p-values for the number of tests (Holm)** across all groups and
  columns compared:

```python
def holm(p):
    order = np.argsort(p); m = len(p); out = np.empty(m); run = 0
    for rank, i in enumerate(order):
        run = max(run, min(1, (m - rank) * p[i])); out[i] = run
    return out
```

- Verdict per gap:
  - **small group**: fewer than 30 people → show greyed out, never call it real
  - **real**: adjusted p < 0.05 and gap ≥ 5 points (10 for NPS)
  - **probably real**: adjusted p < 0.2 and gap ≥ 5 points (10 for NPS)
  - **chance**: otherwise; do not report it as a finding
- For the two groups that matter (the user's pair, or the highest and lowest),
  compare every rating item's top-box with Holm-adjusted tests.

## 7. Drivers

Which rating items explain the KPI. Use **relative weights** (robust when items
are related to each other, which they always are), plus performance (% top-box):

```python
def relative_weights(X, y):        # X: DataFrame of item codes, y: KPI codes
    Z = (X - X.mean()) / X.std(ddof=0); yz = (y - y.mean()) / y.std(ddof=0)
    R = np.corrcoef(Z.T.values); rxy = Z.T.values @ yz.values / len(yz)
    ev, V = np.linalg.eigh(R); lam = V @ np.diag(np.sqrt(np.clip(ev, 1e-10, None))) @ V.T
    beta = np.linalg.solve(lam, rxy)
    raw = (lam ** 2) @ (beta ** 2)            # sums to R²
    return pd.Series(raw / raw.sum() * 100, index=X.columns), beta @ beta
```

- Drivers: rating items answered by 60%+; fill remaining gaps with the median.
- **Leave out other overall/outcome questions** (overall satisfaction when NPS is
  the KPI, likelihood to renew): they measure the result, not its causes.
- Needs 50+ people with a KPI answer and 3+ items; warn when fewer than 10 people
  per item.
- Zones by the average importance and average performance: **fix first**
  (important, rated low), **strengths to protect** (important, rated high),
  **monitor** (less important, low), **nice to have** (less important, high).
- Items with a negative link to the KPI: say so and check their wording.

**Reliability**: for each battery, Cronbach's alpha on the codes. ≥ 0.7 "hang
together well", 0.6–0.7 "partly", < 0.6 "measure different things" (report items
one by one).

## 8. Mindsets (segments)

- Rating items answered by 80%+, people with 80%+ of them answered, excluding
  speeders and straight-liners; needs 100+ people and 5+ items.
- Standardise, k-means for k = 3–6 (`n_init=10`, fixed `random_state`), skip any
  k with a group under 5%, choose the best silhouette. Below 0.12 say the groups
  overlap (tendencies, not categories).
- Per segment: size, KPI, the items rated clearly higher and lower than average
  (standardised gap > 0.2), and the demographic values it over-represents (share
  ≥ 25% and ≥ 1.3× the overall share).
- Name each segment as people would recognise it ("Frustrated small teams"),
  never "Cluster 2". Flag the one that needs action.

## 9. Open-text comments

Code the comments yourself; you read short survey comments better than any
clustering method, in any language.

1. Drop blanks and "nothing / n/a / no comment" answers (count them).
2. If there are more than 400, code a random sample of 400 and say so.
3. Build 4–8 themes a manager would recognise. Each comment can have several
   themes and one tone: negative, mixed or positive.
4. Count themes in code from your coding, never by estimate: % of coded comments,
   tone split, and the split by NPS group (promoter / passive / detractor).
5. Pick 1–2 short, vivid quotes per theme. **Remove emails, phone numbers, links
   and long ID numbers** from every quote, and never quote names.

## 10. Trends

With a wave or period column: KPI per wave with its range, a test of the latest
change, and a check of whether the respondent mix shifted between waves (largest
change in any demographic share ≥ 5 points). A mix shift can move the score on
its own: say so.

## 11. results.json

Keep every number the report uses in one file, for example:

```json
{"meta": {"file": "...", "n_rows": 600, "n_analysed": 588, "removed": {"speeders": 12, "straightliners": 0},
          "generated": "2026-11-16"},
 "health": {"started": 600, "finished": 566, "routed": ["Q6"], "columns": [...]},
 "kpi": {"column": "Q1", "kind": "nps", "value": -6.6, "ci": [-13.5, 0.3], "n": 588,
         "promoters": 38.1, "passives": 17.2, "detractors": 44.7},
 "drivers": {"r2": 76.5, "items": [{"column": "Q3_2", "label": "First-month value", "importance": 28.0,
             "performance": 21.1, "zone": "fix"}]},
 "comparisons": [{"column": "Q9", "groups": [{"value": "1-10 employees", "n": 119, "kpi": -49.6,
                  "ci": [-62, -37], "verdict": "real"}], "pair": {...}}],
 "segments": [...], "themes": [...], "trend": {...}, "caveats": [...]}
```
