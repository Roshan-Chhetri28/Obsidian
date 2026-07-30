---
Date: 2026-07-30
tags:
  - Task
  - Plan
Status:
  - Not Started
  - Important
---

# Meta Ads Attribution — Android (SDK + Conversions API)

## Context

Meta ads currently fire one `Lead` pixel event when someone presses an app-store badge on the website (`spaced-revision-sern-frontend/index.html:157`, helpers in `src/utils/metaPixel.js:9`). That is the last thing Meta ever learns about that person. They install, register, study, maybe pay — and none of it exists as far as Meta is concerned, because the RN app has **zero Meta instrumentation** (verified: no fbsdk in `package.json`, `node_modules`, Podfile, `Info.plist`, gradle, or `AndroidManifest.xml`).

Three consequences:

1. **App Promotion campaigns cannot run at all** — Meta requires a registered app with an SDK.
2. **`Lead` measures the wrong thing** — a badge press is intent, not an install.
3. **No ad can be tied to a student.** Every number stops at the badge.

Goal: make Android installs and **genuine study engagement** visible to Meta, attributed to the ad that produced them.

---

## The central decision: optimise on study, not purchase

Meta's algorithm needs roughly **50 conversions per ad set per week** to exit the learning phase. Measured from PostHog (Android, 28 days, test users excluded):

| Event | Events | People |
|---|---|---|
| `flashcard_answered` | 65,572 | 223 |
| `flashcard_session_started` | 12,353 | 277 |
| `user_registered` | 250 | 249 |
| `purchase_completed` | **8** | 6 |

Purchase is ~2/week. A Purchase-optimised ad set would never leave learning — not slowly, never. Registration is ~62/week but weak: many register and never open a card.

**Raw study volume is a trap.** 3,088 sessions/week come from only ~69 unique people. Meta counts *events*, not people, so optimising on a repeatable event teaches it to find people who study a lot — i.e. your existing retained users. You would spend budget re-acquiring people you already have.

So the event must fire **once per person, ever**. First-time studiers per week (Android): 22, 31, 95, 58, 46, 27, 32 — typically **30–46**. Still under 50, but ad spend adds new users on top, which is the direction you want it to grow.

## Decisions taken

| Decision | Choice |
|---|---|
| Platform | **Android only.** iOS entirely out of scope. |
| Approach | Meta SDK (`react-native-fbsdk-next`) + Conversions API. No MMP. |
| **Primary conversion** | **`Activated` — 10 flashcards answered, once per user** |
| Fallback goal | `CompleteRegistration`, if activation volume proves thin per ad set |
| Purchase | Still sent (client + server), **reporting only** |
| Activation gate | **`users.meta_activated_at` column** — survives reinstalls |
| Cohort label | **UPSC only, reporting only** — never an optimisation filter |
| Web cohort source | Explicit `?cohort=upsc` on UPSC ad URLs |
| App cohort source | Server-derived from the course studied → onboarding pick → `untagged` |
| ATT / extra screens | None |
| Refunds, lookalike seeds, web `Purchase`, SPSC labelling | **Out of scope** |

### Why the cohort label is reporting-only

UPSC gets its own campaign, and Ads Manager already splits every number by campaign. The label does not tell you which ad worked — the campaign does. It tells you what the *user turned out to be*, which differs when a UPSC ad brings someone who then studies SPSC content.

Critically: filtering the optimisation event down to the UPSC slice would cut 30–46 activations/week further below Meta's threshold. **Narrowing the optimisation signal would actively hurt delivery.** Campaigns optimise on unfiltered `Activated`.

### Known limitations (accepted)

- **iOS invisible to Meta** (~20% of IAP purchases). iOS purchases never reach the main backend — `iapService.ts:224` skips server verification; entitlements come from Apple ASSN into `razorpay-webhook-server/src/services/apple.subscription.service.js:646`. Future iOS work starts there.
- Only 12 of 1391 courses carry a cohort tag; ~16–20% of purchases are on untagged courses. Admin tagging pass needed.
- Activation is gated on the **app sync path only**. Someone who studies on the web first (`routes/api/card.js`) will not fire it — deliberate, since the campaign measures app acquisition.

---

## Confirmed Meta-side values

