---
name: nhi-ic-card
description: |
  健保IC卡讀卡整合技能。涵蓋 BhpNhi.dll API 開發、回應碼解碼、成健/BC肝篩檢工作流程、
  IC卡欄位結構、資料上傳格式2.0。支援 .NET、Node.js、Python 等多語言整合方案。

  觸發詞：BhpNhi.dll、CsHis.dll、健保卡讀卡、健保IC卡、IC卡讀卡機、
  BhpNetNhiEx、健保元件、成人健檢API、BC肝篩檢、篩檢資格查詢、
  讀卡API、健保卡欄位、上傳格式2.0、XML上傳、健保IC整合、
  虛擬健保卡、HCA晶片卡、讀卡機整合、多讀卡機、GUID token
---

# Skill: 健保IC卡整合開發參考

本 Skill 整合台灣健保IC卡讀卡系統的完整知識，涵蓋 API 開發、篩檢工作流程、卡片欄位結構、資料上傳格式四大主題。

## 使用時機

當以下任一條件符合時，應自動啟用：

- 提到 **BhpNhi.dll**、**CsHis.dll**、**健保卡讀卡**、**IC卡讀卡機**
- 需要整合 **成人健檢**、**BC肝篩檢** API
- 開發或修改 **.NET 讀卡應用程式**
- 需要處理 **API 回應碼**、**錯誤碼**
- 診所人員詢問**篩檢資格代碼**的意思
- 查詢 **IC 卡欄位結構**、**RegisterPrevent** 解析
- 需要產生**健保卡上傳 XML** 或處理異常上傳情境
- 查詢**就醫類別 (M07)** 與欄位交叉檢核規則

---

## 系統架構

```
HIS 系統
  |
  +-- BhpNhi.dll（封裝層，本專案使用）
  |     +-- 內部呼叫 CsHis.dll（讀卡機控制軟體 6.0）
  |     +-- 內部呼叫 HpaBhp.dll（Token 驗證）
  |     +-- 呼叫國健署 API（成健/BC肝資格查詢）
  |
  +-- 讀卡機硬體 + SAM 卡
  +-- 健保卡控制軟體 6.0 主控台
```

---

## API 方法速查表

```csharp
var bhpNhi = new BhpNhi.Bhp();
```

| 方法 | 用途 | 回傳 | 前置條件 |
|------|------|------|----------|
| `GetPersonData()` | 讀卡 + 建立 Token | JSON → `Person` | 無（**必須最先呼叫**） |
| `ValidAdult()` | 成健資格查詢 | JSON → `ApiResponse` | 需先 GetPersonData |
| `ValidBcLiver()` | BC肝資格查詢 | JSON → `ApiResponse` | 需先 GetPersonData |
| `ValidAll()` | 成健+BC肝同時查詢 | JSON → `ApiResponse` | 需先 GetPersonData |
| `RegisterBcLiver()` | BC肝篩檢登記 | JSON → `ApiResponse` | 需先 GetPersonData |
| `CancelBcLiver()` | BC肝取消登記 | JSON → `ApiResponse` | 需先 GetPersonData |
| `GetVersion()` | DLL 版本 | string | 無 |
| `GetStatus()` | 主控台狀態 | string | 無 |

---

## 回應碼格式 XXYY-ZZ 速記

```
  XX    YY  -  ZZ
  BC肝  成健    身分別

  01 = 未做過（符合資格）    00 = 一般身分
  02 = 年齡不符              01 = 原住民
  03 = 已做過
  00 = 未查
```

快速範例：`0101-00` = 兩者都符合(一般) | `0303-01` = 都做過(原住民)

### 登記/取消碼

| 前綴 | 意義 |
|------|------|
| `1100` | 登記成功 |
| `1200` | 登記失敗（年齡不符） |
| `1300` | 登記失敗（已做過） |
| `2100` | 取消成功 |
| `2200` | 取消失敗（無登記紀錄） |
| `2300` | 取消失敗（逾期） |

### 錯誤碼

| Code | 說明 |
|------|------|
| `9999` | Token 驗證失敗（未先讀卡/網路不通） |
| `9998` | 主控台/SAM/卡片驗證失敗 |
| `9001` | 資料格式檢核不正確 |
| `9002` | 健保卡註記含空白或特殊字元 |

