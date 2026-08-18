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
6. FCM token bloat fix (spaced-revision-sern-backend, branch push/version): staged, ORDER IS LOAD-BEARING
	1. Root cause: client sends snake_case `device_id`/`app_version` but route read camelCase → device_id stored NULL on 100% of tokens → per-device dedup dead → token rotations pile up new active rows (user 8483 had 90 active tokens; ~471 active dupes overall)
	2. Stage 1 (DONE in code, commit + deploy): `push_tokens.js` reads snake_case; `FcmToken.service.js` deactivates a device's other live tokens on register; OpenAPI updated; tests green (FcmToken.test.js 3, push_tokens.test.js 5)
	3. Stage 1 verify after deploy: `SELECT SUM(device_id IS NOT NULL) has_dev, COUNT(*) total FROM fcm_tokens WHERE is_active=1 AND created_at > '<deploy_ts>';` → has_dev must equal total for post-deploy rows
	4. Stage 2 (ONLY after Stage 1 is live on prod, else bloat regrows): run one-time cleanup sweep on PRIMARY (MySQL MCP is a read-only replica) — new script `database/migrations/dedupeFcmTokens.js`
		1. retire ~471 active dupes: keep newest per (user_id, platform), set is_active=0 on the rest
		2. optional purge ~1901 dead rows: `DELETE ... WHERE is_active=0 AND last_seen_at < NOW()-INTERVAL 90 DAY`
		3. dry-run SELECT counts first, wrap in transaction, back up fcm_tokens, off-peak
		4. verify: `SELECT COUNT(*) FROM (SELECT 1 FROM fcm_tokens WHERE is_active=1 GROUP BY user_id,platform HAVING COUNT(*)>1) d;` → 0
	5. Stage 3 (app release, react-native-app): `getDeviceId()` → `react-native-device-info` `getUniqueId()` (dep already installed) for stable IDFV/ANDROID_ID that survives reinstall; replace legacy random id for all installs; fix non-persisted fallback collision; then re-run Stage 2 sweep once to clear migration re-bloat
7. Meta ads attribution (spaced-revision-sern-backend + react-native-app, branch feat/meta-ads-attribution): ORDER IS LOAD-BEARING
	1. Goal: make Android installs + genuine study engagement visible to Meta so app-install campaigns can be measured and optimised. Primary conversion is `Activated` = a user's first 10 flashcards, ONCE per user (purchase is ~2/wk, far below Meta's ~50/ad-set/wk learning threshold; registration is high-volume but weak)
	2. Step 1 — run migration on PROD BEFORE the backend code serves traffic: `node database/migrations/addMetaActivatedAt.js` (adds `users.meta_activated_at DATETIME NULL`, INSTANT on MySQL 8, idempotent). Column currently exists on local dev DB ONLY. The activation gate reads it on every sync push
	3. Step 2 — add env vars on prod: `META_CAPI_ACCESS_TOKEN` (system-user token, SECRET), `META_APP_ID=1348293100400435`, `META_GRAPH_VERSION=v21.0`, and `META_CAPI_ENABLED=false` for now
	4. Step 3 — deploy backend AND the RN app together. Backend alone burns the once-per-user activation silently: the gate stamps `meta_activated_at`, but with CAPI disabled and no app handler deployed, nothing reaches Meta and that user can never fire `Activated` again
	5. Step 4 — flip `META_CAPI_ENABLED=true` only after both are live, then confirm a real `Activated` in Events Manager. Meta will NOT offer `Activated` as a campaign optimisation goal until it has received at least one
	6. Step 5 — RELEASE BUILD CHECK, highest silent-failure risk: `enableProguardInReleaseBuilds = true` and the existing RN keep-rules only cover `com.facebook.react/.hermes/.jni`. New `-keep class com.facebook.**` rules were added, but must be verified on an actual release APK — if wrong, debug keeps working and production sends nothing, with no error anywhere
	7. Step 6 — Play Console, in the SAME submission as the release or Play rejects the build: Advertising ID declaration (AD_ID permission now declared) + Data safety marking Advertising ID both collected AND shared, plus Email/Phone shared (hashed advanced matching)
	8. Gotcha: app events are NOT the pixel Conversions API — `POST /{app_id}/activities`, form-urlencoded, `event=CUSTOM_APP_EVENTS`, `_eventName` not `event_name`. Posting the pixel shape fails with "Object with ID ... does not support this operation", which reads like a permissions problem. `extinfo` is required, positional, 16 slots, indices 9-14 MUST be numbers
	9. Gotcha: Purchase `value` must never come from `iap_purchases.amount` (hardcodes INR) or `courses.price` (catalogue, not charged) — it comes from the client's `priceAmountMicros`/`priceCurrencyCode`
	10. Known gaps (not blockers): `extinfo` device fields not yet wired from `react-native-device-info` so match quality is weaker than it could be; no test covers the iapService Meta call site; web pixel cohort tagging untouched; iOS entirely out of scope; only ~12 of ~1391 courses carry a UPSC/SPSC tag so cohort reporting covers a fraction of the funnel
	11. Cleanup: 3 synthetic smoke-test events were sent to the live dataset while verifying (2 `Activated`, 1 `fb_mobile_purchase`, all with unmatchable identities)
8. QB owner-scoping fix (spaced-revision-sern-backend + spaced-revision-sern-frontend): phase checklist in [[QB Owner Scoping Fix]]
	1. Another educator's MCQs (author 62 "Pedestal Education") were mapped into course 100037 and count toward the progress denominator — Economics meter stuck at 5/9. Only the **mapped** branch of every query was never owner-scoped; the direct branch already is
	2. Blast radius is 8 MCQs total (4 in course 100037 topic 4416, 4 in course 2160 topics 3030/3034); 0 cards. All other mappings are owner-authored, so the other ten courses' totals must not move
	3. **PROD after deploy:** flush `course_progress:*` in Redis (or bump the cache-key prefix) — `analytics.js:91` caches for 300s, so the meter won't move otherwise
	4. **Security:** `PUT /mcq/assign/:mcq_id/:course_id/:subject_id/:topic_id` (`mcq.js:1379`) is `auth`-only — any logged-in user can map any MCQ into any paid course. That is how the foreign content got in. Being fixed in Phase 2
	5. Also fixes, same page: "MCQ · Question 97 of 93" counter overrun and the dead whitespace under the last question in the Question Bank list view
