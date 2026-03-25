---
name: vision-database
description: |
  展望醫療系統 HIS 資料庫查詢技能。根據需求定位正確的資料表、欄位與代碼，
  產出可執行的 SQLite 或 PostgreSQL 查詢語句。同時涵蓋 VFP 報表系統語法。

  觸發詞：展望資料庫、vision_clinic.db、展望HIS、診所資料庫查詢、
  DBF檔案、CO01M、CO02M、CO02H、CO02P、CO03L、CO03M、
  CO05B、CO05O、CO07A、CO09D、CO09S、CO10A、CO10H、CO10P、
  CO11A、CO11H、CO14M、CO18H、CO18HD2、VISHFAM、
  病患查詢、處方記錄、SOAP病歷、檢驗數值、掛號記錄、藥品庫存、
  預防保健查詢、糖尿病照護追蹤、CKD追蹤、家醫計畫、
  庫存月報、存貨月報、進銷存、月結檔、
  庫存異動、期初存量、期末結存、庫位管理、WHI庫存系統、
  醫師主檔、人員查詢、進貨單頭、庫存調整、健檢明細、
  報表系統、報表模組、VFP報表、F_GTFM、INT報表、PRG程式、
  存貨報表、耗用統計、盤點報表、安全存量
---

# Skill: 展望醫療系統資料庫

本 Skill 提供展望診所 HIS 系統的完整資料庫知識。啟用後，Claude 能根據需求找到正確的資料表、欄位和代碼，產出可執行的查詢。

## 使用時機

當以下任一條件符合時，應自動啟用此 Skill：

- 使用者提到**展望資料庫**、**vision_clinic.db**、**HIS 系統**、**診所資料**
- 需要查詢**病患**、**處方**、**檢驗**、**掛號**、**庫存**相關醫療資料
- 使用者提到 **DBF 檔案**或相關資料表名稱（CO01M, CO02M, CO02F, CO02H, CO02P, CO03L, CO03M, CO05B, CO05O, CO07A, CO09D, CO09S, CO10A, CO10H, CO10P, CO11A, CO11H, CO14M, CO18H, CO18HD2, VISHFAM, CO9Dxxxx）
- 使用者詢問**藥品庫存**、**進銷存**、**存貨月報**、**庫存月報**、**庫位管理**
- 使用者詢問**預防保健**、**疾病管理**（糖尿病照護、CKD 等）
- 使用者提到**病歷號** (KCSTMR)、**ICD 診斷碼**、**檢驗項目代碼** (HITEM)
- 使用者詢問**病歷紀錄**、**SOAP Notes**、**醫師記錄**、**看診紀錄內容**
- 使用者詢問**醫師主檔**、**人員資料** (CO07A)
- 使用者詢問**健檢明細**、**檢查細項數值** (CO18HD2)
- 使用者詢問**進貨單頭** (CO10H)、**庫存調整** (CO11A/CO11H)
- 使用者詢問**報表系統**、**VFP 報表**、**INT/PRG 報表程式**、**F_GTFM 框架**

## 使用說明

收到此 Skill 後，依以下流程處理：

1. **確認資料來源** — 判斷目前專案使用的是 SQLite (`vision_clinic.db`) 還是 PostgreSQL（Anchia Clinic Monitor）
2. **從 Decision Tree 定位資料表** — 根據使用者需求，找到該用哪些資料表和欄位
3. **產出查詢語句** — 根據資料來源產出正確語法的查詢（注意 SQLite vs PostgreSQL 差異）
4. **提醒查詢陷阱** — 如遇到 HVAL 數值比較、CO02F 分段合併、DNO 多義性等已知陷阱，主動提醒

---

## 資料來源設定

根據你的專案，資料來源可能不同。使用前請確認：

| 項目 | SQLite（原始查詢） | PostgreSQL（Anchia Clinic Monitor） |
|------|-------------------|-------------------------------------|
| 資料表名 | 大寫 `CO01M` | 小寫 `co01m` |
| 資料庫路徑 | `vision_clinic.db`（需設定） | 由專案連線設定決定 |
| 保留字欄位 | 不需引號 | `"date"`, `"time"`, `"case"` 需雙引號 |
| 額外欄位 | 無 | `record_hash`, `created_at`, `updated_at` |
| 最近 N 筆語法 | `GROUP BY` + `MAX()` | `DISTINCT ON (col)` |

**SQLite 連線範例**：
```bash
sqlite3 "<專案中 vision_clinic.db 的路徑>" "SELECT ..."
```

---

## 需求 → 資料表速查（Decision Tree）

根據你要做的事，找到該用哪些資料表：

### 病患相關

| 我需要... | 主資料表 | 關鍵欄位 | 備註 |
|-----------|---------|----------|------|
| 查病患基本資料（姓名、性別、生日、電話、地址） | **CO01M** | `KCSTMR`(病歷號), `MNAME`, `MSEX`, `MBIRTHDT`, `MPERSONID`(身分證), `MTELH`, `MTELO`, `MADDR` | 主鍵 `KCSTMR` 為 7 位數字字串 |
| 用身分證找病歷號 | **CO01M** | `MPERSONID` → `KCSTMR` | |
| 從家醫計畫找病患 | **VISHFAM** → **CO01M** | `VISHFAM.PAT_PID` = `CO01M.MPERSONID` | VISHFAM 用身分證，非病歷號 |

### 就診與掛號

| 我需要... | 主資料表 | 關鍵欄位 | 備註 |
|-----------|---------|----------|------|
| 就診歷史（含診斷） | **CO03L** | `KCSTMR`, `DATE`, `TIME`, `LABNO`(ICD主診斷), `LABDT`(診斷中文), `LTIME`(完診時間) | `LTIME='000000'` 表示未完診 |
| 掛號記錄（完整） | **CO05O** | `KCSTMR`, `TBKDT`, `TSTS`(狀態), `TNAME`, `TARTIME`(看診順序) | 含費用欄位 |
| 預約記錄 | **CO05B** | `KCSTMR`, `TBKDT`(日期), `TID`(醫師), `TSTS`(狀態), `TARTIME`(順序), `LM`(午別) | `TBKDATE`/`TBKTIME` 已棄用；狀態碼與 CO05O **不同** |
| 今日門診狀態 | **CO05O** | `TBKDT`=今日 + `TSTS IN ('0','1')` | 候診中=0, 看診中=1, 完診=F |
| 看診費用與診斷碼 | **CO03M** | `KCSTMR`, `IDATE`, `LABNO`(主ICD), `LACD01`~`LACD05`(次ICD), `TOT`(總額), `A98`(部分負擔) | |
| 醫師/人員資料 | **CO07A** | `NO`(人員碼), `NAME`(姓名), `ICD9NO` | `NO` 對應 CO03L.LPID |

### 處方與用藥

| 我需要... | 主資料表 | 關鍵欄位 | 備註 |
|-----------|---------|----------|------|
| 處方記錄 | **CO02M** | `KCSTMR`, `IDATE`, `DNO`(醫令碼), `PFQ`(頻率), `PTDAY`(天數), `PTQTY`(數量) | |
| 藥品名稱與資訊 | **CO09D** | `KDRUG`(醫令碼), `DNO`(健保碼), `DDESC`(英文名), `DDESC2`(中文名), `DP2`(健保價) | |
| 處方→藥品 JOIN | **CO02M** + **CO09D** | `CO02M.DNO = CO09D.KDRUG OR CO02M.DNO = CO09D.DNO` | 兩欄都要比對 |

