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

Meta ads currently fire a single `Lead` pixel event when someone presses an app-store badge on the website (`spaced-revision-sern-frontend/index.html:157`, helpers in `src/utils/metaPixel.js:9`). That measures **intent only** — it says nothing about whether the app was installed, who registered, or who paid.

The UPSC cohort is meant to install the mobile app and purchase in-app, while Sikkim PSC (SPSC) stays on the website. Because the RN app has **zero Meta instrumentation** today, Meta cannot attribute a single install or in-app purchase, and App Promotion campaigns cannot be run at all.

Outcome wanted: Android installs, registrations and purchases become visible to Meta, attributed to the ad that produced them, split by cohort — so UPSC campaigns can be measured and optimised.

## Decisions taken

| Decision | Choice |
|---|---|
| Platform | **Android only.** iOS entirely out of scope. |
| Approach | Meta SDK (`react-native-fbsdk-next`) + server-side Conversions API |
| ATT / extra screens | None — Android only, so not applicable |
| Campaign optimisation | `CompleteRegistration` first; switch to `Purchase` when volume clears ~50/wk per ad set |
| Purchase event | Initial purchase only (Android uses `type: 'in-app'` consumables — no renewals exist) |
| Refunds | Out of scope |
| Cohort split | Event parameters + Custom Conversions on one dataset. **No second pixel.** |
| Untagged course | Send `content_category: 'untagged'` **and** log a warning naming the course_id |

## Known limitations (accepted)

- **iOS is invisible to Meta.** Currently ~20% of IAP purchases. Separately, iOS purchases never reach the main backend at all — `iapService.ts:224` skips server verification, and iOS entitlements are granted by Apple ASSN into `razorpay-webhook-server/src/services/apple.subscription.service.js:646`. Any future iOS work must start there, not in the main backend.
- **Volume is below Meta's learning threshold.** ~55 IAP purchases all-time (35 of them course 2190). Hence optimising for registration first.
- Only **12 of 1391 courses** carry a UPSC/SPSC tag; ~20% of real purchases are on untagged courses. Needs an admin tagging pass — not a code fix.

---

## Step 1 — Meta Business setup (human, no code)

Blocking prerequisite. Nothing else works until this exists.

1. Business Settings → Apps → register the app with the **Android package name** (read `applicationId` from `react-native-app/android/app/build.gradle`).
2. Link app → ad account, and add it as a data source on **existing dataset `973157488384012`** (do not create a new pixel).
3. Collect `facebook_app_id` and `facebook_client_token` for Step 2.
4. Generate a **system-user access token** for CAPI (Step 4).
5. Events Manager → rank the AEM 8-event priority list: `Purchase` top, `CompleteRegistration` second.
6. Create two Custom Conversions filtered on `content_category` = `upsc` / `sikkim_psc`.

---

## Step 2 — RN app: SDK + Android native config

Repo: `react-native-app`. Follow `react-native-app/CLAUDE.md` — pseudocode first, tests co-located in `__test__/`, no premature abstraction, folder placement per `docs/dev_documents/FolderPattern.md`.

| File | Change |
|---|---|
| `package.json` | add `react-native-fbsdk-next` |
| `android/app/src/main/res/values/strings.xml` | `facebook_app_id`, `facebook_client_token` |
| `android/app/src/main/AndroidManifest.xml` | `com.facebook.sdk.ApplicationId` + `ClientToken` meta-data; `com.google.android.gms.permission.AD_ID` permission |
| `android/app/src/main/.../MainApplication.kt` | SDK init only if autolinking doesn't cover it — verify before editing |

**Every SDK call must be guarded with `Platform.OS === 'android'`** so iOS is a silent no-op and nothing crashes on an untested platform.

---

## Step 3 — RN app: event layer

New file `src/analytics/metaEvents.ts`, sitting beside the existing `src/analytics/posthog.ts` (same pattern, same folder).

A module rather than inline calls is justified here despite the "no premature abstraction" rule, because `event_id` generation must be identical to the server's — that consistency is the whole point and cannot be duplicated by hand at each call site.

Exports:
- `initMetaSdk()` — no-op on iOS
- `trackRegistration(cohort)`
- `trackPurchase({ transactionId, courseId, value, currency, cohort })`
- `event_id` = the Google Play `transactionId` (the only key shared with the server)

Call sites:

