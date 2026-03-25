---
name: nhri-risk-plugin
description: |
  國衛院 NHRI V4 健康風險評估系統（支援單機元件版 + 國健署 Web API 版）。當涉及以下情境時啟用：
  (1) 計算 CHD（冠心病）、中風、高血壓、糖尿病、MACE 五大疾病風險分數
  (2) 查核 NHRI V4 風險公式係數或計算邏輯
  (3) 驗證風險模型輸入資料的格式和範圍
  (4) 進行跨平台回歸測試，確保不同語言實作的計算結果一致
  (5) 產生 FastAPI/Express/Spring/Go/Rust/.NET API 適配器
  (6) 渲染風險評估報告
  (7) 串接國健署慢性疾病風險評估平台 Web API（需申請帳號）
  (8) 選擇單機元件版或 API 版的決策建議

  觸發詞：NHRI、國衛院風險評估、冠心病風險、中風風險、高血壓風險、糖尿病風險、
  MACE風險、心血管風險、健康風險計算、CVD風險分數、Framingham、
  風險公式係數、回歸測試向量、API適配器、風險報告渲染、
  慢性疾病風險API、國健署API、cdrc API、Web API風險評估、科學算病館
---

# NHRI Risk Plugin

## Overview

Use this skill for all NHRI V4 risk assessment workflows: formula lookup, input validation, risk evaluation, regression testing, API adapter scaffolding, and report rendering. Supports both **standalone component (單機元件版)** and **Web API version (API 版)**.

## When to Use

- Auditing NHRI V4 coefficients or formula logic
- Validating risk model input payloads
- Computing CHD / Stroke / Hypertension / Diabetes / MACE risk scores
- Generating golden regression vectors for cross-platform parity
- Scaffolding FastAPI / Express / Spring API wrappers
- Rendering risk evaluation results into reports
- Calling the NHRI Web API (國健署慢性疾病風險評估平台 API)
- Choosing between standalone vs. API version

## 版本選擇：單機元件版 vs. API 版

本插件支援兩種計算方式，**預設使用單機元件版**：

| | 單機元件版（預設） | API 版 |
|---|---|---|
| 計算方式 | 本地 Python 腳本 | 遠端國健署伺服器 |
| 適合場景 | 大量/批次計算、離線、不可中斷 | 已有 HIS 自建系統、即時串接 |
| 帳號需求 | 不需要 | 需向國健署申請（見下方） |
| 額外回傳 | 需自行實作 | 自動回傳 guideline + riskType |

### 如何選擇？

- **沒有特別需求** → 使用單機元件版（本插件的 scripts/ 和 references/）
- **院所已有自建系統，想即時串接** → 使用 API 版，參見 `references/api-version-guide.md`
- **大量計算、批次處理** → 使用單機元件版（官方建議）

### API 版帳號申請

如需使用 API 版，須先申請帳號：
1. 至國健署慢性疾病風險評估平台下載申請書
2. 填寫後提交給執行單位（社團法人臺灣社會改造協會）
3. 經社改會初審 → 國健署複核 → 帳號啟用
4. 常見問題：`https://cdrc.hpa.gov.tw/qa.html`
5. 開發測試帳號請洽國健署或承辦廠商窗口

## Context Control

1. Read `references/routing-table.md` first to identify the relevant reference.
2. Load only one domain reference per task.
3. Prefer running scripts over loading large docs.
4. Open additional references only if output is insufficient.
5. See `references/context-budget.md` for the full loading policy.

## Routing by Intent

| Intent | Reference | Script |
|---|---|---|
| Formula audit, coefficients | `references/formulas.md` | `run_module.py --module spec-popavg` |
| Input validation | `references/validation-matrix.md` | `run_module.py --module validate` |
| Risk evaluation (standalone) | `references/evaluator-algorithm.md` | `run_module.py --module evaluate` |
| Risk evaluation (API version) | `references/api-version-guide.md` | 直接呼叫國健署 API |
| Regression testing | `references/regression-workflow.md` | `run_module.py --module regress-generate` |
| API adapter scaffold | `references/endpoint-contract.md` | `run_module.py --module api-scaffold` |
| Report rendering | `references/report-structure.md` | `run_module.py --module report` |
| Visualization | `references/visualization-guide.md` | `run_module.py --module report --format html` |
| Version selection / API account | `references/api-version-guide.md` | — |

## Model Rules (V4)

- Models: `chd`, `stroke`, `hypertension`, `diabetes`, `mace`
- Gender routing: `gender == 1` → male branch; else → female branch
- Rounding: Java-style `round(value * 10000) / 100` for risk percentages
- Age range for population averages: 35-70 (index = `age - 35`); outside range → 0
- Risk type thresholds:
  - CHD/Stroke/Diabetes/MACE: `risk >= 20` → type 2, `risk >= 10` → type 1
  - Hypertension: `multipleDiff > 1.25` → type 2, `multipleDiff >= 0.75` → type 1

