# 國健署慢性疾病風險評估平台 — Web API 版使用指南

> 文件來源：國健署「風險評估 Web API 使用說明 v1.2」（2025-11-01）
> 權威來源：本文件為 API 版的規格，若與單機元件版有差異，以此文件為準（API 端點行為由國健署伺服器定義）。

---

## 版本選擇：單機元件版 vs. API 版

| 面向 | 單機元件版（預設） | API 版 |
|------|----------------|-------|
| **計算位置** | 本地（Python 腳本） | 遠端（國健署伺服器） |
| **網路需求** | 不需要 | 需要 HTTPS 連線 |
| **帳號需求** | 不需要 | 需向國健署申請 |
| **係數/模型更新** | 手動更新本地檔案 | 伺服器端自動更新 |
| **適用場景** | 大量計算、批次處理、離線環境、不可中斷需求 | 已有自建系統、需即時串接、少量即時查詢 |
| **維護停機** | 無（本地運行） | 會依國健署資安政策定期/不定期維護 |
| **結果一致性** | 需自行維護跨平台一致性 | 由國健署統一提供計算 |
| **額外回傳** | 需自行實作 | 自動回傳健康指引(guideline)和風險等級(riskType) |

**建議**：
- **預設使用單機元件版**，適合大多數醫療院所的批次計算和整合場景
- 若院所已有 HIS 系統且需即時串接、少量查詢，可考慮 API 版
- 有大量計算、批次計算或不可中斷需求者，**不建議使用 API 版**（官方建議改用元件版）

---

## API 帳號申請流程

```
申請 → 初審 → 複核 → 開通帳號 → 上線使用
(申請單位)  (社改會)  (國健署)  (國健署)   (社改會)
申請書下載  確認文件   署內呈核   帳號啟用   協助上線
填寫       內容/技術                       測試
           窗口
```

### 申請步驟

1. **下載申請書**：至慢性疾病風險評估平台官網下載申請書
2. **填寫並提交**：填寫完成後提交給執行單位（社團法人臺灣社會改造協會）
3. **初審**：社改會確認文件內容及技術窗口
4. **複核**：國健署署內呈核
5. **帳號開通**：國健署啟用帳號
6. **上線測試**：社改會協助上線測試

### 常見問題與教學影片

- 平台常見問題說明頁面：`https://cdrc.hpa.gov.tw/qa.html`
- 開發階段如需測試帳號，請與國健署或承辦廠商聯絡窗口索取

---

## API 基本規格

| 項目 | 值 |
|------|---|
| Base URL | `https://cdrc.hpa.gov.tw` |
| 認證方式 | HTTP Basic Authentication（帳號密碼 Base64 編碼） |
| Content-Type | `application/json` |
| HTTP Method | POST |
| 回應格式 | JSON |
| 數學模型版本 | V4 |

### 認證方式

使用 HTTP Basic Authentication，將「帳號:密碼」以 Base64 編碼後放入 `Authorization` header：

```
Authorization: Basic <base64(帳號:密碼)>
```

**注意**：帳號或密碼錯誤時，伺服器會回覆「維修中」的網頁內容（不是標準的 401 錯誤）。

---

## API 端點一覽

| 疾病 | API URI | 說明 |
|------|---------|------|
| 冠心病 (CHD) | `/api/hra/v4/chd` | 單一疾病評估 |
| 糖尿病 (Diabetes) | `/api/hra/v4/diabetes` | 單一疾病評估 |
| 高血壓 (Hypertension) | `/api/hra/v4/hypertension` | 單一疾病評估 |
| 腦中風 (Stroke) | `/api/hra/v4/stroke` | 單一疾病評估 |
| 心血管不良事件 (MACE) | `/api/hra/v4/mace` | 單一疾病評估 |
| 五種疾病一次評估 | `/api/hra/v4/allmodel` | 一次回傳全部結果 |

---

## 各端點參數詳細規格

### (1) 冠心病 CHD — `/api/hra/v4/chd`

| 參數 | 說明 | 單位 | 性別限制 |
|------|------|------|---------|
| gender | 性別 (0:女性、1:男性) | — | 共用 |
| age | 年齡 | 歲 | 共用 |
| hdlc | 高密度膽固醇 | mg/dl | 共用 |
| waist | 腰圍 | cm | 共用 |
| hbp | 高血壓病史 (0:無、1:有) | — | **限男性** |
| sbp | 收縮壓 | mm/Hg | **限女性** |
| chol | 總膽固醇 | mg/dl | **限女性** |
| tg | 三酸甘油脂 | mg/dl | **限女性** |

