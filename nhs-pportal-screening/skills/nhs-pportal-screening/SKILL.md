---
name: nhs-pportal-screening
description: |
  Web 自動化整合與外部系統嵌入的通用設計原則和實作模式，附帶國健署預防保健篩檢系統
  （pportal.hpa.gov.tw）作為完整參考實作。涵蓋 Session 管理、多視窗生命週期、
  DOM Polling、表單填入、XHR 攔截、登入偵測等 7 大通用原則，
  適用於 Electron、Puppeteer、Playwright、Selenium 等所有瀏覽器自動化框架。

  觸發詞：pportal、國健署篩檢、預防保健查詢、篩檢資格查詢、VoidAPI2、GetBasicData、
  健保卡掃描、XHR攔截、Session管理、Web自動化整合、外部系統嵌入、
  pportal.hpa.gov.tw、DOM Polling、表單自動填入、多視窗生命週期、
  Electron整合、Puppeteer整合、Playwright整合、Selenium整合、
  成健篩檢查詢、BC肝篩檢查詢、大腸癌篩檢、口腔癌篩檢
---

# Web 自動化整合設計模式

包含 7 大通用設計原則（Part 1）和國健署篩檢系統參考實作（Part 2）。

---

# Part 1：通用 Web 自動化設計原則

以下原則適用於所有需要「嵌入外部 Web 系統並自動化操作」的場景。每個原則附帶跨框架範例。

## 適用框架

| 框架 | 類型 | Session 重用 | XHR 攔截 | DOM 操作 |
|------|------|-------------|----------|---------|
| **Electron** | 桌面嵌入 | `persist:xxx` partition | CDP `Network.getResponseBody` | `executeJavaScript` |
| **Puppeteer** | Node.js headless | `userDataDir` | `page.on('response')` | `page.evaluate()` |
| **Playwright** | Node.js headless | `storageState` | `page.on('response')` | `page.evaluate()` |
| **Selenium** | 多語言 WebDriver | `user-data-dir` Chrome option | Proxy / extension | `execute_script()` |
| **Cypress** | 前端 E2E 測試 | `cy.session()` | `cy.intercept()` | `cy.get()` |

---

## 系統概述（國健署篩檢系統 — Part 2 參考實作的背景）

國健署（國民健康署，HPA）提供「預防保健服務資訊查詢」系統，供醫療院所查詢病患的各項篩檢資格。

### 三種資料來源

| 來源 | 方式 | 查詢範圍 | 需求 | 資料性質 |
|------|------|----------|------|----------|
| **Pportal VoidAPI2** | 瀏覽器登入 pportal.hpa.gov.tw | 全項目（10+ 項） | 帳號密碼 + 驗證碼 + 健保卡 | 即時篩檢資格判定 |
| **Pportal GetBasicData** | 同上（伴隨 VoidAPI2 自動觸發） | 健保卡歷史紀錄 | 同上 | 跨院歷史執行記錄 |
| **BhpNhi（健保IC卡）** | 本機 CLI 呼叫 BhpNetNhiEx.exe | 成健 + BC肝（2 項） | 健保IC卡讀卡機 + 控制軟體 | 即時資格判定 |

### 選擇決策：該用哪個？

```
需要查哪些篩檢項目？
├── 只需成健 + BC肝（2 項）
│   └── 有讀卡機嗎？
│       ├── YES → 用 BhpNhi（最快、免登入、免驗證碼）
│       └── NO → 用 Pportal VoidAPI2（需瀏覽器自動化）
│
├── 需要大腸癌、口腔癌、乳攝、HPV、LDCT 等（3+ 項）
│   └── 必須用 Pportal VoidAPI2（BhpNhi 不支援這些項目）
│
└── 需要跨院歷史紀錄（病人在其他醫院做過什麼）
    └── 必須用 Pportal GetBasicData（本機讀卡只有資格，沒有歷史）
```

