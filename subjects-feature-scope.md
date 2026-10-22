# Feature Scope: Subject/Course Organization

Activates the existing `subjects` table and the `subject_id` foreign keys already
sitting unused on `notes`, `quizzes`, `ai_generated_content`, and `ai_chat_messages`.
Scoped before coding, per your request.

---

## 1. Database changes

**Good news: the core schema is already right.** `subjects` exists
(`id`, `user_id`, `name`, `color`, `icon`, `created_at`), RLS is already correct
(`auth.uid() = user_id`), and all four FKs are already `on delete set null` —
deleting a subject safely un-links its content instead of cascading deletes.
No retrofit needed there.

**What actually needs to change:**

| Change | Why |
|---|---|
| Add indexes: `idx_notes_subject_id`, `idx_quizzes_subject_id` *(already exists)*, `idx_ai_chat_messages_subject_id`, `idx_ai_generated_content_subject_id` | These columns have never been queried against; filtering Notes/Practice by subject will do it constantly |
| Add a unique constraint: `(user_id, name)` on `subjects` | Prevents "Biology" + "biology" + "Biology " duplicates from a lightweight create-inline flow with no validation UI |
| **Design decision, not a schema change:** "Uncategorized" is *not* a real subject row — it's just `subject_id = null`, same as today | Avoids needing a placeholder row per user and special-cases deleting it |
| When flashcards ship (Priority 3), `flashcard_decks` should include `subject_id` from day one | Retrofitting a FK onto existing decks later is more work than just including it now — worth sequencing flashcards *after* or *alongside* this feature, not before |

That's it — one migration, small.

---

## 2. Screens needed

I'd avoid a 6th tab for this — subjects are an organizing layer, not a
destination people navigate to on their own. Concretely:

**A. Subject Picker (new, reusable bottom sheet)**
The one new real UI surface. Shows existing subjects as chips (color + icon),
plus a "+ New Subject" inline input (name only for v1 — color/icon
auto-assigned from a fixed rotation, not user-chosen, to avoid building a
color/icon picker for a first version). Used from three places:
- Notes upload (assign a subject when uploading, or after — see flow below)
- Practice tab's manual quiz generator (optional)
- Tutor (to start/switch a subject-scoped general conversation)

**B. Notes tab — add a filter row**
Horizontal chip row at the top: `All | Uncategorized | Biology | Chemistry | +`.
Tapping a chip filters the existing list (client-side filter on already-fetched
notes — no new query needed). Tapping "+" opens the Subject Picker directly.

**C. Practice tab — same filter row on quiz history**, same pattern as Notes.
Quiz-from-note already inherits the note's `subject_id` automatically, no
extra prompt needed there.

**D. Tutor — small subject indicator/switcher in the header**
Today the header only shows a note-context chip when relevant. Add a subject
selector next to it for general (non-note) chats — this actually **reactivates
dead code**: `ai-tutor-chat` already threads on `subject_id` when no `note_id`
is present, but nothing has ever sent one, so every general conversation has
been one flat undifferentiated thread. Wiring this up gets you subject-scoped
tutor memory for free.

**E. Dashboard — optional, not required for v1**
A subject chips strip for quick navigation is a nice next step but isn't load-
bearing for the feature to work. I'd cut it from v1 and revisit once subjects
have real usage data.

No standalone "My Subjects" management screen in v1 — rename/delete can live
as a long-press or swipe action on the chips in the Picker/filter row. A
dedicated management screen is an easy add later if renaming/deleting turns
out to be a common need; not worth building speculatively now.

---

## 3. User flow

```
First note upload (no subjects exist yet):
  Upload a note → Subject Picker opens → only option is "+ New Subject"
  → type "Organic Chemistry" → creates subjects row → note tagged → upload proceeds
  (Skip option available → note stays subject_id: null / "Uncategorized")

Subsequent uploads:
  Upload a note → Subject Picker shows existing chips + "+ New" → pick or create → upload proceeds

Browsing:
  Notes tab → filter chip row → tap "Chemistry" → list filters client-side
  Practice tab → same pattern on quiz history

Quiz generation:
  From a note's "Generate Quiz" button → auto-inherits the note's subject, no prompt
  From Practice tab's manual topic entry → optional subject chip appears once ≥1 subject exists

AI Tutor:
  Default landing → general chat, unscoped (today's behavior, unchanged)
  Tap subject selector in header → switches to that subject's general thread
  "Ask AI About This Note" → unchanged, still note_id-scoped independent of subject

Managing:
  Long-press a chip → rename / delete (delete just un-links content, doesn't touch it)
```

---

## 4. Sequencing note

Given flashcards (Priority 3) are still pending and would want `subject_id`
from the start, I'd suggest one of two orders — your call:
- **Subjects first, then flashcards** (flashcards get subject support natively, no retrofit)
- **Build both together** (slightly more surface area in one pass, but the Subject Picker gets reused by flashcard creation immediately instead of being built and then extended)

I'd lean toward subjects first — smaller, self-contained, and de-risks the
flashcards work rather than bundling two new things at once.

---

## What I need from you before coding

1. Approve the DB changes (indexes + unique constraint — no structural surprises)
2. Confirm the "no dedicated management screen, no color/icon picker" scope
   cuts for v1 are acceptable, or want them included
3. Confirm sequencing: subjects before flashcards, or bundled
