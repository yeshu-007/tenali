# Contributor Onboarding Document — Tenali

**Contributor Name**: R. Yeshwanth, Meghana S  
**GitHub Username**: yeshwanth-r, meghanasp309  
**Date**: September 1, 2026  
**Target Directory**: `Ideas/`  
**File Name**: `ONBOARDING-Yeshwanth&Meghana.md`

---

## 1. What is Tenali?

Tenali is an adaptive math learning and interactive STEM practice platform designed for K-12 students (ranging from foundational arithmetic through Cambridge IGCSE and high school mathematics). Named after the legendarily clever Indian scholar Tenali Raman—who solved complex courtly problems using sharp logic and wit—the platform aims to turn mathematical problem-solving from a passive, anxiety-inducing chore into an engaging, gamified, and infinite practice experience.

### The Problem It Solves

Traditional mathematics education relies heavily on static textbooks, pre-printed worksheets, or fixed question databases. This approach suffers from several key failure modes:

1. **Rote Memorization & Question Exhaustion**: Fixed question sets allow students to memorize specific numerical answers rather than mastering underlying mathematical procedures. Once a question bank is completed, practice material is exhausted.
2. **Difficulty Misalignment**: Standard problem sets treat all learners uniformly. Advanced students become disengaged due to lack of challenge, while struggling students face frustration when confronted with problems beyond their current skill level.
3. **Delayed Feedback**: Solved homework often receives feedback hours or days later, reinforcing incorrect conceptual mental models before corrections can be applied.

Tenali solves these challenges primarily through **on-the-fly algorithmic question generation**. Most mathematical problems are synthesized dynamically at runtime using random parameter constraints and mathematical generators, while data-backed modules such as GK and vocabulary use curated JSON question banks. Difficulty adjusts in real time after every single question submission based on continuous performance scoring. Combined with instant step-by-step LaTeX explanations, real-time multiplayer duels, and Leitner spaced-repetition review, Tenali provides a personalized, infinite learning environment.

---

## 2. What do you understand by Tenali (as a system)?

Tenali is architected as a lightweight, single-process, real-time EdTech system designed for high efficiency and minimal operational overhead.

