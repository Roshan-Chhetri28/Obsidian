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

5. Videos link to subject_id (spaced-revision-sern-backend): run ONE migration script on prod DB — videos group by subject instead of topic
	1. `node database/migrations/runAllVideoSubjectMigrations.js` — that's the whole thing; runs all 3 steps in order and stops on first failure
	2. it wraps: addVideoSubjectId (subject_id + FK + index on `video`/`youtube`, backfilled from topics via topic_id) → addVideoSubjectOrder (creates `video_subject_order` + `youtube_subject_order`, no backfill) → addVideoHidingSubjectId (subject_id + FK + index on `video_hiding_mapping`/`youtube_hiding_mapping`, backfilled from topics)
	3. idempotent — safe to re-run; a partial failure is fixed by just re-running the whole script. Each step is also runnable standalone for debugging
	4. must run BEFORE the backend deploy — the new `/api/video/all/*` queries read subject_id and the order tables, so the video routes 500 without them
	5. `topic_id` is NOT dropped and is STILL WRITTEN — it is the key cloned courses resolve through via topic_mappings. A video with a null topic_id can never reach a clone
	6. expected backfill: video 1196/1196, youtube 286/290 (4 rows have no topic_id, stay invisible as they are today)
	7. rollback = drop the added columns (`video.subject_id`, `youtube.subject_id`, `video_hiding_mapping.subject_id`, `youtube_hiding_mapping.subject_id`) + drop the 2 new tables; no rows are deleted and no column is dropped by the script
