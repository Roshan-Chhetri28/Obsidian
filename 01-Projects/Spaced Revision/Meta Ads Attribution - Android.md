---
Date: 2026-07-29
tags:
  - Task
  - Plan
Status:
  - Not Started
  - Important
---

# Meta Ads Attribution — Android App (SDK + Conversions API)

## Context

Meta ads currently fire a single `Lead` pixel event when someone presses an app-store badge on the website (`spaced-revision-sern-frontend/index.html:157`, helpers in `src/utils/metaPixel.js:9`). That measures **intent only** — nothing about who installed, registered, or paid.

The UPSC cohort is meant to install the app and purchase in-app; Sikkim PSC (SPSC) stays on the website. The RN app has **zero Meta instrumentation**, so Meta cannot attribute a single install or in-app purchase, and App Promotion campaigns cannot run at all.

Goal: make Android installs, registrations and purchases visible to Meta, attributed to the ad that produced them, split by cohort.

## Decisions taken

| Decision | Choice |
|---|---|
| Platform | **Android only.** iOS entirely out of scope. |
| Approach | Meta SDK (`react-native-fbsdk-next`) + server-side Conversions API. No MMP. |
| ATT / extra screens | None |
| Campaign optimisation | `CompleteRegistration` first; switch to `Purchase` when volume clears ~50/wk per ad set |
| Purchase event | Initial purchase only — Android uses `type: 'in-app'` consumables, **no renewals exist** |
| Refunds | **Out of scope** |
| Cohort split | Event parameters + Custom Conversions on one dataset. **No second pixel.** |
| Untagged course | Send `content_category: 'untagged'` **and** log a warning naming the course_id |

## Verified ground truth

| Fact | Evidence |
|---|---|
| Android is the only client→server purchase path | `iapService.ts:184` posts to `/iap/verify-purchase/android`; iOS branch at `:224` returns before any network call |
| Android products are consumables, not subscriptions | `type: 'in-app'`, `finishTransaction({ isConsumable: true })` — every Purchase is an initial purchase |
| Purchase hook point | `iapService.ts:284` — beside `posthog.capture('purchase_completed', { platform: 'android' })` |
| Server commit point | `sevices/iap.service.js:748`, Redis invalidation `:753-754` |
| Duplicate-purchase branch commits and returns early | same file `:583-616` — **must not** fire CAPI |
| Registration emitters (only two) | `authSlice.ts:30` (email), `useAuth.ts:91` (OAuth, gated on backend `isNewUser`) |
| Package name | `com.spacedrevision`, activity `com.spacedrevision.MainActivity` |
| Cohort course IDs (live) | UPSC = 2190, 2201, 2257, 100044 · SPSC = 370, 2160, 2166, 2185, 2204, 100035, 100036, 100037 |
| Untagged purchase share | 7 of 44 Android rows (16%), dominated by course 225 "B.Ed. Entrance Exam" |
| Match-quality inputs | email 100%, name 99.7% (single column), phone 55.7%, no dob/gender. 217 unmatchable emails (166 Apple relay + 51 `@sprv.com`) |
| Build constraints | `enableProguardInReleaseBuilds = true`, `newArchEnabled = false`, hermes on, pnpm + patch-package, autolinking on |

## Known limitations (accepted)

- **iOS invisible to Meta** — ~20% of IAP purchases. iOS purchases never reach the main backend (`iapService.ts:224`); entitlements come from Apple ASSN into `razorpay-webhook-server/src/services/apple.subscription.service.js:646`. Future iOS work starts there.
- **Volume below Meta's learning threshold** — ~55 IAP purchases all-time. Hence registration-first optimisation.
- Only 12 of 1391 courses carry a UPSC/SPSC tag. Admin tagging pass needed, not a code fix.

---

## Step 1 — Meta Business setup (human, no code)

Blocks everything.