---

## 就醫類別 M07 常用代碼

| M07 | 說明 | 就醫序號 |
|-----|------|----------|
| `01` | 西醫門診 | 需取號 |
| `02` | 牙醫門診 | 需取號 |
| `03` | 中醫門診 | 需取號 |
| `04` | 急診 | 需取號 |
| `05` | 住院 | 需取號 |
| `AC` | 預防保健 | ICxx 格式 |
| `AD` | 職災門(急)診 | 不取號 |
| `AE` | 慢連箋領藥 | 不取號 |
| `AF` | 藥局調劑 | 不取號 |

---

## IC 卡四大區段概覽

| 區段 | 欄位數 | 主要內容 |
|------|--------|---------|
| 基本資料 | 8 | 卡號、姓名、身分證、生日、性別 |
| 健保資料段 | 67 | 投保、重大傷病、就醫紀錄(6次)、預防保健註記 |
| 醫療專區 | 26 | 處方紀錄、過敏藥物(3筆) |
| 衛生行政專區 | 17 | 疫苗接種紀錄(6筆) |

> BhpNhi.dll 的 `GetPersonData()` 只讀 6 個欄位（HospId, SamId, CardId, Pid, Birth, RegisterPrevent）。完整卡片資料需用 CsHis.dll。

---

## 篩檢資格規則

> **權威來源**：`references/screening-workflow.md` 是篩檢規則的唯一維護點。以下為摘要，如有出入以該文件為準。

### 成人預防保健

| 對象 | 年齡 | 頻率 |
|------|------|------|
| 一般身分 | 40-64 歲 | 每 3 年 1 次 |
| 一般身分 | 65 歲以上 | 每年 1 次 |
| 原住民 | 55 歲以上 | 每年 1 次 |

### BC肝篩檢

| 對象 | 年齡 | 頻率 |
|------|------|------|
| 一般身分 | 45-79 歲 | **終身 1 次** |
| 原住民 | 40-79 歲 | **終身 1 次** |

---

## 詳細參考文件（按需載入）

以下 references 檔案包含完整細節，Claude 會依據問題自動讀取相關檔案：

| 檔案 | 內容 | 適用對象 |
|------|------|----------|
| [`references/api-development.md`](references/api-development.md) | BhpNhi.dll API 開發、資料模型、回應碼完整表、程式碼範本、疑難排解 Decision Tree | 開發者 |
| [`references/screening-workflow.md`](references/screening-workflow.md) | 成健/BC肝篩檢工作流程、白話代碼對照、登記取消流程、展望HIS整合、FAQ | 診所人員 + 開發者 |
| [`references/card-fields.md`](references/card-fields.md) | IC卡 822+ 欄位結構、四大區段定義、RegisterPrevent 解析、日期格式轉換 | 開發者 |
| [`references/upload-format.md`](references/upload-format.md) | 上傳格式 2.0 XML 結構、MSH/MB1/MB2 欄位規則、異常上傳、補卡、急診住院規則 | 開發者 + 診所人員 |

### 路由指引

- **「怎麼呼叫 API」「程式碼範例」「DLL 載入失敗」** → 讀取 `references/api-development.md`
- **「這個代碼什麼意思」「病人符不符合資格」「登記流程」** → 讀取 `references/screening-workflow.md`
- **「IC卡存了什麼」「欄位格式」「RegisterPrevent 怎麼解」** → 讀取 `references/card-fields.md`
- **「上傳 XML 怎麼寫」「異常上傳」「補卡」「就醫類別」** → 讀取 `references/upload-format.md`

---

## 最常見的整合模式

### 模式 1：門診掛號流程中自動讀卡+查資格

