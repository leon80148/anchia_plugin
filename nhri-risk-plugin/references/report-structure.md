# Report Structure

NHRI V4 風險評估報告的資料結構定義和渲染指引。

## Single Output Record

```json
{
  "model": "chd",
  "status": 0,
  "risk": 20.02,
  "populationAvg": 10.30,
  "multipleDiff": 194.37,
  "riskType": 2,
  "version": 4,
  "band": "high",
  "summary": "High risk level. Recommend clinician review and follow-up planning."
}
```

### 欄位說明

| 欄位 | 型別 | 說明 |
|------|------|------|
| `model` | string | 模型名稱：chd, stroke, hypertension, diabetes, mace |
| `status` | int | 0=成功, 1=驗證失敗 |
| `risk` | float | 10 年風險百分比（2 位小數） |
| `populationAvg` | float | 同年齡族群平均風險百分比（age 35-70 才有值） |
| `multipleDiff` | float | 個人風險 / 族群平均（倍率，age 35-70 才有值） |
| `riskType` | int | 0=lower, 1=elevated, 2=high |
| `version` | int | 評估版本（目前 = 4） |
| `band` | string | riskType 的文字對照：lower / elevated / high |
| `summary` | string | 一句話建議，依 band 自動生成 |

### status=1 時的結構

```json
{
  "model": "chd",
  "status": 1,
  "risk": 0,
  "populationAvg": 0,
  "multipleDiff": 0,
  "riskType": 0,
  "version": 0,
  "band": "",
  "summary": "Validation failed: missing required fields."
}
```

`version: 0` 表示未完成評估。不要在報告中呈現 status=1 的數值。

## Multi-Model Report

```json
{
  "title": "NHRI Risk Summary",
  "generatedAt": "2026-02-16T08:00:00Z",
  "models": [ ...single records... ],
  "overall": {
    "highestBand": "high",
    "highCount": 2,
    "elevatedCount": 1,
    "lowerCount": 2
  }
}
```

`overall` 只統計 `status == 0` 的模型。status=1 的模型不計入。

## Model Names

| Key | 中文名稱 | 完整英文 |
|-----|---------|---------|
| `chd` | 冠心病 | Coronary Heart Disease |
| `stroke` | 腦中風 | Stroke |
| `hypertension` | 高血壓 | Hypertension |
| `diabetes` | 糖尿病 | Diabetes |
| `mace` | 重大心血管事件 | Major Adverse Cardiovascular Events |

## Band 顏色標準

報告渲染時使用一致的顏色：

| Band | Hex Color | 中文標示 | 圖示 |
|------|-----------|---------|------|
| high | `#E53E3E` | 高風險 | 紅色填滿 |
| elevated | `#D69E2E` | 中等風險 | 黃色填滿 |
| lower | `#38A169` | 低風險 | 綠色填滿 |

## 輸出格式

`run_module.py --module report` 支援三種格式：

| Format | 用途 | 特點 |
|--------|------|------|
| `json` | API 回傳、程式間傳遞 | 原始結構，最完整 |
| `markdown` | 文件嵌入、Email 報告 | 表格化呈現，含建議文字 |
| `html` | 網頁嵌入、PDF 轉換 | 含顏色和圖表，可直接展示 |

### Markdown 報告範例

```markdown
# NHRI 風險評估報告

生成時間：2026-02-16

| 項目 | 風險 | 族群平均 | 倍率 | 等級 |
|------|------|---------|------|------|
| 冠心病 | 20.02% | 10.30% | 1.94x | 🔴 高風險 |
| 腦中風 | 8.41% | 9.12% | 0.92x | 🟢 低風險 |
| 高血壓 | 62.15% | 55.30% | 1.12x | 🟡 中等風險 |
```
