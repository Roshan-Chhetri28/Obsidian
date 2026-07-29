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

7. Meta Ads attribution — Android (react-native-app + spaced-revision-sern-backend): Meta SDK + Conversions API. Plan approved, NOT started.
	1. Why: web pixel only fires `Lead` on app-store badge press (intent, not installs or revenue). RN app has ZERO Meta instrumentation → UPSC App Promotion campaigns cannot run or be measured at all.
	2. Scope: **Android only**. iOS out of scope — ~20% of IAP purchases stay invisible to Meta. No ATT prompt, no refund events, no extra screens.
	3. BLOCKING prereq — Meta Business Manager (human steps, nothing else works until done):
		1. register app with Android package name (`applicationId` from `android/app/build.gradle`)
		2. link app → ad account, add as data source on EXISTING dataset `973157488384012` — do NOT create a second pixel
		3. collect `facebook_app_id` + `facebook_client_token`; generate system-user access token for CAPI
		4. Events Manager: rank AEM 8-event priority — Purchase top, CompleteRegistration second
		5. create 2 Custom Conversions filtered on `content_category` = upsc / sikkim_psc
	4. New env vars on prod backend: `META_CAPI_ACCESS_TOKEN`, `META_DATASET_ID`, `META_CAPI_ENABLED` (kill switch). Add to `.env.example` too.
	5. Requires a new **Play Store release** — native change (`strings.xml`, AndroidManifest meta-data + `AD_ID` permission). The CAPI half ships independently of the app release.
	6. Admin task (data, not code): only 12 of 1391 courses carry a UPSC/SPSC tag, and ~20% of actual IAP purchases sit on untagged courses. Tag them via `course_tag_selection` or the cohort split leaks. Tags: UPSC = id 2, SPSC = id 7. Untagged purchases will send `content_category: 'untagged'` + a warning log naming the course_id.
	7. Campaigns optimise for `CompleteRegistration` FIRST, not Purchase — only ~55 IAP purchases all-time, far below Meta's ~50/wk/ad-set learning threshold. Switch the goal to Purchase once volume clears.
	8. RISK — purchase value correctness: `sevices/iap.service.js:383` hardcodes currency "INR", the price comes from the courses table NOT the store, and `:890-891` applies a minor-unit heuristic (`price >= 100 ? price/100 : price`). Send the real store price (`priceAmountMicros`, already available at `iapService.ts:604`) through the verify payload instead. A wrong value is worse than no value — bidding acts on it.
	9. Dedupe: `event_id` = Google Play `transactionId`, identical on the client SDK event and the server CAPI event. Get this wrong and every purchase counts twice → ROAS inflated by exactly 2×, easy to miss.
	10. Known dead end for any future iOS work: iOS purchases never reach the main backend (`iapService.ts:224` skips server verification). iOS entitlements are granted by Apple ASSN into `razorpay-webhook-server/src/services/apple.subscription.service.js:646`. Start there, not in the main backend.
	11. Full plan: `/Users/stardust/.claude/plans/concurrent-dreaming-micali.md`
