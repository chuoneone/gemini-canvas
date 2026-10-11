---
name: gemini-canvas-app
description: 建立或修改「Gemini Canvas 才能執行」的單檔 HTML 工具（學習單／教材／試卷產生器等，呼叫 Gemini API、可匯出 Word 或列印）。只要使用者提到 gemini canvas、Canvas 程式、學習單生成器、教材產生器，或要修改既有的 Canvas 工具檔案，就使用此 skill，即使沒有明說 skill。
---

# Gemini Canvas 單檔工具

使用者在本機請你寫或修改程式，之後**自己整份貼回 Gemini Canvas 執行**。你在本機無法呼叫 Gemini（沒有 key、沒有 Canvas 注入的變數），所以重點是：嚴守 Canvas 環境慣例，讓貼回去就能跑。

## 不可違反的規則

1. **單一 HTML 檔**，HTML／CSS／JS 全寫在同一檔，不拆檔、不用建置工具。例外：既有的 `.tsx`（React）檔就維持 TSX。
2. **`const apiKey = "";` 必須保持空字串**。Canvas 執行時會自動帶入，絕不寫入真實 key，也不要改成讀環境變數或 localStorage。
3. **不要使用 `localStorage` / `sessionStorage`**（Canvas 沙盒環境不可靠）。需要存檔時用 Firebase（見下）。
4. **Firebase 只用 Canvas 注入的全域變數**：`__firebase_config`、`__app_id`、`__initial_auth_token`，且一律用 `typeof X !== 'undefined'` 保護。寫法見 `references/firebase.md`。
5. **交付整份檔案**，不給片段。使用者是整份貼回，片段會造成錯位。修改既有檔時，也輸出完整修改後的檔案（直接改檔案本身）。
6. 介面文字一律**繁體中文**，`<html lang="zh-Hant">`。

## 標準模型（集中成常數，只改一處）

```js
const TEXT_MODEL  = "gemini-2.5-flash-preview-09-2025";
const IMAGE_MODEL = "gemini-3.1-flash-image-preview";
```

- **文字出題**：`https://generativelanguage.googleapis.com/v1beta/models/${TEXT_MODEL}:generateContent?key=${apiKey}`
  - 必須設定 `systemInstruction`。
  - `generationConfig` 必須用 `responseMimeType: "application/json"` 加 `responseSchema`，依題型定義 JSON Schema，確保結構化輸出。
  - 回傳解析：`JSON.parse(result.candidates[0].content.parts[0].text)`。
- **圖片生成（題目需要插圖／圖解時才呼叫）**：`.../models/${IMAGE_MODEL}:generateContent?key=${apiKey}`
  - `generationConfig: { responseModalities: ['IMAGE'], imageConfig: { aspectRatio: "4:3" } }`（或 `"1:1"`）。
  - 取圖：`data.candidates[0].content.parts.find(p => p.inlineData?.data)`，組成 `data:${mimeType};base64,${data}`。
- preview 型號會過期。使用者回報 404／model not found 時，第一個檢查模型名稱。
- 修改舊檔時若遇到其他型號（如 `gemini-2.5-flash-image-preview`），除非使用者要求，**不要順手統一**，但可在回覆中提醒。

## 標準套件（CDN，鎖定版本）

| 用途 | 來源 |
| --- | --- |
| 樣式 | `https://cdn.tailwindcss.com` |
| 存 Word 檔 | `https://cdnjs.cloudflare.com/ajax/libs/FileSaver.js/2.0.5/FileSaver.min.js` |
| 原生 Word（.docx） | `https://unpkg.com/docx@7.3.0/build/index.js`（全域 `docx`；既有檔有 7.1.0／8.5.0，修改時保持該檔原版本，新檔用 7.3.0） |
| 簡易 Word（HTML 轉） | `https://unpkg.com/html-docx-js/dist/html-docx.js` |
| 數學公式預覽 | KaTeX 0.16.10（含 auto-render） |
| 數學公式進 Word | MathJax 3 → MathML → `mathml2omml@0.5.0`（OMML） |
| 打包 | JSZip 3.10.1 |
| 圖示 | Font Awesome 6.4.0 或 lucide |
| 字型 | Google Fonts：Chiron GoRound TC、LXGW WenKai TC、Noto Sans TC |

只載入實際用到的。

## 標準函式

- `fetchWithBackoff(url, options, maxRetries = 5)`：指數退避重試（1s、2s、4s…），`!response.ok` 時丟出含狀態碼與 API 訊息的錯誤。直接複製 `references/template.html` 內的版本。
- 載入狀態：呼叫期間 disable 按鈕、顯示 spinner、錯誤顯示在畫面上（不要只 `console.error`）。
- 多階段生成（例如先出題、再出摘要）用多個獨立的 `generatePassN_…` 函式，各自帶 schema。

## 輸出：列印與 Word

- **列印**：加 `@media print`：`.no-print` 隱藏、`.page-break`／`.prevent-break` 控制分頁、移除陰影與底色、正文用標楷體（`"標楷體","DFKai-SB",serif`）。A4 頁邊距明確指定。
- **Word**：用 `docx` 產生原生 `.docx`，`saveAs(blob, "檔名.docx")`。快速版本可用 `html-docx-js`。數學題要把 LaTeX 轉成 Word 原生公式（OMML），不要輸出圖片。
- 學習單通常有「學生版」與「教師版（含解答）」，列印與匯出時兩者要能分開。

## 使用者情境（設計時的預設）

預設情境：使用者是教師，對象常是需要鷹架的學生。教學工具通常提供：貼入文本、「學生狀況描述」輸入欄、題型選擇（選擇／填空／開放式）、產生後可預覽、列印、匯出 Word。新工具預設包含這些，除非另有指定。

## 工作流程

1. **修改既有檔**：先讀完整檔案，找出要動的區塊；保持 `apiKey = ""`、模型常數、Firebase 初始化不變；改完檢查 HTML 標籤與 JS 括號是否閉合（可用 `node --check` 檢查抽出的 script 語法）。
2. **建立新檔**：從 `references/template.html` 開始，改標題、表單、schema、輸出區；存成使用者工作區內的單一 `.html` 檔，檔名用描述性名稱。
3. **回報時說清楚**：改了什麼、哪些部分**無法在本機驗證**（Gemini 呼叫、Firebase），請使用者貼回 Canvas 實測。

## 無法在本機驗證的事

沒有 key 與 Canvas 注入變數，AI 回傳內容、Firebase 存取都測不到。本機只能驗證：語法、版面、非 AI 邏輯（用假資料取代 `fetchWithBackoff` 回傳）、列印樣式。誠實說明這點，不要宣稱「已測試可用」。