**實務建議**：多數診所同時需要成健 + 其他篩檢項目 → 直接整合 Pportal。BhpNhi 適合只做成健/BC肝的場景，或作為 Pportal 不可用時的 fallback。

---

## 7 大核心設計原則

### 原則 1：Session 重用 — 認證成本最小化 {#principle-1}

**問題**：外部系統的登入流程通常包含驗證碼、MFA 等人工介入步驟，每次查詢都重新登入會嚴重影響效率。

**策略**：將「已認證的瀏覽器環境」視為可重用資源，查詢入口依存活狀態分層處理：

```
查詢請求進入
  │
  ├── 層級 1（最快）：操作頁面仍在 → 直接在頁面上觸發動作
  ├── 層級 2（次快）：主視窗仍在 → 從主視窗導航到操作頁面
  └── 層級 3（最慢）：都不在    → 重建視窗 → 偵測 Cookie 是否有效
      ├── Cookie 有效 → 跳過登入，直接導航
      └── Cookie 過期 → 完整登入流程
```

**關鍵規則**：
- 每一層都要做 `isDestroyed()` 檢查，視窗可能被使用者手動關閉
- Cookie 持久化（Electron `persist:xxx` / Puppeteer userDataDir）是層級 3 的基礎
- 查詢前要重置暫態資料（如換卡查詢要清除上一張卡的快取），避免資料錯配

**跨框架 Session 持久化**：
| 框架 | 實作方式 |
|------|---------|
| Electron | `session.fromPartition('persist:myapp')` |
| Puppeteer | `puppeteer.launch({ userDataDir: './session' })` |
| Playwright | `context.storageState()` → `browser.newContext({ storageState })` |
| Selenium | `options.add_argument('--user-data-dir=./session')` |
| Cypress | `cy.session('login', loginFn)` |

### 原則 2：多視窗生命週期管理 {#principle-2}

**問題**：外部系統常用 `window.open` 開子視窗，若不追蹤引用，無法重用也無法正確清理。

**策略**：
- **建立時保存引用**：`did-create-window` / `window.open` handler 中立即保存
- **關閉時清空引用**：監聽 `closed` 事件，設為 null
- **銷毀時清理資源**：detach debugger、取消計時器、釋放攔截器
- **統一清理入口**：提供 `close()` 方法，由外到內（子視窗 → 主視窗）依序清理

**防禦性檢查模式**：
```javascript
if (this._window && !this._window.isDestroyed()) {
  // 安全操作
}
```

**跨框架子視窗處理**：
| 框架 | 捕捉子視窗 | 監聽關閉 |
|------|-----------|---------|
| Electron | `webContents.on('did-create-window', win => ...)` | `win.on('closed', () => ref = null)` |
| Puppeteer | `page.on('popup', popup => ...)` | `popup.on('close', () => ...)` |
| Playwright | `page.on('popup', popup => ...)` | `popup.on('close', () => ...)` |
| Selenium | `driver.window_handles` + `driver.switch_to.window()` | Manual polling |

### 原則 3：DOM 偵測用 Polling，不用導航事件 {#principle-3}

**問題**：SPA / 同頁 Modal / ASP.NET Partial PostBack 不會觸發頁面導航事件（`did-navigate`），無法用傳統方式偵測 UI 狀態變化。

**策略**：
- 使用 `setInterval` + `executeJavaScript` 定期掃描 DOM
- 每個 polling 都要設 **maxAttempts 上限**，避免永久輪詢
- 偵測成功後立即 `clearInterval`
- 用 **flag 變數**（如 `_autoFillDone`）防止重複執行

**適用場景**：
- 登入 Modal 出現偵測
- 登入後頁面狀態變化偵測
- 使用者手動輸入完成偵測（如驗證碼）

