# StudyApp — Technical Blueprint
*(Replace "StudyApp" with your actual app name throughout — I've used it as a placeholder)*

**Stack:** React Native (Expo) + TypeScript · Supabase (Postgres, Auth, Storage, Edge Functions) · RevenueCat or Stripe for subscriptions

---

## 1. System Architecture

### 1.1 High-Level Overview

```
┌─────────────────────────────────────────────────────────┐
│                    React Native App (Expo)                │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐             │
│  │  Screens   │  │  State    │  │  Services  │             │
│  │ (UI Layer) │──│ (Zustand) │──│  (API SDK) │             │
│  └───────────┘  └───────────┘  └─────┬─────┘             │
└──────────────────────────────────────┼───────────────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    │              Supabase                  │
                    │  ┌────────┐ ┌─────────┐ ┌───────────┐ │
                    │  │  Auth  │ │Postgres │ │  Storage  │ │
                    │  │        │ │  + RLS  │ │(notes/img)│ │
                    │  └────────┘ └─────────┘ └───────────┘ │
                    │  ┌─────────────────────────────────┐  │
                    │  │   Edge Functions (Deno/TS)        │  │
                    │  │  - AI tutor chat proxy            │  │
                    │  │  - Note → flashcard/quiz generator │  │
                    │  │  - Study plan generator            │  │
                    │  │  - Weak-topic analyzer              │  │
                    │  └────────────────┬────────────────┘  │
                    └───────────────────┼───────────────────┘
                                         │
                              ┌──────────┴──────────┐
                              │   Anthropic API      │
                              │  (Claude - tutoring,  │
                              │   note parsing, quiz   │
                              │   generation)          │
                              └───────────────────────┘
```

**Why this shape:**
- The app never calls the AI provider directly — everything goes through Supabase **Edge Functions**. This keeps API keys server-side, lets you rate-limit free-tier usage, log costs per user, and swap AI providers later without touching the app.
- **Row Level Security (RLS)** on every table means a user can only ever see their own data — critical since there's no admin backend yet and you want this secure from day one.
- **Storage** buckets hold uploaded notes (PDF/DOCX/PPTX/images); Edge Functions extract text server-side before sending to Claude.

### 1.2 Client Architecture (React Native / Expo)

```
Navigation:      Expo Router (file-based routing, deep-linking ready)
State:           Zustand (lightweight, no boilerplate) + React Query (server cache)
Forms:           React Hook Form + Zod (validation)
Styling:         NativeWind (Tailwind for RN) — matches design system tokens directly
Auth:            Supabase Auth SDK + expo-secure-store for session persistence
Offline:         WatermelonDB or MMKV + Supabase sync queue for downloaded content
Payments:        RevenueCat (wraps App Store/Play Store subscriptions — avoids building
                 your own receipt validation)
Voice:           expo-speech (TTS) + expo-av / whisper API (STT) for Voice Tutor
```

**Why Zustand + React Query instead of Redux:** less boilerplate, and it cleanly separates *server state* (quizzes, notes, progress — cached/synced via React Query) from *client state* (UI toggles, current quiz session — Zustand). This matters a lot once you add offline mode.

**Why Expo Router:** file-based routes map 1:1 to your screen list below, and it gives you typed routes and deep links (e.g. push notification → jump straight to today's revision session) for free.

### 1.3 Layered Folder Structure (preview — full structure delivered with code)

```
app/                    → Expo Router screens (routes)
src/
  components/           → reusable UI (Button, Card, ProgressRing...)
  features/             → feature modules (ai-tutor, flashcards, planner...)
  services/             → Supabase client, API wrappers
  stores/                → Zustand stores
  hooks/                 → shared hooks
  types/                  → shared TypeScript types (generated from DB schema)
  theme/                  → design tokens (colors, spacing, typography)
```

---

## 2. Database Schema (Supabase / Postgres)

Design principles: every table has `user_id` + RLS policy `auth.uid() = user_id` (or via join for shared/system content); `created_at`/`updated_at` on everything; soft-delete where content might be restored (notes, plans).

### 2.1 Core Tables

**`profiles`** (1:1 with `auth.users`)
| column | type | notes |
|---|---|---|
| id | uuid PK, FK → auth.users.id | |
| full_name | text | |
| avatar_url | text | |
| education_level | text | 'secondary' \| 'university' |
| plan | text | 'free' \| 'premium', default 'free' |
| plan_renews_at | timestamptz | null if free |
| streak_count | int | default 0 |
| last_active_date | date | for streak calculation |
| onboarding_completed | boolean | default false |
| created_at, updated_at | timestamptz | |

**`subjects`** — user's enrolled subjects
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| name | text |
| color | text (hex, for UI) |
| icon | text |
| created_at | timestamptz |

**`study_materials`** — app-provided base content (system-owned, read-only to users)
| column | type |
|---|---|
| id | uuid PK |
| subject_tag | text |
| title | text |
| content | text / jsonb |
| tier_required | text | 'free' \| 'premium' |
| created_at | timestamptz |

**`notes`** — user-uploaded notes (Premium)
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| subject_id | uuid FK → subjects |
| title | text |
| file_path | text | Storage bucket path |
| file_type | text | pdf/docx/pptx/image/text |
| extracted_text | text | populated after Edge Function parsing |
| processing_status | text | pending/processing/done/failed |
| created_at | timestamptz |

**`ai_generated_content`** — everything Claude creates from a note or topic
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| source_note_id | uuid FK nullable | null if generated from a topic, not a note |
| type | text | summary/flashcards/mcq/true_false/fill_blank/short_answer/essay/revision_guide |
| subject_id | uuid FK | |
| content | jsonb | structured payload (see 2.3) |
| created_at | timestamptz |

**`flashcards`** — normalized for spaced-repetition tracking (not just JSON blobs)
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| generated_content_id | uuid FK → ai_generated_content | |
| front | text |
| back | text |
| ease_factor | float | default 2.5 (SM-2 spaced repetition) |
| interval_days | int | default 1 |
| next_review_date | date | |
| times_reviewed | int | default 0 |

**`quizzes`**
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| subject_id | uuid FK |
| generated_content_id | uuid FK nullable | |
| type | text | daily/topic/timed/mock_exam/adaptive |
| title | text |
| created_at | timestamptz |

**`quiz_questions`**
| column | type |
|---|---|
| id | uuid PK |
| quiz_id | uuid FK |
| question_text | text |
| question_type | text | mcq/true_false/fill_blank/short_answer |
| options | jsonb | for MCQ |
| correct_answer | text |
| explanation | text |
| difficulty | text | easy/medium/hard |
| topic_tag | text | used for weak-topic detection |

**`quiz_attempts`**
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| quiz_id | uuid FK |
| score | numeric |
| total_questions | int |
| time_taken_seconds | int |
| completed_at | timestamptz |

**`quiz_answers`** — per-question response, powers weak-topic detection & AI memory
| column | type |
|---|---|
| id | uuid PK |
| attempt_id | uuid FK |
| question_id | uuid FK |
| user_answer | text |
| is_correct | boolean |
| topic_tag | text |

**`study_plans`** — includes normal planner + Exam Survival Mode
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| subject_id | uuid FK |
| mode | text | 'reading_plan' \| 'exam_survival' |
| exam_date | date nullable |
| available_hours_per_day | numeric nullable |
| status | text | active/completed/abandoned |
| created_at | timestamptz |

**`study_sessions`** — individual scheduled sessions within a plan
| column | type |
|---|---|
| id | uuid PK |
| plan_id | uuid FK |
| scheduled_date | date |
| topic | text |
| duration_minutes | int |
| status | text | pending/completed/missed/rescheduled |
| completed_at | timestamptz nullable |

**`assignments`** / **`timetable_entries`** — straightforward planner tables (user_id, title, subject_id, due_date/day+time, recurrence, reminder settings)

**`ai_chat_messages`** — AI Tutor conversation history (powers "AI Memory Tutor")
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| subject_id | uuid FK nullable |
| role | text | user/assistant |
| content | text |
| topic_tags | text[] | extracted for memory/weak-area tracking |
| created_at | timestamptz |

**`weak_topics`** — derived/maintained table, not raw log
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| subject_id | uuid FK |
| topic | text |
| mistake_count | int |
| last_mistake_at | timestamptz |
| mastery_score | numeric | 0–1, decays weak → strong as they improve |

**`progress_snapshots`** — daily rollup for fast dashboard queries (avoid aggregating raw data every load)
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| date | date |
| minutes_studied | int |
| quizzes_completed | int |
| avg_score | numeric |
| exam_readiness_score | numeric nullable |

**`subscriptions`** — mirrors RevenueCat/store state for quick access
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| provider | text | app_store/play_store |
| status | text | active/canceled/expired/trial |
| current_period_end | timestamptz |
| updated_at | timestamptz |

**`ai_usage_log`** — enforces free-tier daily limits, tracks cost
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| feature | text | tutor_chat/note_conversion/quiz_gen/plan_gen |
| tokens_used | int nullable |
| created_at | timestamptz |

### 2.2 Key Relationships
```
auth.users 1─1 profiles
profiles 1─N subjects, notes, quizzes, study_plans, ai_chat_messages
notes 1─N ai_generated_content
ai_generated_content 1─N flashcards (if type=flashcards)
quizzes 1─N quiz_questions 1─N (via quiz_attempts→quiz_answers)
study_plans 1─N study_sessions
```

### 2.3 Example `jsonb` shape for `ai_generated_content.content`
Keeping generated content in `jsonb` (rather than a rigid table per type) lets the AI's output schema evolve without migrations, while flashcards/quiz_questions get normalized out into their own tables *because* those need per-item tracking (spaced repetition, individual answers).

```json
// type: "summary"
{ "sections": [{ "heading": "Cell Structure", "body": "..." }] }

// type: "revision_guide"
{ "key_points": ["..."], "common_mistakes": ["..."], "exam_tips": ["..."] }
```

### 2.4 Row Level Security Pattern
Every user table gets the same two policies:
```sql
create policy "Users manage own rows"
  on notes for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```
`study_materials` (system content) instead uses a policy that checks `tier_required` against the user's `profiles.plan`.

---

## 3. API Structure (Supabase Edge Functions)

The app talks to Supabase directly (via the JS SDK) for standard CRUD — reading notes, writing quiz attempts, etc. Edge Functions are reserved for anything that needs a server (AI calls, file parsing, scheduling logic):

| Function | Trigger | What it does |
|---|---|---|
| `ai-tutor-chat` | User sends chat message | Loads recent `ai_chat_messages` + relevant `weak_topics` for context → calls Claude → streams response → logs to `ai_usage_log`, checks free-tier daily cap first |
| `process-note` | New row in `notes` (or client call after upload) | Downloads file from Storage → extracts text (pdf-parse/mammoth/OCR) → updates `notes.extracted_text` → status `done` |
| `generate-from-note` | User requests flashcards/quiz/summary from a note | Sends `extracted_text` to Claude with a type-specific prompt → writes structured result to `ai_generated_content` (+ normalizes into `flashcards`/`quiz_questions` if applicable) |
| `generate-study-plan` | Exam Survival Mode setup | Takes subject, exam date, available hours → Claude proposes topic breakdown → writes `study_plans` + `study_sessions` rows |
| `reschedule-plan` | Session marked "missed" | Redistributes remaining topics across remaining days, updates `study_sessions` |
| `analyze-weak-topics` | After each `quiz_attempt` | Aggregates `quiz_answers` → upserts `weak_topics` mastery scores |
| `daily-rollup` | Scheduled (cron) | Computes `progress_snapshots` for all active users overnight |
| `revenuecat-webhook` | RevenueCat event | Updates `subscriptions` + `profiles.plan` |

**Why Edge Functions over calling Claude from the client:** free-tier rate limiting has to be enforced server-side or it's trivially bypassed; also keeps your Anthropic API key out of the compiled app.

---

## 4. Design System

### 4.1 Brand Direction
Warm, energetic, focused — closer to Duolingo's encouragement than Notion's minimalism, but with Notion-level information density on dashboard/planner screens. Rounded everything, generous whitespace, one confident accent color rather than a rainbow.

### 4.2 Color Tokens
```
Primary       #6C5CE7  (indigo-violet — "focus" color, used for CTAs, active states)
Primary Light #A29BFE
Secondary     #00D9A3  (mint green — success, streaks, correct answers)
Warning       #FFB020  (exam countdown urgency, missed sessions)
Error         #FF5C5C  (incorrect answers — never harsh red, slightly warm)
Background    #FAFAFC  (light) / #121218 (dark)
Surface       #FFFFFF  (light) / #1C1C24 (dark)
Text Primary  #1A1A2E  (light) / #F2F2F7 (dark)
Text Muted    #6B7280
```

### 4.3 Typography
```
Font: Inter (or Plus Jakarta Sans — both free, geometric, student-friendly)
Display   32/40  bold      — dashboard headers
H1        24/32  bold      — screen titles
H2        20/28  semibold  — section headers
Body      16/24  regular   — main content
Body Sm   14/20  regular   — secondary text
Caption   12/16  medium    — labels, tags
```

### 4.4 Spacing & Radius
```
Spacing scale: 4, 8, 12, 16, 24, 32, 48 (px)
Radius: cards 16px, buttons 12px, pills/chips 999px (full)
Elevation: soft, colored shadows (not gray) — e.g. primary-tinted shadow under CTA buttons
```

### 4.5 Core Components
`Button` (primary/secondary/ghost), `Card`, `ProgressRing` (exam readiness, streak), `StreakFlame`, `SubjectChip`, `FlashcardStack` (swipeable), `QuizOptionTile`, `TimelineItem` (planner), `AIChatBubble`, `UpgradeBanner`, `EmptyState`, `LoadingSkeletons` (not spinners — feels faster).

### 4.6 Motion
Subtle spring transitions (react-native-reanimated) on: streak increment, correct-answer feedback (small bounce + haptic), card swipes, tab transitions. Avoid gratuitous animation on data-heavy screens (dashboard, planner) — keep those snappy.

---

## 5. Feature Implementation Plan (Phased)

**Phase 1 — Foundation**
Auth (Supabase), profile setup, onboarding, design system + core components, navigation shell, dashboard skeleton with mock data.

**Phase 2 — Core Free Experience**
Study materials browsing, basic AI tutor chat (rate-limited), daily/topic quizzes, timetable + assignment tracker, streak tracking, progress dashboard v1.

**Phase 3 — Premium Core**
Subscription paywall + RevenueCat integration, note upload + processing pipeline, AI note converter (summary/flashcards/quiz), spaced-repetition flashcard review.

**Phase 4 — Smart Features**
Exam Survival Mode (plan generation + auto-reschedule), weak-topic detection, AI Memory Tutor context injection, exam readiness score.

**Phase 5 — Differentiators & Polish**
Voice Tutor, offline mode (downloaded notes/flashcards/quizzes), adaptive quizzes, analytics polish, animations pass, accessibility audit.

---

## 6. Open Decisions Before Coding Starts

A few things worth deciding now since they affect the schema/screens:
1. **App name** — still a placeholder; affects bundle ID, Storage bucket naming.
2. **AI cost control for free tier** — suggest: daily cap (e.g. 10 tutor messages/day, tracked via `ai_usage_log`) rather than a hard feature lock, so free users still get a taste.
3. **Which core screens to prototype first** in Phase 1 code — I'd suggest: Splash → Onboarding → Home Dashboard → AI Tutor Chat, since those establish the design system fastest and are highest-visibility.

Let me know if you want changes to the schema or design tokens — once you approve, I'll scaffold the Expo project and start building screen by screen.
