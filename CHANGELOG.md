# Changelog

## 0.2.5

A new report design and more reliable analysis instructions.

### What changes for you

- A new report design: a headline finding, key figures, numbered chapter tabs, a story map of the findings,
  recommended actions and next questions, with colour that means something (teal good, clay problem, ochre watch,
  slate comparison) and the badger logo.
- New charts: a driver matrix with "fix first" and "protect" zones, group scores with likely ranges, two-group
  comparisons, diverging rating bars, personas, comment themes and quote cards.
- Reports work offline and make no network requests (no Chart.js download, system fonts), and tabs work inside app
  previews.
- Ask for a PDF, branded or unbranded, or print the report from a browser: it prints with a cover page and page
  numbers. Custom branding with `.insight-badger.json`.
- Working files are saved in an `insight-badger/<file name>/` folder next to the survey.

### Fixes in the method instructions

- Qualtrics' three header rows and SurveyMonkey's two are read correctly; Google Forms grids and checkboxes too.
- Whole numbers are not read as dates; 7-point and 1–10 scales are not read as NPS; "Neither agree nor disagree",
  "Don't know" and "Not applicable" are handled; thousands separators are not mistaken for multi-select answers.
- Correct NPS likely range; group gaps adjusted for the number of comparisons; groups under 30 never called real.
- Speeders measured against the median time; routing, drop-off and optional comment boxes not counted as problems;
  multi-select blanks mean "not selected".
- Drivers leave out other overall questions; comment themes are coded by Claude in any language and counted in code.
- Personal data never shown and removed from quotes; no raw data rows in the report.

## 0.2.0

- Initial standalone release with `/dig-deep` and `/dig-for`.
