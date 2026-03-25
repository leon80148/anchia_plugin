# Evaluator Algorithm

NHRI V4 風險評估的完整演算法流程，從輸入到輸出。

係數和 S0 值的具體數字請查 `formulas.md`，本文件專注在流程邏輯。

## 演算法流程圖

```
Input JSON
    │
    ▼
[1. Normalize] → parse to float, missing → 0, gender branch
    │
    ▼
[2. Validate] → check required fields (> 0)
    │ fail → return {status:1}
    ▼
[3. Set version=4]
    │
    ▼
[4. Derive BMI] → only for HTN/DM, only if bmi<=0 and h>0 and w>0
    │
    ▼
[5. Compute Z and Risk] → Z = Σ(βi × xi), risk = (1 - S0^exp(Z)) × 100
    │
    ▼
[6. Population Avg] → age 35-70: lookup table, else 0
    │
    ▼
[7. Multiple Diff] → popAvg>0: (risk/popAvg×100) rounded, else 0
    │
    ▼
[8. Risk Type] → threshold classification
    │
    ▼
[9. Output] → complete result object
```

## Step 1: Normalize Input

```python
def normalize(raw_input):
    result = {}
    for field in ALL_FIELDS:
        val = raw_input.get(field, 0)
        result[field] = float(val) if val is not None else 0.0
    # Gender branching
    result['is_male'] = (int(result['gender']) == 1)
    return result
```

- Missing 欄位視為 `0`（不是 null、不是 NaN）。
- `gender == 1` 是唯一走 male branch 的條件。

## Step 2: Validate Model Inputs

- 套用各模型的必填欄位規則（見 validation-rules.md）。
- 任何必填欄位 `<= 0` → `status=1`，立即返回。
- 不繼續計算——避免產出無意義的風險值。

## Step 3: Set Version

- 只在驗證通過後設定 `version = 4`。
- 這讓呼叫端可以用 `version` 欄位存在與否判斷評估是否完成。

## Step 4: Compute BMI If Needed

只有 Hypertension 和 Diabetes 需要 BMI：

```python
if model in ('hypertension', 'diabetes'):
    if bmi <= 0 and height > 0 and weight > 0:
        bmi = weight / ((height / 100) ** 2)
```

注意：BMI 推算只在 `bmi <= 0` 時觸發。如果使用者直接提供了 bmi > 0，就使用提供的值。

## Step 5: Compute Z and Risk

核心公式：

```
Z = β1×x1 + β2×x2 + ... + βn×xn
risk = (1 - S0^(exp(Z))) × 100
```

- `βi`：各欄位的迴歸係數（從 formulas.md 查表）
- `S0`：baseline survival probability（從 formulas.md 查表）
- 結果轉為百分比，Java-compatible 2-decimal rounding

```python
import math
z = sum(coeff * value for coeff, value in zip(betas, inputs))
risk_raw = (1 - math.pow(s0, math.exp(z))) * 100
risk = java_round_2(risk_raw)  # 見 porting-checklist.md
```

## Step 6: Compute Population Average

```python
if 35 <= age <= 70:
    pop_avg_raw = avg_array[age - 35] * 100  # 查表並轉百分比
    pop_avg = java_round_2(pop_avg_raw)
else:
    pop_avg = 0.0
```

`avg_array` 的長度固定為 36（age 35 到 70，inclusive）。

## Step 7: Compute Multiple Difference

```python
if pop_avg > 0:
    multiple_diff = round((risk / pop_avg) * 100) / 100
else:
    multiple_diff = 0.0
```

注意這裡的 `round()` 用 Java-compatible rounding。

## Step 8: Risk Type

| 模型 | 條件 | riskType |
|------|------|----------|
| CHD/Stroke/Diabetes/MACE | risk >= 20 | 2 |
| CHD/Stroke/Diabetes/MACE | risk >= 10 | 1 |
| CHD/Stroke/Diabetes/MACE | risk < 10 | 0 |
| Hypertension | multipleDiff > 1.25 | 2 |
| Hypertension | multipleDiff >= 0.75 | 1 |
| Hypertension | multipleDiff < 0.75 | 0 |

## Step 9: Output

```json
{
  "model": "chd",
  "status": 0,
  "risk": 15.23,
  "populationAvg": 8.41,
  "multipleDiff": 1.81,
  "riskType": 1,
  "version": 4,
  "band": "elevated"
}
```

`band` 由 riskType 映射：`2 → "high"`, `1 → "elevated"`, `0 → "lower"`。