**跨框架 Polling 等效方式**：
```javascript
// Puppeteer/Playwright: waitForSelector (比手動 polling 更好)
await page.waitForSelector('#loginModal', { visible: true, timeout: 10000 });

// Playwright: waitForFunction (自定義條件)
await page.waitForFunction(() => document.querySelector('#status')?.textContent === 'Ready');

// Cypress: 內建重試機制
cy.get('#loginModal', { timeout: 10000 }).should('be.visible');

// Selenium: WebDriverWait
WebDriverWait(driver, 10).until(EC.visibility_of_element_located((By.ID, 'loginModal')));
```

### 原則 4：表單填入 — 模擬真實使用者輸入 {#principle-4}

**問題**：ASP.NET WebForms 的 `__doPostBack` 和各種 JS 框架依賴 DOM 事件鏈（focus → input → change），直接設定 `.value` 不會觸發這些事件。

**策略**（按可靠度排序）：
1. **Electron `insertText()`**：最可靠，模擬 OS 層鍵盤輸入
2. **Puppeteer/Playwright `page.type()`**：逐字輸入，觸發完整事件鏈
3. **手動 dispatch 事件**：設 `.value` 後 dispatch `input` + `change` 事件
4. **直接設 `.value`**：最不可靠，許多框架不會接受

**填入前必做**：focus → select（清除舊值）→ 再輸入

**跨框架表單填入**：
| 框架 | 最佳方式 | 備選方式 |
|------|---------|---------|
| Electron | `webContents.insertText(text)` | `executeJavaScript` dispatch events |
| Puppeteer | `page.type('#field', text)` | `page.$eval('#field', el => el.value = text)` + dispatch |
| Playwright | `page.fill('#field', text)` (自動觸發事件) | `page.type('#field', text)` 逐字 |
| Selenium | `element.send_keys(text)` | `execute_script` 設值 + dispatch |
| Cypress | `cy.get('#field').type(text)` | `cy.get('#field').invoke('val', text).trigger('change')` |

### 原則 5：按鈕/元素匹配 — 精確優先 {#principle-5}

**問題**：同一頁面可能有多個文字相似的按鈕（如「登入」、「服務登入」、「一般登入」），模糊匹配會點錯。

**策略**（按優先度）：
1. **ID 精確匹配**：`document.querySelector('#ctl00_lbtnLogin')`
2. **Placeholder 精確匹配**：`input.placeholder === '請輸入帳號'`
3. **文字精確匹配**：`text === '實體卡查詢'`（用 `===` 不用 `includes`）
4. **文字包含匹配**：`text.includes('篩檢資格查詢')`（僅限無歧義時）
5. **DOM 順序遍歷**：`querySelectorAll` 逐一比對，找到第一個符合的

**避免**：
- `input[type="password"]`（可能匹配到變更密碼欄位）
- `text.includes('登入')`（會匹配到多個 tab 和按鈕）

**跨框架元素定位優先級**：
```javascript
// Playwright (推薦 — 內建語義定位器)
await page.getByRole('button', { name: '登入' }).click();       // 最佳
await page.getByPlaceholder('請輸入帳號').fill('user');           // 次佳
await page.locator('#ctl00_lbtnLogin').click();                   // ID 精確

// Cypress
cy.contains('button', '登入').click();                            // 文字精確
cy.get('#ctl00_lbtnLogin').click();                               // ID 精確

// Selenium
driver.find_element(By.ID, 'ctl00_lbtnLogin').click()            // ID 精確
driver.find_element(By.XPATH, "//button[text()='登入']").click()  // 文字精確
```

### 原則 6：XHR 攔截 — 攔截 Response Body {#principle-6}

**問題**：瀏覽器 `webRequest` API 只能取得 headers，無法取得 response body。自動化場景需要讀取 API 回傳的完整 JSON。

**策略**：
- **Electron**：attach Chrome DevTools Protocol → `Network.enable` → 監聯 `Network.responseReceived` → `Network.getResponseBody`
- **Puppeteer/Playwright**：`page.on('response', r => r.json())`
- **子視窗也要攔截**：`did-create-window` 事件中對新視窗重複設定

