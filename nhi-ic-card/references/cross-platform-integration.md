# 跨平台整合指引

BhpNhi.dll 為 .NET Framework / .NET 元件，其他語言需透過中介方式存取。以下為常見整合方案。

## REST API 包裝（最推薦）

將 DLL 功能包裝成 HTTP API，是最通用的跨語言方案。

**ASP.NET Minimal API 範例**（.NET 6+）：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IBhpNhi, BhpNhiWrapper>();

var app = builder.Build();

app.MapGet("/api/card/status", (IBhpNhi bhp) =>
    Results.Ok(new { status = bhp.GetStatus() }));

app.MapPost("/api/card/read", (IBhpNhi bhp) =>
{
    var json = bhp.GetPersonData();
    return Results.Content(json, "application/json");
});

app.MapPost("/api/screening/check", (IBhpNhi bhp) =>
{
    var json = bhp.ValidAll();
    return Results.Content(json, "application/json");
});

app.MapPost("/api/screening/register-bcliver", (IBhpNhi bhp) =>
{
    var json = bhp.RegisterBcLiver();
    return Results.Content(json, "application/json");
});

app.MapPost("/api/screening/cancel-bcliver", (IBhpNhi bhp) =>
{
    var json = bhp.CancelBcLiver();
    return Results.Content(json, "application/json");
});

app.Run("http://localhost:5100");
```

部署時建議：
- 僅綁定 `localhost` 或內網 IP，避免暴露於外網
- 加入 API Key 驗證中介軟體
- 記錄所有讀卡操作的稽核日誌

## Node.js 整合

**方式一：HTTP 呼叫**（推薦，搭配上述 REST API）：

```javascript
const axios = require('axios');
const CARD_API = 'http://localhost:5100';

async function readCard() {
    const { data } = await axios.post(`${CARD_API}/api/card/read`);
    return data;
}

async function checkScreening() {
    const { data } = await axios.post(`${CARD_API}/api/screening/check`);
    return data;
}
```

**方式二：edge-js 直接呼叫 .NET**（需 Node.js 與 .NET 同機）：

```javascript
const edge = require('edge-js');

const getPersonData = edge.func({
    assemblyFile: 'path/to/BhpNhiWrapper.dll',
    typeName: 'BhpNhiWrapper.Bridge',
    methodName: 'GetPersonData'
});

getPersonData(null, (error, result) => {
    if (error) throw error;
    console.log(JSON.parse(result));
});
```

**方式三：child_process 呼叫 CLI**：

```javascript
const { execSync } = require('child_process');
const result = execSync('dotnet run --project CardReaderCli -- read-card', { encoding: 'utf-8' });
const data = JSON.parse(result);
```

## Python 整合

**方式一：HTTP 呼叫**（推薦）：

```python
import requests

CARD_API = "http://localhost:5100"

def read_card():
    resp = requests.post(f"{CARD_API}/api/card/read")
    return resp.json()

def check_screening():
    resp = requests.post(f"{CARD_API}/api/screening/check")
    return resp.json()
```

**方式二：pythonnet 直接載入 .NET DLL**：

```python
import clr
clr.AddReference(r"C:\path\to\BhpNhi.dll")
from BhpNhi import Bhp

bhp = Bhp()
person_json = bhp.GetPersonData()
```

> **注意**：pythonnet 需安裝對應的 .NET Runtime，且 32/64 位元需與 DLL 一致。

**方式三：subprocess 呼叫 CLI**：

```python
import subprocess, json

result = subprocess.run(
    ["dotnet", "run", "--project", "CardReaderCli", "--", "read-card"],
    capture_output=True, text=True
)
data = json.loads(result.stdout)
```

## Docker 容器化考量

由於讀卡機為 USB 硬體裝置，Docker 容器化有以下限制與注意事項：

1. **USB 裝置傳遞**：需使用 `--device` 參數將讀卡機裝置傳入容器
   ```bash
   docker run --device=/dev/bus/usb/001/003 my-card-reader-app
   ```

2. **Windows 容器**：在 Windows 環境建議使用 Windows Container 或直接在 Host 執行讀卡服務，Linux 容器無法存取 Windows 的 USB 裝置

3. **建議架構**：將系統拆為兩層
   - **讀卡服務**：直接在有讀卡機的 Windows 主機上執行（REST API 模式）
   - **業務邏輯**：容器化部署，透過 HTTP 呼叫讀卡服務

4. **健保卡控制軟體**：健保署的控制軟體 6.0 需安裝在有讀卡機的主機上，無法容器化
