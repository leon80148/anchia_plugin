# Context Budget

Use this policy to keep token load small when working with NHRI risk evaluation.

## Core Principle

每次只載入最少量的參考文件。NHRI 插件的參考文件總量超過 2000 行，一次全部載入會浪費 context window 且降低回答精確度。

## Loading Protocol

1. **先讀 routing-table.md**（~40 行）— 判斷任務類型。
2. **只開一個 domain reference** — routing-table 會指向正確的檔案。
3. **優先跑 script** — 確定性行為用 `run_module.py`，不要靠讀文件推理。
4. **不夠再開第二個** — 只在 script 輸出不足以回答時才載入額外參考。
5. **永遠不複製公式** — `references/formulas.md` 是唯一真相來源（single source of truth）。

## Suggested Max Loading Per Task

| 任務類型 | 必載文件 | 可選文件 | 預估 tokens |
|----------|---------|---------|------------|
| 驗證欄位需求 | routing-table + validation-matrix | validation-rules | ~200 |
| 計算風險值 | routing-table + evaluator-algorithm | formulas | ~250 |
| 解讀報告結果 | routing-table + interpretation-rules | report-structure | ~200 |
| 移植到新語言 | routing-table + porting-checklist | regression-workflow | ~200 |
| 建 API adapter | routing-table + endpoint-contract | platform-recipes | ~250 |
| 跑回歸測試 | routing-table + regression-workflow | — | ~180 |
| 產出視覺化報告 | routing-table + visualization-guide | report-structure | ~300 |

## Anti-Patterns（常見錯誤）

| 錯誤做法 | 為什麼不好 | 正確做法 |
|----------|-----------|---------|
| 一次載入所有 reference | 浪費 ~2000 tokens，降低回答品質 | 用 routing-table 精準載入 |
| 複製 formulas.md 裡的係數到回答中 | 容易抄錯，且佔用 context | 跑 `run_module.py --module spec-popavg` |
| 載入 formulas.md 只為查一個模型的門檻 | formulas.md 是最大的檔案（~300 行） | 先查 validation-rules.md（更簡潔） |
| 同時開 validation-matrix 和 validation-rules | 兩者內容重疊 80% | 選一個：matrix 看全貌，rules 看細節 |

## Script-First 原則

當任務是「計算」或「驗證」時，跑 script 永遠優先於讀文件：

```bash
# 驗證輸入 — 比讀 validation-matrix.md 更準確
python scripts/run_module.py --module validate -- --model chd --json '{"gender":1,"age":52,"hdlc":45,"waist":88}' --pretty

# 計算風險 — 比讀 formulas.md 手算更可靠
python scripts/run_module.py --module evaluate -- --json '{"gender":0,"age":58,"sbp":132,"tg":145}' --pretty
```

只在需要「解釋為什麼」時才載入文件（例如：為什麼這個欄位必填、公式的數學意義）。
