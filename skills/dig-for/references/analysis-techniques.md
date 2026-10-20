# Survey Analysis Techniques Reference

Method library for the survey-specialised insight-badger skills. Each technique includes when to use it, how to implement it in Python, and how to present results to a non-technical audience.

Both skills (`dig-deep` and `dig-for`) should consult this file before running any analysis. **Hard assumption: the input is a survey.** Pick the techniques that fit the detected question types and the user's question — don't run everything blindly.

---

## 1. Question-Type Detection

**When**: Always, immediately after loading. This is the foundation — every downstream method depends on it.

**Goal**: Tag every column with one of: `likert-5`, `likert-7`, `nps`, `csat`, `ces`, `single-choice`, `multi-select-binary`, `multi-select-string`, `open-text`, `numeric`, `demographic`, `datetime`, `metadata`, `unknown`.

**Heuristics — apply in order, first match wins**:

```python
import pandas as pd
import numpy as np
import re

LIKERT_5_TEXT = {
    "strongly disagree", "disagree", "neutral", "agree", "strongly agree",
    "very dissatisfied", "dissatisfied", "neither", "satisfied", "very satisfied",
    "never", "rarely", "sometimes", "often", "always",
    "very poor", "poor", "average", "good", "excellent",
}
LIKERT_7_TEXT = LIKERT_5_TEXT | {
    "somewhat disagree", "somewhat agree",
    "somewhat dissatisfied", "somewhat satisfied",
}
NPS_TEXT = {"detractor", "passive", "promoter"}

DEMOGRAPHIC_HINTS = re.compile(
    r"(age|gender|sex|region|country|state|city|zip|postcode|"
    r"income|education|role|title|department|tenure|company.?size|"
    r"industry|segment|plan|tier|customer.?type)",
    re.IGNORECASE,
)

METADATA_HINTS = re.compile(
    r"(response.?id|respondent.?id|user.?id|email|ip.?address|"
    r"start.?time|end.?time|duration|completed|completion|status|language|locale)",
    re.IGNORECASE,
)

OPEN_TEXT_HINTS = re.compile(
    r"(why|comment|feedback|verbatim|other|please.?explain|"
    r"please.?describe|tell.?us|what.?else|suggestion|specify)",
    re.IGNORECASE,
)


def detect_question_type(series: pd.Series, col_name: str) -> str:
    name = col_name.lower()
    s_clean = series.dropna()
    if len(s_clean) == 0:
        return "unknown"

    # Metadata first
    if METADATA_HINTS.search(name):
        return "metadata"

    # Datetime
    if pd.api.types.is_datetime64_any_dtype(series):
        return "datetime"
    try:
        parsed = pd.to_datetime(s_clean.head(50), errors="coerce")
        if parsed.notna().mean() > 0.8:
            return "datetime"
    except Exception:
        pass

    # Numeric range checks
    numeric = pd.to_numeric(s_clean, errors="coerce")
    if numeric.notna().mean() > 0.9:
        vmin, vmax = numeric.min(), numeric.max()
        unique_vals = set(numeric.dropna().unique())
        # NPS: 0–10 with mostly integer values
        if vmin >= 0 and vmax <= 10 and unique_vals.issubset(set(range(0, 11))):
            if "nps" in name or "recommend" in name or "likely" in name:
                return "nps"
            if len(unique_vals) >= 6:
                return "nps"
        # CSAT 1–5
        if vmin >= 1 and vmax <= 5 and unique_vals.issubset({1, 2, 3, 4, 5}):
            if "csat" in name or "satisfaction" in name or "satisfied" in name:
                return "csat"
            return "likert-5"
        # CES 1–7
        if vmin >= 1 and vmax <= 7 and unique_vals.issubset(set(range(1, 8))):
            if "effort" in name or "ces" in name:
                return "ces"
            return "likert-7"
        # Demographic numeric (age etc.)
        if DEMOGRAPHIC_HINTS.search(name):
            return "demographic"
        # Plain numeric
        return "numeric"

    # Likert by text values
    text_values = set(str(v).strip().lower() for v in s_clean.unique())
    if text_values.issubset(LIKERT_5_TEXT) and 3 <= len(text_values) <= 5:
        return "likert-5"
    if text_values.issubset(LIKERT_7_TEXT) and 4 <= len(text_values) <= 7:
        return "likert-7"
    if text_values & NPS_TEXT:
        return "nps"

    # Multi-select string (semicolon/comma separated)
    if s_clean.dtype == object:
        sep_rate = s_clean.astype(str).str.contains(r"[;|]").mean()
        if sep_rate > 0.2:
            return "multi-select-string"

    # Open-text vs single-choice by length and cardinality
    avg_len = s_clean.astype(str).str.len().mean()
    n_unique = s_clean.nunique()
    n_total = len(s_clean)
    if avg_len > 30 or (n_unique / max(n_total, 1) > 0.5 and avg_len > 10):
        return "open-text"
    if OPEN_TEXT_HINTS.search(name) and avg_len > 15:
        return "open-text"

    # Demographic by name
    if DEMOGRAPHIC_HINTS.search(name) and n_unique <= 30:
        return "demographic"

    # Default to single-choice for low-cardinality categoricals
    if n_unique <= 15:
        return "single-choice"

    return "unknown"
```

**Multi-select binary detection** is structural, not per-column — see § 2.

**Presenting**: Don't show the full classification table to the user. Show only the questionnaire structure summary: "I detected 32 questions across 4 sections — 18 Likert items in 3 matrix batteries, 1 NPS question, 5 single-choice, 2 multi-select, 4 open-text, and 5 demographic columns."

---

## 2. Question Grouping (Matrix & Multi-Select Reconstruction)

**When**: Always, after type detection.

**Matrix questions**: columns sharing a stem with `_1`, `_2`, etc., or sharing a Likert scale with similar names (e.g. `sat_speed`, `sat_quality`, `sat_price`).