## 風險分數臨床意義速查

計算出的風險百分比代表「未來 10 年內發生該事件的機率」。以下是臨床上怎麼解讀這些數字：

### 風險分層與對應行動

| 風險等級 | CHD/Stroke/DM/MACE | 高血壓 (multipleDiff) | 臨床意義 | 建議行動 |
|---------|--------------------|-----------------------|---------|---------|
| **低風險** | < 10% | < 0.75 | 10 年內發生機率低 | 生活型態建議、定期追蹤 |
| **中風險** | 10-19% | 0.75 - 1.25 | 值得關注但不需急迫介入 | 加強生活型態調整、3-6 個月追蹤 |
| **高風險** | ≥ 20% | > 1.25 | 10 年內有顯著發生機率 | 評估是否需要藥物介入、密切追蹤 |

### 怎麼跟病患溝通

> 「您的 10 年冠心病風險是 15%，意思是像您這樣的 100 個人中，大約有 15 個在未來 10 年會發生冠心病事件。這不是說您一定會，但比一般人高，所以我們建議加強控制血壓和血脂。」

### 重要提醒（必須告知使用者）

- **這是群體統計，不是個人預測**：同樣 15% 的人，有人會發生、有人不會
- **不含藥物介入效果**：如果病患已在服用降血壓/降血脂藥物，實際風險可能更低
- **年齡是最大因子**：年輕人風險低不代表可以忽略其他危險因子
- **V4 模型適用範圍**：35-70 歲台灣成年人群體

---

## Additional Platform Support

Beyond FastAPI, Express, and Spring, the scaffold generator also supports Go, Rust, and .NET platforms.

### Go (Gin)

Template: `assets/gin_risk_handler.go.tpl`

```bash
python scripts/run_module.py --module api-scaffold -- --platform gin --output adapters/gin
```

### Rust (Axum)

Template: `assets/axum_risk_handler.rs.tpl`

```bash
python scripts/run_module.py --module api-scaffold -- --platform axum --output adapters/axum
```

### .NET Minimal API

```bash
python scripts/run_module.py --module api-scaffold -- --platform dotnet --output adapters/dotnet
```

## CLI Entrypoint

All scripts are invoked via `scripts/run_module.py`:

```bash
# Evaluate all models
python scripts/run_module.py --module evaluate -- --json "{...}" --pretty

# Validate one model
python scripts/run_module.py --module validate -- --model chd --json "{...}" --pretty

# Population average lookup
python scripts/run_module.py --module spec-popavg -- --model chd --gender male --age 52

# Generate regression vectors
python scripts/run_module.py --module regress-generate -- --per-group 10 --output assets/golden-vectors.sample.json

# Compare regression outputs
python scripts/run_module.py --module regress-compare -- --expected <vectors.json> --actual <actual.json>

# Scaffold API adapter
python scripts/run_module.py --module api-scaffold -- --platform fastapi --output adapters/fastapi

# Render report
python scripts/run_module.py --module report -- --input <result.json> --format markdown
```

## References

| File | Description |
|---|---|
| `references/routing-table.md` | Intent-to-reference routing map |
| `references/context-budget.md` | Low-token loading policy |
| `references/quickstart.md` | Common command patterns |
| `references/formulas.md` | Full equations and coefficients |
| `references/input-output-contract.json` | Canonical I/O contract |
| `references/population-averages.json` | Age-indexed baseline arrays (35-70) |
| `references/validation-rules.md` | Input validation matrix by model |
| `references/validation-matrix.md` | Required fields by model and branch |
| `references/error-codes.json` | Stable validation error codes |
| `references/evaluator-algorithm.md` | Step-by-step evaluation algorithm |
| `references/porting-checklist.md` | Cross-language parity checklist |
| `references/regression-workflow.md` | Recommended regression CI flow |
| `references/actual-output-format.md` | Required regression output shapes |
| `references/endpoint-contract.md` | Canonical API request/response |
| `references/platform-recipes.md` | Per-stack integration notes |
| `references/security-observability.md` | Production API guardrails |
| `references/interpretation-rules.md` | Risk band mapping and narrative hints |
| `references/report-structure.md` | Canonical report fields |
| `references/visualization-guide.md` | Risk visualization charts, color standards, embedding |
| `references/api-version-guide.md` | 國健署 Web API 版完整使用指南（端點、參數、認證、範例） |

## Scripts

| Script | Description |
|---|---|
| `scripts/run_module.py` | Single-entry dispatcher for all operations |
| `scripts/nhri_risk_eval.py` | Reference evaluator (Python) |
| `scripts/validate_input.py` | Input validation CLI |
| `scripts/get_population_avg.py` | Population average lookup |
| `scripts/generate_vectors.py` | Regression vector generator |
| `scripts/compare_outputs.py` | Regression output comparator |
| `scripts/scaffold_adapter.py` | API adapter scaffold generator (supports fastapi, express, spring, gin, axum, dotnet) |
| `scripts/render_report.py` | Report renderer |

