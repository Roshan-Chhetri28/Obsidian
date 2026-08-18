---
Date: 2026-08-18
tags:
  - Task
  - Bug
Status:
  - Urgent
  - Important
---

# Question Bank: foreign-authored MCQs inflate the progress denominator

Branch: TBD · Repos: `spaced-revision-sern-backend`, `spaced-revision-sern-frontend`

## The bug

Course **100037** (owner 3254 "crack spsc"), subject Economics (9995), topic Development Sectors (4416). MCQ meter reads **5/9** forever.

Topic 4416 resolves to 9 MCQs: 4 direct (65038–65041, author 3254) + 5 mapped on 2026-08-15 — id 29294 (author 3254, from course 1863) and **43429–43432 (author 62 "Pedestal Education", from course 2185)**. Those four are another educator's content and must not be counted or shown.

### One topic, three answers

| Surface | Mapped-branch owner rules | Result |
|---|---|---|
| `getCourseProgress` — `routes/api/analytics.js:363-370` (cards `:390-401`) | **none** | **9** ← denominator |
| `GET /mcq/browse/topic/:id` — `routes/api/mcq.js:2214`, `:2253` | `mcq.user_id IN (viewer, educator)` AND `mm.user_id IN (viewer, educator)` | **5** ← correct |
| `GET /mcq/study/course/:id` — `routes/api/mcq.js:1900` | none; `INNER JOIN mcqattribute` hides them by accident | 5 |

`browse/topic` is the reference implementation. Every **direct** branch is already owner-scoped (`analytics.js:360`, `mcq.js:1722`, `:1789`, `:1884`, `:1954`); only the **mapped** branch was never given the same treatment. Same class of leak `Test/routes/countOwnerScope.test.js` guards — that fix covered the direct branch only.

The `study/*` endpoints look correct today only because the attribute rows happen not to exist (43429–43432 have 4–5 `mcqattribute` rows across all users vs ~200 each for the direct four; **zero** of the 329 subscribers has one). That is luck, not scoping — `PUT /mcq/study/:id` (`mcq.js:614-628`) is a real upsert, so one answer creates the row and the leak appears.

### Blast radius: 8 MCQs, 0 cards

Of 8,281 `mcq_mappings` rows across 12 courses, only 8 have an author who is not the course owner — all authored by user 62:

| Course | Topic | MCQ ids |
|---|---|---|
| 100037 | 4416 | 43429, 43430, 43431, 43432 |
| 2160 | 3030, 3034 | 43716, 43722, 43941, 43944 |

Everything else is 3254 mapping their own content between their own courses. All 6,465 `card_mappings` rows are owner-authored — card-side change is a no-op today, done for symmetry. Two of the eight (43430, 43431) were mapped in by **admin user 3**, so they also fail an `mm.user_id` check.

### How it got there

`PUT /mcq/assign/:mcq_id/:course_id/:subject_id/:topic_id` (`mcq.js:1379`) is guarded by `auth` only — **any logged-in user can map any MCQ into any course/subject/topic**. It records `mm.user_id = req.user.id`, so an admin-performed mapping is indistinguishable from a stranger's.

### Explicitly not doing

- No attribute seeding, no backfill — the upsert at `mcq.js:614-628` creates the row on first answer.
- No change to the denominator's `LEFT JOIN mcqattribute` — the meter counts what is in the topic, not what the user has touched.

## Second bug (same page)

`/suscribe/study/.../9994/4455?tab=questionBank` — counter reads "MCQ · Question 97 of 93" and there is dead whitespace under the last question.

Topic 4455 holds 97 live MCQs: 93 by the educator (3254) + **4 authored by a student, user 100701 "Tenzin B"**. Those are *direct*, not mapped, and are already scoped correctly — a normal viewer sees 93, Tenzin sees 97. **That must keep working.**

- **Counter:** numerator `activeIndex + 1` indexes `runMcqs` (`BrowseMcq.jsx:226-230`); denominator `listTotal` is `pagination.total` off the browse response (`:149-152`). `BrowseMcq.jsx:104` already computes `useBackendCounts` — the condition under which the backend total is trustworthy — and `listTotal` never consults it. For an owner the list is not the paginated response at all: `Browse.jsx:322-363` seeds from the full Redux topic list and merges the API page on top.
- **Whitespace:** at `BrowseMcq.jsx:277-283` the inline `height: calc(100dvh - columnTop)` and `xl:overflow-y-auto` are gated by **two different breakpoint tests**. Fixed height + `overflow: visible` = background stops at `100dvh − columnTop`, content runs past, document grows. Either (a) `useViewportFillTop.js:5,31` gates on `window.innerWidth < 1280` which *includes* the scrollbar while Tailwind's `xl:` excludes it — so ~1280–1295px applies the height without the overflow; or (b) `columnTop` goes stale, since the `ResizeObserver` on `document.body` (`:57-58`) is inert once the column scrolls internally and the only real re-measure path is the dep-less `useEffect` at `:71-73`.

---

## Phases

### - [ ] Phase 1 — Owner-scope every mapped branch

Mirror `browse/topic` (`mcq.js:2214`, `:2253`): require **both** `<content>.user_id IN (viewer, owner)` and `<mapping>.user_id IN (viewer, owner)`. Use the owner the surrounding query already scopes by — `courses.user_id` course-scoped, `topics.user_id` topic-scoped, `subjects.user_id` subject-scoped, exactly as each direct branch does.