| Item | Value |
|---|---|
| Meta app | **SpacedRevision ads** (clean — not one of the WhatsApp API apps) |
| **App ID** | `1348293100400435` |
| Business portfolio | `studytip_sciencehacks` — ID `1901160230286930`, Verified |
| Ad account | `488510547291559` |
| Web pixel (separate data source) | `973157488384012` |
| Package name | `com.spacedrevision` (`android/app/build.gradle:94`) |
| Main activity | `com.spacedrevision.MainActivity` |
| Play app id / dev id | app `4975969594176780494`, developer `5081034845630465015` |

**Key hashes** (Meta → app → Android platform):

| Source | Hash |
|---|---|
| Debug keystore (`android/app/debug.keystore`, alias `androiddebugkey`) | `TgbDzI280dxqzC7l+1cdLSRfSRw=` |
| **Play App Signing key** (SHA-1 `C4:E5:…:AC:E2`) — **the production one** | `xOXMRk+mZcGiQ8YT1DXiUd5rrOI=` |
| Upload key (`spaced_signed_keys.jks`, alias `release`) | *pending* |

Convert a Play Console SHA-1 with:
```bash
echo "AB:CD:…" | tr -d ':\n' | xxd -r -p | openssl base64
```

The upload keystore is not in the repo — it exists only as CI secrets (`ANDROID_KEYSTORE_BASE64` in `.github/workflows/android-release.yml:82`; Jenkins `credentials('android_keystore_file')`). Both fingerprints are readable from Play Console → Protected with Play → App signing (`…/app/<appId>/keymanagement`), so the keystore file is never needed.

---

## Step 1 — Meta Business setup (human, no code)

Blocks everything.

1. Create/reuse a Meta app, **Android platform only**. Package `com.spacedrevision`, activity `com.spacedrevision.MainActivity`. Add release **and** upload key hashes — the SDK is untrusted without them.
2. Record **App ID** and **Client Token** (Settings → Advanced).
3. **App events live on the App data source, not inside the web pixel.** In Events Manager the app and the pixel `973157488384012` are *siblings* in the Datasets list, not nested. App events key off the **App ID**; web pixel events stay in their own bucket. The two are joined at the ad-account / business level, not in one dataset. (Earlier drafts of this spec had this wrong.)
4. Business Settings → assign the ad account to **both** the app and the web dataset (Manage), else App Promotion campaigns can't select the app. *Already done — ad account `488510547291559` is attached via the app's "Manage app ads" use case.*
5. System user `capi-server` → assets: dataset + app (Manage) → non-expiring token with `ads_management` + `business_management`. This is `META_CAPI_ACCESS_TOKEN`.
6. Custom Conversion `Activated — UPSC` (filter `content_category = upsc`) **for reporting only**. Do not point a campaign at it.

**Not needed:** registering event parameters (Meta accepts arbitrary ones), pre-declaring the custom event name, domain verification.

**AEM correction:** Aggregated Event Measurement is an iOS/ATT mechanism. Android app events are **not** subject to the 8-event limit. The dataset's event-priority ranking still matters for the **website** pixel (iPhone web visitors) but is not a prerequisite for this work.

**Sequencing gotcha:** a custom event is only selectable as a campaign optimisation goal **after Meta has received at least one**. Ship → fire a real `Activated` → confirm in Events Manager → *then* create the campaign.

⚠️ Do not reuse the Meta app behind `WHATSAPP_ACCESS_TOKEN` / `META_APP_SECRET` in the webhook repo without checking — a WhatsApp Business app has different review and permission state.

---

## Step 2 — RN app: Android native config

`pnpm add react-native-fbsdk-next`. Autolinking handles registration.

⚠️ Check `peerDependencies` against RN 0.79.5 with `newArchEnabled=false` before merging. Pin the last old-bridge major if needed. Do **not** run `pod install`; leave a Podfile comment that fbsdk is intentionally unconfigured on iOS.

