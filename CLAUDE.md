# CLAUDE.md

本文件為 Claude Code (claude.ai/code) 在此儲存庫工作時提供指引。

## 專案簡介

SpeakEasy — 一個 AI 英語口說教練 PWA，為英語初學者（Roy，台北 TMGM 客戶經理）量身打造。AI 家教角色名為「NingNing」。整個應用程式是**單一自包含檔案：`index.html`**（CSS + HTML + 原生 JS，無框架、無依賴、無建置流程）。

## 開發方式

沒有 package.json、建置、lint 或測試工具。開發流程就是：編輯 `index.html`，重新整理瀏覽器。

```bash
# 本地伺服器（必要 — Web Speech API 麥克風只能在 localhost 或 HTTPS 上運作）
python -m http.server 8080
```

- **部署**：GitHub Pages 直接服務 `main` 分支，網址為 https://a0982575833-beep.github.io/speakeasy-english/ — 推送到 `main` 即等於部署。
- **要編輯哪個檔案**：`index.html` 是實際部署的正式版本。`SpeakEasy_2026-04-03.html` 是過時的原始碼副本（落後於正式版；最後同步到 v2.3）— 不要編輯它並期望改動會上線。
- **Service worker**：`service-worker.js` 對靜態資源採用 cache-first（`CACHE_NAME = 'speakeasy-v1'`），對 `googleapis.com` 採用 network-only。若需要讓已安裝的使用者更新快取資源，請調升快取名稱版本。
- **Commit 慣例**：使用版本號前綴訊息，例如 `v2.4: Add VoxCPM2 local TTS integration...`。錯誤修正使用 `Fix: ...`。
- **測試**方式為在瀏覽器中手動測試。語音功能需要 Chrome 或 Safari（Firefox 對 Web Speech API 支援有限）。

## index.html 架構

一個 `<script>` 區塊（約第 335 行起），以 `/* ===== SECTION ===== */` 註解分隔各區段。主要慣例：

- `$` / `$$` — `getElementById` / `querySelectorAll` 的簡寫。
- `S` — 全域狀態物件，載入時從 localStorage 還原。所有 key 使用 `K = 'se_'` 前綴；透過 `sv(key, value)` 儲存。
- 五個分頁（`chat`、`daily`、`shadow`、`words`、`stats`）是以 `switchTab()` 切換的絕對定位 div；整個 App 是固定高度的行動裝置外殼，沒有路由。
- 所有面向使用者的文字都是雙語：英文 + 繁體中文（例如 toast 訊息 `'URL updated'`、`'Gentle — major errors only 溫和模式'`）。新增任何 UI 文字時請遵循此慣例。

### Gemini API 整合

- 直接從瀏覽器呼叫 `gemini-2.5-flash:generateContent`，使用**使用者自行提供的 API 金鑰**（存於 localStorage 的 `se_key`）— 沒有後端。預期使用免費額度金鑰（15 RPM），因此 `gem()` 中對 429/額度錯誤有友善的雙語錯誤訊息。
- 聊天使用 SSE 串流（`streamGenerateContent?alt=sse`），並有非串流的 `gem()` 作為備援；對話上下文最多保留最近 20 輪。
- 家教角色設定與糾錯行為位於 `getSysPrompt()`（動態產生 — 會嵌入當前的 gentle/strict 模式）。

### Emoji 回覆協定（關鍵）

系統提示詞會指示模型在回覆中用 emoji 標記嵌入結構化資料，由 `dispMsg()` 以正規表達式解析：

- `📝 ...` — 中文翻譯行
- `⚠️CORRECTION:` 區塊，包含 `❌`（錯誤）/ `✅`（正確版本）/ `💡`（原因，英文）/ `📝`（原因，中文）— 渲染為糾錯卡片
- `📚VOCAB: word (pos) = Chinese | example` — 渲染為存字按鈕

若修改系統提示詞的輸出格式，**必須**同步更新 `sendMsg()`（TTS 前剝除標記）和 `dispMsg()`（擷取卡片）中對應的解析/剝除正規表達式，反之亦然。

### TTS：speak() 的三層備援

1. **VoxCPM2** — 選用的本地 TTS 伺服器（預設 `http://localhost:8809`，可在設定中修改；每 30 秒自動重新檢查可用性）
2. **Gemini TTS** — `gemini-2.5-flash-preview-tts`，語音「Kore」；回傳原始 PCM，經 `pcmToWav()` 轉換
3. **瀏覽器** `speechSynthesis`

播放速度來自使用者的語速設定（0.7x / 1.0x / 1.3x）。語音*輸入*使用 Web Speech API（`SpeechRecognition`），按住說話、按住期間自動重啟，並有 EN/中 語言切換。

### 其他功能

- **每日情境**：每天透過 Gemini 產生 3 個（工作/社交/日常分類），按日期快取於 localStorage。
- **跟讀模式**：內建固定句子清單；聆聽 → 跟讀 → 比較。
- **遊戲化**：XP 經驗值（`addXP`）、連續天數，以及 `S.stats` 中的常犯錯誤計數。

## 儲存庫檔案

- `index.html` — 應用程式本體（部署版本）
- `manifest.json`、`service-worker.js`、`icons/` — PWA 安裝/離線支援
- `README_2026-04-03.md` — 面向使用者的雙語使用說明
- `Zeabur部署指南_2026-04-03.md` — 替代的 Zeabur 部署指南
- `進度_2026-04-03.md` — 專案進度筆記（已過時：寫著 v1.5，但 git 歷史已到 v2.4+ — 請以 `git log` 為準了解目前功能狀態）
