# Report design and template

Every report is one self-contained HTML file that works offline: inline CSS and
SVG charts, system fonts, no external files, no network requests.

**Build the HTML with a Python script (`build_report.py`) that reads
`results.json`.** Never type a number into the report by hand: the words come
from you, the numbers from the data.

---

## 1. Writing rules

- **Titles state findings.** "Onboarding matters most, and it scores lowest",
  not "Driver analysis". The report title is the single most important finding.
- **Plain language.** Write for someone who has never analysed a survey.
  "statistically significant" → "real, not noise"; "p < 0.05" → "unlikely to be
  chance"; Cronbach's alpha → "the questions hang together"; top-2-box → "% who
  agree or strongly agree".
- **Emphasis carries meaning.** Wrap the problem phrase in
  `<em class="hl-problem">` (clay) and a strength in `<em class="hl-good">`
  (brand colour). At most one of each per sentence.
- **Respect the verdicts** (methods.md §6). Only `real` and `probably real` gaps
  are findings; never report `chance`; mention `small group` only as a caution.
- No emoji. No benchmarks unless the user supplied them.
- Answer: 2–3 sentences. Chapter lede: 1–2 sentences. "So what": one sentence
  about what to do or watch.
- Escape every piece of survey text you insert (`html.escape`).

## 2. Colour means something

| Colour | Meaning | Used for |
|---|---|---|
| Brand teal | brand, good | lead KPI tile, positive answers, strengths, "Protect" |
| Clay | problem to act on | drops, "Fix first", the struggling group, "Act now", negative answers |
| Ochre | watch, uncertain | "Probably real" / "Watch" labels, "Plan next", data-quality flags |
| Slate | comparison, explore further | the comparison group, the dig-for question box, next questions |

Grey everything that is not the point; colour only what the reader should look at.

## 3. Page structure

```
top bar      brand (logo + name) on the left · "Generated on <date>" on the right
hero         mode label · [dig-for: "You asked" + the question] · title · one-line subtitle
             [dig-for: scope chips] · key-figure block (up to 4 tiles, one solid "lead" tile)
tab bar      first tab · divider · numbered chapter tabs · reference tabs on the right
panels       one per tab; previous / next buttons at the bottom
footer       brand · "Generated with Insight Badger · <date> · <file name>"
print only   cover page (brand, title, survey, date, logo) and the key figures
```

**dig-deep tabs:** Overview, then 3–5 numbered chapters (typically Drivers,
Groups, Mindsets, Comments, All ratings), then **All questions** and
**Data & method**.

- *Overview*: the answer; the NPS composition bar (if NPS); "The story in N
  chapters" story map (one card per chapter: tag, chapter number, title, one-line
  summary, "Open …"); "What to do about it" (actions tagged Act now / Plan next /
  Protect) next to "Keep digging" (2–3 follow-up questions with `/dig-for`).
- *Chapters*: kicker ("Chapter 1 · Drivers"), finding title, lede, figures, "So what".
- *All questions*: one card per question; rating batteries as one diverging chart.
- *Data & method*: quality chips, a table of how completely each question was
  answered (with routing notes), methods in plain words, a collapsible codebook.
  **No raw data rows.**

**dig-for tabs:** Answer (answer, 1–3 clay "reason" cards with their number, a
teal "Not the cause" note, actions, next questions), 1–3 numbered chapters
(Evidence, In their words), **How sure we are** (Solid / Watch caveats), and
**Data & method**. The hero uses the slate variant (`class="hero for"`).

Mark a chapter tab whose finding is a problem with `<span class="flag">` (a clay
dot), and add a count badge where useful (`<span class="ct">286</span>`).

## 4. Charts

All charts are inline SVG or HTML drawn to one scale, inside
`<div class="figure"><div class="ftitle">…</div><div class="fsub">…</div> … <div class="src">…</div></div>`.
Every chart title says what it shows. Every SVG gets `role="img"` and an
`aria-label` listing the values. Colour SVG text and marks with the classes in
the template (`c-teal`, `d-clay`, …), never hard-coded colours.

**NPS composition** (HTML): `<div class="npsbar">` with three `<span>` widths =
detractors / passives / promoters %, classes `det`, `pas`, `pro`, plus a legend.

**Driver matrix** (SVG, viewBox `0 0 640 410`, plot x 60–610, y 20–350):
x = importance (0 → max rounded up to 10), y = % rated well (range rounded to
10s, at least 30 points wide). Four zone rectangles split at the average
importance and average performance: top-left `z-slate` NICE TO HAVE, top-right
`z-teal` STRENGTHS TO PROTECT, bottom-left `z-ochre` MONITOR, bottom-right `z-clay`
FIX FIRST (zone labels in the matching `c-*` class, `q-lab`). White grid lines
(`grid-w`). Points coloured by zone (`d-clay` r=9 for fix first, others r=6.5) with
labels in the zone colour; put labels left of points in the right third, and
nudge labels 14px apart when they would overlap.

**Groups with likely ranges** (SVG, width 640, 45px per row, labels at x 10,
plot x 170–620): per group a range line (`ci-teal`, or `ci-clay` for the
highlighted group, `ci-grey` + hollow dot `d-hollow` for groups under 30), a dot,
the value above it, the label and "n = …" on the left. Highlight the group the
story is about with a `pill-clay` row background. A dashed `ref` line marks the
overall score with an ochre "overall …" label. Ticks at nice steps.

**Two-group comparison / dumbbell** (SVG, width 640, 40px per row, labels at x 0,
plot x 190–540 for 0–100%): per item a line between group A (`d-clay`) and group B
(`d-slate`) with the percentages outside the dots and the gap in a `pill-clay`
badge on the right. Gaps that could be chance: grey line (`lk-grey`), hollow dots,
muted labels and the word "chance". Sort by gap.

