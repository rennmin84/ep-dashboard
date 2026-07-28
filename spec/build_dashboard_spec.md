# build_dashboard.py — Spec

**Version**: 1.0  
**Owner**: Eric Wang, Biosense Webster Key Account CAS, Los Angeles  
**Last updated**: April 2026

---

## 目的

將 Salesforce 匯出的 `.xlsx` 案例報表，轉換成可獨立開啟的單一 HTML dashboard。不需要網路、不需要伺服器，雙擊即可在瀏覽器瀏覽。

---

## 依賴套件

```bash
pip3 install pandas openpyxl
```

| 套件 | 用途 |
|------|------|
| `pandas` | 讀取 Excel、資料篩選與聚合 |
| `openpyxl` | pandas 讀取 `.xlsx` 的後端引擎 |

標準函式庫：`json`, `argparse`, `subprocess`, `datetime`, `os`

---

## 使用方式

```bash
# 基本用法（輸出到當前資料夾）
python3 build_dashboard.py "src/Case data.xlsx"

# 產生後自動 commit + push 到 GitHub Pages
python3 build_dashboard.py "src/Case data.xlsx" --push
```

### 參數

| 參數 | 必填 | 說明 |
|------|------|------|
| `xlsx` | 是 | Salesforce 匯出的 `.xlsx` 路徑 |
| `--push` | 否 | 完成後自動 git commit + push |

---

## 輸出檔案

每次執行固定產生兩個檔案：

| 檔案 | 位置 | 說明 |
|------|------|------|
| `index.html` | 根目錄 | GitHub Pages 固定網址用，每次覆寫 |
| `EP_Dashboard_<YYYY-MM-DD>.html` | `dest/` 資料夾 | 版本備份，每週一個 |

`dest/` 資料夾若不存在會自動建立。

**GitHub Pages 網址**：`https://rennmin84.github.io/ep-dashboard/`

---

## 執行流程

```
[1/4] 讀取 Excel
[2/4] 計算聚合資料
[3/4] 渲染 HTML
[4/4] 寫出檔案
[5/5] Push（僅 --push 時執行）
```

### Step 1 — load_data(xlsx_path)

- 讀取 sheet `Case data`，header 在第 9 行（`header=8`）
- 篩選規則（依序執行）：
  1. 刪除 `Name` 為 `Total` 或含 `Copyright / Confidential / Do Not Distribute` 的 footer 行
  2. 只保留 `Name == 'Los Angeles, CA - KA'`（使用 `.str.strip()` 避免空白）
  3. 刪除 `Actual End Date Time` 為空的行
  4. 解析日期，刪除解析失敗的行（NaT）
  5. 只保留 `>= 2024-01-01` 的資料
- 欄位重命名：
  - `Account: Account Name` → `hospital`
  - `Physician: CARTODAY Affiliation Name` → `physician`
  - `Primary Procedure: Work Type Name` → `procedure`
- 新增欄位：`year`, `month`, `ym`（格式 `YYYY-MM`）

### Step 2 — compute_aggregates(df)

產生一個 dict，包含以下 key，全部為 JSON-friendly 的 list of dicts：

**Metadata（`meta`）**

| Key | 型別 | 說明 |
|-----|------|------|
| `last_date` | str | 最新案例日期 `YYYY-MM-DD` |
| `last_year/month/day` | int | 最新日期的年/月/日 |
| `total_cases` | int | 篩選後總案例數 |
| `hospitals` | list | 所有醫院名稱（排序） |
| `n_hospitals/physicians/procedures` | int | 所有時間的唯一數量 |
| `active_hospitals/physicians` | int | 當年 YTD 有案例的數量 |

**YoY 比較邏輯**

- **MTD**：當年當月 1 日到 `last_day`，vs 去年同期
- **YTD**：當年 1/1 到 `last_month/last_day`，vs 去年同期

**主要聚合 key**