```csharp
// 病患插卡 → 讀卡 → 查成健/BC肝 → 顯示結果
public async Task<ScreeningResult> OnCardInserted()
{
    var bhp = new BhpNhi.Bhp();

    // Step 1: 讀卡（必須先做）
    var personJson = bhp.GetPersonData();
    var person = JsonSerializer.Deserialize<PersonResponse>(personJson);
    if (!person.Status) return ScreeningResult.Error(person.Code);

    // Step 2: 查資格
    var validJson = bhp.ValidAll();
    var valid = JsonSerializer.Deserialize<ApiResponse>(validJson);
    if (!valid.Status) return ScreeningResult.Error(valid.Code);

    // Step 3: 解碼 XXYY-ZZ
    return ParseScreeningCode(valid.Code, person.Person);
}

private ScreeningResult ParseScreeningCode(string code, Person patient)
{
    // code = "0101-00"
    var parts = code.Split('-');
    var xxYY = parts[0]; // "0101"
    var zz = parts[1];   // "00"

    return new ScreeningResult
    {
        BcLiver = xxYY[..2] == "01",     // BC肝符合
        Adult = xxYY[2..] == "01",        // 成健符合
        IsAboriginal = zz == "01",        // 原住民
        PatientName = patient.Pid         // 身分證
    };
}
```

### 模式 2：批次查詢（多位病患連續讀卡）

```csharp
// 重點：每次換卡都要重新 GetPersonData()
foreach (var patient in waitingList)
{
    Console.WriteLine($"請插入 {patient.Name} 的健保卡...");
    WaitForCardInsert(); // 等待讀卡機偵測到卡片

    var result = await OnCardInserted();
    results.Add(result);

    Console.WriteLine($"請取出健保卡");
    WaitForCardRemove(); // 等待卡片移除
}
```

---

## 共通注意事項

1. **GetPersonData() 必須最先呼叫** — 它建立 Token 通道，未先呼叫會得到 9999 錯誤。

2. **Status=true 不代表符合資格** — `ApiResponse.Status` 只表示 API 呼叫成功，實際資格看 `Code` 欄位解碼。

3. **BC肝篩檢終身一次** — 不同於成健可定期做，BC肝是終身只有一次。登記(Register)≠執行，確認執行後才登記。

4. **XML 編碼是 Big5** — 上傳 XML 宣告必須用 `encoding="Big5"`，否則退件。

5. **範例程式按鈕事件未綁定** — 官方範例 Designer.cs 中只有 `btn_getConsoleStatus` 有綁定，其餘 7 個按鈕需手動補上。

6. **DLL 參考路徑不一致** — .csproj 寫的路徑與實際不同，建置前需確認。下載的 DLL 可能被 Windows 封鎖，需右鍵解除。

---

## 自動化測試策略

在無實體讀卡機的環境下（如 CI/CD、開發機），仍需確保程式邏輯正確。以下為建議的測試策略。

### Mock 讀卡機模式

建立 `IBhpNhi` 介面，將 `BhpNhi.Bhp` 的方法抽象化，測試時注入 Mock 實作：

```csharp
// 定義介面
public interface IBhpNhi
{
    string GetPersonData();
    string ValidAdult();
    string ValidBcLiver();
    string ValidAll();
    string RegisterBcLiver();
    string CancelBcLiver();
    string GetVersion();
    string GetStatus();
}

// 正式環境：包裝真實 DLL
public class BhpNhiWrapper : IBhpNhi
{
    private readonly BhpNhi.Bhp _bhp = new();
    public string GetPersonData() => _bhp.GetPersonData();
    public string ValidAll() => _bhp.ValidAll();
    // ... 其餘方法同理
}

// 測試環境：回傳預設 JSON
public class MockBhpNhi : IBhpNhi
{
    public string ScenarioCode { get; set; } = "0101-00";

    public string GetPersonData() => JsonSerializer.Serialize(new {
        Status = true,
        Person = new {
            HospId = "0000000000",
            SamId = "000000000000",
            CardId = "000000000000",
            Pid = "A123456789",
            Birth = "0761015",
            RegisterPrevent = "000000000000"
        }
    });

    public string ValidAll() => JsonSerializer.Serialize(new {
        Status = true,
        Code = ScenarioCode,
        Message = "Mock 回應"
    });

    // ... 其餘方法依需求設計回傳值
}
```

在 DI 容器中依環境切換：