| File | Change |
|---|---|
| `res/values/strings.xml` | `facebook_app_id`, `facebook_client_token` (public values shipped in the APK — safe to commit, same as the existing `GOOGLE_WEB_CLIENT_ID`) |
| `AndroidManifest.xml` | `com.facebook.sdk.ApplicationId`, `ClientToken`, `AutoInitEnabled=true`, `AutoLogAppEventsEnabled=true`, `AdvertiserIDCollectionEnabled=true`; plus `com.google.android.gms.permission.AD_ID` |
| `MainApplication.kt` | one line after `SoLoader.init(...)`: `AppEventsLogger.activateApp(this)` |
| `proguard-rules.pro` | **critical** — see below |

`AutoLogAppEventsEnabled=true` produces the automatic **install** and **activate** events — the entire top of the funnel. Never disable.

**ProGuard — the highest-risk item in this plan:**
```
-keep class com.facebook.** { *; }
-keepclassmembers class com.facebook.** { *; }
-dontwarn com.facebook.**
```
`enableProguardInReleaseBuilds = true`, so without these the SDK is stripped **in release only** — debug works, production silently sends nothing. The existing `-keep class com.facebook.react.**` rule covers React Native, not the Facebook SDK. Different packages, same prefix.

**Play Console:** the `AD_ID` permission requires updating the Data safety declaration (Advertising ID → collected, Advertising/Marketing + Analytics) and the Advertising ID declaration in App Content, **in the same submission**, or the release is rejected.

---

## Step 3 — RN app: event layer

New `src/analytics/meta.ts`, sibling of `src/analytics/posthog.ts`. Tests in `src/analytics/__test__/meta.test.ts`. Shape mirrors `src/features/study/analytics.ts` — a typed closed set of names plus thin named functions, not a generic wrapper.

All calls hard-gated on `Platform.OS === 'android'` and individually try/caught, so iOS is a silent no-op.

### Identity

On login/registration: `setUserID(userId)` **and** `setUserData({ email, phone })` — the SDK hashes locally. Use the **same normalisation as the server**: `91` prefix on bare 10-digit numbers, skip masked Apple addresses.

This matters because `Activated` and `CompleteRegistration` are **client-only** — no server twin for registration, and for activation the client copy is the one carrying device identity. If GAID is zeroed (ads personalisation off) they would be orphaned, and orphaned conversions do not train the algorithm. Client-side hashed PII gives them a second matching path.

Call `clearUserData()` + `setUserID(null)` in `signOut` beside `posthog.capture('user_logout')`, so a shared device doesn't attribute the next signup to the previous person.

### `Activated` — the primary event

Server owns the gate; the client fires the SDK event when told to (Step 4b).

```
event_name:       'Activated'        (custom app event)
event_id:         'activated_<userId>'
content_category: <cohort, supplied by the server>
```

`event_id` is deterministic and once-per-user, so a retry or duplicated response can never double-count.

### `CompleteRegistration`

Two call sites, both already gated on an authoritative new-user signal — piggyback on the existing PostHog gate so the two systems can never disagree about who is new:

| Site | Add after the existing `posthog.capture('user_registered', …)` |
|---|---|
| `src/features/auth/store/authSlice.ts:30` (email) | `trackMeta('CompleteRegistration', { registration_method: 'email' })` |
| `src/features/auth/hooks/useAuth.ts:91` (OAuth, inside `if (isNewUser)`) | same, with `registration_method: authMethod` |

Fallback optimisation goal only — not the primary.

### `Purchase` — reporting only

Call site `iapService.ts:284`, **Android branch only** (leave the iOS branch at `:238` untouched).

At `:284` there is no price in scope. Add a module-level `pendingPurchasePrices` Map mirroring the existing `pendingPurchaseCourseIds` Map (same pattern, same lifecycle), populate it in `requestProductPurchase` where the store price is already known, read it at `:284`. `value = priceAmountMicros / 1_000_000`, currency from the store. If absent (pending purchase recovered by `checkPendingPurchases:327` after a restart), re-call `getProductDetails`; if that fails too, **skip the event and warn** — a wrong value is worse than a missing one.

`event_id: purchase.transactionId` — the dedupe key shared with the server.

**Cohort on the client copy.** Both copies share an `event_id`, Meta keeps one, so if only the server copy carried `content_category` the label would be non-deterministic. The `/iap/verify-purchase/android` **response** therefore returns the resolved cohort and the client attaches it — no extra round trip, since `:284` already sits inside the `if (response.data.success)` branch. Both copies then carry the same server-derived value.