1. Create/reuse a Meta app, add the **Android** platform only. Package `com.spacedrevision`, activity `com.spacedrevision.MainActivity`. Add release **and** upload key hashes (`keytool -exportcert … | openssl sha1 -binary | openssl base64`) — the SDK is not trusted without them.
2. Record **App ID** and **Client Token** (Settings → Advanced).
3. Events Manager → dataset `973157488384012` → Connect data source → add the Android app. **Do not create a second pixel.**
4. Business Settings → assign the ad account to both the app and the dataset (Manage permission), else App Promotion campaigns can't select the app.
5. System user `capi-server` → assets: dataset (Manage) + app (Manage) → generate non-expiring token with `ads_management` + `business_management`. This is `META_CAPI_ACCESS_TOKEN`.
6. Event priority ranking: `Purchase`, `CompleteRegistration`, `InitiateCheckout`, `CompleteTutorial`, `ViewContent`, …
7. Custom Conversions on the dataset: `Reg — UPSC`, `Reg — Sikkim PSC`, `Purchase — UPSC`, `Purchase — Sikkim PSC`, plus `Purchase — Untagged` as an ops alarm.

⚠️ Do **not** reuse the Meta app behind `WHATSAPP_ACCESS_TOKEN` / `META_APP_SECRET` in the webhook repo without checking — a WhatsApp Business app has different review and permission state.

---

## Step 2 — RN app: Android native config

`pnpm add react-native-fbsdk-next`. Autolinking handles registration.

⚠️ **Check `peerDependencies` against RN 0.79.5 with `newArchEnabled=false` before merging.** If the current major requires the new architecture, pin the last major supporting the old bridge.

Do **not** run `pod install` — iOS is out of scope. Leave a Podfile comment saying fbsdk is intentionally unconfigured on iOS.

**`res/values/strings.xml`** — `facebook_app_id`, `facebook_client_token`. Both are public values shipped in the APK; safe to commit, same as the existing `GOOGLE_WEB_CLIENT_ID` handling.

**`AndroidManifest.xml`**
- `<uses-permission android:name="com.google.android.gms.permission.AD_ID" />`
- Inside `<application>`: `com.facebook.sdk.ApplicationId`, `ClientToken`, `AutoInitEnabled=true`, `AutoLogAppEventsEnabled=true`, `AdvertiserIDCollectionEnabled=true`

`AutoLogAppEventsEnabled=true` is what produces the automatic **install** and **activate** events — the entire top of the funnel. Never disable.

**`MainApplication.kt`** — one line in `onCreate()` after `SoLoader.init(...)`: `AppEventsLogger.activateApp(this)`. Guarantees activation fires on cold starts where the JS bundle is slow. No `sdkInitialize` needed (AutoInit handles it).

**`proguard-rules.pro` — CRITICAL**
```
-keep class com.facebook.** { *; }
-keepclassmembers class com.facebook.** { *; }
-dontwarn com.facebook.**
```
Release builds run R8. Without these, **the SDK is stripped in release only** — debug works, production silently sends nothing. The existing `-keep class com.facebook.react.**` rule covers React Native, **not** the Facebook SDK.

**Play Console** — the `AD_ID` permission requires updating the Data safety declaration (Advertising ID → collected, for Advertising/Marketing + Analytics) and the Advertising ID declaration in App Content. **A release with `AD_ID` and a stale Data safety form is rejected.** Same submission.

---

## Step 3 — RN app: event layer

New `src/analytics/meta.ts`, sibling of the existing `posthog.ts`. Tests in `src/analytics/__test__/meta.test.ts`. Shape mirrors `src/features/study/analytics.ts` — typed closed set of names plus thin named functions, not a generic wrapper class.

Contract: `isEnabled()` hard-gates on `Platform.OS === 'android'`; `getCohort()`; `trackMeta()`; `trackMetaPurchase()`; `setMetaUser()` / `clearMetaUser()`. Every call try/caught and short-circuited on iOS, so nothing can crash there.

### CompleteRegistration — first-class

