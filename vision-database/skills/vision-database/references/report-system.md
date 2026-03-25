# 展望報表系統參考

> 來源：VISW 系統 INT/PRG 報表程式分析（11 個檔案 × SQL 交叉驗證）
> 本文件涵蓋報表框架、模組目錄、自訂函式、SQL 模式。

---

## 目錄

1. [報表框架（F_GTFM）](#報表框架fgtfm)
2. [報表模組目錄](#報表模組目錄)
3. [自訂函式參考](#自訂函式參考)
4. [SQL 查詢模式](#sql-查詢模式)
5. [VFP 語法速查](#vfp-語法速查)

---

## 報表框架（F_GTFM）

所有 .INT 報表模組共用相同的執行框架，由三個核心函式驅動：

### 執行流程

```
使用者操作 → F_GTFM (條件輸入介面)
           → F_GETVAR() (解析條件)
           → SQL SELECT (查詢資料)
           → F_OPT() (輸出結果)
```

### F_GTFM — 條件輸入格式字串

F_GTFM 是一個格式字串，定義報表的條件輸入介面（日期範圍、選項等）。

**常見格式碼：**

| 格式碼 | 說明 | 範例 |
|--------|------|------|
| `D` | 日期輸入（民國年 YYYMMDD） | 起迄日期 |
| `T` | 文字輸入 | 搜尋關鍵字 |
| `C` | 勾選框 | 是否包含停用品項 |
| `L` | 下拉選單 | 藥品分類篩選 |
| `R` | 單選鈕 | 輸出格式選擇 |

### F_GETVAR() — 解析使用者輸入

將 F_GTFM 介面收集的使用者輸入解析為查詢變數：

| 變數 | 說明 | 格式 |
|------|------|------|
| `F_IBEG` | 起始日期 | `YYYMMDD`（民國年 7 碼）|
| `F_IEND` | 結束日期 | `YYYMMDD` |
| `F_IOPT` | 輸出方式 | 1=列印, 2=檔案, 8=預覽, 9=Excel |
| `SYS_DATE` | 系統日期 | 當天民國年日期 |

### F_OPT() — 輸出控制

| F_IOPT 值 | 輸出方式 | 說明 |
|-----------|---------|------|
| `1` | 列印 | 直接印表機輸出 |
| `2` | 檔案 | 輸出到檔案 |
| `8` | 預覽 | 螢幕預覽 |
| `9` | Excel | 匯出 Excel |

### CREA_COM() — COM 物件建立

用於建立 `TXBCLM0` COM 物件（健保日期選擇器等 ActiveX 控件）。

### TXBCLM0 COM 物件

部分報表使用 TXBCLM0 COM 物件提供進階的日期選擇介面（健保申報期間選擇器）。

---

## 報表模組目錄

### VST 系列 — 統計報表

| 模組 | 說明 |
|------|------|
| VST_010 | 藥品銷售統計 |
| VST_020 | 高費用就診統計 |
| VST_030 | 人員別看診統計 |
| VST_040 | 診斷碼統計 |
| VST_050 | 時段別就診統計 |
| VST_060 | 收入統計 |
| VST_070 | 病患來源統計 |
| VST_080 | 處方統計 |
| VST_090 | ICD 版本自動切換統計 |
| VST_100 | 綜合統計報表 |

### VS4 系列 — 診斷碼統計

| 模組 | 說明 |
|------|------|
| VS4_xxx | Top 50 診斷碼統計（依各種維度排序）|

### VISI 系列 — 藥品進銷存

| 模組 | 說明 |
|------|------|
| VISI12D0 | 藥品進貨明細 |
| VISI12E0 | 藥品銷售明細 |
| VISI12F0 | 藥品庫存明細 |
| VISI1540 | 進銷存彙總 |

### WHR 系列 — 查詢報表

| 模組 | 說明 |
|------|------|
| R1640 | 病患查詢 |
| R1650 | 藥品查詢 |
| R1660 | 組合查詢（病患+藥品+日期）|
| R11A0 | 錯誤處理 |
| R11B0 | 診斷碼轉換（ICD-9 ↔ ICD-10）|

### INV 系列 — 庫存報表

| 函式 | 檔案 | 說明 |
|------|------|------|
| P_INV_21 | INV21P0.INP | 存貨報表 / 月異動狀況表 |
| P_INV_22 | INV22P0.INP | 各類單據異動明細表 |
| P_INV_26 | INV26P0.INP | 產品別耗用統計表 / 排行表 |
| P_INV_27 | INV27P0.INP | 產品別耗用明細表 |
| P_INV_28 | INV28P0.INP | 醫師別耗用統計表 / 排行表 |
| P_INV_29 | INV29P0.INP | 安全存量明細表 |
| P_INV_2A | INV2AP0.INP | 醫療費用日報表 |
| P_INV_2B | INV2BP0.INP | 產品別未耗用統計表 |
| P_INV_2C | INV2CP0.INP | **產品別月異動統計表** |
| P_INV_43 | INV43P0.INP | 盤點明細表 |
| P_INV_44 | INV44P0.INP | 盤盈虧明細表 |
| P_INV_46RPT | INV461P0.INP | **存貨月報表**（標準格式）|
| P_INV_46RPT | INV462P0.INP | **存貨月報表**（基隆礦工醫院專用）|

### SCX 表單

- 100+ 表單檔案，命名慣例 `R[類別][子類別][編號].SCX`
- 每個 SCX 對應一個 FRX 報表版面和 MEM 設定檔

---

## 自訂函式參考

### 框架內建函式

| 函式 | 說明 | 範例 |
|------|------|------|
| `F_GETVAR()` | 解析 F_GTFM 輸入 | 回傳查詢條件變數 |
| `F_OPT()` | 輸出控制 | 根據 F_IOPT 決定輸出方式 |
| `CREA_COM()` | 建立 COM 物件 | ActiveX 控件初始化 |
| `SHOW_DT()` | 顯示日期 | 民國年日期格式化顯示 |
| `ADD_DT()` | 日期加減 | 計算 N 天/月前後的日期 |
| `LEFTX()` | **安全截斷** | 避免在 Big5 中文字元中間截斷 |
| `UNION_CHECK()` | 合併檢查 | 聯集查詢的前置檢查 |
| `INDEX_SEEK()` | 索引查找 | 帶索引的快速查找 |
| `MS()` | 訊息顯示 | 彈出訊息對話框 |
| `T_LM()` | 午別判斷 | 回傳 A=上午/B=下午/C=晚上 |

### 報表自訂函式

| 函式 | 來源 | 說明 |
|------|------|------|
| R11B0.PRG 診斷碼分割 | WHR 系列 | 將複合 ICD 碼拆分為個別診斷碼 |
| VST_090 ICD 版本切換 | VST 系列 | 自動判斷 ICD-9 或 ICD-10 並轉換 |

---

## SQL 查詢模式

### 日期範圍查詢（標準模式）

```sql
-- 所有報表通用的日期+時間範圍篩選
WHERE DATE + TIME >= F_IBEG
  AND DATE + TIME <= F_IEND + "ZZZZZZ"
```

- `"ZZZZZZ"` 是時間哨兵值，代表 23:59:59（ASCII 排序最大值）
- `DATE + TIME` 是字串串接，不是數值相加

### JOIN 模式

**病患+就診（三表 JOIN，含日期+時間串接）：**
```sql
SELECT CO01M.*, CO03L.*, CO02M.*
FROM CO03L
JOIN CO01M ON CO03L.KCSTMR = CO01M.KCSTMR
LEFT JOIN CO02M ON CO03L.KCSTMR = CO02M.KCSTMR
  AND CO03L.DATE = CO02M.IDATE
  AND CO03L.TIME = CO02M.ITIME
WHERE CO03L.DATE + CO03L.TIME >= F_IBEG
  AND CO03L.DATE + CO03L.TIME <= F_IEND + "ZZZZZZ"
```

**進貨（四表 JOIN）：**
```sql
SELECT CO10A.*, CO09D.DDESC, CO10P.PTITL, CO14M.MNAME
FROM CO10A
JOIN CO09D ON CO10A.KDRUG = CO09D.KDRUG
JOIN CO10P ON CO10A.ATYPE = CO10P.PTYP1
LEFT JOIN CO14M ON CO10A.MTYPE = CO14M.MTYPE
  AND CO10A.MNO = CO14M.MNO
```

**庫存（三表 JOIN）：**
```sql
SELECT CO09D.*, CO09S.DQTY, CO14M.MNAME
FROM CO09D
JOIN CO09S ON CO09D.KDRUG = CO09S.KDRUG
LEFT JOIN CO14M ON CO09D.DLVENDOR = CO14M.MNO
```

### 聚合模式

**TOP N 排行：**
```sql
SELECT TOP 50 LABNO, COUNT(*) AS CNT
FROM CO03L
WHERE DATE BETWEEN F_IBEG AND F_IEND
GROUP BY LABNO
ORDER BY CNT DESC
```

**月度統計（二層聚合去重）：**
```sql
-- 第一層：每病患每月只算一次
SELECT SUBSTR(DATE,1,5) AS YM, KCSTMR
FROM CO03L
GROUP BY YM, KCSTMR

-- 第二層：統計不重複人數
SELECT YM, COUNT(*) AS PATIENT_CNT
FROM (第一層)
GROUP BY YM
```

### 動態條件（DO CASE）

```
DO CASE
  CASE 搜尋類型 = "1"  && 依病歷內容
    cWHERE = "STEXT LIKE '%" + 關鍵字 + "%'"
  CASE 搜尋類型 = "2"  && 依藥品
    cWHERE = "DNO = '" + 藥品碼 + "'"
  CASE 搜尋類型 = "3"  && 依診斷碼
    cWHERE = "LABNO LIKE '" + 診斷碼 + "%'"
ENDCASE
```

### CURSOR 操作

```
-- 建立暫存表
CREATE CURSOR AA (KDRUG C(6), BQTY N(15,3), EQTY N(15,3))

-- 寫入資料
INSERT INTO AA (KDRUG, BQTY) VALUES (cKDRUG, nQTY)

-- 掃描處理
SCAN
  IF AA.EQTY <> 0
    -- 處理邏輯
  ENDIF
ENDSCAN
```

> **CURSOR vs TABLE**：CURSOR 存在記憶體，關閉後消失；TABLE 寫入磁碟，需手動刪除。

### $ 包含運算子

```sql
-- VFP 的 $ 運算子等同 SQL 的 LIKE '%...%'
WHERE ATYPE $ C__I_STO_ATYPE
-- 等同
WHERE ATYPE IN ('2', 'I')
-- 或
WHERE C__I_STO_ATYPE LIKE '%' + ATYPE + '%'
```

### 哨兵值

| 值 | 用途 | 說明 |
|----|------|------|
| `"ZZZZZZ"` | 時間上界 | 串接到日期後代表 23:59:59 |
| `"!"` | 空碼篩選 | ASCII 33，用來過濾空白代碼（`LABNO > "!"` 排除空診斷碼）|
| `"@"` | 作廢標記 | CO09S.DIO = `"@"` 表示作廢排除 |
| `"*"` | 停用標記 | CO09D.DSTS = `"*"` 表示停用 |

---

## VFP 語法速查

### 字串函式

| VFP 函式 | 說明 | SQLite 等效 |
|----------|------|------------|
| `ALLTRIM(x)` | 去頭尾空白 | `TRIM(x)` |
| `LEFT(x, n)` | 取左 n 字元 | `SUBSTR(x, 1, n)` |
| `LEFTX(x, n)` | **安全取左**（不截斷 Big5）| 無直接等效 |
| `SUBSTR(x, p, n)` | 子字串 | `SUBSTR(x, p, n)` |
| `STRTRAN(x, a, b)` | 取代 | `REPLACE(x, a, b)` |
| `UPPER(x)` | 大寫 | `UPPER(x)` |
| `SPACE(n)` | n 個空格 | 無 |
| `REPL(x, n)` | 重複 x n 次 | 無 |
| `STRZERO(n, w)` | 零補位數字 | `PRINTF('%0*d', w, n)` |

### 數值/邏輯函式

| VFP 函式 | 說明 | SQLite 等效 |
|----------|------|------------|
| `IIF(cond, a, b)` | 條件運算 | `CASE WHEN cond THEN a ELSE b END` |
| `NVL(x, default)` | 空值替代 | `COALESCE(x, default)` |
| `VAL(x)` | 字串轉數值 | `CAST(x AS REAL)` |
| `ROUND(n, d)` | 四捨五入 | `ROUND(n, d)` |
| `BETWEEN x AND y` | 範圍 | `BETWEEN x AND y` |

### 記錄操作

| VFP 語法 | 說明 |
|----------|------|
| `_TALLY` | 上一次 SQL 影響的記錄數 |
| `RECN()` | 目前記錄號碼 |
| `RECCOUNT()` | 總記錄數 |
| `EOF()` / `BOF()` | 到達末尾/開頭 |
| `SEEK expr IN alias` | 索引查找 |
| `SCAN ... ENDSCAN` | 逐筆掃描迴圈 |

### 巨集替換

```
-- & 運算子：將變數內容當作程式碼執行
cField = "KCSTMR"
SELECT &cField FROM CO01M
-- 等同 SELECT KCSTMR FROM CO01M
```

> ⚠ 巨集替換是 VFP 特有語法，SQLite/PostgreSQL 需改用動態 SQL 或參數化查詢。

---

## 系統常數

### 異動類型代碼（VISI0.VIS 系統參數）

| 參數 | 值 | 說明 |
|------|-----|------|
| `C__I_STO_ATYPE` | `"2I"` | 進貨（2 或 I）|
| `C__I_CON_ATYPE` | `"5O"` | 出貨（5 或 O）|
| `C__I_ST1_ATYPE` | `"4"` | 退廠 |
| `C__I_CO1_ATYPE` | `"3"` | 客退 |
| `C__I_PIO_ATYPE` | `"678"` | 庫存（6=撥入, 7=撥出, 8=調整）|
| `C__I_ADJ_ATYPE` | `"8"` | 調整 |
| `C__I_BEGIN` | `"B"` | 期初量代號 |

### 報表計算參數

| 參數 | 值 | 說明 |
|------|-----|------|
| `N__I_OPEN` | `1` | 每月開帳基準日 |
| `C__I_PRICE` | `"0"` | 報表單價：0=平均成本, 1=末進價, 2=健保價, 3=歷次進價 |
| `C__I_AMT` | `"0"` | 金額計算方向：0=出貨=期初+進-期末, 1=期末=期初+進-出 |
| `N__I_QTY_DECIMAL` | `1` | 數量小數位 |
| `N__I_AMT_DECIMAL` | `0` | 金額小數位 |

### 異常旗標

| 旗標 | 欄位 | 說明 |
|------|------|------|
| `H` | CO18H.HET | 偏高 |
| `L` | CO18H.HET | 偏低 |
| `N` | CO18H.HET | 正常 |

### 消耗類型

| 值 | 欄位 | 說明 |
|----|------|------|
| `"01"` | LCS | 標準消耗 |
