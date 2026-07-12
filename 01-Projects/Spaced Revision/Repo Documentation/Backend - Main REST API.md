---
Date: 2026-07-12
tags:
  - spaced-revision
  - backend
  - documentation
Status: living
---

> Part of [[00 - Repo Documentation Overview]]

# Spaced Revision — Main REST API Backend (`spaced-revision-sern-backend`)

Deep technical reference for the primary backend service. This is the authoritative REST API for the Spaced Revision EdTech platform, serving the web frontend, the React Native app, and internal services.

---

## 1. Overview

**What it does.** A monolithic Express REST API covering the entire product surface: auth/OAuth, courses → subjects → topics → notes/cards/MCQs curriculum, spaced-repetition scheduling, ELO difficulty rating, payments (Razorpay + IAP + EMI), GST/invoicing, live classes, video (HLS), community/discussions/support/doubts, decks & sharing, test series, answer-writing, push notifications, WhatsApp, admin analytics, and WatermelonDB offline sync for the mobile app.

| Aspect | Value |
|---|---|
| **Language/runtime** | Node.js 18+, **CommonJS** (`"type": "commonjs"`, `require`/`module.exports`) |
| **Framework** | Express `^4.21.2` |
| **Datastores** | MySQL 8+ (`mysql2/promise`), Redis (`ioredis`), ClickHouse (`@clickhouse/client`) for OLAP |
| **Job queue** | BullMQ `^5.71` (Redis-backed) + `node-cron` |
| **Port** | `process.env.PORT || 8080` |
| **Package manager** | **pnpm** (`pnpm-lock.yaml`, `pnpm-workspace.yaml` present; `pnpm.overrides` block in `package.json`) — note the workspace-level `CLAUDE.md` mentions Yarn/Volta for the *frontend*, not this repo |
| **Entry point** | `server.js` (despite `package.json` `"main": "index.js"`, which is vestigial — there is no `index.js`) |