The optimisation event. **Must fire exactly once per genuinely new user, never on login.**

Two call sites, both already gated on an authoritative new-user signal:

| Site | Add after the existing `posthog.capture('user_registered', …)` |
|---|---|
| `authSlice.ts:30` (email signup, inside `if (response.success)`) | `setMetaUser(...)` + `trackMeta('CompleteRegistration', { registration_method: 'email', … })` |
| `useAuth.ts:91` (`trackOAuthAuthenticated`, inside `if (isNewUser)`) | same, with `registration_method: authMethod` |

Piggy-backing on the existing PostHog gate means the two systems can never disagree about who is new — the most likely source of a broken funnel.

**Cohort caveat:** `examTagIds` is populated by `StepExam` *after* signup, and OAuth users skip Preferences entirely. So `getCohort()` returns `'untagged'` for most registrations. Fix: fire `CompleteRegistration` as specified with whatever is known, and *additionally* fire `CompleteTutorial` with the real cohort when the exam step completes (in `useOnboardingSelections` where `setExamTagIds` is dispatched — not in `StepExam.tsx`, keep components render-only). Cohort-split registration reporting keys off `CompleteTutorial`; `CompleteRegistration` stays the clean high-volume optimisation event.

Rejected alternative: delaying `CompleteRegistration` until after exam selection — it would drop the event entirely for OAuth users and anyone who taps Skip, exactly the volume the campaign needs.

Add `clearMetaUser()` in `signOut` beside `posthog.capture('user_logout')`, so a shared device doesn't attribute the next signup to the previous person's hashed PII.

### Purchase

Call site `iapService.ts:284`, **Android branch only**. Leave the iOS branch at `:238` untouched.

**The value problem.** At `:284` only `productId`, `courseIdNumber`, `transactionId` are in scope — no price. Fix: add a module-level `pendingPurchasePrices` Map mirroring the existing `pendingPurchaseCourseIds` Map (same pattern, same lifecycle, no new abstraction), populate it in `requestProductPurchase` where the store price is already known, read it at `:284`. `value = micros / 1_000_000`, currency from the store.

Fallback if absent (pending purchase recovered by `checkPendingPurchases:327` after a restart): re-call `getProductDetails`. If that also fails, **skip the event and warn** — a Purchase with a wrong value is worse than a missing one.

Payload carries `event_id: purchase.transactionId` (the dedupe key), `content_ids`, `content_category`, `order_id`, `num_items`.

Optionally also fire `InitiateCheckout` at `:847` beside `purchase_initiated` — cheap, gives the funnel a mid-point while Purchase volume is low.

### Test setup

Mock `react-native-fbsdk-next` in `jest.setup.js` beside the existing `posthog-react-native` mock. Tests: cohort mapping for `[2]` / `[7]` / `[]` / `[2,7]`; iOS no-op asserts zero SDK calls; `amount <= 0` and `NaN` skip; SDK throw does not propagate; `CompleteRegistration` fires once on register and not on failure; `clearMetaUser` on logout; Android purchase uses the **store** price not the course price. Run `npx jest src/analytics src/features/iap`.

---

## Step 4 — Backend: Conversions API

New `services/metaCapi.js` (correctly-spelled `services/`, matching `services/accrualPolicy.js`). CommonJS.

The masked-Apple-email check inlines the two domain constants with a comment pointing at `utils/appleAuthHelpers.js` as canonical.

**Exports:** `normalizeEmail` (lowercase/trim; null for `@privaterelay.appleid.com` and `@sprv.com`), `normalizePhone` (digits only; 10 digits → `91` + digits, covering 98.9% of rows; 12 starting `91` as-is; otherwise null), `normalizeName` (first token → `fn` only — `users.name` is a single column, and a wrong `ln` lowers match quality), `sha256`, `buildUserData`, `sendEvent`, `sendPurchase`.