```csharp
if (Environment.GetEnvironmentVariable("USE_MOCK_CARD") == "true")
    services.AddSingleton<IBhpNhi, MockBhpNhi>();
else
    services.AddSingleton<IBhpNhi, BhpNhiWrapper>();
```

### Integration Test 模式

使用模擬的 XXYY-ZZ 回應碼，測試所有程式分支：

```csharp
[Theory]
[InlineData("0101-00", true, true)]    // 成健+BC肝皆符合（一般身分）
[InlineData("0101-01", true, true)]    // 成健+BC肝皆符合（原住民）
[InlineData("0303-00", false, false)]  // 兩者皆已做過
[InlineData("0201-00", false, true)]   // 成健年齡不符，BC肝符合
[InlineData("0102-00", true, false)]   // 成健符合，BC肝年齡不符
[InlineData("0003-00", false, false)]  // 成健未查，BC肝已做過
public void ValidAll_ShouldParseCorrectly(string code, bool adultEligible, bool bcLiverEligible)
{
    var mock = new MockBhpNhi { ScenarioCode = code };
    var service = new ScreeningService(mock);

    var result = service.CheckEligibility();

    Assert.Equal(adultEligible, result.AdultEligible);
    Assert.Equal(bcLiverEligible, result.BcLiverEligible);
}
```

### 錯誤情境對照表

| 錯誤碼 | 情境描述 | 測試方式 | 預期行為 |
|--------|---------|---------|---------|
| `9999` | Token 驗證失敗（未先讀卡/網路不通） | Mock `GetPersonData()` 回傳 `Status=false, Code="9999"` | 顯示「請先插入健保卡並讀卡」提示，禁用後續操作 |
| `9998` | 主控台/SAM卡/卡片驗證失敗 | Mock 回傳 `Code="9998"` | 顯示「請確認讀卡機已連接、SAM卡已插入」，導引至設定頁 |
| `9001` | 資料格式檢核不正確 | 傳入格式異常的 RegisterPrevent 字串 | 記錄錯誤日誌，顯示「資料格式異常，請聯繫系統管理員」 |
| `9002` | 健保卡註記含空白或特殊字元 | Mock 回傳含特殊字元的 RegisterPrevent | 自動 Trim 處理，或提示「卡片資料異常」 |

### CI/CD 考量

在無實體讀卡機的 CI/CD 環境中執行測試的方式：

1. **環境變數切換**：設定 `USE_MOCK_CARD=true`，所有 DLL 呼叫自動導向 Mock 實作
2. **Mock DLL 方式**：將 `IBhpNhi` 介面的 Mock 實作編譯為獨立 DLL，CI 環境不需安裝讀卡機驅動
3. **HTTP Proxy 方式**：將讀卡功能包裝為 HTTP API（見下節），CI 環境對測試用的 Mock Server 發送請求
4. **測試分層**：
   - **Unit Tests**（CI 必跑）：使用 Mock，測試業務邏輯與代碼解析
   - **Integration Tests**（手動觸發）：需實體讀卡機，在開發機或測試機執行
   - 在 CI pipeline 中以 `[Trait("Category", "Integration")]` 標記，預設排除

---

## 跨平台整合指引

BhpNhi.dll 為 .NET 元件，其他語言需透過中介方式存取。**建議架構：將 DLL 包裝成 REST API**（ASP.NET Minimal API，`localhost:5100`），其他語言透過 HTTP 呼叫。

| 語言 | 推薦方式 | 備選方式 |
|------|---------|---------|
| Node.js | HTTP (axios) | edge-js、child_process |
| Python | HTTP (requests) | pythonnet、subprocess |
| Docker | 讀卡服務在 Host 跑 REST API | 不建議直接容器化（USB 限制） |

> 完整程式碼範例（ASP.NET Minimal API、Node.js 三種方式、Python 三種方式、Docker 架構）→ `references/cross-platform-integration.md`

---

## 首次設定檢查清單

初次建置健保卡讀卡環境時，請依序完成以下步驟：

### 1. 安裝健保卡控制軟體 6.0

- [ ] 從健保署網站下載「健保卡控制軟體 6.0」安裝程式
- [ ] 以系統管理員身分執行安裝
- [ ] 安裝完成後確認桌面或開始功能表出現「健保卡控制軟體主控台」

