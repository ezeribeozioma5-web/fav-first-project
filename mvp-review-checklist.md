# LearnGenie MVP — Final Review Checklist

## 1. Schema migrations run cleanly
✅ Confirmed. Schema is now in `supabase/migrations/20260101000000_initial_schema.sql` (proper Supabase CLI migration format — `supabase db push` will pick it up), with `supabase/schema.sql` kept as an identical copy for manual paste-into-SQL-Editor setups. `supabase/config.toml` added documenting both workflows.

Verified table creation order has no forward-reference errors: `profiles` → `subjects` → `notes` → `ai_generated_content` → `ai_chat_messages` → `ai_usage_log` (+ `check_and_log_ai_usage()`) → `quizzes` → `quiz_questions` → `quiz_attempts` → `quiz_answers`. Every foreign key points to a table already created earlier in the file.

**Fixed during this review:** `profiles.updated_at` existed but nothing was maintaining it — added a `touch_updated_at()` trigger.

## 2. Every table has RLS
✅ Confirmed, table by table:

| Table | Policy |
|---|---|
| `profiles` | select/update where `auth.uid() = id`; insert only via the security-definer signup trigger (no client insert policy — intentional) |
| `subjects`, `notes`, `ai_generated_content`, `ai_chat_messages`, `quizzes`, `quiz_attempts` | `for all` where `auth.uid() = user_id` |
| `quiz_questions` | access via parent `quizzes.user_id` (no direct `user_id` column on the table itself) |
| `quiz_answers` | access via parent `quiz_attempts.user_id` |
| `ai_usage_log` | select-only for the owning user; inserts happen exclusively through `check_and_log_ai_usage()` (`SECURITY DEFINER`), never a direct client write |

No table is missing `enable row level security`.

## 3. Env vars / secrets documented
✅ Confirmed in `README.md`:
- App: `EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_ANON_KEY` (`.env`)
- Edge Functions: `ANTHROPIC_API_KEY` via `supabase secrets set` (`SUPABASE_URL`/`SUPABASE_ANON_KEY` are auto-injected by Supabase, correctly not listed as something you set yourself)

## 4. Auth flow end-to-end
✅ Traced the full path:
`Splash → (session? tabs : onboarding_seen? login : onboarding) → onboarding → login/signup → profiles row auto-created via DB trigger on signup → (tabs)`. Route guarding is centralized once in `app/_layout.tsx` (not duplicated per-screen), so a signed-out user can't land on a tab screen by deep link, and a signed-in user can't get stuck on login. Sign-out clears both `session` and `profile` state and the guard effect kicks them back out immediately.

## 5. Error/edge-case handling

| Scenario | Handling |
|---|---|
| **No internet** | **Added this review.** New `useIsOffline` hook (`@react-native-community/netinfo`) + app-wide `OfflineBanner`. Tutor chat, quiz generation, and note upload all short-circuit with a plain-language offline message instead of attempting (and failing) a network call. |
| **AI daily limit reached** | Enforced server-side in both `ai-tutor-chat` and `generate-quiz` via `check_and_log_ai_usage()`; returns `429` + structured message; client shows the upgrade-prompt state, not a generic error. |
| **Failed uploads** | `notes.tsx` catches Storage/insert/function errors, flips the note card to a `failed` status with a "tap to retry" label. |
| **Failed AI responses** | All three Edge Functions wrap Claude calls in try/catch, translate upstream failures (rate limit, empty response, malformed JSON) into clean user-facing messages, and never leak raw error text. Client surfaces `error.context.message` rather than a stack trace. |

## What changed in this pass
1. `supabase/migrations/` added (CLI-ready), `config.toml` added.
2. `profiles.updated_at` trigger fix.
3. Offline detection: `useIsOffline` hook, `OfflineBanner`, wired into tutor/quiz/notes.

Everything else from prior rounds (schema, Edge Functions, RLS, env docs) held up under review without changes needed.
