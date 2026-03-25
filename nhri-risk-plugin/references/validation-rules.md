# NHRI V4 Validation Rules

各模型的詳細驗證規則，包含邊界案例和實作注意事項。

與 `validation-matrix.md` 的差異：matrix 是速查表，本文件是完整規則說明。

## Global Rule

`age > 0` is mandatory for all models, both genders.

age 的合理範圍建議：1-120（超出此範圍公式仍會計算，但結果無臨床意義）。

## CHD (Coronary Heart Disease)

- Required for both branches: `hdlc > 0`, `waist > 0`
- Female branch (`gender != 1`) requires additional: `sbp > 0`, `chol > 0`, `tg > 0`

Male CHD 只需 3 個欄位（age, hdlc, waist），是所有模型中最簡單的。

## Stroke

- Required for both branches: `sbp > 0`
- Male branch (`gender == 1`) requires additional: `glu > 0`, `tg > 0`
- Female branch (`gender != 1`) requires additional: `waist > 0`

注意：Stroke male 和 female 的額外欄位完全不同。

## Hypertension

- Required for both branches: `sbp > 0`
- Also require one of:
  - `bmi > 0`, or
  - both `height > 0` and `weight > 0`
- Male branch (`gender == 1`) requires additional: `hdlc > 0`, `ldlc > 0`
- Female branch (`gender != 1`) requires additional: `tg > 0`, `glu > 0`

BMI 二擇一規則（body composition rule）：接受直接提供 BMI，或從身高體重推算。
推算公式：`bmi = weight / (height/100)^2`。

## Diabetes

- Required for both branches: `glu > 0`, `tg > 0`
- Also require one of:
  - `bmi > 0`, or
  - both `height > 0` and `weight > 0`
- Male branch (`gender == 1`) requires additional: `chol > 0`
- Female branch (`gender != 1`) requires additional: `waist > 0`, `hdlc > 0`

Diabetes 和 Hypertension 共用 body composition rule。

## MACE (Major Adverse Cardiovascular Events)

- Required for both branches: `sbp > 0`, `hdlc > 0`, `waist > 0`
- Female branch (`gender != 1`) requires additional: `chol > 0`

MACE base 需要最多欄位（3 個），但性別差異最小。

## 驗證行為規則

### 失敗處理

- 驗證失敗時設定 `status = 1` 並立即返回。
- Legacy Java 行為：更新 error text 但不會在第一個失敗處停止（收集所有錯誤）。
- 建議新實作也收集所有錯誤，方便使用者一次修正。

### 門檻值

- 所有欄位使用 strict `> 0` 檢查。
- **不是** `>= 0`：值恰好為 0 應視為「未提供」而非「合法的零值」。
- 臨床上，sbp=0 或 hdlc=0 不存在，所以 `> 0` 是正確的。

### 邊界測試案例

| 案例 | 預期結果 | 說明 |
|------|---------|------|
| `age: 0` | status=1 | age=0 不合法 |
| `hdlc: 0.001` | 通過 | 極小正數仍然 > 0 |
| `waist: -5` | status=1 | 負數不通過 > 0 |
| `bmi: 0, height: 170, weight: 65` | 通過 | body composition rule 的第二條件成立 |
| `bmi: 0, height: 170, weight: 0` | status=1 | 第二條件不成立（weight=0） |
| `gender: 2, age: 50, hdlc: 45, waist: 80` | CHD 通過 | gender=2 走 female branch，但 CHD base 已滿足 |