### Deleted from the earlier design

The client no longer computes cohort. A hardcoded `UPSC_COURSE_IDS` map was going to drift from the DB; the server now resolves it and hands it back — in the sync response for `Activated`, in the verify-purchase response for `Purchase`. That map and its drift caveat are gone.

---

## Step 4 — Backend

### 4a. Migration — one column

```sql
ALTER TABLE users ADD COLUMN meta_activated_at DATETIME NULL;
```

**Append it; do not position it.** On MySQL 8 an `ADD COLUMN … NULL` at the end of the table is an INSTANT operation — no rebuild, no lock. Contrast `database/migrations/addAlternateEmailVerified.js`, which uses `MODIFY … AFTER` for column grouping and whose own header warns that is a full table rebuild.

New script `database/migrations/addMetaActivatedAt.js`, same idempotent shape minus the repositioning: check `INFORMATION_SCHEMA.COLUMNS`, `ADD` if absent, release in `finally`, `pool.end()`.

### 4b. Activation detection — the sync path

`controllers/sqlite.controller.js` → `syncPush` (`:11`, `user_id = req.user.id`). Card progress is written as `INSERT INTO attribute` at `:146`; the transaction commits at `:395`; the response is `{ success, timestamp }` at `:401`; the connection releases at `:413`.

**Why not an AsyncStorage flag on the device:** it resets on reinstall, and reinstalls are common in an install-ad funnel. That would inflate the primary optimisation event — the one number you least want inflated.

**Why not "fire when count == 10":** WatermelonDB sync writes in **batches**, so a user's count jumps 0 → 25 in one sync and never equals 10. The event would silently never fire.

The mechanism:

1. **Before the commit at `:395`**, on the same connection: if `meta_activated_at IS NULL`, count the user's cards (`SELECT COUNT(*) FROM attribute WHERE user_id = ?` — covered by `idx_user_id`). If `>= 10`:
   ```sql
   UPDATE users SET meta_activated_at = NOW()
   WHERE id = ? AND meta_activated_at IS NULL
   ```
   `affectedRows` tells you whether this sync was the one that crossed. The `AND meta_activated_at IS NULL` clause is what carries the correctness — it makes the write idempotent under concurrent syncs and removes any dependence on hitting exactly 10.
2. **After the commit**, extend the `:401` response with `meta_activated: true` and the resolved cohort — only on the transition, never afterwards.
3. **After the commit**, also fire CAPI with the same `event_id: 'activated_<userId>'`, fire-and-forget.

### 4c. Cohort resolver

New `services/metaCohort.js`, two exports:

- `resolveCohortForCourse(connection, courseId)` — `course_tag_selection` → `course_tag` where tag = `'UPSC'` → `'upsc'`, else `'untagged'`
- `resolveCohortForUser(connection, userId)` — primary: `attribute` → `cards.course_id` → `course_tag_selection` → `course_tag`. Fallback: `user_tag_preferences` tag_id 2. Else `'untagged'`.

Deriving from the course actually studied is behavioural rather than self-reported, and it is the only version that can tell you your UPSC ads are misfiring. The onboarding fallback exists because OAuth users skip the Preferences flow entirely.

**Caveat:** cloned courses share cards via `card_mappings`, so a card can belong to a master course while the user studies a clone. If the tag sits on the clone, the join misses — it degrades to `untagged`, never to a wrong answer. That is why the onboarding fallback stays.

**Both resolvers must never throw into their callers.** `resolveCohortForUser` runs inside the activation transaction; an unhandled throw would roll back a user's study sync. Try/catch internally, return `'untagged'`, log a warning. **Losing a reporting label is fine; losing study progress is not.**

Untagged emits `console.warn('[meta-capi] course %s has no UPSC tag', courseId)`. That warning is the worklist — grep Loki weekly.

### 4d. CAPI module

New `services/metaCapi.js` (correctly-spelled `services/`, matching `services/accrualPolicy.js`). CommonJS.

Exports: `normalizeEmail` (lowercase/trim; null for `@privaterelay.appleid.com` and `@sprv.com` — inline the constants with a comment pointing at `utils/appleAuthHelpers.js` as canonical), `normalizePhone` (digits only; 10 digits → `91` + digits, covering 98.9% of rows; 12 starting `91` as-is; else null), `normalizeName` (first token → `fn` only; `users.name` is a single column and a wrong `ln` lowers match quality), `sha256`, `buildUserData`, `sendEvent`, `sendActivated`, `sendPurchase`.

