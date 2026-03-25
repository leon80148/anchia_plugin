# Session 管理與失敗恢復

## Polling 失敗恢復模式

### 問題場景

XHR Polling（等待 VoidAPI2 / GetBasicData 回傳）可能因網路不穩、伺服器超時或頁面導航中斷而失敗。

### Retry 策略

| 策略 | 適用場景 | 實作 |
|------|---------|------|
| 固定間隔重試 | 短暫斷線 | 每 2 秒重試，最多 5 次 |
| 指數退避 | 伺服器過載 | 2s → 4s → 8s → 16s，最多 4 次 |
| 帶抖動的指數退避 | 多用戶同時操作 | 基礎指數退避 + random(0-1s) |

### Retry 函式模板

```javascript
// Playwright / Puppeteer 通用
async function waitForXhrWithRetry(page, urlPattern, options = {}) {
  const { maxRetries = 3, baseDelay = 2000, timeout = 30000 } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await page.waitForResponse(
        res => res.url().includes(urlPattern) && res.status() === 200,
        { timeout }
      );
      return await response.json();
    } catch (err) {
      if (attempt === maxRetries) throw err;
      const delay = baseDelay * Math.pow(2, attempt) + Math.random() * 1000;
      console.log(`Retry ${attempt + 1}/${maxRetries} after ${Math.round(delay)}ms`);
      await page.waitForTimeout(delay);

      // 可選：重新觸發請求（如重新點擊查詢按鈕）
      // await page.click('#queryButton');
    }
  }
}
```

### 失敗後的恢復流程

```
Polling 失敗
├── HTTP 5xx → 指數退避重試
├── Timeout → 檢查頁面是否仍在預期狀態
│   ├── 頁面正常 → 重新觸發查詢
│   └── 頁面已跳轉 → 重新導航至查詢頁
├── 網路斷線 → 等待網路恢復後重試
└── 所有重試失敗 → 記錄錯誤、通知操作者
```

---

## Session 過期處理

### Cookie TTL 監控

Pportal 的 ASP.NET Session Cookie 預設 20 分鐘過期。

| Cookie | 用途 | TTL | 刷新條件 |
|--------|------|-----|---------|
| `ASP.NET_SessionId` | 會話識別 | 20 分鐘（sliding） | 任何 HTTP 請求 |
| `.ASPXAUTH` | 身份驗證 | 取決於伺服器設定 | 登入時設定 |

### 過期偵測

```javascript
// 監控 Session 狀態
function isSessionExpired(response) {
  // 方式 1：被重導到登入頁
  if (response.url().includes('Login.aspx')) return true;

  // 方式 2：回傳特定錯誤碼
  if (response.status() === 302 || response.status() === 401) return true;

  // 方式 3：回傳 HTML 包含登入表單
  const body = await response.text();
  if (body.includes('txtCaptcha') || body.includes('btnLogin')) return true;

  return false;
}
```

### 自動重新登入流程

```
Session 過期偵測
├── 保存當前工作狀態（已查詢的病患ID、已完成的項目）
├── 重新導航至登入頁
├── 自動填入帳號密碼
├── 處理驗證碼（需人工介入或 OCR）
├── 登入成功 → 恢復到之前的工作狀態
└── 登入失敗 → 通知操作者
```

---

## 稽核日誌記錄指南

### 應記錄的事件

| 事件類型 | 記錄內容 | 重要性 |
|---------|---------|--------|
| 登入/登出 | 時間、帳號、IP、成功/失敗 | 高 |
| 病患資料查詢 | 時間、操作者、病患 ID、查詢類型 | 高 |
| 篩檢結果讀取 | 時間、操作者、病患 ID、篩檢項目 | 高 |
| 系統錯誤 | 時間、錯誤類型、堆疊追蹤 | 中 |
| Session 事件 | 建立、過期、刷新 | 低 |

### 日誌格式

```json
{
  "timestamp": "2024-03-15T10:30:00+08:00",
  "level": "INFO",
  "event": "patient_query",
  "actor": "operator_001",
  "patient_id": "A123456789",
  "action": "VoidAPI2_request",
  "result": "success",
  "duration_ms": 2350,
  "metadata": {
    "screening_items_count": 5,
    "session_age_minutes": 12
  }
}
```

### 隱私注意事項

- 日誌中的身分證字號必須遮罩（如 `A1234*****`）
- 不記錄篩檢結果的具體數值
- 日誌保留期限依機構規定（建議至少 3 年）
- 日誌存取需有權限控制