### 檢驗與生理量測

| 我需要... | 主資料表 | 關鍵欄位 | 備註 |
|-----------|---------|----------|------|
| 檢驗數值（抽血等） | **CO18H** | `KCSTMR`, `HDATE`, `HITEM`(項目代碼), `HVAL`(數值), `HDSCP`(說明), `HRESULT`(判定) | `HVAL` 為文字，數值比較需 CAST |
| 生理量測（血壓等） | **CO18H** | 同上，`HITEM` 無 `Z0S` 前綴 | BP 格式為 `收縮壓/舒張壓` |
| 某項檢驗的趨勢 | **CO18H** | `WHERE KCSTMR=? AND HITEM=? ORDER BY HDATE DESC` | |
| 最近一次各項結果 | **CO18H** | 需子查詢取每個 HITEM 的 MAX(HDATE) | 見查詢範例 |
| 健檢細項數值（含單位/參考值） | **CO18HD2** | `CNORNO`(病歷號), `D2DATE`, `KDRUG`(項目碼), `D2NM`(項目名), `D2VL`(數值), `D2UN`(單位), `D2RF`(參考值) | 透過 `CO18H.CNORNO = CO18HD2.CNORNO` 關聯 |
| 健檢報告文字（Memo） | **CO18HD2** | `D2TX`, `D2T0`~`D2T2`, `D2RM`(備註), `D213` | Memo 欄位可存長文字 |

### 病歷紀錄（SOAP Notes）

| 我需要... | 主資料表 | 關鍵欄位 | 備註 |
|-----------|---------|----------|------|
| 醫師看診病歷（SOAP） | **CO02H** | `KCSTMR`, `SDATE`, `STIME`, `STEXT`(內容), `SATB`(段落序號) | 超過 250 字分段，需按 SATB 排序合併 |
| 特定日期的完整病歷 | **CO02H** | `GROUP_CONCAT(STEXT, '' ORDER BY SATB)` 合併 | 同一次看診以 KCSTMR+SDATE+STIME 識別 |
| 主診斷 ICD 碼 | **CO02H** | `SLABNO` | 直接存於病歷記錄中 |

### 檢查報告

| 我需要... | 主資料表 | 關鍵欄位 | 備註 |
|-----------|---------|----------|------|
| 超音波/心電圖/肺功能報告 | **CO02F** | `KCSTMR`, `FDATE`, `FNO`(報告類型), `FTEXT`(內容), `FSQ`(序號) | 超過 250 字分段，需合併 |
| 哪些就診有做檢查 | **CO02M** | `DNO` 為檢查代碼（如 `19009C` 腹超） | |

### 藥品庫存

| 我需要... | 主資料表 | 關鍵欄位 | 備註 |
|-----------|---------|----------|------|
| 藥品庫存量（簡易） | **CO09D** | `KDRUG`, `DQTY2`(庫存), `DORDPOINT`(安全量), `DLVENDOR`(廠商) | 低庫存: `DQTY2 <= DORDPOINT` |
| 目前庫存（精確計算） | **CO09S** + **CO09D** | `SUM(DQTY) WHERE DLCT='O' AND DIO<>'@'` | 排除作廢，限本庫 |
| 庫存異動歷程 | **CO09S** | `KDRUG`, `DDATE`, `DIO`(類型), `DQTY`(數量), `DLCT`(庫位) | DIO: 5=出貨, B=期初, 2=進貨, @=作廢 |
| 進貨/出貨/調整記錄 | **CO10A** + **CO10P** | `KDRUG`, `ADATE`, `AQTY`, `APRICE`, `ATYPE`→`PTYP1` | CO10P.PQTY 決定 +/- 方向 |
| 月消耗量統計 | **CO09S** | `DIO='5'` + `DDATE LIKE 'YYYMM%'` + `SUM(DQTY)` | |
| 存貨月報 / 月結資料 | **CO9Dxxxx** + **CO09D** | `QTYBB`(期初), `QTYEE`(結存), `AMTBB`, `AMTEE` | 需先執行成本結算作業 |
| 進銷存即時計算 | **CO09S** + **CO10A** + **CO02P** | EQTY = BQTY + IQTY - OQTY | 見 `references/inventory-system.md` |
| 異動類型定義 | **CO10P** | `PTYP1`(代碼), `PTITL`(名稱), `PQTY`(+/-方向) | 僅 3 筆記錄 |
| 庫位 / 廠商資訊 | **CO14M** | `MTYPE`+`MNO`(複合鍵), `MNAME` | MTYPE: 1=主要, 3=倉庫 |
| 藥局調劑交易明細 | **CO02P** | `PDATE`, `KDRUG`, `PTQTY`(含正負號) | 不同於 CO02M 處方記錄 |
| 進貨單頭（訂單層級） | **CO10H** | `KDONO`(單號), `ADATE`(日期), `MNO`(供應商), `MTOTAL`(總額), `DRUGLIC`(藥商執照) | 與 CO10A 以 `KDONO` 關聯 |
| 庫存調整明細 | **CO11A** | `ASMNM`(調整單號), `KDRUG`, `ADTYP`(調整類型), `PTQTY`(實際數量), `PDAY` | 與 CO11H 以 `ASMNM` 關聯 |
| 庫存調整單頭 | **CO11H** | `ASMNM`(調整單號), `HTP`(調整類型旗標), `AACD1`~`AACD6` | 盤點/調整/報廢的單頭資訊 |

### 預防保健與疾病管理

| 我需要... | 主資料表 | 關鍵欄位 | 備註 |
|-----------|---------|----------|------|
| 成人健檢記錄 | **CO03L** | `LISRS IN ('3D','21','22','3E','23','24')` | 一階/二階 |
| 癌症篩檢記錄 | **CO03L** | `LISRS='85'`(大腸癌), `LISRS='95'`(口腔癌) | |
| 疫苗接種記錄 | **CO03L** | `LISRS='AU'`(流感), `LISRS='VU'`(新冠), `LISRS='DU'`(肺炎鏈球菌) | |
| 糖尿病照護追蹤 | **CO02M** | `DNO IN ('P1406C'~'P1410C','P7001C','P7002C')` | 見代碼表 |
| CKD 照護追蹤 | **CO02M** | `DNO IN ('P4301C','P4302C')` | |
| 代謝症候群追蹤 | **CO02M** | `DNO IN ('P7501C','P7502C','P7503C')` | |
| 家醫計畫成員 | **VISHFAM** | `CASE_TYPE`(A=健保收案, B=一般), 健康風險: `HBP`(高血壓), `ASCVD` 等 | |

### 報表系統（VFP INT/PRG）

