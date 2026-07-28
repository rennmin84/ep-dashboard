# EP Case Dashboard — 專案規格書 v5（中文版）

**版本**：對應 HTML v1.7 + build_dashboard.py v1.2（模板外部化 + splash screen + recent cases）  
**負責人**：Eric Wang，Biosense Webster 洛杉磯 Key Account CAS
**最後更新**：2026-04-26  
**狀態**：維護中

---

## 給 Claude 的操作說明

你是在 Claude Code / VS Code 中運行的 Claude。使用者會附上此規格書加上目前的 HTML 或 Python 檔案，請讀完後依照使用者的下一個需求繼續修改。

**三條工作原則**：

1. **用 `str_replace`，不要重寫整份檔案。** 模板 `dashboard.html.tmpl` 約 230KB（純 HTML/CSS/JS，不含 JSON/base64），`build_dashboard.py` 約 130KB（內嵌 favicon + 兩個 logo 的 base64）。修改任一檔案前先用小範圍 view 確認位置；若修改範圍超過 100 行，先停下來詢問使用者。

2. **先討論再動手。** Eric 偏好先看選項和取捨，再決定要做哪個。提供 2–3 種方案並列出優缺點，等他確認後再實作。他常用繁體中文回覆，請配合他的語言，但 UI 文字保持英文。

3. **每一批修改後做健全性檢查。** 用 §10 的 Python 驗證腳本確認 CSS 結構、資料 blob 和核心功能都完好。

---

## §1 專案目的

Eric 管理 Biosense Webster LA Key Account 區域。每週 Salesforce 會匯出一份案例記錄（`.xlsx`）。他希望有一個獨立的 HTML dashboard，可以：

- 在任何瀏覽器開啟（手機或桌機），不需安裝、不需伺服器
- 以單一檔案附件寄給同事或主管
- 每週用一個指令從最新的 Salesforce 匯出重新產生

Dashboard 顯示區域、醫院、醫師的績效，包含 YoY 比較、手術類型分佈，以及同時在多家醫院執業的醫師跨院分析。

---

## §2 最終交付物

**兩個檔案：**
- `EP_Dashboard_<date>.html`（位於 `dest/` 資料夾）— 版本備份
- `index.html`（根目錄）— GitHub Pages 固定網址

**產生指令：**
```bash
python3 build_dashboard.py "src/Case data.xlsx"          # 只產生檔案
python3 build_dashboard.py "src/Case data.xlsx" --push   # 產生 + 推送到 GitHub Pages
```

**GitHub Pages 網址**：`https://rennmin84.github.io/ep-dashboard/`

**專案檔案結構**：
- `build_dashboard.py` — Python 資料處理 + 模板注入（約 400 行）
- `dashboard.html.tmpl` — HTML/CSS/JS 模板（外部檔案，啟動時 `read_text` 載入）
- `resource/` — 啟動畫面與 logo 素材
  - `biosense-webster-inc-logo-vector.svg` — splash 用 BW logo（會被 `load_bw_svg()` 改色 + 裁框）
  - `all-gas.png`、`no-brakes.png` — splash 用 word-art（PIL 重存以去除 EXIF Orientation）
- `src/` — Salesforce xlsx 匯出檔
- `dest/` — 每次產出的版本備份 HTML
- `index.html` — 根目錄產出，GitHub Pages 服務

**Placeholder**（`render_html()` 會逐一替換）：
`{{JSON_DATA}}`、`{{FAVICON_B64}}`、`{{HLOGO_B64}}`、`{{BW_LOGO_B64}}`、`{{BW_SVG}}`、`{{ALL_GAS_B64}}`、`{{NO_BRAKES_B64}}`

---

## §3 使用者與使用情境

- **主要使用者**：Eric — 桌機（1440px+）和手機（iPhone）都會用
- **次要對象**：BWI 主管、區域總監 — Eric 可能會把 HTML 以 email 寄出作為報告
- **敏感資料**：包含醫師和醫院的案例活動。頁尾顯示「Confidential · Internal Use Only」。不可加入 email、電話，或任何超出 Salesforce 匯出範圍的個人資料。

---

## §4 資料來源