**How to run** (`package.json` scripts):
```bash
npm start        # nodemon server   (alias: npm run dev / npm run server)
npm test         # jest --verbose --onlyChanged --runInBand
npm run test:watch
```
`docker-compose up` brings up MySQL + backend together (per the repo's own `CLAUDE.md`). `prepare` installs Husky git hooks.

**Server bootstrap** (`server.js`): the module exports the Express `app` for Supertest and only calls `startServer()` when `require.main === module`. `startServer()`:
1. Establishes a MySQL connection (`connectToMySQL.getConnection()`) — fatal on failure (`process.exit(1)`).
2. `app.listen(PORT)`.
3. Conditionally starts BullMQ workers if `RUN_CRON === 'true'` (study-reminder, overtake, EMI reminder, mcq-added).
4. Starts the ELO cron unless `RUN_ELO_CRON === 'false'`.
5. Wires graceful shutdown (`SIGTERM`/`SIGINT`/`unhandledRejection`/`uncaughtException`) that closes workers → shared Redis → HTTP server, with a 10s force-exit.

---

## 2. Directory structure

```
spaced-revision-sern-backend/
├── server.js                # App composition: middleware, ~70 route mounts, /metrics, /health, error handler, bootstrap
├── passport.js              # Passport strategies (Google + Apple) — Facebook is a dep but NOT wired
├── package.json             # CommonJS; deps incl. bullmq, mysql2, ioredis, firebase-admin, razorpay, prom-client
├── jest.config.js           # Test config (coverage, markers)
├── routes/
│   ├── api/                 # 67 route files — one per feature (auth.js, course.js, mcq.js, payment.js, …)
│   │   ├── public/          #   public-token-gated read-only routes
│   │   └── mgm_team/        #   internal team-admin routes
│   └── docs/                # 25 *.openapi.js JSDoc blocks (Swagger/ReDoc source) + _components.openapi.js
├── controllers/             # 18 controllers — thin glue (course.controller.js, Payment.controller.js, iap.controller.js, …)
├── services/                # 10 modules — CORRECTLY spelled (accrual*, synchronization.service, activity*, istCalendar, …)
├── sevices/                 # 14 modules — MISSPELLED but LIVE (QueryExecuterFactory 86KB, iap.service, invoice.service, PushToUser.service, …)
├── database/                # 105 files + 13 subdirs — one SQL module per feature (schema-as-code + query helpers)
│   ├── mysqlConnector.js    #   THE pool. getConnection() + pool export
│   ├── migrations/          #   one-off migration scripts (unique indexes, column adds)
│   └── {app,coupons,flashcards,hls,team,test_series,webhook,fcm,discussion_forum,username,…}/
├── middleware/              # auth.js, admin.js, public.middleware.js, requireInternalSecret.js, deckAccess.js, createCache.js(stub)
├── validators/             # 7 express-validator schemas (address, deck, iap, live_classes, testSeries, video, discussionForum)
├── cron/                   # eloUpdateCron.js (node-cron) — the ONLY node-cron job
├── notifications/          # BullMQ queues/workers/schedulers (study-reminder, overtake, emiReminder, mcqAdded) + Bull Board
├── config/                 # mysqlConnector-adjacent infra: redisClient, redisNotificationClient, prometheus.client,
│                           #   loki.logger, clickhouse, s3Client, multer, posthogServer, swagger.config, ' appleAuthConfig'
├── utils/                  # eloRating, signedUrlGenerator.util, s3UploadHandler.util, topicCascade, parseBankStatement,
│                           #   generateUserName, appleAuthHelpers, ApiResponse, list/ (runListQuery builder), …
├── scripts/                # ops scripts (backfillEmiReminders, generateJwtForUser, issueCouponForCourse, testOvertake)
├── Test/                   # Jest suites mirroring source layout (routes/, services/, middleware/, helpers/, utils/, …)
├── docs/                   # Markdown source-of-truth (API_INVENTORY, DATABASE_SCHEMA, ARCHITECTURE, DOMAIN_GLOSSARY, …)
├── keys/                   # TLS certs (ca.crt/client.crt/client.key) for Hostinger Redis mTLS + signing keys
├── public/                 # self-hosted redoc.standalone.js
└── logs/, coverage/, .husky/
```

Two-directory count worth flagging: **`services/` (10 files, correctly spelled) and `sevices/` (14 files, misspelled) both exist and are both imported.** See §9.

---

## 3. Request lifecycle

The documented pattern is **Route → Controller → Service/Database module → MySQL pool**. In practice the repo is mixed: newer/list endpoints are cleanly layered through a controller; many older endpoints inline the SQL directly in the route handler.

### Example A — clean layered path (`GET /api/course/premium-list`)

1. **Mount** (`server.js:151`): `app.use("/api/course", courseRoute)`.
2. **Route** (`routes/api/course.js:11`) imports the controller and delegates:
   ```js
   const courseController = require("../../controllers/course.controller")
   // ...router.get("/premium-list", courseController.getPremiumCoursesList)
   ```
3. **Controller** (`controllers/course.controller.js`) is thin glue — it owns the SQL *shape* and whitelist, then calls the list helper:
   ```js
   async function getPremiumCoursesList(req, res, next) {
     try {
       const result = await runListQuery(req.query, PREMIUM_COURSES_LIST_CONFIG)
       res.json(result)
     } catch (error) { next(error) }
   }
   ```
   `PREMIUM_COURSES_LIST_CONFIG` declares `select`/`from`/`baseWhere`/`filterMap`/`sortMap`/`searchExprs` — a **whitelist** so only named columns can be filtered/sorted.
4. **Data layer** (`utils/list/runListQuery.js`) is pure (never touches `req`/`res`). It composes safe clauses via `utils/list/QueryBuilder.js` + `utils/list/mysqlAdapter.js`, gets a pooled connection, runs the query with `?` placeholders, and returns `{ data, meta }`.
5. **Pool** (`database/mysqlConnector.js`) hands back the connection; the helper releases it in a `finally`.

### Example B — inline route path (`POST /api/auth` — password login)

Fully self-contained in `routes/api/auth.js:149`:
1. `express-validator` `check()` array validates `email`/`user_name`/`password`; `validationResult(req)` → 400 on failure.
2. `connection = await getConnection()`.
3. `SELECT * FROM users WHERE email = ? AND is_deleted = 0` (parameterized), reject if deleted / no password.
4. `bcrypt.compare(password, user.password)`.
5. Side effects: `INSERT INTO user_login_logs …` and `UPDATE users SET last_active_at = NOW(), last_push_sent_at = NULL, last_inactivity_push_index = NULL` (this last-active reset appears on *every* login path — it re-arms the inactivity-push state machine).
6. Issue JWT: `jwt.sign({ user: { id } }, JWT_SECRET, { expiresIn: "365d" })` → `res.json({ token, is_admin })`.
7. `finally { if (connection) connection.release() }`.

**The universal conventions:** every handler `getConnection()` → `connection.execute(sql, params)` with `?` placeholders → `connection.release()` in `finally` → errors forwarded with `next(error)` to the global handler. There is exactly one pool (`database/mysqlConnector.js`); do not instantiate a second.

---

## 4. Authentication & authorization

### JWT issuance
Login endpoints (password, Google id-token, Android, Apple id-token, OAuth callbacks) all sign the same minimal payload:
```js
jwt.sign({ user: { id: user.id } }, process.env.JWT_SECRET, { expiresIn: "365d" })
```
- App/API tokens use a **365-day** expiry (very long-lived — mobile UX driven).
- `passport.serializeUser` (used only by the web OAuth redirect flow) signs with `{ expiresIn: "1d" }`.
- Token is returned in JSON (`{ token }`) for app clients, or as a redirect query param (`${WEBSITE_URL}/login?jwt=${token}`) for browser OAuth.

### Middleware (`middleware/`)

| File | Guard | Behavior |
|---|---|---|
| `auth.js` | **JWT only** | Reads header **`x-auth-token`**, `jwt.verify(token, JWT_SECRET)`, sets `req.user = decoded.user` (i.e. `{ id }`). 401 on missing/invalid. No DB hit. |
| `admin.js` | **JWT + admin** | Verifies JWT, then `SELECT is_admin FROM users WHERE id = ?`. 401 if not admin. Attaches `req.user.is_admin`. **Use alone for admin routes — do NOT chain with `auth`** (it already verifies the token). |
| `public.middleware.js` | **Public API token** | Reads `Authorization: Bearer <jwt>`, decodes the header `kid` (supports key rotation via `JWT_KEYS = { 'public-v1': JWT_PUBLIC_SECRET }`), verifies HS256 with `issuer: 'SpacedRevision Pvt. Ltd.'`, `audience: 'public-access'`, and asserts `payload.type === 'public'`. Attaches `req.publicSession`. |
| `requireInternalSecret.js` | **Service-to-service** | Compares header `x-internal-secret` against `INTERNAL_API_SECRET` using `crypto.timingSafeEqual` (length-checked first). Used by e.g. the EMI-reminder internal endpoint called by the webhook server. |
| `deckAccess.js` | **Per-deck RBAC** | `deckAccess(requiredRoles)` factory. Resolves the caller's role for `:deck_id` (owner via `topics.user_id`, else `deck_shares` with `status='accepted'`), attaches `req.deckAccess = { role, deck_id }`. Roles: `owner` (always passes) `> admin > read_write > read_only`. |
| `createCache.js` | — | **Empty stub** — a `({maxAge,key}) => (req,res,next) => {}` no-op; caching is done inline with `redisClient` instead (§6). |

`server.js` applies `auth` inline at mount time for a few routers, e.g. `app.use('/api/iap', auth, inAppPurchasesRoute)` and `app.use("/api", auth, require("./routes/api/rewardPoints.routes"))`.

### Passport strategies (`passport.js`)
The file opens with a ~120-line commented-out legacy block; the live code registers **two** strategies:

- **Google** (`passport-google-oauth20`): env-guarded at import (`GOOGLE_CLIENT_ID/SECRET/CALLBACK_URL` required or `process.exit(1)`). `verify()` checks `federated_credentials (provider, subject)`; links to an existing `users.email` row or inserts a new user with a generated `username` (`utils/generateUserName`), a v4 `uuid` (`UUID_TO_BIN`), `email_verified = 1`. Blocks `is_deleted` accounts. Resets `last_active_at`.
- **Apple** (`passport-apple`, `passReqToCallback`): verifies the identity token via `config/ appleAuthConfig` (note the leading space in the filename — see §9) and `utils/appleAuthHelpers.normalizeAppleUserIdentity` (handles Apple private-relay `isCustomEmail`). Same federated-credential link/create flow.

Passport runs **stateless** — `app.use(passport.initialize())` only; `passport.session()` is commented out, and OAuth callbacks use `passport.authenticate(..., { session: false }, cb)`.

Beyond Passport, `routes/api/auth.js` also has **token-based** OAuth endpoints for mobile:
- `POST /api/auth/google/idtoken` and `/google/android` — verify a Google **ID token** against **multiple audiences** (`GOOGLE_WEB_CLIENT_ID` / `GOOGLE_IOS_CLIENT_ID` / `GOOGLE_ANDROID_CLIENT_ID`) via `google-auth-library`'s `OAuth2Client.verifyIdToken`, then find-or-create the user and issue the app JWT.
- `POST /api/auth/apple/idtoken` — verifies the Apple identity token directly (no browser redirect).

`is_deleted` (soft-deleted accounts) is checked on *every* login path and returns a "contact support" message.

---

## 5. Database layer

### Connection pool (`database/mysqlConnector.js`)
Single shared `mysql2/promise` pool — the one true pool:
```js
const pool = mysql.createPool({
  host: MYSQL_HOST, user: MYSQL_USER, password: MYSQL_PASSWORD,
  database: MYSQL_DB_NAME, port: MYSQL_DB_PORT,
  waitForConnections: true, connectionLimit: 100, queueLimit: 0,
})
module.exports = { getConnection, pool }
```
- `connectionLimit` is **hardcoded to 100** — the `MYSQL_CONNECTION_LIMIT`/`MYSQL_QUEUE_LIMIT` env vars exist in `.env` but are **not read** by the pool.
- `getConnection()` optionally **wraps `execute()`/`query()`** to record Prometheus DB metrics (operation + table parsed from the SQL). Gated by `ENABLE_DB_METRICS !== 'false'`; `parseSQL()` sniffs the leading keyword and extracts the table name (handling backticks/schema-qualified names).
- Both `{ getConnection, pool }` are exported. Route/controller code typically uses `getConnection()` + `release()`; BullMQ workers use `pool.execute(...)` directly.

### `database/` modules — schema-as-code + query helpers
The `database/` directory (105 files) is **not migrations**. Each file is a per-feature SQL module that exports *both*:
1. **Idempotent schema functions** — `createCourseTable()` runs `CREATE TABLE IF NOT EXISTS courses (…)`, `addForeignKeyToCourses()`, etc. (`database/coursetable.js`).
2. **Query helpers** consumed by routes/controllers.

There is no ORM. Feature-scoped subdirs (`database/app/`, `database/flashcards/`, `database/hls/`, `database/webhook/`, `database/team/`, `database/coupons/`, `database/test_series/`, …) group related tables. Actual schema changes ship as one-off scripts in **`database/migrations/`** (e.g. `addRazorpayPaymentIdUnique.js`, `addBankStatementReferenceUnique.js`, `addVideoTopicId.js`, `runAllMappingMigrations.js`).

### Query conventions
- Always `?` placeholders; never string-interpolate user input. Whitelist table/column names when parameterized (see `runListQuery` filter/sort maps).
- MySQL 8 — CTEs and window functions are used freely.
- **Big query facade:** `sevices/QueryExecuterFactory.js` (~86 KB) is a static-method mega-class (`static async executeQuery(sql, params)` that self-releases the connection) providing hundreds of feature queries (discussions, analytics, push fan-out, etc.). `AnswerWritingScoresQueryExecutor.js` and `LiveClassQueryExecutor.js` follow the same pattern.
- **List builder:** `utils/list/` (`runListQuery` + `QueryBuilder` + `mysqlAdapter`) is the reusable whitelisted/filterable/sortable/paginated read path returning `{ data, meta }`.

### ClickHouse (OLAP)
- `config/clickhouse.js` exports a `@clickhouse/client` instance (`CLICKHOUSE_HOST/USER/PASSWORD/DB`).
- Activity logs live in ClickHouse: `services/activityLog.service.js` (cursor-paginated insert/query) + `services/activityAnalytics.service.js`. `config/clickhouseActivityLog.js` is a **deprecated legacy compatibility shim** re-exporting those services and translating cursor pagination back to page-based.
- Exposed via `app.use("/api/clickhouse", clickHouseRoute)`.

---

## 6. Redis usage

There are **two independent Redis clients** for two distinct purposes:

### 1. App cache — `config/redisClient.js`
- An `ioredis` client wrapped in `trackedRedis` (records Prometheus cache-hit/miss/latency metrics and infers a `cache_type` from the key prefix).
- Tuned for a **request-path cache, not a queue**: `maxRetriesPerRequest: 3`, `enableOfflineQueue: false`, a bounded `retryStrategy` (gives up after 10 attempts), `connectTimeout: 5000`. Errors are swallowed so the app keeps serving if Redis is down.
- Used inline in **13 route files** as a look-aside cache: `get` / `setex(key, ttl, JSON)` / `del`. Examples:
  ```js
  const cached = await redisClient.get(`user_collections:${user_id}`)   // collections.js
  await redisClient.setex(cacheKey, 1800, JSON.stringify(rows))          // course.js (30-min TTL)
  await redisClient.del("public_courses")                               // invalidation on write (admin.js, course.js)
  ```
  Key namespaces seen: `user_courses:`, `user_collections:`, `user_decks:`, `public_courses`, `course_progress:`, `notescontent:`, `app-version:`, `pdf`.

### 2. BullMQ / locks — `config/redisNotificationClient.js`
- Exports a single `sharedRedis` ioredis client with **`maxRetriesPerRequest: null`** (mandatory for BullMQ).
- Dual-mode connection: if `HOSTINGER_REDIS_HOST/PORT` are set it connects over **TLS with mTLS client certs** read from `keys/{ca,client}.{crt,key}`; otherwise plain `REDIS_HOST:REDIS_PORT` (default `127.0.0.1:6379`).
- Shared by every queue, worker, and scheduler, and also for **daily/cooldown locks** using atomic `SET key val NX PX ttl` (e.g. study-reminder's `notif_locked:<uid>:<istDate>`, mcq-added's per-course fire-date lock, overtake cooldowns).

**Sessions:** the JWT is fully stateless — there is no server-side session store. `cookie-session` is a dependency but `passport.session()` is disabled.

---

## 7. Background jobs & cron

Two orthogonal gates control background work (so only one process instance runs them): **`RUN_ELO_CRON`** for the cron and **`RUN_CRON`** for the BullMQ workers.

### node-cron — ELO update (`cron/eloUpdateCron.js`)
- Scheduled `'0 3 * * *'` (daily 3:00 AM) in `Asia/Kolkata`, started unless `RUN_ELO_CRON === 'false'`.
- `runEloUpdateJob()` runs two resumable backfills in sequence, each in batches of 5000: **MCQ ELO** (`database/topicEloTable.backfillEloFromLogs` — `mcq_attribute_logs` → `mcq.level` + `topic_elo`) then **Card ELO** (`database/cardTopicEloTable.backfillCardEloFromLogs` — `attribute_logs` → `cards.level` + `card_topic_elo`). Only processes logs where `user_level IS NULL`.
- An `isRunning` flag prevents overlap; errors are swallowed so cron keeps scheduling. Status/manual-trigger helpers are exposed and surfaced via `POST/GET /api/elo-cron` (admin). The single ELO math source of truth is `utils/eloRating.js`.

### BullMQ workers (`notifications/`, started when `RUN_CRON === 'true'`)
All share `sharedRedis`. Four queues:

1. **`study-reminder`** (`queue.js`, `worker.js`, `studyReminderScheduler.js`) — a **self-chaining JIT** reminder:
   - `scheduleFromStudyAction(userId)` fires on the user's first study action of an IST day, taking a per-user daily Redis lock (`notif_locked:<uid>:<date>`, TTL = ms until next IST midnight), then enqueues a Day-1 job delayed `STUDY_REMINDER_DELAY_MS` (24 h).
   - The worker recomputes due counts (`services/revisionDueCounts`); **stops if `totalDue === 0`**, otherwise builds an FCM payload (rotating title by `day`, deep-link route from `services/lastStudiedTopic`), sends via `sevices/PushToUser.service.sendPushToUser`, stamps `users.last_push_sent_at`, and captures a PostHog `notification_sent` event.
   - It then **chains** the next day via `setImmediate(scheduleFromWorker(...))` (deferred because BullMQ can't remove a locked/active job), stopping at `MAX_STUDY_REMINDER_MISSED_DAYS` (15). Custom jobId `reminder-push-<userId>` (no `:` allowed).

2. **`overtake`** (`overtakeQueue.js`, `overtakeWorker.js`) — leaderboard "someone overtook you" push. Stores weekly top-N rank snapshots in Redis (`lb:snap:<course>:<week>`, TTL ~8 days), diffs against the live ranking (`controllers/analytics.controller.computeCourseRanking`), and enforces a per-recipient/per-course cooldown (`overtake:cd:<user>:<course>`, default 6 h). `TOP_N` default 20.

3. **`emi-reminder`** (`emiReminderQueue.js`, `emiReminderWorker.js`, `emiReminderScheduler.js`) — self-chaining EMI due-date reminders. One delayed job per `(user, course)` re-reads `community.expires_at` on each fire and reschedules to the next "mark": `BEFORE_DAYS = [7,3,1,0]` then `AFTER_DAYS = [3,7]` (overdue). Bounded — the chain stops after the last mark. Re-armed for the next installment by the **razorpay-webhook-server** hitting the internal endpoint `POST /api/emi-reminder` (guarded by `requireInternalSecret`). `EMI_REMINDER_UNIT_MS` can compress "days" to seconds for testing.

4. **`mcq-added-notification`** (`mcqAddedQueue.js`, `mcqAddedWorker.js`, `mcqAddedScheduler.js`) — daily "new MCQs added to your enrolled course" push. An educator adding an MCQ fire-and-forget schedules one job **per course per day** (Redis NX lock keyed by fire date), delivered at `MCQ_NOTIF_HOUR:MINUTE` IST (default 17:30). At fire time it counts trailing-24 h new MCQs and fans out to actively-enrolled students (`services/synchronization.service.communitySubscriptionValidSql`).

### Bull Board dashboard
`routes/api/queuesBoard.js` mounts `@bull-board/express` at **`/admin/queues`** for the `study-reminder` and `emi-reminder` queues. In `server.js` it's gated by an optional `BULL_BOARD_ACCESS_KEY` (via `?key=`, `x-bull-board-key`, or `Authorization: Bearer`).

---

## 8. Key integrations

| Integration | Where | Notes |
|---|---|---|
| **Razorpay** | `controllers/Payment.controller.js`, `controllers/checkout.controller.js`, `routes/api/payment.js`; `RAZORPAY_KEY_ID/SECRET` | Order creation + verification. **Webhooks are handled by the separate `razorpay-webhook-server`**, not here. Accrual math is shared/copy-synced (`services/accrualPolicy.js`, `accrualLedger.js`, `accrualValidation.js`). |
| **In-App Purchase** | `sevices/iap.service.js` (~40 KB), `controllers/iap.controller.js`, `validators/iap.validator.js`, `routes/api/iap.routes.js` | Apple/Google receipt validation. Mounted `app.use('/api/iap', auth, …)`. |
| **AWS S3 / CloudFront** | `config/s3Client.js`, `utils/signedUrlGenerator.util.js`, `utils/s3UploadHandler.util.js` | `generateS3PutSignedUrl` / `generateS3GetSignedUrl` (presigned S3) and `generateCloudFrontSignedUrl` / `generateDownloadUrl` (CDN, `@aws-sdk/cloudfront-signer`). Buckets `AWS_S3_BUCKET_NAME_PDFS` / `_RAW_VIDEOS`. |
| **File uploads** | `express-fileupload` (global, **skips `/api/video`** so multer can parse those), `config/multer.js` | `imageUploader` = memoryStorage, 10 MB, jpg/png/jpeg only; `pdfUploader` = diskStorage under `uploads/pdfs`, `.pdf` only. Ad-hoc `/api/uploadImage` & `/api/uploadCourseImage` write to local disk. |
| **Firebase FCM** | `utils/fireBase.js`, `sevices/PushToUser.service.js`, `sevices/FcmToken.service.js`, `sevices/NotificationRouter.service.js` | Admin SDK from `FIREBASE_ADMIN_KEY_PATH`. **In dev (no key) it exports a mock** `messaging()` so push calls no-op. Multicast via `sendEachForMulticast`. |
| **WhatsApp** | `routes/api/whatsapp.js`; `WHATSAPP_ACCESS_TOKEN`, `_PHONE_NUMBER_ID`, `_BUSINESS_ACCOUNT_ID` | Cloud API messaging + SSE event stream. |
| **VideoSDK** (live classes) | `controllers/live_classes.controller.js`, `sevices/LiveClassQueryExecutor.js`; `VIDEOSDK_API_KEY/SECRET_KEY/API_ENDPOINT` | Live-class rooms. |
| **PostHog (server)** | `config/posthogServer.js` (`POSTHOG_PROJECT_API_KEY`, host default `posthog.spacedrevision.in`) | Server-side `capture()`; disabled/no-op if key unset. |
| **Prometheus** | `config/prometheus.client.js` | Custom registry, prefix `SpacedRevision_Node_Server_`. `httpMetricsMiddleware` (early in the chain) + DB-query + Redis-op metrics. **`GET /metrics`** protected by `METRICS_ACCESS_KEY` (warns loudly if unset). |
| **Loki + Winston** | `config/loki.logger.js` | **Dev = console only (never ships to Loki)**; prod = `winston-loki` to `LOKI_HOST`. Helpers `logger.logError`, `logRequestError`, `handleAndRespond`. Request-logging middleware in `server.js` skips `/health` & `/metrics`. `morgan('dev')` also enabled. |
| **Swagger / ReDoc** | `config/swagger.config.js`, `routes/docs/*.openapi.js` | `swagger-jsdoc` scans `./routes/**/*.js` + `server.js`. Served at **`/api-docs`** (Swagger UI), **`/redoc`** (self-hosted `public/redoc.standalone.js`, with a relaxed per-route CSP), raw spec at **`/api-docs.json`**. Security scheme `authToken` = header `x-auth-token`. |
| **Health** | `GET /health` | `{ status, timestamp, uptime }`, always 200. |

---

## 9. Notable patterns, conventions & gotchas

- **Misspelled `sevices/` is real and load-bearing.** There are two sibling directories: `services/` (correctly spelled, 10 files — sync, accrual, activity analytics, IST calendar) and **`sevices/`** (misspelled, 14 files — `QueryExecuterFactory.js`, `iap.service.js`, `invoice.service.js`, `PushToUser.service.js`, `NotificationRouter.service.js`, `CouponExecutor.js`, `RewardPointsService.js`, `address.service.js`, `deck.service.js`, `gstr1.service.js`, `FcmToken.service.js`, …). Both are imported throughout (e.g. `require('../sevices/PushToUser.service')` in the workers). Do not "fix" the spelling without updating every import.
- **`config/ appleAuthConfig.js` has a literal leading space in its filename.** Imports must match exactly: `require("./config/ appleAuthConfig")` / `require("../../config/ appleAuthConfig")`.
- **`server.js` is the entry point, not `index.js`.** `package.json` `"main": "index.js"` is vestigial; `nodemon server` runs `server.js`. It exports `app` so Supertest imports it via `require('../../server.js')` (there is no `app.js`).
- **Facebook OAuth is a dependency but not wired** (`passport-facebook` is in `package.json`; no strategy is registered).
- **`connectionLimit` is hardcoded to 100** — `MYSQL_CONNECTION_LIMIT`/`MYSQL_QUEUE_LIMIT` env vars are defined but ignored by the pool.
- **`payment.status` is unreliable** — it often stays `'created'` after a successful charge. Use `payment_id IS NOT NULL` as the success predicate. Two payment tables coexist: `payment` (Razorpay) and `direct_payments` (manual/cash); many admin reports UNION both.
- **Synced tables must SOFT-delete.** WatermelonDB sync (`services/synchronization.service.js` → `getTableChanges`) reports deletions via `deleted_at > lastPulledAt`. Hard-`DELETE` on a synced table (`topics`, `subjects`, `cards`, `mcq`, `notesheading`, `notescontent`, …) never reaches the app. Delete endpoints must `UPDATE … SET deleted_at = NOW()`, and every read must add `deleted_at IS NULL`. Topic soft-delete cascade lives in `utils/topicCascade.js` (single source; called by topic/subject deletes). The sync push endpoint is why the global JSON body limit is **10 MB** (`JSON_BODY_LIMIT`).
- **DATE columns + IST off-by-one.** The pool doesn't set `dateStrings`, so mysql2 returns `DATE` as a JS `Date` at driver-local midnight; formatting with `getUTC*()` shifts the day backward on IST hosts. Cast in SQL: `DATE_FORMAT(col, '%Y-%m-%d') AS col` (pattern in `routes/api/emiValidator.js`).
- **Cloned courses share content via `*_mappings` tables; videos have no `video_mappings`.** Any query listing course/topic videos must UNION mapped source topics (via `topic_mappings`) or it under-reports on cloned courses (see the video-mapping regression note in the repo `CLAUDE.md`).
- **Validators come in two flavors.** Older routes use inline `express-validator` `check([...])` arrays + `validationResult`; newer feature routes use `validators/*.js` `checkSchema({...})` objects (e.g. `validators/video.validator.js`, `address.validator.js`, `iap.validator.js`).
- **Global error handler** (`server.js`) forwards `next(error)` to a single handler that maps multipart/busboy parse errors to **400**, otherwise `err.status || 500`, and includes the stack only when `NODE_ENV !== 'production'`.
- **404 handler** returns `{ success:false, message:"Route not found", path }`.
- **`RUN_CRON` / `RUN_ELO_CRON` gating** exists so only one instance runs background jobs — critical when scaling horizontally.
- **Docs-first discipline (enforced by repo `CLAUDE.md`).** Any route change must update both `docs/API_INVENTORY.md` and the matching `routes/docs/<feature>.openapi.js`; schema changes update `docs/DATABASE_SCHEMA.md`; new cron/queue/service updates `docs/ARCHITECTURE.md`. A feature "isn't done" until its `Test/` suite exists and passes (`npx jest Test/<path>.test.js`, since `npm test` runs `--onlyChanged`).
- **Secret-leak footnote:** `mysqlConnector.js` deliberately never logs `MYSQL_PASSWORD` — a prior leak into a committed `backfill.log` is called out in a comment (the `backfill.log` file is still present in the repo root).

---

## 10. Environment variables

Keys referenced in code/config (values redacted). Grouped by concern:

**Core / server**
`PORT`, `NODE_ENV` (unset/`development`/`dev` = dev mode), `SERVER_NAME`, `BACKEND_URL`, `FRONTEND_URL`, `WEBSITE_URL`, `JSON_BODY_LIMIT`, `RUN_CRON`, `RUN_ELO_CRON`, `ENABLE_DB_METRICS`

**Auth / JWT**
`JWT_SECRET`, `JWT_PUBLIC_SECRET`, `INTERNAL_API_SECRET`, `INTERNAL_SERVICE_KEY`
`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`, `GOOGLE_WEB_CLIENT_ID`, `GOOGLE_IOS_CLIENT_ID`, `GOOGLE_ANDROID_CLIENT_ID`
`APPLE_OAUTH_CLIENT_ID`, `APPLE_OAUTH_CLIENT_SECRET`, `APPLE_OAUTH_KEY_ID`, `APPLE_OAUTH_TEAM_ID`, `APPLE_OAUTH_CALLBACK_URL`

**MySQL**
`MYSQL_HOST`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_DB_NAME`, `MYSQL_DB_PORT` (`MYSQL_CONNECTION_LIMIT`, `MYSQL_QUEUE_LIMIT` defined but unused)

**Redis**
`REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `HOSTINGER_REDIS_HOST`, `HOSTINGER_REDIS_PORT` (TLS via `keys/{ca,client}.{crt,key}`)

**ClickHouse**
`CLICKHOUSE_HOST`, `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD`, `CLICKHOUSE_DB`

**AWS S3 / CloudFront**
`AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_BUCKET_NAME_PDFS`, `AWS_S3_BUCKET_NAME_RAW_VIDEOS`, `CLOUDFRONT_DOMAIN`, `CLOUDFRONT_KEY_PAIR_ID`, `CLOUDFRONT_PRIVATE_KEY`

**Payments / GST**
`RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `WEBHOOKS_API`, `WEBHOOKS_API_KEY`

**Firebase / Push / notifications**
`FIREBASE_ADMIN_KEY_PATH`, `FIREBASE_PROJECT_ID`
`STUDY_REMINDER_DELAY_MS`, `MAX_STUDY_REMINDER_MISSED_DAYS`, `OVERTAKE_DEBOUNCE_MS`, `OVERTAKE_TOP_N`, `OVERTAKE_COOLDOWN_MS`, `OVERTAKE_SNAPSHOT_TTL_SEC`
`EMI_REMINDER_BEFORE_DAYS`, `EMI_REMINDER_AFTER_DAYS`, `EMI_REMINDER_UNIT_MS`
`MCQ_NOTIF_HOUR`, `MCQ_NOTIF_MINUTE`, `MCQ_NOTIF_TEST_DELAY_MS`

**Monitoring / analytics**
`METRICS_ACCESS_KEY`, `BULL_BOARD_ACCESS_KEY`, `LOKI_HOST`, `LOKI_JOB`, `LOKI_AUTH_HEADER`, `LOG_LEVEL`, `POSTHOG_PROJECT_API_KEY`, `POSTHOG_HOST`

**Integrations / misc**
`WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_BUSINESS_ACCOUNT_ID`, `VIDEOSDK_API_KEY`, `VIDEOSDK_SECRET_KEY`, `VIDEOSDK_API_ENDPOINT`, `ANKI_EXTRACT_URL`, `EMAIL_USERNAME`, `EMAIL_PASSWORD`, `EMAIL_SENDER`, `SESSION_TTL_SECONDS`

---

*Reference generated from source inspection of `server.js`, `passport.js`, `package.json`, and representative files across `routes/`, `controllers/`, `services/`, `sevices/`, `database/`, `middleware/`, `cron/`, `notifications/`, `validators/`, `config/`, and `utils/`. For per-route detail see `docs/API_INVENTORY.md` + `routes/docs/*.openapi.js`; for schema detail see `docs/DATABASE_SCHEMA.md`.*