| 我需要... | 參考資源 | 關鍵知識 | 備註 |
|-----------|---------|----------|------|
| 了解報表框架與執行流程 | `references/report-system.md` | F_GTFM 語法、F_GETVAR/F_OPT 函式 | 所有 .INT 報表共用此框架 |
| 查找特定報表模組 | `references/report-system.md` | VST/VISI/WHR 系列、SCX 表單 | 14 大類 50+ 模組 |
| 撰寫 VFP SQL 查詢 | `references/report-system.md` | 日期範圍、JOIN、聚合、CURSOR | 含 ZZZZZZ 哨兵值、$ 運算子 |
| 使用 VFP 內建函式 | `references/report-system.md` | ALLTRIM, STRZERO, LEFTX, IIF, NVL | LEFTX 處理中文安全截斷 |
| 理解報表的條件輸入介面 | `references/report-system.md` | F_GTFM 格式字串、TXBCLM0 COM | 日期選擇器、選項框 |

---

## 資料表 Schema 精簡參考

### CO01M - 病患主檔（主鍵: KCSTMR）

| 欄位 | 說明 | 範例 |
|------|------|------|
| KCSTMR | 病歷號 (7位) | `0000001` |
| MNAME | 姓名 | `王小明` |
| MSEX | 性別 (1=男, 2=女) | `1` |
| MBIRTHDT | 生日 (民國年) | `0761015` |
| MPERSONID | 身分證字號 | `A123456789` |
| MTELH / MTELO | 電話 | |
| MADDR | 地址 | |
| MREMARK | 備註 | |

### CO02M - 處方記錄（複合鍵: KCSTMR+IDATE+ITIME+DNO）

| 欄位 | 說明 | 範例 |
|------|------|------|
| KCSTMR | 病歷號 | |
| IDATE | 就診日期 (民國年) | `1141127` |
| ITIME | 就診時間 | `140530` |
| DNO | 藥品/醫令代碼 | `P1407C` |
| WICTM | 處方內容說明 | |
| PFQ | 頻率 (QD/BID/TID/QID/PRN) | `TID` |
| PTDAY | 天數 | `7` |
| PTQTY | 總數量 | `2100` |

### CO02H - 醫師病歷紀錄 SOAP Notes（複合鍵: KCSTMR+SDATE+STIME+SATB）

| 欄位 | 說明 | 範例 |
|------|------|------|
| KCSTMR | 病歷號 | `0000001` |
| SDATE | 看診日期 (民國年) | `1120801` |
| STIME | 看診時間 | `103819` |
| SIDS | 診間代碼 (01=主診間) | `01` |
| SID | 醫師代碼 (1 或 2) | `2` |
| SATB | 段落序號 (1,2,...9,A,B,C,D,E) | `1` |
| STYP | 段落類型 (O=首段, 空=接續段) | `O` |
| SLM | 身分別 (A=健保, 9=其他, 1=重大傷病, 3=福保, 4=榮民) | `A` |
| SDAY | 給藥天數 | `28` |
| SLABNO | 主診斷 ICD-10 碼 | `E1165` |
| STEXT | 病歷內容 (max 250 字/段) | SOAP 文字 |

**分段儲存機制**：STEXT 每段上限 250 字元，長病歷拆成多筆記錄。
- `SATB='1'` + `STYP='O'`：首段（每次看診必有）
- `SATB='2','3'...` + `STYP=''`：接續段
- 同一次看診以 `KCSTMR + SDATE + STIME` 識別
- 最多可達 14 段（3,500 字元）

### CO02F - 檢查報告（複合鍵: KCSTMR+FDATE+FTIME+FNO+FSQ）

| 欄位 | 說明 | 範例 |
|------|------|------|
| KCSTMR | 病歷號 | |
| FDATE / FTIME | 日期/時間 | |
| FNO | 報告類型 (P1=心電圖, P2=腹超, P3=甲超, P5/P6=肺功能) | `P2` |
| FTEXT | 報告內容 (max 250 字) | |
| FSQ | 分段序號 | `01` |

### CO03L - 就診記錄（複合鍵: KCSTMR+DATE+TIME）

| 欄位 | 說明 | 範例 |
|------|------|------|
| KCSTMR | 病歷號 | |
| DATE / TIME | 就診日期/時間 | |
| LABNO | 主診斷 ICD | `E11` |
| LABDT | 主診斷中文 | |
| LISRS | 預防保健卡序代碼 | `3D` |
| LPID | 身分別 (A=健保, 9=其他, 空=自費) | `A` |
| LTIME | 完診時間 (000000=未完診) | `150230` |
| DAYQTY | 處方天數 | |

### CO03M - 看診費用（複合鍵: KCSTMR+IDATE+ITIME）

| 欄位 | 說明 |
|------|------|
| LABNO | 主診斷 ICD |
| LACD01~LACD05 | 次要診斷 ICD (最多 5 組) |
| TOT | 總金額 |
| A98 | 部分負擔 |
| RTOT | 申報總額 |

### CO05O - 門診掛號（複合鍵: KCSTMR+TBKDT+TBKTIME）

| 欄位 | 說明 | 範例 |
|------|------|------|
| TBKDT | 掛號日期 | |
| TBKTIME | 掛號時間 | |
| TSTS | 狀態 (0=候診, 1=看診中, E=預約, F=完診, H=取消) | `F` |
| LM | 身分別 | `A` |
| TARTIME | 看診順序號 | `00015` |
| TNAME | 姓名 | |

### CO05B - 預約掛號（複合鍵: KCSTMR+TBKDT+TID+TARTIME）

| 欄位 | 說明 | 範例 |
|------|------|------|
| KCSTMR | 病歷號 (7位左補零) | `0002368` |
| TBKDT | **預約日期**（民國年 YYYMMDD） | `1150408` |
| ~~TBKDATE~~ | ~~預約日期（舊欄位）~~ | **已棄用，值為空字串，勿使用** |
| ~~TBKTIME~~ | ~~預約時段（舊欄位）~~ | **已棄用，值為空字串，勿使用** |
| TID | 醫師代碼 | `2` |
| TIDS | 時段代碼 | `01` |
| TSTS | **預約狀態**（見下方狀態碼） | `S` |
| TARTIME | **報到序號/看診順序** | `0005` |
| TNOTE | 預約備註 | `檢`=檢查, `複`=複診 |
| LM | 午別 (A=上午, B=下午, C=晚上) | `A` |
| BNAME | 病患姓名（預約時帶入） | `曾欣儀` |
| TDUTY | 診間/值班資訊 | `B201林` (診間代碼+醫師姓) |
| TIDSRM | 診間代碼 | `2` |

**CO05B.TSTS 狀態碼**（⚠ 與 CO05O.TSTS 為不同編碼系統）：

| 狀態碼 | 說明 | 分類 |
|--------|------|------|
| `""` / `0` / `1` | 空值/待確認/已確認 | 有效 |
| `S` | 已排約 (Scheduled) | 有效 |
| `U` | 已報到 (Checked-in) | 有效 |
| `E` | 看診中 (In progress) | 有效 |
| `F` | 已完診 (Finished) | 有效 |
| `C` | 已取消 | 取消 |
| `X` | 已刪除 | 取消 |
| `D` | 已作廢 | 取消 |
| `H` | 電話取消 | 取消 |

> 篩選有效預約：`WHERE TSTS NOT IN ('C','X','D','H')`。排名排序用 `ORDER BY TARTIME ASC`。

### CO07A - 醫師/人員主檔（主鍵: NO，14,119 筆）

