<p align="center"><img src="assets/logo.png" width="120" alt="Insight Badger logo"></p>

# Insight Badger

Survey insights for people who don't analyse surveys for a living. Drop a survey export from Qualtrics,
SurveyMonkey, Google Forms, Typeform, Microsoft Forms, SPSS or any CSV/XLSX, and get a plain-language,
storytelling report: what drives your score, who is unhappy, what people say in their own words, and what to do
next.

No spreadsheet formulas, no SPSS, no coding.

## Skills

### `/dig-deep`: explore a survey

Hand over a file and Insight Badger works out the rest: the questionnaire structure, the KPI, what drives it, which
groups differ, the customer mindsets in your data and the themes in open comments.

Try: *"dig deep into survey.csv"*, *"what's interesting in these responses?"*

### `/dig-for`: answer one question

You have a question; Insight Badger asks one or two clarifying questions, then delivers a focused brief that leads
with the answer.

Try: *"what's driving our NPS?"*, *"why are small customers less happy?"*, *"compare enterprise vs SMB on
onboarding"*, *"what do detractors complain about?"*

## The report

Every run produces one self-contained HTML file that works offline:

- **Header** with the headline finding and the key figures (score, likely range, change since last wave, completion).
- **Overview / Answer** tab: the answer in two or three sentences, a story map of the chapters, recommended actions
  and the next questions to dig into.
- **Chapter tabs**: drivers (importance vs performance, with "fix first" and "protect" zones), group gaps with
  likely ranges, two-group comparisons, customer mindsets as personas, comment themes with quotes.
- **Reference tabs**: every question as a card, data quality and plain-language methods.
- **Colour means something**: teal for strengths, clay for problems, ochre for "watch", slate for comparisons.

Open the report in a web browser for the best view; app previews may show it as one long page.

### PDF, branded or unbranded

Ask for a PDF ("…and give me an unbranded PDF"), or open the report in a browser and print it (Cmd/Ctrl+P, then
Save as PDF). The PDF has a cover page, one chapter per page and page numbers.

- **Insight Badger branded**: logo on the cover, credit in the footer.
- **Unbranded**: no logo or product name, ready to send as your own.

### Your own brand

Put a `.insight-badger.json` file in your project folder (or any parent folder):

```json
{
  "brand": "custom",
  "name": "Acme Research",
  "logo": "brand/acme-logo.png",
  "primary": "#1F5FAD",
  "paper": "Letter",
  "confidential": true
}
```

`brand` can be `insight-badger` (default), `none` or `custom`.

## What it handles

- **Exports from every major platform**, including Qualtrics' three header rows and SurveyMonkey's two, Google
  Forms grids and checkboxes, XLSX and SPSS, plus an optional codebook.
- **Question types**: NPS, agreement/satisfaction/frequency/quality scales (5- or 7-point, text or numbers), CSAT,
  CES, numeric ratings, single choice, multi-select, open text, numbers and demographics.
- **"Don't know" and "Not applicable"** are left out of percentages.
- **Data quality**: completion, speeders, straight-liners, routing (skip logic) and drop-off, in plain words.
- **Sound statistics**: correct NPS likely ranges, survey weights, gaps adjusted for the number of comparisons,
  groups under 30 people never called significant.
- **Drivers**, **customer mindsets** and **comment themes** coded by Claude in any language.

## What this plugin runs and sends

- **What it runs**: Claude writes and runs a short Python analysis for your file (pandas, numpy, scipy,
  scikit-learn) and saves its working files and the report in `insight-badger/<file name>/` next to the survey. You
  approve these steps as usual. If a package is missing, Claude asks before installing it.
- **PDF**: when you ask for a PDF, Claude asks before printing the report with your local Chrome or Edge.
- **Network**: none. Reports use inline charts and system fonts and fetch nothing from the internet.
- **Personal data**: names, emails, phone numbers and IP addresses never appear in reports, and are removed from
  quotes. No raw data rows appear in reports.
- **Credentials**: none are read or needed.

## Licence

Apache 2.0
