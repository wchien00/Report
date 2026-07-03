# 達誠報價單產出助手 - 系統規格說明書 (System Specification)

本文件詳細記錄了「達誠報價單產出助手」的技術規格、資料結構、運作邏輯及防禦性設計（防雷處理）。此文件旨在幫助開發人員與 AI 助手快速理解本專案，並在進行後續維護或功能擴充時避免改壞既有功能。

---

## 📂 專案檔案結構 (Directory Structure)

專案位於 `C:\Users\Laiji\Desktop\報價單&班表`，其結構如下：
```text
報價單&班表/
├── N月報價單 - 工作表1.pdf    # 原始報價單範本 (Excel 匯出)
├── N月報價單.xlsx             # 原始報價單 Excel 檔案
├── quotation_spec.md         # 本系統規格說明書
└── web-app/
    └── index.html            # 報價單助手主程式 (單頁應用 SPA)
```

---

## 💻 技術規格與環境 (Technical Stack)

本專案採用**輕量化、免伺服器託管**的純前端架構：

*   **開發語言**：
    *   **HTML5**：定義網頁結構與各功能元件。
    *   **CSS3 (Vanilla CSS)**：控制自適應手機介面（Mobile-First）及正式 A4 預覽版面。
    *   **JavaScript (ES6+)**：處理使用者互動、本機快取、自動日期推算及資料更新。
*   **外部依賴套件 (透過 CDN 載入)**：
    *   `html2pdf.js (v0.10.1)`：負責將網頁 A4 區域轉換為 PDF 檔案。
    *   `html2canvas (v1.4.1)`：負責將 A4 區域渲染為 Canvas 畫布，用以儲存為 PNG 圖片。
    *   `Google Fonts (Noto Sans TC)`：提供美觀的繁體中文字型，確保跨裝置文字排版一致。
*   **運行環境**：智慧型手機/電腦瀏覽器（如 Chrome, Safari, Edge, LINE 內建瀏覽器）。
*   **資料儲存**：Web Storage API 的 `localStorage`，資料 100% 存在使用者本機設備。

---

## 💾 資料模型 (Data Model)

全域狀態儲存於變數 `appState` 中，其完整的 JSON 結構如下：

```javascript
let appState = {
  // 1. 大直會館排班自動計算參數 (輔助)
  calcYear: 2026,          // 計算年份 (西元)
  calcMonth: 6,            // 計算月份 (1-12)
  calcRate: 500,           // 時薪 (元/小時)
  calcThuHours: 4.0,       // 週四單次工時
  calcWeekendHours: 2.0,   // 週末單次工時

  // 2. 報價單項目列表 (可動態增刪)
  quoteItems: [
    {
      id: 1,               // 項目唯一識別碼 (Number)
      name: "中山會館",    // 項目名稱，支援多行文字 (String)
      unit: "式",          // 單位 (String)
      qty: 1,              // 數量 (Number)
      price: 78000,        // 單價 (Number)
      remark: ""           // 項目備註 (String，可留空供手寫)
    },
    // ... 其他項目
  ],

  // 3. 公司與報價單抬頭設定
  companyInfo: {
    title: "達誠專業環保清潔社",      // 報價抬頭 (公司名稱)
    clientName: "悅禾莊園",          // 客戶名稱
    contactName: "溫瑞萍 0981-881-556", // 聯絡人資訊
    projectName: "清潔工程",         // 工程名稱
    projectLocation: "",           // 工程地點
    quoteDate: "115.06.28",        // 報價日期 (民國格式 YYY.MM.DD)
    taxRate: 5,                    // 稅率 (5 代表 5% 加值稅，0 代表未稅)
    owner: "廖玉山",                // 負責人
    phone: "0961-325-251",         // 公司電話
    notes: "本報價單未列事項若需施作另行報價" // 底部大備註聲明
  }
};
```

---

## 🔄 資料流與運作機制 (Data Flow & Execution Loop)