**輸入**：Salesforce 報告匯出，`.xlsx` 格式。

**檔案結構**：
- Sheet 名稱：`Case data`
- Header 在**第 9 行**（`header=8`）
- 資料從第 10 行開始
- 使用欄位：
  - `Name` — 區域名稱（篩選條件如下）
  - `Actual End Date Time` — 案例結束時間戳記
  - `Account: Account Name` — 醫院（重新命名為 `hospital`）
  - `Physician: CARTODAY Affiliation Name` — 醫師（重新命名為 `physician`）
  - `Primary CAS: Name` — 案例負責 CAS（重新命名為 `cas`，只用於 Recent Cases 列表，顯示時取 first name）
  - `Primary Procedure: Work Type Name` — 手術（重新命名為 `procedure`）

**Schema 驗證**：載入 xlsx 時 `_validate_schema()` 會檢查 `REQUIRED_COLUMNS` 是否全部存在；若 Salesforce 改名會立刻 raise 並列出實際與預期欄位（避免無聲產出錯誤的 dashboard）。

**篩選規則**（依序執行）：
1. 刪除 `Name` 為 `Total` 或包含 `Copyright`、`Confidential`、`Do Not Distribute` 的行（Salesforce footer 行）
2. 只保留 `Name == 'Los Angeles, CA - KA'`，其他區域如 `LA County, CA` 一律捨棄
3. 刪除 `Actual End Date Time` 為空的行（2020 年前的早期記錄常缺此欄位）
4. 用 `pd.to_datetime(errors='coerce')` 解析日期，再刪除 NaT
5. 只保留 `>= 2024-01-01` 的資料（dashboard 用 CY vs LY 比較，需要至少兩個完整年度）

**已知特殊情況**：
- Salesforce 有時在區域名稱後加空白（`Los Angeles, CA - KA `），已用 `.str.strip()` 處理
- `WST LOS ANGL VA MED  CENTER`（West LA VA）名稱中間有兩個空格，這是 Salesforce 的原始格式，請勿修改
- 某家醫院在當年 YTD 可能有零案例，視為非活躍（從活躍清單移除，如需要可保留在 all hospitals 中）

---

## §5 產生的聚合資料

Python pipeline 輸出一個 JSON 物件，所有聚合都是 plain list of dicts（JSON-friendly）。

### Metadata
```
meta: {
  last_date: "YYYY-MM-DD"
  last_year, last_month, last_day     # 整數
  total_cases                         # 篩選後總案例數
  hospitals                           # 所有醫院名稱（排序）
  n_hospitals, n_physicians, n_procedures    # 所有時間的唯一數量
  active_hospitals, active_physicians         # 當年有 ≥1 案例
}
```

### 區域層級
- `region_monthly` — `[{ym, count}]` 自 2024-01 起每個月
- `region_mtd_current`, `region_mtd_ly` — int，月至今案例數（period-matched LY）
- `region_mtd_ly_full` — int，去年整個對應月份案例數（不裁切到 `last_day`）
- `region_ytd_current`, `region_ytd_ly` — int，年至今案例數
- `region_procedure_ytd` — `[{procedure, count}]` 當年 YTD，降冪排列
- `region_procedure_ytd_ly` — 同上，去年

### 醫院層級
- `hospital_ranking_detail` — `[{hospital, count, md_count, pct_of_territory, count_ly}]`，當年 YTD，降冪，只含有案例的醫院
- `hospital_ytd_ranking` — 簡化版，只含 `{hospital, count}`，供下拉選單用
- `hospital_monthly` — `[{hospital, ym, count}]` 所有月份
- `hospital_procedure_ytd` — `[{hospital, procedure, count}]` 當年 YTD
- `mtd_current`, `mtd_ly`, `mtd_ly_full`, `ytd_current`, `ytd_ly` — `[{hospital, count}]`（`mtd_ly_full` 為去年整月）

### 醫院內醫師
- `hospital_physician_ytd` — `[{hospital, physician, count}]`
- `hospital_physician_monthly` — `[{hospital, physician, ym, count}]`
- `hospital_physician_procedure_ytd` — `[{hospital, physician, procedure, count}]`
- `phys_mtd_current`, `phys_mtd_ly`, `phys_mtd_ly_full`, `phys_ytd_current`, `phys_ytd_ly` — `[{hospital, physician, count}]`

