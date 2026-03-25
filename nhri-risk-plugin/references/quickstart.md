# Quick Start

NHRI V4 風險評估工具的快速上手指南。

## 前置條件

```bash
# 確認 Python 版本 >= 3.8
python --version

# 確認 scripts 目錄存在
ls scripts/run_module.py
```

## 1. 評估所有模型（最常用）

```bash
python scripts/run_module.py --module evaluate -- \
  --json '{"gender":0,"age":58,"sbp":132,"tg":145,"chol":210,"hdlc":52,"glu":101,"waist":82,"ldlc":129,"height":160,"weight":63,"smoke":0,"hbp":1,"diabetes":0}' \
  --pretty
```

預期輸出（節錄）：
```json
{
  "chd": { "status": 0, "risk": 5.23, "riskType": 0, "band": "lower" },
  "stroke": { "status": 0, "risk": 8.41, "riskType": 0, "band": "lower" },
  "hypertension": { "status": 0, "risk": 62.15, "riskType": 1, "band": "elevated" },
  "diabetes": { "status": 0, "risk": 12.87, "riskType": 1, "band": "elevated" },
  "mace": { "status": 0, "risk": 9.76, "riskType": 0, "band": "lower" }
}
```

## 2. 驗證單一模型的輸入

```bash
python scripts/run_module.py --module validate -- \
  --model chd \
  --json '{"gender":1,"age":52,"hbp":1,"waist":88,"hdlc":45}' \
  --pretty
```

驗證通過時輸出 `status: 0`，失敗時 `status: 1` 並列出缺少的欄位。

## 3. 產生回歸測試向量

```bash
python scripts/run_module.py --module regress-generate -- \
  --per-group 3 \
  --output assets/golden-vectors.sample.json
```

每個 model × gender 組合產生 3 筆測試資料，共 30 筆向量。

## 4. 建立 API adapter 骨架

```bash
python scripts/run_module.py --module api-scaffold -- \
  --platform fastapi \
  --output adapters/fastapi \
  --service-name nhri-risk-api
```

支援的 platform：`fastapi`、`express`、`spring`。

## 5. 產出 Markdown 報告

```bash
python scripts/run_module.py --module report -- \
  --input assets/sample-evaluator-output.json \
  --format markdown
```

支援的 format：`markdown`、`json`、`html`。

## 常見問題

| 症狀 | 原因 | 解法 |
|------|------|------|
| `ModuleNotFoundError` | 缺少依賴 | `pip install -r requirements.txt` |
| `status: 1` 但資料看起來對 | 欄位值為 0 或 null | 確認所有必填欄位 `> 0`（見 validation-matrix.md） |
| 輸出的 risk 和 Java 版本差一點 | 浮點數精度差異 | 確認使用 Java-compatible rounding（見 porting-checklist.md） |
| `--model` 參數無效 | model 名稱大小寫錯誤 | 必須小寫：`chd`、`stroke`、`hypertension`、`diabetes`、`mace` |
