# LearnGenie — MVP Structure Review

Everything below reflects what's actually in the delivered `learngenie-mvp.zip`, plus the schema/flow decisions needed before the three Edge Functions get built.

---

## 1. App Architecture

```
┌────────────────────────────┐
│   React Native (Expo)        │
│   Screens → Zustand/RQ →     │
│   supabase-js client          │
└──────────────┬───────────────┘
               │
      ┌────────┴─────────┐
      │     Supabase       │
      │  Auth / Postgres /  │
      │  Storage / RLS       │
      │                       │
      │  Edge Functions (Deno)│  ← NOT YET BUILT
      │  - ai-tutor-chat       │
      │  - generate-quiz        │
      │  - process-note          │
      └────────┬──────────────┘
               │
      ┌────────┴────────┐
      │  Anthropic API    │
      └──────────────────┘
```

**Rule that hasn't changed:** the app never talks to Claude directly. Every AI call goes through an Edge Function, so the API key stays server-side and daily limits can't be bypassed by a modified client.

---

## 2. Folder Structure (as delivered)

```
learngenie/
├── app/                          ← Expo Router (file-based routes)
│   ├── index.tsx                 ← animated splash, redirects on mount
│   ├── onboarding.tsx
│   ├── (auth)/
│   │   ├── login.tsx
│   │   └── signup.tsx
│   ├── (tabs)/
│   │   ├── _layout.tsx           ← tab bar config
│   │   ├── index.tsx             ← home dashboard
│   │   ├── tutor.tsx             ← AI tutor chat
│   │   ├── notes.tsx             ← notes upload
│   │   ├── quiz.tsx              ← quiz generator
│   │   └── profile.tsx
│   └── _layout.tsx               ← fonts, auth listener, route guard
├── src/
│   ├── components/                ← Button, Card, StreakBadge
│   ├── services/supabase.ts       ← single client instance
│   ├── stores/useAuthStore.ts     ← session + profile (Zustand)
│   ├── theme/tokens.ts            ← colors, spacing, type scale, limits
│   └── types/database.ts          ← hand-written types (→ auto-gen later)
├── supabase/
│   └── schema.sql
└── README.md
```

Where the three Edge Functions will live once built: `supabase/functions/ai-tutor-chat/`, `supabase/functions/generate-quiz/`, `supabase/functions/process-note/` — each a self-contained Deno script deployed via the Supabase CLI.

---

## 3. Frontend Flow

```
Splash (index.tsx)
  → session exists?        → (tabs)/index  [Home]
  → onboarding_seen?       → (auth)/login
  → neither                → onboarding → (auth)/login

(auth)/login ⇄ (auth)/signup
  → on success → (tabs)/index

(tabs) — persistent tab bar
  Home ──┬── quick action → Notes
         ├── quick action → Tutor
         └── quick action → Quiz
```

Route guarding happens once, centrally, in `app/_layout.tsx` (checks `segments[0]` against session state) — individual screens don't each re-implement auth checks.

---

## 4. Supabase Integration Approach

| Concern | Approach |
|---|---|
| Auth | `supabase.auth` directly from client; session persisted via `AsyncStorage`, synced into `useAuthStore` |
| Reads/writes on user's own data | Direct client calls (`supabase.from(...)`), protected by RLS — no Edge Function needed for plain CRUD |
| File uploads | Client uploads straight to Storage bucket `notes` (private, path-scoped by `user_id`), then inserts a `notes` row |
| Anything touching Claude | Routed through Edge Functions only (`supabase.functions.invoke(...)`) |
| Daily limit enforcement | Server-side, inside Edge Functions, via the `check_and_log_ai_usage()` Postgres function — client-side counters are optimistic UI only |
| Type safety | `src/types/database.ts` hand-maintained now; swap for `supabase gen types` once schema stabilizes |

---

## 5. Database Schema (finalized for MVP + near-term additions)

### 5.1 Already in `schema.sql`
- **`profiles`** — 1:1 with `auth.users`; plan, streak_count, xp_points, onboarding flag. Auto-created via trigger on signup.
- **`subjects`** — user-scoped tagging for notes/chat/content.
- **`notes`** — file metadata + `processing_status` (`pending → processing → done/failed`) + `extracted_text`.
- **`ai_generated_content`** — flexible `jsonb` store for summaries/flashcards/quiz question sets, linked to a `source_note_id` or standalone.
- **`ai_chat_messages`** — full tutor conversation history, `role` + `content` + optional `topic_tags`.
- **`ai_usage_log`** + `check_and_log_ai_usage()` — atomic check-and-increment for daily caps.

### 5.2 New tables needed before the Edge Functions are useful (adding now)

**`quizzes`**
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| subject_id | uuid FK nullable |
| source_note_id | uuid FK nullable |
| topic | text |
| title | text |
| created_at | timestamptz |

**`quiz_questions`**
| column | type |
|---|---|
| id | uuid PK |
| quiz_id | uuid FK |
| question_text | text |
| question_type | text (mcq/true_false/fill_blank/short_answer) |
| options | jsonb nullable |
| correct_answer | text |
| explanation | text |
| topic_tag | text nullable |