**過濾原則**：只攔截目標 XHR（用 URL pattern + type 過濾），忽略靜態資源，避免效能影響。

**跨框架 XHR 攔截**：
```javascript
// Puppeteer
page.on('response', async response => {
  if (response.url().includes('VoidAPI2')) {
    const json = await response.json();
    // process json
  }
});

// Playwright
page.on('response', async response => {
  if (response.url().includes('VoidAPI2')) {
    const json = await response.json();
  }
});

// Playwright (更好的方式 — waitForResponse)
const response = await page.waitForResponse(r => r.url().includes('VoidAPI2'));
const data = await response.json();

// Cypress
cy.intercept('POST', '**/VoidAPI2').as('voidApi');
cy.wait('@voidApi').then(interception => {
  const body = interception.response.body;
});
```

### 原則 7：登入狀態偵測 — 頁面語義推斷 {#principle-7}

**問題**：外部系統通常沒有公開的「是否已登入」API，需從頁面 DOM 推斷。

**策略**：偵測「僅登入後才會出現的 UI 元素」，常見指標：
- **登出按鈕/連結**：出現「登出」文字 → 已登入
- **功能入口連結**：出現「篩檢資格查詢」等需登入才可見的連結 → 已登入
- **使用者名稱顯示**：頁面顯示帳號或姓名 → 已登入

**注意**：偵測需延遲執行（如 500ms），等待動態內容渲染完成。

---

---

# Part 2：國健署預防保健篩檢系統（參考實作）

以下為 Part 1 原則的完整實作範例，展示如何將上述 7 大原則應用於真實系統。

## Pportal 系統（Web Portal）

### URL 與頁面結構

- **首頁**: `https://pportal.hpa.gov.tw/Web/Notice.aspx`
- **技術框架**: ASP.NET WebForms
- **登入方式**: 首頁的「服務登入」按鈕 → 同頁 Modal（非新頁面）
- **所有 AJAX 回傳**: 包裹在 ASP.NET `{"d": ...}` 格式中

### 登入 Modal DOM 結構

登入 modal 是 ASP.NET 的同頁面彈出框，有以下關鍵元素：

#### Tab 切換
- **「一般登入」tab** → 切換到 `#tab1`
- **「服務登入」tab** → 切換到 `#tab2`（帳號密碼登入在此）
- **「憑證登入」tab** → 切換到 `#tab3`

#### 表單欄位

| 用途 | ID | Placeholder | type |
|------|-----|------------|------|
| 帳號 | `ctl00_txtAccount` | `請輸入帳號` | text |
| 密碼 | `ctl00_txtPassword` | `請輸入密碼` | password |
| 驗證碼 | `ctl00_txtCaptcha` (推測) | 含「驗證碼」 | text |
| 登入按鈕 | `ctl00_lbtnLogin` | — | submit/anchor |

#### 危險的相似欄位（必須避免匹配到）

| 欄位 | ID | Placeholder | 說明 |
|------|-----|------------|------|
| 變更密碼-舊密碼 | `ctl00_txt_OldPswd` | `請輸入您目前密碼` | `input[type="password"]` 會先匹配到此欄位 |
| 變更密碼-新帳號 | `ctl00_txt_NewPAccount` | `請輸入帳號` | 在 DOM 中可能比 `txtAccount` 更早出現 |
| 顯示密碼核取框 | `ctl00_chkShowPassword` | — | `input[name*="Password"]` 會匹配到 |

### 登入後流程

1. 登入成功 → 頁面導向篩檢查詢頁
2. 查詢操作在**子視窗**中開啟（`window.open`）
3. 子視窗中使用者點擊「實體卡查詢」或「虛擬卡查詢」
4. 系統讀取健保卡後，依序觸發多個 XHR

---

## Screening.aspx XHR 端點總覽

查詢觸發後，子視窗會依序發出以下 XHR 請求：

