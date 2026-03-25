# Pportal API 完整參考

## GetBasicData — 健保卡歷史紀錄

### 端點

```
POST https://pportal.hpa.gov.tw/Screening/Screening.aspx/GetBasicData
```

### 用途

讀取健保卡內儲存的**預防保健服務歷史紀錄**。這些是病患在**所有醫療院所**的執行記錄（跨院），不限於本院。

### 回傳格式

```json
{
  "d": {
    "__type": "UB.Screening.Screening+CardBasicData",
    "CardID": "000013375450",
    "Name": "楊伊雅              ",
    "PId": "I2******76",
    "EPId": "I200XXX976",
    "Birthday": "0861227",
    "Gender": "F",
    "CardIssueDate": "0920307",
    "Note": "1",
    "Age": "29歲",
    "PreventDatas": [
      { "Item": "04", "ServiceDate": "1141001", "HospCode": "3522013684", "ServiceCode": "  " },
      { "Item": "03", "ServiceDate": "1140807", "HospCode": "3522023662", "ServiceCode": "31" },
      { "Item": "12", "ServiceDate": "1140527", "HospCode": "3522013684", "ServiceCode": "  " }
    ]
  }
}
```

### CardBasicData 欄位說明

| 欄位 | 說明 | 格式 |
|------|------|------|
| `CardID` | 健保卡卡號 | 12 位數字 |
| `Name` | 姓名（可能含尾隨空白） | 字串，需 trim |
| `PId` | 身分證字號（遮罩） | `X0******00` |
| `EPId` | 加密身分證字號 | `X000XXX000` |
| `Birthday` | 生日 | 7 位民國年 `YYYMMDD` |
| `Gender` | 性別 | `"M"` 男 / `"F"` 女 |
| `CardIssueDate` | 發卡日 | 7 位民國年 `YYYMMDD` |
| `Age` | 年齡（含「歲」字） | 如 `"29歲"` |
| **`PreventDatas`** | **預防保健歷史紀錄陣列** | 見下方 |

### PreventDatas 陣列格式

| 欄位 | 說明 | 格式 |
|------|------|------|
| `Item` | 服務代碼（2 碼） | 見「服務代碼對應表」 |
| `ServiceDate` | 執行日期 | 7 位民國年 `YYYMMDD` |
| `HospCode` | 執行院所代碼 | 10 位數字 |
| `ServiceCode` | 項目代碼 | 2 字元（可能為空白） |

**注意**: 同一 Item 可能有多筆記錄（不同日期、不同院所）。通常取最新一筆顯示。

### 與 VoidAPI2 的關係

| 面向 | GetBasicData | VoidAPI2 |
|------|-------------|----------|
| 資料性質 | 歷史紀錄（已做過什麼） | 資格判定（現在能做什麼） |
| 資料範圍 | 跨院（所有醫療院所） | 跨院（國健署整體判定） |
| 呼叫時序 | 先回傳 | 後回傳 |
| 資料粒度 | 逐筆（日期、院所） | 逐項（符合/不符合） |
| 典型用途 | 顯示「健保卡最近記錄」 | 顯示「是否符合資格」 |

### 整合策略

1. 攔截 GetBasicData → 暫存 `PreventDatas`，建立以 `Item` 代碼為 key 的 map（保留每個 Item 最新一筆）
2. 攔截 VoidAPI2 → 解析 `agreeResult`，同時附帶已暫存的健保卡紀錄一起送出
3. 前端透過 Item 代碼對應表，將健保卡紀錄匹配到對應的篩檢項目行

---

## VoidAPI2 — 篩檢資格判定 API

### 端點

```
POST https://pportal.hpa.gov.tw/Screening/Screening.aspx/VoidAPI2
```

### 回傳格式