### 醫師層級（所有醫院合計）
- `phys_all_ytd_ranking` — `[{physician, count}]` 當年 YTD，降冪
- `phys_monthly_all` — `[{physician, ym, count}]`
- `phys_procedure_ytd_all` — `[{physician, procedure, count}]`
- `phys_mtd_all`, `phys_mtd_ly_all`, `phys_mtd_ly_full_all`, `phys_ytd_all`, `phys_ytd_ly_all` — `[{physician, count}]`（`phys_mtd_ly_full_all` 為去年整月）
- `phys_hospitals` — `[{physician, hospital, count}]` YTD，按醫師排序後再按 count 降冪，用於識別跨院醫師
- `physician_primary_hospital` — `[{physician, hospital}]` 每位醫師的主要醫院（YTD 優先，fallback 全時間）

### Region Summary Top 10
- `physician_ranking_detail` — `[{physician, count, count_ly}]`，當年 YTD Top 10

### Recent Cases（最近案例明細）
最近 30 筆案例，按日期降冪排序，欄位經過字串處理（避免在前端再處理）：

每筆 row 結構：`{date: "M/D", year: int, hospital: str, physician_dr: "Dr. <Lastname>", type: <short_type>, cas: <first_name>}`

- `recent_cases_by_hospital` — `{<hospital>: [row, ...]}`，每家醫院最多 30 筆
- `recent_cases_by_physician` — `{<physician>: {<hospital>: [row, ...], "__all__": [row, ...]}}`
  - 每位醫師、每家醫院各取 30 筆；`__all__` key 收錄該醫師全部醫院合計的最近 30 筆
  - 跨院醫師在 Physician Detail 切換 Hospital tab 時用對應 key；`All` tab 用 `__all__`

**字串處理 helper**（在 `build_dashboard.py`）：
- `_norm_name(s)` — 全大寫名字轉首字大寫，並過濾單字母 middle initial（保留多字 middle name）
- `_dr_lastname(name)` — 「Dr. 」+ 姓
- `_first_name(name)` — first name only（用於 CAS 欄）
- `_short_date(d)` — `M/D`（無補零）
- `_short_type(t)` — `Structural Heart + ULS` → `ULS`，其餘原樣

### YoY 邏輯
- **MTD**：當月 1 日到 `last_day`，對比去年同期（例如 `2026-04-01` 到 `2026-04-17` vs `2025-04-01` 到 `2025-04-17`）
- **MTD 整月對比**：`*_mtd_ly_full` 系列為去年整個對應月份的合計，用於非 period-matched 的次要顯示
- **YTD**：`2026-01-01` 到 `2026-04-17`，對比 `2025-01-01` 到 `2025-04-17`

---

## §6 三個頁面的內容

導航：桌機（≥768px）左側 sidebar，手機底部 tab bar。任何頁面都沒有返回按鈕。

### 啟動畫面 — Splash Screen

開啟頁面時最先出現，2.9 秒後自動淡出消失（CSS transition 0.6s，DOM 在 3.4s 完全移除）。

- **背景**：紅色 radial gradient（`#FF1F00 → #C81100 → #A50E00`，從中心向外）
- **頂部**：Biosense Webster 白色 SVG logo（`{{BW_SVG}}`，由 `load_bw_svg()` 從 `resource/biosense-webster-inc-logo-vector.svg` 載入並把 `#38404A` 換成 `#ffffff`、裁切 viewBox）。寬度 `clamp(200px, 50vw, 280px)`。
- **中央**：兩張白字 word-art PNG（`ALL GAS.` / `NO BRAKES.`，間距 13px），高度 `clamp(43px, 10.8vw, 65px)`
- **底部**：細白橫線分隔 + `LA KEY ACCOUNT` 字樣（14px、字重 700、字距 2.8px）
- **動畫**：三段 `splashFade`（0s / 0.3s / 0.6s 依序淡入）

