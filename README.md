# Leon's Claude Code Plugins

Claude Code 插件市集 — 專為台灣醫療診所、內容創作者和行銷人員打造的工具集。每個插件可獨立安裝，幫助 Claude Code 在特定領域提供更精準的輔助。

## Plugins

### 醫療系統整合

#### `nhi-ic-card` — 健保IC卡讀卡整合

整合台灣健保IC卡讀卡系統的完整開發知識。涵蓋 BhpNhi.dll / CsHis.dll API 呼叫方式、回應碼解碼、成人健檢與 BC 肝篩檢的工作流程、IC卡 12 區欄位結構、以及資料上傳格式 2.0 的 XML 規範。支援 .NET、Node.js、Python 等多語言整合方案，也包含虛擬健保卡（HCA）和跨平台讀卡機整合指引。

#### `nhri-risk-plugin` — 國衛院 NHRI V4 健康風險評估

國家衛生研究院 V4 版健康風險評估系統的完整實作參考。支援五大疾病風險計算：冠心病（CHD）、中風、高血壓、糖尿病、MACE。提供單機元件版與國健署 Web API 版兩種整合路線，包含風險公式係數查表、輸入驗證規則、跨語言回歸測試向量、API 適配器（FastAPI / Express / Spring / Go / Rust / .NET）腳手架產生、以及風險報告渲染功能。

#### `vision-database` — 展望醫療系統 HIS 資料庫查詢

展望診所 HIS 系統的完整資料庫知識庫。能根據需求定位正確的資料表、欄位與代碼，產出可執行的 SQLite / PostgreSQL 查詢語句。涵蓋病患主檔（CO01M）、處方記錄（CO02M/H/P）、檢驗數值（CO03L/M）、掛號、藥品庫存進銷存、預防保健、慢性病照護追蹤等模組，以及 VFP 報表系統語法參考。

---

### 內容創作

#### `health-education-writer` — 多領域教育科普文章生成器

生成有溫度且專業可信的教育科普文章。支援七大領域：醫療衛教（預設）、理財教育、法律常識、營養知識、科技安全、育兒教養、自訂領域。每篇文章自動包含警示訊號、自我照護/行動建議，並可輸出多種平台格式：部落格、社群貼文、電子報、Podcast 逐字稿、LinkedIn 文章、Twitter/X 串文。

#### `medical-content-team` — 醫療內容創作 Agent 團隊

由 6 位 AI Agent 組成的虛擬內容審查團隊，透過 4 輪結構化討論流程產出高品質專業內容。團隊包含：主持人（流程把關）、醫學記者（撰稿執行）、科學守門人（事實查核）、病患代言人（可讀性審查）、魔鬼代言人（批判與偏見檢測）、流量獵人（社群傳播優化）。支援醫療、科技、財經、法律、行銷五大領域的內容審查。

---

### 行銷工具

#### `seo-geo-optimizer` — SEO 與 GEO 整合策略工具

基於普林斯頓大學 GEO-bench 研究成果與 2025-2026 年最新實踐的整合策略框架。提供四大工作模式：內容分析、內容撰寫、技術實作、策略規劃。幫助網站內容同時在傳統搜尋引擎和 AI 生成式引擎（ChatGPT、Perplexity、Gemini、Claude、Google AI Overviews）中獲得最大能見度。涵蓋 llms.txt 建立、Schema JSON-LD 標記、知識圖譜最佳化（KGO）、E-E-A-T 信號強化、AI 能見度追蹤等實務操作。

---

### 基礎設施

#### `continuous-learning` — Instinct-Based 自動學習系統

透過 hooks 自動觀察 Claude Code session 中的工具使用，偵測行為模式並產生 instincts（原子化學習行為 + 信心度評分）。累積的 instincts 可進化為完整的 skills、commands 或 agents。基於 [everything-claude-code](https://github.com/affaan-m/everything-claude-code) continuous-learning-v2（MIT 授權）。

---

## 安裝方式

```bash
# 安裝想要的 plugin
claude plugin install health-education-writer --marketplace leon80148/anchia_plugin
claude plugin install nhi-ic-card --marketplace leon80148/anchia_plugin
# ... 其他 plugin 同理
```

## 本地測試

```bash
# 測試單一 plugin
claude --plugin-dir ./health-education-writer
claude --plugin-dir ./medical-content-team
```

## 推薦的其他 Plugin Marketplaces

| Marketplace | 說明 | 安裝指令 |
|---|---|---|
| **Claude Official Plugins** | Anthropic 官方插件 | `/plugin marketplace add anthropics/claude-plugins-official` |
| **Everything Claude Code** | 全方位開發技能套件（40+ skills） | `/plugin marketplace add affaan-m/everything-claude-code` |
| **Jeffallan Claude Skills** | 社群貢獻技能集 | `/plugin marketplace add Jeffallan/claude-skills` |

## License

MIT