**範例（女性）**：
```json
{"age":50, "gender":0, "waist":85, "sbp":120, "chol":201, "hdlc":60, "tg":120}
```

### (2) 糖尿病 Diabetes — `/api/hra/v4/diabetes`

| 參數 | 說明 | 單位 | 性別限制 |
|------|------|------|---------|
| gender | 性別 (0:女性、1:男性) | — | 共用 |
| age | 年齡 | 歲 | 共用 |
| height | 身高 | cm | 共用 |
| weight | 體重 | kg | 共用 |
| glu | 空腹血糖 | mg/dl | 共用 |
| tg | 三酸甘油脂 | mg/dl | 共用 |
| chol | 總膽固醇 | mg/dl | **限男性** |
| waist | 腰圍 | cm | **限女性** |
| hdlc | 高密度膽固醇 | mg/dl | **限女性** |

**備註**：API 亦接受上傳 `bmi` 來取代 `height` + `weight`。

**範例（女性）**：
```json
{"age":43, "gender":0, "height":165, "weight":60, "waist":74, "glu":91, "tg":85, "hdlc":60}
```

### (3) 高血壓 Hypertension — `/api/hra/v4/hypertension`

| 參數 | 說明 | 單位 | 性別限制 |
|------|------|------|---------|
| gender | 性別 (0:女性、1:男性) | — | 共用 |
| age | 年齡 | 歲 | 共用 |
| sbp | 收縮壓 | mm/Hg | 共用 |
| height | 身高 | cm | 共用 |
| weight | 體重 | kg | 共用 |
| hdlc | 高密度膽固醇 | mg/dl | **限男性** |
| ldlc | 低密度膽固醇 | mg/dl | **限男性** |
| glu | 空腹血糖 | mg/dl | **限女性** |
| tg | 三酸甘油脂 | mg/dl | **限女性** |
| smoke | 吸菸習慣 (0:無、1:有) | — | **限女性** |

**備註**：API 亦接受上傳 `bmi` 來取代 `height` + `weight`。

**範例（男性）**：
```json
{"age":50, "gender":1, "height":165, "weight":60.0, "sbp":119, "hdlc":53, "ldlc":97}
```

### (4) 腦中風 Stroke — `/api/hra/v4/stroke`

| 參數 | 說明 | 單位 | 性別限制 |
|------|------|------|---------|
| gender | 性別 (0:女性、1:男性) | — | 共用 |
| age | 年齡 | 歲 | 共用 |
| sbp | 收縮壓 | mm/Hg | 共用 |
| glu | 空腹血糖 | mg/dl | **限男性** |
| tg | 三酸甘油脂 | mg/dl | **限男性** |
| waist | 腰圍 | cm | **限女性** |
| hbp | 高血壓病史 (0:無、1:有) | — | **限女性** |
| smoke | 吸菸習慣 (0:無、1:有) | — | **限女性** |
| diabetes | 糖尿病病史 (0:無、1:有) | — | **限女性** |

**範例（女性）**：
```json
{"age":40, "gender":0, "hbp":0, "diabetes":1, "smoke":1, "waist":80, "sbp":120}
```

### (5) 心血管不良事件 MACE — `/api/hra/v4/mace`

| 參數 | 說明 | 單位 | 性別限制 |
|------|------|------|---------|
| gender | 性別 (0:女性、1:男性) | — | 共用 |
| age | 年齡 | 歲 | 共用 |
| sbp | 收縮壓 | mm/Hg | 共用 |
| hdlc | 高密度膽固醇 | mg/dl | 共用 |
| waist | 腰圍 | cm | 共用 |
| chol | 總膽固醇 | mg/dl | **限女性** |
| smoke | 吸菸習慣 (0:無、1:有) | — | **限女性** |

**範例（男性）**：
```json
{"age":35, "gender":1, "sbp":130, "waist":80, "hdlc":60}
```

### (6) 五種疾病一次評估 — `/api/hra/v4/allmodel`

參數為前述 5 種疾病所需的參數聯集。當某疾病所需的參數不完整時，server 會略過不計算該疾病。