### 2. 連接讀卡機硬體

- [ ] 將 IC 卡讀卡機連接至電腦 USB 埠
- [ ] 確認 Windows 裝置管理員中出現智慧卡讀卡機裝置（無驚嘆號）
- [ ] 若驅動未自動安裝，手動安裝讀卡機廠商提供的驅動程式

### 3. 插入 SAM 卡

- [ ] 將健保署核發的 SAM 卡插入讀卡機的 SAM 卡槽（通常在讀卡機背面或側面）
- [ ] SAM 卡晶片面朝上或依讀卡機說明書方向插入
- [ ] 確認 SAM 卡已到底、接觸良好

### 4. 啟動主控台並驗證狀態

- [ ] 開啟「健保卡控制軟體主控台」
- [ ] 確認主控台顯示「就緒」或綠燈狀態
- [ ] 在程式中呼叫 `GetStatus()` 確認回傳正常

```csharp
var bhp = new BhpNhi.Bhp();
var status = bhp.GetStatus();
Console.WriteLine($"主控台狀態: {status}");
// 預期回傳：表示就緒的狀態字串
```

### 5. 測試讀取健保卡

- [ ] 插入一張有效的健保卡
- [ ] 呼叫 `GetPersonData()` 讀取卡片資料

```csharp
var personJson = bhp.GetPersonData();
Console.WriteLine(personJson);
// 預期回傳：包含 Status=true 的 JSON，含 Person 物件
```

- [ ] 確認回傳的 JSON 中 `Status` 為 `true`
- [ ] 確認 `Person.Pid`（身分證字號）與卡片持有人一致

### 6. 常見首次設定問題

| 症狀 | 可能原因 | 解決方式 |
|------|---------|---------|
| `GetStatus()` 無回應或例外 | 主控台未啟動 | 啟動健保卡控制軟體主控台 |
| `GetPersonData()` 回傳 9998 | SAM 卡未插入或接觸不良 | 重新插入 SAM 卡，確認方向正確 |
| `GetPersonData()` 回傳 9999 | 網路不通或 Token 服務異常 | 檢查網路連線，確認可連至健保署伺服器 |
| DLL 載入失敗 `DllNotFoundException` | DLL 路徑不正確或被 Windows 封鎖 | 確認 .csproj 的 DLL 參考路徑，右鍵 DLL 檔案→內容→解除封鎖 |
| `BadImageFormatException` | 32/64 位元不符 | 確認專案的目標平台與 DLL 位元數一致（通常為 x86） |
| 讀卡機在裝置管理員有驚嘆號 | 驅動程式未安裝或版本不符 | 更新讀卡機驅動程式 |
| 主控台顯示「SAM 卡驗證失敗」 | SAM 卡過期或損壞 | 聯繫健保署更換 SAM 卡 |

---

## 虛擬健保卡整合指南

### 概述

虛擬健保卡是健保署推出的數位憑證方案，透過 QR Code 或雲端 API 驗證身份，無需實體 IC 卡。

### 整合方式比較

| 方式 | 流程 | 適用場景 | 需要讀卡機 |
|------|------|---------|-----------|
| QR Code 掃描 | 病患出示 App QR Code → 診所掃描 → 雲端驗證 | 門診現場 | 否（需掃碼設備） |
| Cloud API | 系統直接呼叫雲端 API 驗證 | 遠距醫療、線上預約 | 否 |
| 實體卡 + 虛擬卡並行 | 同時支援兩種方式 | 過渡期診所 | 是（實體卡用） |

### Cloud API 認證流程（OAuth 2.0）

```
1. 診所系統向健保署 OAuth Server 取得 Access Token
   POST /oauth/token
   grant_type=client_credentials
   client_id={診所代碼}
   client_secret={密鑰}

2. 使用 Access Token 呼叫虛擬健保卡 API
   GET /api/v1/virtual-card/verify
   Authorization: Bearer {access_token}
   X-Patient-QRCode: {qr_code_content}

3. 回傳病患基本資料（與實體卡 GetPersonData 相同結構）
```

### QR Code 整合

