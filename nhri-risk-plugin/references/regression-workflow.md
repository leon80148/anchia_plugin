# Regression Workflow

將 NHRI V4 邏輯移植到另一個技術棧時，用回歸測試確保行為一致。

## Goal

Detect behavior drift when NHRI V4 logic is moved to another stack.
核心原則：參考實作（Python）的輸出是 ground truth，所有移植版本必須精確匹配。

## Workflow Steps

### Step 1: 產生黃金向量（Golden Vectors）

```bash
python scripts/run_module.py --module regress-generate -- \
  --per-group 10 \
  --output assets/golden-vectors.json
```

這會為每個 `model × gender` 組合產生 10 筆測試案例，共 100 筆向量。

### Step 2: 在目標實作中執行向量

將 `golden-vectors.json` 餵入你的移植版本，產出 `actual-output.json`。
格式必須與參考輸出一致（相同欄位名、相同結構）。

### Step 3: 比對結果

```bash
python scripts/run_module.py --module regress-compare -- \
  --expected assets/golden-vectors.json \
  --actual actual-output.json
```

### Step 4: CI 整合

```yaml
# GitHub Actions 範例
- name: NHRI Regression Test
  run: |
    python scripts/run_module.py --module regress-compare -- \
      --expected assets/golden-vectors.json \
      --actual actual-output.json
  # 任何不匹配都會回傳 exit code 1
```

## Coverage Guidance — 必要的測試邊界

| 維度 | 必測值 | 為什麼重要 |
|------|--------|-----------|
| Gender | male (1), female (0) | 每個模型有不同的性別分支 |
| Age boundaries | 34, 35, 70, 71 | 35-70 有 population average，邊界外則為 0 |
| BMI 來源 | 直接提供 bmi vs 從 height+weight 推算 | 推算邏輯是常見的移植漏洞 |
| 極端值 | sbp=280, glu=400, hdlc=15 | 測試是否溢出或精度丟失 |
| 剛好為 0 的欄位 | waist=0（CHD 必填）| 驗證邏輯是否正確拒絕 |
| 全欄位提供 | 所有欄位都有值 | 測試「最完整」的 happy path |
| 最少欄位 | 只提供每個模型的必填欄位 | 測試「最精簡」的 happy path |

## Acceptance Rules

| 欄位 | 比對方式 | 容許差異 |
|------|---------|---------|
| `status` | exact match | 0 |
| `riskType` | exact match | 0 |
| `version` | exact match | 0 |
| `risk` | numeric tolerance | `1e-9` |
| `populationAvg` | numeric tolerance | `1e-9` |
| `multipleDiff` | numeric tolerance | `1e-9` |
| `band` | exact match | — |

容許 `1e-9` 是為了跨語言浮點數表示的微小差異。如果差異 > `1e-4`，通常表示演算法有實質錯誤。

## CI Recommendation

- **平時 CI**：只跑 `regress-compare`，向量檔 committed 在 repo 中。
- **基準更新時**（如修改公式係數）：重新跑 `regress-generate`，review diff 後 commit 新向量。
- **PR Gate**：regression test 失敗時阻擋 merge。

## 常見 Regression 失敗原因

| 失敗模式 | 症狀 | 修復方向 |
|----------|------|---------|
| Rounding 差異 | risk 差 0.01 | 檢查是否使用 Java-compatible rounding |
| 性別分支錯誤 | 只有 female 案例失敗 | 檢查 `gender != 1`（非 `gender == 0`） |
| BMI 推算遺漏 | Hypertension/Diabetes 結果差異 | 檢查 `bmi <= 0 && height > 0 && weight > 0` 的 derive 邏輯 |
| Population avg 邊界 | age=34 或 71 的 multipleDiff 不同 | 確認 age 範圍是 `35..70` inclusive |
