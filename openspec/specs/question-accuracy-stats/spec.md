# Spec: question-accuracy-stats

## Purpose

Defines how per-question accuracy statistics (attempt count and correct count) are stored in and read from Firebase Firestore. Statistics are loaded at app startup, displayed as dynamic accuracy badges during the quiz, and written atomically after each submission.

---

## Requirements

### Requirement: app 啟動時從 Firebase 讀取每題作答統計
`initApp()` 在完成 CSV 解析後 SHALL 一次性讀取 Firestore `question_stats` collection 的全部文件，並將結果存入 `state.questionStats`（`{ [id]: { attemptCount, correctCount } }`）。此讀取不依賴 Firebase Auth 狀態（Firestore 規則 read: if true）。

#### Scenario: 成功讀取統計資料
- **WHEN** initApp() 完成 CSV 解析
- **THEN** 系統 SHALL 呼叫 getDocs(collection(db, "question_stats"))
- **THEN** `state.questionStats` SHALL 以各題 id 為 key，儲存對應的 `{ attemptCount, correctCount }`

#### Scenario: question_stats 中尚無某題的統計
- **WHEN** 某題 id 在 question_stats collection 中不存在文件
- **THEN** `state.questionStats[id]` SHALL 為 `undefined`，視同 `{ attemptCount: 0, correctCount: 0 }`

---

### Requirement: 答題頁面顯示動態答對率 badge
`renderQuestion()` SHALL 從 `state.questionStats[q.id]` 讀取統計資料來顯示答對率 badge，不再從 question 物件的 `attemptCount`/`correctCount` 屬性讀取。

#### Scenario: 有統計資料時顯示 badge
- **WHEN** `state.questionStats[q.id]` 存在且 `attemptCount > 0`
- **THEN** 答對率 badge SHALL 顯示，數值為 `correctCount / attemptCount * 100`，並依百分比套用對應顏色（≥70% 綠色、>40% 橘色、其餘紅色）

#### Scenario: 無統計資料時隱藏 badge
- **WHEN** `state.questionStats[q.id]` 不存在，或 `attemptCount === 0`
- **THEN** 答對率 badge SHALL 隱藏（className 含 hidden）

---

### Requirement: 使用者提交答題後批次寫入每題統計
`finishQuiz()` SHALL 在測驗提交時使用 `writeBatch` 對本次測驗中的每道題目原子性地遞增 `attemptCount`，若答對則同時遞增 `correctCount`。使用 `{ merge: true }` 確保文件不存在時自動建立。

#### Scenario: 批次寫入所有題目統計
- **WHEN** 使用者提交測驗（finishQuiz() 執行）且 Firebase Auth 已就緒
- **THEN** 系統 SHALL 建立一個 writeBatch，對每道題目執行 `batch.set(doc(db, "question_stats", q.id), { attemptCount: increment(1), correctCount: increment(isCorrect ? 1 : 0) }, { merge: true })`
- **THEN** 系統 SHALL 呼叫 `batch.commit()` 完成批次寫入

#### Scenario: Auth 未就緒時跳過統計寫入
- **WHEN** `state.isAuthReady` 為 false
- **THEN** 系統 SHALL 跳過 question_stats 的批次寫入（與現有 leaderboard 寫入的跳過邏輯一致）