```csharp
// 掃描 QR Code 後的驗證流程
public async Task<PatientInfo> VerifyVirtualCard(string qrCodeContent)
{
    // 1. 確認 Token 有效
    await EnsureValidToken();

    // 2. 呼叫驗證 API
    var response = await _httpClient.GetAsync(
        $"{_baseUrl}/api/v1/virtual-card/verify?qr={Uri.EscapeDataString(qrCodeContent)}");

    // 3. 處理回傳
    if (!response.IsSuccessStatusCode)
    {
        var error = await response.Content.ReadAsStringAsync();
        throw new VirtualCardException($"驗證失敗: {response.StatusCode} - {error}");
    }

    return await response.Content.ReadFromJsonAsync<PatientInfo>();
}
```

### 虛擬卡錯誤代碼

| 代碼 | 說明 | 處理方式 |
|------|------|---------|
| VC001 | QR Code 已過期 | 請病患重新產生 QR Code |
| VC002 | QR Code 格式錯誤 | 檢查掃描品質，重新掃描 |
| VC003 | 虛擬卡未啟用 | 請病患至健保快易通 App 啟用 |
| VC004 | Token 過期 | 重新取得 Access Token |
| VC005 | 診所未授權 | 向健保署申請虛擬卡服務 |

---

## 多讀卡機支援

### 場景

診所有多個診間，每個診間配備一台讀卡機，系統需同時管理多台讀卡機。

### 實作方式

| 方式 | 適用場景 | 說明 |
|------|---------|------|
| 多實例 | 每診間獨立系統 | 每個程序連接各自的讀卡機，最簡單 |
| COM Port 路由 | 單一系統多讀卡機 | 依 COM Port 編號區分讀卡機 |
| USB HID 識別 | 多台同型讀卡機 | 用 USB 裝置路徑區分 |

### COM Port 管理

```csharp
// 多讀卡機管理器
public class MultiReaderManager
{
    private readonly Dictionary<string, BhpNhiClient> _readers = new();

    // 註冊讀卡機（診間名稱 → COM Port）
    public void RegisterReader(string roomName, string comPort)
    {
        var client = new BhpNhiClient(comPort);
        _readers[roomName] = client;
    }

    // 指定診間讀取
    public string ReadCard(string roomName)
    {
        if (!_readers.TryGetValue(roomName, out var client))
            throw new InvalidOperationException($"診間 {roomName} 無已註冊的讀卡機");
        return client.GetPersonData();
    }

    // 列出所有可用讀卡機
    public IEnumerable<string> ListAvailableReaders()
        => _readers.Where(r => r.Value.GetStatus().Contains("ready"))
                   .Select(r => r.Key);
}
```

---

## 問題排查決策樹

當讀卡或查詢出問題時，按以下流程逐步排除：

```
GetPersonData() 回傳什麼？
  │
  ├── 例外（Exception）
  │     ├── DllNotFoundException → DLL 路徑不對或被 Windows 封鎖
  │     │     → 確認 .csproj 參考路徑、右鍵 DLL → 解除封鎖
  │     ├── BadImageFormatException → 32/64 位元不符
  │     │     → 專案改為 x86 或確認 DLL 位元數
  │     └── 其他例外 → 主控台可能沒啟動
  │           → 確認桌面有開啟「健保卡控制軟體主控台」
  │
  ├── Status = false
  │     ├── Code = "9999" → Token 失敗
  │     │     ├── 是否第一次呼叫？ → 可能網路不通（健保署伺服器）
  │     │     └── 之前成功過？ → Session 逾時，重啟主控台
  │     ├── Code = "9998" → 硬體驗證失敗
  │     │     ├── SAM 卡有插好？ → 重插 SAM 卡
  │     │     ├── 健保卡有插？ → 插卡後再試
  │     │     └── 讀卡機在裝置管理員有驚嘆號？ → 重裝驅動
  │     └── Code = "9001"/"9002" → 資料格式問題
  │           → 健保卡可能有損壞，換張卡試試
  │
  └── Status = true → 讀卡成功！
        │
        接著 ValidAll() 回傳什麼？
        ├── Status = true, Code = "XXYY-ZZ" → 正常，解碼 Code
        └── Status = false → Token 可能在中間過期
              → 重新呼叫 GetPersonData() 後再試
```