**規則**：splash 是品牌與情緒元素，動畫節奏與素材尺寸已調過，請勿任意改動。如需更換 word-art，重新匯出 PNG 後放回 `resource/`，build script 會自動 base64 嵌入。

### 頁面 1 — Region Summary

由上到下：
1. 頁面標題「Region Summary」＋副標題「Los Angeles, CA – KA」
2. **Territory banner**（Coverage YTD）— 2 個圓形圖示統計：醫院數、醫師數
3. **資料截止日** — 文字「Data through <日期>」＋顯示「TODAY」或「N DAYS AGO」的 pill
4. **Territory Performance** — 2 個 KPI 卡片（MTD、YTD），含 YoY delta
5. **Monthly Trend** — 折線圖，當年紅色實線加漸層填充，去年灰色虛線（「Show PY」按鈕切換）
6. **Trend & Procedure**（桌機雙欄）— 左：折線圖；右：「Procedure Type」甜甜圈圖＋手術清單
   - 手術清單顯示 Top 6 ＋ Others（若 >6）。每行：彩色圓點、名稱、數量、進度條、百分比、YoY pill
7. **Hospital Ranking** — 6 個醫院卡片格（桌機 2 欄，手機 1 欄），點擊跳轉 Hospital Detail
8. **Top Physicians** — Top 10 醫師清單（非圖表），點擊跳轉 Physician Detail

### 頁面 2 — Hospital Detail

1. 頁面標題「Hospital Detail」＋副標題「Individual Performance」
2. 醫院下拉選單
3. 資料截止日
4. **Performance** — 2 個 KPI 卡片（MTD、YTD）
5. **Monthly Volume** — 折線圖（同 Region，含 Show PY 切換）
6. **Case Analysis**（桌機雙欄）：
   - 左：甜甜圈「Procedure Type」（Top 6 ＋ Others）
   - 右：甜甜圈「Top Physicians」（**Top 10 ＋ Others**，藍色系）。圖表標頭有提示文字「Tap a slice to open physician」。點擊扇形跳轉到該醫師的 Physician Detail。
7. **Recent Cases** — 表格，欄位 `Date / Physician / Type / CAS`，最多 30 筆。資料來自 `recent_cases_by_hospital[<hospital>]`，跨年度時插入分隔列（mono 字型、淺背景、顯示年份）。

### 頁面 3 — Physician Detail

1. 頁面標題「Physician Detail」＋副標題「Individual Performance」
2. 醫師下拉選單 — 依 YTD 當年數量排列。跨院醫師在名稱下方顯示分院資訊，例如 `Carlos Macias · 49 / UCLA 34 · Saint John's 15`
3. 資料截止日
4. **Hospital Comparison section**（單一醫院執業者隱藏）— 折線圖，每家醫院一條線（僅當年，紅＋藍，無漸層填充）
5. **Hospital Tab** — `All / UCLA / Saint John's / ...`。單一醫院執業者隱藏 tab bar。
6. **Performance** — 2 個 KPI 卡片，依選擇的 tab 篩選
7. **Monthly Volume** — 折線圖，依 tab 篩選（含 Show PY 切換）
8. **Procedure Type** — 甜甜圈，依 tab 篩選（Top 6 ＋ Others）
9. **Recent Cases** — 表格，欄位 `Date / Hospital / Type / CAS`，最多 30 筆。資料依 tab 切換：`All` 用 `recent_cases_by_physician[<physician>]["__all__"]`，特定醫院 tab 用 `recent_cases_by_physician[<physician>][<hospital>]`。

---

## §7 設計系統

### 顏色（CSS 變數，定義於 `:root`）
```
--bg:       #fafaf5      奶油白（頁面背景）
--bg2:      #f5f5f0      略深的奶油色（圖示背景、tab 背景）
--text:     #1a1a1a      主要文字
--text2:    #5a574c      次要文字
--text3:    #8a8578      第三層（標籤）
--text4:    #a8a498      第四層（備注）
--border:   #e8e5dc      中性米色（預設邊框、Others 顏色）
--border2:  #f0ede3      較淺的邊框
--green:    #2d7a4f   --green-bright: #4fb37a   --green-bg: rgba(79,179,122,0.15)
--red:      #c93838   --red-bright:   #e86b6b   --red-bg:   rgba(232,107,107,0.15)
--gold:     #b4aa82   --gold-bg:      rgba(180,170,130,0.15)
--bw:       #EB1700      Biosense Webster 品牌紅（主色）
--bw-orange:#ee892d      Biosense Webster 品牌橘（sticky header、logo 背景）
```