**Diverging ratings** (HTML): rows `lk-row` = label · track · top-box %. The track
has a centre line at `neg_max / (neg_max + pos_max)`; disagree segments (`s-sd`,
`s-d`) grow left from it, agree segments (`s-a`, `s-sa`) right. Give `.lk-neg`
`left:0; right:<100-mid>%` and `.lk-pos` `left:<mid>%; right:0`, and size each
segment as a % **of its own half** (value / neg_max or value / pos_max), otherwise
the bars collapse. Neutral answers are not drawn. Highlighted rows get `hl`.

**Themes** (HTML): rows `theme` = name · bar · %. Bar length = share of coded
comments (scaled to the largest), split into `neg` / `mix` / `pos` by tone.

**Personas** (HTML): `<article class="persona">` cards with name, share, size bar,
KPI, "Rate highly", "Rate low", "More often", a quote. Colour classes in order:
none (teal), `c1` (ochre), `c2` (slate), `c3`; the segment that needs action gets
`focus` and a "Needs action" `pflag`.

**Quotes** (HTML): `<figure class="quote neg|pos">` with theme label, quote and who
said it (NPS group · a demographic). Never names or contact details.

**Trend** (SVG): line through wave values with a shaded `band-teal` likely range;
last point `d-clay` when the drop is real.

## 5. Branding

Settings come from a `.insight-badger.json` in the working folder or any parent:

```json
{"brand": "insight-badger | none | custom", "name": "Acme Research", "logo": "brand/logo.png",
 "primary": "#1F5FAD", "paper": "A4 | Letter", "confidential": false}
```

- **insight-badger** (default): `reference/logo.png` (next to this file) as a
  base64 data URI in `--logo`; name "Insight Badger", tagline "Survey insights";
  footer "Generated with Insight Badger".
- **none**: no logo, no product name; the top bar shows the survey name; no credit.
- **custom**: their logo and name; footer "Prepared by <name>". Derive the teal
  family from `primary`: `--teal` = primary, `--teal-deep` = same hue at 72%
  lightness, `--teal-mid` / `--teal-pale` / `--teal-wash` = primary mixed with
  white at 45% / 85% / 92%.

## 6. PDF

The template's print styles add a cover page, one chapter per page and page
numbers. When the user asks for a PDF:

1. Ask before running anything. If Chrome, Chromium or Edge is installed, print
   headless (add `--no-sandbox` only when running as root):
   `<chrome> --headless=new --disable-gpu --no-pdf-header-footer --print-to-pdf=<report>.pdf file://<report>.html`
2. For an unbranded PDF, build a copy of the report with brand `none` and print that.
3. Otherwise, tell the user to open the report in a browser and print it
   (Cmd/Ctrl+P, then Save as PDF).

## 7. Checks before you hand over the report

- Every number in the text matches `results.json`.
- No `http` URLs, no `<link>`, no `<script src>` in the file.
- Tabs are `<button>` elements with `data-tab`; story cards and pager buttons use
  `data-go`. **Never use `href="#…"` links for navigation**: inside app previews
  they replace the report with a blank page.
- The `<html>` tag starts with `class="no-js"` and the inline script switches it to
  `js`, so without scripts every section shows as one long page.
- Open the file once (or take a screenshot if you can) and look at every tab.

---

## 8. Template

### Head and top bar

```html
<!DOCTYPE html><html lang="en" class="no-js"><head><meta charset="utf-8">
<script>document.documentElement.className='js';</script>
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>{plain-text title}</title><meta name="generator" content="Insight Badger">
<style>{CSS below, with __PAPER__ replaced by A4 or letter}
:root{--logo:url('data:image/png;base64,{logo}');{custom colour variables}}</style></head>
<body class="deep-report|for-report[ unbranded][ nologo]" data-default-brand="insight-badger|none|custom">
<div class="appbar wrap screen-only"><div class="inner">
  <div class="brand"><span class="mark" role="img" aria-label="logo"></span>
    <span class="name">Insight Badger<small>Survey insights</small></span></div>
  <span class="stamp num">Generated on 16 Nov 2026</span>
</div></div>
```

### Print cover (shown only when printing)

```html
<section class="cover[ for]">{brand block}[<span class="conf-label">Confidential</span>]
  <div class="band"><span class="kind">Full exploration</span><h1>{title}</h1>
  <p class="meta num">{survey} · {wave} · {n} responses · {date}</p></div><div class="bigmark"></div></section>
<div class="print-only"><div class="stats print-stats num">{stat tiles}</div></div>
```

### Hero

```html
<header class="hero[ for] wrap"><div class="inner"><div class="hero-text">
  <span class="mode">Full exploration · /dig-deep</span>
  <!-- dig-for: <div class="asked"><span class="k">You asked</span><span class="q">&ldquo;…&rdquo;</span></div> -->
  <h1>Customers rely on the product, but <em class="hl-problem">a slow start</em> is costing recommendations</h1>
  <p class="sub">One line on what the report covers.</p>
  <!-- dig-for: <div class="scope"><span>Companies of 1-10 vs 1,000+</span>…</div> -->
</div>
<div class="stats num">
  <div class="stat lead"><span class="k">Net Promoter Score</span><span class="v">−7</span><span class="d">likely −14 to 0</span></div>
  <div class="stat alert"><span class="k">Since 2026 Q2</span><span class="v">−11</span><span class="d">from −1</span></div>
  <div class="stat"><span class="k">Responses</span><span class="v">588</span><span class="d">of 600 in the file</span></div>
  <div class="stat"><span class="k">Finished</span><span class="v">94%</span><span class="d">566 of 600</span></div>
</div></div></header>
```

### Tabs and panels