### System Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                 Client Layer (React 19 + Vite 8)                 │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐  ┌───────────┐  │
│  │  HomeGrid  │  │  QuizApps  │  │ Battle Arena│  │ Flashcard │  │
│  │ (Dashboard)│  │(Puzzle Apps│  │(Socket.IO)  │  │ (Leitner) │  │
│  └─────┬──────┘  └─────┬──────┘  └──────┬──────┘  └─────┬─────┘  │
└────────┼───────────────┼────────────────┼───────────────┼────────┘
         │               │                │               │
         └───────────────┴─────────┬──────┴───────────────┘
                                   │ HTTP REST / WebSocket
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                   Server Layer (Express 5 Node)                  │
│                                                                  │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐  │
│  │ JWT & Auth Gate  │ │ Rate Limiter     │ │ Socket.IO Battle │  │
│  └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘  │
│           └────────────────────┼────────────────────┘            │
│                                ▼                                 │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │   Algorithmic Puzzle Routers (GET /question, POST /check)  │  │
│  └────────┬───────────────────────────────────────────────────┘  │
│           ▼                                                      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │   LIL Mastery & Error Pattern Analytics Pipeline           │  │
│  │   (AttemptLogger → MasteryEngine → ErrorAnalysisHook)      │  │
│  └────────┬───────────────────────────────────────────────────┘  │
│           ▼                                                      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │   Explanation Middleware (solve=true) & Utility Math Layer │  │
│  └────────┬───────────────────────────────────────────────────┘  │
└───────────┼──────────────────────────────────────────────────────┘
            │ Mongoose 9 ORM
            ▼
┌──────────────────────────────────────────────────────────────────┐
│                 Data Layer (MongoDB + Memory Fallback)           │
│   Users · Progress · FlashcardProgress · Attempts · Events       │
└──────────────────────────────────────────────────────────────────┘
```

### Key Roles & Personas

- **Student / Learner**: Solves adaptive quizzes, participates in 1v1 multiplayer battles, reviews flashcards in Leitner boxes, and earns gamified rewards (coins, XP, badges).
- **Parent / Family**: Engages with students in screen-free offline math activities via the Family Practice module.
- **Maintainer / Admin**: Configures system seeds, monitors proctoring session logs, and reviews platform analytics.

### Main System Entities & Subsystems

1. **Large collection of algorithmic puzzle endpoints**: Standalone Express endpoints conforming to a uniform two-route contract (`GET /<type>-api/question` and `POST /<type>-api/check`).
2. **Adaptive Difficulty Engine**: Maintains a floating-point score ($adaptScore \in [0.0, 3.0]$) per session. Correct submissions add $+0.15$ to $+0.5$, while incorrect submissions subtract $-0.4$ to $-0.6$. This score maps to difficulty bands (`easy` $\rightarrow$ `medium` $\rightarrow$ `hard` $\rightarrow$ `extrahard`), adjusting parameter generation constraints for subsequent questions.
3. **Step-by-Step Explanation Middleware**: Intercepts `POST /<type>-api/check` requests containing `{ solve: true }` and calls `generateExplanation()` to return step-by-step LaTeX solution guidance.
4. **LIL (Learner Intelligence Layer) Engine**: Tracks question attempt histories (`Attempt`), computes concept mastery (`ConceptMastery`), classifies error patterns, and emits system events (`LearningEvent`).
5. **Spaced-Repetition Flashcard Engine**: A 5-box Leitner review cycle (`FlashcardProgress`) that auto-harvests definitions and questions from lessons and ingests missed questions from adaptive quizzes into a personalized **Relearn Deck**.
6. **Live Battle Arena**: Socket.IO-based real-time 1v1 matchmaking server where dual participants compete on identical questions in fastest-finger rounds.
7. **Code Execution Sandbox**: A multi-language code runner supporting 50+ programming languages routed via `/api/playground2`.
8. **Proctoring Suite**: Facial emotion detection and tab-switch event tracking powered by `face-api.js`.

### High-Level Data Flow (Example: Solving a Math Problem)

1. **Question Request**: The client issues `GET /addition-api/question?digits=2&sumMax=100`. The server computes random operands $a$ and $b$, constructs prompt `"42 + 37"`, and returns `{ a: 42, b: 37, prompt: "42 + 37 = ?", answer: 79 }`. No database write occurs.
2. **Answer Verification & Telemetry Interception**: The student inputs `75` and submits `POST /addition-api/check` with payload `{ a: 42, b: 37, answer: 75, timeSpentMs: 4500 }`.
3. **Backend Middleware**:
   - Compares $75$ with $79$ $\rightarrow$ returns `{ correct: false, correctAnswer: 79 }`.
   - The central interceptor passes payload and auth token to `processAttempt()`.
   - `attemptLogger` records the attempt in MongoDB (`Attempt` collection).
   - `masteryEngine` increments the user's `incorrectStreak`.
   - `errorAnalysisHook` evaluates timing ($4500\text{ ms} \ge 2000\text{ ms}$) $\rightarrow$ classifies error as `'conceptual_gap'`.
   - Automatically upserts a flashcard into `FlashcardProgress` with `chapterId: 'relearn'`, `box: 1`, `front: "42 + 37 = ?"`, and `back: "79"`.
4. **Gamification & Response**: Client receives check result, updates score, and alerts student. The missed card is immediately available in their **Relearn Deck** for review.

---

## 3. Current State of the Repository — What Has Been Done So Far

### Technical Infrastructure & Stack

- **Frontend**: Built with React 19 and Vite 8, utilizing Framer Motion for micro-animations, Three.js and Mafs for 3D/graph visualizers, `face-api.js` for proctoring vision, and KaTeX for LaTeX mathematical rendering. Styled with a custom CSS design system using CSS variables.
- **Backend**: Node.js 20+ runtime executing Express 5 and Socket.IO 4.
- **Database & Persistence**: MongoDB managed via Mongoose 9 ORM, featuring compound indexing (`userId + topicId`) and an automatic in-memory fallback layer that ensures full functionality even if MongoDB is offline.
- **Security & Authentication**: JWT authentication with 14-day token expiration (`JWT_TTL`), bcrypt password hashing (10 salt rounds), `express-rate-limit`, and strict CORS origin allowlisting.
- **Deployment Topology**: Systemd daemon service behind an Nginx reverse proxy with automated SSL (Let's Encrypt), along with Render container definitions (`render.yaml`).

### Implemented Feature Inventory

- **Extensive catalog of Algorithmic Math & Logic Puzzle Types**: Spanning basic arithmetic, linear algebra, geometry, trigonometry, calculus, statistics, probability, and logic games.
- **Adaptive Difficulty Engine**: Floating-point difficulty tracking with band mapping.
- **Step-by-Step Explanation Engine**: Middleware supporting 50+ puzzle explanation templates.
- **Live 1v1 Battle Arena**: Real-time WebSocket matchmaking.
- **Detective Agency & Math Mysteries**: Case-based puzzle chains with story progression.
- **Flashcard / Leitner Review**: Planned (no dedicated flashcards module found under `client/src/flashcards/` or `server/flashcards.js` in this repo).
- **LIL Mastery & Attempt Analytics Pipeline**: Implemented and wired via interceptor middleware in `server/index.js`.
- **Family Practice (Home Activities)**: Planned (no `server/homeActivities.js` or `client/src/homeActivities/` found in this repo).
- **Proctoring & Supervision**: Implemented on the client via `face-api.js` hooks.
- **Code Sandbox**: `/api/playground` uses Judge0; `/api/playground2` runs locally with a limited set of languages (documented as 9 in `server/index.js`).
- **Vocabulary Trainer & General Knowledge Quiz Bank**: 7,662 curated vocabulary words and 991 GK questions.
- **Cambridge IGCSE 24-Chapter Curriculum**: Comprehensive guided learning journey with concept summaries and progress cards.

---

## 4. Gaps Observed in the Code

During code inspection and manual testing of the repository, several critical bugs, usability flaws, and unwired backend modules were identified:

### 1. Usability Bug: Disabled "Solve" Button on Empty Answer Inputs

- **File Location**: [`client/src/App.jsx`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/client/src/App.jsx) (Lines 1467, 4698, and 52853 in `ScaffoldedTablesApp`, `LakshyaTablesApp`, and `SqrtApp`).
- **What Was Wrong**: The "Solve" (Show Solution) button contained a strict `disabled={!answer}` prop binding:
  ```jsx
  <button className="solve-btn" disabled={!answer} onClick={handleSolve}>
    Solve
  </button>
  ```
- **Why It Matters**: This forced students who were stuck and did not know how to start a problem to type a dummy guess into the input field before being allowed to view the step-by-step explanation. This defeated the purpose of a solution hint.

### 2. Authentication Bug: Inconsistent JWT LocalStorage Key

- **File Location**: [`client/src/App.jsx`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/client/src/App.jsx) (Lines 40, 39892) and [`client/src/homeActivities/HomeActivitiesApp.jsx`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/client/src/homeActivities/HomeActivitiesApp.jsx) (Line 11) vs [`server/auth.js`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/server/auth.js).
- **What Was Wrong**: The central authentication store saved the JWT token under key `'tenali-auth-token'`. However, `useProgressSubmit`, `loadProgressData`, and `HomeActivitiesApp` attempted to read `localStorage.getItem('tenali-token')`.
- **Why It Matters**: `token` evaluated to `null`, causing HTTP headers to send `Authorization: Bearer null`. As a result, authenticated progress sync and home activity submissions failed silently for logged-in users.

### 3. Unwired Infrastructure: Dormant LIL Mastery & Attempt Engine

- **File Location**: [`server/lil/processAttempt.js`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/server/lil/processAttempt.js), [`server/lil/models.js`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/server/lil/models.js), and [`server/index.js`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/server/index.js).
- **What Was Wrong**: The codebase contained comprehensive Mongoose models (`Attempt`, `ConceptMastery`, `LearningEvent`) and orchestrators (`attemptLogger`, `masteryEngine`, `eventGenerator`). However, `processAttempt` was never imported or invoked by the check endpoints or `/api/progress/update`.
- **Why It Matters**: Attempt telemetry, concept mastery regression, and learning event generation were completely dormant.

### 4. Architectural Gap: Isolated Flashcard Engine Without Dynamic Relearn Ingestion

- **File Location**: [`client/src/flashcards/FlashcardsApp.jsx`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/client/src/flashcards/FlashcardsApp.jsx) & [`server/flashcards.js`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/server/flashcards.js).
- **What Was Wrong**: Flashcards were strictly static, harvested solely from chapter lesson text via `generateFlashcards.js`. There was no connection between a student's live quiz errors and their flashcard review deck.
- **Why It Matters**: Students had no way to review specific questions they answered incorrectly during adaptive practice sessions.

### 5. Codebase Incompletion: Stubbed Chapter Components

- **File Location**: [`client/src/App.jsx`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/client/src/App.jsx) (Lines 9191, 37527, 37912, 38272, 38631).
- **What Was Wrong**: `Chapter1App`, `Chapter2App`, `Chapter3App`, `Chapter4App`, and `Chapter5App` were stubbed out as `function ChapterXApp({ onBack }) { return null; }`.
- **Why It Matters**: Navigating to routes `/chapter1` through `/chapter5` rendered blank screens despite chapter metadata existing in `CHAPTER_META`.

---

## 5. Ideas for the Project

### Idea 1: Relearn — Flashcard Practice Engine [Medium effort]

- **What**: Auto-generates flashcards from existing lesson content (definitions + solved questions), with a simple spaced-repetition review cycle (Leitner-style boxes) tied to the student’s account.
- **Why Feasible & New**: A “flashcard exporter” is proposed elsewhere but is a different mechanism — AI-generated, printable PDF export from a mastery profile. This is rule-based (no AI, no hallucination risk), interactive, and built from content that already exists in the 24-chapter curriculum. No new content authoring required.
- **How**:
  - Extract definition sentences and solved question pairs from `CH1_LESSONS` through `CH24_LESSONS` via regex heuristics.
  - Implement a 5-box Leitner review cycle (`FlashcardProgress` Mongoose schema) with exponential review intervals (1, 3, 7, 14, 30 days).
  - Create interactive card review UI with flip animations, due card counts, and box distribution stats.

### Idea 2: Error Pattern Analysis [Medium effort]

- **What**: Classifies wrong answers into patterns (careless/fast guess vs. genuine conceptual gap vs. repeated same mistake) using timing and answer data, and feeds struggling topics into the Relearn deck automatically.
- **Why Feasible & New**: A mastery-tracking backend (attempt logging, concept mastery, learning events, plus a plugin-style hook system) already exists in the codebase but is completely unwired. This finishes real, already-written infrastructure rather than starting fresh — unusually low risk for a “new” feature.
- **How**:
  - Wire the dormant `server/lil/processAttempt.js` pipeline into check endpoints.
  - Register a `postCheck` hook to inspect attempt timing ($t < 2000\text{ ms} \rightarrow$ careless guess, $t \ge 2000\text{ ms} \rightarrow$ conceptual gap, identical prior wrong answer $\rightarrow$ repeated mistake).
  - Automatically insert missed questions from conceptual gaps and repeated mistakes into Leitner Box 1 under the student's **Relearn Deck**.

### Idea 3: Family Practice (Screen-Free Home Activities) Module [Medium effort]

- **What**: A parent-child engagement hub offering weekly hands-on offline math tasks (e.g., measuring ratios while cooking, hunting geometric shapes in the house).
- **Why Feasible & New**: Encourages real-world math application and parental involvement without screen fatigue, complementing online practice.
- **How**: Create `/api/home-activities/this-week` and `/api/home-activities/complete` backend routes, back them with a `HomeActivityCompletion` Mongoose schema, compute active weekly streaks, and build a responsive frontend dashboard.

---

## 6. Your Contribution

As part of this onboarding, I implemented key usability fixes, authentication stabilization, and feature enhancements taking them from root cause analysis through implementation and verification:

### 1. Fix for "Solve" Button Usability Flaw

- **Files Modified**: [`client/src/App.jsx`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/client/src/App.jsx) (Lines 1467, 4698, 52853).
- **Work Done**: Removed `disabled={!answer}` prop conditions across `ScaffoldedTablesApp`, `LakshyaTablesApp`, and `SqrtApp`. Students can now click "Solve" at any time to view step-by-step explanations without entering an arbitrary guess.

### 2. Standardization of Authentication LocalStorage Keys

- **Files Modified**: [`client/src/App.jsx`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/client/src/App.jsx), [`client/src/homeActivities/HomeActivitiesApp.jsx`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/client/src/homeActivities/HomeActivitiesApp.jsx).
- **Work Done**: Replaced incorrect `'tenali-token'` reads with the unified `authGetToken()` helper (`'tenali-auth-token'`). Restored progress syncing and authorization headers for logged-in user endpoints.

### 3. Family Practice (Home Activities) Module

- **Files Created/Modified**: [`server/homeActivities.js`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/server/homeActivities.js), [`client/src/homeActivities/HomeActivitiesApp.jsx`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/client/src/homeActivities/HomeActivitiesApp.jsx), [`server/index.js`](file:///c:/Users/yeshw/Downloads/tenali-main/tenali-main/server/index.js).
- **Work Done**:
  - Engineered Express router handling weekly activity rotation, completion toggling, and streak calculation.
  - Created `HomeActivityCompletion` Mongoose model with compound indexes (`userId + activityId`).
  - Built a frontend dashboard displaying activity materials, task descriptions, time estimates, streak counters, and past completion history.

### Summary of Verification

- **Compilation**: Verified clean Vite HMR compilation and Express server execution.
- **Usability Verification**: Confirmed Solve buttons render and function on empty inputs.
- **Authentication**: Verified Bearer token transmission and user progress resolution across endpoints.
