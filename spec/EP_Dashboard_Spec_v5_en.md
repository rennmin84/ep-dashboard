# EP Case Dashboard — Project Specification v5 (English)

**Version**: HTML v1.7 + `build_dashboard.py` v1.2 (template externalized + splash screen + recent cases)
**Owner**: Eric Wang, Biosense Webster Key Account CAS, Los Angeles
**Last updated**: 2026-04-26
**Status**: Active — maintenance

---

## How to use this document

You are Claude in Claude Code / VS Code. The user will attach this spec plus the current HTML or Python file. Read both, then continue refinement based on the user's next request.

**Three operating rules**:

1. **Prefer `str_replace` over full rewrites.** The template `dashboard.html.tmpl` is ~230KB (pure HTML/CSS/JS, no JSON/base64); `build_dashboard.py` is ~130KB (embeds the favicon + two logo base64 strings). Before editing either file, run a small-range `view` first to confirm the location. If a change truly requires restructuring >100 lines, stop and ask the user first.

2. **Discuss before implementing.** Eric prefers to see options and trade-offs before code. Present 2–3 approaches with pros/cons, then wait for his pick. He often responds in Traditional Chinese — mirror that, but keep UI text in English.

3. **Sanity-check after every batch.** After a group of edits, run the Python validation script in §10 to confirm the CSS structure, data blob, and core features are still intact.

---

## §1 Project purpose

Eric manages the Biosense Webster LA Key Account territory. Each week Salesforce exports a case log as `.xlsx`. Eric wants a self-contained HTML dashboard he can:

- Open in any browser (mobile or desktop), no install, no server
- Send to colleagues / managers as a single file attachment
- Regenerate weekly from the latest Salesforce export with one command

The dashboard shows territory, hospital, and physician performance with YoY comparisons, procedure mixes, and cross-hospital analysis for physicians who operate at multiple sites.

---

## §2 Final deliverable

**Two output files:**
- `EP_Dashboard_<date>.html` (under `dest/`) — versioned snapshot
- `index.html` (repo root) — served by GitHub Pages at a stable URL

**Build commands:**
```bash
python3 build_dashboard.py "src/Case data.xlsx"          # generate files only
python3 build_dashboard.py "src/Case data.xlsx" --push   # generate + push to GitHub Pages
```

**GitHub Pages URL**: `https://rennmin84.github.io/ep-dashboard/`

**Project file layout**:
- `build_dashboard.py` — Python data pipeline + template injection (~400 lines)
- `dashboard.html.tmpl` — HTML/CSS/JS template (external file, loaded via `read_text` at startup)
- `resource/` — splash and logo assets
  - `biosense-webster-inc-logo-vector.svg` — BW logo used on splash (recolored + cropped by `load_bw_svg()`)
  - `all-gas.png`, `no-brakes.png` — splash word-art (re-saved with PIL to strip EXIF Orientation)
- `src/` — Salesforce xlsx exports
- `dest/` — versioned snapshot HTMLs (one per build)
- `index.html` — root-level output served by GitHub Pages

**Placeholders** (replaced one by one in `render_html()`):
`{{JSON_DATA}}`, `{{FAVICON_B64}}`, `{{HLOGO_B64}}`, `{{BW_LOGO_B64}}`, `{{BW_SVG}}`, `{{ALL_GAS_B64}}`, `{{NO_BRAKES_B64}}`

---

## §3 Users and use cases

- **Primary user**: Eric — opens on desktop (1440px+) and phone (iPhone)
- **Secondary audience**: BWI managers, regional directors — Eric may email the HTML as a one-file report
- **Sensitive data**: case-level physician and hospital activity. Footer reads "Confidential · Internal Use Only". Do not add email addresses, phone numbers, or any PII beyond what Salesforce exports.

---

## §4 Data source

**Input**: Salesforce report export, `.xlsx` format.

**File structure**:
- Sheet name: `Case data`
- Headers on **row 9** (`header=8`)
- Data starts row 10
- Columns used:
  - `Name` — territory name (filter rule below)
  - `Actual End Date Time` — case completion timestamp
  - `Account: Account Name` — hospital (renamed to `hospital`)
  - `Physician: CARTODAY Affiliation Name` — physician (renamed to `physician`)
  - `Primary CAS: Name` — case-owning CAS (renamed to `cas`; only used in Recent Cases, displayed as first name)
  - `Primary Procedure: Work Type Name` — procedure (renamed to `procedure`)

