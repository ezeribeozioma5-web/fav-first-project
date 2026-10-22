# StudyNova — Codebase Review & Implementation Plan

Reviewed the existing project before proposing anything below — current file
tree, schema, design tokens, and patterns are summarized here so the plan is
grounded in what's actually built, not assumed.

---

## 0. One discrepancy to flag before anything else

The brief states the official tagline is **"StudyNova — Your Study
Companion."** The current app (approved during the rebrand) uses **"Study
smarter. Achieve more."** — it's in `src/theme/tokens.ts` (`APP_TAGLINE`) and
rendered on the splash screen and both auth screens.

Per this brief's own instruction not to rename/replace existing approved
work without cause, I'm not changing it silently. **Please confirm:** keep
the existing tagline, or replace it with the new one? One-line fix either
way, just needs a decision before I touch it.

---

## 1. Architecture recap (what this plan builds on, not around)

- **Stack:** Expo Router (file-based nav) + TypeScript, NativeWind v4 for
  styling, Zustand for client state, Supabase (Postgres + Auth + Storage +
  Edge Functions)
- **Pattern already established and reused repeatedly:** a parent/child table
  pair for generated content (`quizzes`→`quiz_questions`,
  `flashcard_decks`→`flashcards`), an Edge Function per AI generation task
  (`generate-quiz`, `generate-summary`, `generate-flashcards`), RLS on every
  user-owned table (`auth.uid() = user_id`), Premium gating checked
  **server-side** in Edge Functions, not just hidden in UI
- **Design system:** light/dark tokens in `tailwind.config.js` +
  `src/theme/tokens.ts`, shared components (`Button`, `Card`, `EmptyState`,
  `Skeleton`, `OfflineBanner`, `SubjectChip`/`Picker`/`FilterRow`)
- **5 tabs today:** Home, AI Tutor, Notes, Practice (Quiz), Profile
- **Zero native code** — this is a stock Expo Go-compatible project so far;
  that matters a lot for a couple of items below

---

## 2. Feature-by-feature status

### ✅ Already built (do not rebuild)
- Splash, Onboarding, Auth, Home Dashboard, AI Tutor, Notes (upload → AI
  summary → quiz/flashcard generation → note-context chat), Quiz module
  (generation → full taking flow → scoring → history), Flashcards (generation
  → flip/swipe study → progress tracking), Subjects (personal organization),
  Education Stage/Class + Exam Track taxonomy (schema only, no collection UI
  yet), Light/Dark themes, Supabase foundation + RLS patterns, streak
  tracking (`profiles.streak_count`)

### 🟡 Partially there — real infrastructure exists, feature-specific layer doesn't
| Requested feature | What already exists | What's missing |
|---|---|---|
| **Daily Brief** | streak, subjects, notes/quiz data all queryable | no assignments/exams/study-plan data to show "upcoming," no dedicated screen |
| **Weekly Reflection** | `quiz_attempts`, `quiz_answers.topic_tag` (weak/strong subject signal already flagged as backlog in earlier rounds), `ai_usage_log` | no rollup query/screen, no "achievements" concept |
| **Semester Dashboard** | quiz/subject data | no assignments/exams/study-plan tables, no "weekly progress" rollup |
| **Study Health** | quiz/session data exists as raw signal | no analysis layer, no dedicated screen |
| **Premium Features (expansion)** | plan gating pattern fully proven (`profiles.plan`, checked server-side) | offline downloads, AI search, advanced analytics don't exist yet — but the *mechanism* to gate them does |
| **Global Search (non-AI part)** | all the underlying tables (notes, chats, quizzes, flashcards, subjects, study_materials) | no search index/query layer, no UI |

### 🔴 Missing entirely — new tables and/or new subsystems needed
- **GPA Hub** (Calculator, CGPA, Predictor, Target GPA, Goal Tracker,
  Semester Projection) — zero data model for courses/credit-units/grades
  exists. Needs new tables entirely.
- **Focus Mode** (Pomodoro, task list, distraction-free UI) — no timer
  infra, no "today's tasks" concept yet
- **Study Planner / Assignment & Exam Tracker** — no `assignments`, `exams`,
  or `study_sessions` tables. Everything above (Daily Brief, Semester
  Dashboard, Smart Notifications, Smart Exam Mode) depends on this existing
  first.
- **Smart Exam Mode** — depends on Study Planner existing, plus a new
  revision-schedule generation Edge Function
- **Global AI-powered search** — needs a new Edge Function, and realistically
  a `pgvector` embeddings approach for anything beyond keyword matching
- **Widgets** — see constraint below
- **Smart Notifications** — see constraint below
- **Offline note downloads** — no local-storage/download-caching layer exists

