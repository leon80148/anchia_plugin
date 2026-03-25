# Validation Matrix

各模型 × 性別分支的必填欄位速查矩陣。

## Global Rule

`age > 0` is required for **every** model, both genders.

## 總覽矩陣

`●` = 必填，`○` = 不需要，`◐` = 僅此性別必填

| 欄位 | CHD M | CHD F | Stroke M | Stroke F | HTN M | HTN F | DM M | DM F | MACE M | MACE F |
|------|-------|-------|----------|----------|-------|-------|------|------|--------|--------|
| age | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| sbp | ○ | ◐ | ● | ● | ● | ● | ○ | ○ | ● | ● |
| chol | ○ | ◐ | ○ | ○ | ○ | ○ | ◐ | ○ | ○ | ◐ |
| hdlc | ● | ● | ○ | ○ | ◐ | ○ | ○ | ◐ | ● | ● |
| ldlc | ○ | ○ | ○ | ○ | ◐ | ○ | ○ | ○ | ○ | ○ |
| tg | ○ | ◐ | ◐ | ○ | ○ | ◐ | ● | ● | ○ | ○ |
| glu | ○ | ○ | ◐ | ○ | ○ | ◐ | ● | ● | ○ | ○ |
| waist | ● | ● | ○ | ◐ | ○ | ○ | ○ | ◐ | ● | ● |
| bmi* | ○ | ○ | ○ | ○ | ● | ● | ● | ● | ○ | ○ |

`*` bmi: `bmi > 0` OR (`height > 0` AND `weight > 0`)

## 各模型詳細規則

### CHD
- Base required: `hdlc > 0`, `waist > 0`
- Female branch (`gender != 1`) also requires: `sbp > 0`, `chol > 0`, `tg > 0`

### Stroke
- Base required: `sbp > 0`
- Male branch (`gender == 1`) also requires: `glu > 0`, `tg > 0`
- Female branch (`gender != 1`) also requires: `waist > 0`

### Hypertension
- Base required: `sbp > 0`
- Body composition rule: `bmi > 0` OR (`height > 0` AND `weight > 0`)
- Male branch (`gender == 1`) also requires: `hdlc > 0`, `ldlc > 0`
- Female branch (`gender != 1`) also requires: `tg > 0`, `glu > 0`

### Diabetes
- Base required: `glu > 0`, `tg > 0`
- Body composition rule: `bmi > 0` OR (`height > 0` AND `weight > 0`)
- Male branch (`gender == 1`) also requires: `chol > 0`
- Female branch (`gender != 1`) also requires: `waist > 0`, `hdlc > 0`

### MACE
- Base required: `sbp > 0`, `hdlc > 0`, `waist > 0`
- Female branch (`gender != 1`) also requires: `chol > 0`

## 驗證範例

### 合格輸入（CHD 男性）

```json
{"gender": 1, "age": 52, "hdlc": 45, "waist": 88}
```
→ `status: 0`（通過：age、hdlc、waist 都 > 0）

### 不合格輸入（CHD 男性缺 waist）

```json
{"gender": 1, "age": 52, "hdlc": 45}
```
→ `status: 1`（失敗：waist 缺失或為 0）

### 不合格輸入（Hypertension 女性缺 glu）

```json
{"gender": 0, "age": 45, "sbp": 130, "tg": 150, "bmi": 24}
```
→ `status: 1`（失敗：female HTN 需要 `glu > 0`）

## Notes

- Legacy Java validation updates error text but does not stop at first failure（歷史遺留行為）。
- For compatibility, keep field thresholds strict (`> 0`, not `>= 0`).
- 建議：API adapter 的 400 錯誤訊息中列出所有缺少的欄位，幫助使用者一次修正。