**範例**：
```json
{"age":50, "gender":0, "height":174, "weight":70, "waist":76, "sbp":119,
 "chol":150, "hdlc":80, "glu":102, "ldlc":100, "tg":105,
 "smoke":0, "hbp":0, "diabetes":0}
```

**回應中的 model 欄位對應**：

| model 值 | 疾病 |
|---------|------|
| CHDRiskV3 | 冠心病 |
| DiabetesRiskV3 | 糖尿病 |
| HypertensionRiskV3 | 高血壓 |
| StrokeRiskV3 | 腦中風 |
| MACERisk | 心血管不良事件 |

> 注意：model 名稱中的 `V3` 是歷史命名，實際使用的數學模型版本由 `ver` 欄位決定（目前為 4）。

---

## API 回應格式

### 單一疾病端點回應

```json
{
  "code": 0,
  "message": "",
  "data": [
    {
      "risk": 8.0,
      "populationAvg": 5.66,
      "multipleDiff": 1.52,
      "ver": 4,
      "seq": 0,
      "guideline": "<<A 限酒>>男性每天酒精少於 20g...<<B 減重>>...",
      "riskType": 0
    }
  ]
}
```

### 回應欄位說明

| 欄位 | 型別 | 說明 |
|------|------|------|
| code | number | 執行狀態碼，0=正常，非0=錯誤 |
| message | string | 錯誤訊息（code=0 時為空字串） |
| data | array | 評估結果陣列 |
| data[].risk | number | 個人十年內可能罹病之風險值(%) |
| data[].populationAvg | number | 相同年齡/性別的族群平均風險值(%) |
| data[].multipleDiff | number | 個人與族群平均值之倍數差異 |
| data[].ver | number | 數學模型版本（目前為 4） |
| data[].seq | number | 序號 |
| data[].guideline | string | 健康指引內容（含衛教建議） |
| data[].riskType | number | 風險等級：0=低風險, 1=中風險, 2=高風險 |
| data[].model | string | 疾病識別（僅 allmodel 端點回傳） |

### 錯誤回應範例

```json
{"code":101, "message":"收縮壓 sbp 是必要的參數，請使用合理的數字", "data":[]}
```

---

## 官網計算邏輯（API 不自動套用，需院所自行實作）

### 生理數值允許範圍

| 中文名稱 | 欄位 | 允許範圍 |
|---------|------|---------|
| 身高 | height | 80-220 cm |
| 體重 | weight | 20-300 kg |
| 血糖 | glu | 20-999 mg/dl |
| 腰圍 | waist | 20-200 cm |
| 高密度膽固醇 | hdlc | < 500 mg/dl |
| 低密度膽固醇 | ldlc | < 594 mg/dl |
| 收縮壓 | sbp | 50-300 mm/Hg |
| 總膽固醇 | chol | 20-999 mg/dl |
| 三酸甘油酯 | tg | 10-9500 mg/dl |

### 病史排除邏輯（官網行為，API 仍會回傳但不具參考性）

- 已勾選**糖尿病史** → server 仍回傳糖尿病風險值，但**不具參考性**，官網不顯示
- 已勾選**高血壓病史** → server 仍回傳高血壓風險值，但**不具參考性**，官網不顯示
- 空腹血糖 > 126 mg/dl → 疑似糖尿病患者，官網**不計算**糖尿病風險值
- 收縮壓 > 140 mmHg → 疑似高血壓患者，官網**不計算**高血壓風險值

### 風險等級判定

| 等級 | riskType | CHD/Stroke/DM/MACE | 高血壓 |
|------|----------|--------------------|----|
| 低風險 | 0 | Risk < 10 | Risk/PopulationAvg < 0.75 |
| 中風險 | 1 | 10 <= Risk < 20 | 0.75 <= Risk/PopulationAvg <= 1.25 |
| 高風險 | 2 | Risk >= 20 | Risk/PopulationAvg > 1.25 |

---

## 開發範例

### Python (requests)

```python
import requests
import base64

BASE_URL = "https://cdrc.hpa.gov.tw"
USERNAME = "你的帳號"
PASSWORD = "你的密碼"

def evaluate_chd(data: dict) -> dict:
    """呼叫冠心病 API 端點"""
    credentials = base64.b64encode(f"{USERNAME}:{PASSWORD}".encode()).decode()
    headers = {
        "Authorization": f"Basic {credentials}",
        "Content-Type": "application/json",
        "Accept": "application/json"
    }
    response = requests.post(
        f"{BASE_URL}/api/hra/v4/chd",
        json=data,
        headers=headers
    )
    return response.json()

# 範例：50歲女性
result = evaluate_chd({
    "age": 50, "gender": 0,
    "waist": 85, "sbp": 120,
    "chol": 201, "hdlc": 60, "tg": 120
})
print(result)
```

