# gemini-canvas

製作可以放在 **Gemini Canvas** 裡執行的程式的 AI Skill（單檔 HTML 工具：學習單／教材／試卷產生器，呼叫 Gemini API，可列印或匯出 Word）。

適用於 Claude Code、Antigravity 等支援 `SKILL.md` 的 AI 工具。

## 它做什麼

讓 AI 在你的電腦上寫或修改程式時，嚴守 Canvas 環境慣例，貼回 Gemini Canvas 就能跑：

- 單一 HTML 檔、Tailwind CDN、不用建置工具
- `apiKey = ""` 保持空字串（由 Canvas 執行時帶入）
- 標準模型常數、`systemInstruction` + `responseSchema` 結構化輸出、`fetchWithBackoff` 重試
- 圖片生成、列印樣式、匯出原生 Word（docx）
- Firebase 只用 Canvas 注入的變數

## 安裝（複製整個 repo 到 skills 資料夾）

**Claude Code**（只在某專案用）：

```bash
git clone https://github.com/chuoneone/gemini-canvas.git .claude/skills/gemini-canvas-app
```

**Antigravity**（只在某專案用）：

```bash
git clone https://github.com/chuoneone/gemini-canvas.git .agent/skills/gemini-canvas-app
```

要所有專案通用，改放使用者層級資料夾（Claude Code：`~/.claude/skills/`；Antigravity：`~/.gemini/config/skills/`）。安裝後**新開對話**才會載入。

> 注意：Antigravity 不認資料夾連結（junction／symlink），必須是真正的資料夾。

## 使用

直接說：「幫我做一個 Gemini Canvas 的學習單產生器」，或「幫我修這個 Canvas 程式」。AI 會產出整份 HTML，你整份貼回 Canvas 執行。

## 結構

```text
SKILL.md                  # 規則與工作流程
references/template.html  # 可直接起手的樣板
references/firebase.md    # Canvas 專用 Firebase 初始化
```

## 限制

- AI 在本機無法呼叫 Gemini API，也沒有 Canvas 注入的變數，**AI 回傳內容與 Firebase 需貼回 Canvas 實測**。
- 預設模型為 preview 型號，可能過期；遇 404 先檢查檔案開頭的 `TEXT_MODEL`／`IMAGE_MODEL`。