**最常見的 3 個坑（依發生頻率排序）：**

1. **主控台沒開**：每次開機都要手動啟動，或設為開機自啟
2. **DLL 被 Windows 封鎖**：從網路下載的 DLL 預設被封鎖，右鍵 → 內容 → 勾選「解除封鎖」
3. **GetPersonData 沒先呼叫就直接 ValidAll**：所有查詢操作都依賴 GetPersonData 建立的 Token 通道

---

## 邊界與失敗模式

**這個技能誠實的邊界：**

| 情況 | 為什麼失敗 | 怎麼辨認 |
|------|-----------|---------|
| DLL 路徑或版本不對 | BhpNhi.dll 依賴特定版本的 CsHis.dll + 作業系統版本 | `DllNotFoundException` 或 `BadImageFormatException` |
| 硬體環境缺少 | 無讀卡機、SAM 卡未插入、控制軟體未啟動 | 9998 或 9999 錯誤碼 |
| 健保署後端服務中斷 | API 呼叫依賴健保署伺服器 | `Status=false` 但本機硬體正常 |
| 同時處理多病患 | BhpNhi.dll 設計是單次單卡，無非同步 API | 序列化排隊讀卡，不能平行化 |
| 在 CI/CD 環境執行 | 需要實體硬體 | 必須切換 Mock 模式 |

**這個技能不能告訴你的事：**
- 具體的網路防火牆規則（各院所環境不同）
- SAM 卡到期日（需聯繫健保署）
- 健保署 API 目前的服務狀態（需要查健保署公告）

## 成功的樣子

**本機環境就緒確認流程（照順序做）：**

1. `GetStatus()` 回傳正常狀態字串 ✓
2. 插入有效健保卡後，`GetPersonData()` 回傳 `Status=true` ✓
3. `ValidAll()` 回傳 `XXYY-ZZ` 格式且可正確解碼 ✓
4. Mock 測試：`0101-00` 解碼為「成健符合＋BC肝符合＋一般身分」✓

每個步驟成功才往下走。任何步驟失敗，先找根本原因再繼續。

---

## Token 生命週期表

### 各類 Token 的有效期

| Token 類型 | 有效期 | 刷新方式 | 用途 |
|-----------|--------|---------|------|
| SAM 卡 Token | 依卡片設定（通常 1 年） | 更換 SAM 卡 | 讀卡機身份驗證 |
| Session Token (主控台) | 啟動時產生，關閉時失效 | 重啟主控台 | 本機 API 呼叫 |
| OAuth Access Token (虛擬卡) | 3600 秒（1 小時） | 用 refresh_token 或重新認證 | 雲端 API 呼叫 |
| OAuth Refresh Token | 7 天 | 重新登入 | 延長 Access Token |
| QR Code (虛擬卡) | 5 分鐘 | 病患重新產生 | 單次驗證 |

### Token 監控建議

```
Token 監控流程：
├── 啟動時：檢查 SAM 卡有效期，距過期 30 天內發出警告
├── 每 50 分鐘：自動刷新 OAuth Access Token（在到期前 10 分鐘）
├── 每次呼叫：檢查回傳的 HTTP Status
│   ├── 401 → Token 過期，立即刷新
│   └── 403 → 權限問題，通知管理者
└── 每日：記錄 Token 使用統計（成功/失敗次數）
```

---

## 相關插件

本插件與以下插件形成完整的臨床工作流：

| 工作流步驟 | 插件 | 說明 |
|-----------|------|------|
| 1. 讀卡取得身份 | **本插件（nhi-ic-card）** | GetPersonData → ValidAll |
| 2. 查詢國健署篩檢資格 | **nhs-pportal-screening** | 用身分證號到 pportal 查全項目篩檢資格 |
| 3. 查詢 HIS 歷史紀錄 | **vision-database** | 用 KCSTMR 查詢就診、處方、檢驗歷史 |
| 4. 計算健康風險 | **nhri-risk-plugin** | 用檢驗數值計算 10 年心血管風險 |
