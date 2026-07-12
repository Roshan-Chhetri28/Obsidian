---
Date: 2026-07-12
tags:
  - spaced-revision
  - frontend
  - react
  - documentation
Status: living
---

> Part of [[00 - Repo Documentation Overview]]

# Spaced Revision — Web Frontend (`spaced-revision-sern-frontend`)

> Deep technical reference for the React 18 / Vite web client of Spaced Revision.
>
> The repo already ships two authoritative in-tree docs — `docs/ARCHITECTURE.md` (wiring diagram) and `docs/COMPONENT_INVENTORY.md` (every component folder → route → Redux slice). This note distills and cross-checks them against the actual source and highlights the load-bearing gotchas.

---

## 1. Overview

The web frontend is a single-page React 18 app built with Vite 7, Redux (classic thunk style on top of Redux Toolkit's `configureStore`), React Router 6, MUI 5 for the component layer, and TailwindCSS 3 + DaisyUI for utility styling. It talks to the Express backend over JWT (`x-auth-token`), streams chat over Socket.IO, does web push via Firebase Cloud Messaging, takes payments via Razorpay, and instruments itself with PostHog + a home-grown ClickHouse activity log.

### Commands (`package.json`)

| Command | What it does |
|---|---|
| `npm run dev` | Vite dev server, **port 3000**, `--host` enabled (LAN/tunnel accessible) |
| `npm run build` | Production build → `dist/` |
| `npm run preview` | Serve the production build locally |
| `npm run lint` | ESLint over `js,jsx`, `--report-unused-disable-directives --max-warnings 0` (**zero-warnings enforced**) |
| `npm run lint:fix` | Same, with `--fix` |
| `npm run prettier` | `prettier . --write` |
| `npm test` | `vitest --run --reporter verbose` (single run) |
| `npm run test:watch` | Vitest watch mode |
| `npm run test:coverage` | `vitest run --coverage` (v8, per-glob gates — see §2) |
| `npm run prepare` | `husky` (installs git hooks) |

### Package manager &amp; runtime — read this carefully

There is a real **mismatch between what Volta pins and what the project actually uses**:

- **Lockfile present:** `pnpm-lock.yaml` (354 KB) + `pnpm-workspace.yaml`. **There is no `yarn.lock` or `package-lock.json`.** The project has migrated to **pnpm**.
- **CI (`.github/workflows/ci.yml`)** uses `pnpm/action-setup@v4` (pnpm 10) on **Node 20**, runs `pnpm install --frozen-lockfile` then `pnpm test:coverage`.
- **But `package.json` `volta` still pins `node: 18.20.6` and `yarn: 1.22.22`.** The workspace-level `CLAUDE.md` also says "Yarn 1.22 via Volta, Node 18.20.6." Those are stale.
- **Practical rule (from the repo's own `CLAUDE.md`):** install new deps with **pnpm** so `pnpm-lock.yaml` stays in sync — CI runs `--frozen-lockfile` and a drifted lockfile fails the build.

`pnpm-workspace.yaml` also carries a large `overrides` block (security bumps for axios, dompurify, protobufjs, prismjs, undici, ws, vite, etc.) and an `allowBuilds` allowlist (`@firebase/util`, `core-js`, `esbuild`, `protobufjs`) — pnpm 10 blocks post-install scripts by default, so build-script packages must be opted in there. The same overrides are mirrored under `package.json` `overrides` / `pnpm.overrides` for redundancy.

---

## 2. Build &amp; tooling

### Vite (`vite.config.js`)

The config is a function of `mode` and loads `VITE_`-prefixed env with `loadEnv`. Notable parts:

- **Plugins:** `@vitejs/plugin-react` (Babel Fast Refresh) + a **custom inline plugin `replace-env-in-service-worker`** (see §7 — templates the Firebase service worker).
- **Dev server:** `port: 3000`, and `allowedHosts: ['.ngrok-free.dev', '.ngrok.io', 'localhost']` so ngrok tunnel URLs work without editing config each session. **No proxy is configured** — the app hits the backend directly at `localhost:8080/api/`; this works because the backend runs open CORS.
- **`optimizeDeps.include`:** `@mui/material`, `@mui/material/styles`, `@mui/icons-material` — forces Vite to pre-bundle MUI's barrel exports (otherwise hundreds of tiny module requests in dev).
- **Vitest block lives inside `vite.config.js`** (`test: { environment: 'jsdom', globals: true, setupFiles: ['./Test/setup.js'] }`), including a critical alias:
  ```js
  alias: [{ find: /^.*ecosystem\.config(\.js)?$/, replacement: resolve(__dirname, 'Test/stubs/ecosystemConfig.js') }]
  ```
  Because `ecosystem.config.js` is **gitignored** (absent on a fresh CI checkout), any app module importing it would fail to load under test. The alias redirects those imports to a committed stub (`Test/stubs/ecosystemConfig.js`) with localhost URLs.

### Coverage gates (v8)

Coverage is scoped to `src/utils/**`, `src/helpers/**`, `src/reducers/**`, `src/actions/**` (components are intentionally excluded — ~325 components tested via E2E, not units). Per-glob thresholds act as a regression ratchet:

| Glob | lines / funcs / branches / statements |
|---|---|
| `src/helpers/**` | 95 / 95 / 90 / 95 |
| `src/utils/**` | 85 / 85 / 80 / 85 |
| `src/reducers/**` | 90 / 85 / 80 / 90 |
| `src/actions/**` | 55 / 65 / 25 / 55 |

SDK/DOM-bound utils (`socket.js`, `firebase.js`, `mathjax.js`, `setAuthToken.js`, etc.) and admin-only actions are explicitly excluded from the gate.

### ESLint (`.eslintrc.cjs`)

Extends `eslint:recommended`, `plugin:react/recommended`, `plugin:react/jsx-runtime`, `plugin:react-hooks/recommended`, `plugin:import/recommended`, plus `eslint-config-prettier`. `no-unused-vars` is an **error** (args ignored via `^_`); `react/prop-types` and `react/no-unescaped-entities` are off. `import/resolver` resolves from `src`. **`--max-warnings 0` means any warning fails lint.** The repo convention is to fix lint issues rather than suppress them with disable comments.

### Prettier (`.prettierrc`)

`trailingComma: all`, `tabWidth: 2`, `semi: true`, `singleQuote: true`, `printWidth: 120`, `endOfLine: lf`, plugin `prettier-plugin-tailwindcss` (auto-sorts Tailwind classes).

### PostCSS (`postcss.config.js`)

`tailwindcss` + `autoprefixer`.

### Husky (`.husky/pre-commit`)

One line: **`npm test`**. Every commit runs the full Vitest suite locally. (Installed via the `prepare` script.)

### CI (`.github/workflows/ci.yml`)

Triggers on PR + push to `main` and `workflow_dispatch`. Single `test` job: checkout → pnpm 10 → Node 20 (pnpm cache) → `pnpm install --frozen-lockfile` → **`pnpm test:coverage`**. A failing test or a coverage-gate miss blocks merge.

### `ecosystem.config.js` — NOT a PM2 file here (major gotcha)

Despite the name (which elsewhere in this monorepo denotes PM2 configs), in **this** repo `ecosystem.config.js` at the repo root is the **API base-URL config module**. It exports `address`, `aiAddress`, `chatAddress`, `answerWritingAddress`, `transcoderSocketAddress`, branching on `process.env.NODE_ENV`:

```js
const b   = 'http://localhost:8080/api/';         // dev backend
const aws = 'https://aws.spacedrevision.com/api/'; // prod backend
// ...
if (process.env.NODE_ENV === 'production') { address = aws; /* ... */ }
else { address = b; /* ... */ }
export { address, aiAddress, chatAddress, answerWritingAddress, transcoderSocketAddress };
```

It is **gitignored** (see `.gitignore`) and therefore absent on CI — hence the Vitest stub alias above. It relies on Vite statically replacing `process.env.NODE_ENV` at build time (a React-compat behavior). A future third environment (staging) would break this two-branch design; the intended migration is `import.meta.env.MODE`.

---

## 3. App bootstrap

Two files, in order: `src/main.jsx` → `src/App.jsx`.

### `src/main.jsx` (entry, ~55 lines)

At module load it does four things:

1. **Service-worker registration** — registers `/firebase-messaging-sw.js` with `{ scope: '/' }`, waiting for `installing → activated` and posting `SKIP_WAITING` to a waiting worker. SW failures are logged, never thrown, so they can't block app start.
2. **MUI theme** — `const muiTheme = createTheme()` (named import is deliberate to avoid a Vite `createTheme_default` interop bug).
3. **PostHog** — `<PostHogProvider apiKey={VITE_PUBLIC_POSTHOG_PROJECT_TOKEN} options={{ api_host: VITE_PUBLIC_POSTHOG_HOST, defaults: '2026-01-30' }}>`.
4. **React 18 root** — renders `<App />`.

**Provider hierarchy (outermost → in):**

```
<React.StrictMode>
  <ThemeProvider theme={muiTheme}>          // MUI (main.jsx)
    <PostHogProvider ...>                    // PostHog (main.jsx)
      <App>
        <Provider store={store}>             // Redux (App.jsx)
          <Router>                           // BrowserRouter (App.jsx)
            <PostHogIdentitySync/> <DashboardCleanupWrapper/>
            <ScrollToTop/> <RouteTracker/> <ButtonTracker/>
            <Navbar/> <Alert/> <Routes/> <FooterPanel/>
```

Note there is **no global `TabStateProvider`, `ThemeProvider`(app-level), or `AlertProvider`** in the hierarchy. Alerts are a Redux slice rendered by a chrome component; `TabStateProvider` wraps only one subtree (see §7).

### `src/App.jsx` (~320 lines)

Module-level side effects (run before any render):
- `disableReactDevTools()` when `import.meta.env.MODE === 'production'`.
- `if (localStorage.token) setAuthToken(localStorage.token)` — sets the axios default `x-auth-token` header **synchronously before first paint**, so authed calls in `useEffect` work on first render.
- At the very bottom of the file: `library.add(fab, fas, far)` — registers the entire FontAwesome icon set once.

Inside the `App` component:
- **`loadUser()` once** — dispatched in a `useEffect` guarded by a `useRef` so React 18 StrictMode's double-mount doesn't double-fire.
- **Session-conflict popup** — listens for the global `window` event `session-popup` (dispatched by the axios interceptor, §9), shows `<SessionPopup/>`, redirects to `/` on close.
- **FCM init** — after mount, if a token exists, `await navigator.serviceWorker.ready` → `requestNotificationPermission()` → `sendTokenToBackend(token)`.
- **`PostHogIdentitySync`** and **`DashboardCleanupWrapper`** — small components defined *inside* `App` so they sit within `Router`/`Provider` context (§4, §7).
- **Chrome suppression** — `<Navbar/>` and `<FooterPanel/>` are hidden when the path starts with `/admin`, `/ai/`, or `/download`. These flags are computed once from `window.location.pathname` at mount (not reactive to navigation) — this is fine because each of those areas renders entirely distinct component trees anyway.

---

## 4. State management

### Store (`src/store.js`)

```js
const store = configureStore({
  reducer: rootReducer,              // src/reducers/index.js
  preloadedState: {},                // no SSR/hydration; everything starts empty
  devTools: process.env.NODE_ENV !== 'production',
});
```

`configureStore` (Redux Toolkit) is used **only for its defaults** — it wires up `redux-thunk` and Redux DevTools. The codebase does **not** use `createSlice`/`createAsyncThunk`; it uses classic hand-written reducers (`switch (action.type)`) and classic thunk action creators. That's the coexisting-legacy-style situation, and a known future refactor target.

### Reducers (`src/reducers/index.js`) — 38 slices

Combined with **plain `combineReducers` from `redux`** (not RTK):

```
alert, auth, course, subject, topic, card, community, user, suscribe, support,
threads, mcq, heading, content, leaves, strategy, carduser, mcquser, practice,
youtube, notification, doubt, video, ai, couponReducer, message, unreadReducer,
visitor, pdf, analytics, answerWriting, mcqFlag, activityLog, courseRevisionDues,
deck, deckSharing
```

State shape is `state.<slice>`. **Two slices are renamed at import time** — `couponReducer` (file `coupons.js`) and `unreadReducer` (file `unread.js`) — so components read `state.couponReducer` / `state.unreadReducer`, not `state.coupons` / `state.unread`. Reducers hydrate initial state from `localStorage` where relevant, e.g. `auth`:

```js
const initialState = { token: localStorage.getItem('token'), isAuthenticated: null, isAdmin: 0, loading: true, user: null };
```

### Actions (`src/actions/`) — 41 files

Async thunks of the classic form `export const foo = (args) => async (dispatch, getState) => { ... }`. Canonical example (`actions/auth.js`):

```js
export const loadUser = () => async (dispatch) => {
  if (!localStorage.token) { dispatch({ type: AUTH_ERROR, ... }); return; }  // skip 401 for anon
  setAuthToken(localStorage.token);
  try { const res = await axios.get(address + 'auth'); dispatch({ type: USER_LOADED, payload: res.data }); }
  catch (err) { handleError(err, dispatch, AUTH_ERROR); }
};
```

Conventions worth knowing:
- **`setAlert(msg, type)` (`actions/alert.js`)** is the canonical toast mechanism — dispatch it; the `layout/Alert.jsx` chrome reads `state.alert` and renders.
- **`handleError(error, dispatch, failType)`** in `auth.js` is the shared error→alert+dispatch helper; it deliberately swallows the `"No token, authorization denied"` message so anonymous visitors don't see it.
- **OAuth** (`googleLogin`/`facebookLogin`/`appleLogin`) stashes `oauth_intent`/`oauth_provider` in `sessionStorage` and does a full-page `window.open(${address}auth/${provider}, '_self')` redirect.

### `src/actions/types.js` — action strings **and** the API config re-export

`types.js` holds all action-type string constants **and, unusually, re-exports the API base URLs**:

```js
import { address as envAddress, aiAddress as envAiAddress, answerWritingAddress, chatAddress } from '../../ecosystem.config.js';
// ...
export { address, uploadAddress, aiAddress, answerWritingAddress, chatAddress, razorpayKey };
```

So many actions do `import { address, LOGIN_SUCCESS } from './types'` — pulling both a constant and a base URL from the same module. `uploadAddress` is pinned to the live host (`https://api.spacedrevision.com/api/`) regardless of env, and `razorpayKey` comes from `VITE_RAZORPAY_KEY`.

### No redux-persist

There is **no `redux-persist`**. Persistence is manual and ad-hoc via `localStorage`: the `auth` reducer writes/reads `token`, `is_admin`, `ownerStatus`; `setAuthToken` mirrors the token into the axios default header. (The mobile app uses redux-persist; the web app does not.)

### PostHog-in-thunks convention

The standing convention is to put PostHog `identify`/`capture` inside Redux thunks rather than hook wrappers, so events fire regardless of caller. In practice on the web app:
- **Identity** is handled centrally in `PostHogIdentitySync` (in `App.jsx`) — it reads `state.auth` and calls `posthog.identify(distinctId, {...})` on login / `posthog.reset()` on explicit logout (`distinctId` = lowercased email, falling back to `String(user.id)`).
- **Custom product events** are recorded through the **activity actions** (`actions/activity.js` → `record_activity(...)`), used by `RouteTracker`/`ButtonTracker` (§7), which write to the backend ClickHouse activity log. Direct `posthog.capture` appears in ~8 components. Grepping `src/actions/` shows no raw `posthog` imports — the analytics-in-thunks pattern here is expressed via `record_activity`, not the PostHog SDK.

---

## 5. Routing

React Router 6 with `BrowserRouter`, all routes declared inline in `src/App.jsx` (`<Routes>` around line 207–308). The catch-all `<Route path="*" element={<ErrorPage/>} />` is declared **first** inside `<Routes>` — harmless in RR6 because ranking (not order) decides the match, and `*` always ranks last.

### Route map (grouped)

**Public (no guard):** `/` Landing, `/whyitworks`, `/courses`, `/login`, `/register`, `/forgot-password`, `/reset-password`, `/verify-email`, `/aboutsr`, `/contactus`, `/howtostudy`, `/howtorememberfaster`, `/howtouse`, `/terms`, `/privacy-policy`, `/educator`, `/dashboard` (+ `/dashboard/community-courses`, `/dashboard/my-courses`), `/download` (mobile store redirect), `/community/:course_name/:course_id`, `/deck/join/:token` (public deck invite), `/team` + `/team/dashboard` (separate team auth), `/admin` → redirects to `/admin/login`, `/admin/login`.

**`<PrivateRoute>`-wrapped (require auth):** `/course/:course_name/:course_id` (Subjects), `/edit/:course_name/:course_id`, the `/study/...` family (CourseStudy / SubjectStudy / CardsByTopic), `/suscribe/:course_name/:course_id`, the `/suscribe/study/...` family (each additionally wrapped in `<CheckCourseAccess>`), `/course/:course_name/test/:course_id`, `/test-series/mcqs/:test_id`, `/test-series/:test_id/performance`, `/decks` (+ `:deck_id`, `/study`, `/share`), `/streak`, `/chat` (+ `/:course_name/:course_id`, `/directMessage/:id`), `/question-bank`, `/notifications`, `/support`, `/discuss/:course_id`, `/ai/:course_name/:course_id`, `/settings`, `/invoices`, `/referral`, `/liveclass/student|educator/...`, `/course/:course_name/:course_id/analytics`.

**`<AdminRoute>`-wrapped (require `is_admin`):** `/admin/dashboard`, `/admin/:page` (the `:page` param drives the admin sub-view inside `AdminDashboard`).

### Guards (`src/components/routing/`)

**`PrivateRoute.jsx`** — a `connect`-ed component reading `state.auth.{isAuthenticated, loading}`. Its key subtlety: it checks `localStorage.getItem('token')` **first**. If there is no token, it treats the visitor as anonymous *immediately* (without waiting for `auth.loading` to settle) and redirects to `/login?redirect=<encoded path>` — this avoids firing private API calls that would 401. It also has a special case: an anonymous hit to `/suscribe/:name/:id` carrying `?from=community-courses|my-courses` is redirected to the public `/community/:name/:id` page. Only when a token *is* present does it fall back to the usual `isAuthenticated && !loading` gating. Returns `<Outlet/>` when allowed.

**`AdminRoute.jsx`** — reads `state.auth` and derives `isAdmin = Boolean(state.auth?.isAdmin)`. Returns `null` while `loading`, else `<Navigate to="/admin/login"/>` unless `isAuthenticated && isAdmin`.

**`auth/CheckCourseAccess.jsx`** (not in `routing/`) — a *content-level* guard wrapping the `/suscribe/study/...` routes. It hits the backend to confirm owner / active subscriber / trial-visitor eligibility and renders children only on success (else an access-error page). It touches the `visitor` and `course` slices.

---

## 6. Component organization

Feature-based folders under `src/components/` — **45 folders** (the inventory is the source of truth: `docs/COMPONENT_INVENTORY.md`, which maps every folder → entry component → route → Redux slices). File counts give a rough sense of surface area:

| Area | Folder(s) | ~files | Purpose |
|---|---|---|---|
| **Admin** | `admin/` | 56 | Admin dashboard + sub-pages: accounts, reconciliation, lists, app-versions, direct payments, analytics, etc. Rendered under `/admin/:page` |
| **Analytics** | `analytics/` | 29 | Course analytics, streak dashboards, progress charts (Chart.js / Recharts) |
| **Subscribe / study** | `suscribe/` | 27 | Subscriber study hub — the tabbed course experience (hosts `TabStateProvider`) |
| **Team portal** | `team/` | 24 | Separate `/team` login + dashboard (distinct auth from main app) |
| **Layout / marketing** | `layout/` | 23 | Navbar, Alert, Footer/FooterPanel, ScrollToTop, Spinner, Landing, Courses, Whyitworks, Educator |
| **Dashboard** | `dashboard/` | 18 | Logged-in home: Dashboard, MyCourses, CommunityCoursesWrapper, Chat, Support, Settings |
| **Notes** | `notes/` | 11 | Rich-text notes editor (TipTap) + rendering |
| **Payments** | `payment/` | 10 | Razorpay flows: UnifiedPayment, EMI, installments, conversation credit, DirectPay |
| **MCQ** | `mcq/` | 10 | MCQ study/review UI |
| **Community** | `community/` | 10 | Public community-course browsing |
| **Cards** | `cards/` | 10 | Flashcard study (CardsByTopic etc.) |
| **Auth** | `auth/` | 9 | Login/Register/Forgot/Reset/VerifyEmail + `CheckCourseAccess`, admin login |
| **Test series** | `test_series/` | 8 | Timed MCQ test series + performance views |
| **Notifications** | `notification/` | 7 | In-app notification center |
| Others | `subjects/`, `question-bank/`, `ai/`, `decks/`, `content/`, `common/`, `celebration/`, `video/`, `tracking/`, `study/`, `shared/`, `reuseable_component/`, `pdfs/`, `liveclass/`, `doubt/`, `youtube/`, `topics/`, `terms/`, `routing/`, `practice/`, `editor/`, `edit/`, `chat/`, `answer_writing/`, `session/`, `referral_programme/`, `invoice/`, `info/`, `error/`, `discussion_forums/`, `app/` | 1–6 each | Focused feature areas |

**Before adding a component**, the repo's `CLAUDE.md` mandates: scan `docs/COMPONENT_INVENTORY.md`, grep `src/components|actions|reducers` for the feature name + synonyms, and extend/compose rather than fork. Update the inventory when adding a folder/page, and ship unit tests in the same change.

---

## 7. Contexts &amp; cross-cutting

### `src/context/TabStateContext.jsx` — the only React context

Preserves per-tab UI state (expanded items, scroll positions, selected items, inputs, preview state) as the user switches between the sub-tabs of a course view. Key facts:

- **Sections:** `study`, `videos`, `readPdf`, `test`, `answerWriting` (`DEFAULT_TAB_STATE`).
- **Persists to `sessionStorage`** under key `tabStates` (survives reload, resets on tab close — deliberately *not* `localStorage`).
- API: `saveState(section, stateOrUpdater)`, `getState(section)`, `clearState(section)`, `saveScrollPosition`/`getScrollPosition`, `clearAllStates`. `getState` reads from a `ref` (not the state value) so the callback is stable and won't retrigger consumers' effects.
- **Scoping gotcha:** `TabStateProvider` is **not** mounted globally in `App.jsx`. It wraps exactly one subtree — `src/components/suscribe/Suscribe.jsx`. `useTabState()` throws outside a provider, so only the subscribe/study area may use it.

### Custom hooks (`src/hooks/`)

Just one file: **`useMathJax.js`** — `useMathJax()` waits for global MathJax readiness and returns `{ mathJax, isLoading, error }` (backed by `src/utils/mathjax.js`'s `waitForMathJax()`). Most components use the `<MathHTML>` / lazy math renderers in `src/components/shared/` instead of the raw hook.

### Global trackers (`src/components/tracking/`)

Rendered as invisible children inside `<Router>`; both write to the backend activity log via `record_activity` (`actions/activity.js`):

- **`RouteTracker.jsx`** — on every `location.pathname`/`search` change, emits `record_activity('PAGE_VIEW', { path, previousRoute, search, marketing })`, folding in UTM/marketing attribution from `utils/marketingAttribution.js`.
- **`ButtonTracker.jsx`** — a single capture-phase `document` click listener (event delegation). It resolves the nearest `button, [role="button"], .MuiButton-root, .btn, input[type=button|submit]`, skips disabled / `data-no-track="true"`, derives a stable identifier (id > data-id > aria-label > data-label > name > text > meaningful class), and emits `record_activity('BUTTON_CLICK', {...})`.
- **`TrackedButton.jsx`** — an explicit per-button wrapper for intentional event tagging.

Net effect: **three analytics pipelines** run — PostHog (product analytics + auto-capture), Firebase Analytics (`getAnalytics(app)` in `utils/firebase.js`), and the ClickHouse activity log via `record_activity`. Overlap is intentional but worth knowing.

### Firebase Cloud Messaging (web push)

- **`src/utils/firebase.js`** — initializes Firebase from `VITE_FIREBASE_*` env, calls `getAnalytics(app)`, and (only when `serviceWorker` exists) `getMessaging(app)` + a foreground `onMessage` handler that shows a native `new Notification(...)` (FCM suppresses foreground pushes, so this re-surfaces them). `requestNotificationPermission()` asks permission, waits for `navigator.serviceWorker.ready`, and returns the FCM token via `getToken({ vapidKey: VITE_FIREBASE_VAPID_KEY, serviceWorkerRegistration })`.
- **`public/firebase-messaging-sw.js`** — the background push handler (`onBackgroundMessage` → `self.registration.showNotification`), imported via `importScripts(...firebase 10.7.1 compat...)`. It ships with **placeholders** `__FIREBASE_API_KEY__`, `__FIREBASE_AUTH_DOMAIN__`, `__FIREBASE_PROJECT_ID__`, `__FIREBASE_MESSAGING_SENDER_ID__`, `__FIREBASE_APP_ID__` (note: `storageBucket`/`measurementId` are **not** injected into the SW).
- **The custom Vite plugin `replace-env-in-service-worker`** substitutes those placeholders: in **dev** via a `configureServer` middleware that templates `/firebase-messaging-sw.js` per request from `public/`; in **prod** via `closeBundle()` which reads `public/firebase-messaging-sw.js`, replaces, and writes `dist/firebase-messaging-sw.js`. Without this plugin the SW would ship literal `__FIREBASE_*__` strings and fail to init.

### Session-conflict flow (cross-cutting)

`actions/auth.js` registers a single global `axios.interceptors.response` handler: on a `401` whose `msg === 'Session invalidated or logged in elsewhere'`, it clears `token`/`is_admin`/`ownerStatus` from `localStorage` and dispatches a `window` `session-popup` event; `App.jsx` listens and shows `<SessionPopup/>`.

---

## 8. Styling

Layered system: **TailwindCSS 3 + DaisyUI** for utilities/components, **MUI 5** for the React component layer, plus hand-written CSS in `src/index.css`.

### Tailwind / DaisyUI (`tailwind.config.cjs`)

- `content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}']`.
- **`darkMode: 'selector'`** — dark styles apply under a `.dark` class on the root. This is toggled imperatively: `document.documentElement.classList.toggle('dark', isDark)` in `layout/Navbar.jsx`, and `.add/.remove('dark')` in `admin/AdminDashboard.jsx`. There is **no Redux dark-mode slice** (`actions/theme.js` is effectively empty); dark mode is DOM-class-driven, read back where needed via `document.documentElement.classList.contains('dark')` (e.g. in `MCQTestSeries.jsx`).
- `theme.extend.fontFamily.manrope = ['Manrope', 'sans-serif']`.
- Plugins: `daisyui` + `@tailwindcss/typography`.
- **DaisyUI theme (`mytheme`) primary is `#033A84` (blue), secondary `#ffbd59` (amber)** — success `#619D1C`, error `#C75146`, etc.

> **Brand-color gotcha:** The workspace brand palette is dark teal **`#1F4445`** / **`#2F6767`**, but the **DaisyUI theme primary is a blue (`#033A84`)**, not the brand teal. The brand teal appears only in hand-written CSS (`index.css` scrollbars `#2F6767`, focus box-shadow `rgba(31,68,69,…)`, `.brand-scroll`, `.table-scroll`) and in per-component Tailwind classes — **not** in the DaisyUI token set. Don't assume `btn-primary` renders the brand teal; it renders `#033A84`.

### Fonts

- **League Spartan (primary)** — loaded via a local `@font-face` in `index.css` from `assets/League_Spartan/LeagueSpartan-VariableFont_wght.ttf`, and set as the global `body/html` font stack. A `.force-font *` utility applies it with `!important` where needed.
- **Manrope (secondary)** — loaded from Google Fonts in `index.html` (`family=Manrope:wght@400;600;700`) and exposed as the Tailwind `font-manrope` utility.
- Page background: `#F6F7FB` light, `#111827` (gray-900) dark.

### `src/index.css` highlights

Tailwind `@layer` directives; DaisyUI overrides (`.btn { @apply rounded }`, `.alert-*`, `.input-custom` with dark variants); rich-text table styling for `.tiptap/.prose/.rich-html/.rendered-html/.content-html` (light + dark); a family of keyframe animations (`jump`, `shimmer`, `shake`, `slideIn`, `pulse`, `checkmark`) with matching `.animate-*` classes used by forms; custom scrollbar themes (`.scrollbar`, `.brand-scroll`, `.table-scroll`, `.no-scrollbar`); and a base-layer reset restoring `disc`/`decimal` list markers for raw HTML lists that lack Tailwind `list-*` classes (important for rendered note content).

---

## 9. API layer

### Base URLs

Resolved in `ecosystem.config.js` (repo root) and re-exported through `src/actions/types.js`. Five endpoints:

| Export | Dev | Prod |
|---|---|---|
| `address` (main backend) | `http://localhost:8080/api/` | `https://aws.spacedrevision.com/api/` |
| `aiAddress` (AI content service) | `http://localhost:8000` (config) | `https://ai.spacedrevision.com` |
| `chatAddress` (Socket.IO chat) | `http://localhost:9000` | `https://chat.spacedrevision.com` |
| `answerWritingAddress` (analytics/eval) | `http://localhost:5000/api/` | `https://answerwriting.spacedrevision.in/api/` |
| `transcoderSocketAddress` (HLS logs) | `http://localhost:9000` | `https://logs.transcoder.spacedrevision.in` |

`uploadAddress` is hard-pinned to the live host regardless of env.

### Auth header — no central axios instance

**There is no shared axios client.** Every component/action imports raw `axios` and concatenates `${address}<endpoint>`. Auth is threaded via a global default header:

```js
// src/utils/setAuthToken.js
const setAuthToken = (token) =>
  token ? (axios.defaults.headers.common['x-auth-token'] = token)
        : delete axios.defaults.headers.common['x-auth-token'];
```

Called on module load in `App.jsx` (from `localStorage.token`) and again on login success. So **`x-auth-token` rides on every request automatically** — components don't set it manually (the repo convention is to rely on this, not re-add the header). A handful of admin components redundantly pass an explicit `{ headers: { 'x-auth-token': localStorage.token } }` — harmless but not the pattern to copy.

### Interceptors

Exactly one, registered in `actions/auth.js`: the response interceptor for the session-invalidated 401 (→ `session-popup` event, §7). There is **no** global request interceptor and **no** uniform error envelope — each call site does its own `.catch`, most routing through the shared `handleError(...)` helper which converts backend `{ msg }` / `{ errors: [...] }` into `setAlert` toasts.

### Public (anonymous) token

`src/utils/publicToken.js` manages a separate unauthenticated session token for public/community browsing (cleared on login via `clearPublicToken()`); it has a manual mock at `src/utils/__mocks__/publicToken.js` for tests.

### Realtime

- **Chat:** `src/utils/socket.js` exports a module-level Socket.IO singleton `io(chatAddress, { path: '/chat', autoConnect: true })` — the **non-default `/chat` path is mandatory** (the chat server won't accept the default path). `ensureAuthed(id, username)` emits an `auth` event and resolves on `auth-success`, memoizing the authed user id so it authenticates once. Consumed by `chat/ChatRoom.jsx` and `chat/DirectChatRoom.jsx`. Reuse this singleton — don't open a second connection.
- **Transcoder logs:** `src/utils/transcoderSocket.js`, a separate Socket.IO connection to `transcoderSocketAddress`, used by the video/transcoder UI for HLS progress.

### Other integrations loaded outside React

`index.html` loads (via `<script>`): **Razorpay** checkout (`checkout.razorpay.com/v1/checkout.js`, invoked as `new window.Razorpay(...)` in payment components), **MathJax 3** (CDN, configured inline with `$...$` / `$$...$$` delimiters), **Google Tag Manager** (`GTM-KWPMWPTL`), and **Meta Pixel** (`973157488384012`, wrapped by `utils/metaPixel.js` helpers `trackLogin`/`trackSignup`). Extensive SEO/OpenGraph/Twitter + LD+JSON (`EducationalOrganization`, `FAQPage`) structured data also lives in `index.html`.

---

## 10. Notable patterns &amp; gotchas

1. **`ecosystem.config.js` is the API-URL config, not PM2** — and it's **gitignored**, so it's absent on CI (Vitest aliases it to `Test/stubs/ecosystemConfig.js`). If a fresh clone's dev server can't resolve `address`, this file is why.
2. **pnpm is the real package manager**, despite Volta pinning yarn/Node 18 and the workspace CLAUDE.md saying "Yarn." Install with pnpm; CI runs `--frozen-lockfile` on Node 20 and a stale `pnpm-lock.yaml` fails the build.
3. **`types.js` is dual-purpose** — action-type strings *and* the API base-URL/razorpay re-exports. Importing "just a constant" from it also pulls in the env config.
4. **No centralized axios / no redux-persist / no RTK slices.** Raw `axios` + `address` string concatenation; `x-auth-token` via `axios.defaults`; classic reducers + classic thunks; manual `localStorage` persistence. Follow the existing style rather than introducing `createSlice`/`apiClient`/`persist` piecemeal.
5. **Two Redux slices are renamed on import** — consume `state.couponReducer` and `state.unreadReducer` (files `coupons.js` / `unread.js`).
6. **`PrivateRoute` checks `localStorage.token` before Redux auth state** — anonymous visitors are redirected to `/login?redirect=...` immediately (and community-course deep links get special handling), specifically to avoid firing 401-generating private API calls.
7. **Chrome-suppression (`/admin`, `/ai/`, `/download`) is computed once from `window.location` at mount**, not reactively — safe only because those areas render disjoint trees.
8. **DaisyUI `primary` is blue (`#033A84`), not the brand teal.** Brand teal `#1F4445`/`#2F6767` lives only in `index.css` and per-component classes. Verify color intent against the actual token.
9. **Dark mode is DOM-class-driven** (`.dark` on `documentElement`, toggled in Navbar/AdminDashboard), not a Redux slice; `actions/theme.js` is essentially empty. Read state with `classList.contains('dark')`.
10. **The Firebase service worker is templated by a custom Vite plugin** — placeholders `__FIREBASE_*__` are replaced in dev (per-request middleware) and prod (`closeBundle`). Editing the SW means editing `public/firebase-messaging-sw.js` (source), not `dist/`. Only 5 of 7 Firebase fields are injected into the SW.
11. **Two push code paths for the same payload** — foreground pushes go through `utils/firebase.js` `onMessage` → `new Notification(...)`; background pushes go through the SW `onBackgroundMessage` → `self.registration.showNotification(...)`.
12. **`TabStateProvider` is not global** — it wraps only `suscribe/Suscribe.jsx`; `useTabState()` throws elsewhere. State lives in `sessionStorage` (`tabStates`).
13. **Three analytics pipelines run concurrently** (PostHog, Firebase Analytics, ClickHouse activity log). `ButtonTracker` auto-instruments *every* click via document-level delegation — opt a button out with `data-no-track="true"`.
14. **Husky pre-commit runs the full Vitest suite** and CI enforces per-glob coverage floors — a new util/reducer/action without tests can drop coverage below the gate and block merge. Tests mirror `src/` under `Test/`, use `renderWithProviders`/`renderWithRoutes` (`Test/test-utils.jsx`) and the shared `__mocks__/axios.js` manual mock (`vi.mock('axios')`).
15. **StrictMode double-mount is handled explicitly** — `loadUser()` is `useRef`-guarded so it doesn't dispatch twice in dev.
16. **Videos are topic-scoped, not heading-scoped** (per repo `CLAUDE.md`): link/list via `video|youtube/topic/:topic_id`; `video/all/...` responses are bucketed by topic with no `heading_id`. Don't reintroduce `video/heading/:id` routes.

### Key file reference

| Concern | Path |
|---|---|
| Entry / providers | `src/main.jsx` |
| App shell, routes, trackers, FCM init | `src/App.jsx` |
| Store | `src/store.js` |
| Root reducer (38 slices) | `src/reducers/index.js` |
| Action types + API-URL re-export | `src/actions/types.js` |
| API base URLs (gitignored) | `ecosystem.config.js` (root) + `Test/stubs/ecosystemConfig.js` |
| Auth actions + axios interceptor | `src/actions/auth.js` |
| Auth header helper | `src/utils/setAuthToken.js` |
| Route guards | `src/components/routing/{PrivateRoute,AdminRoute}.jsx`, `src/components/auth/CheckCourseAccess.jsx` |
| Tab-state context | `src/context/TabStateContext.jsx` |
| MathJax hook | `src/hooks/useMathJax.js` |
| Trackers | `src/components/tracking/{RouteTracker,ButtonTracker,TrackedButton}.jsx` |
| FCM (client + SW) | `src/utils/firebase.js`, `public/firebase-messaging-sw.js` |
| Chat / transcoder sockets | `src/utils/socket.js`, `src/utils/transcoderSocket.js` |
| Global styles / fonts | `src/index.css`, `tailwind.config.cjs` |
| Build config + SW plugin + Vitest | `vite.config.js` |
| CI / hooks | `.github/workflows/ci.yml`, `.husky/pre-commit` |
| Authoritative in-tree docs | `docs/ARCHITECTURE.md`, `docs/COMPONENT_INVENTORY.md`, `Test/UNIT_TESTING_GUIDE.md` |