### 呼叫時序

```
使用者點擊「查詢」（實體卡/虛擬卡）
  │
  ├── 1. GetHospData        → 取得醫院名稱
  ├── 2. CheckAccount        → 驗證登入帳號
  ├── 3. GetNoticeToSync     → 同步公告資訊
  │
  ├── 4. GetBasicData ★      → 健保卡基本資料 + 歷史紀錄（PreventDatas）
  ├── 5. WriteInputLog       → 寫入查詢紀錄
  │
  └── 6. VoidAPI2 ★          → 篩檢資格判定結果（agreeResult）
```

**重點**: GetBasicData **先於** VoidAPI2 回傳。攔截時需先暫存 GetBasicData，待 VoidAPI2 回傳時合併送出。

### 端點明細

| 端點 | 用途 | 回傳範例 | 是否需攔截 |
|------|------|----------|-----------|
| `GetHospData` | 醫院名稱 | `{"d": "安家診所"}` | 否 |
| `CheckAccount` | 驗證帳號 | `{"d": {"PersonSNO": "168195", "PAccount": "anchia"}}` | 否 |
| `GetNoticeToSync` | 公告同步 | `{"d": [{...}]}` | 否 |
| **`GetBasicData`** | **健保卡紀錄** | **見下方** | **是** |
| `WriteInputLog` | 寫入查詢紀錄 | `{"d": "前台資料存檔成功"}` | 否 |
| **`VoidAPI2`** | **篩檢資格判定** | **見下方** | **是** |

---

## GetBasicData 與 VoidAPI2 API 摘要

> 完整回傳格式、欄位說明、服務代碼對應表 → `references/pportal-api-reference.md`

**GetBasicData** (`POST .../GetBasicData`)：回傳健保卡歷史記錄（`PreventDatas` 陣列），每筆含 `Item`(服務碼)、`ServiceDate`(民國年)、`HospCode`(院所碼)。**先於 VoidAPI2 回傳，需暫存後合併。**

**VoidAPI2** (`POST .../VoidAPI2`)：回傳當前篩檢資格判定（`agreeResult`），值為 `"1"`=符合、`"0"`=不符合、`"?"`=不適用。主要欄位：Adult、BcLiver、Stool、Breast、Oral、Pss、Hpv、Ldct、Gf。

**整合順序**：GetBasicData 先到 → 暫存 `PreventDatas` → VoidAPI2 到達 → 合併 `agreeResult` → 輸出完整結果

---


---

## BhpNhi 代碼格式（健保IC卡本機查詢）

### ValidAll 回傳格式：`XXYY-ZZ`

```
XX: BC肝資格
  00 = 未查
  01 = 符合資格
  02 = 年齡不符
  03 = 已做過

YY: 成健資格
  00 = 未查
  01 = 符合資格
  02 = 年齡不符
  03 = 已做過

ZZ: 身分別
  00 = 一般
  01 = 原住民
```

**範例**: `0101-00` = BC肝符合 + 成健符合 + 一般身分

### 錯誤代碼

| 代碼 | 意義 |
|------|------|
| `9999` | Token 驗證失敗 |
| `9998` | 讀取健保控制軟體/健保卡/SAM 驗證失敗 |
| `9001` | 資料傳入格式檢核不正確 |
| `9002` | 個案健保卡註記含空白或特殊字元 |
| `0000` | Token 驗證成功（但無查詢結果） |

### BC肝登記/取消代碼

| 動作 | 代碼 | 意義 |
|------|------|------|
| 登記成功 | `1100-ZZ` | BC肝篩檢登記成功 |
| 年齡不符 | `1200-ZZ` | 登記失敗：年齡不符合 |
| 已做過 | `1300-ZZ` | 登記失敗：已做過 |
| 取消成功 | `2100-ZZ` | BC肝篩檢登記取消成功 |
| 未登記 | `2200-ZZ` | 取消失敗：未在本院所登記 |
| 逾期 | `2300-ZZ` | 取消失敗：作業已逾期 |