```python
from collections import defaultdict

def group_matrix_questions(df, type_map):
    groups = defaultdict(list)
    for col, qtype in type_map.items():
        if qtype not in ("likert-5", "likert-7", "csat", "ces"):
            continue
        # Stem: column with trailing _digit or _word stripped
        m = re.match(r"^(.*?)[_\.\-](\d+|[a-zA-Z]+)$", col)
        stem = m.group(1) if m else col
        groups[stem].append(col)
    return {stem: cols for stem, cols in groups.items() if len(cols) >= 2}
```

**Multi-select binary batteries**: columns sharing a stem with low-cardinality 0/1 or yes/no values. Often exported by Qualtrics as `Q5_1`, `Q5_2`, … or `Q5_optA`, `Q5_optB`, ….

```python
def detect_multi_select_binary(df, type_map):
    groups = defaultdict(list)
    for col in df.columns:
        s = df[col].dropna()
        if s.empty:
            continue
        unique = set(str(v).strip().lower() for v in s.unique())
        is_binary = unique.issubset({"0", "1", "true", "false", "yes", "no", "y", "n", "checked", "unchecked", ""})
        if not is_binary:
            continue
        m = re.match(r"^(.*?)[_\.\-]([\w]+)$", col)
        if m:
            groups[m.group(1)].append(col)
    return {stem: cols for stem, cols in groups.items() if len(cols) >= 2}
```

**Multi-select string**: split the cell on common delimiters and re-pivot.

```python
def explode_multi_select_string(series, delimiters=r"[;,|]"):
    expanded = series.dropna().astype(str).str.split(delimiters).explode().str.strip()
    options = expanded.unique()
    pivot = pd.DataFrame({opt: series.dropna().astype(str).str.contains(re.escape(opt), na=False)
                          for opt in options})
    return pivot
```

**Presenting**: Mention groupings in plain language: "I noticed `Q3_1` through `Q3_5` are five items measuring different aspects of customer experience — I'll analyse them together as one matrix question."

---

## 3. Optional Codebook Ingestion

**When**: If a second file is provided alongside the survey CSV.

**Accepted formats**:

- **CSV codebook** with at minimum: `column_name`, `question_text`. Optionally: `response_options`, `section`, `question_type_hint`.
- **JSON codebook** with the same fields.
- **Excel codebook** (Qualtrics export style) — read first sheet.

```python
def load_codebook(path):
    if path.endswith(".json"):
        return pd.read_json(path)
    if path.endswith(".csv"):
        return pd.read_csv(path)
    if path.endswith((".xls", ".xlsx")):
        return pd.read_excel(path, sheet_name=0)
    return None

# Use codebook to override inferred labels and types
def apply_codebook(type_map, question_text_map, codebook_df):
    if codebook_df is None:
        return type_map, question_text_map
    for _, row in codebook_df.iterrows():
        col = row.get("column_name")
        if col not in type_map:
            continue
        if pd.notna(row.get("question_text")):
            question_text_map[col] = row["question_text"]
        if pd.notna(row.get("question_type_hint")):
            type_map[col] = row["question_type_hint"]
    return type_map, question_text_map
```

If no codebook is provided, infer `question_text` from the column header (truncate long ones for chart titles, keep full text in the Info tab).

**Presenting**: If a codebook was used, mention it once in the Info tab. If not, no mention needed.

---

## 4. Likert Analysis (top-2-box, bottom-2-box, net-agree)

**When**: For every `likert-5` and `likert-7` column.

**Top-2-box** (T2B): proportion of respondents giving the highest two responses. The standard headline metric in market research because it compresses an ordinal distribution into a single intuitive number.

**Bottom-2-box** (B2B): proportion giving the lowest two.

**Net-agree** (or net-positive): `T2B − B2B`.

