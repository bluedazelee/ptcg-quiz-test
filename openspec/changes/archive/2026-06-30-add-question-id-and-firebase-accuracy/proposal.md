## Why

目前每道題目的答對率數據（回答數、正確數）靜態儲存在 CSV 試算表中，需要手動維護且無法自動累積使用者的實際答題結果。此外，答題結果（result code）儲存了大量題目文字與解說，造成 Firebase 讀寫與儲存量浪費。

## What Changes

- **BREAKING** 移除 CSV 中的 `回答數`、`正確數` 欄位
- CSV 新增 `ID` 欄位（手動填入，跨題庫全域唯一，作為題目 key）
- CSV 新增 `狀態` 欄位（值：`有效` / `無效`；欄位不存在時預設全部有效）
- 新增 Firebase `question_stats/{id}` collection，於使用者提交答題後原子性累計每題的作答次數與答對次數
- 測驗頁及題庫計數只顯示 `狀態=有效` 的題目；`狀態=無效` 的題目在 lookup 頁面標示「已廢棄」，已從題庫完全移除的題目標示「已移除」
- `results/{code}` 的 `answers` 陣列精簡為 `{ id, userAnswers, isCorrect }`，刪除重複儲存的題目文字、圖片、解說、正確答案

## Capabilities

### New Capabilities

- `question-identity`: CSV ID 與狀態欄位的解析規則，以及題目有效/廢棄狀態在各畫面的行為定義
- `question-accuracy-stats`: 透過 Firebase 追蹤每題作答統計，並在答題頁面顯示動態答對率 badge
- `lean-result-storage`: 精簡的 result 儲存格式，透過 question ID 在查詢時從 CSV 重建題目詳情

### Modified Capabilities

<!-- 無現有 specs，故略 -->

## Impact

- **CSV 格式**：新增 `ID`、`狀態` 欄位；移除 `回答數`、`正確數` 欄位（需同步更新 Google 試算表 A、B 兩份）
- **parseData()**：新增 id、isActive 欄位解析；移除 attemptCount、correctCount
- **initApp()**：新增建立 `state.questionMap`（含廢棄題目）及從 Firebase 讀取 `question_stats`
- **startQuiz() / selectBank()**：題目列表及計數改為過濾 isActive=true
- **finishQuiz()**：新增批次寫入 question_stats（writeBatch + increment）；精簡 answers 格式
- **renderQuestion()**：答對率 badge 改從 `state.questionStats[q.id]` 讀取
- **renderLookupResult()**：改用 id 查 questionMap 重建題目內容；處理廢棄/已移除兩種狀態
- **Firebase imports**：新增 `increment`、`writeBatch`
- **Firestore 安全規則**：`question_stats` collection 需要 `write: if request.auth != null`，`read: if true`