**Guard:** if `user_data` ends up with no keys at all, do not send — Meta rejects it and the rejection is silent in aggregate reporting. `external_id` is always present (userId always known), which is why the 217 masked-email users still match on `external_id` + `fn`.

**Payload:** `POST /{META_GRAPH_VERSION}/{META_DATASET_ID}/events` with `event_name: 'Purchase'`, `event_time` from `purchaseTimeMillis / 1000`, `event_id` = Google order id, `action_source: 'app'`, `user_data`, `custom_data`, `app_data`.

⚠️ **`action_source: 'app'` requires `app_data.extinfo`.** Without a usable array Meta may accept the event but exclude it from app-campaign attribution — a silent failure that looks like "events are arriving, why is nothing attributed". `extinfo` is positional; slot 0 (`'a2'` for Android) and slot 1 (package name) must be right. Have the client attach device fields via an optional `app_context` on the verify request (`react-native-device-info` is already a dependency), validated in `validators/iap.validator.js`. Verify in Test Events that the event is classified as **app** before shipping.

### Where the send happens

1. **Before** the commit at `:748`, while the connection is checked out, read buyer PII and cohort into locals — avoids a second pool checkout and keeps the release-in-`finally` discipline.
2. **After** the commit and after Redis invalidation `:753-754`, before the return: `sendPurchase({...}).catch(() => {})` — fire-and-forget, never awaited, never thrown. A Meta outage must never roll back or slow a purchase.
3. **Do not** hook the duplicate-purchase branch at `:583-616` — it commits and returns `{ duplicate: true }`, and firing there double-counts on every Google retry.
4. **Do not** hook `processApplePurchase:1023` or `/iap/verify-purchase/apple`. Unreachable from the app today; hooking them creates a second Purchase source if iOS is ever wired up. Leave a comment at `iap.controller.js:92` saying so.

`sendEvent`: axios `timeout: 3000`, one retry on network error or 5xx, `console.warn('[meta-capi] …')` on final failure, never throws. Feature flag `META_CAPI_ENABLED !== 'true'` returns early — ships dark, enabled by env change with no redeploy.

### Cohort resolution (authoritative)

Purchase-time cohort comes from the **course**, not the buyer's declared interest:

```sql
SELECT t.tag FROM course_tag_selection s
JOIN course_tag t ON t.id = s.tag_id
WHERE s.course_id = ? AND t.tag IN ('UPSC','SPSC') LIMIT 1
```

`UPSC → upsc`, `SPSC → sikkim_psc`, no row → `untagged`. No course carries both (verified), so `LIMIT 1` is safe.

Untagged: send `'untagged'`, never omit, and `console.warn('[meta-capi] course %s has no UPSC/SPSC tag …', courseId)`. That warning is the worklist — grep Loki weekly. Fires on ~16% of Android purchases today.

### Value and currency

`insertIAPPurchase` writes `amount = courses.price` and hardcodes `currency = "INR"` at `:383` — the *catalogue* price, not what Google charged.

1. **Never** derive Meta `value` from `iap_purchases.amount` or `courses.price` when a better source exists.
2. Use the client-supplied store price: `priceAmountMicros / 1_000_000` + `priceCurrencyCode`, added as optional validated fields on the verify request.
3. Fall back to `course.price` + `'INR'` only when absent — and warn, so you can see how often the good path misses.
4. **Skip the event if `value` is null, NaN, or `<= 0`.** A zero-value Purchase poisons value optimisation; a missing one merely under-reports.
5. Do **not** fix the DB `amount`/`currency` columns here — real bug, needs a backfill decision, would make the PR unreviewable. File separately.

### Env vars

`META_CAPI_ENABLED=false` (ship dark), `META_CAPI_ACCESS_TOKEN`, `META_DATASET_ID=973157488384012`, `META_GRAPH_VERSION=v21.0`, `META_TEST_EVENT_CODE` (empty in prod). Add to `.env.example` and the env matrix in `docs/ARCHITECTURE.md`. Do not reuse the name `META_APP_SECRET` — taken in the webhook repo for WhatsApp, means something else.