---

## 健保卡預防保健服務記錄（RegisterPrevent）

健保卡內記錄的保健服務記錄，每組 21 bytes，最多 6 組：

```
位置 0-1   (2 bytes): 服務代碼（對應「服務代碼對應表」）
位置 2-8   (7 bytes): 檢查日期（民國年 YYYMMDD）
位置 9-18  (10 bytes): 醫院代碼
位置 19-20 (2 bytes): 項目代碼
```

---

## 自動化登入技術要點

### 1. 登入表單是同頁 Modal

登入表單不會產生頁面導航（no navigation），是 JavaScript 動態顯示的 modal。因此：
- 不能用頁面載入事件偵測 modal 出現
- 需要用 **polling（定時查詢 DOM）** 偵測 modal 是否出現
- 按鈕文字匹配需精確：只匹配「服務登入」，避免匹配到「一般登入」tab

### 2. 表單欄位匹配策略

用 `placeholder` 精確匹配最可靠：
- 帳號：`placeholder === '請輸入帳號'`
- 密碼：`placeholder === '請輸入密碼'`

**不要用** `input[type="password"]`（會匹配到變更密碼的 `ctl00_txt_OldPswd`）。

### 3. ASP.NET WebForms 填入注意

ASP.NET 的 `__doPostBack` 機制依賴 JavaScript 事件。直接設定 `input.value` 可能不會觸發 ASP.NET 的 change 事件。建議：
- 使用模擬鍵盤輸入（如 Electron 的 `insertText`、Puppeteer 的 `page.type`）
- 或在設值後手動觸發 `input`/`change` 事件

### 4. 驗證碼

- 為 4 碼數字的圖形驗證碼，無法自動辨識
- 需等使用者手動輸入
- 可用 polling 偵測驗證碼欄位的值長度 ≥ 4 後自動點擊「登入」按鈕

### 5. Cookie 持久化

登入成功後的 session cookie 應保留，避免每次都需重新登入：
- Electron：使用 `session.fromPartition('persist:xxx')`
- Puppeteer/Playwright：儲存 cookies 到檔案
- 一般瀏覽器自動化：使用 user data directory

### 6. XHR 攔截策略

需攔截 response body（不只 headers），常用方法：
- **Electron**: Chrome DevTools Protocol（CDP）的 `Network.getResponseBody`
- **Puppeteer/Playwright**: `page.on('response')` + `response.json()`
- **瀏覽器擴充**: `chrome.devtools.network` 或 `fetch` monkey-patching

### 7. 子視窗攔截

國健署的篩檢頁面在新視窗中開啟（`window.open`），對子視窗也需要設定 XHR 攔截。
（參照原則 2「多視窗生命週期管理」和原則 6「XHR 攔截」）

---

## 性別判斷

判斷病患性別以決定是否顯示女性專屬篩檢項目時，應綜合多個來源：

| 來源 | 欄位 | 值 |
|------|------|-----|
| 本院資料庫 | `msex` | `'1'` 男 / `'2'` 女 |
| VoidAPI2 回傳 | `Sex` | `"M"` 男 / `"F"` 女 |
| GetBasicData 回傳 | `Gender` | `"M"` 男 / `"F"` 女 |

建議：任一來源判定為女性即顯示女性專屬項目（子宮頸抹片、乳攝影、HPV）。

---

## 注意事項