| Key | 說明 |
|-----|------|
| `region_monthly` | `[{ym, count}]` 每月案例數 |
| `region_mtd_current / ly` | int，region MTD |
| `region_ytd_current / ly` | int，region YTD |
| `region_procedure_ytd / ly` | `[{procedure, count}]` |
| `hospital_ranking_detail` | `[{hospital, count, md_count, pct_of_territory, count_ly}]` YTD，只含有案例的醫院 |
| `hospital_ytd_ranking` | `[{hospital, count}]` 簡化版，供下拉選單用 |
| `hospital_monthly` | `[{hospital, ym, count}]` |
| `hospital_procedure_ytd` | `[{hospital, procedure, count}]` |
| `hospital_physician_ytd` | `[{hospital, physician, count}]` |
| `hospital_physician_monthly` | `[{hospital, physician, ym, count}]` |
| `hospital_physician_procedure_ytd` | `[{hospital, physician, procedure, count}]` |
| `mtd_current / ly` | `[{hospital, count}]` |
| `ytd_current / ly` | `[{hospital, count}]` |
| `phys_mtd_current / ly` | `[{hospital, physician, count}]` |
| `phys_ytd_current / ly` | `[{hospital, physician, count}]` |
| `phys_all_ytd_ranking` | `[{physician, count}]` 所有醫院合計，YTD 降冪 |
| `phys_monthly_all` | `[{physician, ym, count}]` |
| `phys_procedure_ytd_all` | `[{physician, procedure, count}]` |
| `phys_mtd_all / ly_all` | `[{physician, count}]` |
| `phys_ytd_all / ly_all` | `[{physician, count}]` |
| `phys_hospitals` | `[{physician, hospital, count}]` YTD，用於判斷跨院醫師 |
| `physician_primary_hospital` | `[{physician, hospital}]` 每位醫師的主要醫院（YTD 優先，fallback 全時間） |
| `physician_ranking_detail` | `[{physician, count, count_ly}]` Top 10 YTD |

### Step 3 — render_html(agg)

將 `HTML_TEMPLATE` 中的三個 placeholder 替換：

| Placeholder | 替換內容 |
|-------------|---------|
| `{{FAVICON_B64}}` | Favicon 的 base64 字串 |
| `{{HLOGO_B64}}` | Biosense Webster logo 的 base64 字串 |
| `{{JSON_DATA}}` | `json.dumps(agg)` 的結果 |

### Step 4 — 寫出檔案

- 寫出 `index.html`（根目錄）
- 寫出 `dest/EP_Dashboard_<today>.html`

### Step 5 — Git push（`--push` 時）

```
git add index.html dest/EP_Dashboard_<today>.html
git diff --cached --quiet  # 若無變更則跳過 commit
git commit -m "dashboard update <today> (data through <last_date>)"
git push
```

若 staged 內容與上次相同（`git diff --cached` 無輸出），跳過 commit 和 push，避免錯誤。

---

## 腳本結構

```
build_dashboard.py
├── FAVICON_B64           # Favicon base64（PNG）
├── HLOGO_B64             # Biosense Webster logo base64（PNG）
├── HTML_TEMPLATE         # 完整 HTML，含 {{FAVICON_B64}} {{HLOGO_B64}} {{JSON_DATA}} placeholder
│
├── load_data(xlsx_path) -> pd.DataFrame
├── compute_aggregates(df) -> dict
├── render_html(agg) -> str
└── main()
```

單一檔案設計：不依賴外部 template.html 或 data.json，腳本本身即可獨立發送給同事使用。

---

## 已知注意事項

- Salesforce 有時在 territory name 後附加空白，已用 `.str.strip()` 處理
- `WST LOS ANGL VA MED  CENTER`（West LA VA）名稱含兩個空格，是 Salesforce 原始格式，請勿修改
- `HTML_TEMPLATE` 使用 Python raw string（`r"""`），其中的反斜線為 JS 程式碼的一部分，不需跳脫
- Push 前需完成 GitHub 認證，執行一次 `gh auth setup-git` 即可永久設定
