# Platform Recipes

將 NHRI V4 評估邏輯整合到不同 Web 框架的具體作法。

## FastAPI (Python)

直接在 process 內呼叫 evaluator，零序列化開銷。

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from nhri_risk_eval import evaluate_all

app = FastAPI()

class RiskInput(BaseModel):
    gender: int = Field(..., ge=0, le=1)
    age: int = Field(..., gt=0, le=120)
    sbp: float = 0; tg: float = 0; chol: float = 0
    hdlc: float = 0; glu: float = 0; waist: float = 0
    ldlc: float = 0; height: float = 0; weight: float = 0
    smoke: int = 0; hbp: int = 0; diabetes: int = 0

@app.post("/api/v4/evaluate")
def evaluate(body: RiskInput):
    result = evaluate_all(body.dict())
    return result
```

重點：
- Pydantic 處理型別驗證，evaluator 處理業務驗證（必填欄位 `> 0`）。
- 不要重新命名 evaluator 的輸出欄位。

## Express (Node.js)

adapter 層盡量薄，核心邏輯放 evaluator module。

```javascript
const express = require('express');
const { evaluateAll } = require('./nhri-risk-eval');

const app = express();
app.use(express.json());

app.post('/api/v4/evaluate', (req, res) => {
  const { model, ...input } = req.body;
  if (!input.age || !input.gender === undefined) {
    return res.status(400).json({ error: 'age and gender are required' });
  }
  const result = model
    ? evaluateAll(input, [model])
    : evaluateAll(input);
  res.json(result);
});
```

重點：
- `gender` 可為 0，用 `=== undefined` 檢查，不要用 truthy check。
- 所有 5 個 model 回傳相同 JSON 結構。

## Spring Boot (Java)

Controller 最小化，model dispatch 放 Service bean。

```java
@RestController
@RequestMapping("/api/v4")
public class RiskController {
    @Autowired private RiskEvaluatorService evaluator;

    @PostMapping("/evaluate")
    public ResponseEntity<Map<String, ModelResult>> evaluate(
            @Valid @RequestBody RiskInput input) {
        return ResponseEntity.ok(evaluator.evaluateAll(input));
    }
}
```

重點：
- DTO 欄位名稱必須與 canonical contract 一致。
- Java 的 `Math.round()` 行為是 NHRI V4 的基準，其他語言要對齊這個行為。

## Cross-Platform Rules

| 規則 | 說明 |
|------|------|
| Model 名稱小寫 | `chd`、`stroke`、`hypertension`、`diabetes`、`mace` |
| 輸出欄位不改名 | `risk`、`populationAvg`、`multipleDiff`、`riskType`、`band` |
| 精度對齊 Java | 使用 Java-compatible 2-decimal rounding（見 porting-checklist.md） |
| status code 一致 | 200 成功、400 輸入驗證失敗、500 非預期錯誤 |
| Content-Type | 一律 `application/json; charset=utf-8` |

## 選擇建議

| 情境 | 推薦平台 | 原因 |
|------|---------|------|
| 快速原型 / 內部工具 | FastAPI | 直接 import evaluator，零額外開發 |
| 已有 Node.js 後端 | Express | 最少的架構變動 |
| 醫院 HIS 整合（Java 生態） | Spring Boot | 與既有 Java 基礎設施相容 |
