# EP Case Dashboard — Project Specification v4

**Version**: 1.7 (current HTML build)
**Owner**: Eric Wang, Biosense Webster Key Account CAS, Los Angeles
**Last updated**: April 2026
**Status**: Active — iterative refinement

---

## How to use this document

You are Claude in Claude Code / VS Code. The user will attach this spec plus the current HTML file (`EP_Dashboard_2026-04-19_v1.7.html` or a later version). Read both, then continue refinement based on the user's next request.

**Three operating rules**:

1. **Prefer `str_replace` over full rewrites.** The HTML is ~290KB with an embedded 240KB JSON data blob and a 5KB base64 logo. Full rewrites waste tokens, risk breaking the data blob, and make diffs unreadable. Use `str_replace` for any change. If a change truly requires restructuring >100 lines, stop and ask the user first.

2. **Discuss before implementing.** Eric prefers to see options and trade-offs before code. Present 2–3 approaches with pros/cons, then wait for his pick. He often responds in Traditional Chinese — mirror that but keep UI text in English.

3. **Sanity-check after every batch.** After a group of edits, run a Python snippet that checks CSS brace balance, verifies new patterns exist, and confirms no dead references remain. See `§10 Validation` for the standard check.

---

## §1 Project purpose

Eric manages the Biosense Webster LA Key Account territory. Each week Salesforce exports a case log as `.xlsx`. Eric wants a self-contained HTML dashboard he can:

- Open in any browser (mobile or desktop), no install, no server
- Send to colleagues / managers as a single file attachment
- Regenerate weekly from the latest Salesforce export with one command

The dashboard shows territory, hospital, and physician performance with YoY comparisons, procedure mixes, and cross-hospital analysis for physicians who operate at multiple sites.

---

## §2 Final deliverable

**A single Python script: `build_dashboard.py`**

```bash
python build_dashboard.py <path-to-excel.xlsx> [--output path-to-output.html]
```

Behavior:
1. Read the Excel file
2. Run the data pipeline (filter, aggregate, compute YoY)
3. Emit a complete standalone HTML file

The script bundles:
- A Python data pipeline (pandas-based, ~150 lines)
- An HTML template as a Python string (~500 lines of HTML/CSS/JS)
- A base64-encoded Biosense Webster logo (~5KB)
- A JSON injection step that replaces a placeholder in the template with the computed aggregates

Output is one `.html` file the user can double-click to open. No external runtime dependencies (Chart.js loads from CDN at render time).

The current workflow has the HTML and pipeline as two separate artifacts. **The final handoff deliverable combines them.** See §11 for the build-script structure.

---

## §3 Users and use cases

- **Primary user**: Eric — opens on desktop (1440px+) and phone (iPhone)
- **Secondary audience**: BWI managers, regional directors — Eric may email the HTML as a one-file report
- **Sensitive data**: Case-level physician and hospital activity. Footer reads "Confidential · Internal Use Only". Do not add email addresses, phone numbers, or any PII beyond what Salesforce exports.

---

## §4 Data source

**Input**: Salesforce report export, `.xlsx` format.

**File structure**:
- Sheet name: `Case data`
- Headers on **row 9** (index 8 when `header=8`)
- Data starts row 10
- Columns used:
  - `Name` — territory name (filter rule below)
  - `Actual End Date Time` — case completion timestamp
  - `Account: Account Name` — hospital (rename to `hospital`)
  - `Physician: CARTODAY Affiliation Name` — physician (rename to `physician`)
  - `Primary Procedure: Work Type Name` — procedure (rename to `procedure`)

**Filter rules** (apply in this order):
1. Drop rows where `Name` is `Total`, or contains `Copyright`, `Confidential`, `Do Not Distribute` (these are footer rows from Salesforce)
2. Keep only rows where `Name == 'Los Angeles, CA - KA'`. Other territories like `LA County, CA` must be discarded.
3. Drop rows where `Actual End Date Time` is null. Early records (pre-2020) often lack this field.
4. Parse date with `pd.to_datetime(errors='coerce')`, then drop NaT.
5. Keep only dates `>= 2024-01-01`. The dashboard uses CY vs LY (last year), so two full years is the minimum retention.

