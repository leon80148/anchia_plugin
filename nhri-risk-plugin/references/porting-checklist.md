# Porting Checklist

將 NHRI V4 邏輯從 Java/Python 參考實作移植到其他語言時的檢查清單。

## 移植步驟（依序執行）

### 1. Gender 分支規則
- [x] `gender == 1` → male branch
- [x] `gender != 1`（所有非 1 的值）→ female branch
- **常見錯誤**：只處理 `gender == 0` 為 female，忘記 `gender == 2` 等值也走 female。

### 2. 驗證門檻
- [x] 所有必填欄位用 `> 0` 檢查（strict greater than，不是 `>= 0`）
- [x] 驗證失敗時設 `status = 1` 並立即返回
- **常見錯誤**：用 `>= 0` 導致 0 值通過驗證。

### 3. Rounding 行為（最常出錯的地方）
- [x] 使用 Java-compatible rounding（round half up），不是 bankers rounding（round half even）
- [x] 風險值保留 2 位小數

不同語言的 rounding 行為：

| 語言 | 預設 round() 行為 | 需要修正？ |
|------|------------------|-----------|
| Java | Round half up | 否（基準） |
| Python | Round half even (bankers) | **是** — 用 `Decimal` + `ROUND_HALF_UP` |
| JavaScript | Round half up | 否 |
| C# | Round half even (bankers) | **是** — 用 `MidpointRounding.AwayFromZero` |
| Go | 無內建 round | **是** — 自行實作 `math.Floor(x*100+0.5)/100` |

```python
# Python 正確的 rounding
from decimal import Decimal, ROUND_HALF_UP
def java_round_2(value):
    return float(Decimal(str(value)).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP))
```

### 4. 計算順序（不可更改）
```
1. validate     → 檢查必填欄位
2. set version  → version = 4
3. derive BMI   → 若 bmi <= 0 且 height > 0 且 weight > 0
4. compute Z    → 套用公式係數
5. compute risk → 1 - S0^(exp(Z))，轉為百分比
6. compute populationAvg → age 35-70 查表
7. compute multipleDiff  → risk / populationAvg * 100 再 round
8. compute riskType      → 依閾值分類
```

### 5. Population Average 年齡窗口
- [x] `35 <= age <= 70` 時查表（`avgArray[age - 35]`）
- [x] 超出範圍時 `populationAvg = 0`、`multipleDiff = 0`
- **常見錯誤**：off-by-one，用 `age - 34` 或不含 70。

### 6. Risk Type 閾值
- [x] CHD/Stroke/Diabetes/MACE：`risk >= 20 → 2`，`risk >= 10 → 1`，其他 → 0
- [x] Hypertension：`multipleDiff > 1.25 → 2`，`multipleDiff >= 0.75 → 1`，其他 → 0

### 7. 輸出欄位名稱
- [x] 保持不變：`model`, `status`, `risk`, `populationAvg`, `multipleDiff`, `riskType`, `version`, `band`
- **常見錯誤**：把 `populationAvg` 改成 `population_avg`（snake_case）。

### 8. 回歸測試
- [x] 移植完成後執行回歸測試（見 regression-workflow.md）
- [x] 所有 100 筆黃金向量通過

## 移植簽核

完成以上所有項目後，確認：
- [ ] 回歸測試 100% 通過
- [ ] 數值欄位容許差異 < 1e-9
- [ ] 所有 5 models × 2 genders = 10 分支都有覆蓋
- [ ] Edge case（age=34, age=71, bmi=0 with height/weight）通過
