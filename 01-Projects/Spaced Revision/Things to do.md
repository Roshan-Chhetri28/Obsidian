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