**Schema validation**: when loading the xlsx, `_validate_schema()` checks that every entry in `REQUIRED_COLUMNS` is present. If Salesforce renames a column, it raises immediately and prints the actual vs. expected columns, so we never silently produce a broken dashboard.

**Filter rules** (apply in this order):
1. Drop rows where `Name` is `Total`, or contains `Copyright`, `Confidential`, `Do Not Distribute` (Salesforce footer rows)
2. Keep only rows where `Name == 'Los Angeles, CA - KA'`. Other territories like `LA County, CA` must be discarded.
3. Drop rows where `Actual End Date Time` is null (early pre-2020 records often lack this field).
4. Parse date with `pd.to_datetime(errors='coerce')`, then drop NaT.
5. Keep only dates `>= 2024-01-01`. The dashboard uses CY vs LY, so two full years is the minimum retention.

**Known quirks**:
- Salesforce sometimes emits the territory name with trailing whitespace (`Los Angeles, CA - KA `). Already handled via `.str.strip()`.
- `WST LOS ANGL VA MED  CENTER` (West LA VA) has **two spaces** before "CENTER" — Salesforce's canonical form. Do not change it; it is used as a dictionary key.
- A hospital may have zero YTD cases in the current year. Treat it as inactive (drop from active lists, retain in all-hospital lists if needed).

---

## §5 Aggregates produced

The Python pipeline emits a single JSON object. Every aggregate is a plain list of dicts (JSON-friendly).

### Metadata
```
meta: {
  last_date: "YYYY-MM-DD"
  last_year, last_month, last_day     # integers
  total_cases                         # int, count after all filters
  hospitals                           # sorted list of all hospital names
  n_hospitals, n_physicians, n_procedures    # all-time unique counts
  active_hospitals, active_physicians         # had ≥1 case in current year
}
```

### Region-level
- `region_monthly` — `[{ym, count}]` every month since 2024-01
- `region_mtd_current`, `region_mtd_ly` — int, month-to-date cases (period-matched LY)
- `region_mtd_ly_full` — int, last year's *entire* matching month (not clipped to `last_day`)
- `region_ytd_current`, `region_ytd_ly` — int, year-to-date cases
- `region_procedure_ytd` — `[{procedure, count}]` YTD current year, sorted desc
- `region_procedure_ytd_ly` — same for last year

### Hospital-level
- `hospital_ranking_detail` — `[{hospital, count, md_count, pct_of_territory, count_ly}]`, YTD current year, sorted desc. Only includes hospitals with ≥1 YTD case.
- `hospital_ytd_ranking` — simplified `{hospital, count}` for dropdowns
- `hospital_monthly` — `[{hospital, ym, count}]` all months
- `hospital_procedure_ytd` — `[{hospital, procedure, count}]` current YTD
- `mtd_current`, `mtd_ly`, `mtd_ly_full`, `ytd_current`, `ytd_ly` — `[{hospital, count}]` (`mtd_ly_full` is the full last-year month)

### Physician-in-hospital
- `hospital_physician_ytd` — `[{hospital, physician, count}]`
- `hospital_physician_monthly` — `[{hospital, physician, ym, count}]`
- `hospital_physician_procedure_ytd` — `[{hospital, physician, procedure, count}]`
- `phys_mtd_current`, `phys_mtd_ly`, `phys_mtd_ly_full`, `phys_ytd_current`, `phys_ytd_ly` — `[{hospital, physician, count}]`

### Physician-level (all-hospital aggregated)
- `phys_all_ytd_ranking` — `[{physician, count}]` YTD current, sorted desc
- `phys_monthly_all` — `[{physician, ym, count}]`
- `phys_procedure_ytd_all` — `[{physician, procedure, count}]`
- `phys_mtd_all`, `phys_mtd_ly_all`, `phys_mtd_ly_full_all`, `phys_ytd_all`, `phys_ytd_ly_all` — `[{physician, count}]` (`phys_mtd_ly_full_all` is the full last-year month)
- `phys_hospitals` — `[{physician, hospital, count}]` YTD, sorted by physician then count desc. Used to identify cross-hospital physicians.
- `physician_primary_hospital` — `[{physician, hospital}]` each physician's primary hospital (YTD-first, falls back to all-time)

