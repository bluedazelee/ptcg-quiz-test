## Context

PTCG 線上測驗系統是一個單頁應用（單一 index.html），題庫資料來自 Google Sheets 公開 CSV，統計與排行榜資料儲存於 Firebase Firestore，使用匿名驗證。

目前每題的作答統計（回答數、正確數）由題庫維護者手動填入 CSV，答對率 badge 顯示靜態資料。答題結果（result code）在 Firestore 的 `results` collection 中每份文件儲存完整的題目文字、圖片 URL、選項、解說，導致每份文件約 6,000+ 字元。

## Goals / Non-Goals

**Goals:**
- CSV 新增 `ID`、`狀態` 欄位；移除 `回答數`、`正確數` 欄位
- 使用者提交答題後，自動寫入 Firebase `question_stats` 累計各題統計
- 答對率 badge 改為從 Firebase 讀取動態資料
- `results/{code}` 的 answers 陣列精簡，lookup 時從已載入的 CSV 資料重建題目內容
- `狀態=無效` 的題目在測驗頁和計數中被忽略，但在 lookup 頁面保留顯示並標示狀態

**Non-Goals:**
- 不實作即時（real-time listener）的答對率更新；初次載入後靜態顯示
- 不遷移 CSV 中既有的回答數/正確數歷史資料到 Firebase
- 不修改 Firebase 安全規則以外的後端設定
- 不支援管理員後台介面操作廢棄狀態

## Decisions

### D1：question_stats 統計資料在 app 啟動時一次性讀取

**決定**：在 `initApp()` 完成 CSV 載入後，用 `getDocs(collection(db, "question_stats"))` 一次讀取全部統計資料，存入 `state.questionStats`（`{ [id]: { attemptCount, correctCount } }`）。

**理由**：答對率 badge 不需要即時反映其他使用者的答題，一次讀取足夠。避免 real-time listener 產生持續的讀取費用，也避免在答題過程中 badge 數字跳動。

**替代方案**：real-time listener → 過度複雜且費用較高，排除。

---

### D2：finishQuiz() 使用 writeBatch 批次寫入 question_stats

**決定**：建立單一 `writeBatch`，對本次測驗涉及的每道題目呼叫 `batch.set(ref, { attemptCount: increment(1), correctCount: increment(isCorrect ? 1 : 0) }, { merge: true })`，最後一次 `batch.commit()`。

**理由**：單次網路來回取代多次個別寫入，減少延遲。`merge: true` 讓文件不存在時自動建立，無需預先初始化。`increment()` 確保並發安全，避免 race condition。

**替代方案**：逐題 setDoc → 多次網路往返，效率差，排除。

---

### D3：questions 分成兩個用途的資料集

**決定**：
- `state.questionsA` / `state.questionsB`：parseData() 回傳全部題目（含廢棄），每筆含 `isActive` 旗標
- `state.questionMap`：`{ [id]: question }`，從兩個題庫合併，包含廢棄題目，供 lookup 時查詢
- quiz 流程及計數改為在需要時 filter `isActive === true`

**理由**：廢棄題目的詳細資訊（title、explanation 等）在 lookup 顯示舊 result 時仍需要。若 parseData() 直接過濾，lookup 無法重建廢棄題目的詳細內容。單一資料來源，減少重複解析。

---

### D4：result answers 只儲存 id、userAnswers、isCorrect

**決定**：`answers` 陣列從 `{ title, isCorrect, userAnswers, correctAnswers, explanation, images }` 精簡為 `{ id, userAnswers, isCorrect }`。lookup 時從 `state.questionMap` 依 id 重建題目內容。

**理由**：節省約 10 倍的 Firestore 寫入/讀取量。lookup 已在 app 內（CSV 已載入），不需要 result document 再次儲存靜態內容。

**風險**：若題目被完全從 CSV 刪除，lookup 無法重建題目文字 → 以「已移除」狀態呈現，仍顯示 userAnswers 和 isCorrect（見 D5）。

---

### D5：lookup 對廢棄/移除題目的顯示策略

**決定**：
- `questionMap` 找到 && `isActive=true` → 正常顯示
- `questionMap` 找到 && `isActive=false` → 顯示題目全部內容 + 黃色「已廢棄」badge
- `questionMap` 找不到（題目已從 CSV 刪除）→ 灰色卡片，顯示「題目已移除」、仍顯示 userAnswers 和 isCorrect

**理由**：保留使用者知道自己對錯的資訊，即使題目已廢棄或移除。

## Risks / Trade-offs

- **[風險] CSV 選項文字修改** → userAnswers 儲存選項文字，若選項文字後來被修改，lookup 顯示的使用者回答與當前選項不符。**緩解**：題目一旦上線不修改選項文字（已確認為使用慣例）。

- **[風險] question_stats 初始讀取在 auth 就緒前完成** → `initApp()` 與 `onAuthStateChanged` 並行執行，讀取 question_stats 不需要 auth（read: if true），因此不受影響。

- **[風險] writeBatch 超過 500 操作上限** → 單次測驗題數遠低於 500 題，不會觸發。

- **[Trade-off] 答對率 badge 不即時** → 使用者在本次 session 內看到的統計是載入時的快照，提交後不會看到自己這次貢獻的更新。可接受，因為統計是輔助資訊。

## Migration Plan

1. 更新 Google 試算表 A 和 B，新增 `ID`、`狀態` 欄位，移除 `回答數`、`正確數` 欄位，為所有現有題目填入 ID 和 `有效`
2. 更新 Firestore 安全規則，允許 `question_stats` collection：`read: if true`，`write: if request.auth != null`
3. 部署新版 index.html
4. 歷史統計從零開始累積（不遷移舊 CSV 資料）

**Rollback**：若需還原，重新部署舊版 index.html；Firestore 資料不受影響（leaderboard 和 results collection 格式相容）。

## Open Questions

（無，設計於探索階段已與使用者確認）