1. **VoidAPI2 / GetBasicData 回傳格式可能隨國健署改版而變化**，需定期確認
2. **agreeResult 值為字串型別**（`"1"` 不是 `1`）
3. **篩檢項目名稱必須與 VoidAPI2 回傳完全一致**（如「定量免疫法糞便潛血檢查」不能簡寫）
4. **BhpNhi 需要健保IC卡讀卡機和控制軟體**，Pportal 只需帳號密碼
5. **驗證碼為 4 碼數字**，是圖形驗證碼，無法自動辨識
6. **登入頁面使用 ASP.NET WebForms 的 `__doPostBack` 機制**，表單提交非標準 HTML form submit
7. **GetBasicData 先於 VoidAPI2 回傳**，攔截時需注意時序：先暫存健保卡紀錄，待 VoidAPI2 到達時合併
8. **PreventDatas 同一 Item 可能有多筆記錄**（如多次流感施打），取最新一筆即可
9. **`hisGetBasicData` ≠ `GetBasicData`**：前者是 pportal 網頁呼叫本機讀卡軟體的 client-side API，後者是 pportal 伺服器回傳的 server-side XHR。讀卡失敗時會出現「等待 hisGetBasicData 回應超時」錯誤

---

## Polling 失敗恢復與 Session 管理

XHR Polling 失敗時採用**指數退避重試**（2s → 4s → 8s）。ASP.NET Session Cookie 預設 20 分鐘過期（sliding），偵測方式：URL 包含 `Login.aspx`、HTTP 302/401、或 body 含 `txtCaptcha`。

> Retry 函式模板、Session 過期處理程式碼、稽核日誌格式 → `references/session-management.md`

---

## 邊界與失敗模式

**這個技能對應的是「設計模式知識」，不是執行層：**

| 情況 | 為什麼失敗 | 實際症狀 |
|------|-----------|---------|
| 國健署改版 pportal DOM 結構 | 所有 selector 和 API endpoint 都可能變 | 登入 modal 找不到、VoidAPI2 格式不同 |
| ASP.NET Session 不穩定 | 閒置超過 20 分鐘自動登出 | 下一個查詢突然需要重新登入 |
| 驗證碼無法自動化 | 圖形驗證碼設計目的就是阻擋自動化 | 技能設計中「等人工輸入」是預期行為，不是缺陷 |
| 子視窗無法攔截 XHR | 忘記對子視窗也設定 CDP / response listener | GetBasicData 或 VoidAPI2 無法被攔截 |
| GetBasicData 先到、VoidAPI2 後到，但程式碼只等一個 | 時序問題：兩個都要攔截 | 病患資料只有一半 |

### 錯誤恢復決策樹

查詢失敗時，按以下順序診斷：

```
查詢失敗
├── HTTP 302 / URL 含 Login.aspx / body 含 txtCaptcha？
│   └── YES → Session 過期
│       ├── 嘗試 1：重用 Session Cookie，自動導回查詢頁
│       ├── 嘗試 2：保留 partition/userDataDir，重新載入首頁
│       └── 嘗試 3：需人工重新登入（驗證碼）→ 通知操作者
│
├── selector 找不到元素（#ctl00_txtAccount 等）？
│   └── YES → DOM 結構可能已改版
│       ├── 嘗試 1：檢查元素是否在 iframe 內（國健署偶爾加 iframe 包裝）
│       ├── 嘗試 2：用 document.querySelectorAll('[id*=txtAccount]') 模糊搜尋
│       └── 嘗試 3：記錄新 DOM 結構，更新 selector 對應表
│
├── XHR 攔截到但 response body 為空或格式異常？
│   └── YES → API 回傳格式變更
│       ├── 嘗試 1：印出完整 response body 比對欄位名稱是否改名
│       ├── 嘗試 2：檢查 Content-Type 是否從 JSON 變 XML
│       └── 嘗試 3：在測試環境重新抓取完整回傳格式，更新解析邏輯
│
└── VoidAPI2 / GetBasicData 完全攔截不到？
    └── 檢查子視窗是否正確設定 listener（原則 2 + 原則 6）
        ├── Electron：CDP session 是否綁定到正確的 webContents
        ├── Puppeteer/Playwright：page.on('response') 是否在 newPage 事件後設定
        └── 如果子視窗 URL 變更，更新 URL pattern 匹配邏輯
```

**恢復後必做**：每次手動恢復後，更新整合健康檢查清單中的對應項目通過標準。