| 欄位 | 說明 | 範例 |
|------|------|------|
| NO | 人員代碼（對應 CO03L.LPID） | `2` |
| NAME | 姓名 (80 字元) | `林醫師` |
| ICD9NO | 醫事人員代碼 | |

> 完整 21 欄位詳見 `references/schema-verified.md`

### CO18H - 檢驗報告與生理量測（複合鍵: KCSTMR+HDATE+HITEM）

| 欄位 | 說明 | 範例 |
|------|------|------|
| KCSTMR | 病歷號 | |
| HDATE | 檢驗日期 | `1141127` |
| HITEM | 項目代碼 | `Z0SHbA1c` |
| HDSCP | 項目說明 | `HbA1C,醣化血色素` |
| HVAL | 數值（文字型態） | `6.5` |
| HRESULT | 結果判定 | `偏高` |

### CO18HD2 - 健檢/病歷明細（複合鍵: CNORNO+HITEM+D2SS，162,045 筆）

| 欄位 | 說明 | 範例 |
|------|------|------|
| KCSTMR | 病歷號 | `0000024` |
| CNORNO | 病歷號碼（FK → CO18H.CNORNO） | |
| D2DATE / D2TIME | 檢查日期/時間 | `1141127` |
| KDRUG | 項目代碼（FK → CO09D.KDRUG） | |
| D2NM | 項目名稱 (100 字元) | `HbA1C,醣化血色素` |
| D2VL | 數值結果 (200 字元，文字型態) | `6.5` |
| D2UN | 單位 | `%` |
| D2RF | 參考值範圍 | `4.0-6.0` |
| D2SS | 序號 | |
| D2TX, D2T0~D2T2 | Memo 文字欄位（可存長文字） | |
| D2RM | 備註 | |
| D213 | 擴充 Memo 欄位 | |

> CO18HD2 vs CO18H：CO18H 存放單一數值結果 (HVAL)，CO18HD2 存放結構化的檢查明細（含單位、參考值、Memo 文字）。兩者透過 `CNORNO` 關聯。

### CO09D - 藥品主檔（主鍵: KDRUG，共 102 欄位）

| 欄位 | 說明 |
|------|------|
| KDRUG | **醫令碼**（主鍵，C(6)） |
| DNO | 藥品編號（C(12)） |
| KNO | **健保碼**（C(12)，與 DNO 不同） |
| DDESC / DDESC2 | **藥品名稱** / **商品名** |
| DUM1 | **單位**（存貨月報使用） |
| DTYPE | **藥品分類**（2=內服, 3=外用, 4=注射, 12=材料，用 `VAL(DTYPE)` 比較） |
| DSTS | **狀態**（`'*'`=停用） |
| DPRICE | 單價（存貨月報用） |
| DLPRI | **末進價**（最後一次進貨單價） |
| DP2 | 健保給付價 |
| DQTY2 | 目前庫存量 |
| DORDPOINT | 安全庫存量 |
| DLVENDOR | 供應商 |
| DBCODE | **條碼** |
| DGRP | 群組碼（存貨月報篩選） |
| DRESET | 重設/啟用日期（篩選有效品項） |
| DSITEM | **自費分類**（報表篩選用） |
| DPLACE | 儲位代碼 |

> 完整 102 欄位詳見 `references/inventory-system.md`

### CO09S - 庫存餘額檔（複合索引: DLCT+KDRUG+DDATE+...+DIO，僅 6 欄位）

| 欄位 | 說明 |
|------|------|
| DLCT | **庫位**：`'O'`=本庫, `'D'`=其他庫位（空白視為 `'O'`） |
| KDRUG | 藥品代碼 |
| DDATE | 異動日期（YYYMMDD） |
| DIO | **異動類型**（對應 CO10P.PTYP1）：`'B'`=期初, `'2'`=進貨, `'5'`=出貨, `'@'`=作廢 |
| DQTY | 庫存數量 |
| DAMT | 庫存金額 |

> 期初存量 SEEK 語法：`'O' + KDRUG + 月初日期 + '**B'`

### CO10A - 庫存異動交易檔（索引: ADATE）

| 欄位 | 說明 |
|------|------|
| ATYPE | **異動類型**（對應 CO10P.PTYP1）：2/I=進貨, 5/O=出貨, 3=客退, 4=退廠, 6=撥入, 7=撥出, 8=調整 |
| KDONO | **單據號碼** |
| KDRUG | 藥品代碼 |
| ADATE | 異動日期（YYYMMDD） |
| MTYPE + MNO | 庫位類型+代碼（對應 CO14M） |
| AQTY | **異動數量**（正負方向由 CO10P.PQTY 決定） |
| APRICE | 單價 |
| ATOTAL | 異動金額 |
| AREMARK | 備註（進貨時為有效期限，西元 YYYYMMDD） |
| AITEM | 項次 |
| AUM | 單位 |

### CO10H - 進貨單頭檔（主鍵: KDONO，0 筆）

| 欄位 | 說明 |
|------|------|
| KDONO | **單據號碼**（對應 CO10A.KDONO） |
| ADATE | 進貨日期（YYYMMDD） |
| MNO | 供應商代碼（FK → CO14M） |
| MTOTAL | 訂單總額 |
| DRUGLIC | 藥商執照號碼 |

> 記錄數為 0（目前無進貨單頭資料），完整 22 欄位詳見 `references/schema-verified.md`

### CO11A - 庫存調整明細（複合鍵: ASMNM+KDRUG，620 筆）

| 欄位 | 說明 |
|------|------|
| ASMNM | **調整單號**（FK → CO11H.ASMNM） |
| KDRUG | 藥品代碼（FK → CO09D） |
| ADTYP | 調整類型 |
| PQTY / PTQTY | 預計數量 / 實際數量 |
| PDAY | 天數 |

> 完整 23 欄位詳見 `references/schema-verified.md`

### CO11H - 庫存調整單頭（主鍵: ASMNM，75 筆）

| 欄位 | 說明 |
|------|------|
| ASMNM | **調整單號** |
| HTP | 調整類型旗標 |
| AACD1~AACD6 | 分類代碼欄位 |
| AICD / AITM | 對應 ICD/項目 |

> 完整 23 欄位詳見 `references/schema-verified.md`

### CO10P - 異動類型定義檔（主鍵: PTYP1，僅 3 筆）

| 欄位 | 說明 |
|------|------|
| PTYP1 | **異動類型代碼**（與 CO10A.ATYPE、CO09S.DIO 關聯） |
| PTITL | **名稱**（報表取前 4 字元） |
| PQTY | **數量方向**：`"+"`=入庫, `"-"`=出庫 |

### CO02P - 交易明細檔（索引: PDATE，約 28 萬筆）

| 欄位 | 說明 |
|------|------|
| KCSTMR | 客戶代碼 |
| PDATE | 交易日期（YYYMMDD） |
| PTIME | 交易時間 |
| KDRUG | 藥品代碼 |
| PTQTY | **異動數量**（實際扣庫量，含正負號） |

> **注意**：CO02P（藥局調劑明細）≠ CO02M（醫師處方記錄）。CO02P.PTQTY 含正負號，CO10A.AQTY 為絕對值。