### Top 10 for Region Summary
- `physician_ranking_detail` — `[{physician, count, count_ly}]`, top 10 by YTD current year

### Recent Cases
The 30 most recent cases, sorted by date desc, with string fields pre-processed in Python (so the frontend doesn't have to re-process them):

Each row: `{date: "M/D", year: int, hospital: str, physician_dr: "Dr. <Lastname>", type: <short_type>, cas: <first_name>}`

- `recent_cases_by_hospital` — `{<hospital>: [row, ...]}`, up to 30 rows per hospital
- `recent_cases_by_physician` — `{<physician>: {<hospital>: [row, ...], "__all__": [row, ...]}}`
  - 30 rows per physician × hospital; the `__all__` key holds the most recent 30 across all of that physician's hospitals
  - On Physician Detail, switching the Hospital tab uses the matching key; the `All` tab uses `__all__`

**String helpers** (in `build_dashboard.py`):
- `_norm_name(s)` — uppercase name → title case, filtering out single-letter middle initials (multi-word middle names are kept)
- `_dr_lastname(name)` — `"Dr. " + lastname`
- `_first_name(name)` — first name only (used in the CAS column)
- `_short_date(d)` — `M/D` (no leading zeros)
- `_short_type(t)` — `Structural Heart + ULS` → `ULS`; otherwise unchanged

### YoY logic
- **MTD (period-matched)**: day 1 of the current month through `last_day`, compared to the same date range in last year (e.g., `2026-04-01` to `2026-04-17` vs. `2025-04-01` to `2025-04-17`)
- **MTD (full-month)**: the `*_mtd_ly_full` series holds last year's entire matching month, for any secondary display that isn't period-matched
- **YTD**: `2026-01-01` to `2026-04-17` vs. `2025-01-01` to `2025-04-17`

---

## §6 Three pages, what each contains

Navigation: sidebar on desktop (≥768px), tab bar on mobile. No back button on any page.

### Splash screen

First thing shown when the page opens; auto-fades after 2.9s (CSS transition 0.6s, DOM removed at 3.4s).

- **Background**: red radial gradient (`#FF1F00 → #C81100 → #A50E00`, center → edge)
- **Top**: Biosense Webster white SVG logo (`{{BW_SVG}}`, produced by `load_bw_svg()` reading `resource/biosense-webster-inc-logo-vector.svg`, recoloring `#38404A` to `#ffffff`, and cropping the viewBox). Width `clamp(200px, 50vw, 280px)`.
- **Middle**: two white word-art PNGs (`ALL GAS.` / `NO BRAKES.`, 13px gap), height `clamp(43px, 10.8vw, 65px)`
- **Bottom**: thin white separator + `LA KEY ACCOUNT` text (14px, weight 700, letter-spacing 2.8px)
- **Animation**: three-stage `splashFade` (0s / 0.3s / 0.6s staggered fade-in)

**Rule**: the splash is a brand/emotion element; timing and asset sizes have been tuned. Do not change them ad-hoc. To swap word-art, re-export the PNG into `resource/` — the build script base64-embeds it automatically.

### Page 1 — Region Summary

Top to bottom:
1. Page title "Region Summary" + subtitle "Los Angeles, CA – KA"
2. **Territory banner** (Coverage YTD) — 2 circular-icon stats: hospitals count, physicians count
3. **Data-through line** — text "Data through <date>" + pill showing "TODAY" or "N DAYS AGO"
4. **Territory Performance** — 2 KPI cards (MTD, YTD) with YoY delta
5. **Monthly Trend** — line chart, CY red solid with gradient fill, LY dashed gray (toggle via "Show PY" button)
6. **Trend & Procedure** (desktop 2-up) — left: the line chart; right: doughnut "Procedure Type" + list
   - Procedure list shows Top 6 + Others (if >6). Each row: colored dot, name, count, progress bar, %, YoY pill
7. **Hospital Ranking** — grid of 6 hospital cards (2 cols desktop, 1 col mobile); click to navigate to Hospital Detail
8. **Top Physicians** — list of top 10 physicians (not a chart); click to navigate to Physician Detail

### Page 2 — Hospital Detail

1. Page title "Hospital Detail" + subtitle "Individual Performance"
2. Hospital dropdown
3. Data-through line
4. **Performance** — 2 KPI cards (MTD, YTD)
5. **Monthly Volume** — line chart (same as Region; Show PY toggle, gradient fill on CY)
6. **Case Analysis** (desktop 2-up):
   - Left: doughnut "Procedure Type" (Top 6 + Others)
   - Right: doughnut "Top Physicians" (**Top 10 + Others**, blue palette). Hint text "Tap a slice to open physician" in the chart header. Clicking a slice navigates to that physician's Physician Detail page.
7. **Recent Cases** — table with columns `Date / Physician / Type / CAS`, up to 30 rows. Data comes from `recent_cases_by_hospital[<hospital>]`. When rows span more than one year, a year-separator row is inserted (mono font, light background, shows the year).

### Page 3 — Physician Detail

1. Page title "Physician Detail" + subtitle "Individual Performance"
2. Physician dropdown — sorted by YTD current count. For cross-hospital physicians, the dropdown item shows per-hospital breakdown below the name, e.g., `Carlos Macias · 49 / UCLA 34 · Saint John's 15`
3. Data-through line
4. **Hospital Comparison section** (hidden for single-hospital physicians) — line chart with one line per hospital (CY only, red + blue, no gradient fill)
5. **Hospital Tab** — `All / UCLA / Saint John's / ...`. Hidden entirely for single-hospital physicians.
6. **Performance** — 2 KPI cards scoped to the selected tab
7. **Monthly Volume** — line chart scoped to the selected tab (Show PY toggle)
8. **Procedure Type** — doughnut scoped to the selected tab (Top 6 + Others)
9. **Recent Cases** — table with columns `Date / Hospital / Type / CAS`, up to 30 rows. Data switches with the tab: `All` uses `recent_cases_by_physician[<physician>]["__all__"]`; a specific hospital tab uses `recent_cases_by_physician[<physician>][<hospital>]`.

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
--bw-orange:#ee892d      Biosense Webster brand orange (legacy; logo background)
```

### Palettes (JS constants)
```js
// Stepped red for procedure doughnuts + progress bars
const PC = ['#A01528','#EB1700','#F04D5E','#F48091','#F8A8B5','#FAC4CC','#FCDDE2','#FEE8EC'];

// 10-step blue for Top Physicians doughnut (deep → light)
const PHC = ['#1B395C','#234D79','#2C6097','#3573B5','#4786C9','#659AD1','#83AEDA','#A1C2E3','#BFD5EC','#DDE9F5'];

// Neutral beige for Others slice (matches --border)
const OTHERS_COLOR = '#e8e5dc';
```

**Rule**: when a doughnut has an Others slice, its color must be `OTHERS_COLOR`, never a pale shade of the main palette (pale shades blend into adjacent medium shades). Use `palWithOthers(pal, n, hasOthers)`.

### Typography
- **Sans-serif**: `-apple-system, 'SF Pro Display', 'Helvetica Neue', Arial, sans-serif` (system font stack — no external font loading)
- **Monospace**: `'SF Mono', Menlo, 'Courier New', monospace` (used for dates, Recent Cases tables, year-separator rows)
- Both exposed as CSS variables `--sans` / `--mono` on `:root`
- **No serif fonts**. Instrument Serif has been removed; do not reintroduce.
- **No Google Fonts**. Earlier builds used DM Sans + JetBrains Mono via Google Fonts — replaced with the system stack to remove external requests.

### Section title (highlighter style)
```css
.section-title {
  font-size: 20px;
  font-weight: 700;
  display: inline-block;
  background: linear-gradient(transparent 62%, rgba(235,23,0,0.18) 62%, rgba(235,23,0,0.18) 92%, transparent 92%);
  padding: 0 4px;
}
```
**No left red bar**. The highlighter band sits behind the lower half of the text. Used for every section header on every page.

### Sticky header
- Background: `var(--bw)` (`#EB1700`, BWI brand red)
- Logo: 18px tall (white Biosense heart, base64 PNG)
- Title: 13px, weight 700, white
- Date pill: 9px mono, white at 90% opacity
- No back button on any page
- Note: earlier builds used `--bw-orange`; unified to brand red so the sticky header carries the splash visuals through to the rest of the UI

### Sidebar (desktop, ≥768px)
- 220px fixed width, left side
- Brand: "LA Key Account" / "Biosense Webster"
- Nav items: Region, Hospital, Physician — icon + label

### Footer (all pages)
- Full width (not constrained to 960px)
- Content centered in `.page-footer-inner` (max-width 960px)
- Two lines:
  - `© <year> Eric Wang · LA Key Account Dashboard <VER>`
  - `Confidential · Internal Use Only · Source: Salesforce`
- No email. No "Data as of" date.

### Hospital name abbreviations
```js
const HABBR = {
  'RONALD REAGAN UCLA MEDICAL CENTER': 'UCLA',
  'PROVIDENCE SAINT JOHNS HEALTH CENTER': "Saint John's",
  'KECK HOSPITAL OF USC': 'Keck',
  'USC ARCADIA HOSPITAL': 'Arcadia',
  'CEDARS SINAI MEDICAL CENTER': 'Cedars',
  'SOUTH CALIFORNIA PERMANENTE MEDICAL GROUP': 'Kaiser',
  'WST LOS ANGL VA MED  CENTER': 'West LA'   // note: two spaces
};
```

---

## §8 Key helper functions (JS)

Keep these signatures stable — other code depends on them.

### `fold6(data, keyField)` and `foldN(data, keyField, n)`
Folds a sorted array into Top N + Others. Returns:
```
{ labels, values, hasOthers, othersCount, othersTotal }
```
`fold6` is a thin wrapper around `foldN(..., 6)` for procedure doughnuts (3 places).
`foldN(..., 10)` is used for the Top Physicians doughnut on Hospital Detail.

### `palWithOthers(pal, n, hasOthers)`
Returns the first `n` colors from `pal`. If `hasOthers` is true, replaces the last color with `OTHERS_COLOR`.
**Every doughnut `backgroundColor` must use this helper** — never `PC.slice(0, n)` directly.

### `gradFill(color)`
Returns a Chart.js background callback that creates a vertical linear gradient from `color + '40'` (25% alpha at top) to `color + '00'` (0% alpha at bottom). Used for the CY line on Monthly Volume charts.
**Rule**: do NOT apply to dashed LY lines or to the Hospital Comparison chart (overlapping gradients muddy the visuals).

### `hAbbr(h)` / `tc(s)` / `goH(h)` / `goPD(p)` / `uPDHComp()`
- `hAbbr` — hospital name abbreviator (via `HABBR`)
- `tc` — title-case an all-caps string
- `goH` — navigate to Hospital Detail
- `goPD` — navigate to Physician Detail
- `uPDHComp` — draws the Hospital Comparison chart on Physician Detail

---

## §9 Interaction patterns

- **Event delegation with `data-*` attributes**. Do NOT use inline `onclick` with string interpolation — quote-escaping is fragile.
- **Line chart null handling**: for months in the future relative to `last_date`, set the data to `null` (not `0`); with `spanGaps: true` the line stops after the last real data point.
- **YoY pill edge cases**:
  - `count_ly === 0 && count > 0` → "NEW" (gold pill)
  - `count_ly === 0 && count === 0` → "—"
  - Otherwise → `(+|-)N%`, green or red
- **Cross-hospital physician detection**: a physician with `phys_hospitals` entries spanning more than one hospital. Only these get the Hospital Comparison chart + Hospital Tab.
- **iOS dark-mode protection**: `<meta name="color-scheme" content="light">` + `html,body{background:#fafaf5}` + `-webkit-text-size-adjust:100%`. Do not remove — without them, iOS Safari may invert colors in dark mode.

---

## §10 Validation script

Run after each batch of edits:

```python
with open('index.html') as f:
    s = f.read()

style = s[s.index('<style>') + 7 : s.index('</style>')]
opens, closes = style.count('{'), style.count('}')
assert opens == closes, f'CSS brace mismatch: {opens} vs {closes}'

assert 'D=JSON.parse(`{"meta"' in s, 'data blob corrupted'

# Leftover placeholders (all should have been replaced by render_html)
for ph in ['{{JSON_DATA}}', '{{HLOGO_B64}}', '{{FAVICON_B64}}',
           '{{BW_LOGO_B64}}', '{{BW_SVG}}', '{{ALL_GAS_B64}}', '{{NO_BRAKES_B64}}']:
    assert ph not in s, f'unreplaced placeholder: {ph}'

dead = ['var(--serif)', 'Instrument Serif', '.back-btn', 'physician-section',
        'phys-kpis', 'phys-stat', 'view-profile', 'buildPDD(', 'selP(', 'sPL', 'cP=']
for d in dead:
    assert d not in s, f'dead reference still present: {d}'

for fn in ['function foldN', 'function fold6', 'function palWithOthers',
           'function gradFill', 'function hAbbr', 'const HLOGO',
           'const HABBR', 'const OTHERS_COLOR', "VER='"]:
    assert fn in s, f'missing: {fn}'

# Splash screen required elements
assert 'id="splash"' in s, 'missing splash container'
assert 'splashFade' in s, 'missing splash animation keyframe'
assert 'LA KEY ACCOUNT' in s, 'missing splash text'

assert 'linear-gradient(transparent 62%,rgba(235,23,0,0.18)' in s
assert s.count("borderWidth:2,borderColor:'#fff'") >= 4
assert "'#A01528','#EB1700','#F04D5E'" in s
assert "'#1B395C','#234D79','#2C6097'" in s

print(f'All checks passed. Size: {len(s)} bytes')
```

---

## §11 `build_dashboard.py` structure

```
build_dashboard.py
├── FAVICON_B64   = "..."   # embedded base64
├── HLOGO_B64     = "..."   # white Biosense heart for sticky header
├── BW_LOGO_B64   = "..."   # grayscale BW logo for footer
├── HTML_TEMPLATE = (Path(...) / "dashboard.html.tmpl").read_text(...)
│                            # external template loaded at startup from the same directory
├── REQUIRED_COLUMNS = [
│     "Name",
│     "Actual End Date Time",
│     "Account: Account Name",
│     "Physician: CARTODAY Affiliation Name",
│     "Primary CAS: Name",
│     "Primary Procedure: Work Type Name",
│   ]
│
├── String helpers (for Recent Cases)
│   ├── _norm_name(s)       — all-caps → title case, drops single-letter middle initials
│   ├── _dr_lastname(name)  — "Dr. <Lastname>"
│   ├── _first_name(name)   — first name only (used for CAS column)
│   ├── _short_date(d)      — "M/D"
│   └── _short_type(t)      — "Structural Heart + ULS" → "ULS"
│
├── _validate_schema(df, xlsx_path)
│     If Salesforce renames a column, raise immediately and print actual vs. expected
│
├── load_data(xlsx_path) -> pd.DataFrame
│     Read xlsx (sheet="Case data", header=8) → schema check
│     → apply every §4 filter → rename `cas`/`physician` and apply _norm_name
│     → return cleaned df (with year/month/ym derived columns)
│
├── compute_aggregates(df) -> dict
│     Produce every §5 JSON key, including:
│     - region / hospital / physician-in-hospital / physician-level MTD/YTD
│       (including the *_ly_full series)
│     - hospital_ranking_detail (with md_count, pct_of_territory, count_ly)
│     - phys_hospitals + physician_primary_hospital (cross-hospital detection)
│     - recent_cases_by_hospital / recent_cases_by_physician
│       (up to 30 rows each, pre-processed via the string helpers above)
│
├── load_bw_svg(path="resource/biosense-webster-inc-logo-vector.svg") -> str
│     Read SVG → replace #38404A with #ffffff (white)
│     → rewrite viewBox to crop ("-150 195 620 118", text mark only)
│     → add width:100%;height:auto on <svg>
│     → return "" if the file is missing (no crash)
│
├── load_png_b64(path) -> str
│     Read PNG and re-save via PIL to strip EXIF Orientation
│     (Affinity exports often carry rotation metadata that browsers misapply)
│     → base64-encoded string; if PIL is absent, skip the strip step and continue
│
├── render_html(agg) -> str
│     Replace each of 7 placeholders in order: FAVICON_B64, HLOGO_B64, BW_LOGO_B64,
│     BW_SVG (via load_bw_svg), ALL_GAS_B64, NO_BRAKES_B64 (via load_png_b64 on
│     `resource/all-gas.png` and `resource/no-brakes-12.png`),
│     JSON_DATA → return the final HTML
│
└── main()
      Parse argv (xlsx path, --push flag)
      If no xlsx is given, pick the newest `src/*.xlsx` (ignoring `~$` temp files)
      Call load_data / compute_aggregates / render_html
      Write index.html (root) and dest/EP_Dashboard_<date>.html
      With --push: git add → only commit if there are staged changes → push
```

**Design choices**:
- Template + assets externalized: `dashboard.html.tmpl` + `resource/` keep HTML/CSS/JS separate from Python logic, so editing the template doesn't fight Python string escaping
- Base64 logos still live inside `build_dashboard.py` (small, rarely change; swap by replacing the `*_B64` constant)
- Placeholder style `{{NAME}}` won't collide with any valid JS/CSS/HTML token
- Schema validation fails loudly with "raise + list columns", so a Salesforce column rename never produces a silently wrong dashboard
- Python deps: `pandas`, `openpyxl`, optional `Pillow` (without it the EXIF strip step is skipped but the build still runs)

---

## §12 Open decisions / pending polish

These items may evolve in future sessions. **Do not change without explicit direction:**

- **Dashboard title**: currently "LA Key Account Dashboard"
- **Color palette**: may switch to official Biosense Webster brand colors if a style guide becomes available
- **Multi-territory support**: Salesforce exports now also contain `LA County, CA`; future versions may support multi-territory mode
- **Fold thresholds**: procedure doughnut at Top 6, physician doughnut at Top 10
- **Hospital Comparison chart**: currently CY only; may add LY toggle if Eric wants trend comparisons

---

## §13 Communication norms

- Eric writes in a mix of Traditional Chinese and English — match his register
- For design decisions, lead with 2–3 options and trade-offs; wait for his pick before implementing
- After implementation, summarize what changed in a table so Eric can target tests
- Version numbers follow `v<major>.<minor>`; any user-visible change gets a minor bump

---

## Appendix — Version history

| Version | Key changes |
|---------|-------------|
| v1.1 | Initial build — 3 pages, Chart.js, cream+red palette |
| v1.2 | Footer (© + version + confidentiality); Hospital Tab moved below Hospital Comparison; sticky header slimmed; back button removed; procedure doughnut Top 6 + Others |
| v1.3 | Footer border spans full viewport; email address and "Data as of" line removed |
| v1.4 | Section title 16px → 20px; desktop `.chart-row` gained `margin-bottom:14px` |
| v1.5 | Data refreshed through 2026-04-17 |
| v1.6 | All doughnut Others slices unified on `OTHERS_COLOR` (`#e8e5dc`) |
| v1.7 | Top Physicians doughnut: Top 6 → **Top 10 + Others**; `PHC` palette expanded to 10 steps; generic `foldN` helper added |
| v1.8 | Added splash screen (red background + BW logo + ALL GAS / NO BRAKES + LA KEY ACCOUNT, 2.9s auto-fade); HTML template extracted from `build_dashboard.py` into standalone `dashboard.html.tmpl`; added `resource/` directory for splash assets; new helpers `load_bw_svg`, `load_png_b64`, `_validate_schema`, and `REQUIRED_COLUMNS` constant; sticky header background changed from orange `--bw-orange` to brand red `--bw` to carry the splash visuals through |
| v1.9 | Hospital Detail and Physician Detail gained **Recent Cases** tables (up to 30 rows, columns Date/Physician(or Hospital)/Type/CAS, year-separator row when rows cross years); `Primary CAS: Name` added to `REQUIRED_COLUMNS`; `compute_aggregates` gained `recent_cases_by_hospital`, `recent_cases_by_physician`, and the `*_mtd_ly_full` series (last year's full matching month); added string helpers (`_norm_name`, `_dr_lastname`, `_first_name`, `_short_date`, `_short_type`); line-chart toggle relabeled "Show LY" → "Show PY"; font stack switched to pure system fonts (SF Pro Display / SF Mono), removing the DM Sans + JetBrains Mono Google Fonts imports; splash word-art height tuned from `clamp(48px,12vw,72px)` to `clamp(43px,10.8vw,65px)`, gap 14px → 13px |