`buildUserData` returns `{ em, ph, fn, external_id }`, omitting null inputs. **If it ends up empty, do not send** — Meta rejects it and the rejection is silent in aggregate reporting.

Send `external_id: sha256(String(userId))` on every event, matching the client's `setUserID`. **Correction worth recording:** `external_id` is not a bootstrap identifier — Meta cannot resolve it alone. It becomes a join key only once co-observed with a GAID or real email. It does **not** rescue the 217 masked-email users on its own.

Payload: `POST /{META_GRAPH_VERSION}/{META_APP_ID}/events`, `action_source: 'app'`, plus `app_data`. **App events post against the App ID (`1348293100400435`), not the web pixel id** — confirm the exact path against Meta's Conversions API for App Events docs before implementing, since this is the piece most likely to have shifted.

⚠️ **`action_source: 'app'` requires `app_data.extinfo`.** Without a usable array Meta may accept the event but exclude it from app-campaign attribution — reads as "events arriving, attribution zero". `extinfo` is positional; slot 0 (`'a2'`) and slot 1 (package name) must be right. Have the client attach device fields via an optional `app_context` (`react-native-device-info` is already a dependency), validated server-side. **Verify in Test Events that the event is classified as an app event before shipping.**

`sendEvent`: axios `timeout: 3000`, one retry on network error or 5xx, `console.warn('[meta-capi] …')` on final failure, never throws. Early-return unless `META_CAPI_ENABLED === 'true'` — ships dark.

### 4e. Purchase CAPI hook

`sevices/iap.service.js` → `processAndroidPurchase`. Read PII + cohort before the commit at `:748`; fire `sendPurchase(...).catch(() => {})` after the commit and after the Redis invalidation at `:753-754`. Never awaited, never thrown.

**Do not** hook the duplicate-purchase branch at `:583-616` — it commits and returns `{ duplicate: true }`, and firing there double-counts on every Google retry. **Do not** hook `processApplePurchase:1023`; leave a comment at `iap.controller.js:92` saying why.

**Value:** never derive from `iap_purchases.amount` or `courses.price`. `:383` hardcodes `"INR"` and the price is the catalogue price, not what Google charged; `:890-891` applies a minor-unit heuristic. Use the client-supplied `priceAmountMicros` / `priceCurrencyCode` (new optional validated fields on `/iap/verify-purchase/android`), fall back to `course.price` + `'INR'` with a warning, and **skip the event if value is null/NaN/≤ 0**.

### 4f. Env vars

`META_CAPI_ENABLED=false`, `META_CAPI_ACCESS_TOKEN`, `META_APP_ID=1348293100400435`, `META_GRAPH_VERSION=v21.0`, `META_TEST_EVENT_CODE`. Add to `.env.example` and the env matrix in `docs/ARCHITECTURE.md`.

**Not** `META_DATASET_ID` / the pixel id — app events key off the App ID (see Step 1.3). And do not reuse the name `META_APP_SECRET`, which is taken in the webhook repo for WhatsApp and means something else.

### 4g. Tests + docs (mandatory before close)

- `Test/services/metaCapi.test.js` — table-driven email/phone/name normalisation; empty `user_data` → no HTTP call; bad value → no call; flag off → no call; axios rejection resolves without throwing; payload shape (`action_source === 'app'`, `extinfo[0] === 'a2'`).
- `Test/services/metaCohort.test.js` — UPSC-tagged course → `'upsc'`; untagged → `'untagged'`; studied-course path wins over onboarding; onboarding fallback when no study match; DB error → `'untagged'` + warn, no throw.
- `Test/controllers/sqliteActivation.test.js` — crossing 10 sets the column and returns the flag once; a second sync returns no flag; a batch jumping 0 → 25 fires exactly once; an already-activated user is untouched; **a resolver throw does not roll back the sync transaction**.
- `Test/services/iapMetaHook.test.js` — `sendPurchase` called once after commit, **not** on the duplicate branch, rejection doesn't reject `processAndroidPurchase`.