### CO14M - 庫位/廠商主檔（複合鍵: MTYPE+MNO，約 85 筆）

| 欄位 | 說明 |
|------|------|
| MTYPE | **庫位類型**：`'1'`=主要, `'3'`=倉庫 |
| MNO | **庫位/廠商代碼** |
| MNAME | 名稱 |
| MTEL1 / MFAX / MEMAIL | 聯絡資訊 |
| MDRUGLIC | 藥商執照號碼 |

### CO9Dxxxx - 月結成本結算檔

> 由成本結算作業 (`VISI_CREATECO9D()`) 產生。檔名：`CO9D` + 年月代碼。

| 欄位 | 說明 |
|------|------|
| KDRUG / DLCT | 藥品代碼 / 庫位 |
| QTYBB / AMTBB | **期初**存貨數量 / 金額 |
| QTYEE / AMTEE | **本期結存**數量 / 金額 |
| DPRICE | 單價 |
| 各異動類型欄位 | 由 CO10P 動態產生 |

### VISHFAM - 家醫計畫（主鍵: PAT_PID）

| 欄位 | 說明 |
|------|------|
| PAT_PID | 身分證字號 |
| PAT_NAMEC | 姓名 |
| CASE_TYPE | A=健保收案, B=一般, C=特殊, D=結案 |
| MEMB_TYPE | 會員類型 |
| HBP, ASCVD, BEF, DIS | 健康風險標記 |

---

## 跨表關聯圖

```
CO01M (病患主檔, 主鍵: KCSTMR)
  │
  ├── KCSTMR ──→ CO02M (處方), CO02F (報告), CO02H (病歷SOAP),
  │                CO02P (藥局調劑), CO03L (就診), CO03M (費用),
  │                CO05B (預約), CO05O (掛號), CO18H (檢驗)
  │
  └── MPERSONID ←→ VISHFAM.PAT_PID (身分證關聯)

CO02H ←→ CO02M   以 (KCSTMR + SDATE/IDATE + STIME/ITIME) 關聯病歷與處方
CO02M ←→ CO02F   以 (KCSTMR + IDATE/FDATE + ITIME/FTIME) 關聯處方與檢查報告
CO03L ←→ CO03M   以 (KCSTMR + DATE + TIME) 關聯就診與費用
CO05B ──→ CO01M   以 KCSTMR 關聯病患主檔
CO05O ⊃ CO05B    CO05B 為預約掛號，CO05O 為門診掛號（⚠ TSTS 編碼不同）

CO02M.DNO ──→ CO09D.KDRUG 或 CO09D.DNO (處方→藥品)
CO09D.KDRUG ←── CO09S.KDRUG (庫存餘額)
CO09D.KDRUG ←── CO10A.KDRUG (庫存異動)
CO09D.KDRUG ←── CO02P.KDRUG (藥局調劑)
CO10A.ATYPE ──→ CO10P.PTYP1 (異動類型定義)
CO10A.MTYPE+MNO ──→ CO14M.MTYPE+MNO (庫位/廠商)
CO10A.KDONO ──→ CO10H.KDONO (進貨單頭)
CO11A.ASMNM ──→ CO11H.ASMNM (調整單頭)
CO11A.KDRUG ──→ CO09D.KDRUG (調整藥品)
CO18H.CNORNO ──→ CO18HD2.CNORNO (健檢明細)
CO18HD2.KDRUG ──→ CO09D.KDRUG (檢查項目對應藥品主檔)
CO03L.LPID ──→ CO07A.NO (就診醫師/人員)
CO9Dxxxx ←── CO09D.KDRUG (月結成本檔，需先結算)
```

---

## 核心資料慣例

### 日期格式

所有日期欄位為民國年 7 位字串 `YYYMMDD`：
- `西元年 = 民國年 + 1911`
- `1150212` → 民國 115 年 2 月 12 日 → `2026-02-12`
- 因格式固定，**可直接字串比較大小** (`>`, `<`, `BETWEEN`)

### 時間格式

`HHMMSS` 6 位字串，如 `140530` = 14:05:30

### 計算今日民國年日期

```
西元年份 - 1911 → 補零到 3 位 → 接月(2位)日(2位)
2026-02-12 → 115 → '1150212'
```

### 日期計算速查（臨床查詢常用）

在查詢中經常需要「N 個月前」或「N 年前」的日期。以下為 SQLite 計算方式：

```sql
-- 今日民國年日期（SQLite）
SELECT PRINTF('%03d', CAST(STRFTIME('%Y', 'now') AS INTEGER) - 1911)
    || STRFTIME('%m%d', 'now') AS today_roc;
-- → '1150220'

-- 3 個月前
SELECT PRINTF('%03d', CAST(STRFTIME('%Y', 'now', '-3 months') AS INTEGER) - 1911)
    || STRFTIME('%m%d', 'now', '-3 months') AS three_months_ago;
-- → '1141120'

-- 1 年前
SELECT PRINTF('%03d', CAST(STRFTIME('%Y', 'now', '-1 years') AS INTEGER) - 1911)
    || STRFTIME('%m%d', 'now', '-1 years') AS one_year_ago;
-- → '1140220'
```

**PostgreSQL 版本：**
```sql
SELECT LPAD((EXTRACT(YEAR FROM CURRENT_DATE) - 1911)::TEXT, 3, '0')
    || TO_CHAR(CURRENT_DATE, 'MMDD') AS today_roc;

SELECT LPAD((EXTRACT(YEAR FROM CURRENT_DATE - INTERVAL '3 months') - 1911)::TEXT, 3, '0')
    || TO_CHAR(CURRENT_DATE - INTERVAL '3 months', 'MMDD') AS three_months_ago;
```

**實用組合**：在臨床工作流查詢中替換硬編碼日期：
```sql
-- 動態版「DM 逾期未追蹤 HbA1c」
... HAVING last_hba1c_date IS NULL
   OR last_hba1c_date < (
     SELECT PRINTF('%03d', CAST(STRFTIME('%Y','now','-3 months') AS INTEGER)-1911)
         || STRFTIME('%m%d','now','-3 months')
   )
```

---

## 檢驗項目代碼速查（CO18H.HITEM）

### 生理量測（無前綴）

| HITEM | 說明 | 格式/單位 |
|-------|------|-----------|
| BP | 血壓 | `收縮壓/舒張壓` 如 `177/97` (mmHg) |
| BT | 體溫 | °C |
| BW | 體重 | kg |
| BH | 身高 | cm |
| BMI | BMI | 數值 |
| HR | 心率 | bpm |
| WC | 腰圍 | cm |
| AC | 飯前血糖(即時) | mg/dL |
| PC | 飯後血糖(即時) | mg/dL |

### 檢驗項目（Z0S 前綴）

