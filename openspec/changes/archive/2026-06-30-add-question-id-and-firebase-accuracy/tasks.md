## 1. Firebase 準備

- [x] 1.1 更新 Firestore 安全規則，新增 `question_stats` collection：`read: if true`、`write: if request.auth != null`
- [x] 1.2 在 index.html Firebase imports 新增 `increment`、`writeBatch`

## 2. CSV 欄位變更

- [x] 2.1 更新 Google 試算表 A（招式效果順序）：新增 `ID`、`狀態` 欄位，移除 `回答數`、`正確數` 欄位，為所有現有題目填入唯一 ID 及 `有效`
- [x] 2.2 更新 Google 試算表 B（「消除」效果）：同上，ID 跨題庫唯一

## 3. parseData() 更新

- [x] 3.1 新增 `id` 欄位的標頭偵測（`h === 'id'`），映射至 `colMap.id`
- [x] 3.2 新增 `狀態` 欄位的標頭偵測（`h === '狀態' || h === 'status'`），映射至 `colMap.status`
- [x] 3.3 移除 `回答數`、`正確數` 欄位的標頭偵測（`colMap.attempts`、`colMap.corrects`）
- [x] 3.4 在 question 物件中新增 `id: row[colMap.id]?.trim()` 欄位
- [x] 3.5 在 question 物件中新增 `isActive` 欄位：`colMap.status` 不存在時為 `true`；存在時檢查值是否為 `無效`（trim 後比對）
- [x] 3.6 從 question 物件移除 `attemptCount`、`correctCount` 欄位

## 4. initApp() 更新

- [x] 4.1 CSV 解析完成後，建立 `state.questionMap`：遍歷 questionsA 和 questionsB（含廢棄），以 `q.id` 為 key 存入
- [x] 4.2 呼叫 `getDocs(collection(db, "question_stats"))` 讀取統計資料，建立 `state.questionStats = { [id]: { attemptCount, correctCount } }`
- [x] 4.3 在 state 物件宣告中新增 `questionMap: {}`、`questionStats: {}` 兩個初始屬性

## 5. 題庫計數與測驗啟動更新

- [x] 5.1 `selectBank()` 中的 count 顯示改為 `state.questionsA.filter(q => q.isActive).length`（B 同理）
- [x] 5.2 `startQuiz()` 中 `state.questions` 賦值改為在取出對應題庫後，再 `.filter(q => q.isActive)` 後 shuffle

## 6. renderQuestion() 更新

- [x] 6.1 答對率 badge 邏輯改從 `state.questionStats[q.id]` 讀取 `attemptCount`、`correctCount`，移除對 `q.attemptCount`、`q.correctCount` 的參照

## 7. finishQuiz() 更新

- [x] 7.1 建立精簡的 answers 陣列，每個元素格式為 `{ id: q.id, userAnswers: userAns, isCorrect }`，移除 title、images、correctAnswers、explanation
- [x] 7.2 在 leaderboard 寫入後，建立 `writeBatch`，對每道題目執行 `batch.set(doc(db, "question_stats", q.id), { attemptCount: increment(1), correctCount: increment(isCorrect ? 1 : 0) }, { merge: true })`，最後呼叫 `batch.commit()`
- [x] 7.3 question_stats 批次寫入包在 `state.isAuthReady` 條件內（與 leaderboard 寫入同層級）

## 8. renderLookupResult() 更新

- [x] 8.1 重構渲染邏輯：每道 answer 先用 `answer.id` 查 `state.questionMap`
- [x] 8.2 實作「正常顯示」路徑：questionMap 找到 && `isActive=true`，從 questionMap 取 title、images、correctAnswers、explanation 渲染，`userAnswers`、`isCorrect` 來自 answer
- [x] 8.3 實作「已廢棄」路徑：questionMap 找到 && `isActive=false`，同 8.2 顯示所有題目內容，標題旁附加黃色「已廢棄」badge
- [x] 8.4 實作「已移除」路徑：questionMap 找不到，顯示「題目已移除」灰色卡片，仍顯示 `userAnswers` 及 isCorrect（對/錯）