### 調色盤（JS 常數）
```js
// 手術甜甜圈 + 進度條用的漸層紅
const PC = ['#A01528','#EB1700','#F04D5E','#F48091','#F8A8B5','#FAC4CC','#FCDDE2','#FEE8EC'];

// Top Physicians 甜甜圈用的 10 步藍（深 → 淺）
const PHC = ['#1B395C','#234D79','#2C6097','#3573B5','#4786C9','#659AD1','#83AEDA','#A1C2E3','#BFD5EC','#DDE9F5'];

// Others 扇形用的中性米色（與 --border 一致）
const OTHERS_COLOR = '#e8e5dc';
```

**規則**：甜甜圈的 Others 扇形必須用 `OTHERS_COLOR`，不可用調色盤的淺色端（否則視覺上難以區分）。使用 `palWithOthers(pal, n, hasOthers)` helper。

### 字型
- **Sans-serif**：`-apple-system, 'SF Pro Display', 'Helvetica Neue', Arial, sans-serif`（系統字型堆疊，不從 Google Fonts 載外部字型）
- **等寬**：`'SF Mono', Menlo, 'Courier New', monospace`（用於日期、Recent Cases 表格、年份分隔列）
- 兩者皆以 CSS 變數 `--sans` / `--mono` 定義於 `:root`
- **不使用 serif 字型**。Instrument Serif 已移除，不可重新引入。
- **不引入 Google Fonts**：早期版本曾用 DM Sans + JetBrains Mono，已改為純系統字型避免外部請求

### Section title（螢光筆樣式）
```css
.section-title {
  font-size: 20px;
  font-weight: 700;
  display: inline-block;
  background: linear-gradient(transparent 62%, rgba(235,23,0,0.18) 62%, rgba(235,23,0,0.18) 92%, transparent 92%);
  padding: 0 4px;
}
```
**沒有左側紅色邊條**。螢光筆色帶在文字下半部後方。每個頁面的每個 section header 都用這個樣式。

### Sticky header
- 背景：`var(--bw)`（`#EB1700`，BWI 品牌紅）
- Logo：18px 高（白色 Biosense heart，base64 PNG）
- 標題：13px，字重 700，白色
- 日期 pill：9px mono，白色 90% 透明度
- 任何頁面都沒有返回按鈕
- 註：早期版本曾使用橘色 `--bw-orange`，目前已統一為品牌紅，與 splash screen 視覺延續

### Sidebar（桌機，≥768px）
- 220px 固定寬度，左側
- 品牌：「LA Key Account」/「Biosense Webster」
- 導航項目：Region、Hospital、Physician — 圖示 + 標籤

### Footer（所有頁面）
- 全寬（不受 960px 限制）
- 內容置中於 `.page-footer-inner`（max-width 960px）
- 兩行：
  - `© <year> Eric Wang · LA Key Account Dashboard <VER>`
  - `Confidential · Internal Use Only · Source: Salesforce`
- 不加 email，不加「Data as of」日期

### 醫院名稱縮寫
```js
const HABBR = {
  'RONALD REAGAN UCLA MEDICAL CENTER': 'UCLA',
  'PROVIDENCE SAINT JOHNS HEALTH CENTER': "Saint John's",
  'KECK HOSPITAL OF USC': 'Keck',
  'USC ARCADIA HOSPITAL': 'Arcadia',
  'CEDARS SINAI MEDICAL CENTER': 'Cedars',
  'SOUTH CALIFORNIA PERMANENTE MEDICAL GROUP': 'Kaiser',
  'WST LOS ANGL VA MED  CENTER': 'West LA'   // 注意兩個空格
};
```

---

## §8 關鍵 JS Helper 函式

這些函式的簽名請保持穩定，其他程式碼依賴它們。