| 分類 | HITEM | 說明 |
|------|-------|------|
| **糖尿病** | Z0SHbA1c | 醣化血色素 |
| | Z0SAC | 空腹血糖 |
| | Z0SINS | Insulin |
| | Z0SHOMA | HOMA-IR |
| **腎功能** | Z0SCRE | Creatinine 肌酸酐 |
| | Z0SEGFR | eGFR |
| | Z0SBUN | BUN |
| | Z0SMIA | 微小白蛋白 |
| | Z0SUPCR | 尿蛋白/肌酸酐比值 |
| | Z0SK / Z0SNA | 鉀/鈉離子 |
| **血脂** | Z0SCHO | 總膽固醇 |
| | Z0STG | 三酸甘油脂 |
| | Z0SLDL | LDL-C |
| | Z0SHDL | HDL-C |
| **肝功能** | Z0SGOT | AST |
| | Z0SGPT | ALT |
| | Z0SGGT | r-GT |
| | Z0SALKP | 鹼性磷酸酶 |
| | Z0STBIL / Z0SDBIL | 總/直接膽紅素 |
| | Z0SALB | 白蛋白 |
| **甲狀腺** | Z0STSH | TSH |
| | Z0SFT4 / Z0SFT3 | Free T4 / Free T3 |
| **血液** | Z0SHB | 血色素 |
| | Z0SWBC | 白血球 |
| | Z0SRBC | 紅血球 |
| | Z0SPLT | 血小板 |
| | Z0SMCV / Z0SMCH | MCV / MCH |
| **發炎** | Z0SCRP | CRP |
| | Z0SHSCRP | hs-CRP |
| **尿酸** | Z0SUA | Uric Acid |
| **腫瘤標記** | Z0SPSA / Z0SFPSA | PSA / Free PSA |
| | Z0SCEA | CEA |
| | Z0SAFP | AFP |
| **尿液** | Z0SUPRO | 尿蛋白 |
| | Z0SUGLU | 尿糖 |
| **B/C肝** | Z0SHBEP0~07 | B肝系列 |
| | Z0SHCV / Z0SHCVR | C肝抗體/病毒量 |
| **維生素** | Z0SVITD | Vitamin D |
| | Z0SB12 | Vitamin B12 |
| | Z0SFOL | 葉酸 |
| | Z0SFe / Z0SFER | 鐵/鐵蛋白 |

---

## 代碼對照表

### 疾病管理醫令代碼（CO02M.DNO）

| 分類 | 代碼 | 名稱 |
|------|------|------|
| 糖尿病 | P1406C | 初診 |
| | P1407C | 照護方案 |
| | P1408C | 定期追蹤 |
| | P1409C | 持續追蹤管理 |
| | P1410C | 出院追蹤 |
| | P7001C | DKD-複診 |
| | P7002C | DKD-年度 |
| 腎臟病 | P4301C | CKD 照護 |
| | P4302C | CKD 追蹤 |
| 代謝症候群 | P7501C | 照護 |
| | P7502C | 追蹤 |
| | P7503C | 定期追蹤 |

### 預防保健代碼（CO03L.LISRS）

| 代碼 | 項目 |
|------|------|
| 3D, 21, 22 | 成人健檢一階 |
| 3E, 23, 24 | 成人健檢二階 |
| 85 | 大腸癌篩檢 |
| 95 | 口腔癌篩檢 |
| AU | 流感疫苗 |
| VU | 新冠疫苗 |
| DU | 肺炎鏈球菌疫苗 |

### 其他代碼

| 代碼系統 | 欄位 | 值 |
|---------|------|-----|
| 性別 | CO01M.MSEX | 1=男, 2=女 |
| 身分別 | CO03L.LPID / CO05O.LM / CO02H.SLM | A=健保, 9=其他, 1=重大傷病, 3=福保, 4=榮民, 空=自費 |
| 看診狀態 | CO05O.TSTS | 0=候診, 1=看診中, E=預約, F=完診, H=取消 |
| **預約狀態** | CO05B.TSTS | S=已排約, U=已報到, E=看診中, F=完診, 0=待確認, 1=已確認, C/X/D/H=取消（⚠ 與 CO05O 不同編碼） |
| 藥品類別 | CO09D.DTYPE | 2=內服, 3=外用, 4=注射, 12=材料（用 `VAL(DTYPE)` 比較） |
| 庫存異動 | CO09S.DIO / CO10P.PTYP1 | B=期初, 2/I=進貨, 5/O=出貨, 3=客退, 4=退廠, 6=撥入, 7=撥出, 8=調整, @=作廢 |
| 庫位 | CO09S.DLCT / CO14M.MTYPE | DLCT: O=本庫, D=其他；MTYPE: 1=主要, 3=倉庫 |
| 處方頻率 | CO02M.PFQ | QD=日1次, BID=日2次, TID=日3次, QID=日4次, PRN=需要時 |

---

## 常用查詢範例

> 以下為 SQLite 語法。更多查詢範例請參考 `references/query-examples.md`。

```sql
-- 病患搜尋（依姓名）
SELECT KCSTMR, MNAME, MSEX, MBIRTHDT FROM CO01M WHERE MNAME LIKE '%王%';

-- 最近就診歷史
SELECT DATE, TIME, LABNO, LABDT FROM CO03L WHERE KCSTMR = ? ORDER BY DATE DESC LIMIT 10;

-- 處方含藥品名稱
SELECT m.IDATE, m.DNO, d.DDESC2, m.PFQ, m.PTDAY
FROM CO02M m JOIN CO09D d ON m.DNO = d.KDRUG OR m.DNO = d.DNO
WHERE m.KCSTMR = ? ORDER BY m.IDATE DESC;

-- 最新 HbA1c
SELECT HDATE, HVAL, HRESULT FROM CO18H
WHERE KCSTMR = ? AND HITEM = 'Z0SHbA1c' ORDER BY HDATE DESC LIMIT 1;

-- SOAP 病歷合併（合併分段）
SELECT GROUP_CONCAT(STEXT, '') AS SOAP
FROM (SELECT STEXT FROM CO02H WHERE KCSTMR = ? AND SDATE = ? ORDER BY SATB ASC);

-- 低庫存藥品
SELECT KDRUG, DDESC2, DQTY2, DORDPOINT FROM CO09D
WHERE DQTY2 <= DORDPOINT ORDER BY (DQTY2 / NULLIF(DORDPOINT, 0)) ASC;
```

---

## 邊界與失敗模式

| 情況 | 為什麼失敗 | 辨認方式 |
|------|-----------|---------|
| 要求查詢病患當天的即時掛號狀態 | 資料是從 DBF 同步過來的，有時間差 | `CO05O.TSTS` 可能不是最新的 |
| 使用 `CO02M.DNO` 做跨院比較 | DNO 是院所自訂碼，不同院所同一藥品的 DNO 不同 | 跨院查詢要用健保碼 `CO09D.DNO` |
| 在 SQLite 用 DISTINCT ON | SQLite 不支援 DISTINCT ON（是 PostgreSQL 語法） | 改用 GROUP BY + MAX() 子查詢 |
| 直接把 HVAL 做數值大小比較 | HVAL 是文字型態，字串比較 "10" < "9" | 必須 `CAST(HVAL AS REAL)` |
| 多段病歷只查第一筆 | CO02H 按 SATB 分段，只取 `SATB='1'` 會截斷長病歷 | 加 `GROUP_CONCAT(STEXT, '' ORDER BY SATB)` |
| 跨 SQLite/PostgreSQL 直接複製查詢 | 表名大小寫、保留字、特定函數（GROUP_CONCAT vs STRING_AGG）都不同 | 換環境前逐一確認差異 |

