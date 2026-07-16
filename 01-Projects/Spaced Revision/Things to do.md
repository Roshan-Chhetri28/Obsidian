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
	6. add env vars on prod (defaults exist in scheduler, set explicit): `MCQ_NOTIF_HOUR=17`, `MCQ_NOTIF_MINUTE=30`

5. Video subject-bucketing (spaced-revision-sern-backend, branch feat/practice-tagging-admin): run DB migration on prod BEFORE the new backend code serves traffic
	1. `node database/migrations/addVideoSubjectId.js`
	2. adds `subject_id` (+FK, index) to `video`, `youtube`, `video_hiding_mapping`, `youtube_hiding_mapping`; backfills from each row's topic
	3. creates `video_subject_order` + `youtube_subject_order` and backfills display order — expect **2236 rows across 152 buckets**, contiguous 1..N, 0 collisions
	4. idempotent — re-run skips the backfill once the order tables are non-empty (it is all-or-nothing on whole-table emptiness, NOT per-bucket)
	5. order backfill preserves the order educators actually saw (ranked by display topic NAME -> priority -> id); to rebuild from scratch, empty both order tables first
	6. heads-up: `GET /api/video/all/:course_id` returns slightly MORE videos after this (432 vs 429 for course 2160, 379 vs 369 for 1112) — dedupe is now by (kind, id) not by URL, so distinct rows sharing a URL stop being silently swallowed. Expected, not a bug.
	7. After change: 
		1. Buckets: ORDER BY subjects.priority, subjects.name — surfaced as the new subject_order field.
		2. Within a bucket: the order tables' priority 1..N, then upload date ASC, then id — one sequence spanning HLS and YouTube, seeded from the old name-based order so nothing visibly moved.