Docs: `docs/DATABASE_SCHEMA.md` (new column), `docs/ARCHITECTURE.md` (new services, env vars, fire-and-forget contract), `docs/API_INVENTORY.md` + `routes/docs/iap.openapi.js` (new optional request fields; new `cohort` field on the verify-purchase response; new `meta_activated` + `cohort` fields on the sync response), `docs/DOMAIN_GLOSSARY.md` (cohort, activation), CLAUDE.md Domain gotcha (*Meta CAPI value must never come from `iap_purchases.amount`*).

---

## Step 5 — Website: cohort label

UPSC ad destination URLs carry `?cohort=upsc`.

| File | Change |
|---|---|
| `src/utils/marketingAttribution.js` | add `'cohort'` to `MARKETING_KEYS` — one line |
| `src/utils/metaPixel.js` | read stored attribution; attach `content_category` to `trackAppDownload` + `trackSignup` when cohort is `upsc` |
| `Test/utils/metaPixel.test.js`, `Test/utils/marketingAttribution.test.js` | Vitest, per the frontend CLAUDE.md |

The plumbing already exists: `RouteTracker.jsx:19` calls `captureMarketingAttribution(location.search)` on every route change, persisting first-touch params (already including `fbclid` and UTMs) to sessionStorage. So the value survives the visitor browsing around before pressing the badge.

No new component, no route change, no landing page work. Not `trackLogin` — nothing to learn there.

**Whitelist the value — accept only `'upsc'`.** Otherwise `?cohort=<anything>` is persisted and shipped to Meta, letting a crafted URL inject junk categories into your reporting.

**Validate at read time in `metaPixel.js`, not at capture time.** `marketingAttribution.js` also feeds `record_activity`, where raw values are legitimately useful for analytics. Filter only where the value crosses into Meta.

**When the param is absent, omit `content_category` entirely** rather than sending `'untagged'`. Most web traffic is organic, so labelling all of it would be noise. (The app does the opposite — there `untagged` is meaningful and drives the tagging worklist.)

Because the dataset is shared, web events must keep `action_source: 'website'` and app events `'app'`, or web signups get counted in app-campaign reporting.

---

## Deduplication

The client and server both send `Activated` and `Purchase`. Meta collapses them when **`event_name` and `event_id` both match within 48 hours**.

- `Activated` → `activated_<userId>`
- `Purchase` → the Google Play `transactionId`, live shape `GPA.3301-6302-2094-33260`, byte-identical on both sides with no prefix or transformation

Get this wrong and every conversion counts twice: reported ROAS doubles, bidding optimises against fiction, and you find out weeks later when Ads Manager and MySQL disagree by exactly 2×.

Document the contract in exactly two places so it cannot drift: a comment at `iapService.ts:284` and one on `sendPurchase`, each naming the other.

**Why both sides at all:** the client carries the GAID, which is the only thing that links the event to the ad click — the server has no access to it. The server carries verified truth from the database plus hashed PII, surviving app kills and network failures. Neither alone is sufficient.

---

## Verification