### curl

```bash
curl -X POST https://cdrc.hpa.gov.tw/api/hra/v4/chd \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $(echo -n '帳號:密碼' | base64)" \
  -d '{"age":50,"gender":0,"waist":85,"sbp":120,"chol":201,"hdlc":60,"tg":120}'
```

### .NET (HttpClient)

```csharp
var client = new HttpClient();
client.BaseAddress = new Uri("https://cdrc.hpa.gov.tw");
byte[] cred = Encoding.UTF8.GetBytes("帳號:密碼");
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Basic", Convert.ToBase64String(cred));
client.DefaultRequestHeaders.Accept.Add(
    new MediaTypeWithQualityHeaderValue("application/json"));

var content = new StringContent(jsonData, Encoding.UTF8, "application/json");
var response = await client.PostAsync("/api/hra/v4/chd", content);
var result = await response.Content.ReadAsStringAsync();
```

### Java (URLConnection)

```java
String url = "https://cdrc.hpa.gov.tw/api/hra/v4/chd";
String credentials = Base64.getEncoder()
    .encodeToString("帳號:密碼".getBytes("UTF-8"));

URLConnection conn = new URL(url).openConnection();
conn.setRequestProperty("Authorization", "Basic " + credentials);
conn.setRequestProperty("Content-Type", "application/json");
conn.setDoOutput(true);

try (PrintWriter out = new PrintWriter(conn.getOutputStream())) {
    out.print(jsonData);
    out.flush();
}
// 讀取回應...
```

---

## API 版常見問題與故障排除

| 問題 | 原因 | 解決方法 |
|------|------|---------|
| 回覆「維修中」的 HTML 網頁 | 帳號或密碼錯誤 | 確認 Basic Auth 的帳號密碼正確 |
| HTTP 404 | URL 錯誤 | 確認 URL 格式為 `/api/hra/v4/{model}` |
| code=101, message 有錯誤訊息 | 必要參數缺失或格式錯誤 | 檢查該端點所需的必要參數 |
| populationAvg 和 multipleDiff 為 0 | 年齡不在 35-70 歲範圍 | 模型僅適用 35-70 歲 |
| 風險值回傳但不具參考性 | 受測者已有該疾病病史 | 有糖尿病/高血壓病史時忽略對應風險值 |
| 連線逾時 | 伺服器維護中 | 國健署會定期/不定期維護，稍後重試 |
| 參數名稱錯誤無回應 | 參數名稱大小寫不符 | 參數名稱為 **case-sensitive** |
| allmodel 少回傳某疾病 | 該疾病所需參數不完整 | server 會略過參數不足的疾病 |

### 重要注意事項

1. **參數大小寫敏感**：`sbp` 和 `SBP` 是不同的，務必使用小寫
2. **數字可用字串格式**：`"age": "45"` 與 `"age": 45` 都可接受
3. **認證成功 + URL 正確 → 一律 HTTP 200**：業務邏輯錯誤以 JSON 的 `code` 欄位回報
4. **生理數據可有小數點**：如 `"weight": 60.5`
5. **服務可能中斷**：依國健署資安政策會定期/不定期維護
6. **大量計算場景建議用元件版**：API 版不保證高可用性

---

## 單機元件版 vs API 版的參數差異

API 版在某些模型的性別分支中，要求額外的布林參數（`hbp`、`smoke`、`diabetes`），這些在單機元件版的 validation-matrix 中可能未完整列出：

| 模型 | 額外布林參數 | 性別限制 | 說明 |
|------|-----------|---------|------|
| CHD | hbp | 限男性 | 高血壓病史 |
| Stroke | hbp, smoke, diabetes | 限女性 | 病史 + 吸菸 |
| Hypertension | smoke | 限女性 | 吸菸習慣 |
| MACE | smoke | 限女性 | 吸菸習慣 |

**建議**：使用 allmodel 端點時，一律傳入所有欄位（含 hbp、smoke、diabetes），避免因參數不足被略過。