---

## 3. Technical constraints worth flagging now, not after building

1. **Widgets require native code.** This project is currently a clean Expo
   Go-compatible app — no custom native modules, no EAS dev client needed.
   Home screen widgets (iOS `WidgetKit`, Android App Widgets) are **not
   achievable in Expo Go** under any circumstance — they need a custom
   development build (`expo-dev-client` + EAS Build) at minimum, likely a
   config plugin or bare native module. This is a real scope/tooling
   decision, not a coding task: it moves the project off "test instantly in
   Expo Go" for good, for this feature specifically. Recommend treating
   Widgets as its own separate initiative, scoped and approved on its own,
   not bundled into this round.
2. **Smart Notifications** — locally-scheduled notifications
   (`expo-notifications`) work fine in Expo Go on this SDK. Remote push
   notifications have had shifting Expo Go support across recent SDKs and
   are more reliable in a dev build. Recommend building the local/scheduled
   piece (exam countdowns, study reminders) first — it covers most of what's
   described — and treating remote push as a later, separately-scoped step.
3. **Offline downloads** are buildable in Expo Go (`expo-file-system` +
   local caching of note text/generated content), no native step required.
4. **GPA scale** — Nigerian tertiary institutions mix 5.0 and 4.0 scales
   depending on the institution. Given this app's existing Nigerian
   education taxonomy (JSS/SSS/100L–500L, WAEC/NECO/JAMB), **please confirm
   which scale(s) to support** — this materially changes the grade-point
   lookup table and whether it needs to be per-institution-configurable.

---

## 4. Proposed new database tables (additive only — nothing existing touched)

Following the project's established pattern (parent/child pairs, RLS on
every table, indexes on FKs):

- **`courses`** — a semester course a student is taking (title, credit
  units, `subject_id` link, semester/year label)
- **`grades`** — a recorded grade for a course (letter/score, grade points,
  links to `courses`)
- **`gpa_targets`** — a student's target GPA/CGPA goal
- **`assignments`** — title, due date, subject/course link, priority,
  completion status
- **`exams`** — title, date, subject/course link, countdown-relevant
- **`study_sessions`** — planned study blocks (date, duration, subject,
  linked task)
- **`weekly_reflections`** — generated rollup snapshots (could instead be a
  computed view rather than a stored table — worth deciding at build time
  based on query cost, not necessary to decide now)

All of these would get `user_id` + RLS scoped exactly like every existing
table, `subject_id` FKs where relevant (reusing Subjects, not duplicating
it), and indexes on the FK columns from the start — the lesson already
learned twice in this project (Subjects sitting dormant, `ai_chat_messages`
needing a retrofit) is to add the FK and index together, not after.

---

## 5. Recommended implementation order

Reasoning: **Study Planner is the load-bearing dependency** for half the
other features (Daily Brief, Semester Dashboard, Smart Exam Mode, Smart
Notifications all need assignments/exams/sessions to exist first). GPA Hub
is fully independent and can happen in parallel or before. Everything
native-code-dependent (Widgets) or infrastructure-heavy (AI Search) goes
last, scoped separately.

1. **Study Planner + Assignment & Exam Tracker** — foundational tables,
   unlocks everything downstream
2. **GPA Hub** — independent, self-contained, high student-value, no
   dependencies on step 1
3. **Daily Brief** — thin dashboard layer over steps 1+2 plus existing
   streak/subject data
4. **Focus Mode** — mostly self-contained (Pomodoro timer, task list pulled
   from step 1), small AI Tutor shortcut integration
5. **Semester Dashboard + Weekly Reflection + Study Health** — these three
   are natural to build together, all rollup/analytics views over data that
   exists by this point
6. **Smart Exam Mode (Premium)** — depends on step 1's scheduling
   infrastructure, adds the revision-plan generation layer on top
7. **Smart Notifications (local/scheduled only)** — depends on steps 1 and 6
   existing to have something to notify about
8. **Global Search (keyword first)** — independent, but low value until
   there's enough content across modules to search
9. **Premium feature gating expansion + offline downloads** — wraps up once
   the features being gated (steps 1–8) actually exist
10. **Global AI-powered search (Premium) + Widgets + remote push
    notifications** — separately scoped later; each has real technical
    prerequisites (embeddings infra, native build tooling, push
    infrastructure) beyond what this phase should absorb

---

## What I need from you before writing any code

1. Tagline: keep "Study smarter. Achieve more." or switch to "StudyNova —
   Your Study Companion"?
2. GPA scale(s) to support (5.0, 4.0, both/configurable)?
3. OK with Widgets being scoped and approved separately, given the native
   build requirement?
4. OK with the order above, or reprioritize?
