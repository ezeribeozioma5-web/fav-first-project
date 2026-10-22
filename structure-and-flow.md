# LearnGenie — Current Structure & User Flow

## User flow (as built)

```
Splash (1.4s, animated logo)
  │
  ├─ session exists ──────────────────→ Home Dashboard
  ├─ no session, onboarding seen ─────→ Login
  └─ no session, first launch ────────→ Onboarding (3 slides, skippable)
                                             │
                                             ▼
                                          Login ⇄ Signup
                                             │ (success)
                                             ▼
                                        Home Dashboard
                                             │
                        ┌────────────────────┼────────────────────┐
                        ▼                    ▼                    ▼
                  Upload Notes          AI Tutor Chat        Practice Quiz
                (pick file → Storage   (message → cap check   (topic → cap check
                 → process-note →       → Claude → stored      → Claude → quiz +
                 extracted text)        both turns → reply)     questions saved)
                        │
                        ▼
              (Phase 2, not yet built)
              generate-quiz / flashcards
              from the processed note
```

Tab bar (Home / AI Tutor / Notes / Practice / Profile) is persistent once past auth — this is the "Upload Notes → Understand with AI → Generate Practice → Improve Learning" loop from the brief, currently wired end-to-end through Upload → Tutor/Quiz, with the "note → quiz" connection present in `generate-quiz` (accepts `source_note_id`) but no dedicated button on the Notes screen yet linking the two (worth flagging as a near-term polish item).

## Folder structure (unchanged from last review, now with migrations)

```
learngenie/
├── app/                        Expo Router screens
│   ├── index.tsx                splash
│   ├── onboarding.tsx
│   ├── (auth)/login.tsx, signup.tsx
│   ├── (tabs)/index.tsx, tutor.tsx, notes.tsx, quiz.tsx, profile.tsx
│   └── _layout.tsx              fonts, auth listener, route guard, offline banner
├── src/
│   ├── components/               Button, Card, StreakBadge, OfflineBanner
│   ├── hooks/useIsOffline.ts
│   ├── services/supabase.ts
│   ├── stores/useAuthStore.ts
│   ├── theme/tokens.ts
│   └── types/database.ts
├── supabase/
│   ├── config.toml
│   ├── schema.sql                (kept in sync with the migration, for manual paste)
│   ├── migrations/20260101000000_initial_schema.sql
│   └── functions/
│       ├── _shared/ (cors, supabaseClient, anthropic)
│       ├── ai-tutor-chat/
│       ├── generate-quiz/
│       └── process-note/
└── README.md
```

## Screenshots
See `screen-preview.html` — a static mockup of the 5 priority screens built from the app's actual design tokens (not a live render; open it in a browser for the closest visual approximation without running the simulator).
