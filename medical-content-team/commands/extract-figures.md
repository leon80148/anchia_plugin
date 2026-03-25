# PDF 重點圖表擷取工具

從 PDF 論文中系統性擷取重點圖表（Tables / Figures），輸出為獨立 PNG 圖檔。

**PDF 檔案**：$ARGUMENTS

---

## 執行指引

### Phase 1：環境檢查

1. **確認工具可用**：
   ```bash
   which pdftoppm && which python3 && python3 -c "from PIL import Image; print('Pillow OK')"
   ```
   - 需要：`pdftoppm`（Poppler）、`python3`、`Pillow`
   - 如果缺少 pdftoppm：`brew install poppler`
   - 如果缺少 Pillow：`pip3 install pillow`

2. **確認 PDF 檔案存在**：解析 `$ARGUMENTS` 取得 PDF 路徑。如果為空，使用 AskUserQuestion 詢問。

### Phase 2：閱讀 PDF 識別圖表

1. **使用 Read tool 逐頁閱讀 PDF**（使用 pages 參數，每次最多 20 頁）
2. **列出所有需要擷取的圖表清單**，包含：
   - 圖表編號與標題（例如 Table 1, Figure 2）
   - 所在頁碼
   - 內容簡述
3. **向使用者確認**：顯示圖表清單，詢問是否需要調整（增減項目）

### Phase 3：轉換 PDF 頁面為 PNG

1. **建立輸出目錄**：
   ```bash
   mkdir -p [PDF所在目錄]/pdf_figures
   ```

2. **使用 pdftoppm 將包含圖表的頁面轉為 PNG**：
   ```bash
   pdftoppm -png -r 300 -f [起始頁] -l [結束頁] [PDF路徑] [輸出目錄]/page
   ```
   - 固定使用 300 DPI 確保清晰度
   - 只轉換包含圖表的頁面，不需要全部轉換

### Phase 4：定位與裁切圖表

1. **使用 Read tool 查看每張轉換後的全頁 PNG**，目視確認每個圖表的位置

2. **使用 Python + Pillow 裁切每個圖表**：
   ```python
   from PIL import Image

   page = Image.open("page-XX.png")

   # 裁切指定區域 (left, upper, right, lower)
   figure = page.crop((x1, y1, x2, y2))

   # 關鍵：確保任一邊都不超過 2000px（Claude API 多圖限制）
   w, h = figure.size
   if max(w, h) > 2000:
       ratio = 2000 / max(w, h)
       figure = figure.resize((int(w * ratio), int(h * ratio)), Image.LANCZOS)

   figure.save("output_name.png")
   ```

3. **裁切座標估算指南**（以 300 DPI、Letter 紙張為例，全頁約 2550 x 3300 px）：
   - 頁面上方邊距：約 y=50-100
   - 頁首（期刊標頭）：約 y=0-80
   - 正文開始：約 y=80-150
   - 頁面左右邊距：約 x=50-200（左）、x=2350-2500（右）
   - 頁碼區：約 y=3200-3300
   - 建議裁切時左右留足寬度（x: 50 到 2500），重點調整上下邊界（y）

   **底部截斷防範（重要）**：
   - 圖表的 caption/legend/footnote 通常在圖下方，容易被截斷
   - 雙欄排版中，caption 文字常出現在圖表右下角而非正下方
   - **y_end 座標一律多加 100-150px 緩衝空間**，寧可多切一點下方空白，也不要截斷說明文字
   - 表格的腳註（如 * 號說明、CI 定義）往往在表格最下方，要特別留意

4. **命名規範**：
   - 表格：`table[N]_[簡短描述].png`（例如 `table1_characteristics.png`）
   - 圖形：`figure[N]_[簡短描述].png`（例如 `figure1_dose_response.png`）
   - 使用英文小寫加底線

### Phase 5：品質驗證

1. **逐一使用 Read tool 檢視每張裁切後的圖片**，確認：
   - 內容完整（標題、數據、腳註都包含在內）
   - 文字清晰可讀
   - 沒有多餘的空白或截斷

2. **如發現問題，調整裁切座標重新裁切**：
   - 內容被截斷 → 擴大裁切範圍（調整 y1/y2）
   - 太多空白 → 縮小裁切範圍
   - 文字模糊 → 檢查是否過度縮放，必要時只裁切不縮放

3. **最終尺寸檢查**：所有圖片任一邊必須 ≤ 2000px

### Phase 6：清理與交付

1. **刪除暫存的全頁 PNG**（page-XX.png）
2. **列出最終成果清單**，格式：

   | 檔案名稱 | 內容 | 尺寸 |
   |---------|------|------|
   | table1_xxx.png | 描述 | WxH |
   | figure1_xxx.png | 描述 | WxH |

3. **輸出目錄位置**：告知使用者所有圖片存放在 `[PDF目錄]/pdf_figures/`

---

## 重要注意事項

- **2000px 限制**：Claude API 在多圖請求時，每張圖的任一邊不得超過 2000px，否則會觸發 `invalid_request_error`。這是本 skill 存在的核心原因。
- **300 DPI 標準**：低於 300 DPI 會導致表格文字模糊；高於 300 DPI 會增加處理時間且通常不必要。
- **裁切優於縮放**：盡量精準裁切減少無用區域，而非裁切大面積後縮放。縮放會降低文字清晰度。
- **腳註很重要**：學術表格的腳註包含關鍵定義和統計說明，裁切時務必包含。
- **迭代驗證**：每張圖都要用 Read tool 目視確認，不要假設座標正確。

## 常見失敗與排除

| 症狀 | 原因 | 解法 |
|------|------|------|
| `pdftoppm` 輸出空白圖片 | PDF 是掃描影像但有加密保護 | 嘗試 `pdftoppm -upw "" -opw ""` 或請使用者提供解鎖密碼 |
| Read tool 讀不到文字內容 | PDF 是純圖片掃描檔（無 OCR 文字層） | 跳過 Phase 2 的文字識別，直接轉全頁 PNG 後目視定位圖表 |
| 裁切後文字模糊 | resize 壓縮太多，或原始 DPI 不足 | 先試不 resize（確認是否真的超過 2000px）；原始 DPI 低時可嘗試 `-r 450` |
| 跨頁表格只擷取到一半 | 表格橫跨兩頁 | 分別裁切兩頁的部分，用 `Image.new()` 拼接成一張完整圖 |
| 雙欄排版圖表被切半 | 裁切座標只覆蓋單欄寬度 | 雙欄論文的全寬圖表 x 範圍要用 50-2500，不是 50-1250 |