```html
<nav class="tabbar wrap" aria-label="Report sections"><div class="inner tabs" role="tablist">
  <button type="button" class="tab" role="tab" aria-controls="p-overview" data-tab="overview">Overview</button>
  <span class="divider" aria-hidden="true"></span>
  <button type="button" class="tab" role="tab" aria-controls="p-ch-drivers" data-tab="ch-drivers"><span class="no">1</span>Drivers<span class="flag"></span></button>
  …
  <button type="button" class="tab ref" role="tab" aria-controls="p-questions" data-tab="questions">All questions<span class="ct num">28</span></button>
  <button type="button" class="tab" role="tab" aria-controls="p-data" data-tab="data">Data &amp; method</button>
</div></nav>
<main>
  <section class="panel wrap active" id="p-overview" role="tabpanel"><div class="inner">
    <p class="answer">NPS is <em class="hl-problem">below zero</em>. …</p>
    <div class="head"><span class="kicker">The story in 4 chapters</span></div>
    <div class="storymap">
      <div class="story tone-problem" data-go="ch-drivers" role="link" tabindex="0"><div class="top">
        <span class="tag">Fix first</span><span class="ch">Chapter 1</span></div>
        <h3>Onboarding matters most, and scores lowest</h3><p>One-line summary.</p><span class="go">Open Drivers →</span></div>
    </div>
    <div class="split"><div><h3>What to do about it</h3><div class="actions">
      <div class="action now"><span class="tag">Act now</span>…</div>
      <div class="action next"><span class="tag">Plan next</span>…</div>
      <div class="action protect"><span class="tag">Protect</span>…</div></div></div>
    <div><h3>Keep digging</h3><div class="nextq"><div><span>Why are small companies less happy?</span><code>/dig-for</code></div></div></div></div>
    <div class="pager"><button type="button" class="nx" data-go="ch-drivers"><span>Next</span><b>1 · Drivers →</b></button></div>
  </div></section>
  <section class="panel wrap" id="p-ch-drivers" role="tabpanel"><div class="inner">
    <div class="head"><span class="kicker">Chapter 1 · Drivers</span><h2>…</h2></div>
    <p class="lede">…</p>
    <div class="figure">…</div>
    <div class="sowhat"><b>So what</b>…</div>
    <div class="pager"><button type="button" data-go="overview"><span>Previous</span><b>← Overview</b></button>
      <button type="button" class="nx" data-go="ch-groups"><span>Next</span><b>2 · Groups →</b></button></div>
  </div></section>
  …
</main>
<footer class="foot wrap"><div class="inner">{brand block}<span><span class="credit">Generated with Insight Badger · </span>16 Nov 2026 · survey.csv</span></div></footer>
<script>{JS below}</script></body></html>
```

Tab tones: story cards and tags take `tone-problem`, `tone-strength`, `tone-watch`
or nothing (slate). Kickers: `kicker` (teal), `kicker slate`, `kicker ochre`.
Confidence chips: `<span class="conf">Real, not noise</span>`,
`<span class="conf watch">Probably real</span>`, `<span class="conf grey">Could be chance</span>`.

### CSS