**Known quirks**:
- Salesforce sometimes emits the territory name with extra whitespace (`Los Angeles, CA - KA ` with trailing space). Use `.str.strip()` if you see mismatches.
- The `West LA VA` hospital has an unusual name: `WST LOS ANGL VA MED  CENTER` (two spaces before "CENTER"). Preserve this exact string — it is the Salesforce canonical form and is used as a dictionary key.
- One hospital may have zero YTD cases in the current year. Treat it as inactive (drop from active lists, retain in `all hospitals` if needed).

---

## §5 Aggregates produced

The Python pipeline emits a single JSON object with these keys. Every aggregate is a plain list of dicts (JSON-friendly).

### Metadata
```
meta: {
  last_date: "YYYY-MM-DD"             # most recent case date
  last_year, last_month, last_day     # integers
  total_cases                         # int, count after all filters
  hospitals                           # sorted list of all hospital names
  n_hospitals, n_physicians, n_procedures    # all-time unique counts
  active_hospitals, active_physicians         # had ≥1 case in current year
}
```

### Region-level
- `region_monthly` — `[{ym, count}]` every month since 2024-01
- `region_mtd_current`, `region_mtd_ly` — int, month-to-date cases
- `region_ytd_current`, `region_ytd_ly` — int, year-to-date cases
- `region_procedure_ytd` — `[{procedure, count}]` YTD current year, sorted desc
- `region_procedure_ytd_ly` — same for last year

### Hospital-level
- `hospital_ranking_detail` — `[{hospital, count, md_count, pct_of_territory, count_ly}]`, YTD current year, sorted desc. Only includes hospitals with ≥1 YTD case.
- `hospital_ytd_ranking` — simplified version, just `{hospital, count}` for dropdowns
- `hospital_monthly` — `[{hospital, ym, count}]` all months
- `hospital_procedure_ytd` — `[{hospital, procedure, count}]` current YTD
- `mtd_current`, `mtd_ly`, `ytd_current`, `ytd_ly` — `[{hospital, count}]`

### Physician-in-hospital
- `hospital_physician_ytd` — `[{hospital, physician, count}]`
- `hospital_physician_monthly` — `[{hospital, physician, ym, count}]`
- `hospital_physician_procedure_ytd` — `[{hospital, physician, procedure, count}]`
- `phys_mtd_current`, `phys_mtd_ly`, `phys_ytd_current`, `phys_ytd_ly` — `[{hospital, physician, count}]`

### Physician-level (all-hospital aggregated)
- `phys_all_ytd_ranking` — `[{physician, count}]` YTD current, sorted desc
- `phys_monthly_all` — `[{physician, ym, count}]` across all hospitals
- `phys_procedure_ytd_all` — `[{physician, procedure, count}]`
- `phys_mtd_all`, `phys_mtd_ly_all`, `phys_ytd_all`, `phys_ytd_ly_all` — `[{physician, count}]`
- `phys_hospitals` — `[{physician, hospital, count}]` YTD, sorted by physician then count desc. Used to identify cross-hospital physicians.
- `physician_primary_hospital` — `[{physician, hospital}]` the hospital each physician does most of their work at (falls back to all-time if no YTD)

### Top 10 for Region Summary
- `physician_ranking_detail` — `[{physician, count, count_ly}]`, top 10 by YTD current year

### YoY logic
- **MTD comparison**: case count from day 1 of current month up to and including `last_day` in current month vs. same date range in last year (e.g., `2026-04-01` to `2026-04-17` vs. `2025-04-01` to `2025-04-17`)
- **YTD comparison**: `2026-01-01` to `2026-04-17` vs. `2025-01-01` to `2025-04-17`

---

## §6 Three pages, what each contains

Navigation: sidebar on desktop (≥768px), tab bar on mobile. No back button in sticky header — tab bar / sidebar handles all navigation.

### Page 1 — Region Summary

Sections top to bottom:
1. **Page title**: "Region Summary" + subtitle "Los Angeles, CA – KA"
2. **Territory banner** (`Coverage YTD`) — 2 circular-icon stats: Hospitals count, Physicians count
3. **Data-through line** — text "Data through <date>" + pill showing "TODAY" / "N DAYS AGO"
4. **Section: Territory Performance** — 2 KPI cards (MTD, YTD) with YoY delta
5. **Section: Monthly Trend** — line chart, CY red solid with gradient fill, LY dashed gray (toggle via "Show LY" button)
6. **Section: Trend & Procedure** (desktop only label) — 2-up on desktop: left is the line chart above, right is a doughnut + list "By Procedure Type"
   - Procedure list shows Top 6 procedures + Others row (if >6). Each row: colored dot, procedure name, count, progress bar, %, YoY pill
