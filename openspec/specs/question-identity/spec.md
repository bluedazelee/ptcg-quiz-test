# Spec: question-identity

## Purpose

Defines how questions are identified and filtered across the quiz lifecycle. Each question carries a unique `id` parsed from CSV, an `isActive` status flag, and a global `questionMap` that enables lookup by id — including for deprecated or removed questions.

---

## Requirements

### Requirement: CSV 必須包含 ID 欄位
每道題目在 CSV 中 SHALL 有一個 `ID` 欄位，其值在題庫 A 和題庫 B 合併後全域唯一。parseData() 須將此欄位解析為 question 物件的 `id` 屬性（字串）。

#### Scenario: 正常解析含 ID 欄位的 CSV
- **WHEN** CSV 標頭包含 `ID` 或 `id` 欄位
- **THEN** 每個解析出的 question 物件 SHALL 有 `id` 屬性，值為對應列的 ID 欄位字串

#### Scenario: 建立 questionMap 供 lookup 使用
- **WHEN** initApp() 完成兩份 CSV 的解析
- **THEN** `state.questionMap` SHALL 包含題庫 A 和 B 的全部題目，以 `id` 為 key，value 為 question 物件（含廢棄題目）

---

### Requirement: CSV 必須包含狀態欄位，控制題目的有效性
CSV 的 `狀態` 欄位 SHALL 只接受 `有效` 或 `無效` 兩個值。parseData() 須將此欄位解析為 question 物件的 `isActive` 布林屬性。若 CSV 不含 `狀態` 欄位，全部題目 SHALL 預設為 `isActive=true`。

#### Scenario: 狀態欄位值為「有效」
- **WHEN** 某列的 `狀態` 欄位值為 `有效`
- **THEN** 該 question 物件的 `isActive` SHALL 為 `true`

#### Scenario: 狀態欄位值為「無效」
- **WHEN** 某列的 `狀態` 欄位值為 `無效`
- **THEN** 該 question 物件的 `isActive` SHALL 為 `false`

#### Scenario: CSV 不含狀態欄位（向後相容）
- **WHEN** CSV 標頭不存在 `狀態` 或 `status` 欄位
- **THEN** 所有解析出的 question 物件的 `isActive` SHALL 為 `true`

---

### Requirement: 測驗頁面及題庫計數只包含有效題目
`isActive=false` 的題目 SHALL 不出現在測驗流程中，且不計入題庫題數顯示。

#### Scenario: 開始測驗只載入有效題目
- **WHEN** 使用者點擊「開始測驗」
- **THEN** `state.questions` SHALL 只包含 `isActive=true` 的題目

#### Scenario: 題庫計數排除廢棄題目
- **WHEN** 選擇頁面顯示題庫計數
- **THEN** 顯示的題數 SHALL 等於該題庫中 `isActive=true` 的題目數量

---

### Requirement: lookup 頁面標示廢棄或已移除的題目
在查詢答題結果時，若某道題目已被標為無效或已從 CSV 移除，系統 SHALL 在對應的答題卡片上標示狀態，並仍顯示使用者的作答對錯。

#### Scenario: 顯示廢棄題目（isActive=false）
- **WHEN** lookup 頁面渲染某道題目，且該題目 id 存在於 questionMap 但 `isActive=false`
- **THEN** 卡片 SHALL 正常顯示題目內容，並附加「已廢棄」badge

#### Scenario: 顯示已移除題目（id 不在 questionMap）
- **WHEN** lookup 頁面渲染某道題目，且該題目 id 不存在於 questionMap
- **THEN** 卡片 SHALL 顯示「題目已移除」文字，仍顯示 userAnswers 及 isCorrect 資訊