```css
/* Insight Badger report. Light theme only.
   Colour carries meaning: brand teal = brand and good, clay = problem to act on,
   ochre = watch / uncertain, slate = comparison and next steps. */
:root {
  color-scheme: light;
  --bg: #FEFBF1;
  --surface: #FFFFFF;
  --ink: #39403A;
  --muted: #6C736D;
  --line: #E8E3D4;
  --neutral: #D9D4C5;
  --teal: #3F8585;
  --teal-deep: #2B6262;
  --teal-mid: #8DBDB7;
  --teal-pale: #E1EFEB;
  --teal-wash: #EEF5F1;
  --clay: #C4643A;
  --clay-deep: #A04E2A;
  --clay-mid: #E3A385;
  --clay-pale: #F9E7DC;
  --ochre: #C9962E;
  --ochre-deep: #87631A;
  --ochre-pale: #FAF0D5;
  --slate: #5A6E96;
  --slate-deep: #3F5178;
  --slate-pale: #E6EBF4;
  /* System fonts only: reports never fetch anything from the internet. */
  --font-display: "Avenir Next", "Segoe UI Variable Display", "Segoe UI", -apple-system, BlinkMacSystemFont,
                  "Helvetica Neue", Roboto, Arial, sans-serif;
  --font-body: -apple-system, BlinkMacSystemFont, "Segoe UI", Inter, Roboto, "Helvetica Neue", Arial, sans-serif;
}
* { box-sizing: border-box; }
[hidden] { display: none !important; }
html { -webkit-text-size-adjust: 100%; }
body { background: var(--bg); color: var(--ink); font-family: var(--font-body); font-size: 15px; line-height: 1.6; margin: 0; }
h1, h2, h3, h4 { font-family: var(--font-display); text-wrap: balance; margin: 0; line-height: 1.2; color: var(--ink); }
p { margin: 0; }
.num { font-variant-numeric: tabular-nums; }
.wrap { padding-left: max(16px, env(safe-area-inset-left, 0px)); padding-right: max(16px, env(safe-area-inset-right, 0px)); }
.inner { max-width: 1040px; margin: 0 auto; }
a:focus-visible, button:focus-visible, summary:focus-visible { outline: 2px solid var(--teal); outline-offset: 2px; border-radius: 6px; }
em.hl-problem { font-style: normal; color: var(--clay-deep); }
em.hl-good { font-style: normal; color: var(--teal-deep); font-weight: 600; }

/* App bar */
.appbar { background: var(--surface); border-bottom: 1px solid var(--line); padding-block: 12px; }
.appbar .inner { display: flex; align-items: center; gap: 10px 18px; flex-wrap: wrap; justify-content: space-between; }
.brand { display: flex; align-items: center; gap: 9px; font: 600 15px var(--font-display); color: var(--teal-deep); }
.brand .name { display: grid; line-height: 1.15; }
.brand .name small { font: 500 11.5px var(--font-body); color: var(--muted); }
.stamp { font-size: 13px; color: var(--muted); white-space: nowrap; }
.mark { flex: none; width: 38px; height: 38px; background: var(--logo) center / contain no-repeat; }


/* Hero */
.hero { background: var(--teal-wash); padding-block: 30px 26px; border-bottom: 1px solid var(--teal-pale); }
.hero.for { background: var(--slate-pale); border-bottom-color: #D7DFEC; }
.hero .inner { display: grid; grid-template-columns: minmax(0, 1fr) 330px; gap: 28px 40px; align-items: end; }
.hero.nostats .inner { grid-template-columns: minmax(0, 1fr); }
.hero-text { display: grid; gap: 12px; min-width: 0; }
.mode { justify-self: start; display: inline-flex; align-items: center; gap: 7px; font-size: 12px; font-weight: 700; letter-spacing: .06em; text-transform: uppercase; color: var(--teal-deep); background: var(--surface); border-radius: 999px; padding: 4px 12px 4px 10px; }
.mode::before { content: ""; width: 7px; height: 7px; border-radius: 50%; background: var(--teal); }
.hero.for .mode { color: var(--slate-deep); }
.hero.for .mode::before { background: var(--slate); }
.hero h1 { font-size: clamp(26px, 4vw, 38px); font-weight: 600; max-width: 24ch; }
.hero .sub { font-size: 16px; max-width: 58ch; }
.asked { display: grid; gap: 2px; padding-left: 14px; border-left: 3px solid var(--slate); }
.asked .k { font-size: 11px; font-weight: 700; letter-spacing: .07em; text-transform: uppercase; color: var(--slate-deep); }
.asked .q { font-family: var(--font-display); font-size: 17px; font-weight: 500; color: var(--slate-deep); }
.scope { display: flex; flex-wrap: wrap; gap: 6px; font-size: 12.5px; }
.scope span { background: var(--surface); color: var(--slate-deep); border-radius: 999px; padding: 2px 10px; }
.stats { background: var(--surface); border-radius: 14px; display: grid; grid-template-columns: 1fr 1fr; overflow: hidden; border: 1px solid var(--line); }
.stat { padding: 14px 16px; display: grid; gap: 2px; align-content: start; min-width: 0; }
.stat:nth-child(even) { border-left: 1px solid var(--line); }
.stat:nth-child(n+3) { border-top: 1px solid var(--line); }
.stat:last-child:nth-child(odd) { grid-column: span 2; }
.stat .k { font-size: 12px; color: var(--muted); font-weight: 500; }
.stat .v { font: 600 26px/1.1 var(--font-display); overflow-wrap: anywhere; }
.stat .d { font-size: 12px; color: var(--muted); }
.stat.lead { background: var(--teal-deep); color: var(--surface); }
.stat.lead .k, .stat.lead .d { color: var(--teal-pale); }
.stat.lead .v { font-size: 34px; }
.stat.alert .v { color: var(--clay-deep); }
.stat.good .v { color: var(--teal-deep); }
.hero.for .stat.lead { background: var(--slate-deep); }
.hero.for .stat.lead .k, .hero.for .stat.lead .d { color: var(--slate-pale); }

/* Tabs */
.tabbar { position: sticky; top: env(safe-area-inset-top, 0px); z-index: 5; background: var(--surface); border-bottom: 1px solid var(--line); }
.tabs { display: flex; gap: 2px; overflow-x: auto; scrollbar-width: none; }
.tabs::-webkit-scrollbar { display: none; }
.tab { flex: none; display: flex; align-items: center; gap: 8px; padding: 14px 14px 12px; font: 500 14px var(--font-body); color: var(--muted); background: none; border: 0; border-bottom: 3px solid transparent; white-space: nowrap; cursor: pointer; }
.tab:hover { color: var(--ink); }
.tab .no { font: 600 11px var(--font-display); width: 19px; height: 19px; border-radius: 5px; display: grid; place-items: center; background: var(--bg); color: var(--muted); border: 1px solid var(--line); }
.tab .ct { font-size: 11.5px; font-weight: 600; color: var(--muted); background: var(--bg); border-radius: 999px; padding: 0 7px; border: 1px solid var(--line); }
.tab .ct.watch { background: var(--ochre-pale); border-color: var(--ochre-pale); color: var(--ochre-deep); }
.tab .flag { width: 7px; height: 7px; border-radius: 50%; background: var(--clay); }
.tab[aria-selected="true"] { color: var(--teal-deep); font-weight: 600; border-bottom-color: var(--teal); }
.tab[aria-selected="true"] .no { background: var(--teal); color: var(--surface); border-color: var(--teal); }
.for-report .tab[aria-selected="true"] { color: var(--slate-deep); border-bottom-color: var(--slate); }
.for-report .tab[aria-selected="true"] .no { background: var(--slate); border-color: var(--slate); }
.tab.ref { margin-left: auto; }
.tabs .divider { flex: none; width: 1px; background: var(--line); margin: 12px 6px; }

/* Panels */
.panel { padding-block: 34px 72px; }
.panel > .inner { max-width: 860px; display: grid; gap: 22px; }
.panel h2 { font-size: clamp(21px, 2.6vw, 26px); font-weight: 600; max-width: 34ch; }
.kicker { font-size: 12px; letter-spacing: .07em; text-transform: uppercase; font-weight: 600; color: var(--teal-deep); }
.kicker.slate { color: var(--slate-deep); }
.kicker.ochre { color: var(--ochre-deep); }
.head { display: grid; gap: 8px; }
.lede { max-width: 64ch; }
.muted { color: var(--muted); }
.answer { font-family: var(--font-display); font-size: clamp(19px, 2.4vw, 22px); line-height: 1.45; font-weight: 500; max-width: 44ch; }
.figure { background: var(--surface); border: 1px solid var(--line); border-radius: 12px; padding: 18px 18px 14px; min-width: 0; }
.figure .ftitle { font-weight: 600; font-size: 14px; }
.figure .fsub { font-size: 12.5px; color: var(--muted); margin-top: 2px; }
.figure svg { display: block; width: 100%; height: auto; margin-top: 12px; }
.figure .src { font-size: 12px; color: var(--muted); margin-top: 10px; display: flex; flex-wrap: wrap; gap: 6px 14px; align-items: center; }
.sowhat { background: var(--teal-wash); border-radius: 12px; padding: 12px 16px; font-size: 14.5px; display: grid; gap: 2px; }
.sowhat b { font-size: 11px; letter-spacing: .07em; text-transform: uppercase; color: var(--teal-deep); }
.conf { display: inline-flex; align-items: center; gap: 6px; font-size: 12px; font-weight: 600; padding: 2px 9px; border-radius: 999px; background: var(--teal-pale); color: var(--teal-deep); white-space: nowrap; }
.conf.watch { background: var(--ochre-pale); color: var(--ochre-deep); }
.conf.grey { background: var(--bg); color: var(--muted); border: 1px solid var(--line); }
.conf::before { content: ""; width: 6px; height: 6px; border-radius: 50%; background: currentColor; }
.legend { display: flex; flex-wrap: wrap; gap: 6px 14px; font-size: 12px; color: var(--muted); margin-top: 12px; }
.legend i { display: inline-block; width: 10px; height: 10px; border-radius: 3px; margin-right: 5px; vertical-align: -1px; }
.legend i.hollow { background: var(--surface); border: 2px solid var(--neutral); }

/* Story map and actions */
.storymap { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 12px; }
.story { --c: var(--slate); --cp: var(--slate-pale); --cd: var(--slate-deep); display: grid; gap: 6px; align-content: start; background: var(--surface); border: 1px solid var(--line); border-radius: 12px; padding: 14px 16px; text-decoration: none; color: var(--ink); transition: border-color .15s; cursor: pointer; }
.story:focus-visible { outline: 2px solid var(--teal); outline-offset: 2px; }
.story:hover { border-color: var(--c); }
.tone-problem { --c: var(--clay); --cp: var(--clay-pale); --cd: var(--clay-deep); }
.tone-strength { --c: var(--teal); --cp: var(--teal-pale); --cd: var(--teal-deep); }
.tone-watch { --c: var(--ochre); --cp: var(--ochre-pale); --cd: var(--ochre-deep); }
.story .top { display: flex; justify-content: space-between; align-items: center; gap: 8px; }
.story .tag { font-size: 11px; font-weight: 700; letter-spacing: .06em; text-transform: uppercase; color: var(--cd); background: var(--cp); padding: 2px 8px; border-radius: 999px; }
.story .ch { font-size: 12px; color: var(--muted); }
.story h3 { font-size: 16px; font-weight: 600; }
.story p { font-size: 13.5px; color: var(--muted); }
.story .go { font-size: 13px; font-weight: 600; color: var(--cd); }
.split { display: grid; grid-template-columns: 1fr 1fr; gap: 22px; }
.split h3, .label-h { font: 600 12px var(--font-body); letter-spacing: .07em; text-transform: uppercase; color: var(--muted); margin-bottom: 10px; }
.actions { display: grid; gap: 8px; }
.action { border-radius: 10px; padding: 10px 14px; font-size: 14px; display: grid; gap: 3px; }
.action .tag { font-size: 11px; font-weight: 700; letter-spacing: .06em; text-transform: uppercase; }
.action.now { background: var(--clay-pale); } .action.now .tag { color: var(--clay-deep); }
.action.next { background: var(--ochre-pale); } .action.next .tag { color: var(--ochre-deep); }
.action.protect { background: var(--teal-pale); } .action.protect .tag { color: var(--teal-deep); }
.nextq { display: grid; gap: 8px; }
.nextq div { background: var(--slate-pale); border-radius: 10px; padding: 10px 14px; font-size: 14px; display: flex; justify-content: space-between; gap: 12px; flex-wrap: wrap; align-items: center; color: var(--slate-deep); }
.nextq code { font-size: 12px; color: var(--surface); background: var(--slate); padding: 2px 8px; border-radius: 6px; white-space: nowrap; }
.reasons { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 12px; }
.reason { background: var(--clay-pale); border-radius: 12px; padding: 14px 16px; display: grid; gap: 6px; align-content: start; }
.reason .k { font-size: 11px; font-weight: 700; letter-spacing: .06em; text-transform: uppercase; color: var(--clay-deep); }
.reason .v { font: 600 26px/1.1 var(--font-display); color: var(--clay-deep); }
.reason p { font-size: 13.5px; }
.same { background: var(--teal-wash); border-radius: 12px; padding: 12px 16px; font-size: 14px; }
.same b { color: var(--teal-deep); }
.caveats { list-style: none; padding: 0; margin: 0; display: grid; gap: 10px; font-size: 14px; }
.caveats li { display: grid; grid-template-columns: 70px 1fr; gap: 10px; align-items: baseline; }

/* NPS composition */
.npsbar { display: flex; gap: 3px; height: 28px; margin-top: 12px; font-size: 12px; font-weight: 600; }
.npsbar span { display: grid; place-items: center; border-radius: 6px; min-width: 0; overflow: hidden; }
.npsbar .det { background: var(--clay); color: var(--surface); }
.npsbar .pas { background: var(--neutral); color: var(--ink); }
.npsbar .pro { background: var(--teal); color: var(--surface); }

/* SVG */
svg text { font-family: var(--font-body); fill: var(--ink); font-size: 12px; }
svg .t-muted { fill: var(--muted); font-size: 11px; }
svg .t-strong { font-weight: 600; }
svg .q-lab { font-size: 11px; font-weight: 700; letter-spacing: .06em; }
svg .c-teal { fill: var(--teal-deep); }
svg .c-clay { fill: var(--clay-deep); font-weight: 600; }
svg .c-ochre { fill: var(--ochre-deep); }
svg .c-slate { fill: var(--slate-deep); }
svg .c-grey { fill: var(--muted); }
svg .z-teal { fill: var(--teal-wash); }
svg .z-clay { fill: var(--clay-pale); }
svg .z-ochre { fill: var(--ochre-pale); }
svg .z-slate { fill: var(--slate-pale); }
svg .grid { stroke: var(--line); stroke-width: 1; fill: none; }
svg .grid-w { stroke: var(--surface); stroke-width: 1.5; fill: none; }
svg .axis { stroke: var(--muted); stroke-width: 1; fill: none; }
svg .ref { stroke: var(--ochre); stroke-width: 1.5; stroke-dasharray: 4 3; fill: none; }
svg .d-teal { fill: var(--teal); }
svg .d-clay { fill: var(--clay); }
svg .d-ochre { fill: var(--ochre); }
svg .d-slate { fill: var(--slate); }
svg .d-hollow { fill: var(--surface); stroke: var(--neutral); stroke-width: 2; }
svg .ci-teal { stroke: var(--teal-mid); stroke-width: 3; stroke-linecap: round; fill: none; }
svg .ci-clay { stroke: var(--clay-mid); stroke-width: 3; stroke-linecap: round; fill: none; }
svg .ci-grey { stroke: var(--neutral); stroke-width: 3; stroke-linecap: round; fill: none; }
svg .lk-clay { stroke: var(--clay-pale); stroke-width: 8; stroke-linecap: round; fill: none; }
svg .lk-grey { stroke: var(--line); stroke-width: 6; stroke-linecap: round; fill: none; }
svg .pill-clay { fill: var(--clay-pale); }
svg .band-teal { fill: var(--teal-pale); }
svg .line-teal { stroke: var(--teal); stroke-width: 2.5; fill: none; }

/* Diverging ratings */
.likert { display: grid; gap: 10px; margin-top: 14px; }
.lk-row { display: grid; grid-template-columns: minmax(96px, 180px) minmax(0, 1fr) 46px; gap: 10px; align-items: center; font-size: 13px; }
.lk-row .lab { text-align: right; line-height: 1.25; }
.lk-row.hl .lab { font-weight: 600; color: var(--clay-deep); }
.lk-row.small .lab, .lk-row.small .t2b { color: var(--muted); }
.lk-track { position: relative; height: 18px; }
.lk-track .mid { position: absolute; top: -4px; bottom: -4px; width: 1px; background: var(--muted); }
.lk-neg { position: absolute; top: 0; bottom: 0; display: flex; flex-direction: row-reverse; }
.lk-pos { position: absolute; top: 0; bottom: 0; display: flex; }
.lk-neg span, .lk-pos span { height: 100%; }
.s-d { background: var(--clay-mid); }
.s-sd { background: var(--clay); border-radius: 3px 0 0 3px; }
.s-a { background: var(--teal-mid); }
.s-sa { background: var(--teal); border-radius: 0 3px 3px 0; }
.lk-row .t2b { font-weight: 600; text-align: right; color: var(--teal-deep); }
.lk-row.hl .t2b { color: var(--clay-deep); }

/* Horizontal bars (single choice, multi-select) */
.hbars { display: grid; gap: 8px; margin-top: 12px; }
.hbar { display: grid; grid-template-columns: minmax(90px, 220px) minmax(0, 1fr) 48px; gap: 10px; align-items: center; font-size: 13px; }
.hbar .lab { line-height: 1.25; overflow-wrap: anywhere; }
.hbar .track { height: 14px; background: var(--bg); border-radius: 3px; overflow: hidden; }
.hbar .track span { display: block; height: 100%; background: var(--teal); border-radius: 3px; }
.hbar .track span.slate { background: var(--slate); }
.hbar .pct { font-weight: 600; text-align: right; }

/* Personas */
.personas { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 12px; }
.persona { --c: var(--teal); --cd: var(--teal-deep); --cp: var(--teal-pale); background: var(--surface); border: 1px solid var(--line); border-radius: 12px; padding: 16px; display: grid; gap: 10px; align-content: start; }
.persona.c1 { --c: var(--ochre); --cd: var(--ochre-deep); --cp: var(--ochre-pale); }
.persona.c2 { --c: var(--slate); --cd: var(--slate-deep); --cp: var(--slate-pale); }
.persona.c3 { --c: var(--teal-mid); --cd: var(--teal-deep); --cp: var(--teal-wash); }
.persona.focus { --c: var(--clay); --cd: var(--clay-deep); --cp: var(--clay-pale); background: var(--cp); border-color: var(--c); }
.persona .top { display: flex; justify-content: space-between; gap: 10px; align-items: baseline; }
.persona h3 { font-size: 16px; font-weight: 600; color: var(--cd); display: flex; align-items: center; gap: 8px; }
.persona h3::before { content: ""; width: 10px; height: 10px; border-radius: 50%; background: var(--c); flex: none; }
.persona .share { font: 600 24px var(--font-display); color: var(--cd); }
.persona .sizebar { height: 6px; background: var(--cp); border-radius: 3px; }
.persona.focus .sizebar { background: var(--surface); }
.persona .sizebar span { display: block; height: 100%; border-radius: 3px; background: var(--c); }
.persona dl { display: grid; grid-template-columns: auto 1fr; gap: 3px 10px; margin: 0; font-size: 13px; }
.persona dt { color: var(--muted); }
.persona dd { margin: 0; }
.persona q { font-size: 13px; font-style: italic; color: var(--muted); }
.persona .pflag { justify-self: start; font-size: 11px; font-weight: 700; letter-spacing: .06em; text-transform: uppercase; color: var(--surface); background: var(--clay); padding: 2px 8px; border-radius: 999px; }

/* Themes and quotes */
.themes { display: grid; gap: 12px; margin-top: 14px; }
.theme { display: grid; grid-template-columns: minmax(100px, 200px) minmax(0, 1fr) 44px; gap: 10px; align-items: center; font-size: 13px; }
.theme .bar { height: 16px; display: flex; gap: 2px; }
.theme .bar span { border-radius: 3px; }
.theme .bar .neg { background: var(--clay); }
.theme .bar .pos { background: var(--teal); }
.theme .bar .mix { background: var(--neutral); }
.theme .pct { font-weight: 600; text-align: right; }
.quotes { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 12px; }
.quote { --c: var(--slate); --cd: var(--slate-deep); background: var(--surface); border: 1px solid var(--line); border-radius: 12px; padding: 14px 16px; display: grid; gap: 8px; align-content: start; margin: 0; }
.quote.neg { --c: var(--clay); --cd: var(--clay-deep); }
.quote.pos { --c: var(--teal); --cd: var(--teal-deep); }
.quote .th { font-size: 11px; font-weight: 700; letter-spacing: .06em; text-transform: uppercase; color: var(--cd); display: flex; gap: 6px; align-items: center; }
.quote .th::before { content: ""; width: 8px; height: 8px; border-radius: 50%; background: var(--c); }
.quote p { font-size: 14px; overflow-wrap: anywhere; }
.quote p::before { content: "\201C"; font: 600 28px/0 var(--font-display); color: var(--c); vertical-align: -10px; margin-right: 3px; }
.quote .who { font-size: 12px; color: var(--muted); }

/* Question cards */
.qsec { font-size: 12px; font-weight: 700; letter-spacing: .07em; text-transform: uppercase; color: var(--muted); margin-top: 8px; }
.qcard .qmeta { font-size: 12px; color: var(--muted); margin-top: 4px; display: flex; flex-wrap: wrap; gap: 4px 12px; }
.bignum { font: 600 30px/1.1 var(--font-display); color: var(--teal-deep); }

/* Data & method */
.quality { display: flex; flex-wrap: wrap; gap: 8px; font-size: 13px; }
.quality span { background: var(--surface); border: 1px solid var(--line); border-radius: 999px; padding: 4px 12px; }
.quality span.flag { background: var(--ochre-pale); border-color: var(--ochre-pale); color: var(--ochre-deep); }
.quality span.good { background: var(--teal-pale); border-color: var(--teal-pale); color: var(--teal-deep); }
.table-wrap { overflow-x: auto; }
table { width: 100%; border-collapse: collapse; font-size: 13px; }
th, td { text-align: left; padding: 8px 10px; border-bottom: 1px solid var(--line); vertical-align: top; }
th { font-size: 11.5px; letter-spacing: .05em; text-transform: uppercase; color: var(--muted); font-weight: 600; }
td .badge { display: inline-block; font-size: 11.5px; font-weight: 600; padding: 1px 8px; border-radius: 999px; }
.badge.green { background: var(--teal-pale); color: var(--teal-deep); }
.badge.yellow { background: var(--ochre-pale); color: var(--ochre-deep); }
.badge.red { background: var(--clay-pale); color: var(--clay-deep); }
ul.methods { margin: 0; padding-left: 18px; display: grid; gap: 6px; font-size: 14px; }

/* Pager and footer */
.pager { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-top: 18px; border-top: 1px solid var(--line); padding-top: 18px; }
.pager button { display: grid; gap: 2px; text-align: left; font: inherit; cursor: pointer; color: var(--ink); background: var(--surface); border: 1px solid var(--line); border-radius: 12px; padding: 12px 16px; }
.pager button:hover { border-color: var(--teal); }
.pager button.nx { text-align: right; grid-column: 2; }
.pager span { font-size: 12px; color: var(--muted); }
.pager b { font-family: var(--font-display); font-weight: 600; font-size: 15px; }
.foot { border-top: 1px solid var(--line); padding-block: 22px 32px; background: var(--surface); }
.foot .inner { display: flex; flex-wrap: wrap; gap: 10px 18px; align-items: center; justify-content: space-between; font-size: 12.5px; color: var(--muted); }
.foot .brand { font-size: 14px; }
.foot .mark { width: 30px; height: 30px; }
.cover, .print-only { display: none; }

/* Tabs need scripts. Without them (some app previews), every section shows as
   one long page and the controls that need scripts are hidden. */
.js [role="tabpanel"]:not(.active) { display: none; }
.no-js .tabbar, .no-js .pager, .no-js .story .go { display: none !important; }
.no-js [role="tabpanel"] + [role="tabpanel"] { border-top: 1px solid var(--line); }
.nologo .mark { display: none; }

@media (max-width: 900px) {
  .hero .inner { grid-template-columns: minmax(0, 1fr); }
  .stats { grid-template-columns: repeat(4, minmax(0, 1fr)); }
  .stat:nth-child(even) { border-left: 0; }
  .stat + .stat { border-left: 1px solid var(--line); }
  .stat:nth-child(n+3) { border-top: 0; }
  .tab.ref { margin-left: 0; }
}
@media (max-width: 640px) {
  .stats { grid-template-columns: 1fr 1fr; }
  .stat + .stat { border-left: 0; }
  .stat:nth-child(even) { border-left: 1px solid var(--line); }
  .stat:nth-child(n+3) { border-top: 1px solid var(--line); }
  .storymap, .split, .personas, .quotes { grid-template-columns: minmax(0, 1fr); }
  .lk-row { grid-template-columns: 96px minmax(0, 1fr) 38px; font-size: 12px; }
  .theme { grid-template-columns: 112px minmax(0, 1fr) 36px; font-size: 12px; }
  .hbar { grid-template-columns: 110px minmax(0, 1fr) 40px; font-size: 12px; }
  .pager { grid-template-columns: minmax(0, 1fr); }
  .pager button.nx { grid-column: 1; }
}

/* ── Print / PDF ── */
@page { size: __PAPER__; margin: 16mm 14mm 18mm; }
@page {
  @bottom-right { content: "Page " counter(page) " of " counter(pages); font: 8pt -apple-system, "Segoe UI", Roboto, Arial, sans-serif; color: #6C736D; }
}
@media print {
  body { background: #FFFFFF; font-size: 10.5pt; }
  .appbar, .tabbar, .pager, .hero, .screen-only { display: none !important; }
  .cover, .print-only { display: block; }
  .cover { break-after: page; min-height: 240mm; position: relative; padding-top: 10mm; }
  .cover .brand { margin-bottom: 40mm; }
  .cover .band { background: var(--teal-wash); border-radius: 10px; padding: 14mm 12mm; display: grid; gap: 4mm; }
  .cover.for .band { background: var(--slate-pale); }
  .cover .kind { font-size: 9pt; font-weight: 700; letter-spacing: .08em; text-transform: uppercase; color: var(--teal-deep); }
  .cover h1 { font-size: 28pt; }
  .cover .meta { color: var(--muted); font-size: 10pt; }
  .cover .bigmark { width: 42mm; height: 42mm; background: var(--logo) center / contain no-repeat; position: absolute; bottom: 10mm; left: 0; }
  .cover .conf-label { position: absolute; top: 10mm; right: 0; font-size: 9pt; font-weight: 700; letter-spacing: .08em; text-transform: uppercase; color: var(--clay-deep); }
  .print-stats { display: grid !important; grid-template-columns: repeat(4, 1fr); gap: 3mm; margin-bottom: 6mm; }
  .print-stats .stat { border: 1px solid var(--line) !important; border-radius: 8px; }
  [role="tabpanel"], .js [role="tabpanel"]:not(.active) { display: block !important; }
  [role="tabpanel"] { break-before: page; padding-block: 0; }
  [role="tabpanel"]:first-of-type { break-before: auto; }
  .figure, .quote, .persona, .story, .reason, .action, tr, .lk-row, .theme { break-inside: avoid; }
  .foot { break-before: avoid; border: 0; padding-top: 6mm; }
  body.unbranded .brand, body.unbranded .bigmark, body.unbranded .foot .brand, body.unbranded .credit { display: none !important; }
  a { color: inherit; text-decoration: none; }
  * { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
}
```