| Event | Where | Note |
|---|---|---|
| App activate | app init / `src/app/Providers.tsx` | automatic once SDK initialised |
| `CompleteRegistration` | auth flow — alongside the existing PostHog identify in `src/features/auth/hooks/useAuth.ts` / `store/authSlice.ts` | **most important event** — campaigns optimise on it. Must fire exactly once per new user, not on every login |
| `Purchase` | `src/features/iap/services/iapService.ts:284` — the **Android** branch, next to the existing `posthog.capture('purchase_completed')` | ignore the iOS branch at `:238` |

Cohort client-side reads onboarding picks via `selectOnboardingExamTagIds` (`src/features/onboarding/store/onboardingSelectionSlice.ts:81`), already populated by `StepExam.tsx`.

Tests: `src/analytics/__test__/metaEvents.test.ts`; update existing IAP tests. Run `npx jest`.

---

## Step 4 — Backend: Conversions API

Repo: `spaced-revision-sern-backend`. CommonJS, conventions per its `CLAUDE.md`.

New files:
- `utils/metaHash.js` — SHA-256 normalisation helpers. Lowercase + trim email; phone to E.164 (**~99% of stored numbers are bare 10-digit with no country code — prepend `91`**; ~87 outliers already have `+` or `91`). Skip masked Apple addresses by reusing the existing `utils/appleAuthHelpers.js` → `isMaskedAppleEmail()` (217 users, 1.5%, would never match).
- `services/metaCapi.js` — builds `user_data` / `custom_data`, POSTs to the Graph API.

Hook point: `sevices/iap.service.js` (directory misspelling is real) → `processAndroidPurchase`, **after the commit at `:748`**. Wrapped in try/catch and fire-and-forget — a Meta API failure must never fail or delay a purchase.

Cohort server-side: join `course_tag_selection` → `course_tag` on `course_id`. Tag `UPSC` = id 2, `SPSC` = id 7; no course carries both. No match → `'untagged'` + warning log naming the course_id.

**Purchase value — the thing most likely to silently produce wrong data.** Do not blindly reuse the existing amount. `sevices/iap.service.js:383` hardcodes currency `"INR"`, the price comes from the **courses table rather than the store**, and `:890-891` applies a minor-unit heuristic (`price >= 100 ? price/100 : price`). Recommended: pass the real store price from the client — `priceAmountMicros` is already available from `getProductDetails` (`iapService.ts:604`) — through the existing `/iap/verify-purchase/android` payload, and fall back to the course price only when absent. Sending a wrong value is worse than sending none, because bidding will act on it.

Env vars (add to `.env.example`): `META_CAPI_ACCESS_TOKEN`, `META_DATASET_ID`, `META_CAPI_ENABLED` (kill switch).

Tests per `docs/UNIT_TESTING_GUIDE.md`: `Test/utils/metaHash.test.js`, `Test/services/metaCapi.test.js`. Run `npx jest Test/<path>`.

Docs to update: `docs/ARCHITECTURE.md` (new outbound service), plus a Domain gotcha in the backend `CLAUDE.md` about the value/currency trap.

---

## Step 5 — Web pixel cohort tagging (small)

`spaced-revision-sern-frontend/src/utils/metaPixel.js:9` — add `content_category` to the existing `Lead` call so web and app events split by the same parameter.

---

## Verification

1. **Events Manager → Test Events** with a `test_event_code` — confirm app events and CAPI events both arrive.
2. **Dedupe check** — one sandbox purchase must produce **one** conversion, not two. This is the failure that inflates ROAS by exactly 2× and is easy to miss.
3. **Event Match Quality** score on the dataset — expect moderate: email 100% (minus 1.5% masked), phone ~55%, no DOB/gender columns exist.
4. `npx jest` in both repos.
5. Android debug build + real Play sandbox purchase, end to end.
6. Confirm `CompleteRegistration` fires once for a new signup and **not** on subsequent logins.

## Risks

- **Purchase value correctness** (above) — the only item here that produces confidently wrong numbers rather than missing ones.
- ~20% of purchased courses untagged → cohort leakage until an admin tags them.
- Low conversion volume → Purchase-optimised ad sets would not exit learning; hence registration-first.
- Native change → needs a Play Store release; the CAPI half can ship independently of it.

## Production checklist

- [ ] Meta Business Manager setup (Step 1) — blocks everything
- [ ] Env vars on prod: `META_CAPI_ACCESS_TOKEN`, `META_DATASET_ID`, `META_CAPI_ENABLED`
- [ ] Tag the ~20% of purchased courses missing a UPSC/SPSC tag
- [ ] New Play Store release (native change)
- [ ] Verify dedupe produces one conversion per purchase, not two
