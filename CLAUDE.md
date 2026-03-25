# CLAUDE.md — Plugin Marketplace 專案記憶

## 專案概述

這是一個 Claude Code 插件市集（Plugin Marketplace），包含多個可獨立安裝的 skills、agents 和 commands。每個插件幫助 Claude Code 在特定領域提供更好的輔助。

## 目錄結構慣例

本專案存在兩種目錄結構，各自保持一致：

### Nested 結構（skills 類插件）

```
plugin-name/
├── skills/
│   └── plugin-name/
│       ├── SKILL.md          # 主文件（必須）
│       └── references/       # 參考文件（選填）
│           ├── ref1.md
│           └── ref2.md
└── .claude-plugin/
    └── plugin.json           # 插件元資料（選填）
```

適用：health-education-writer, seo-geo-optimizer, vision-database

### Flat 結構（系統整合類插件）

```
plugin-name/
├── SKILL.md                  # 主文件（必須）
├── references/               # 參考文件（選填）
├── scripts/                  # 腳本（選填）
├── assets/                   # 模板/資源（選填）
└── .claude-plugin/
    └── plugin.json           # 插件元資料（選填）
```

適用：nhi-ic-card, nhri-risk-plugin

### Multi-Agent 結構

```
plugin-name/
├── SKILL.md                  # 主文件（建議加上）
├── agents/                   # Agent 定義
│   └── agent_name.md
├── commands/                 # 指令定義
│   └── command_name.md
└── references/               # 共享參考
```

適用：medical-content-team

### Full-Stack 結構（含 hooks + agents + commands + scripts）

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json           # 插件元資料 + hooks 自動註冊
├── skills/
│   └── plugin-name/
│       └── SKILL.md          # 主文件（必須）
├── agents/                   # Agent 定義
├── commands/                 # 指令定義
├── hooks/                    # Hook 腳本
├── scripts/                  # CLI 工具
└── seeds/                    # 種子資料
```

適用：continuous-learning

## SKILL.md 撰寫規範

### 必要元素

1. **YAML Frontmatter**：
   - `name`：插件名稱（kebab-case）
   - `description`：多行描述，包含觸發時機和觸發詞（必須含繁體中文觸發詞）

2. **內容結構**：
   - 概述/使用時機
   - 核心功能說明
   - 工作流程/決策樹
   - 範例/模板
   - 參考文件索引

### 品質規則

- **行數指引**：SKILL.md 目標 150-600 行
- **參考檔案**：每個檔案至少 50 行，禁止建立空的 stub 檔案
- **檔案路徑**：SKILL.md 中引用的所有參考檔案路徑必須實際存在
- **觸發詞**：description 欄位必須包含繁體中文觸發詞
- **錯誤處理**：系統整合類插件（nhi-ic-card, nhri-risk-plugin）必須包含錯誤處理/故障排除段落

### 語言慣例

- 主要語言：繁體中文
- 技術術語保持英文原文
- 程式碼和指令使用英文

## 插件清單

| 插件 | 結構 | 領域 | 說明 |
|------|------|------|------|
| health-education-writer | nested | 內容 | 多領域教育科普文章生成器 |
| medical-content-team | multi-agent | 內容 | 6 Agent 協作內容審查團隊 |
| nhi-ic-card | flat | 醫療系統 | 健保IC卡讀卡整合 |
| nhri-risk-plugin | flat | 醫療系統 | 國衛院健康風險評估 |
| vision-database | nested | 醫療系統 | 展望 HIS 資料庫查詢 |
| seo-geo-optimizer | nested | 行銷 | SEO 與 GEO（生成式引擎最佳化）整合策略工具 |
| continuous-learning | full-stack | 基礎設施 | Instinct-based 自動學習系統（hooks + observer） |

## 開發注意事項

### Windows 環境

- 本專案在 Windows 環境開發
- Shell 使用 bash（Git Bash）
- 路徑使用正斜線（forward slashes）
- 建立新檔案時使用 Write tool，不使用 shell 的 heredoc（Windows 相容性）

### Git 慣例

- 主分支：main
- Commit message 語言：英文
- 每次改動應包含明確的 scope（如 `feat(nhi-ic-card): add virtual card guide`）

## 自動學習機制

本專案包含 `continuous-learning` 插件，提供 instinct-based 自動學習能力：

- **Hooks 觀察**：PreToolUse/PostToolUse hooks 自動捕捉每次工具使用
- **Instincts**：原子化學習行為，附帶信心度評分（0.3-0.9）
- **進化**：累積的 instincts 可叢集化為 skills/commands/agents
- **種子資料**：`continuous-learning/seeds/plugin-marketplace.yaml` 包含 7 條從前兩輪迭代學到的經驗
- **指令**：`/instinct-status`、`/evolve`、`/instinct-export`、`/instinct-import`

## 從迭代學到的經驗

以下經驗已轉為 instincts（見 `continuous-learning/seeds/plugin-marketplace.yaml`）：

1. 禁止建立空的 stub 參考檔案（每個 .md >= 50 行）
2. 提交前必須驗證 SKILL.md 引用的所有路徑實際存在
3. Multi-Agent 結構也需要 SKILL.md
4. 系統整合插件必須有 troubleshooting 內容
5. SKILL.md 目標 150-600 行
6. description 欄位必須含繁體中文觸發詞
7. Windows 上建立 .md 檔案需使用 Python（避開 hook 限制）