### JavaScript

```js
(function () {
  /* Tabs switch in place. They never follow a link: inside app previews
     (sandboxed frames) a '#section' link replaces the whole page with a blank one. */
  var panels = Array.prototype.slice.call(document.querySelectorAll('[role="tabpanel"]'));
  var tabs = Array.prototype.slice.call(document.querySelectorAll('.tab[data-tab]'));
  var first = panels.length ? panels[0].id.replace(/^p-/, '') : '';

  function readHash() {
    try { return (window.location.hash || '').slice(1); } catch (e) { return ''; }
  }
  function writeHash(name) {
    // Lets a link open a given tab. Some previews forbid it; the tabs work regardless.
    try { if (window.history && history.replaceState) history.replaceState(null, '', '#' + name); } catch (e) {}
  }

  function show(name, opts) {
    opts = opts || {};
    var panel = document.getElementById('p-' + name);
    if (!panel) { name = first; panel = document.getElementById('p-' + name); }
    if (!panel) return;
    panels.forEach(function (p) { p.classList.toggle('active', p === panel); });
    tabs.forEach(function (t) {
      var on = t.getAttribute('data-tab') === name;
      t.setAttribute('aria-selected', on ? 'true' : 'false');
      t.tabIndex = on ? 0 : -1;
      if (on) {
        var row = t.parentElement;
        var l = t.offsetLeft - row.offsetLeft;
        if (l < row.scrollLeft || l + t.offsetWidth > row.scrollLeft + row.clientWidth) row.scrollLeft = Math.max(0, l - 16);
      }
    });
    if (opts.scroll) {
      var bar = document.querySelector('.tabbar');
      var top = panel.getBoundingClientRect().top + window.pageYOffset - (bar ? bar.offsetHeight : 0);
      if (window.pageYOffset > top) window.scrollTo(0, top);
    }
    if (opts.record) writeHash(name);
  }

  document.addEventListener('click', function (e) {
    var el = e.target.closest ? e.target.closest('[data-tab], [data-go]') : null;
    if (!el) return;
    e.preventDefault();
    show(el.getAttribute('data-tab') || el.getAttribute('data-go'), { scroll: true, record: true });
  });
  document.addEventListener('keydown', function (e) {
    var el = e.target;
    if (el && el.getAttribute && el.getAttribute('data-go') && (e.key === 'Enter' || e.key === ' ')) {
      e.preventDefault();
      show(el.getAttribute('data-go'), { scroll: true, record: true });
    }
    if (el && el.getAttribute && el.getAttribute('role') === 'tab' && (e.key === 'ArrowRight' || e.key === 'ArrowLeft')) {
      var i = tabs.indexOf(el) + (e.key === 'ArrowRight' ? 1 : -1);
      var next = tabs[(i + tabs.length) % tabs.length];
      next.focus();
      show(next.getAttribute('data-tab'), { record: true });
    }
  });
  window.addEventListener('hashchange', function () { show(readHash(), { scroll: true }); });
  show(readHash(), {});
})();
```