### Tests + docs (mandatory before close)

`Test/services/metaCapi.test.js` — table-driven over email/phone/name normalisation, empty `user_data` → no HTTP call, bad value → no call, flag off → no call, axios rejection resolves without throwing, payload shape (`action_source === 'app'`, `extinfo[0] === 'a2'`).
`Test/services/iapMetaHook.test.js` — called once after commit, **not** on the duplicate branch, rejection doesn't reject `processAndroidPurchase`.

Docs: `docs/ARCHITECTURE.md` (new service + env vars + fire-and-forget contract), `docs/API_INVENTORY.md` + `routes/docs/iap.openapi.js` (new optional request fields), `docs/DOMAIN_GLOSSARY.md` (cohort), CLAUDE.md Domain gotcha (*Meta CAPI value must never be read from `iap_purchases.amount`*).

---

## Step 5 — Web pixel cohort tagging

`spaced-revision-sern-frontend/src/utils/metaPixel.js` — add `content_category` (cohort) to `trackAppDownload` and `trackSignup`, keeping `'app_download'` on `content_name`. Because the dataset is shared, **every web event must keep `action_source: 'website'`** and every app event `'app'` — otherwise web signups get counted in app-campaign reporting.

Open item: the web cohort source (course page tag, or web onboarding pick) hasn't been traced yet.

---

## Deduplication

**Key = the Google Play order ID.** Client sends `purchase.transactionId` at `:284`; server sends `data.transactionId` from the same field. Live shape: `GPA.3301-6302-2094-33260`. Identical raw string both sides, no prefix, no transformation.

Meta dedupes when `event_name` **and** `event_id` match within 48h; the server fires seconds after the client, comfortably inside.

Document the contract in exactly two places so it cannot drift: a comment at `iapService.ts:284` and one on `sendPurchase`, each naming the other.

**Client/server cohort disagreement:** both events share an `event_id`, Meta keeps one, so which `content_category` survives is non-deterministic — which quietly breaks the Custom Conversions. Fix: have the client send the **course-derived** value too, via a small static map in `meta.ts` seeded from the verified IDs above. That list will drift from the DB; acceptable only because the server is authoritative and its warning log is the drift detector. Document the asymmetry in a comment. If you skip the client-side map, do not create cohort Custom Conversions on `Purchase` at all.

---

## Verification