**這個技能無法做的事：**
- 判斷哪些資料被標記為「作廢」或「錯誤輸入」（展望系統可能沒有刪除標記）
- 解讀醫師在 STEXT 中自由輸入的縮寫（每位醫師習慣不同）

## 查詢陷阱 (Gotchas)

1. **HVAL 是文字** — 數值比較必須 `CAST(HVAL AS REAL)`；血壓 (BP) 格式為 `收縮壓/舒張壓`，需用 `SUBSTR`+`INSTR` 拆分
2. **CO02F 報告分段** — 超過 250 字元拆成多筆 (FSQ 序號)，查詢必須合併
3. **CO02H 病歷分段** — STEXT 每段上限 250 字元，按 SATB 序號排序合併。同一次看診以 `KCSTMR+SDATE+STIME` 識別。SLM/SDAY/SLABNO 等欄位僅在首段 (`SATB='1'`) 有意義
4. **SQLite 無 DISTINCT ON** — 用 `GROUP BY` + `MAX()` 或相關子查詢替代
5. **CO02M.DNO 多義性** — 可能是藥品碼、檢查碼或疾病管理碼，依前綴判斷
6. **CO02M → CO09D 關聯** — `DNO` 可能對應 `KDRUG` 或 `DNO`，JOIN 需 `OR`
7. **PostgreSQL 保留字** — co03l 的 `date`, `time`, `case` 欄位需用雙引號
8. **CO09S.DDATE 特殊格式** — 期初庫存為 `YYYMM**`，過濾時注意
9. **CO10A.AREMARK** — 進貨時為有效期限（西元 `YYYYMMDD`），調整時通常為空
10. **CO10A.AQTY vs CO02P.PTQTY** — AQTY 為絕對值（正負由 CO10P.PQTY 決定），PTQTY 含正負號（直接扣庫量）
11. **CO14M JOIN 需複合鍵** — 必須用 `MTYPE+MNO` 關聯，不能只用 MNO
12. **CO9Dxxxx 需先結算** — 存貨月報表讀取月結檔 `CO9Dxxxx.DBF`，必須先執行成本結算作業才存在
13. **CO09S.DLCT 庫位** — 查庫存必須指定 `DLCT='O'`（本庫），空白也視為本庫
14. **DTYPE 數值比較** — 系統用 `VAL(DTYPE)` 轉數值，不是字串比較（`'2'` 不是 `'02'`）
15. **CO05B.TBKDATE/TBKTIME 已棄用** — 這兩個舊欄位在目前資料庫中值皆為空字串，預約日期必須用 `TBKDT`
16. **CO05B.TSTS ≠ CO05O.TSTS** — 兩張表的狀態碼是不同的編碼系統。篩選有效預約用 `CO05B.TSTS NOT IN ('C','X','D','H')`，不要套用 CO05O 的狀態碼邏輯
17. **CO05B 病患姓名優先順序** — `CO05O.TNAME` > `CO05B.BNAME` > `CO01M.MNAME`，取姓名時用 COALESCE 依序嘗試
18. **CO18HD2.D2VL 是 200 字元文字欄位** — 比 CO18H.HVAL (80 字元) 更長，同樣需要 CAST 才能做數值比較
19. **CO18HD2 vs CO18H 選擇** — CO18H 適合快速查單一數值結果 (HVAL)；CO18HD2 提供結構化明細（含單位 D2UN、參考值 D2RF、Memo 文字），但需透過 CNORNO 關聯
20. **CO03L 內嵌檢驗欄位** — CO03L 有 A0~A21 等直接儲存檢驗結果的欄位（92 欄位），但主要檢驗數值查詢仍建議走 CO18H/CO18HD2
21. **CO07A.NO 對應 CO03L.LPID** — 查詢醫師姓名時用 `CO07A.NO = CO03L.LPID` 關聯，不是 KCSTMR
22. **VFP 報表日期+時間查詢** — 報表框架使用 `DATE + TIME >= F_IBEG` 而非分開比較；時間上界加 `"ZZZZZZ"` 哨兵值代表 23:59:59
23. **LEFTX vs LEFT** — VFP 中 `LEFT()` 可能在中文字元（Big5 雙位元組）中間截斷，應使用 `LEFTX()` 安全截斷

---

## 臨床工作流查詢食譜

以下是診所日常最常遇到的臨床場景，每個都附帶可直接使用的查詢。

### 場景 A：糖尿病患者逾期未追蹤 HbA1c

「哪些糖尿病患者超過 3 個月沒做 HbA1c？」

```sql
SELECT p.KCSTMR, p.MNAME, p.MBIRTHDT,
       MAX(h.HDATE) AS last_hba1c_date,
       MAX(h.HVAL) AS last_hba1c_value
FROM CO01M p
JOIN CO03L l ON p.KCSTMR = l.KCSTMR AND l.LABNO LIKE 'E11%'
LEFT JOIN CO18H h ON p.KCSTMR = h.KCSTMR AND h.HITEM = 'Z0SHbA1c'
GROUP BY p.KCSTMR, p.MNAME, p.MBIRTHDT
HAVING last_hba1c_date IS NULL
   OR last_hba1c_date < '1141101'  -- 替換為「3個月前的民國年日期」
ORDER BY last_hba1c_date ASC;
```

### 場景 B：高血壓未達標病患清單

「哪些高血壓患者最近血壓仍 ≥ 140/90？」

```sql
SELECT p.KCSTMR, p.MNAME, h.HDATE, h.HVAL,
       CAST(SUBSTR(h.HVAL, 1, INSTR(h.HVAL, '/') - 1) AS INTEGER) AS sbp,
       CAST(SUBSTR(h.HVAL, INSTR(h.HVAL, '/') + 1) AS INTEGER) AS dbp
FROM CO01M p
JOIN CO03L l ON p.KCSTMR = l.KCSTMR AND (l.LABNO LIKE 'I10%' OR l.LABNO LIKE 'I11%')
JOIN CO18H h ON p.KCSTMR = h.KCSTMR AND h.HITEM = 'BP'
WHERE h.HDATE = (SELECT MAX(h2.HDATE) FROM CO18H h2 WHERE h2.KCSTMR = h.KCSTMR AND h2.HITEM = 'BP')
  AND (CAST(SUBSTR(h.HVAL, 1, INSTR(h.HVAL, '/') - 1) AS INTEGER) >= 140
    OR CAST(SUBSTR(h.HVAL, INSTR(h.HVAL, '/') + 1) AS INTEGER) >= 90)
ORDER BY sbp DESC;
```

### 場景 C：CKD 病患 eGFR 惡化追蹤

「哪些 CKD 患者的 eGFR 在過去一年內下降超過 5？」