### `fold6(data, keyField)` 和 `foldN(data, keyField, n)`
將排序後的陣列折疊成 Top N ＋ Others。回傳：
```
{ labels, values, hasOthers, othersCount, othersTotal }
```
`fold6` 是 `foldN(..., 6)` 的薄包裝，用於手術甜甜圈（3 處）。
`foldN(..., 10)` 用於 Hospital Detail 的 Top Physicians 甜甜圈。

### `palWithOthers(pal, n, hasOthers)`
取調色盤前 `n` 個顏色。若 `hasOthers` 為 true，最後一個顏色替換為 `OTHERS_COLOR`。
**所有甜甜圈的 `backgroundColor` 都必須用這個函式**，不可直接 `PC.slice(0, n)`。

### `gradFill(color)`
回傳 Chart.js background callback，建立從 `color+'40'`（上方 25% 透明）到 `color+'00'`（下方 0%）的垂直漸層。用於 Monthly Volume 折線圖的當年線。
**規則**：不可用於去年虛線或 Hospital Comparison 圖（多線重疊漸層會混亂）。

### `hAbbr(h)` / `tc(s)` / `goH(h)` / `goPD(p)` / `uPDHComp()`
- `hAbbr`：醫院名稱縮寫
- `tc`：全大寫轉首字大寫
- `goH`：跳轉到 Hospital Detail
- `goPD`：跳轉到 Physician Detail
- `uPDHComp`：繪製 Physician Detail 的 Hospital Comparison 圖

---

## §9 互動模式

- **Event delegation 搭配 `data-*` 屬性**。不可用 inline `onclick` 加字串插值（引號跳脫很脆弱）。
- **折線圖 null 處理**：未來月份設為 `null`（不是 `0`），搭配 `spanGaps: true`，圖線在最後一個真實資料點後停止。
- **YoY pill 邊界情況**：
  - `count_ly === 0 && count > 0` → 顯示「NEW」（金色 pill）
  - `count_ly === 0 && count === 0` → 顯示「—」
  - 其他 → `(+|-)N%`，綠色或紅色
- **跨院醫師判斷**：`phys_hospitals` 中有超過一家醫院的資料，才顯示 Hospital Comparison 圖和 Hospital Tab。
- **iOS 深色模式防護**：`<meta name="color-scheme" content="light">` ＋ `html,body{background:#fafaf5}` ＋ `-webkit-text-size-adjust:100%`，不可移除。

---

## §10 驗證腳本

每批修改後執行：