1. **Test Events, app** — install a debug build, confirm `CompleteRegistration` fires once on fresh signup and **not** on a subsequent login.
2. **Test Events, server** — set `META_TEST_EVENT_CODE`, run a sandbox purchase, confirm the event appears labelled **Server** and is classified as an **app** event. If it shows as web, `extinfo` is wrong.
3. **Release-build check** — build a release APK and confirm events arrive **from the release build**, not just debug. This is the ProGuard trap.
4. **Dedupe** — one sandbox purchase with the test code shows both App and Server events (Test Events deliberately doesn't dedupe); confirm the two `event_id` values are byte-identical. Then without the test code, wait 30–60 min and confirm the live dataset shows **one** Purchase, not two. N purchases must produce N, not 2N.
5. **Event Match Quality** — target > 6.0. Expected: `em` ~98.5%, `ph` ~56%, `fn` ~99.7%, `external_id` 100%. Verify a masked-email user produces an event with `em` absent but `external_id` + `fn` present, and that it is **accepted**. If EMQ sits below 5, phone coverage is the highest-leverage fix.
6. `npx jest` in both repos.
7. **First 7 days:** daily compare Meta `CompleteRegistration` vs PostHog `user_registered` (Android) — a large gap means ProGuard stripping or a misplaced event. Daily compare Meta `Purchase` vs `SELECT COUNT(*) FROM iap_purchases WHERE platform='android'`. Weekly grep Loki for `[meta-capi]`.

---

## Sequencing

| Ship | Contents | Store release? |
|---|---|---|
| **0** | All Meta Business setup | no — blocks 1 and 3 |
| **1** | Backend CAPI + validator + OpenAPI + docs + tests, `META_CAPI_ENABLED=false` | **no** — deployable alone |
| **2** | Web pixel cohort params | no |
| **3** | RN app: SDK, native config, ProGuard, `meta.ts`, call sites, store-price + `app_context` fields, tests. **Play Data safety update in the same submission.** | **yes — Play review** |
| **4** | Flip `META_CAPI_ENABLED=true`; launch campaigns optimising for `CompleteRegistration` | no |
| **5** | After ~50 Purchases/wk per ad set, switch optimisation to `Purchase` | no — gated on measured volume, not a date |

Ship 1 alone is useful and zero store risk: server-side Purchase attribution for the existing installed base immediately, and it validates the payload and dedupe key before the app release is locked. Ship 3 is the long pole (Play review + Data safety). Start it as soon as Ship 0 yields the App ID and Client Token.

---

## Risks — ordered by how quietly they fail

1. **ProGuard strips the SDK in release only.** Debug works, production sends nothing, discovered weeks later. The existing `-keep class com.facebook.react.**` does not cover it.
2. **Malformed `app_data.extinfo` with `action_source: 'app'`.** Events accepted, never attributed. Looks like "events arriving but attribution zero".
3. **Untagged courses** — 16% of Android purchases, dominated by the top-selling course 225. If nobody reads the warnings, cohort reporting silently covers only 84% of revenue.
4. **Client/server cohort disagreement** on `Purchase` — non-deterministic survivor breaks Custom Conversions.
5. **Wrong `value`** from `courses.price` + hardcoded INR — plausible-looking but wrong ROAS silently misdirects budget.
6. **`CompleteRegistration` firing on login** if it's ever moved off the `isNewUser` / `register.fulfilled` gates onto a screen mount or auth-state effect. Worst possible failure given campaigns optimise on it. Locked by test.
7. **Shared dataset cross-contamination** — only `action_source` separates web and app registrations.
8. **pnpm + patch-package + autolinking** — native deps under pnpm's symlinked `node_modules` occasionally fail Gradle resolution. Budget time for a clean `pnpm install` + `./gradlew clean assembleRelease`.
9. **Someone wires iOS later** and creates a second Purchase source. Leave the explicit comment.

## Separate bug found while planning (not in scope)

`razorpay-webhook-server/src/services/apple.subscription.service.js` divides Apple's `price` by 100, but Apple reports **milliunits** — the divisor should be 1000. Result: iOS `iap_purchases.amount` is **10× inflated** (course 2190 at ₹399 stored as `5990.00`; course 2204 at ₹999 stored as `12990.00`). Needs its own ticket and a backfill decision. Do not let anyone wire that column into Meta later.

## Open items / unverified

- Meta App ID, Client Token, Business Manager ownership of dataset `973157488384012` — placeholders throughout.
- Whether the existing WhatsApp Meta app can host the Android platform.
- `react-native-fbsdk-next` compatibility with RN 0.79.5 on the old architecture.
- Whether Meta tolerates a partially-empty `extinfo` array (positional and documented, but untested).
- The web frontend's cohort source for `metaPixel.js`.

## Production checklist

- [ ] Meta Business Manager setup (Step 1) — blocks everything
- [ ] Env vars on prod: `META_CAPI_ENABLED`, `META_CAPI_ACCESS_TOKEN`, `META_DATASET_ID`, `META_GRAPH_VERSION`, `META_TEST_EVENT_CODE`
- [ ] ProGuard keep rules verified against a **release** build
- [ ] Play Data safety + Advertising ID declaration updated in the same submission as `AD_ID`
- [ ] Tag the ~16% of purchased courses missing a UPSC/SPSC tag
- [ ] Verify dedupe produces one conversion per purchase, not two
