# VocabFlow

> 查一個字，就變成我的字

一個給中文母語者的英文單字學習 PWA。查完的字不會查完就忘——會進入間隔重複排程，用字卡、選擇題、拼字三種方式，把「查過」變成「記得」，再變成「會用」。

單一 HTML 檔、無後端、資料全存在你自己的瀏覽器裡。可以加到手機主畫面當 app 用，離線也能開。

## 特色

- **AI 字典**：串接 Google Gemini 產生完整字典條目——音節切分、重音標示、美式／英式音標、多個義項、例句與翻譯、字根拆解與同根字
- **間隔重複（SRS）**：每個字有熟練階段與到期日，間隔為 1 / 3 / 7 / 16 / 35 / 70 天。答對延長、答錯歸零
- **三種練習模式**
  - **Cards**：字卡翻牌，先回想再看答案
  - **Quiz**：看英文選中文，練「認得」
  - **Spell**：看中文拼英文，練「會用」（容錯一個字母）
- **答錯重排**：答錯的字會排到本回合最後重出，直到答對；計分只看第一次作答
- **深色模式**：跟隨系統外觀設定
- **完全離線可用**：除了查新字需要連網，其餘功能離線皆可運作

## 開始使用

### 1. 取得 Gemini API Key

這個 app 採 **BYOK（Bring Your Own Key）**——你用自己的金鑰，不經過任何第三方伺服器。

到 [Google AI Studio](https://aistudio.google.com/apikey) 免費申請一組 API key。

### 2. 開啟 app

直接開啟 `index.html` 即可，或用任意靜態伺服器：

```bash
python3 -m http.server 8642
```

第一次查單字時，app 會跳出面板請你貼上 API key。金鑰只會存在你瀏覽器的 localStorage，不會傳送到 Google 以外的任何地方。

### 3.（選用）加到手機主畫面

用手機瀏覽器開啟後選擇「加入主畫面」，就會像原生 app 一樣運作。**注意：這需要 HTTPS 環境**（`localhost` 也可以），用區網 IP 直接開的話 Service Worker 不會註冊，離線功能與安裝都不會生效。

## 隱私

- **API key** 存在 localStorage，只用於直接呼叫 Google Gemini API
- **單字庫與學習進度** 全部存在瀏覽器 localStorage，不會上傳任何地方
- 沒有後端、沒有帳號、沒有追蹤

反過來說：**清除瀏覽器網站資料就會失去所有單字**。目前尚未實作匯出備份功能（規劃中）。

## 部署

純靜態網站，把整個資料夾丟到任何靜態託管服務即可（Netlify、Vercel、GitHub Pages、Cloudflare Pages⋯）。不需要任何建置流程。

## 技術架構

| 項目 | 說明 |
|---|---|
| 架構 | 單一 `index.html`，內含全部 HTML / CSS / JS，無框架、無相依套件 |
| AI | Gemini `gemini-2.5-flash-lite`，備援 `gemini-2.5-flash` |
| 發音 | 三層降級：Free Dictionary API 真人音檔 → Google TTS → 瀏覽器 `speechSynthesis` |
| 儲存 | localStorage（`vocabflow_bank` / `vocabflow_gemini_key` / `vocabflow_dict_cache`） |
| 離線 | Service Worker——頁面走網路優先（更新自動生效），靜態資源走快取優先 |

## 已知限制

- **iOS 獨立視窗模式無法完全攔截離開**：app 已整合瀏覽器歷史，Android 返回鍵、瀏覽器返回鍵、iOS 邊緣滑動返回都會「回上一層」，在首頁才詢問是否離開。但從 App 切換器上滑關閉是作業系統層級動作，任何網頁技術都攔不到
- **搭配詞由 AI 生成**：不等同 NetSpeak、Ozdic 這類真實語料庫的頻率統計
- **快取的舊字缺新欄位**：改版新增欄位後，先前查過的字會正常顯示但缺少新區塊，重查一次即更新

## 開發路線

- [ ] 單字庫匯出／匯入 JSON 備份
- [ ] 3D 單字筆記：搭配詞（依語法槽位分組）、語塊（多字詞條與模組化變化）、使用者自己的造句與 AI 批改
- [ ] 搭配詞挖空練習題
- [ ] 聽力模式：只播發音，選中文意思
- [ ] 連續學習天數 streak

## 授權

[MIT](LICENSE)
