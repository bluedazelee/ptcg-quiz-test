# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-page PTCG (Pokémon Trading Card Game) quiz app in Traditional Chinese (zh-TW). No build system — the entire application is a single `index.html` file with inline CSS and a `<script type="module">` block.

## Running Locally

Open `index.html` directly in a browser. There is no dev server, no `npm install`, no build step.

**Prerequisite:** `firebase-env.js` must exist in the project root (it is gitignored). This file exports the Firebase config object and is required for the app to load. A local copy already exists at this path — never commit it.

## Architecture

All application logic lives in the `<script type="module">` block at the bottom of `index.html`.

**State machine via views:** The app has five named views (`loading`, `error`, `start`, `quiz`, `result`), each a `<div>` with an `id` prefixed `view-`. `showView(name)` hides all views and reveals the target one.

**Central `state` object** holds all runtime state: shuffled questions array, current question index, user answers map (keyed by question index), nickname, leaderboard data, pagination state, the last score/time for rank lookup, and `lastResultCode` (the 6-character result code generated after each quiz).

**Data flow:**
1. On load, `initApp()` fetches questions from a public Google Sheets CSV URL (`CSV_URL`)
2. `parseCSV()` → `parseData()` normalizes rows into question objects with `{ title, image, options, correctAnswers, isMultiple, explanation }`
3. Answer column supports numeric indices (e.g., `"1,3"`) or literal text values; multiple answers (comma/、-separated) mark a question as `isMultiple`
4. Questions are shuffled with Fisher-Yates before the start screen appears

**Firebase:**
- Anonymous auth via `signInAnonymously` — required before writing scores
- `onAuthStateChanged` triggers `listenToLeaderboard()`, which sets up a real-time `onSnapshot` on the `leaderboard` Firestore collection
- On quiz completion, `finishQuiz()` writes `{ nickname, score, correctCount, totalQuestions, timestamp, time, quizBank }` to `leaderboard`
- Leaderboard is sorted client-side: score descending, then `time` (epoch ms) ascending for tiebreaking
- `state.lastScore` and `state.lastTime` are used together to find the current player's rank in the sorted list
- Firestore imports used: `collection`, `addDoc`, `serverTimestamp`, `onSnapshot`, `doc`, `setDoc`, `getDoc`, `runTransaction`

**Result code feature:**
- After `finishQuiz()` writes to `leaderboard`, it calls `generateUniqueCode(data)` which uses `runTransaction` to atomically check-and-write a 6-character code (charset: `ABCDEFGHJKLMNPQRSTUVWXYZ23456789`) to the `results` Firestore collection, retrying up to 5 times on collision
- The code and full answer details (`answers[]` array with per-question `title`, `isCorrect`, `userAnswers`, `correctAnswers`, `explanation`, `images`) are stored in `results/{code}`
- `state.lastResultCode` holds the generated code for the current session; `renderResultCode()` displays it on the result view
- The start view has a lookup button that opens a modal; `lookupResultCode()` calls `getDoc` to fetch and display any past attempt by code
- The `results` collection requires Firestore security rules: `write: if request.auth != null`, `read: if true`

**Global function exposure:** Functions called from inline HTML `onclick` attributes must be assigned to `window.*` explicitly. Current exposed functions include: `startQuiz`, `nextQuestion`, `prevQuestion`, `openLeaderboard`, `closeLeaderboard`, `setLeaderboardFilter`, `prevPage`, `nextPage`, `openImageModal`, `closeImageModal`, `restartApp`, `selectBank`, `copyResultCode`, `openResultLookup`, `closeResultLookup`, `lookupResultCode`.

## Deployment

Pushes to `main` trigger the GitHub Actions workflow (`.github/workflows/deploy.yml`), which:
1. Generates `firebase-env.js` from the `FIREBASE_API_KEY` GitHub secret
2. Deploys the entire directory to GitHub Pages

The `FIREBASE_API_KEY` secret must be set in the GitHub repo settings for deployment to work.