```sql
SELECT p.KCSTMR, p.MNAME,
       early.HVAL AS egfr_early, early.HDATE AS date_early,
       recent.HVAL AS egfr_recent, recent.HDATE AS date_recent,
       ROUND(CAST(early.HVAL AS REAL) - CAST(recent.HVAL AS REAL), 1) AS decline
FROM CO01M p
JOIN CO18H recent ON p.KCSTMR = recent.KCSTMR AND recent.HITEM = 'Z0SEGFR'
JOIN CO18H early ON p.KCSTMR = early.KCSTMR AND early.HITEM = 'Z0SEGFR'
WHERE recent.HDATE = (SELECT MAX(HDATE) FROM CO18H WHERE KCSTMR = p.KCSTMR AND HITEM = 'Z0SEGFR')
  AND early.HDATE = (SELECT MAX(HDATE) FROM CO18H WHERE KCSTMR = p.KCSTMR AND HITEM = 'Z0SEGFR' AND HDATE < recent.HDATE)
  AND CAST(early.HVAL AS REAL) - CAST(recent.HVAL AS REAL) > 5
ORDER BY decline DESC;
```

### 場景 D：今日門診病患的完整看診摘要

「給我今天每位病人的診斷、處方、檢驗一頁看完」

```sql
SELECT
    o.TARTIME AS 順序,
    o.TNAME AS 姓名,
    o.KCSTMR,
    l.LABNO AS 主診斷,
    l.LABDT AS 診斷名稱,
    GROUP_CONCAT(DISTINCT d.DDESC2 || '(' || m.PFQ || ' x' || m.PTDAY || '天)', ', ') AS 處方
FROM CO05O o
LEFT JOIN CO03L l ON o.KCSTMR = l.KCSTMR AND l.DATE = o.TBKDT
LEFT JOIN CO02M m ON o.KCSTMR = m.KCSTMR AND m.IDATE = o.TBKDT
LEFT JOIN CO09D d ON m.DNO = d.KDRUG OR m.DNO = d.DNO
WHERE o.TBKDT = '1150220'  -- 替換為今日民國年日期
  AND o.TSTS = 'F'         -- 已完診
GROUP BY o.KCSTMR, o.TARTIME
ORDER BY CAST(o.TARTIME AS INTEGER);
```

### 場景 E：庫存精確計算（含期初、進出貨、結存）

「算出本月份每個藥品的進銷存明細」

```sql
-- 本庫目前庫存（排除作廢）
SELECT S.KDRUG, D.DDESC2 AS 藥品名稱, D.DUM1 AS 單位,
       SUM(S.DQTY) AS 目前庫存
FROM CO09S S
JOIN CO09D D ON S.KDRUG = D.KDRUG
WHERE S.DIO <> '@' AND (S.DLCT = 'O' OR S.DLCT = '')
GROUP BY S.KDRUG, D.DDESC2, D.DUM1
HAVING SUM(S.DQTY) <> 0
ORDER BY D.DDESC2;

-- 月份進銷存彙總（含異動類型名稱）
SELECT D.KDRUG, D.DDESC2,
       P.PTITL AS 異動類型,
       CASE WHEN P.PQTY = '+' THEN SUM(A.AQTY) ELSE -SUM(A.AQTY) END AS 異動量,
       SUM(A.ATOTAL) AS 金額
FROM CO10A A
JOIN CO09D D ON A.KDRUG = D.KDRUG
JOIN CO10P P ON A.ATYPE = P.PTYP1
WHERE A.ADATE BETWEEN '1150301' AND '1150331'
GROUP BY D.KDRUG, D.DDESC2, P.PTITL, P.PQTY
ORDER BY D.KDRUG;
```

### 場景 F：某藥品即將過期批次

「哪些藥品在 3 個月內會到期？」

```sql
SELECT d.KDRUG, d.DDESC2, d.DQTY2 AS 庫存量,
       a.AREMARK AS 效期,
       SUBSTR(a.AREMARK, 1, 4) || '/' || SUBSTR(a.AREMARK, 5, 2) || '/' || SUBSTR(a.AREMARK, 7, 2) AS 效期格式
FROM CO10A a
JOIN CO09D d ON a.KDRUG = d.KDRUG
WHERE a.ATYPE = '2'  -- 進貨
  AND LENGTH(a.AREMARK) = 8
  AND a.AREMARK <= '20260520'  -- 替換為 3 個月後的西元日期
  AND a.AREMARK >= '20260220'  -- 替換為今日西元日期
  AND d.DQTY2 > 0
ORDER BY a.AREMARK ASC;
```

---

## 適用情境 → 資料表路由

| 情境 | 核心資料表 |
|------|-----------|
| 糖尿病管理儀表板 | CO03L(診斷) + CO18H(HbA1c/eGFR) + CO02M(照護碼) + CO01M |
| 藥品庫存管理 | CO09D(主檔) + CO09S(餘額) + CO10A(異動) + CO10P(類型) + CO14M(廠商) |
| 存貨月報/進銷存 | CO9Dxxxx(月結檔) + CO09D + CO10A + CO02P(調劑明細) |
| 預防保健追蹤 | VISHFAM → CO01M(MPERSONID=PAT_PID) → CO03L(LISRS) |
| 病歷查詢分析 | CO02H(SOAP, 需合併SATB) + CO02M+CO09D(處方) |
| 門診即時看板 | CO05O(TBKDT=今日) + CO01M + CO03L |
| 預約掛號管理 | CO05B(TBKDT=日期, TSTS排除C/X/D/H) + CO01M(姓名) |
| 健檢結構化明細 | CO18HD2(含單位/參考值/Memo) ←CNORNO→ CO18H ←KCSTMR→ CO01M |
| 醫師/人員查詢 | CO07A(NO) ←LPID→ CO03L(就診記錄) |
| 進貨單管理 | CO10H(單頭) ←KDONO→ CO10A(明細) ←KDRUG→ CO09D |
| 庫存調整管理 | CO11H(單頭) ←ASMNM→ CO11A(明細) ←KDRUG→ CO09D |
| VFP 報表開發/分析 | `references/report-system.md`（F_GTFM 框架 + 模組目錄 + SQL 模式） |

## 成功的樣子

**每次查詢完成後確認：**

1. **筆數合理性**：SELECT COUNT(*) 先確認回傳筆數是否符合預期（不是 0，也不是異常地多）
2. **分段資料完整**：如果查 CO02H 或 CO02F，確認有 GROUP_CONCAT 合併分段
3. **HVAL 數值比較用 CAST**：任何含數值過濾的 CO18H 查詢，都有 CAST(HVAL AS REAL)
4. **日期格式正確**：確認查詢中的日期是民國年格式（7 位），不是西元年

---

## 參考文件

| 文件 | 內容 |
|------|------|
| `references/query-examples.md` | 完整查詢範例（SOAP 合併、儀表板、追蹤率等） |
| `references/data-quality.md` | 資料品質審計查詢（重複、孤兒、日期合理性） |
| `references/performance.md` | 索引建議、Materialized View、分頁、EXPLAIN |
| `references/full-text-search.md` | SOAP Notes 全文搜尋（SQLite FTS5 / PostgreSQL tsvector） |
| `references/inventory-system.md` | 庫存系統完整參考（計算邏輯、系統參數、報表程式、進階查詢） |
| `references/advanced-topics.md` | 資料匿名化指南、VISHFAM 系統 Schema 說明 |
| `references/schema-verified.md` | **14 張表完整欄位定義**（三輪驗證版，含型態、長度、記錄數、ER 圖） |
| `references/report-system.md` | **VFP 報表系統**（F_GTFM 框架、50+ 模組目錄、SQL 模式、函式速查） |
