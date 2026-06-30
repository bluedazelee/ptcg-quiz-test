# PTCG 線上測驗系統

單頁式 Pokémon 卡牌遊戲（PTCG）規則測驗 App，以繁體中文操作介面呈現，支援單選與多選題、題目附圖、雙題庫選擇、即時雲端排行榜，以及答題代碼分享與查詢功能。行動裝置最佳化，採全螢幕答題佈局。

## 快速開始（本地開發）

1. 在專案根目錄建立 `firebase-env.js`（此檔案已被 `.gitignore` 排除，**不可上傳**）：
   ```js
   export const firebaseConfig = {
     apiKey: "<你的 Firebase API Key>",
     authDomain: "ptcg-quiz-test.firebaseapp.com",
     projectId: "ptcg-quiz-test",
     storageBucket: "ptcg-quiz-test.firebasestorage.app",
     messagingSenderId: "151102927798",
     appId: "1:151102927798:web:7262d98845b866d10f0717"
   };
   ```
2. 直接用瀏覽器開啟 `index.html`。

無需 `npm install`、無需 build 步驟。

## 功能

- **雙題庫**：招式效果順序（A）、「消除」效果（B），開始前可切換
- **單選 / 多選題**：答案欄填多個值自動判斷為多選題
- **題目附圖**：單張或多張卡牌圖片，點擊可放大；多張以橫向捲動呈現
- **正確率 badge**：題目右上角顯示所有作答者的歷史正確率
- **即時排行榜**：依分數排序，同分以完成時間早者優先；支援依題庫篩選
- **答題代碼**：測驗完成後生成 6 位代碼，可分享給他人查閱逐題明細
- **行動裝置最佳化**：答題頁全螢幕佈局，題目區域獨立捲動，導覽按鈕固定於視窗底部

## 題庫管理

題目從 Google 試算表公開發布的 CSV 自動讀取。修改試算表內容後，使用者重新整理頁面即可看到最新題目。

**CSV 欄位格式：**

| 欄位 | Header 關鍵字 | 說明 |
|---|---|---|
| 題目 | 含「題」或 `question` | 必填 |
| 圖片 | 含「圖」或 `image` | 選填，填入圖片 URL，可多欄（多圖） |
| 選項 | 含「選」或 `option` | 可多欄，每欄一個選項 |
| 答案 | 含「答案」或 `answer` | 數字索引（1-based）或文字，多選以逗號或、分隔 |
| 詳解 | 含「解」或 `explanation` | 選填 |
| 回答數 | `回答數` | 選填，用於計算正確率 badge |
| 正確數 | `正確數` | 選填，用於計算正確率 badge |

## 部署

推送至 `main` 分支後，GitHub Actions 自動：
1. 從 `FIREBASE_API_KEY` secret 生成 `firebase-env.js`
2. 部署至 GitHub Pages

需在 GitHub 專案的 Settings → Secrets 設定 `FIREBASE_API_KEY`。

## 答題代碼功能

測驗完成後，系統自動生成一組 **6 位唯一代碼**（如 `A3K9PX`）並存入 Firestore。

- 代碼顯示於結算頁，可一鍵複製分享給他人（如教師、同學）
- 任何人在開始頁點擊「用代碼查詢答題結果」，輸入代碼即可查看該次的逐題對錯明細
- 代碼永久有效，查詢不需登入

**Firestore 安全規則需新增 `results` 集合：**

```
match /results/{code} {
  allow write: if request.auth != null;
  allow read:  if true;
}
```

## 技術架構

- **前端**：純 HTML + Tailwind CSS（CDN）+ Vanilla JS（ES module）
- **後端**：Firebase Anonymous Auth + Cloud Firestore（排行榜：`leaderboard` 集合；答題明細：`results` 集合）
- **題庫**：Google Sheets → 公開 CSV → 前端解析（支援兩個獨立試算表）
- **部署**：GitHub Pages via GitHub Actions
