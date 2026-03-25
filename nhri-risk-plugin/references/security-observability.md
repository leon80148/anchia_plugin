# Security and Observability

NHRI 風險評估 API 的安全性、可靠性和可觀測性指引。

## Security

### PHI/PII 保護（最高優先級）

風險評估的輸入包含受保護的健康資訊（Protected Health Information），必須嚴格控管：

| 資料類別 | 欄位 | 可否記錄到 log |
|----------|------|---------------|
| 人口學 | gender, age | 可（已去識別化） |
| 生理量測 | sbp, height, weight, waist | 不建議（可間接識別） |
| 檢驗數值 | chol, hdlc, ldlc, tg, glu | 不可（臨床資料） |
| 病史 | smoke, hbp, diabetes | 不可（診斷資料） |

### 輸入驗證

```
1. Request size limit: 最大 10 KB（風險評估輸入不可能超過此大小）
2. Model allow-list: 只接受 ["chd","stroke","hypertension","diabetes","mace"]
3. 數值範圍檢查: age 1-120, sbp 50-300, 其他 >= 0
4. 型別檢查: 所有數值欄位必須為 number，gender 必須為 0 或 1
```

### 認證與授權

- 非公開端點必須要求認證（API Key 或 JWT）。
- 區分「查詢」和「批次評估」的權限層級。
- 內部服務間呼叫建議使用 mTLS。

## Reliability

### HTTP Status 對照

| Status | 情境 | Response body |
|--------|------|---------------|
| 200 | 評估成功（含 evaluator status=1 的驗證失敗） | 完整 evaluator output |
| 400 | request body 格式錯誤、缺少必要欄位 | `{"error": "描述"}` |
| 401 | 未認證 | `{"error": "unauthorized"}` |
| 429 | 超過速率限制 | `{"error": "rate limited", "retryAfter": 60}` |
| 500 | 非預期的 adapter/evaluator 錯誤 | `{"error": "internal error", "requestId": "..."}` |

注意：evaluator 本身的 `status: 1`（輸入不足以計算）仍回傳 HTTP 200，因為 API 層面的處理是成功的。

### 速率限制建議

- 單一 API Key：100 requests/min
- 批次端點：10 requests/min（每次最多 50 筆）
- 超限時回傳 429 + `Retry-After` header

## Observability

### 必要的 log 欄位

```json
{
  "requestId": "uuid-v4",
  "timestamp": "ISO-8601",
  "model": "chd",
  "status": 0,
  "riskType": 2,
  "latencyMs": 12,
  "evaluatorVersion": 4
}
```

不要記錄：`age`、`sbp`、`glu` 等任何臨床輸入值。

### Metrics 建議

| Metric | Type | Labels |
|--------|------|--------|
| `nhri_evaluate_total` | Counter | model, status |
| `nhri_evaluate_duration_ms` | Histogram | model |
| `nhri_validation_failures` | Counter | model, missing_field |

### Health Check

提供 `/health` 端點回傳 evaluator 版本和可用狀態：

```json
{ "status": "ok", "evaluatorVersion": 4, "models": ["chd","stroke","hypertension","diabetes","mace"] }
```

## Versioning

- Response 中永遠包含 `version` 欄位（目前 = 4）。
- URL 版本化：`/api/v4/evaluate`。
- 若未來 V5 上線，保留 V4 端點至少 6 個月。