1. **Release-build check first** — build a release APK and confirm events arrive **from the release build**, not just debug. This is the ProGuard trap.
2. **Test Events, app** — confirm `Activated` fires when a user crosses 10 cards and **does not** fire on subsequent syncs. Confirm `CompleteRegistration` fires once on fresh signup and not on later logins.
3. **Test Events, server** — with `META_TEST_EVENT_CODE` set, confirm the CAPI event appears labelled **Server** and is classified as an **app** event. If it shows as web, `extinfo` is wrong.
4. **Dedupe** — with the test code set, both App and Server copies appear (Test Events deliberately doesn't dedupe); confirm the `event_id`s are byte-identical. Then without it, confirm the live dataset shows **one** conversion, not two.
5. **Event Match Quality** — target > 6.0. Expected: `em` ~98.5%, `ph` ~56%, `fn` ~99.7%, `external_id` 100%.
6. **Web** — load with `?cohort=upsc`, press a badge, confirm `Lead` carries `content_category: upsc`. Load without it, confirm the field is absent.
7. `npx jest` (backend, RN) and `npm test` (web).
8. **First 7 days** — daily compare Meta `Activated` against a PostHog first-time-`flashcard_answered` count; a large gap means ProGuard stripping or a broken gate. Weekly grep Loki for `[meta-capi]`.

---

## Sequencing

| Ship | Contents | Store release? |
|---|---|---|
| **0** | Meta Business setup | no — blocks 1 and 3 |
| **1** | Migration + activation detection + cohort resolver + CAPI module + Purchase hook + tests + docs, `META_CAPI_ENABLED=false` | **no** — deployable alone |
| **2** | Web cohort param | no |
| **3** | RN app: SDK, native config, ProGuard, `meta.ts`, identity, event call sites, store-price + `app_context` fields, tests. **Play Data safety update in the same submission.** | **yes — Play review** |
| **4** | Flip `META_CAPI_ENABLED=true`, fire one real `Activated`, confirm in Events Manager, **then** create the campaign optimising on unfiltered `Activated` | no |

Run the migration before Ship 1 code reaches traffic. Ship 3 is the long pole — start it as soon as Ship 0 yields the App ID and Client Token.

---

## Risks — ordered by how quietly they fail

1. **ProGuard strips the SDK in release only.** Debug works, production sends nothing.
2. **Missing the Play App Signing key hash.** Google re-signs your bundle with *its* key before delivery, so the certificate on a real user's device is not your upload key. Register only the upload hash and every Play install is untrusted while your local release build works fine. Covered — `xOXMRk+mZcGiQ8YT1DXiUd5rrOI=` is registered — but re-check if the signing key is ever rotated.
3. **Malformed `app_data.extinfo`** with `action_source: 'app'` — events accepted, never attributed.
3. **Activation gate fires more than once, or never.** Batch syncs are why the count can't be compared to exactly 10, and why the `AND meta_activated_at IS NULL` guard carries the correctness.
4. **Untagged courses** — ~16–20% of purchases; the label silently covers only part of the funnel if nobody reads the warnings.
5. **Wrong purchase `value`** from the catalogue price + hardcoded INR.
6. **`CompleteRegistration` firing on login** if ever moved off the `isNewUser` / `register.fulfilled` gates.
7. **Volume per ad set** — ~30–46 activations/week organically is still under Meta's ~50/wk threshold. Use **few ad sets** so conversions concentrate; `CompleteRegistration` is the fallback goal.
9. **pnpm + patch-package + autolinking** — budget time for a clean `pnpm install` + `./gradlew clean assembleRelease`.

*(An earlier draft listed "shared dataset cross-contamination" here — dropped. App and web are separate data sources, so web and app `CompleteRegistration` were never at risk of being conflated.)*

## Separate bug found while planning (not in scope)

`razorpay-webhook-server/src/services/apple.subscription.service.js` divides Apple's `price` by 100, but Apple reports **milliunits** — the divisor should be 1000. iOS `iap_purchases.amount` is **10× inflated** (course 2190 at ₹399 stored as `5990.00`; course 2204 at ₹999 as `12990.00`). Own ticket, needs a backfill decision. Do not wire that column into Meta later.

## Open items

- Meta App ID / Client Token / Business Manager ownership of dataset `973157488384012` — placeholders.
- Whether the existing WhatsApp Meta app can host the Android platform.
- `react-native-fbsdk-next` compatibility with RN 0.79.5 on the old architecture.
- Whether Meta tolerates a partially-empty `extinfo` array.
- **Does the app have `phone_number` in memory** when identity is set? 55% of users have one, but the auth response may not carry it.

## Production checklist

- [ ] Meta Business Manager setup (Step 1) — blocks everything
- [ ] Run `node database/migrations/addMetaActivatedAt.js` on prod before Ship 1 serves traffic
- [ ] Env vars on prod: `META_CAPI_ENABLED`, `META_CAPI_ACCESS_TOKEN`, `META_DATASET_ID`, `META_GRAPH_VERSION`, `META_TEST_EVENT_CODE`
- [ ] ProGuard keep rules verified against a **release** build
- [ ] Play Data safety + Advertising ID declaration updated in the same submission as `AD_ID`
- [ ] Add `?cohort=upsc` to every UPSC ad destination URL
- [ ] Tag the ~16–20% of purchased courses missing a UPSC tag
- [ ] Verify dedupe produces one conversion per event, not two