**`quiz_attempts`** (MVP-lightweight — full scoring history is Phase 2, but recording *that* a quiz happened is needed now to make "quizzes done" on the dashboard real instead of hardcoded `0`)
| column | type |
|---|---|
| id | uuid PK |
| user_id | uuid FK |
| quiz_id | uuid FK |
| score | numeric nullable |
| total_questions | int |
| completed_at | timestamptz nullable |

### 5.3 Progress tracking — MVP scope
Full `progress_snapshots` analytics (weekly rollups, exam readiness score) stay deferred to Phase 2 as originally planned. For the MVP, "progress" is derived directly and cheaply from existing tables:
- **Streak** → `profiles.streak_count` + `last_active_date`, incremented by a small Postgres function called whenever a quiz is completed or a note is processed.
- **Weekly stats on Home** (currently hardcoded `0`) → replace with a lightweight query: `count(quiz_attempts)` and `sum` over the last 7 days. No new table required yet — added once the query cost actually matters.

This keeps the MVP additions minimal: **3 new tables** (`quizzes`, `quiz_questions`, `quiz_attempts`), everything else reuses what's already built.

---

## 6. Design System (unchanged, confirmed as final for MVP)

**Colors:** Primary `#6C5CE7` (indigo-violet), Secondary `#00D9A3` (mint), Warning `#FFB020`, Danger `#FF5C5C`, backgrounds/surfaces with light+dark variants — all defined in `tailwind.config.js` and mirrored in `src/theme/tokens.ts`.

**Typography:** Inter, 6-step scale (Display 32 → Caption 12), weights 400–700.

**Components built:** `Button` (primary/secondary/ghost, haptic feedback, loading state), `Card`, `StreakBadge`. Still needed as screens grow: `EmptyState` (notes screen currently inlines its empty state — worth extracting), `ChatBubble` (currently inlined in tutor.tsx), `LoadingSkeleton`.

**UI patterns established:** rounded-card (16px) surfaces, pill-shaped tags/badges, soft primary-tinted CTAs, status-driven text color (muted → primary → secondary/danger) for async states like note processing.

---

## 7. Data Flow — the three flows the Edge Functions must implement

### 7.1 Notes: upload → processing → AI summary
```
1. Client picks file → uploads to Storage bucket `notes/{user_id}/{id}-{filename}`
2. Client inserts `notes` row, status = 'pending'
3. process-note Edge Function (triggered by client call right after insert,
   OR by a Storage webhook — recommend client call for MVP, simpler to debug):
     a. Downloads file from Storage
     b. Extracts text (pdf-parse / mammoth / OCR depending on file_type)
     c. Updates notes.extracted_text, processing_status = 'done' (or 'failed')
4. Client polls or subscribes (Supabase Realtime) to the notes row for status
5. Once 'done', user can request a summary/flashcards/quiz →
   generate-from-note logic (can live inside generate-quiz or a 4th function
   later — for MVP, folding "generate summary" into process-note's follow-up
   step is out of scope; only flashcard/quiz generation ships now)
```

### 7.2 AI Tutor messages: storage model
```
1. Client calls ai-tutor-chat Edge Function with { message, subject_id? }
2. Function:
     a. Checks daily cap via check_and_log_ai_usage(user_id, 'tutor_chat', 10)
        → if false, return 429-style error, client shows upgrade prompt
     b. Loads last N ai_chat_messages for context (short rolling window,
        not the entire history — keeps token cost bounded)
     c. Calls Claude with system prompt + recent context + new message
     d. Inserts BOTH the user message and assistant reply into
        ai_chat_messages (two rows) — so history is always reconstructable
        from the DB alone, not just from client state
     e. Returns { reply } to client
```

### 7.3 Quiz generation
```
1. Client calls generate-quiz with { topic } OR { source_note_id }
2. Function:
     a. Checks daily cap via check_and_log_ai_usage(user_id, 'quiz_generation', 3)
     b. If source_note_id given, pulls notes.extracted_text as context;
        else uses topic string directly
     c. Calls Claude, requesting strict JSON: array of questions with
        type/options/correct_answer/explanation/topic_tag
     d. Inserts one `quizzes` row + N `quiz_questions` rows
     e. Returns the quiz_id (+ questions) to client
3. Client (Phase 2 UI) presents questions, writes a `quiz_attempts` row on completion
```

---

## 8. What I need from you before building

1. **Approve the 3 new tables** (`quizzes`, `quiz_questions`, `quiz_attempts`) — or flag if you'd rather keep quiz storage inside the existing `ai_generated_content` jsonb blob for now and normalize later.
2. **process-note trigger**: client-invoked right after upload (simpler, recommended for MVP) vs. Storage webhook (more automatic, more moving parts) — I'll default to client-invoked unless you say otherwise.
3. Anything in the flows above that doesn't match what you had in mind?

Once you confirm, I'll add the 3 tables to `schema.sql` and build the Edge Functions against this exact structure.