```json
{
  "d": {
    "__type": "UB.Models.QueryResult",
    "ReturnCode": "0000",
    "Sex": "F",
    "Age": "29",
    "VerifyResult": true,
    "QueryStatus": true,
    "Native": "0",
    "agreeResult": {
      "Adult": "0", "BcLiver": "0", "Stool": "0",
      "Breast": "?", "Oral": "0", "Pss": "?", "Hpv": "?",
      "Ldct": "?", "Gf": "0",
      "Child1": "0", "Child2": "0", "Child3": "0",
      "Child4": "0", "Child5": "0", "Child6": "0", "Child7": "0",
      "SmokingH": "1", "SmokingM": "1", "Prenatal": "1"
    },
    "ErrorMsg": "0000",
    "ResultMsg": "0000000?0?00000000111??0",
    "BcLiverNeedInformedConsent": false,
    "AdultNeedInformedConsent": false
  }
}
```

### VoidAPI2 頂層欄位

| 欄位 | 說明 |
|------|------|
| `ReturnCode` | `"0000"` 表示成功 |
| `Sex` | 性別：`"M"` 男 / `"F"` 女（來自健保卡） |
| `Age` | 年齡（數字字串） |
| `Native` | 原住民身分：`"0"` 一般、`"1"` 原住民 |
| `agreeResult` | 各項篩檢資格判定（主要資料） |
| `BcLiverNeedInformedConsent` | BC 肝是否需知情同意 |
| `AdultNeedInformedConsent` | 成健是否需知情同意 |

### agreeResult 代碼

| 原始值 | 意義 | 標準化值 | 前端顯示 |
|--------|------|----------|----------|
| `"1"` | 符合資格 | `"1"` | ○ 符合 |
| `"0"` | 不符合資格 | `"2"` | ✗ 不符合 |
| `"?"` | 不適用/無法判定 | `"3"` | △ 待確認 |

### agreeResult 欄位對應

| API Key | 中文名稱 | 說明 | 性別限制 |
|---------|---------|------|----------|
| `Adult` | 成人預防保健 | 成人健康檢查 | 無 |
| `BcLiver` | BC肝篩檢 | BC 型肝炎篩檢 | 無 |
| `Stool` | 定量免疫法糞便潛血檢查 | 腸癌篩檢 | 無 |
| `Breast` | 婦女乳房檢查 | 乳癌篩檢（乳攝影） | 女性 |
| `Oral` | 口腔黏膜檢查 | 口腔篩檢 | 無 |
| `Pss` | 婦女子宮頸抹片檢查 | 子宮頸抹片 | 女性 |
| `Hpv` | 婦女人類乳突病毒檢測服務 | HPV 檢測 | 女性 |
| `Ldct` | 胸部低劑量電腦斷層檢查 | 低劑量肺部 CT | 無 |
| `Gf` | 糞便抗原檢測胃幽門螺旋桿菌 | 胃幽門螺旋桿菌 | 無 |

---

## 服務代碼對應表

適用於 GetBasicData 的 `PreventDatas[].Item` 以及健保卡 RegisterPrevent 記錄：

| 代碼 | 服務名稱 | 常用簡稱 | VoidAPI2 對應 Key |
|------|----------|----------|-------------------|
| `01` | 兒童預防保健 | 兒保 | Child1~7 |
| `02` | 成人預防保健 | 成健 | Adult |
| `03` | 婦女子宮頸抹片檢查 | 子宮頸抹片 | Pss |
| `04` | 流行性感冒疫苗 | 流感 | — |
| `05` | 兒童牙齒預防保健 | 兒牙 | — |
| `06` | 婦女乳房檢查 | 乳攝影 | Breast |
| `07` | 定量免疫法糞便潛血檢查 | 腸篩 | Stool |
| `08` | 口腔黏膜檢查 | 口篩 | Oral |
| `09` | 兒童常規疫苗 | 兒疫 | — |
| `10` | 肺炎鏈球菌疫苗 | 肺鏈 | — |
| `11` | 戒菸服務 | 戒菸 | — |
| `12` | COVID-19 疫苗 | 新冠 | — |
| `13` | 婦女人類乳突病毒檢測 | HPV | Hpv |
| `14` | 低劑量電腦斷層 | LDCT | Ldct |
| `15` | 胃幽門螺旋桿菌 | 幽桿 | Gf |

**注意**: 流感(04)、肺鏈(10)、新冠(12) 等疫苗項目在 GetBasicData 有紀錄，但 VoidAPI2 的 agreeResult 中沒有對應欄位（疫苗資格不由 pportal 判定）。