## 設定與環境錯誤

**初次使用時常見的環境問題：**

| 錯誤訊息 | 原因 | 解決方法 |
|---------|------|---------|
| `ModuleNotFoundError: No module named 'numpy'` | Python 依賴未安裝 | `pip install numpy scipy` |
| `FileNotFoundError: population-averages.json` | 在錯誤的目錄執行腳本 | 必須在 `nhri-risk-plugin/` 根目錄執行 |
| `python: command not found` | Python 3 未安裝或 PATH 未設定 | 使用 `python3` 或確認 PATH 含 Python 3 |
| `JSONDecodeError` on input file | 輸入 JSON 格式錯誤 | 用 `python -m json.tool input.json` 驗證格式 |
| `KeyError: 'age'` | 輸入缺少必填欄位 | 執行 `validate_input.py` 先驗證輸入 |
| 計算結果為 `null` 或 `NaN` | 輸入值為 null 或超出範圍 | 確認全部欄位都有有效數值 |
| `Permission denied: scripts/run_module.py` | 腳本沒有執行權限（Linux/Mac） | `chmod +x scripts/*.py` |

## 邊界與失敗模式

**使用前要知道的限制：**

| 情況 | 為什麼失敗 | 如何確認 |
|------|-----------|---------|
| 輸入超出年齡範圍（< 35 或 > 70） | 模型設計範圍是 35-70，超出時 population average 回傳 0 | 執行前先驗證 age 欄位 |
| 性別欄位傳入 0 或其他非 1/2 的值 | 預設走 female branch，可能導致錯誤分層 | validate module 會報錯 |
| HbA1c 或其他非直接輸入欄位 | 這些不是 NHRI V4 的輸入欄位（模型只用列出的參數） | 確認 input-output-contract.json |
| 跨平台計算結果不一致 | 浮點數捨入差異：Java round vs Python round | 使用 Java-style `round(value * 10000) / 100` |
| 把風險 % 直接當臨床診斷 | 這是統計模型，不是個人診斷工具 | 輸出加上「依群體統計計算，個人差異需臨床判斷」聲明 |

**這個技能的硬邊界：**
- V4 係數是固定的，不接受自定義係數
- 只支援 5 個模型（CHD、中風、高血壓、糖尿病、MACE）
- 不包含藥物介入後的風險重計算

## 完整評估範例（從輸入到報告）

**場景**：55 歲男性，門診做完成人健檢，想看心血管風險。

**Step 1 — 準備輸入**：

```json
{
  "gender": 1,
  "age": 55,
  "sbp": 145,
  "dbp": 88,
  "chol": 220,
  "hdlc": 38,
  "ldlc": 150,
  "tg": 180,
  "bmi": 27.5,
  "smoke": 1,
  "diabetes": 0,
  "hbp": 1,
  "waist": 92,
  "glu": 105
}
```

**Step 2 — 執行**：

```bash
python scripts/run_module.py --module evaluate -- --json '{"gender":1,"age":55,...}' --pretty
```

**Step 3 — 結果解讀**：

| 模型 | 風險值 | 分層 | 臨床意義 |
|------|--------|------|---------|
| CHD | 18.5% | type 1 (中高風險) | 10 年內約 5 分之 1 機率 |
| Stroke | 12.3% | type 1 (中風險) | 需關注，但非最緊急 |
| Hypertension | multipleDiff 1.8 | type 2 (高風險) | 已在服藥但控制不佳 |
| Diabetes | 8.2% | 低風險 | 但 FPG 105 已接近前期 |
| MACE | 22.1% | type 2 (高風險) | 綜合心血管事件風險高 |

**Step 4 — 給病患的建議摘要**：

> 您的整體心血管風險偏高（MACE 22%）。主要因為：吸菸 + 血壓控制不佳 + HDL 偏低。
> 建議優先：(1) 戒菸（最大單一可改善因子） (2) 與醫師討論調整降壓藥 (3) 增加運動提升 HDL。

---

## 成功的樣子

**跨平台驗證（最重要的成功標準）：**

1. 用相同輸入執行 Python 參考實作和你的目標平台（Java/Go/Rust 等）
2. CHD 風險值差距應 < 0.01%（浮點誤差範圍內）
3. 使用 `regress-compare` 確認 golden vectors 一致

```bash
# 快速驗證
python scripts/run_module.py --module regress-compare \
  --expected assets/golden-vectors.sample.json \
  --actual your_platform_output.json
```

通過 regression 比對 = 這個平台的移植是正確的。

---

## Assets