本系統的運作遵循**單向資料流**與**快取持久化**原則：

```text
[使用者在編輯頁輸入/調整] 
         │
         ▼
[即時寫入 appState 全域變數] ───► [執行 saveStateToLocalStorage() 寫入快取]
         │
         ▼ (使用者切換至「預覽與分享」)
[呼叫 syncPreviewTexts() 與 renderQuotationTable()]
         │
         ▼
[更新 DOM：帶入抬頭、重新計算小計/稅金/總計，渲染 A4 表格]
```

*   **自動存檔**：所有 `input` 與 `textarea` 的 `input` 或 `change` 事件都會即時將資料存入 `localStorage`，防範因關閉瀏覽器導致進度遺失。
*   **報價單與日曆解耦**：本機已無日曆畫面，改由使用者填寫年份/月份後，透過 JS 計算該月週四與週末（六日）日期，產生文字覆蓋寫入 `quoteItems` 的項目 6。

---

## 🛡️ 已知痛點與防禦性設計 (Known Issues & Workarounds)

為了解決手機瀏覽器與 PDF 套件的底層 bug，專案實作了以下**防禦性程式碼 (Defensive Code)**，在未來的修改中**絕對不可移除**：

### 1. PDF 頂部空白與內容位移 bug 
*   **痛點**：在手機上若使用者向下滾動畫面後點擊「下載 PDF」，`html2pdf.js` (內部 `html2canvas`) 會擷取當前視窗的滾動高度，造成產出的 PDF 頂部出現大片空白，且下方內容被擠出紙張被裁切。
*   **解決方案**：在 `downloadPDF()`、`downloadImage()`、`shareToLine()` 的 `html2canvas` 參數中，強制寫死 **`scrollY: 0` 與 `scrollX: 0`**，這會強制截圖從元素最頂部開始。

### 2. A4 尺寸定位與 CSS 縮放衝突 bug
*   **痛點**：為了在手機上完整預覽 A4 寬度（210mm），必須使用 CSS 的 `transform: scale()` 進行縮小。然而直接對 PDF 擷取元素進行縮放，會導致輸出的 PDF 解析度變差或定位跑掉。
*   **解決方案**：
    *   建立一個外層容器 `.preview-scaler` 專門負責在手機畫面上進行 `scale` 縮小。
    *   內層實體擷取元素 `#printable-area` 則**不套用任何 transform 縮放**。
    *   `.a4-page` 高度設定為 **`296mm`** (比標準 A4 的 297mm 縮小 1mm)，且設定 **`overflow: hidden`**。這 1mm 的安全容差能完全防止 PDF 引擎因像素四捨五入而產生第二頁空白頁。

### 3. 固定欄寬與手寫備註空間
*   **痛點**：如果項目的備註欄為空，HTML 表格會自動壓縮該欄位寬度，導致列印出來後沒有空間手寫備註。
*   **解決方案**：
    *   在 `.a4-table` 中設定 **`table-layout: fixed;`**，強制欄寬不受文字內容影響。
    *   表頭設定固定百分比，其中「備註」欄位嚴格設定為 **`10%`**。
    *   設定 `.a4-table td` 的高度為 **`38px`**，確保單行項目在空白時也能維持足夠的格子高度供手寫。
    *   設定底端備註的內容容器最小高度 **`min-height: 48px`**，即使清空備註，下方也會留下 3 行左右的物理空白區供手寫。

---

## 🔮 未來擴充規劃 (Feature Roadmap)

1.  **獨立班表輸出功能**：
    *   在底部導覽列預留擴充空間，未來可加入「輸出班表」Tab。
    *   該功能將獨立產出另一張專供員工看的月曆或排班表 PDF/圖片。
2.  **歷史報價單管理**：
    *   提供本地儲存多份報價單的功能（目前僅支援暫存一份最新修改）。
    *   在編輯頁面頂端增加「讀取舊檔/儲存新檔」下拉選單。