7. **Section: Hospital Ranking** — grid of 6 hospital cards (2 cols desktop, 1 col mobile), click to navigate to Hospital Detail
8. **Section: Top Physicians** — list of top 10 physicians (not a chart), click to navigate to Physician Detail

### Page 2 — Hospital Detail

1. **Page title**: "Hospital Detail" + subtitle "Individual Performance"
2. **Hospital dropdown** — select one hospital
3. **Data-through line**
4. **Section: Performance** — 2 KPI cards (MTD, YTD)
5. **Section: Monthly Volume** — same line chart pattern as Region (Show LY toggle, gradient fill on CY)
6. **Section: Case Analysis** — desktop 2-up:
   - Left: doughnut "By Procedure Type" (Top 6 + Others)
   - Right: doughnut "Top Physicians" (**Top 10 + Others**, blue palette). Hint text "Tap a slice to open physician" in chart header. Clicking a slice navigates to that physician's Physician Detail page.

### Page 3 — Physician Detail

1. **Page title**: "Physician Detail" + subtitle "Individual Performance"
2. **Physician dropdown** — lists all physicians sorted by YTD current count. For cross-hospital physicians, the dropdown item shows per-hospital breakdown below the name, e.g., `Carlos Macias · 49 / UCLA 34 · Saint John's 15`
3. **Data-through line**
4. **Hospital Comparison section** (hidden if physician practices at only 1 hospital) — line chart with one line per hospital (CY only, red + blue, no gradient). Used to see how a physician's activity splits across sites.
5. **"Drill down by hospital" divider** — `.section-title` style (highlighter)
6. **Hospital Tab** — `All / UCLA / Saint John's / ...` tab bar. Clicking changes the KPI/line/donut below. For single-hospital physicians the tab bar is hidden.
7. **Section: Performance** — 2 KPI cards (MTD, YTD) scoped to the selected tab
8. **Section: Monthly Volume** — line chart scoped to the selected tab
9. **Section: Procedure Mix** — doughnut scoped to the selected tab (Top 6 + Others)

---

## §7 Design system

### Colors (CSS variables at `:root`)
```
--bg:       #fafaf5      cream white (page background)
--bg2:      #f5f5f0      slightly darker cream (icon bg, tab bg)
--text:     #1a1a1a      primary text
--text2:    #5a574c      secondary text
--text3:    #8a8578      tertiary text (labels)
--text4:    #a8a498      quaternary (footnotes)
--border:   #e8e5dc      neutral beige (default border, Others color)
--border2:  #f0ede3      lighter border
--green:    #2d7a4f   --green-bright: #4fb37a   --green-bg: rgba(79,179,122,0.15)
--red:      #c93838   --red-bright:   #e86b6b   --red-bg:   rgba(232,107,107,0.15)
--gold:     #b4aa82   --gold-bg:      rgba(180,170,130,0.15)
--bw:       #EB1700      Biosense Webster brand red (primary)
--bw-orange:#ee892d      Biosense Webster brand orange (sticky header, logo background)
```

### Palettes (JS constants, after `--bw-orange`)
```js
// Stepped red for procedure doughnuts + progress bars
const PC = ['#A01528','#EB1700','#F04D5E','#F48091','#F8A8B5','#FAC4CC','#FCDDE2','#FEE8EC'];

// 10-step blue for Top Physicians donut (deep → light)
const PHC = ['#1B395C','#234D79','#2C6097','#3573B5','#4786C9','#659AD1','#83AEDA','#A1C2E3','#BFD5EC','#DDE9F5'];

// Neutral beige for Others slice (matches --border color)
const OTHERS_COLOR = '#e8e5dc';
```

**Rule**: when a doughnut has an Others slice, its color must be `OTHERS_COLOR`, never a pale shade of the main palette. Pale palette shades blend into adjacent medium shades and cause confusion. Use the helper `palWithOthers(pal, n, hasOthers)`.

### Typography
- **Sans-serif only**: DM Sans (Google Fonts). Weights 400, 500, 600, 700, 800.
- **Monospace**: JetBrains Mono (for dates, small numeric labels)
- **Instrument Serif is removed**. Do not reintroduce serif fonts — Eric prefers a clean modern sans-only look.

