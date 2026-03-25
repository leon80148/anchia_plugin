# Routing Table

NHRI V4 插件的任務導向路由表。**先讀這個檔案**，再根據任務類型只載入需要的 reference。

## 使用方式

1. 判斷你的任務屬於下方哪一列。
2. 只載入該列指定的 reference 檔案。
3. 優先跑 script 而非讀更多文件。

詳細的 context budget 策略見 `references/context-budget.md`。

---

## Spec and Coefficients

- **Intent**: formula audit, coefficient checks, age baseline values, risk type rules
- **Open**: `references/formulas.md`
- **Script**: `python scripts/run_module.py --module spec-popavg -- --model chd --gender male --age 52`
- **何時需要**：需要看具體的 β 係數或 S0 值時

## Input Validation

- **Intent**: request pre-checks, required fields by branch, form/API validation
- **Open**: `references/validation-matrix.md`（速查）或 `references/validation-rules.md`（詳細）
- **Script**: `python scripts/run_module.py --module validate -- --model chd --json "{...}" --pretty`
- **何時需要**：建 API 端點的輸入驗證、debug 為什麼某筆資料回傳 status=1

## Evaluation

- **Intent**: compute risk for one model or all models
- **Open**: `references/evaluator-algorithm.md`
- **Script**: `python scripts/run_module.py --module evaluate -- --json "{...}" --pretty`
- **何時需要**：計算風險值、理解演算法流程

## Result Interpretation

- **Intent**: understand what risk values mean, clinical implications
- **Open**: `references/interpretation-rules.md`
- **何時需要**：解讀報告結果、產出患者溝通建議

## Regression Testing

- **Intent**: generate golden vectors or compare cross-language outputs
- **Open**: `references/regression-workflow.md`
- **Script generate**: `python scripts/run_module.py --module regress-generate -- --per-group 10 --output <vectors.json>`
- **Script compare**: `python scripts/run_module.py --module regress-compare -- --expected <vectors.json> --actual <actual.json>`
- **何時需要**：移植完成後的驗證、CI 整合

## Porting to Another Language

- **Intent**: move NHRI V4 logic to Python/JS/C#/Go with fidelity
- **Open**: `references/porting-checklist.md`
- **Also useful**: `references/regression-workflow.md`（移植後驗證）
- **何時需要**：將評估邏輯從 Java 移植到其他語言

## API Adapter

- **Intent**: scaffold FastAPI/Express/Spring wrappers
- **Open**: `references/endpoint-contract.md`
- **Also useful**: `references/platform-recipes.md`（各平台範例）、`references/security-observability.md`（安全指引）
- **Script**: `python scripts/run_module.py --module api-scaffold -- --platform fastapi --output adapters/fastapi`
- **何時需要**：建 REST API 包裝

## Report Rendering

- **Intent**: convert scoring outputs to markdown/text/json summary
- **Open**: `references/report-structure.md`
- **Script**: `python scripts/run_module.py --module report -- --input <result.json> --format markdown`
- **何時需要**：產出風險報告

## Visualization

- **Intent**: risk charts, color standards, dashboard embedding, PDF generation, accessibility
- **Open**: `references/visualization-guide.md`
- **Script**: `python scripts/run_module.py --module report -- --format html`
- **何時需要**：建圖表、嵌入 dashboard

## Web API Version (國健署 API 版)

- **Intent**: call NHRI remote API instead of local computation, API account setup, version selection
- **Open**: `references/api-version-guide.md`
- **何時需要**：使用者想用國健署遠端 API 而非本地元件版計算、需要了解 API 帳號申請流程、比較兩版差異
- **注意**：預設建議使用單機元件版，API 版需另外申請帳號

---

## 任務組合指引

有些任務會跨多個領域，以下是常見的組合：

| 複合任務 | 載入順序 |
|----------|---------|
| 建完整 API 並部署 | endpoint-contract → platform-recipes → security-observability |
| 移植並驗證 | porting-checklist → regression-workflow |
| 產出完整報告 | evaluator-algorithm → report-structure → visualization-guide |
| Debug 結果不正確 | validation-rules → evaluator-algorithm → formulas |
| 串接國健署遠端 API | api-version-guide（含端點、認證、範例） |
| 選擇單機版 or API 版 | api-version-guide（版本比較表） |
