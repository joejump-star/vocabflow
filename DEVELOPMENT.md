# 開發者說明

給想改造或貢獻 VocabFlow 的人。

## 架構原則

**全部程式碼在 `index.html` 一支檔案裡**（HTML / CSS / JS 都在內），沒有框架、沒有建置流程、沒有相依套件。

這是刻意的選擇：這個 app 要能被複製、離線執行、丟到任何靜態空間就能跑。代價是單檔偏長，靠註解分區來維持可讀性。

想找某段程式，搜尋這些分隔線註解：

```
API KEY MANAGEMENT      BYOK 金鑰管理
DATA: Built-in dictionary  內建字典（無金鑰時的示範資料）
STATE                   全域狀態與 localStorage 讀寫
NAVIGATION              畫面切換與瀏覽器歷史整合
SEARCH                  查詢入口與輸入驗證
3D 筆記術：八大語塊公式    CHUNK_TYPES 與兩套 prompt
GEMINI API              查詢與回應解析
DICTIONARY PAGE         字典頁渲染
備份與還原               匯出／匯入
畢業機制                 「我記熟了」
SRS 間隔重複排程
FLASHCARD / QUIZ / SPELLING
```

## 資料模型

### 詞條（word entry）

單字與語塊共用同一個結構，用 `type` 區分。

| 欄位 | 說明 |
|---|---|
| `word` | 小寫，作為 `DICTIONARY` 與快取的 key |
| `display` | 正常書寫形式（語塊需要保留大小寫與標點），顯示一律用這個 |
| `type` | `word` 或 `chunk` |
| `chunkType` | 語塊才有，八大公式之一（見下） |
| `pos` / `phonetic` / `phoneticUK` / `syllables` / `stress` | 單字的字典資訊 |
| `meanings[]` | `{ zh, en, example, exampleZh }` |
| `etymology` | `{ parts, logic }` 字根拆解 |
| `related[]` | `{ w, zh }` 同字根親戚字 |
| `collocations[]` | **3D**：`{ pattern, items: [{ en, zh }] }`，依語法槽位分組 |
| `chunks[]` | **3D**：`{ en, zh, type }`，這個字常出現的語塊 |
| `variations[]` | **3D**：`{ en, zh }`，語塊的模組化變化，由弱到強排序 |
| `template` | **3D**：`{ en, zh }`，造句練習模板，`___` 為挖空處 |
| `register` | `formal` / `neutral` / `casual` |
| `note` | 使用提醒一句話 |

存進單字庫後另外加上：`level`（`new`/`learning`/`mastered`）、`addedAt`、`stage`、`due`。

**向下相容很重要**：使用者的舊資料不會有新欄位。渲染時一律用 `(data.xxx || [])` 之類的防禦寫法，缺欄位就不顯示該區塊，重查一次即補齊。

### 八大公式（`CHUNK_TYPES`）

出自王梓沅「3D 英文筆記術」，是語塊的分類法：

| key | 中文 | 例 |
|---|---|---|
| `idiom` | 片語 | out of the blue |
| `phrasal` | 片語動詞 | set up the chairs |
| `grammatical` | 文法搭配 | provide sb with sth |
| `lexical` | 語彙搭配 | mild symptoms |
| `starter` | 慣用句頭 | I was wondering… |
| `midsentence` | 慣用句中 | play an important role in sth |
| `situational` | 情境式語塊 | Tell me about it. |
| `frame` | 句子框架 | As far as sb is concerned… |

## 儲存與容量上限

**這是這個 app 最硬的限制，改功能前務必了解。**

資料全部存在 `localStorage`：

| key | 內容 | 備註 |
|---|---|---|
| `vocabflow_bank` | 單字庫（含 SRS 排程） | 核心資料，不可重建 |
| `vocabflow_dict_cache` | 查詢結果快取 | 上限 200 筆，**可重建**（空間不足時第一個被犧牲） |
| `vocabflow_graduated` | 已精通的字 | 只存字串與時間，很小 |
| `vocabflow_gemini_key` | API 金鑰 | **絕不可寫進備份檔或程式碼** |
| `vocabflow_last_backup` | 上次備份時間 | |