### Section titles (highlighter style)
```css
.section-title {
  font-family: var(--sans);
  font-size: 20px;
  font-weight: 700;
  color: var(--text);
  margin-bottom: 14px;
  line-height: 1.3;
  display: inline-block;
  background: linear-gradient(transparent 62%, rgba(235,23,0,0.18) 62%, rgba(235,23,0,0.18) 92%, transparent 92%);
  padding: 0 4px;
  letter-spacing: .1px;
}
```

**No left red bar**. The highlighter band sits behind the lower half of the text. Used for every section header on every page.

### Sticky header (every page)
- Background: `var(--bw-orange)` (`#ee892d`)
- Padding: `7px 16px`
- Logo: 18px tall (white Biosense heart on orange, base64 PNG)
- Title: 13px, weight 700, white
- Date pill: 9px mono, white with 90% opacity
- No back button on any page. Navigation via tab bar (mobile) or sidebar (desktop).

### Sidebar (desktop only, ≥768px)
- 220px fixed width, left side
- Brand: "LA Key Account" / "Biosense Webster"
- Nav items: Region, Hospital, Physician — icon + label

### Footer (all pages)
- Full width of scroll-area (not constrained to page-inner's 960px)
- Top border spans full viewport (subtract sidebar width on desktop)
- Content centered via `.page-footer-inner` max-width 960px wrapper
- Two lines:
  - `© <year> Eric Wang · LA Key Account Dashboard <VER>`
  - `Confidential · Internal Use Only · Source: Salesforce`
- No email. No "Data as of" date (date is already in sticky header).

### Hospital name abbreviations
```js
const HABBR = {
  'RONALD REAGAN UCLA MEDICAL CENTER': 'UCLA',
  'PROVIDENCE SAINT JOHNS HEALTH CENTER': "Saint John's",
  'KECK HOSPITAL OF USC': 'Keck',
  'USC ARCADIA HOSPITAL': 'Arcadia',
  'CEDARS SINAI MEDICAL CENTER': 'Cedars',
  'SOUTH CALIFORNIA PERMANENTE MEDICAL GROUP': 'Kaiser',
  'WST LOS ANGL VA MED  CENTER': 'West LA'
};
function hAbbr(h) { return HABBR[h] || tc(h).split(' ').slice(0,2).join(' '); }
```

---

## §8 Key helper functions (JS)

Keep these signatures stable. Other code depends on them.

### `fold6(data, keyField)` and `foldN(data, keyField, n)`
Folds a sorted array into Top N + Others. Returns:
```
{
  labels:      [name1, ..., nameN, 'Others (K)']
  values:      [count1, ..., countN, othersSum]
  hasOthers:   bool
  othersCount: int   # K = number of folded items
  othersTotal: int   # sum of folded counts
}
```
If `data.length <= n`, no folding: `hasOthers=false`, all items returned.

`fold6` is a thin wrapper around `foldN(..., 6)`. Used for procedure doughnuts (3 places).

`foldN(..., 10)` is used for the Top Physicians doughnut in Hospital Detail.

### `palWithOthers(pal, n, hasOthers)`
Returns the first `n` colors from `pal`. If `hasOthers` is true, replaces the last color with `OTHERS_COLOR`.

Use this for every doughnut `backgroundColor`. Never do `PC.slice(0, n)` directly — if `n > pal.length` you get undefined, and even when it fits, the pale end of the palette doesn't read as "other" visually.

### `gradFill(color)`
Returns a Chart.js background callback that creates a vertical linear gradient from `color + '40'` (25% alpha) at top to `color + '00'` (0% alpha) at bottom. Used for the red CY line in Monthly Volume charts.

**Rule**: Apply to single-line charts and the CY line of dual-line (Show LY) charts. Do NOT apply to dashed LY lines or to the Hospital Comparison chart (multiple overlapping fills muddy the visuals).

### `hAbbr(h)`
Hospital name abbreviator using `HABBR` map. Falls back to title-casing first two words.

### `tc(s)`
Title-case a string: `'RONALD REAGAN UCLA MEDICAL CENTER'` → `'Ronald Reagan Ucla Medical Center'`.

### `goH(h)`, `goPD(p)`
Navigation shortcuts: `goH` switches to Hospital Detail for a given hospital; `goPD` switches to Physician Detail for a given physician.

### `uPDHComp()`
Draws the Hospital Comparison chart for the currently selected physician. Hidden via `pdHCompSection.style.display` toggle when the physician practices at only one hospital.

---

## §9 Interaction patterns

- **Event delegation with `data-*` attributes**. Do NOT use inline `onclick` with string interpolation — quote-escaping is fragile. Pattern:
  ```html
  <div data-phys="Eric Buch">Eric Buch</div>
  ```
  ```js
  container.onclick = function(e) {
    const it = e.target.closest('[data-phys]');
    if (it) goPD(it.dataset.phys);
  };
  ```

- **Line chart null handling**: For months in the future relative to `last_date`, set data to `null` (not `0`). Chart.js with `spanGaps: true` draws the line up to the last real data point and stops. Past months with zero cases remain as `0`.

- **YOY pill edge cases**:
  - `count_ly === 0 && count > 0` → show "NEW" (gold pill)
  - `count_ly === 0 && count === 0` → show "—"
  - Otherwise → `(+|-)N%` with green/red

- **Cross-hospital physician detection**: a physician has `phys_hospitals` entries for more than one hospital name. Show the Hospital Comparison chart + Hospital Tab only for these.

- **iOS dark mode prevention**: `<meta name="color-scheme" content="light">` plus `html,body{background:#fafaf5}` plus `-webkit-text-size-adjust:100%`. Do not remove these — without them, iOS Safari sometimes inverts colors in dark mode.

---

## §10 Validation

After any batch of edits, run this Python sanity check:

```python
with open('dashboard.html') as f:
    s = f.read()

# 1. CSS brace balance
style = s[s.index('<style>') + 7 : s.index('</style>')]
opens, closes = style.count('{'), style.count('}')
assert opens == closes, f'CSS brace mismatch: {opens} vs {closes}'

# 2. Data blob intact (the embedded JSON)
assert 'D=JSON.parse(`{"meta"' in s, 'data blob corrupted'

# 3. No dead references from removed features
dead = ['var(--serif)', 'Instrument Serif', '.back-btn', 'physician-section',
        'phys-kpis', 'phys-stat', 'view-profile', 'buildPDD(', 'selP(', 'sPL', 'cP=']
for d in dead:
    assert d not in s, f'dead reference still present: {d}'

# 4. Core helpers and features
for fn in ['function foldN', 'function fold6', 'function palWithOthers',
           'function gradFill', 'function hAbbr', 'const HLOGO',
           'const HABBR', 'const OTHERS_COLOR', "VER='"]:
    assert fn in s, f'missing: {fn}'

# 5. Section title uses highlighter
assert 'linear-gradient(transparent 62%,rgba(235,23,0,0.18)' in s

# 6. Sticky header is orange and slim
assert 'padding:7px 16px' in s

# 7. All 5 doughnuts use 2px white border (gaps between slices)
assert s.count("borderWidth:2,borderColor:'#fff'") >= 4  # at least 4 (procedure×3 + physician×1)

# 8. Palette integrity
assert "'#A01528','#EB1700','#F04D5E'" in s  # stepped red
assert "'#1B395C','#234D79','#2C6097'" in s  # 10-step blue

print(f'All checks passed. Size: {len(s)} bytes')
```

If any assertion fails, stop and investigate before continuing. A corrupted data blob in particular is not recoverable by re-editing — you'd need to reinject from the Python pipeline.

---

## §11 Build script structure (`build_dashboard.py`)

The final deliverable. Merges the current two-step workflow into one.

```
build_dashboard.py
├── HTML_TEMPLATE = """..."""         # the full HTML with {{JSON_DATA}} placeholder
├── HLOGO_B64 = "iVBOR..."            # Biosense Webster white-on-orange logo
├── FAVICON_B64 = "iVBOR..."          # Biosense Webster black-on-white favicon
│
├── load_data(xlsx_path) -> pd.DataFrame
│     Read xlsx with header=8. Apply all filters from §4. Return cleaned df.
│
├── compute_aggregates(df) -> dict
│     Produce every JSON key listed in §5. Return the dict.
│
├── render_html(agg, template) -> str
│     1. Inject the logo base64 into the HLOGO JS constant
│     2. Inject the favicon base64 into the <link rel="icon"> tag
│     3. Inject json.dumps(agg, separators=(',',':')) at the {{JSON_DATA}} placeholder
│     4. Return final HTML string
│
└── main()
      Parse argv (xlsx path, --output flag)
      Call the three functions in sequence
      Write to output path (default: EP_Dashboard_<YYYY-MM-DD>.html)
```

### Design choices for the script

- **Single file**. No separate template.html, no data.json. Everything embedded as Python string constants. This is what makes `build_dashboard.py` self-contained — Eric can email the script itself to a colleague who can then generate the dashboard from their own xlsx.
- **Template placeholder**: use `{{JSON_DATA}}` (Jinja-style) not `{{JSON}}` or `$JSON` because it won't collide with any valid JS, CSS, or HTML token. A simple string `.replace('{{JSON_DATA}}', json_str)` is enough — no templating library dependency.
- **Logo injection**: same approach. Placeholders `{{HLOGO_B64}}` and `{{FAVICON_B64}}` in the template, substituted at render time.
- **Python dependencies**: `pandas`, `openpyxl` (for xlsx read). Nothing else. Document in a top-of-file docstring.

### Path forward for getting there

Currently the HTML has logo and JSON already embedded, not as placeholders. To extract:

1. Find the current HLOGO base64 string — replace it inline with `{{HLOGO_B64}}`, save the string elsewhere
2. Find the current favicon base64 — same treatment
3. Find the JSON blob (`const D=JSON.parse(`...`);`) — replace content with `{{JSON_DATA}}`
4. Copy everything into a Python triple-quoted string called `HTML_TEMPLATE`
5. The Python pipeline (already written, see Appendix A) becomes the `load_data` + `compute_aggregates` functions

The user may ask Claude Code to do this extraction as one of the later tasks. It's best done as one focused session, with careful validation that the rendered HTML matches the current HTML byte-for-byte when rendered from the same xlsx.

---

## §12 Open decisions / pending polish

These are items the user may raise in future sessions:

- **Dashboard title**: current is "LA Key Account Dashboard". May evolve.
- **Color palette**: may switch to official Biosense Webster brand colors if a style guide becomes available. Current cream+red+orange is a design choice, not mandated.
- **Territory expansion**: the current filter keeps only `Los Angeles, CA - KA`. Eric's Salesforce exports now also contain `LA County, CA` (a different territory). Future versions may support multi-territory mode.
- **Procedure list fold threshold**: currently Top 6 + Others for procedure doughnuts, Top 10 + Others for physician doughnut. These thresholds were debated. Expect Eric may tweak.
- **Hospital Comparison chart**: currently only CY. May need LY toggle if Eric wants trend comparisons.

Do not touch these without explicit direction.

---

## §13 Communication norms

- Eric writes in Traditional Chinese and English, often mixing. Respond in matching register.
- He values trade-off analysis over direct execution for design decisions. Offer 2–3 options when a choice is consequential.
- Sanity checks are welcome — he has run into bugs from ungated edits before.
- Version numbers in the footer (`VER` constant) bump with each meaningful release. The pattern is `v<major>.<minor>`. Minor bumps for any user-visible change.
- After implementation, always summarize what changed with a table — Eric uses this to test specific interactions.

---

## Appendix A — Current Python data pipeline (reference)

This is the working pipeline. It produces the JSON consumed by the HTML. Use as starting point for `compute_aggregates()` in the build script.

```python
import pandas as pd

def load_data(xlsx_path):
    df = pd.read_excel(xlsx_path, header=8)

    # Filter 1: drop Salesforce footer rows
    junk = df['Name'].isin(['Total']) | df['Name'].astype(str).str.contains(
        'Copyright|Confidential|Do Not Distribute', na=False)
    df = df[~junk]

    # Filter 2: territory
    df = df[df['Name'] == 'Los Angeles, CA - KA'].copy()

    # Filter 3: drop null date
    date_col = 'Actual End Date Time'
    df = df[df[date_col].notna()].copy()

    # Parse and filter date range
    df['date'] = pd.to_datetime(df[date_col], errors='coerce')
    df = df[df['date'].notna()].copy()
    df = df[df['date'] >= '2024-01-01'].copy()

    # Rename for brevity
    df = df.rename(columns={
        'Account: Account Name': 'hospital',
        'Physician: CARTODAY Affiliation Name': 'physician',
        'Primary Procedure: Work Type Name': 'procedure'
    })
    df['year'] = df['date'].dt.year
    df['month'] = df['date'].dt.month
    df['ym'] = df['date'].dt.strftime('%Y-%m')
    return df


def compute_aggregates(df):
    last_date = df['date'].max()
    last_year, last_month, last_day = int(last_date.year), int(last_date.month), int(last_date.day)

    def mtd_f(d, y, m, day): return d[(d['year']==y) & (d['month']==m) & (d['date'].dt.day <= day)]
    def ytd_f(d, y, day_end): return d[(d['year']==y) & (d['date'] <= pd.Timestamp(y, last_month, day_end))]

    mtd_cy, mtd_ly = mtd_f(df, last_year, last_month, last_day), mtd_f(df, last_year-1, last_month, last_day)
    ytd_cy, ytd_ly = ytd_f(df, last_year, last_day), ytd_f(df, last_year-1, last_day)

    hospitals_all = sorted(df['hospital'].unique())

    meta = {
        'last_date': last_date.strftime('%Y-%m-%d'),
        'last_year': last_year, 'last_month': last_month, 'last_day': last_day,
        'total_cases': len(df),
        'hospitals': hospitals_all,
        'n_hospitals': len(hospitals_all),
        'n_physicians': int(df['physician'].nunique()),
        'n_procedures': int(df['procedure'].nunique()),
        'active_hospitals': int(ytd_cy['hospital'].nunique()),
        'active_physicians': int(ytd_cy['physician'].nunique()),
    }

    region_ytd_total = len(ytd_cy)

    hospital_ranking_detail = []
    for h in hospitals_all:
        h_ytd = ytd_cy[ytd_cy['hospital'] == h]
        h_ly = ytd_ly[ytd_ly['hospital'] == h]
        if len(h_ytd) == 0: continue
        hospital_ranking_detail.append({
            'hospital': h,
            'count': int(len(h_ytd)),
            'md_count': int(h_ytd['physician'].nunique()),
            'pct_of_territory': round(len(h_ytd) / region_ytd_total * 100) if region_ytd_total else 0,
            'count_ly': int(len(h_ly))
        })
    hospital_ranking_detail.sort(key=lambda x: -x['count'])

    phys_ytd_grouped = ytd_cy.groupby('physician').size().sort_values(ascending=False).head(10)
    physician_ranking_detail = []
    for p, c in phys_ytd_grouped.items():
        physician_ranking_detail.append({
            'physician': p, 'count': int(c),
            'count_ly': int(len(ytd_ly[ytd_ly['physician'] == p]))
        })

    # Per-hospital primary for each physician (YTD, fallback all-time)
    phys_hospitals_grouped = ytd_cy.groupby(['physician', 'hospital']).size().reset_index(
        name='count').sort_values(['physician', 'count'], ascending=[True, False])
    primary = {}
    for _, r in phys_hospitals_grouped.iterrows():
        if r['physician'] not in primary:
            primary[r['physician']] = r['hospital']
    for p in df['physician'].unique():
        if p not in primary:
            sub = df[df['physician'] == p].groupby('hospital').size().sort_values(ascending=False)
            if len(sub) > 0:
                primary[p] = sub.index[0]

    agg = {
        'meta': meta,
        'region_monthly': [{'ym':ym,'count':int(c)} for ym,c in df.groupby('ym').size().sort_index().items()],
        'region_mtd_current': int(len(mtd_cy)),
        'region_mtd_ly': int(len(mtd_ly)),
        'region_ytd_current': region_ytd_total,
        'region_ytd_ly': int(len(ytd_ly)),
        'region_procedure_ytd': [{'procedure':p,'count':int(c)} for p,c in ytd_cy['procedure'].value_counts().items()],
        'region_procedure_ytd_ly': [{'procedure':p,'count':int(c)} for p,c in ytd_ly['procedure'].value_counts().items()],
        'hospital_ranking_detail': hospital_ranking_detail,
        'physician_ranking_detail': physician_ranking_detail,
        'hospital_monthly': [{'hospital':h,'ym':ym,'count':int(c)} for (h,ym),c in df.groupby(['hospital','ym']).size().sort_index().items()],
        'hospital_procedure_ytd': [{'hospital':h,'procedure':p,'count':int(c)} for (h,p),c in ytd_cy.groupby(['hospital','procedure']).size().items()],
        'hospital_physician_ytd': [{'hospital':h,'physician':p,'count':int(c)} for (h,p),c in ytd_cy.groupby(['hospital','physician']).size().items()],
        'hospital_physician_monthly': [{'hospital':h,'physician':p,'ym':ym,'count':int(c)} for (h,p,ym),c in df.groupby(['hospital','physician','ym']).size().sort_index().items()],
        'hospital_physician_procedure_ytd': [{'hospital':h,'physician':p,'procedure':pr,'count':int(c)} for (h,p,pr),c in ytd_cy.groupby(['hospital','physician','procedure']).size().items()],
        'hospital_ytd_ranking': [{'hospital':h['hospital'],'count':h['count']} for h in hospital_ranking_detail],
        'mtd_current': [{'hospital':h,'count':int(c)} for h,c in mtd_cy.groupby('hospital').size().items()],
        'mtd_ly': [{'hospital':h,'count':int(c)} for h,c in mtd_ly.groupby('hospital').size().items()],
        'ytd_current': [{'hospital':h,'count':int(c)} for h,c in ytd_cy.groupby('hospital').size().items()],
        'ytd_ly': [{'hospital':h,'count':int(c)} for h,c in ytd_ly.groupby('hospital').size().items()],
        'phys_mtd_current': [{'hospital':h,'physician':p,'count':int(c)} for (h,p),c in mtd_cy.groupby(['hospital','physician']).size().items()],
        'phys_mtd_ly': [{'hospital':h,'physician':p,'count':int(c)} for (h,p),c in mtd_ly.groupby(['hospital','physician']).size().items()],
        'phys_ytd_current': [{'hospital':h,'physician':p,'count':int(c)} for (h,p),c in ytd_cy.groupby(['hospital','physician']).size().items()],
        'phys_ytd_ly': [{'hospital':h,'physician':p,'count':int(c)} for (h,p),c in ytd_ly.groupby(['hospital','physician']).size().items()],
        'phys_all_ytd_ranking': [{'physician':p,'count':int(c)} for p,c in ytd_cy.groupby('physician').size().sort_values(ascending=False).items()],
        'phys_monthly_all': [{'physician':p,'ym':ym,'count':int(c)} for (p,ym),c in df.groupby(['physician','ym']).size().sort_index().items()],
        'phys_procedure_ytd_all': [{'physician':p,'procedure':pr,'count':int(c)} for (p,pr),c in ytd_cy.groupby(['physician','procedure']).size().items()],
        'phys_mtd_all': [{'physician':p,'count':int(c)} for p,c in mtd_cy.groupby('physician').size().items()],
        'phys_mtd_ly_all': [{'physician':p,'count':int(c)} for p,c in mtd_ly.groupby('physician').size().items()],
        'phys_ytd_all': [{'physician':p,'count':int(c)} for p,c in ytd_cy.groupby('physician').size().items()],
        'phys_ytd_ly_all': [{'physician':p,'count':int(c)} for p,c in ytd_ly.groupby('physician').size().items()],
        'phys_hospitals': [{'physician':r['physician'],'hospital':r['hospital'],'count':int(r['count'])} for _,r in phys_hospitals_grouped.iterrows()],
        'physician_primary_hospital': [{'physician':p,'hospital':h} for p,h in primary.items()],
    }
    return agg
```

---

## Appendix B — Version history (last 7 iterations)

| Version | Key changes |
|---------|-------------|
| v1.1 | Initial build — 3 pages, Chart.js, cream+red palette |
| v1.2 | Footer added (© + version + confidentiality); Hospital Tab repositioned below Hospital Comparison on Physician Detail; sticky header slimmed to 2/3 height; back button removed (navigation via tab bar/sidebar); Top 6 + Others for procedure doughnuts; legend line red |
| v1.3 | Footer border spans full scroll-area width (not constrained to 960px); email address and "Data as of" line removed |
| v1.4 | Section title font-size 16px → 20px; desktop `.chart-row` gained `margin-bottom:14px` (fixes Hospital Ranking butting against Monthly Trend) |
| v1.5 | Data refreshed to 2026-04-17 |
| v1.6 | All doughnut Others slices use `OTHERS_COLOR` (`#e8e5dc`) — fixes Top Physicians Others being indistinguishable from last palette color |
| v1.7 | Top Physicians doughnut: Top 6 + Others → **Top 10 + Others**; `PHC` palette expanded 6 → 10 colors (deep navy → pale sky); `foldN` generic helper added; dynamic subtitle `"Top 10 + Others"` vs `"All X physicians"` based on whether folding occurred |

---

End of specification.