- [ ] `routes/api/analytics.js` — `getCourseProgress` MCQ subquery `:363-367`; cards mirror `:390-401` ← **this is the 5/9 fix**, denominator drops to 5
- [ ] `routes/api/mcq.js` — `study/topic` `:1738`, `study/subject` `:1805`, `study/course` `:1900`, `study/user` `:1970`
- [ ] `routes/api/card.js` — corresponding `card_mappings` branches
- [ ] `services/revisionDueCounts.js` — `:29-34` cards, `:51-58` MCQs
- [ ] `utils/courseRevisionDues.js` — `card_mappings` + `mcq_mappings` branches (~`:40-75`)

`routes/api/appNextTopic.js:129,201` and `getTopicCompletion` (`analytics.js:540`) call `getCourseProgress`, so they inherit the fix.

### - [ ] Phase 2 — Close the write-side hole

- [ ] `PUT /mcq/assign/...` (`mcq.js:1379`) — reject unless caller owns the destination course/topic or is an admin
- [ ] Record `mm.user_id` as the **destination course owner**, not `req.user.id`, so admin-performed mappings aren't hidden by the new check. Same in `mapTopicCards` (`notes.js:680`) and `mapTopicMCQs` (`:713`)
- [ ] Optional cosmetic: backfill `mm.user_id` for rows 43430, 43431 (already excluded by author)

### - [ ] Phase 3 — Cache

`analytics.js:91` caches `course_progress:${user_id}:${course_id}:${type}` for 300s — the meter won't move for up to 5 min after deploy.

- [ ] **PROD:** flush `course_progress:*` once at deploy, or bump the cache-key prefix so old entries orphan
- [ ] Do NOT add a `redis.keys()` scan to any request path (`invalidateCourseProgressCache` at `analytics.js:25-51` has that branch)

### - [ ] Phase 4 — Frontend counter

- [ ] `src/components/suscribe/BrowseMcq.jsx` — `listTotal` (`:149-152`) falls back to `runMcqs.length` whenever the run isn't purely backend-paged; reuse the `useBackendCounts` condition at `:104` instead of only checking for a level filter
- [ ] Clamp so the numerator can never exceed the denominator

### - [ ] Phase 5 — Dead whitespace

- [ ] `useViewportFillTop.js:5,31` — replace `window.innerWidth < XL` with `window.matchMedia('(min-width: 1280px)').matches`, subscribing to that MediaQueryList instead of the bare `resize` listener
- [ ] `BrowseMcq.jsx:279-280` — apply `overflow-y: auto` inline alongside the inline height rather than via the `xl:` class
- [ ] Re-measure on layout changes above the column: observe the previous siblings (`Study.jsx:510` header, `Browse.jsx:770` toolbar) instead of `document.body`

### - [ ] Phase 6 — Tests

Pattern: `Test/routes/countOwnerScope.test.js` — `jest.mock('../../database/mysqlConnector')` + `setupTestDb` (`Test/helpers/testDb.js:20`), namespaced fixture ids, text markers as assertion handles.

- [ ] `Test/routes/mappedOwnerScope.test.js` (new) — topic with owner's own MCQs + one mapped from the owner's other course + one authored by a third educator and mapped in. Assert `analytics/course-progress`, `browse/topic`, `study/topic`, `study/subject`, `study/course`, `revisondues` all return the same set and all exclude the third educator's item — **including when the viewer has been given an `mcqattribute` row for it**, which is what makes it fail against today's code
- [ ] Same file: `PUT /mcq/assign/...` rejects a non-owner/non-admin caller and records `mm.user_id` as the course owner
- [ ] `Test/routes/countOwnerScope.test.js` stays green
- [ ] `Test/components/suscribe/BrowseMcq.test.jsx` — run longer than `pagination.total` ⇒ counter clamps
- [ ] `Test/hooks/useViewportFillTop.test.js` — `top` is null in exactly the cases where `@media (min-width:1280px)` doesn't match

---

## Verification

- [ ] `npm test` in `spaced-revision-sern-backend`
- [ ] `npm test` + `npm run lint` in `spaced-revision-sern-frontend`
- [ ] Topic 4416 returns **5** from all three surfaces (today: 9 / 5 / 5) — `getCourseProgress` MCQ subquery, `browse/topic` count for a generic viewer, `study/topic` for a subscriber
- [ ] The 8 foreign-authored MCQs (43429–43432 in 100037; 43716, 43722, 43941, 43944 in 2160) appear in **no** subscriber-facing count or list
- [ ] Totals for the other ten courses **unchanged** — all their mappings are owner-authored, so any diff means the scoping is too tight
- [ ] Browser, as a course-100037 subscriber, after flushing `course_progress:*`: Economics meter reads **5/5**, Development Sectors lists exactly 5 questions
- [ ] Regression: topic 4455 still shows **93** to a normal subscriber and **97** to user 100701 — Tenzin's four are direct, not mapped
- [ ] Topic 4455 in browser: counter never exceeds its total at any scroll position, no dead space under the last question. Check at a window width in the **1280–1300px band** first, then full width
