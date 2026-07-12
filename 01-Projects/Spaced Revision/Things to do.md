---
Date: 2026-05-20
tags:
  - Task
Status:
  - Not Urgent
  - Important
---
1. Write test for
	1. web-hook
	2. frontend: test 
	3. analytics
2. Fix the video upload notification in web-socket server 
	1. when event 'successful' is received then the notification API should be called
3. Anki markup fix (spaced-revision-sern-backend): run backfill migration on prod DB — fixes cloze/image-occlusion gibberish in already-imported deck cards
	1. `node database/migrations/normalizeDeckCardMarkup.js`
	2. flush Redis deck cache: `redis-cli --scan --pattern 'user_decks:*' | xargs redis-cli DEL`
	3. idempotent — safe to re-run; re-run updates 0 rows

4. Notification work (spaced-revision-sern-backend, branch main — uncommitted): commit + deploy
	1. MCQ-added push: fixed bug where educator's name showed instead of receiver — title now greets recipient ("Hi <student>, new MCQ added")
	2. MCQ-added push: body now "New MCQ(s) in <topic> - <course>" (no subject); tap deep-links into that topic's MCQs (route 'Study') instead of just the course
	3. Study reminder: due counts >5 shown as "5+", word "cards" -> "flashcards", body "5+ flashcards and 5+ MCQs due for revision 🔥"
	4. Refactor: study-reminder body logic extracted to notifications/studyReminderMessage.js
	5. Tests added: mcqAddedWorker.test.js (5), studyReminderMessage.test.js (9) — full suite green (774 pass)
