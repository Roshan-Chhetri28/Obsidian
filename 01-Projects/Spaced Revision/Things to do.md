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

5. Videos link to subject_id (spaced-revision-sern-backend): run 3 migrations on prod DB — videos group by subject instead of topic
	1. `node database/migrations/addVideoSubjectId.js` — adds subject_id + FK + index to `video` and `youtube`, backfills from topics via topic_id
	2. `node database/migrations/addVideoSubjectOrder.js` — creates `video_subject_order` + `youtube_subject_order` (per-subject video ordering, no backfill)
	3. `node database/migrations/addVideoHidingSubjectId.js` — adds subject_id + FK + index to `video_hiding_mapping` and `youtube_hiding_mapping`, backfills from topics
	4. run in that order; all idempotent — safe to re-run, second run is a no-op
	5. must run BEFORE the backend deploy — the new `/api/video/all/*` queries read subject_id and the order tables, so the routes 500 without them
	6. `topic_id` is NOT dropped and is STILL WRITTEN — it is the key cloned courses resolve through via topic_mappings. A video with a null topic_id can never reach a clone
	7. expected backfill: video 1196/1196, youtube 286/290 (4 rows have no topic_id, stay invisible as they are today)
	8. rollback = drop the added columns + drop the 2 new tables; no rows are deleted by any of the three