```python
with open('index.html') as f:
    s = f.read()

style = s[s.index('<style>') + 7 : s.index('</style>')]
opens, closes = style.count('{'), style.count('}')
assert opens == closes, f'CSS brace mismatch: {opens} vs {closes}'

assert 'D=JSON.parse(`{"meta"' in s, 'data blob 損毀'

# 殘留 placeholder 檢查（這些都應該已被 render_html 替換掉）
for ph in ['{{JSON_DATA}}', '{{HLOGO_B64}}', '{{FAVICON_B64}}',
           '{{BW_LOGO_B64}}', '{{BW_SVG}}', '{{ALL_GAS_B64}}', '{{NO_BRAKES_B64}}']:
    assert ph not in s, f'未替換的 placeholder: {ph}'

dead = ['var(--serif)', 'Instrument Serif', '.back-btn', 'physician-section',
        'phys-kpis', 'phys-stat', 'view-profile', 'buildPDD(', 'selP(', 'sPL', 'cP=']
for d in dead:
    assert d not in s, f'殘留的 dead reference: {d}'

for fn in ['function foldN', 'function fold6', 'function palWithOthers',
           'function gradFill', 'function hAbbr', 'const HLOGO',
           'const HABBR', 'const OTHERS_COLOR', "VER='"]:
    assert fn in s, f'缺少: {fn}'

# Splash screen 必要元素
assert 'id="splash"' in s, '缺少 splash 容器'
assert 'splashFade' in s, '缺少 splash 動畫 keyframe'
assert 'LA KEY ACCOUNT' in s, '缺少 splash 字樣'

assert 'linear-gradient(transparent 62%,rgba(235,23,0,0.18)' in s
assert s.count("borderWidth:2,borderColor:'#fff'") >= 4
assert "'#A01528','#EB1700','#F04D5E'" in s
assert "'#1B395C','#234D79','#2C6097'" in s

print(f'全部檢查通過。大小: {len(s)} bytes')
```

---

## §11 build_dashboard.py 結構

```
build_dashboard.py
├── FAVICON_B64   = "..."   # 內嵌 base64
├── HLOGO_B64     = "..."   # Sticky header 用的白色 Biosense heart
├── BW_LOGO_B64   = "..."   # Footer 用的灰階 BW logo
├── HTML_TEMPLATE = (Path(...) / "dashboard.html.tmpl").read_text(...)
│                            # 啟動時從同層目錄讀取外部模板
├── REQUIRED_COLUMNS = [
│     "Name",
│     "Actual End Date Time",
│     "Account: Account Name",
│     "Physician: CARTODAY Affiliation Name",
│     "Primary CAS: Name",
│     "Primary Procedure: Work Type Name",
│   ]
│
├── 字串處理 helpers（給 Recent Cases 用）
│   ├── _norm_name(s)       — 全大寫 → 首字大寫，過濾單字 middle initial
│   ├── _dr_lastname(name)  — "Dr. <Lastname>"
│   ├── _first_name(name)   — first name only（CAS 欄位用）
│   ├── _short_date(d)      — "M/D"
│   └── _short_type(t)      — "Structural Heart + ULS" → "ULS"
│
├── _validate_schema(df, xlsx_path)
│     若 Salesforce 匯出欄位改名，立刻 raise 並列出實際與預期欄位
│
├── load_data(xlsx_path) -> pd.DataFrame
│     讀取 xlsx（sheet="Case data", header=8）→ schema 檢查
│     → 套用 §4 所有篩選 → rename `cas`/`physician` 並套 _norm_name
│     → 回傳清理後的 df（含 year/month/ym 衍生欄）
│
├── compute_aggregates(df) -> dict
│     產生 §5 所有 JSON key，包含：
│     - 區域 / 醫院 / 醫院內醫師 / 醫師層級各維度的 MTD/YTD（含 *_ly_full）
│     - hospital_ranking_detail（含 md_count、pct_of_territory、count_ly）
│     - phys_hospitals + physician_primary_hospital（跨院判斷）
│     - recent_cases_by_hospital / recent_cases_by_physician
│       （每組最多 30 筆，row 用上面字串 helper 預先處理好）
│
├── load_bw_svg(path="resource/biosense-webster-inc-logo-vector.svg") -> str
│     讀取 SVG → 把 #38404A 改 #ffffff（白）
│     → 改 viewBox 裁切（"-150 195 620 118" 只取文字字標部分）
│     → 為 <svg> 加上 width:100%;height:auto 樣式
│     → 找不到檔案時回傳空字串（不會 crash）
│
├── load_png_b64(path) -> str
│     讀取 PNG 並用 PIL 重存以去除 EXIF Orientation tag
│     （Affinity 匯出常帶旋轉資訊，瀏覽器會誤轉）
│     → base64 編碼字串；PIL 沒裝時跳過剝除步驟仍可執行
│
├── render_html(agg) -> str
│     依序替換 7 個 placeholder：FAVICON_B64、HLOGO_B64、BW_LOGO_B64、
│     BW_SVG（call load_bw_svg）、ALL_GAS_B64、NO_BRAKES_B64（call load_png_b64
│     讀 `resource/all-gas.png` 與 `resource/no-brakes-12.png`）、
│     JSON_DATA → 回傳完整 HTML
│
└── main()
      解析參數（xlsx 路徑、--push 旗標）
      若沒帶 xlsx，自動挑 `src/*.xlsx` 中最新的（忽略 `~$` 暫存檔）
      呼叫 load_data / compute_aggregates / render_html
      寫出 index.html（根目錄）和 dest/EP_Dashboard_<date>.html
      若 --push：git add → 若有 staged 變更才 commit → push
```

**設計選擇**：
- 模板與素材外部化：`dashboard.html.tmpl` + `resource/` 讓 HTML/CSS/JS 與 Python 邏輯分離，編輯模板不必動 Python 字串轉義
- Base64 logo 仍直接放在 `build_dashboard.py` 內（因尺寸小、變動少；如需更換可整段替換 `*_B64` 常數）
- Placeholder 使用 `{{NAME}}` 風格，不會與任何 JS/CSS/HTML token 衝突
- Schema 驗證用「raise + 列出欄位」失敗，避免 Salesforce 改欄名後產出無聲錯誤的 dashboard
- Python 依賴：`pandas`、`openpyxl`，可選 `Pillow`（沒裝時 EXIF 不會剝除，但仍能跑）

---

## §12 待決定事項

以下項目未來可能調整，**沒有明確指示前請勿修改**：

- **Dashboard 標題**：目前為「LA Key Account Dashboard」
- **色彩調色盤**：若 Biosense Webster 正式 style guide 釋出，可能調整
- **多區域支援**：Salesforce 匯出已包含 `LA County, CA`，未來可能支援多區域模式
- **手術清單折疊門檻**：目前手術甜甜圈 Top 6，醫師甜甜圈 Top 10
- **Hospital Comparison 圖**：目前只有當年，未來可能需要去年對比

---

## §13 溝通規範

- Eric 用繁體中文和英文混合書寫，請配合他的語言
- 設計決策優先提供 2–3 個選項和取捨分析，等確認後再實作
- 實作後用表格摘要變更內容，方便 Eric 針對性測試
- 版本號規則：`v<major>.<minor>`，任何使用者可見的變更都需要 minor bump

---

## 附錄 — 版本歷史

| 版本 | 主要變更 |
|------|---------|
| v1.1 | 初始版本 — 3 個頁面、Chart.js、奶油＋紅色調色盤 |
| v1.2 | Footer（© + 版本 + 保密聲明）；Hospital Tab 移到 Hospital Comparison 下方；sticky header 縮小；移除返回按鈕；手術甜甜圈 Top 6 ＋ Others |
| v1.3 | Footer 邊框跨全寬；移除 email 和「Data as of」行 |
| v1.4 | Section title 字級 16px → 20px；桌機 `.chart-row` 加 `margin-bottom:14px` |
| v1.5 | 資料更新至 2026-04-17 |
| v1.6 | 所有甜甜圈 Others 扇形統一用 `OTHERS_COLOR`（`#e8e5dc`） |
| v1.7 | Top Physicians 甜甜圈：Top 6 → **Top 10 ＋ Others**；`PHC` 調色盤擴充為 10 色；新增 `foldN` 通用 helper |
| v1.8 | 新增啟動畫面 splash screen（紅底 + BW logo + ALL GAS / NO BRAKES + LA KEY ACCOUNT，2.9s 自動淡出）；HTML 模板從 `build_dashboard.py` 拆出為獨立檔案 `dashboard.html.tmpl`；新增 `resource/` 資料夾存放 splash 素材；新增 helper `load_bw_svg`、`load_png_b64`、`_validate_schema` 與 `REQUIRED_COLUMNS` 常數；sticky header 背景由橘 `--bw-orange` 改為品牌紅 `--bw`，與 splash 視覺延續 |
| v1.9 | Hospital Detail 與 Physician Detail 新增 **Recent Cases** 表格（最多 30 筆，欄位 Date/Physician(or Hospital)/Type/CAS，跨年度插入年份分隔列）；`Primary CAS: Name` 加入 `REQUIRED_COLUMNS`；`compute_aggregates` 新增 `recent_cases_by_hospital`、`recent_cases_by_physician`，以及 `*_mtd_ly_full` 系列（去年整月對比）；新增字串處理 helper（`_norm_name`、`_dr_lastname`、`_first_name`、`_short_date`、`_short_type`）；折線圖切換按鈕文字由「Show LY」改為「Show PY」；字型堆疊改為純系統字型（SF Pro Display / SF Mono），移除 DM Sans 與 JetBrains Mono 的 Google Fonts 引用；splash word-art 高度由 `clamp(48px,12vw,72px)` 調整為 `clamp(43px,10.8vw,65px)`、間距 14px → 13px |