### 實測數字

| 項目 | 數值 |
|---|---|
| 一個「完整規格」的字（3-4 義項＋字根＋搭配詞） | **約 2.6 KB** |
| iOS Safari localStorage 上限 | **約 5 MB** |
| 桌面 Chrome 實測 | 約 49 MB |
| **實際安全上限（以 iPhone 為準）** | **約 1,500 字** |

加上 3D 欄位後，單筆會再長一些，安全線要往下抓。

效能不是瓶頸：實測 5,000 字時，每次作答存檔約 29ms、字庫頁渲染約 41ms（桌面）。**容量才是**。

### 空間不足的處理

`saveBank()` 會回傳成功與否，流程是三段式：

1. 正常寫入
2. 失敗 → 刪掉 `vocabflow_dict_cache`（可重建）後重試
3. 還是失敗 → 設 `storageFull` 旗標、跳警示、首頁顯示紅色橫幅、催促匯出備份

**絕對不要把儲存錯誤靜默吞掉。** 這個 app 曾經有過這個 bug：空間滿了卻不告知，使用者以為進度有存、實際上邊學邊掉，重開才發現全沒了。任何寫入 localStorage 的新程式碼都要走 `trySet()`。

## Gemini 查詢

- **BYOK**：金鑰由使用者自帶，存在他自己的瀏覽器，透過 `x-goog-api-key` header 傳送
- 模型：`gemini-2.5-flash-lite`，404 時退到 `gemini-2.5-flash`
- `thinkingConfig: { thinkingBudget: 0 }` — 查字典不需要推理，關掉大幅縮短等待
- `maxOutputTokens: 2600` — 加了 3D 欄位後調高的，再加欄位要記得往上調
- 輸入含空白 → 走 `chunkPrompt()`，否則走 `wordPrompt()`

回應解析一律做欄位驗證與長度截斷（見 `geminiDictionary()` 的 return），不要相信模型會照格式回。

**搭配詞的品質限制**：Gemini 是生成式模型，給的搭配詞不等同 NetSpeak、Ozdic 這類真實語料庫的頻率統計。prompt 裡已要求「只給真實高頻」，但仍是近似。

## 安全性

- 所有外部資料（查詢結果、使用者輸入、字庫內容）進 `innerHTML` 前**必須**經過 `esc()`
- 動態內容**不要**用 inline `onclick` 嵌資料，改用 `data-*` 屬性 + `addEventListener`
- `lookupWord()` 有輸入格式驗證，放寬時要小心

## 導覽與歷史

畫面切換接上 `history` API，讓手機返回鍵表現得像原生 app：

- `showScreen(id)` 純渲染 / `goToScreen(id, meta, replace)` 前進 / `goBack()` 返回
- `navStack` 記錄堆疊，歷史底部固定留一格「護欄」，確保第一次返回被 app 接住
- `popstate` 順序：先關彈窗 → 回上一層 → 已在首頁則詢問離開
- 新增彈窗時記得加進 `closeOpenModal()` 的清單

## 本機開發

```bash
python3 -m http.server 8642
```

改完直接重整。注意 **Service Worker 會快取**，開發時若拿到舊版，到 DevTools → Application → Service Workers 取消註冊，或執行：

```js
navigator.serviceWorker.getRegistrations().then(rs => rs.forEach(r => r.unregister()));
caches.keys().then(ks => ks.forEach(k => caches.delete(k)));
```

## 發佈

推到 `main` 就自動上線（GitHub Pages，branch=main / path=/），約一分鐘生效。

改動時記得更新首頁版本號（搜尋 `home-version`），格式 `vX.Y · MMDD`，那是驗證手機有沒有載到新版的唯一依據。
