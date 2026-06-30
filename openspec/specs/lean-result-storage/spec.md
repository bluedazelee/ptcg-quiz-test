# Spec: lean-result-storage

## Purpose

Defines the lean format for storing quiz results in Firestore. Only user-behaviour data is persisted in the `results` collection; question content (title, images, options, correct answers, explanation) is never stored and is instead reconstructed from the in-memory `questionMap` at lookup time.

---

## Requirements

### Requirement: answers 陣列只儲存使用者行為資料
`finishQuiz()` 寫入 `results/{code}` 時，answers 陣列中每個元素 SHALL 只包含 `{ id, userAnswers, isCorrect }`，不儲存可從 CSV 重建的題目文字、圖片、選項、解說、正確答案。

#### Scenario: finishQuiz 寫入精簡的 answers 格式
- **WHEN** 使用者提交測驗，系統準備寫入 results collection
- **THEN** answers 陣列中每個元素 SHALL 包含且只包含：`id`（字串）、`userAnswers`（字串陣列）、`isCorrect`（布林值）

#### Scenario: userAnswers 儲存選項文字而非索引
- **WHEN** 使用者選擇某個選項
- **THEN** `userAnswers` SHALL 儲存使用者所選選項的完整文字，而非選項在陣列中的索引

---

### Requirement: lookup 頁面從 questionMap 重建題目詳情
`renderLookupResult()` SHALL 使用 `answer.id` 從 `state.questionMap` 查找對應題目，取得 `title`、`images`、`correctAnswers`、`explanation` 後再渲染顯示，不依賴 result document 中儲存這些欄位。

#### Scenario: 正常題目的 lookup 顯示
- **WHEN** lookup 頁面渲染某道題目，且 `answer.id` 存在於 `state.questionMap` 且 `isActive=true`
- **THEN** 卡片 SHALL 顯示來自 questionMap 的 title、images、correctAnswers、explanation
- **THEN** 卡片 SHALL 顯示來自 result document 的 userAnswers 和 isCorrect

#### Scenario: 廢棄題目的 lookup 顯示
- **WHEN** lookup 頁面渲染某道題目，且 `answer.id` 存在於 `state.questionMap` 但 `isActive=false`
- **THEN** 卡片 SHALL 顯示題目所有內容（title、correctAnswers、explanation、images）
- **THEN** 卡片 SHALL 在標題區附加「已廢棄」badge（黃色/灰色樣式）
- **THEN** 卡片 SHALL 顯示 userAnswers 和 isCorrect

#### Scenario: 已移除題目的 lookup 顯示
- **WHEN** lookup 頁面渲染某道題目，且 `answer.id` 不存在於 `state.questionMap`
- **THEN** 卡片 SHALL 顯示「題目已移除」文字取代 title
- **THEN** 卡片 SHALL 仍顯示 userAnswers（使用者當時的回答）及 isCorrect（對/錯）
- **THEN** 卡片 SHALL 不嘗試顯示 correctAnswers、explanation 或 images（因為無從取得）
