

 Context

 The Razorpay webhook currently reads purchase metadata (user_id, course_id, couponCode, payment_type, tokens, final_price, rewardPointsUsed) from the client-supplied
 Razorpay notes object — which the frontend sets, so it is untrusted and easy to drift. We already moved payment_type to orders (webhook reads orders.payment_type).
 Now move the rest of the purchase metadata onto the orders row so the webhook selects everything from the DB and notes becomes a fallback only.

 orders already has user_id, course_id, amount (= final charged amount). payment.type and orders.payment_type already include self_study (the first todo is already
 satisfied — verify only). Add three new orders columns and stamp them at create-order; webhook reads them.

 Decisions (confirmed): AI-token (conversation) purchase also writes an orders row (keeps existing conversation_order); webhook resolves the coupon by looking up the coupon row from orders.coupon_id (new getCouponById) and runs existing discount/referral logic unchanged.

 Data model — new orders columns (migration in database/selfStudy.js, guarded, re-run safe)

 - coupon_id INT NULL, FK → coupons(id) ON DELETE SET NULL (coupons PK is id INT).
 - reward_points INT UNSIGNED NOT NULL DEFAULT 0.
 - ai_token INT UNSIGNED NOT NULL DEFAULT 0.

 Add one guarded fn per column (pattern: existing columnExists), wire into run(), re-run node database/selfStudy.js.

 Stamp at create-order (spaced-revision-sern-backend/routes/api/payment.js)

 Every orders INSERT gains coupon_id, reward_points, ai_token:
 - POST /create-order full path (~:383): coupon_id = coupon?.id ?? null, reward_points = discountValidation.adjustedRewardPoints, ai_token = 0. (coupon already
 fetched ~`:141`.)
 - POST /create-order self_study path (~:110): coupon_id null, reward_points 0, ai_token 0 (flat tier).
 - POST /create-emi-order (~:796): coupon_id = coupon?.id, reward_points = adjusted, ai_token 0.
 - POST /create-installment-order (~:1346): coupon_id null, reward_points = adjusted, ai_token 0.
 - POST /credit-order (~:1933, conversation credits): add a new orders INSERT alongside the existing conversation_order insert — payment_type='conversation_credit',
 ai_token = tokens, reward_points = rewardPointsUsed ?? 0 (add to body destructure), coupon_id null. This makes orders.payment_type resolve correctly in the webhook
 (no more notes fallback for conversation).

 Webhook (razorpay-webhook-server/src/services/webhook.service.js)

 Extend the existing SELECT … FROM orders WHERE order_id = ? (currently only payment_type) to also fetch coupon_id, reward_points, ai_token, user_id, course_id,
 amount. Then prefer the orders row over notes for every field (fall back to notes only when no order row, for safety):
 - user_id, course_id ← orders (else notes).
 - rewardPointsUsed ← orders.reward_points (else notes).
 - tokens ← orders.ai_token (else notes.tokens).
 - final_price ← orders.amount (else notes.final_price).
 - coupon: orders.coupon_id → Coupons.getCouponById(db, coupon_id) (new method in src/helpers/Coupons.js, mirror of getCouponByCode) → pass couponRow.code downstream
 as the existing couponCode, so processPurchase / EmiPayment / Payment helpers and referral logic stay unchanged.
 - payment_type resolution unchanged (already orders-based).

 Keep notes destructure as the fallback layer; do not remove it.

 Docs + tests

 - docs/DATABASE_SCHEMA.md (backend + webhook): document orders.coupon_id/reward_points/ai_token, ownership (main backend stamps, webhook reads).
 - routes/docs/payment.openapi.js: note the new stamped columns (no request-body change — they're derived server-side).
 - webhook docs/WEBHOOK_FLOWS.md + docs/INTEGRATION_CONTRACTS.md: webhook now sources coupon/reward/tokens/user/course from orders; notes = fallback.
 - Test/routes/payment.test.js: assert create-order stamps coupon_id+reward_points; credit-order stamps an orders row with
 payment_type='conversation_credit'+ai_token.

 Verification

 1. Run node database/selfStudy.js → 3 new columns; confirm via sql-mcp (coupon_id FK, reward_points/ai_token UNSIGNED).
 2. create-order with coupon + reward points → orders row has correct coupon_id, reward_points.
 3. credit-order → orders row payment_type='conversation_credit', ai_token=<tokens>; conversation_order row still written.
 4. Simulate webhook on each order_id → it reads metadata from orders (log/confirm no reliance on notes); coupon/referral + reward deduction + token credit still
 work.
 5. payment.type / orders.payment_type already contain self_study (verify, no change).

 ---
 Self-Study Purchase Tier

 Context

 Educators currently sell a course as one thing: full access at one price. Product wants a second, cheaper tier — Self-Study — where the buyer gets all course content
 (flashcards, MCQs, notes, test series) except videos. Live classes run through external Google Meet, so they need no in-app gating; only video endpoints get
 filtered.

 Each course can optionally offer self-study at an educator-set flat price (no discounts, no coupons, no early-bird, no EMI). Learners choose between "Full Course"
 and "Self-Study" on the payment page (Figma plan-selector). The system records which tier a user bought so video access can be gated per enrollment.

 Decisions (confirmed with product owner):
 - Block videos only in code (live classes are external Google Meet).
 - Self-study price is educator-set per course; if unset, tier is not offered.
 - Self-study is a flat price — no coupon / early-bird / reward points / EMI.
 - Self-study access duration = same as full payment.

 Data model changes

 ┌───────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │   Table   │                                                     Change                                                     │
 ├───────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ courses   │ add self_study_price DECIMAL(10,2) NULL — educator-set; null/0 = tier not offered                              │
 ├───────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ community │ add community_type ENUM('full_payment','self_study') NOT NULL DEFAULT 'full_payment' — per-enrollment tier     │
 ├───────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ orders    │ add payment_type ENUM('full_payment','EMI','conversation_credit','self_study') NOT NULL DEFAULT 'full_payment' │
 ├───────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ payment   │ extend type ENUM to add 'self_study'                                                                           │
 └───────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 All 4 ALTERs go in one new script spaced-revision-sern-backend/database/selfStudy.js (not spread across table files). Each ALTER guarded by a column-exists / enum
 check so re-runs are idempotent. End the script with process.exit(0) so the run signals completion. Run once via node database/selfStudy.js.

 Tier carried via orders.payment_type, not Razorpay notes. orders is created by the main backend (spaced-revision-sern-backend) at create-order with payment_type
 stamped. Payment success is processed by the separate razorpay-webhook-server, which looks up the order and sets community.community_type + payment.type from
 orders.payment_type. Orders = single source of truth for purchase type.

 Backend

 ▎ Order is created by main backend at create-order (stamps orders.payment_type). Payment success → community/payment rows are written by the separate
 ▎ razorpay-webhook-server on the Razorpay event, reading orders.payment_type to set community.community_type + payment.type. (Webhook flow detail: see "Webhook
 ▎ server" section below, pending explorer.)

 Migrations — single script database/selfStudy.js

 One new file in spaced-revision-sern-backend/database/. Uses the repo DB pool (getConnection() from database/mysqlConnector.js), runs all 4 ALTERs guarded
 (column/enum exists check, pattern paymentable.js:84-95), logs each, ends with process.exit(0). Run: node database/selfStudy.js.
 - ALTER TABLE courses ADD COLUMN self_study_price DECIMAL(10,2) NULL DEFAULT NULL
 - ALTER TABLE community ADD COLUMN community_type ENUM('full_payment','self_study') NOT NULL DEFAULT 'full_payment'
 - ALTER TABLE orders ADD COLUMN payment_type ENUM('full_payment','EMI','conversation_credit','self_study') NOT NULL DEFAULT 'full_payment'
 - ALTER TABLE payment MODIFY type ENUM('full payment','emi','conversation credit','self_study') NOT NULL DEFAULT 'full payment' (existing literals use spaces; new
 value uses underscore per product)

 A) create-order — stamp orders.payment_type (routes/api/payment.js:39-368)

 1. Destructure plan from body; const isSelfStudy = plan === 'self_study'.
 2. Add self_study_price to course SELECT (payment.js:65-68).
 3. After if(!course) guard (:72): if isSelfStudy, validate self_study_price > 0 (else 400 "self-study not offered"), set finalOrderAmount =
 Math.ceil(self_study_price), and skip the whole discount/coupon/reward/EMI block (:74-304) — wrap it in else. finalOrderAmount (outer let) feeds the unchanged
 Razorpay options (:308-313).
 4. Guard reward-points response augmentation (:317-329) behind if(!isSelfStudy).
 5. orders INSERT (:330-348): add payment_type column = isSelfStudy ? 'self_study' : 'full_payment'. Wire the other order creators to write their value too so the
 column is always meaningful: create-emi-order (~`:759) → 'EMI'; conversation-credit order route → 'conversation_credit'`.
 6. Keep setting Razorpay order notes.payment_type (existing behavior the webhook reads) for backward-compat, but the webhook will prefer the orders lookup below.

 B) Payment success is written by the webhook server — see "Webhook server" section

 The payment + community rows for paid course purchases are inserted by razorpay-webhook-server, not by the main backend. Main-backend store-details /
 community/:course_id are the synchronous client / free-enroll paths and should also stamp tier from orders for consistency:

 - store-details (payment.js:1534, has order_id at :1548): SELECT payment_type FROM orders WHERE order_id = ? → set payment.type ('self_study' else 'full payment').
 For self_study wrap coupon/discount validation (:1583-1690) in if(!isSelfStudy). Thread type into CouponExecutor.insertOrder (sevices/CouponExecutor.js:106-114) —
 add type col + placeholder; processPurchase (:292-309) passes through with || 'full payment' fallback (keeps EMI payment.js:960 / checkout.controller.js callers
 correct).
 - community/:course_id (community.js:142): currently only gets final_price (:154); frontend also sends order_id. SELECT payment_type FROM orders WHERE order_id = ? →
 communityType (fallback: latest order for user+course, else 'full_payment'). Add community_type to the INSERT (:203-208). For self_study set payment/total_payment =
 course self_study_price (add to SELECT :178-188). Duration unchanged.

 Access predicate communitySubscriptionValidSql (services/synchronization.service.js:65-71) is tier-agnostic — unchanged.

 C) Read queries tolerating new enum (verify only, mostly no change)

 invoice.service.js:161/193/197 (passthrough; optional WHEN p.type='self_study' THEN 0 to avoid phantom invoice discount), stats.js:767/1452, admin.js:1202/1961/2056,
 Payment.controller.js:251/393 (derives type from community.emi, buckets self_study as "Full Payment" — extend to join community.community_type only if product wants
 self_study split in educator stats — flag to product).

 Docs + tests (mandatory per backend CLAUDE.md)

 - docs/DATABASE_SCHEMA.md: new columns + enum value. docs/API_INVENTORY.md + routes/docs/{payment,community}.openapi.js: new plan (create-order) and order_id
 (community) body fields.
 - Test/routes/payment.test.js + Test/routes/community.test.js (§8 boilerplate): self_study create-order flat price + skips coupon; missing/zero self_study_price →
 400; orders.payment_type stamped; community_type + payment.type = self_study. Run npx jest Test/routes/<file>.test.js.

 Webhook server (razorpay-webhook-server/)

 Authoritative writer of payment + community for paid purchases. Today it reads the tier from Razorpay notes.payment_type (webhook.service.js:161-169, values emi /
 conversation_credits / default) and never queries orders. Switch the tier source to orders.payment_type (per your instruction) and add the self_study branch.

 1. Read tier from orders — in handlePaymentCaptured (src/services/webhook.service.js) after beginTransaction (~`:182), before the dispatch (:252-367): SELECT
 payment_type FROM orders WHERE order_id = ?using the Razorpaypayment.entity.order_id. Normalize → resolvedType ('EMI'→emi, 'conversation_credit'→conversation,
 'self_study'→self_study, 'full_payment'/missing→default). Use resolvedTypefor dispatch instead ofnotes.payment_type(fall back to notes if no order row, for
 safety).user_id/course_id/couponCode/final_price` still come from notes.
 2. Dispatch branch (webhook.service.js:252-367): add else if (resolvedType === 'self_study') → goes through the default course-purchase path
 (Payment.storePaymentDatabase) but passes communityType: 'self_study'. (Same duration/access as full payment; flat amount already set at create-order.)
 3. community INSERTs add community_type (default 'full_payment'):
   - src/helpers/Payment.js — new-row INSERT (:364-374), trial→paid UPDATE (:236-246), complementary INSERT (~`:503`).
   - src/helpers/EmiPayment.js — INSERTs (:428-439, complementary ~`:507/:275) — pass 'full_payment'` (EMI is never self_study).
   - Thread a communityType param into Payment.storePaymentDatabase.
 4. payment.type = 'self_study': in src/helpers/Coupons.js processPurchase (:595) set paymentTypeEnum from resolvedType (self_study → 'self_study', emi → 'emi', else
 'full payment'); flows to insertOrder type column (:210-240, index [17]).
 5. Docs (mandatory per webhook CLAUDE.md): docs/WEBHOOK_FLOWS.md (new orders lookup + self_study branch), docs/DATABASE_SCHEMA.md (community_type,
 orders.payment_type, payment.type value), docs/INTEGRATION_CONTRACTS.md (tier now read from orders not notes).

 Note: accrualPolicy.js is copy-synced between main backend and webhook (per backend CLAUDE.md) — self_study touches neither, no sync risk.

 Video gating

 Gate the video-serving endpoints so a community_type='self_study' enrollee gets no videos. Affected:
 - routes/api/video.js — topic videos (GET /topic/:topic_id ~186), course videos (GET /course/:course_id ~573), all-course grouped (GET /all/:course_id ~636)
 - routes/api/youtube.js — parallel YouTube endpoints

 Reuse existing per-user community lookup already done for the trial filter (video.js:620-622 reads community[0].trial). Add: if that community row's
 community_type='self_study', return empty video list (or skip video branch). Owner/educator path unaffected.

 Course create/edit

 routes/api/course.js POST (/) and PUT (/:id) must accept and persist self_study_price. Find the course insert/update column lists and add the field (nullable).

 Frontend

 Educator add/edit course (spaced-revision-sern-frontend/)

 Add a "Self-Study" toggle + price input, mirroring the existing discount-toggle pattern.
 - src/components/dashboard/AddCourse.jsx — formData state (48-63), toggle/price block modeled on discount toggles (379-499), include self_study_price in submit
 payload (99-150).
 - src/components/dashboard/MyCourses.jsx — same in edit form: formData (47-62), toggleEdit populate (136-155), save payload (162-179), price block (~514-524 /
 580-699).
 - Toggle unchecked by default. When on, show price input. DaisyUI toggle toggle-primary.
 - src/actions/courses.js — addCourse (59-82) / editCourse (109-133) already POST/PUT whole formData; no change needed if field is in formData.

 Learner plan selector (Figma 2325:2242 desktop / 2338:4099 mobile)

 Two cards "Full Course" vs "Self-Study" with feature comparison, shown only when course.self_study_price > 0.
 - src/components/suscribe/Heading.jsx (~320-389) + src/components/payment/PaymentOptions.jsx — render plan choice. Selecting Self-Study: show flat self_study_price
 as the total, hide coupon/early-bird/EMI/reward UI, and on pay send plan:'self_study' to create-order; then send plan:'self_study' to store-details and order_id +
 plan to community/:course_id so backend reads the tier from the orders row.
 - Comparison rows from Figma: Live Classes ✓/✕, Videos ✓/✕, Flashcards ✓, MCQs ✓, Notes ✓, Test Series ✓. Self-study shows ✕ on Live Classes + Videos visually; only
 Videos enforced in backend (live classes = external Google Meet).
 - src/actions/courses.js create-order/store-details/subscribe thunks: pass plan and order_id through. Verify the subscribe thunk that calls community/:course_id
 includes order_id (Razorpay response already has it).

 Repos touched

 1. spaced-revision-sern-backend — migration script, create-order, store-details, community/:course_id, video gating, course create/edit, docs, tests.
 2. razorpay-webhook-server — orders lookup, self_study dispatch, community_type + payment.type writes, docs.
 3. spaced-revision-sern-frontend — educator add/edit form, learner plan selector, thunks.

 Verification
     │ routes/api/course.js POST (/) and PUT (/:id) must accept and persist self_study_price. Find the course insert/update column lists and add the field (nullable).                  │
     │                                                                                                                                                                                  │
     │ Frontend                                                                                                                                                                         │
     │                                                                                                                                                                                  │
     │ Educator add/edit course (spaced-revision-sern-frontend/)                                                                                                                        │
     │                                                                                                                                                                                  │
     │ Add a "Self-Study" toggle + price input, mirroring the existing discount-toggle pattern.                                                                                         │
     │ - src/components/dashboard/AddCourse.jsx — formData state (48-63), toggle/price block modeled on discount toggles (379-499), include self_study_price in submit payload          │
     │ (99-150).                                                                                                                                                                        │
     │ - src/components/dashboard/MyCourses.jsx — same in edit form: formData (47-62), toggleEdit populate (136-155), save payload (162-179), price block (~514-524 / 580-699).         │
     │ - Toggle unchecked by default. When on, show price input. DaisyUI toggle toggle-primary.                                                                                         │
     │ - src/actions/courses.js — addCourse (59-82) / editCourse (109-133) already POST/PUT whole formData; no change needed if field is in formData.                                   │
     │                                                                                                                                                                                  │
     │ Learner plan selector (Figma 2325:2242 desktop / 2338:4099 mobile)                                                                                                               │
     │                                                                                                                                                                                  │
     │ Two cards "Full Course" vs "Self-Study" with feature comparison, shown only when course.self_study_price > 0.                                                                    │
     │ - src/components/suscribe/Heading.jsx (~320-389) + src/components/payment/PaymentOptions.jsx — render plan choice. Selecting Self-Study: show flat self_study_price as the       │
     │ total, hide coupon/early-bird/EMI/reward UI, and on pay send plan:'self_study' to create-order; then send plan:'self_study' to store-details and order_id + plan to              │
     │ community/:course_id so backend reads the tier from the orders row.                                                                                                              │
     │ - Comparison rows from Figma: Live Classes ✓/✕, Videos ✓/✕, Flashcards ✓, MCQs ✓, Notes ✓, Test Series ✓. Self-study shows ✕ on Live Classes + Videos visually; only Videos      │
     │ enforced in backend (live classes = external Google Meet).                                                                                                                       │
     │ - src/actions/courses.js create-order/store-details/subscribe thunks: pass plan and order_id through. Verify the subscribe thunk that calls community/:course_id includes        │
     │ order_id (Razorpay response already has it).                                                                                                                                     │
     │                                                                                                                                                                                  │
     │ Repos touched                                                                                                                                                                    │
     │                                                                                                                                                                                  │
     │ 1. spaced-revision-sern-backend — migration script, create-order, store-details, community/:course_id, video gating, course create/edit, docs, tests.                            │
     │ 2. razorpay-webhook-server — orders lookup, self_study dispatch, community_type + payment.type writes, docs.                                                                     │
     │ 3. spaced-revision-sern-frontend — educator add/edit form, learner plan selector, thunks.                                                                                        │
     │                                                                                                                                                                                  │
     │ Verification                                                                                                                                                                     │
     │                                                                                                                                                                                  │
     │ 1. Migration: run node database/selfStudy.js → prints each ALTER result + exits 0. Confirm all 4 schema changes via sql-mcp.                                                     │
     │ 2. Educator: add course with self-study price → courses.self_study_price set. Edit: toggle off → price cleared/null.                                                             │
     │ 3. Learner UI: open paid course → plan selector shows only when self_study_price>0; pick Self-Study → coupon/EMI/reward hidden, Razorpay amount == self_study_price exactly.     │
     │ 4. Order: confirm orders.payment_type='self_study' after create-order.                                                                                                           │
     │ 5. Post-payment (webhook path): community.community_type='self_study', payment.type='self_study'. Full-payment purchase → both full_payment/full payment. EMI/conversation       │
     │ unchanged.                                                                                                                                                                       │
     │ 6. Video gate: self_study user → video endpoints empty; flashcards/MCQs/notes/test-series full. Full-payment user → videos present. Educator/owner always sees videos.           │
     │ 7. Boot both backends (npm run dev) and run new Jest suites (npx jest Test/routes/...).                                                                                          │
     ╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
