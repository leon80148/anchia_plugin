---
name: continuous-learning-v2
description: |
  Instinct-based 自動學習系統。透過 hooks 觀察 session 中的工具使用，
  自動偵測模式並產生 instincts（原子化學習行為），附帶信心度評分。
  可將 instincts 進化為 skills、commands 或 agents。
  觸發詞：自動學習、持續學習、instinct、學習機制、模式偵測、
  continuous learning、pattern detection、行為學習
---

# Continuous Learning v2 — Instinct-Based 自動學習系統

進階學習系統，將 Claude Code session 中的活動轉化為可重用知識。
透過原子化「instincts」（小型學習行為 + 信心度評分）實現自動學習。

基於 [everything-claude-code](https://github.com/affaan-m/everything-claude-code) continuous-learning-v2（MIT 授權）。

## 使用時機

- 設定從 Claude Code session 自動學習
- 透過 hooks 設定 instinct-based 行為擷取
- 調整已學習行為的信心度閾值
- 審查、匯出或匯入 instinct 庫
- 將 instincts 進化為完整的 skills、commands 或 agents

## v1 vs v2 比較

| 特性 | v1 | v2 |
|------|----|----|
| 觀察方式 | Stop hook（session 結束） | PreToolUse/PostToolUse（100% 可靠） |
| 分析位置 | 主 context | 背景 agent（Haiku） |
| 粒度 | 完整 skills | 原子化 instincts |
| 信心度 | 無 | 0.3-0.9 加權 |
| 進化路徑 | 直接產生 skill | Instincts -> 叢集 -> skill/command/agent |
| 分享機制 | 無 | 匯出/匯入 instincts |

## Instinct 模型

一個 instinct 是一個小型學習行為：

```yaml
---
id: prefer-functional-style
trigger: "when writing new functions"
confidence: 0.7
domain: "code-style"
source: "session-observation"
---

# Prefer Functional Style

## Action
Use functional patterns over classes when appropriate.

## Evidence
- Observed 5 instances of functional pattern preference
- User corrected class-based approach to functional on 2025-01-15
```

**特性：**
- **原子化** — 一個觸發條件，一個動作
- **信心度加權** — 0.3 = 暫定, 0.9 = 近乎確定
- **領域標籤** — code-style, testing, git, debugging, workflow 等
- **證據支持** — 追蹤哪些觀察產生了它

## 運作流程

```
Session 活動
      |
      | Hooks 捕捉 prompts + 工具使用（100% 可靠）
      v
+---------------------------------------------+
|         observations.jsonl                   |
|   （prompts, tool calls, outcomes）          |
+---------------------------------------------+
      |
      | Observer agent 讀取（背景, Haiku）
      v
+---------------------------------------------+
|          模式偵測                             |
|   - 使用者修正 -> instinct                   |
|   - 錯誤解決 -> instinct                     |
|   - 重複工作流 -> instinct                   |
+---------------------------------------------+
      |
      | 建立/更新
      v
+---------------------------------------------+
|         instincts/personal/                  |
|   - prefer-functional.md (0.7)               |
|   - always-test-first.md (0.9)               |
|   - use-zod-validation.md (0.6)              |
+---------------------------------------------+
      |
      | /evolve 叢集化
      v
+---------------------------------------------+
|              evolved/                        |
|   - commands/new-feature.md                  |
|   - skills/testing-workflow.md               |
|   - agents/refactor-specialist.md            |
+---------------------------------------------+
```

## 快速開始

### 1. 安裝插件

```bash
# 透過 marketplace 安裝
/plugin install continuous-learning@leon-claude-plugins

# 或本地測試
claude --plugin-dir ./continuous-learning
```

安裝後，hooks 會自動註冊（透過 plugin.json）。

### 2. 初始化目錄結構

Python CLI 會自動建立，也可手動建立：

```bash
mkdir -p ~/.claude/homunculus/{instincts/{personal,inherited},evolved/{agents,skills,commands}}
touch ~/.claude/homunculus/observations.jsonl
```

### 3. 匯入種子 Instincts（選用）

本插件包含針對 plugin marketplace 專案的種子 instincts：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/scripts/instinct-cli.py" import "${CLAUDE_PLUGIN_ROOT}/seeds/plugin-marketplace.yaml" --force
```

### 4. 使用指令

```
/instinct-status     # 顯示已學習 instincts 及信心度
/evolve              # 叢集化相關 instincts 為 skills/commands
/instinct-export     # 匯出 instincts 供分享
/instinct-import     # 從他人匯入 instincts
```

## 指令一覽

| 指令 | 說明 |
|------|------|
| `/instinct-status` | 顯示所有已學習 instincts 及信心度 |
| `/evolve` | 叢集化相關 instincts 為 skills/commands/agents |
| `/instinct-export` | 匯出 instincts 供分享 |
| `/instinct-import <file>` | 從他人匯入 instincts |

## 設定

編輯 `~/.claude/homunculus/config.json`（選用）：

```json
{
  "version": "2.0",
  "observation": {
    "enabled": true,
    "store_path": "~/.claude/homunculus/observations.jsonl",
    "max_file_size_mb": 10,
    "archive_after_days": 7
  },
  "instincts": {
    "personal_path": "~/.claude/homunculus/instincts/personal/",
    "inherited_path": "~/.claude/homunculus/instincts/inherited/",
    "min_confidence": 0.3,
    "auto_approve_threshold": 0.7,
    "confidence_decay_rate": 0.05
  },
  "observer": {
    "enabled": true,
    "model": "haiku",
    "run_interval_minutes": 5,
    "patterns_to_detect": [
      "user_corrections",
      "error_resolutions",
      "repeated_workflows",
      "tool_preferences"
    ]
  },
  "evolution": {
    "cluster_threshold": 3,
    "evolved_path": "~/.claude/homunculus/evolved/"
  }
}
```

## 檔案結構

```
~/.claude/homunculus/
  identity.json           # 使用者 profile、技術層級
  observations.jsonl      # 當前 session 觀察紀錄
  observations.archive/   # 已處理的觀察紀錄
  instincts/
    personal/             # 自動學習的 instincts
    inherited/            # 從他人匯入的 instincts
  evolved/
    agents/               # 產生的專家 agents
    skills/               # 產生的 skills
    commands/             # 產生的 commands
```

## 信心度評分

信心度隨時間演化：

| 分數 | 含義 | 行為 |
|------|------|------|
| 0.3 | 暫定 | 建議但不強制 |
| 0.5 | 中等 | 相關時套用 |
| 0.7 | 強 | 自動核准套用 |
| 0.9 | 近乎確定 | 核心行為 |

**信心度提升**：
- 模式被重複觀察到
- 使用者未修正建議的行為
- 來自其他來源的相似 instincts 一致

**信心度下降**：
- 使用者明確修正行為
- 長時間未觀察到模式
- 出現矛盾證據

## 好 Instinct vs 壞 Instinct（具體對比）

### 好的 Instinct ✅

```yaml
id: use-group-concat-for-co02h
trigger: "when querying CO02H SOAP notes"
confidence: 0.8
domain: "database-query"
---
# Always GROUP_CONCAT CO02H records by SATB

## Action
When querying CO02H, always use GROUP_CONCAT(STEXT, '' ORDER BY SATB)
to merge segmented SOAP notes. Never return just SATB='1'.

## Evidence
- User corrected a query that only returned first segment (2025-01-15)
- Same pattern observed in 3 subsequent database sessions
- Confirmed in vision-database SKILL.md gotchas section
```

**為什麼好**：觸發條件具體（查 CO02H 時）、動作明確（用 GROUP_CONCAT）、有反例（不要只取 SATB='1'）、證據來自多次觀察。

### 壞的 Instinct ❌

```yaml
id: write-better-code
trigger: "when writing code"
confidence: 0.6
domain: "general"
---
# Write Better Code

## Action
Always write clean, maintainable code.

## Evidence
- General observation from multiple sessions
```

**為什麼壞**：觸發條件太泛（寫任何程式碼時）、動作是空話（「寫好的程式碼」誰不想？）、沒有具體的「做什麼」、證據模糊。

### 判斷公式

一個 instinct 值得保留，當且僅當：
1. 你能想到**它不適用的具體情況**（有邊界 = 有意義）
2. 另一個人看到它會知道**要改變什麼行為**（可執行）
3. 它的觸發條件**不超過一句話**（太長 = 太複雜，應該拆成多個）

---

## 為何用 Hooks 而非 Skills 觀察？

v1 依賴 skills 來觀察。Skills 是機率性的 — 基於 Claude 的判斷約 50-80% 觸發。

Hooks **100% 觸發**，確定性的。這意味著：
- 每個工具呼叫都被觀察
- 不會遺漏任何模式
- 學習是全面的

## 停用觀察

若需暫時停用觀察（例如處理敏感資料）：

```bash
touch ~/.claude/homunculus/disabled    # 停用
rm ~/.claude/homunculus/disabled       # 重新啟用
```

## 隱私說明

- 觀察紀錄 **僅存在本地** 機器上
- 僅 **instincts**（模式）可被匯出
- 不分享任何實際程式碼或對話內容
- 你完全控制匯出的內容

## 故障排除

### 觀察紀錄未寫入
1. 確認 hooks 已在 `settings.json` 或 plugin.json 中設定
2. 檢查 `~/.claude/homunculus/` 目錄是否存在
3. 確認未建立 `disabled` 檔案
4. Windows 上確認 python 指令可用

### Observer 未啟動
1. 執行 `bash start-observer.sh status` 檢查狀態
2. 檢查 `~/.claude/homunculus/observer.log` 中的錯誤
3. 確認 `claude` CLI 已安裝

### Instincts 未產生
1. 確認累積足夠觀察（10+）
2. 執行 `/instinct-status` 檢查現有 instincts
3. 手動執行 observer 分析

## 邊界與失敗模式

**學習系統本身的限制，誠實面對：**

| 情況 | 為什麼失敗 | 怎麼辨認 |
|------|-----------|---------|
| 觀察數量不足（< 10 條） | Observer 沒有足夠的模式樣本，instinct 品質低 | 生成的 instinct 過於籠統或不準確 |
| 使用者的行為本身不一致 | 學習到的是矛盾的模式 | 兩個 instinct 互相衝突 |
| 信心度過高但實際上不可靠 | 觀察量少但重複多，系統誤以為是強模式 | 驗證一個高信心 instinct 實際上不適用 |
| Hooks 沒有觸發 | settings.json 設定錯誤，或 disabled 檔案存在 | observations.jsonl 沒有新內容 |
| 觀察到的是異常工作流，而不是一般模式 | 某次特殊任務產生了錯誤的 instinct | Instinct 只在很特定的情況下適用 |

**這個系統不能替代的事：**
- 明確的使用者指令（CLAUDE.md 中的規則比 instinct 優先）
- 對話記憶（instincts 是模式，不是特定對話的記憶）
- 你主動思考什麼值得學習（系統觀察，但你來裁決）

## 成功的樣子

**一個有效的 instinct 長這樣：**

1. **觸發條件明確**：「當我在做 X 的時候」而不是「大多數時候」
2. **動作具體**：「用 Y 而不是 Z」而不是「要做得更好」
3. **有反例**：知道什麼情況下這個 instinct 不適用
4. **信心度和實際使用次數相符**：如果你只看過一次，信心度不應超過 0.4

**系統健康指標：**
- 每週至少有 5 條新觀察
- Instincts 目錄有實際在成長
- 執行 `/instinct-status` 時，至少有一條 instinct 信心度 ≥ 0.7

---

## 相關連結

- [everything-claude-code](https://github.com/affaan-m/everything-claude-code) - 原始 v2 架構來源
- [Homunculus](https://github.com/humanplane/homunculus) - v2 架構靈感

---

*Instinct-based learning: 教導 Claude 你的模式，一次一個觀察。*