**Mean score**: report with the caveat that the scale is ordinal (the gap between "agree" and "strongly agree" isn't necessarily the same as between "neutral" and "agree"). Useful as a secondary readout.

```python
LIKERT_5_ORDER_AGREE = ["strongly disagree", "disagree", "neutral", "agree", "strongly agree"]
LIKERT_5_ORDER_SAT = ["very dissatisfied", "dissatisfied", "neither", "satisfied", "very satisfied"]

def likert_metrics(series, scale=5):
    s = series.dropna()
    if pd.api.types.is_numeric_dtype(s):
        # Numeric coding 1..scale
        top_2 = s.isin([scale, scale - 1]).mean()
        bottom_2 = s.isin([1, 2]).mean()
        mean_score = s.mean()
        distribution = s.value_counts(normalize=True).reindex(range(1, scale + 1), fill_value=0)
    else:
        s_lower = s.astype(str).str.strip().str.lower()
        # Identify scale order from values present
        for order in (LIKERT_5_ORDER_AGREE, LIKERT_5_ORDER_SAT):
            if set(s_lower.unique()).issubset(set(order)):
                top_2 = s_lower.isin(order[-2:]).mean()
                bottom_2 = s_lower.isin(order[:2]).mean()
                mapped = s_lower.map({v: i + 1 for i, v in enumerate(order)})
                mean_score = mapped.mean()
                distribution = s_lower.value_counts(normalize=True).reindex(order, fill_value=0)
                break
        else:
            top_2 = bottom_2 = mean_score = np.nan
            distribution = s_lower.value_counts(normalize=True)
    return {
        "top_2_box": top_2,
        "bottom_2_box": bottom_2,
        "net_agree": top_2 - bottom_2,
        "mean": mean_score,
        "distribution": distribution,
        "n": len(s),
    }
```

**Presenting**: "78 % of respondents agree or strongly agree that onboarding was clear (T2B). Only 6 % disagree (B2B), giving a net positive of 72 points — a strong result."

In charts: stacked horizontal bar (B2B in red, neutral in grey, T2B in teal), with the T2B percentage callout on the right.

---

## 5. NPS / CSAT / CES

### Net Promoter Score (NPS)

**When**: For every `nps` column.

```python
def nps_score(series):
    s = pd.to_numeric(series, errors="coerce").dropna()
    s = s[(s >= 0) & (s <= 10)]
    n = len(s)
    if n == 0:
        return None
    promoters = (s >= 9).mean()
    passives = ((s >= 7) & (s <= 8)).mean()
    detractors = (s <= 6).mean()
    nps = (promoters - detractors) * 100
    # 95 % confidence interval (approximate, for proportion difference)
    se = np.sqrt(((promoters * (1 - promoters)) + (detractors * (1 - detractors))) / n)
    ci_half = 1.96 * se * 100
    return {
        "nps": nps,
        "promoters_pct": promoters * 100,
        "passives_pct": passives * 100,
        "detractors_pct": detractors * 100,
        "n": n,
        "ci_low": nps - ci_half,
        "ci_high": nps + ci_half,
    }
```

**Presenting**: "Your NPS is 42 (95 % confidence: 36–48), based on 412 responses. 58 % are promoters, 26 % passives, 16 % detractors. For SaaS the typical NPS range is 30–50, so this sits in healthy territory." NPS visual: a horizontal stacked bar (red detractors, grey passives, green promoters) with the score callout.

### CSAT

```python
def csat_score(series):
    s = pd.to_numeric(series, errors="coerce").dropna()
    s = s[(s >= 1) & (s <= 5)]
    n = len(s)
    if n == 0:
        return None
    csat_pct = (s >= 4).mean() * 100  # Top-2-box on 5-point
    return {"csat_pct": csat_pct, "mean": s.mean(), "n": n}
```

**Presenting**: "84 % of respondents rate themselves satisfied or very satisfied (CSAT)."

### CES (Customer Effort Score)

For 7-point ease scales (`1 = very difficult … 7 = very easy`). Higher is better. Report mean and % easy (top-2-box).

---

## 6. Skip Logic & Missingness Interpretation

**When**: Always, but interpret survey-specifically.

In a survey, missing data has three sources:

- **Skip logic** — the respondent wasn't asked the question (legitimate, not a quality issue).
- **Item nonresponse** — the respondent was asked but didn't answer.
- **Drop-off** — the respondent abandoned the survey before reaching the question.

**Distinguish them**:

```python
def classify_missingness(df, type_map):
    is_missing = df.isnull()
    classifications = {}
    for col in df.columns:
        miss_rate = is_missing[col].mean()
        if miss_rate < 0.02:
            classifications[col] = ("none", miss_rate)
            continue
        # Co-missingness: if missingness in this column is highly correlated
        # with missingness in another column, that suggests skip logic.
        co_miss = is_missing.corrwith(is_missing[col]).drop(col).abs()
        if (co_miss > 0.7).any():
            partner = co_miss.idxmax()
            classifications[col] = ("skip-logic", miss_rate, partner)
            continue
        # Drop-off: increasing missingness with column position
        # (heuristic — works when columns are in survey order)
        classifications[col] = ("item-nonresponse", miss_rate)
    return classifications
```

**Drop-off detection**: plot missingness rate by column position. A monotonically increasing missingness from a certain point onwards is a classic drop-off curve.

```python
def detect_dropoff(df, type_map):
    rates = df.isnull().mean()
    # Monotonic increase from some position?
    diffs = rates.diff().fillna(0)
    if (diffs > 0.05).sum() >= 3:
        return True, rates
    return False, rates
```

**Presenting**: For skip-logic missingness, do not flag as a quality issue. Note it in the Info tab: "`Q12` was only asked of respondents who answered 'Yes' to `Q11` — 60 % missingness here is expected." For item-nonresponse, flag yellow if > 10 %, red if > 25 %.

Generic missingness alarms (MCAR / MAR / MNAR) are inappropriate for survey data with skip logic — do not report them.

---

## 7. Single-Choice Analysis

**When**: For every `single-choice` column.

Compute the response distribution, modal answer, and the share of respondents giving each option. If the question is ordered (e.g. age bands, frequency), preserve order; else sort by frequency.

```python
def single_choice_summary(series, ordered=False, order=None):
    s = series.dropna()
    counts = s.value_counts()
    pct = (counts / len(s) * 100).round(1)
    if ordered and order:
        counts = counts.reindex(order, fill_value=0)
        pct = pct.reindex(order, fill_value=0)
    return {
        "n": len(s),
        "modal": counts.idxmax(),
        "distribution": pct,
    }
```

**Presenting**: Donut chart (max 6 slices; group long tails into "Other"). Caption: "62 % of respondents say they use the product daily, 24 % weekly, and 14 % less often."

---

## 8. Multi-Select Analysis

**When**: For every `multi-select-binary` battery and `multi-select-string` column.

Convert to a respondent-by-option binary matrix. Report:

- **Option frequency** (% of respondents selecting each option) — base for the chart
- **Mean number of options selected** (engagement signal)
- **Top combinations** (which options co-occur most often)

```python
def multi_select_summary(binary_matrix):
    n = len(binary_matrix)
    option_pct = (binary_matrix.sum(axis=0) / n * 100).round(1).sort_values(ascending=False)
    n_selected = binary_matrix.sum(axis=1)
    # Top combinations
    combo_strings = binary_matrix.apply(
        lambda row: "+".join(sorted(binary_matrix.columns[row.astype(bool)])),
        axis=1,
    )
    top_combos = combo_strings.value_counts().head(5)
    return {
        "n": n,
        "option_pct": option_pct,
        "mean_selected": n_selected.mean(),
        "top_combos": top_combos,
    }
```

**Presenting**: Horizontal bar chart sorted by frequency. Caption: "On average respondents picked 2.3 features. The most-cited combination is 'Speed + Reliability' (chosen by 18 %)."

---

## 9. Open-Text NLP (Lightweight)

**When**: For every `open-text` column with ≥ 30 non-empty responses.

The lightweight pipeline (TF-IDF + clustering + VADER sentiment) is fast and dependency-light. Heavier methods (BERTopic / sentence-transformers) are out of scope unless the user explicitly opts in.

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
import numpy as np

def theme_open_text(series, max_themes=6, min_responses=30):
    s = series.dropna().astype(str).str.strip()
    s = s[s.str.len() > 5]
    if len(s) < min_responses:
        return None  # fall back to word-frequency

    vec = TfidfVectorizer(
        max_features=500,
        stop_words="english",
        ngram_range=(1, 2),
        min_df=2,
    )
    X = vec.fit_transform(s)

    # Pick k by silhouette
    best_k, best_score, best_labels = 2, -1, None
    for k in range(2, min(max_themes, len(s) // 10) + 1):
        km = KMeans(n_clusters=k, random_state=42, n_init=10)
        labels = km.fit_predict(X)
        if len(set(labels)) > 1:
            score = silhouette_score(X, labels, sample_size=min(500, len(s)))
            if score > best_score:
                best_k, best_score, best_labels = k, score, labels

    # Per-theme: top terms, exemplar quotes, sentiment
    analyzer = SentimentIntensityAnalyzer()
    feature_names = np.array(vec.get_feature_names_out())
    themes = []
    km_final = KMeans(n_clusters=best_k, random_state=42, n_init=10).fit(X)
    for i in range(best_k):
        mask = best_labels == i
        center = km_final.cluster_centers_[i]
        top_terms = feature_names[np.argsort(-center)[:6]].tolist()
        # Closest responses to centroid
        cluster_X = X[mask]
        if cluster_X.shape[0] == 0:
            continue
        distances = np.linalg.norm(cluster_X.toarray() - center, axis=1)
        cluster_responses = s[mask].reset_index(drop=True)
        exemplars = cluster_responses.iloc[np.argsort(distances)[:3]].tolist()
        # Sentiment
        sentiments = [analyzer.polarity_scores(r)["compound"] for r in cluster_responses]
        themes.append({
            "id": i,
            "size": int(mask.sum()),
            "top_terms": top_terms,
            "exemplars": exemplars,
            "mean_sentiment": float(np.mean(sentiments)) if sentiments else 0.0,
        })
    return {"themes": themes, "n_responses": len(s)}
```

After clustering, **Claude should rewrite the top-terms into a human theme name** ("Pricing concerns", "Onboarding friction", "Praise for support"). Don't show raw cluster labels to the user.

**Fallback for small samples (< 30 responses)**: skip clustering. Show top 20 most frequent meaningful words, overall sentiment, and a sample of 5 randomly-chosen quotes.

**Presenting**: One card per theme — theme name, size (% of responses), 2–3 exemplar quotes in italic blockquotes, sentiment chip (positive / negative / mixed). Avoid technical terms like "TF-IDF" or "silhouette".

---

## 10. Demographics & Cross-Tabs

**When**: At least one demographic column exists and a KPI exists.

For each demographic (cap at 5 most informative), break the KPI by demographic values. Use the appropriate statistical test (§ 14).

```python
def crosstab_kpi_by_demo(df, kpi_col, kpi_type, demo_col):
    sub = df[[kpi_col, demo_col]].dropna()
    if kpi_type in ("likert-5", "likert-7", "csat"):
        # Top-2-box per demographic
        if pd.api.types.is_numeric_dtype(sub[kpi_col]):
            top_thresh = sub[kpi_col].max() - 1
            sub["top_box"] = sub[kpi_col] >= top_thresh
        else:
            top_2 = LIKERT_5_ORDER_AGREE[-2:]
            sub["top_box"] = sub[kpi_col].astype(str).str.lower().isin(top_2)
        result = sub.groupby(demo_col)["top_box"].agg(["mean", "count"]).rename(
            columns={"mean": "top_2_box_pct", "count": "n"}
        )
        result["top_2_box_pct"] = result["top_2_box_pct"] * 100
        return result.sort_values("top_2_box_pct", ascending=False)
    if kpi_type == "nps":
        # NPS per demographic
        results = {}
        for val, group in sub.groupby(demo_col):
            res = nps_score(group[kpi_col])
            if res:
                res["group"] = val
                results[val] = res
        return pd.DataFrame(results).T
    return sub.groupby(demo_col)[kpi_col].agg(["mean", "count"])
```

**Surface only meaningful gaps**: ≥ 5 percentage points on top-2-box, or ≥ 10 points on NPS, AND statistically significant (p < 0.05). Skip the rest.

**Presenting**: Side-by-side bar chart, sorted descending. Caption: "Enterprise customers rate onboarding 12 points higher than SMBs (84 % vs. 72 % top-2-box, n=210, n=180)."

---

## 11. Driver Analysis

**When**: A KPI exists and ≥ 5 Likert items also exist as candidate drivers.

Driver analysis answers: **which questions are most strongly associated with the KPI?**

**Method options**:

- **Pearson / Spearman correlation**: simple, interpretable, but only marginal effects.
- **Random Forest feature importance**: captures non-linearity and interactions.
- **Relative weights / dominance analysis** (preferred for interpretability): decomposes R² into per-predictor contribution. Robust to multicollinearity.

```python
from sklearn.ensemble import RandomForestRegressor, RandomForestClassifier
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import StandardScaler
import numpy as np

def driver_analysis(X_likert, y_kpi, kpi_type="numeric"):
    # Coerce Likert text to numeric if needed
    X = X_likert.apply(lambda c: pd.to_numeric(c, errors="coerce") if not pd.api.types.is_numeric_dtype(c) else c)
    X = X.fillna(X.median())
    y = pd.to_numeric(y_kpi, errors="coerce")
    mask = y.notna()
    X, y = X[mask], y[mask]

    # Spearman correlations
    spearman = X.apply(lambda c: c.corr(y, method="spearman")).abs().sort_values(ascending=False)

    # Random Forest importance
    if kpi_type in ("nps", "numeric", "csat", "likert-5", "likert-7", "ces"):
        rf = RandomForestRegressor(n_estimators=200, random_state=42, max_depth=6, min_samples_leaf=20)
    else:
        rf = RandomForestClassifier(n_estimators=200, random_state=42, max_depth=6, min_samples_leaf=20)
    rf.fit(X, y)
    rf_imp = pd.Series(rf.feature_importances_, index=X.columns).sort_values(ascending=False)

    # Relative weights (Johnson) — orthogonalised predictors
    X_std = StandardScaler().fit_transform(X)
    y_std = (y - y.mean()) / y.std()
    R = np.corrcoef(X_std.T)
    eigvals, eigvecs = np.linalg.eigh(R)
    eigvals[eigvals < 1e-9] = 1e-9
    delta = np.diag(np.sqrt(eigvals))
    lambda_mat = eigvecs @ delta @ eigvecs.T
    beta = np.linalg.lstsq(lambda_mat, X_std.T @ y_std / len(y_std), rcond=None)[0]
    rel_weights = (lambda_mat ** 2) @ (beta ** 2)
    rel_weights = pd.Series(rel_weights, index=X.columns).sort_values(ascending=False)
    rel_weights_pct = rel_weights / rel_weights.sum() * 100

    return pd.DataFrame({
        "spearman": spearman,
        "rf_importance": rf_imp,
        "relative_weight_pct": rel_weights_pct,
    }).sort_values("relative_weight_pct", ascending=False)
```

**Use relative weights as the primary driver chart** (sums to 100 %, easy to read). Cross-check with Random Forest — if rankings diverge sharply, mention it.

**Presenting**: Horizontal bar chart of top 8 drivers, labelled with the question text (truncated). Caption: "Onboarding clarity is the single biggest driver of recommend rate — it accounts for 22 % of why your NPS is what it is. Pricing perception is second at 14 %."

---

## 12. Construct Reliability (Cronbach's Alpha)

**When**: A matrix battery (§ 2) is being collapsed into a composite score for driver analysis or segmentation.

Cronbach's alpha measures whether the items in a battery measure one underlying construct.

```python
def cronbachs_alpha(df_items):
    items = df_items.apply(lambda c: pd.to_numeric(c, errors="coerce")).dropna()
    k = items.shape[1]
    if k < 2 or len(items) < 10:
        return None
    item_vars = items.var(axis=0, ddof=1)
    total_var = items.sum(axis=1).var(ddof=1)
    return (k / (k - 1)) * (1 - item_vars.sum() / total_var)
```

**Interpretation**:

- α ≥ 0.9 → excellent (may indicate redundancy)
- α 0.8–0.9 → good
- α 0.7–0.8 → acceptable
- α 0.6–0.7 → questionable, use with caution
- α < 0.6 → unreliable, do not collapse to composite

If α is low, present each item separately rather than as a composite.

**Presenting (in user-facing text)**: "These 5 questions measure roughly the same thing — they hang together well — so I've combined them into a single 'satisfaction' score." Don't mention "alpha" or "Cronbach" by name unless the user is technical.

---

## 13. Attitudinal Segmentation

**When**: ≥ 100 respondents, ≥ 5 Likert items.

Attitudinal segmentation finds groups of respondents with similar response patterns on opinion / attitude questions — far more useful than demographic segmentation for understanding behaviour.

```python
from sklearn.cluster import KMeans
from sklearn.mixture import GaussianMixture
from sklearn.metrics import silhouette_score
from sklearn.preprocessing import StandardScaler

def attitudinal_segments(df_likert, max_k=6):
    X = df_likert.apply(lambda c: pd.to_numeric(c, errors="coerce")).fillna(df_likert.apply(
        lambda c: pd.to_numeric(c, errors="coerce")
    ).median())
    X_scaled = StandardScaler().fit_transform(X)

    best_k, best_score = 3, -1
    for k in range(3, max_k + 1):
        km = KMeans(n_clusters=k, random_state=42, n_init=10)
        labels = km.fit_predict(X_scaled)
        score = silhouette_score(X_scaled, labels, sample_size=min(500, len(X)))
        if score > best_score:
            best_k, best_score = k, score

    final = KMeans(n_clusters=best_k, random_state=42, n_init=10).fit(X_scaled)
    seg_labels = final.predict(X_scaled)
    profile = pd.DataFrame(X).assign(segment=seg_labels).groupby("segment").mean()
    sizes = pd.Series(seg_labels).value_counts().sort_index()
    return {
        "labels": seg_labels,
        "profile_means": profile,
        "sizes": sizes,
        "silhouette": best_score,
        "k": best_k,
    }
```

**Naming segments**: for each segment, look at which items are most distinctive (largest gap from grand mean). Combine with the KPI (NPS / CSAT) per segment to coin a name.

Examples of useful names: "Engaged advocates", "Frustrated power users", "Lukewarm passives", "Price-sensitive sceptics", "New and curious".

**Cross with demographics**: for each segment, show its demographic composition (% enterprise, % SMB, age distribution). Often surprises emerge — e.g. "Frustrated power users" might be disproportionately the most senior buyers.

**Presenting**: A segment card per cluster: name, size (%), KPI score, top 2 distinguishing items, demographic skew, exemplar verbatim quote (if open-text exists). Don't show silhouette scores or cluster centroids in user-facing text.

---

## 14. Statistical Significance for Surveys

**When**: Whenever you compare groups.

### Comparing top-2-box across two groups

**z-test for proportions** is the right test (not t-test on means):

```python
from statsmodels.stats.proportion import proportions_ztest

def two_proportion_test(success_a, n_a, success_b, n_b):
    counts = np.array([success_a, success_b])
    nobs = np.array([n_a, n_b])
    stat, p = proportions_ztest(counts, nobs)
    p_a = success_a / n_a
    p_b = success_b / n_b
    diff_pp = (p_a - p_b) * 100
    return {"diff_pp": diff_pp, "p_value": p}
```

### Comparing distributions across 3+ groups

**Chi-squared test of independence**:

```python
from scipy.stats import chi2_contingency

def chi_squared_distribution(series, group):
    ct = pd.crosstab(series, group)
    chi2, p, dof, expected = chi2_contingency(ct)
    n = ct.sum().sum()
    cramers_v = np.sqrt(chi2 / (n * (min(ct.shape) - 1)))
    return {"chi2": chi2, "p": p, "cramers_v": cramers_v}
```

### Comparing mean Likert across groups

**Use with the ordinal caveat**. Kruskal-Wallis is more defensible than ANOVA for ordinal data:

```python
from scipy.stats import kruskal, mannwhitneyu

def likert_groups_test(series, group):
    groups = [g.dropna() for _, g in series.groupby(group)]
    if len(groups) == 2:
        stat, p = mannwhitneyu(groups[0], groups[1])
    else:
        stat, p = kruskal(*groups)
    return {"stat": stat, "p": p}
```

### NPS comparison

NPS is a difference of proportions; use the same z-test logic on (promoters - detractors).

**Presenting**: Translate. "p < 0.05" → "this gap is real, not noise". "Cramér's V = 0.18" → "a meaningful but moderate connection". Never show raw p-values in user-facing prose unless asked.

---

## 15. Trend & Longitudinal Analysis

**When**: A datetime column or wave column exists with at least 3 time points.

Surveys are often run multiple times. Track the KPI and key Likert items across waves:

```python
def trend_by_wave(df, wave_col, kpi_col, kpi_type):
    if kpi_type == "nps":
        return df.groupby(wave_col).apply(lambda g: nps_score(g[kpi_col])["nps"])
    if kpi_type in ("likert-5", "likert-7", "csat"):
        return df.groupby(wave_col).apply(lambda g: likert_metrics(g[kpi_col])["top_2_box"]) * 100
    return df.groupby(wave_col)[kpi_col].mean()
```

**Compositional shift check**: when the KPI moves between waves, check if the demographic mix also shifted. A "drop in NPS" might just be more SMBs in the latest wave.

```python
def check_compositional_shift(df, wave_col, demo_col):
    return pd.crosstab(df[wave_col], df[demo_col], normalize="index") * 100
```

**Change-point detection**: for waves ≥ 6, fit a piecewise linear trend to detect inflection points.

**Presenting**: Line chart with confidence bands. Caption: "NPS dropped 8 points between Q2 and Q3. About 3 of those points are explained by a higher share of SMB respondents in Q3 — the underlying sentiment dropped 5 points among comparable customers."

---

## 16. HTML Report Template

### Branding

- **Primary**: `#3F8585` (teal — accents, active tabs, links, primary chart series)
- **Light background**: `#FEFBF1` (warm cream — page background)
- **Dark**: `#39403A` (charcoal — text, headings)

### Tab Structure

Reports have **up to five** tabs. Skip any tab that isn't relevant (e.g. no Segments tab if segmentation was skipped):

1. **Insights** (default): KPI summary card, headline findings, driver bar chart, top demographic gaps, segment overview, key takeaways. Capped — does not include every question.
2. **Questions**: collapsible list, one card per question with the right visualisation per type. Group cards by section if detected.
3. **Segments**: attitudinal segment profiles — sizes, KPI per segment, distinguishing Likert items, demographic composition, exemplar verbatim per segment.
4. **Data**: health check table (traffic-light per column with skip-logic interpretation), response quality flags (straight-liners, speeders, completion), missingness map.
5. **Info**: report metadata, codebook view (column → detected type → question text → response options), techniques applied, raw data sample (first 20 rows).

### Question-Card Layouts

For the **Questions** tab, render one card per question. Layout depends on detected type:

- **Likert** → stacked horizontal bar (B2B red, neutral grey, T2B teal) with T2B percentage callout. Show n, mean (with ordinal caveat in tooltip).
- **NPS** → stacked horizontal bar (red detractors, grey passives, green promoters) with NPS score callout. Show n and 95 % CI.
- **CSAT / CES** → vertical bar of distribution with top-2-box callout.
- **Single-choice** → donut (cap to 6 slices, group rest as "Other"). Show n.
- **Multi-select** → horizontal bar of option frequencies, sorted descending. Show mean number of options selected.
- **Open-text** → theme cards (one card per theme): theme name, size %, sentiment chip, 2–3 exemplar quotes in `<blockquote>`.
- **Demographic** → small donut or bar in the Info tab only (not in Questions).
- **Numeric** → histogram with mean / median callout.

### Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Insight Badger — [Report Title]</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>
  <style>
    :root {
      --primary: #3F8585;
      --primary-light: #e8f4f4;
      --primary-dark: #2d6363;
      --bg: #FEFBF1;
      --card: #ffffff;
      --text: #39403A;
      --muted: #6b726c;
      --green: #2d8659;
      --yellow: #b8860b;
      --red: #c0392b;
      --border: #d4d1c7;
    }

    * { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
      background: var(--bg);
      color: var(--text);
      line-height: 1.6;
    }

    /* ── Header ── */
    .report-header {
      background: var(--text);
      color: var(--bg);
      padding: 1.5rem 2rem;
      display: flex;
      align-items: center;
      gap: 1rem;
    }
    .report-header h1 {
      font-size: 1.4rem;
      font-weight: 600;
      color: var(--bg);
      margin: 0;
      border: none;
    }
    .report-header .subtitle {
      font-size: 0.85rem;
      color: #a0a59a;
      margin-top: 0.15rem;
    }

    /* ── Tabs ── */
    .tab-bar {
      display: flex;
      background: var(--text);
      padding: 0 2rem;
      gap: 0;
      border-bottom: 3px solid var(--primary);
      overflow-x: auto;
    }
    .tab-btn {
      padding: 0.7rem 1.5rem;
      background: none;
      border: none;
      color: #a0a59a;
      font-size: 0.95rem;
      font-weight: 500;
      cursor: pointer;
      border-bottom: 3px solid transparent;
      margin-bottom: -3px;
      transition: color 0.2s, border-color 0.2s;
      white-space: nowrap;
    }
    .tab-btn:hover { color: var(--bg); }
    .tab-btn.active {
      color: var(--bg);
      border-bottom-color: var(--primary);
      font-weight: 600;
    }

    /* ── Tab content ── */
    .tab-content { display: none; padding: 2rem; max-width: 1100px; margin: 0 auto; }
    .tab-content.active { display: block; }

    /* ── Cards ── */
    .card {
      background: var(--card);
      border-radius: 8px;
      padding: 1.5rem;
      margin-bottom: 1.5rem;
      box-shadow: 0 1px 3px rgba(57,64,58,0.08);
      border: 1px solid var(--border);
    }
    .card-compact { padding: 1rem 1.25rem; }

    h2 { color: var(--text); font-size: 1.15rem; margin-bottom: 0.75rem; }
    h3 { color: var(--primary-dark); font-size: 1rem; margin-bottom: 0.5rem; }

    /* ── KPI hero ── */
    .kpi-hero {
      display: flex;
      align-items: baseline;
      gap: 1rem;
      flex-wrap: wrap;
      margin-bottom: 0.75rem;
    }
    .kpi-hero .number {
      font-size: 3rem;
      font-weight: 700;
      color: var(--primary);
      line-height: 1;
    }
    .kpi-hero .label { color: var(--muted); font-size: 0.95rem; }
    .kpi-hero .ci { color: var(--muted); font-size: 0.85rem; }

    /* ── Tables ── */
    table { width: 100%; border-collapse: collapse; margin: 0.75rem 0; }
    th, td {
      padding: 0.5rem 0.75rem;
      border-bottom: 1px solid var(--border);
      text-align: left;
      font-size: 0.9rem;
    }
    th { background: var(--bg); font-weight: 600; color: var(--text); }

    /* ── Badges ── */
    .badge {
      display: inline-block;
      padding: 2px 10px;
      border-radius: 4px;
      font-size: 0.8rem;
      font-weight: 600;
    }
    .badge-green { background: #e6f5ec; color: var(--green); }
    .badge-yellow { background: #fff8e1; color: var(--yellow); }
    .badge-red { background: #fde8e8; color: var(--red); }
    .badge-neutral { background: #ebeeec; color: var(--muted); }

    /* ── Charts ── */
    .chart-container { position: relative; height: 300px; margin: 1rem 0; }
    .chart-container.short { height: 200px; }
    .caption { font-size: 0.85rem; color: var(--muted); margin-top: 0.25rem; }

    /* ── Callouts ── */
    .takeaway {
      background: var(--primary-light);
      border-left: 4px solid var(--primary);
      padding: 1rem 1.25rem;
      border-radius: 0 6px 6px 0;
      margin: 0.75rem 0;
    }
    .callout-warn {
      background: #fff8e1;
      border-left: 4px solid #d4a017;
      padding: 1rem 1.25rem;
      border-radius: 0 6px 6px 0;
      margin: 0.75rem 0;
    }

    /* ── Question cards (Questions tab) ── */
    .q-section-title {
      font-size: 0.8rem;
      font-weight: 700;
      text-transform: uppercase;
      letter-spacing: 0.05em;
      color: var(--muted);
      margin: 1.5rem 0 0.5rem 0;
    }
    .q-card .q-text { font-weight: 600; color: var(--text); margin-bottom: 0.5rem; }
    .q-card .q-meta { font-size: 0.8rem; color: var(--muted); margin-bottom: 0.75rem; }

    /* ── Theme cards (open-text) ── */
    .theme-card {
      background: var(--card);
      border-radius: 6px;
      padding: 1rem 1.25rem;
      margin-bottom: 0.75rem;
      border: 1px solid var(--border);
    }
    .theme-card blockquote {
      border-left: 3px solid var(--primary);
      padding: 0.25rem 0 0.25rem 0.75rem;
      margin: 0.5rem 0;
      color: var(--text);
      font-style: italic;
      font-size: 0.9rem;
    }

    /* ── Segment cards (Segments tab) ── */
    .segment-card {
      background: var(--card);
      border-radius: 8px;
      padding: 1.25rem 1.5rem;
      margin-bottom: 1rem;
      border: 1px solid var(--border);
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 1rem;
    }
    .segment-card .seg-name {
      font-size: 1.1rem;
      font-weight: 700;
      color: var(--primary-dark);
      grid-column: 1 / -1;
    }

    /* ── Info list ── */
    .info-grid {
      display: grid;
      grid-template-columns: 160px 1fr;
      gap: 0.4rem 1rem;
      font-size: 0.9rem;
    }
    .info-grid dt { font-weight: 600; color: var(--muted); }
    .info-grid dd { color: var(--text); }

    /* ── Chart.js color palette ──
       Primary series: #3F8585
       Secondary: #39403A
       Tertiary: #6db3b3
       Quaternary: #8fcccc
       Accent warm: #c9a227
       Detractor red: #c0392b
       Promoter green: #2d8659
       Background: #FEFBF1
    */
  </style>
</head>
<body>

  <!-- ── Header ── -->
  <div class="report-header">
    <div>
      <h1>Insight Badger — [Report Title]</h1>
      <div class="subtitle">Generated on [date] · [n] respondents · [m] questions</div>
    </div>
  </div>

  <!-- ── Tab bar ── -->
  <div class="tab-bar">
    <button class="tab-btn active" onclick="switchTab('insights', this)">Insights</button>
    <button class="tab-btn" onclick="switchTab('questions', this)">Questions</button>
    <button class="tab-btn" onclick="switchTab('segments', this)">Segments</button>
    <button class="tab-btn" onclick="switchTab('data', this)">Data</button>
    <button class="tab-btn" onclick="switchTab('info', this)">Info</button>
  </div>

  <!-- ══════════ TAB: Insights ══════════ -->
  <div id="tab-insights" class="tab-content active">
    <!-- KPI hero card -->
    <div class="card">
      <div class="kpi-hero">
        <span class="number">[KPI]</span>
        <span class="label">[KPI label, e.g. "Net Promoter Score"]</span>
        <span class="ci">95 % CI: [low]–[high] · n = [N]</span>
      </div>
      <div class="caption">[One-sentence interpretation]</div>
    </div>

    <!-- Key Takeaways -->
    <div class="card">
      <h2>Key Takeaways</h2>
      <!-- .takeaway divs -->
    </div>

    <!-- Top drivers -->
    <div class="card">
      <h2>What drives [KPI]</h2>
      <div class="chart-container"><canvas id="chart-drivers"></canvas></div>
      <div class="caption">[Plain explanation of the chart]</div>
    </div>

    <!-- Top demographic gaps -->
    <div class="card">
      <h2>How groups differ</h2>
      <!-- .takeaway divs with gap descriptions -->
    </div>

    <!-- Segment overview (if segmentation ran) -->
    <div class="card">
      <h2>Attitudinal segments</h2>
      <!-- summary table or 2-3 sentence overview, link to Segments tab for detail -->
    </div>
  </div>

  <!-- ══════════ TAB: Questions ══════════ -->
  <div id="tab-questions" class="tab-content">
    <div class="q-section-title">Section 1 · [Section name]</div>
    <!-- One .card.q-card per question, with type-appropriate visualisation -->
  </div>

  <!-- ══════════ TAB: Segments ══════════ -->
  <div id="tab-segments" class="tab-content">
    <!-- One .segment-card per cluster -->
    <!--
      Inside a segment card:
        .seg-name
        Left column: KPI score, distinguishing items list, exemplar verbatim
        Right column: small donut for demographic composition
    -->
  </div>

  <!-- ══════════ TAB: Data ══════════ -->
  <div id="tab-data" class="tab-content">
    <div class="card">
      <h2>Survey Health Check</h2>
      <table>
        <thead><tr><th>Column</th><th>Type</th><th>Completeness</th><th>Notes</th><th>Status</th></tr></thead>
        <tbody><!-- one row per column with traffic-light badge --></tbody>
      </table>
    </div>

    <div class="card">
      <h2>Response Quality</h2>
      <!-- straight-liners, speeders, completion rate, drop-off curve if relevant -->
    </div>

    <div class="card">
      <h2>Skip-Logic Map</h2>
      <!-- columns whose missingness is explained by routing rules -->
    </div>
  </div>

  <!-- ══════════ TAB: Info ══════════ -->
  <div id="tab-info" class="tab-content">
    <div class="card">
      <h2>Report Information</h2>
      <dl class="info-grid">
        <dt>File</dt>           <dd>[filename]</dd>
        <dt>Codebook</dt>       <dd>[filename or "inferred from headers"]</dd>
        <dt>Respondents</dt>    <dd>[N]</dd>
        <dt>Questions</dt>      <dd>[M]</dd>
        <dt>Generated</dt>      <dd>[date and time]</dd>
        <dt>Analysis Mode</dt>  <dd>[dig-deep or dig-for]</dd>
      </dl>
    </div>

    <div class="card">
      <h2>Codebook</h2>
      <table>
        <thead><tr><th>Column</th><th>Type</th><th>Question Text</th><th>Response Options</th></tr></thead>
        <tbody><!-- one row per column --></tbody>
      </table>
    </div>

    <div class="card">
      <h2>Techniques Applied</h2>
      <!-- bulleted list -->
    </div>

    <div class="card">
      <h2>Raw Data Sample</h2>
      <div style="overflow-x: auto; max-height: 400px; overflow-y: auto;">
        <!-- first 20 rows -->
      </div>
    </div>
  </div>

  <script>
    function switchTab(tabName, btn) {
      var contents = document.querySelectorAll('.tab-content');
      for (var i = 0; i < contents.length; i++) { contents[i].classList.remove('active'); }
      var buttons = document.querySelectorAll('.tab-btn');
      for (var i = 0; i < buttons.length; i++) { buttons[i].classList.remove('active'); }
      document.getElementById('tab-' + tabName).classList.add('active');
      btn.classList.add('active');
    }
  </script>

</body>
</html>
```

### Chart Color Palette

```javascript
const COLORS = {
  primary:    '#3F8585',  // main teal — Likert top-box, NPS in aggregate
  dark:       '#39403A',  // secondary text
  tertiary:   '#6db3b3',
  quaternary: '#8fcccc',
  warm:       '#c9a227',
  detractor:  '#c0392b',  // NPS detractors, B2B
  passive:    '#a0a59a',  // NPS passives, neutral
  promoter:   '#2d8659',  // NPS promoters, T2B
  bg:         '#FEFBF1',
  gridLines:  '#d4d1c7'
};
```

### Adapting the Template

- Populate only the sections relevant to the analysis performed. Empty cards must be omitted.
- The **Insights** tab is always the default active tab.
- Skip the **Segments** tab entirely if segmentation didn't run (don't show an empty tab).
- Skip the **Questions** tab in `dig-for` reports unless multiple questions are central to the analysis.
- Every chart needs a clear plain-language title and a one-sentence `.caption` explaining what it shows and why it matters.
- Keep the Data and Info tabs factual and compact — the Insights tab is where the narrative lives.
- **No emoji anywhere** — not in headings, badges, captions, takeaways, or any generated text. Plain text everywhere.

---

## 17. Generic Fallback (only when needed)

For columns that don't match any survey question type (e.g. a numeric measurement column in a hybrid dataset), fall back to:

- **Numeric**: descriptive stats (mean, median, std, percentiles), histogram, simple correlation with the KPI.
- **Datetime**: range, gaps, frequency cadence.
- **Unknown**: report sample values in the Info tab and skip from analysis.

Generic methods (PCA, MCAR/MAR/MNAR analysis, Isolation Forest, Mahalanobis distance, Simpson's paradox detection) are **not appropriate for survey data** by default and should not be applied unless the user explicitly requests them. They were designed for continuous measurement data and produce misleading results on Likert / categorical responses.