| File | Description |
|---|---|
| `assets/golden-vectors.sample.json` | Sample regression vectors |
| `assets/express_app.js.tpl` | Express adapter template |
| `assets/fastapi_app.py.tpl` | FastAPI adapter template |
| `assets/spring_risk_controller.java.tpl` | Spring adapter template |
| `assets/report-template.md` | Markdown report layout template |
| `assets/sample-evaluator-output.json` | Sample evaluator output for testing |
| `assets/sample-report.md` | Sample rendered report |
| `assets/gin_risk_handler.go.tpl` | Go Gin adapter template |
| `assets/axum_risk_handler.rs.tpl` | Rust Axum adapter template |

---

## CI/CD Pipeline（GitHub Actions）

### 基本 Workflow

```yaml
name: NHRI Risk CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt

      # 輸入驗證測試
      - name: Run validation tests
        run: python scripts/run_module.py --module validate -- --test-suite all

      # 回歸測試
      - name: Generate regression vectors
        run: python scripts/run_module.py --module regress-generate -- --per-group 5 --output /tmp/vectors.json

      - name: Run regression comparison
        run: python scripts/run_module.py --module regress-compare -- --expected assets/golden-vectors.sample.json --actual /tmp/vectors.json

      # 所有模型全量評估
      - name: Evaluate all models (smoke test)
        run: |
          python scripts/run_module.py --module evaluate -- \
            --json '{"gender":1,"age":50,"sbp":130,"dbp":85,"chol":200,"hdlc":50,"ldlc":130,"tg":150,"bmi":25,"smoke":0,"diabetes":0,"hbp":0,"waist":85,"glu":100}' \
            --pretty

  scaffold:
    runs-on: ubuntu-latest
    needs: validate
    strategy:
      matrix:
        platform: [fastapi, express, spring, gin, axum, dotnet]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - name: Scaffold ${{ matrix.platform }} adapter
        run: python scripts/run_module.py --module api-scaffold -- --platform ${{ matrix.platform }} --output /tmp/${{ matrix.platform }}
      - name: Verify scaffold output exists
        run: ls -la /tmp/${{ matrix.platform }}/
```

### 建議的 CI 觸發條件

| 事件 | 執行內容 |
|------|---------|
| push to main | 全量回歸測試 + scaffold 驗證 |
| pull request | 驗證測試 + 回歸比對 |
| release tag | 全量測試 + 產生所有平台 adapter |
| schedule (weekly) | 完整回歸套件 + 效能基準 |

---

## Web UI Scaffolding

### 目錄結構

```
nhri-risk-web/
├── src/
│   ├── components/
│   │   ├── RiskForm.tsx          # 輸入表單
│   │   ├── RiskResult.tsx        # 結果顯示
│   │   ├── RiskChart.tsx         # 風險視覺化圖表
│   │   └── ModelSelector.tsx     # 模型選擇器
│   ├── hooks/
│   │   └── useRiskEvaluation.ts  # API 呼叫 hook
│   ├── utils/
│   │   ├── validation.ts         # 前端輸入驗證
│   │   └── formatters.ts         # 結果格式化
│   ├── types/
│   │   └── risk.ts               # TypeScript 型別定義
│   └── App.tsx
├── public/
├── package.json
└── tsconfig.json
```

### 前端輸入驗證（與後端一致）

| 欄位 | 範圍 | 型別 |
|------|------|------|
| age | 35-70 | integer |
| gender | 1 (男) / 2 (女) | integer |
| sbp | 80-250 | number |
| dbp | 40-150 | number |
| tc | 100-400 | number |
| hdl | 20-120 | number |
| bmi | 15-50 | number |
| waist | 50-150 | number |
| fpg | 50-300 | number |
| smoke | 0 / 1 | boolean → integer |
| dm | 0 / 1 | boolean → integer |
| hpt_med | 0 / 1 | boolean → integer |

---

## 模型版本管理

### 版本命名規則

```
NHRI-V{major}.{minor}.{patch}
例：NHRI-V4.1.0
```

| 欄位 | 變更時機 |
|------|---------|
| major | 模型架構改變（如 V3 → V4） |
| minor | 係數更新、新增模型 |
| patch | Bug 修復、文件更新 |

### Golden Vector 管理

每次模型更新時：
1. 使用舊版本和新版本分別生成 golden vectors
2. 比較差異，確認變更僅在預期範圍內
3. 更新 `assets/golden-vectors.sample.json`
4. 在 commit message 中註明係數變更細節
5. 為舊版本建立 git tag 以便回溯

### 多版本並存（建議結構）

未來版本更新時，建議的檔案組織方式：

```
references/
├── formulas.md              # 目前版本（V4，已存在）
├── formulas-v3.md           # 歷史版本（V3，版本更新時建立）
└── migration-v3-to-v4.md    # 版本遷移指南（版本更新時建立）
```