**已知的系統性風險：**
- pportal 是外部系統，任何改版都可能讓現有整合失效，需要主動維護
- 這個技能提供的 DOM selector 是基於特定時間點的觀察，不保證永久有效

## 成功的樣子

**完整整合驗證流程（按順序）：**

1. Session 層級：登入後 30 分鐘內不需重新登入 ✓
2. XHR 攔截：GetBasicData 和 VoidAPI2 都被攔截到 ✓
3. 資料完整性：`agreeResult` 包含至少 9 個欄位（Adult, BcLiver, Stool... 等）✓
4. 時序正確：GetBasicData 先到，VoidAPI2 後到，兩者合併後再輸出 ✓
5. 錯誤處理：測試 Session 過期情境，確認自動重新登入流程能觸發 ✓

---

## 整合健康檢查清單

國健署會不定期更新 pportal 系統。每次更新或定期（每月建議一次）執行以下檢查：

### 自動化可驗證項目

| # | 檢查項目 | 驗證方式 | 通過標準 |
|---|---------|---------|---------|
| 1 | 登入頁面載入 | 訪問首頁 URL，檢查 HTTP 200 | 頁面包含「服務登入」文字 |
| 2 | 登入 Modal DOM | Polling `#ctl00_txtAccount` | 10 秒內找到元素 |
| 3 | 帳號/密碼欄位 | Placeholder 匹配 `請輸入帳號`/`請輸入密碼` | 精確匹配 |
| 4 | 登入按鈕 | `#ctl00_lbtnLogin` 存在 | 元素可點擊 |
| 5 | 登入成功偵測 | 頁面含「登出」或功能入口連結 | 30 秒內偵測到 |
| 6 | 子視窗開啟 | `window.open` 攔截 | 取得子視窗引用 |
| 7 | GetBasicData XHR | 攔截 URL 含 `GetBasicData` | 回傳含 `PreventDatas` |
| 8 | VoidAPI2 XHR | 攔截 URL 含 `VoidAPI2` | 回傳含 `agreeResult` |
| 9 | agreeResult 欄位完整 | 檢查 9 個必要欄位 | Adult, BcLiver, Stool, Breast, Oral, Pss, Hpv, Ldct, Gf 都存在 |
| 10 | 時序正確 | GetBasicData 回傳時間 < VoidAPI2 回傳時間 | GetBasicData 先到 |

### 手動驗證項目（需人工介入）

| # | 檢查項目 | 方法 |
|---|---------|------|
| 11 | 驗證碼圖片 | 確認驗證碼圖片可顯示、4 碼輸入欄位正常 |
| 12 | 實際查詢結果 | 用已知病患查詢，比對結果與預期 |
| 13 | Session 持久性 | 登入後等 15 分鐘，確認不需重新登入 |

### 更新後快速驗證指令（概念）

```javascript
// 自動化檢查腳本骨架（Playwright 版）
async function healthCheck(page) {
  // 1. 首頁載入
  await page.goto('https://pportal.hpa.gov.tw/Web/Notice.aspx');
  assert(await page.textContent('body')).includes('服務登入');

  // 2. 登入 Modal DOM 結構
  await page.click('text=服務登入');
  const account = await page.waitForSelector('#ctl00_txtAccount', { timeout: 10000 });
  assert(await account.getAttribute('placeholder')).toBe('請輸入帳號');

  // 3. XHR 端點存在性（需實際登入後驗證）
  // ... 後續需要人工輸入驗證碼
  console.log('DOM 結構檢查通過，XHR 驗證需手動完成');
}
```

---

## 參考文件

| 文件 | 內容 |
|------|------|
| `references/pportal-api-reference.md` | GetBasicData/VoidAPI2 完整回傳格式、欄位說明、服務代碼對應表 |
| `references/session-management.md` | Polling Retry 模板、Session 過期偵測、稽核日誌格式 |
