# Varipulse Adoption — Hospital detail

## 目的
追蹤 Varipulse（BWI 新一代 PFA catheter）在各醫院 AF 手術中的使用量與滲透率。
取代假設：原本 AF 都用 Farapulse；Varipulse 進來之後想看成長與佔比。

## 範圍
- **這個版本只動 Hospital detail。** Region summary 和 Physician detail 暫不動。
- 電腦版與手機版都要有，內容、位置、格式相同。

## 資料定義

| 名稱 | 定義 |
|---|---|
| AF case | `Primary Procedure: Work Type Name == "A FIB"` |
| Varipulse case | AF case 且 `BWI Ablation Catheter` 欄位含字串 `VARIPULSE`（不分大小寫；同欄多 catheter 例如 `STSF; VARIPULSE` 也算 1 台） |
| Farapulse case | AF total − Varipulse（直接相減，不查 `Competitive Products` 欄位） |
| Adoption rate | Varipulse / AF total（百分比，1 位小數） |
| MTD | 當前年份、當前月份、日期 ≤ last_day（沿用 dashboard 既有 MTD 邏輯） |
| YTD | 當前年份、日期 ≤ last_date（沿用 dashboard 既有 YTD 邏輯） |
| 不在 A FIB 但 BWI 欄含 Varipulse 的 case | **不計入**（目前共 7 筆：SVT / A Flutter / Concomitant / EP Study / Right AT） |

分母為 0 時（該醫院當期沒有 AF case）：顯示 `0 / 0 • —`，bar 留空。

## 視覺設計

**位置**：Hospital detail 頁，Top Physicians 區塊**下方**、Recent Cases 區塊**上方**。電腦版與手機版同位置。

**內容**：兩條水平堆疊條 + 數字標註。

```
Varipulse Adoption
─────────────────────────────────────
MTD (May 1–9)
█████████░░░░░░░░░░░  5 / 18  •  27.8%

YTD 2026
███░░░░░░░░░░░░░░░░░  23 / 142 •  16.2%
─────────────────────────────────────
```

- 條的寬度 = adoption rate（Varipulse / AF）
- 條的填滿色 = Varipulse（品牌色），底色 = Farapulse（灰）
- 右側標註：`Varipulse / AF total • 百分比`
- 不加 vs LY 同期比較
- 標題 `Varipulse Adoption`
- MTD 標籤帶月份範圍（如 `MTD (May 1–9)`），YTD 標籤帶年份（如 `YTD 2026`）

## 實作位置

- **資料**：[build_dashboard.py](../build_dashboard.py) 的 `compute_aggregates`，每家醫院新增 4 個欄位 — `af_mtd`, `varipulse_mtd`, `af_ytd`, `varipulse_ytd`（或包成 `varipulse_adoption: {mtd: {af, varipulse}, ytd: {af, varipulse}}`）
- **前端**：[dashboard.html.tmpl](../dashboard.html.tmpl) 的 Hospital detail section，在 Top Physicians 和 Recent Cases 之間插入新區塊
- **不用** 動 region 跟 physician 頁面
