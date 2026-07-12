---
Date: 2026-07-12
tags:
  - spaced-revision
  - react-native
  - mobile
  - documentation
Status: living
---

> Part of [[00 - Repo Documentation Overview]]

# SpacedRevision — React Native Mobile App (Architecture &amp; Internals)

> Deep-dive technical reference for the `react-native-app` repo: the iOS + Android mobile client for the Spaced Revision spaced-repetition learning platform. React Native 0.79.5, React 19, TypeScript, Redux Toolkit, React Navigation v7, WatermelonDB offline layer, and two custom native HLS-video modules.

> **Doc accuracy note:** Several conventions have moved on from older summaries. Where the current code differs from a widely-quoted description, this doc records the **actual current state** and flags the drift explicitly. The three biggest ones: (1) the package manager is now **pnpm**, not Yarn/npm; (2) the source tree is **feature-first** (`src/features/*`), not `store → services → data/repositories → ui`; (3) the Redux-Persist whitelist is now tiny (`auth`, `theme`, `user`, `onboardingSelections`) because offline data moved into WatermelonDB (SQLite).

---

## 1. Overview

**What it is.** A cross-platform (iOS + Android) React Native app. App id `com.spacedrevision`, display name `SpacedRevision`, marketing version `2.4.4` (`package.json` + `app.json`). Bundle/registration name comes from `app.json` (`name: "SpacedRevision"`) and is wired in `index.js` via `AppRegistry.registerComponent(appName, () => App)`.

**Core stack**
- **React Native `0.79.5`, React `19.0.0`** (`package.json`). **New Architecture is OFF** (see §7) — Hermes is ON.
- **TypeScript `~5.8.3`**, config in `tsconfig.json` extends `@react-native/typescript-config`, adds `"jsx": "react-native"` and `"experimentalDecorators": true` (needed for WatermelonDB `@field` decorators).
- **Redux Toolkit `2.8.1`** + `react-redux 9` + `redux-persist 6`.
- **React Navigation v7** (`native`, `native-stack`, `bottom-tabs`, `stack`).
- **Jest 29** + `@testing-library/react-native` + `ts-jest`.
- **Reanimated 3**, Gesture Handler, Screens, Safe Area Context.

**Run commands** (`package.json` `scripts`):

| Command | Action |
|---|---|
| `npm start` | `npx react-native start --reset-cache` — Metro with cache reset |
| `npm run ios` | `react-native run-ios` |
| `npm run and` | `react-native run-android` (note: alias is `and`, not `android`) |
| `npm test` | `jest` |
| `npm run test:version` | Runs only `__tests__/appVersion.test.ts` |
| `npm run lint` | `eslint .` |
| `npm run clean:android` | `cd android && ./gradlew clean` |
| `npm run clean:ios` | `cd ios && pod deintegrate && pod install` |
| `npm run clean` | Both clean steps |
| `npm run reset` | `clean` then `start` |
| `npm run version:status \| :patch \| :minor \| :major \| :set` | Drives `scripts/version-manager.js` (keeps iOS/Android/`package.json` versions in sync) |
| `npm run install:debug` | Bundles JS + `gradlew assembleDebug` + `adb install` the arm64 debug APK |
| `postinstall` | `node scripts/prepare-nitro-modules-headers.js` — pre-creates `react-native-nitro-modules` Android headers so CMake doesn't fail building `react-native-iap` |

**Package manager: pnpm.** `package.json` declares `"packageManager": "pnpm@11.10.0"` and `"engines": { "node": ">=22.5" }`. CI (`.github/workflows/*`) uses `pnpm install --frozen-lockfile`. There is a `pnpm-workspace.yaml` (only an `allowBuilds` allowlist for native postinstalls) and a `pnpm-lock.yaml`. `pnpm.overrides` in `package.json` pins many transitive deps for security (e.g. `axios ^1.16.0`, `tar ^7.5.3`, `ws` version floors). **Both a `package-lock.json` and `pnpm-lock.yaml` exist** — pnpm is authoritative; the npm lockfile is legacy. Many `scripts` still shell out to `npm run …` internally (e.g. `clean` → `npm run clean:android && npm run clean:ios`), so a working `npm` is still handy locally.

**`legacy-peer-deps`.** `.npmrc` contains a single line: `legacy-peer-deps=true`. This is needed because React 19 / RN 0.79 dependency peer ranges are strict, and several libraries (chart-kit, mathjax, etc.) declare older peers; without it, `npm install` would hard-fail on peer conflicts.

**Babel strips console logs in production.** `babel.config.js`:
```js
...(process.env.NODE_ENV === 'production' || process.env.BABEL_ENV === 'production'
  ? [['transform-remove-console', { exclude: ['error', 'warn'] }]]
  : []),
'react-native-reanimated/plugin', // must be last
```
So `console.log/info/debug` are removed in release builds, but `console.error`/`console.warn` are preserved for crash reporting. The config also enables legacy decorators + loose class-properties (for WatermelonDB), and has a per-file `overrides` block forcing `['typescript','decorators-legacy']` parser plugins on files under `data/local/watermelon/`. The app-wide `logger` (`src/shared/utils/logger.ts`) is the preferred logging channel over raw `console`.

---

## 2. Clean-architecture layers (the real folder pattern)

The canonical structure is documented in-repo at `docs/dev_documents/FolderPattern.md` and enforced by `CLAUDE.md`. It is **feature-first**, not the older `store → services → data/repositories → ui` layering. Top-level `src/`:

```
src/
├── app/          # Thin bootstrap: App.tsx, Providers.tsx, screens/ (Force/OptionalUpdate, HLSTest)
├── navigation/   # All navigators + route types (RootNavigator, MainTabNavigator, …)
├── features/     # One folder per product feature (see below)
├── data/         # Shared data layer: api/ (Axios), local/ (WatermelonDB), remote/ (repos)
├── store/        # Global Redux wiring: store.ts, rootReducer.ts, hooks.ts, migrations.ts, slices/, selectors/
├── shared/       # Feature-agnostic reuse: components/, hooks/, context/, providers/, utils/, constants/, theme/, types/
├── config/       # googleSigninConfig.ts (env/service config)
├── analytics/    # posthog.ts, notificationAnalytics.ts
├── assets/       # fonts, images, lottie
├── services/     # (legacy holdover — see note)
└── hooks/        # (legacy holdover — useVideoQualityMonitor.ts only)
```

**Each `features/<feature>/` is a vertical slice** owning its own `screens/`, `components/`, `hooks/`, `store/` (slice), `services/` (repository), and `__test__/`. The 16 features are: `ai-chat`, `analytics`, `answer-writing`, `auth`, `chat`, `courses`, `dashboard`, `decks`, `friends`, `iap`, `notifications`, `onboarding`, `profile`, `study`, `test-series`.

The four conceptual layers and how they connect (using auth as the worked example):

1. **UI layer — `features/*/screens` + `features/*/components`.** Screens are thin: per `CLAUDE.md`, "screen files only render." All logic lives in a **ViewModel hook** (`use<Name>ViewModel`) — e.g. `features/auth/hooks/useLoginViewModel.ts`, `features/dashboard/hooks/useDashboardViewModel.ts`. This is the enforced convention.

2. **State layer — `features/*/store` (Redux slices) + global `src/store`.** Slices use `createSlice` + `createAsyncThunk`. Example: `features/auth/store/authSlice.ts` defines `register`, `loginWithEmail`, `loginWithGoogle`, `loginWithApple` thunks. Hooks/components dispatch these via typed `useAppDispatch` / `useAppSelector` (`src/store/hooks.ts`).

3. **Service / repository layer — `features/*/services/*Repository.ts` (remote) and `src/data/remote/`.** These wrap the Axios client and own the HTTP surface. Example: `features/courses/services/coursesRepository.ts` exports `getUserOwnedCourses()` (`GET /course`), `getAllCourse()` (`GET /course/public`), each tolerating multiple response shapes and re-throwing after logging. Thunks call repositories; repositories call `ApiService`.

4. **Data-access layer — `src/data`.** Three sub-layers:
   - `data/api/ApiService.ts` — the single Axios instance + interceptors (§8).
   - `data/local/` — **WatermelonDB / SQLite offline store** (18 `models/`, 15 `queries/` repositories, `schema.ts`, `sync/`). This is the offline-first cache that replaced persisting course data in Redux.
   - `data/remote/` — cross-feature remote repositories not owned by a single feature (`addressRepository.ts`, `feedBackRepository.ts`, `userPreferencesRepository.ts`).

**End-to-end flow (sign-in):** `LoginScreen` → `useLoginViewModel` / `useAuth` (`features/auth/hooks/useAuth.ts`) → `dispatch(loginWithEmail())` thunk (`authSlice.ts`) → `authService.signInWithEmail()` → `ApiService` → backend. On success the thunk fires PostHog `identifyAndLinkAnon` + `user_login_succeeded` **inside the thunk** (deliberate: fires regardless of caller — hook vs direct dispatch), stores `token`/`user` in the `auth` slice, then `useAuth` chains `loadUser()` and `checkAndAcceptTerms()`.

> **Legacy holdovers to know about.** `src/services/` (e.g. `videoService.ts`, `appVersion.service.ts`, `doubtService.ts`, `pdfService.ts`) and `src/hooks/` (`useVideoQualityMonitor.ts`) predate the feature-first migration and still exist at the top level. New code should follow `FolderPattern.md`; treat these as not-yet-migrated. `CLAUDE.md` explicitly says: "Do not invent new top-level folders or put feature-specific code in `shared/`."

---

## 3. State management

**Store config — `src/store/store.ts`.** `configureStore` wraps `rootReducer` in `persistReducer`. `serializableCheck` is disabled (non-serializable payloads like Dates/class instances flow through some slices). Exports `store`, `persistor`, and the typed `RootState` / `AppDispatch` / `AppStore`.

**Typed hooks — `src/store/hooks.ts`.** Uses RTK's `.withTypes` helpers:
```ts
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppStore    = useStore.withTypes<AppStore>();
```

**Root reducer — `src/store/rootReducer.ts`.** `combineReducers` over **31 slices** pulled from feature folders and `store/slices/` — e.g. `auth`, `user`, `subscribedCourses`, `allcourse`, `courseSubjects`, `courseStudyTopics`, `trail` (IAP trial), `mcq`, `cards`, `testSeris` (sic), `chat`, `video`, `analytics`, `theme`, `answerWriting`, `notification`, `ai`, `aiChat`, `pdfSignedUrl`, `profileSetup`, `revisionDues`, `friends`, `network`, `decks`, `courseTags`, `educators`, `onboardingSelections`, `forYou`, `search`, `visitor`. A **global logout reset** wraps `combineReducers`:
```ts
const rootReducer = (state, action) =>
  action?.type === logout.type ? appReducer(undefined, action) : appReducer(state, action);
```
So dispatching `logout` wipes the entire store to initial state in one shot (individual slices also `addCase(logout, () => initialState)` for belt-and-suspenders, e.g. `networkSlice.ts`).

**Redux Persist + AsyncStorage — `store.ts`.**
```ts
const PERSIST_VERSION = 3;
const persistConfig = {
  key: 'root', version: 3, storage: AsyncStorage,
  whitelist: ['auth', 'theme', 'user', 'onboardingSelections'],
  migrate: async (state) => {
    const normalized = normalizePersistedRootState(state);
    delete normalized.allcourse;          // never rehydrate legacy cached catalogs
    delete normalized.subscribedCourses;
    return normalized;
  },
};
```
> **Important drift from older docs.** The whitelist is now just **`auth`, `theme`, `user`, `onboardingSelections`**. Course catalogs (`allcourse`, `subscribedCourses`) are **explicitly deleted** during migration and are **no longer persisted in Redux** — they live in WatermelonDB now (offline-first). `preferences` is not a slice; user preferences persist through `user`/`onboardingSelections` + a remote repository. Bump `PERSIST_VERSION` whenever the persisted shape changes so `migrate` re-runs on existing installs.

**Migration hygiene — `src/store/migrations.ts`.** `normalizePersistedRootState` defensively coerces malformed rehydrated slices (e.g. a non-array `courses`) into safe shapes to prevent white-screen crashes after a cold start from a corrupt persist blob.

**Slice conventions.** Async work uses `createAsyncThunk` with `rejectWithValue`; `extraReducers` handle `pending/fulfilled/rejected`. Per `CLAUDE.md`, optimistic updates are dispatched immediately and reconciled after the API resolves. Simple example — `store/slices/networkSlice.ts` (`setNetworkStatus`, resets on `logout`). Rich example — `authSlice.ts` (four OAuth/email thunks, `pendingPreferencesOnboarding`/`firstTimeOnboardingChecked` onboarding flags, analytics fired inside thunks).

---

## 4. Navigation

React Navigation v7. The tree is assembled across `src/navigation/`.

**Root — `navigation/RootNavigator.tsx`.** A `createNativeStackNavigator` (`RootStackParamList`) that swaps between two screens based on `state.auth.isLoggedIn`:
```tsx
{!authState?.isLoggedIn
  ? <RootStack.Screen name="Auth" component={AuthNavigator} />
  : <RootStack.Screen name="App"  component={AppStackNavigator} />}
```
This file is the app's nerve center (~1170 lines). Beyond routing it owns: the **`NavigationContainer`** (with theme derived from `useTheme()`), **splash lifecycle** (a `SplashScreen` overlay shown until dashboard data is prefetched, with a 9s safety-net fallback), **push-notification setup** (Firebase Messaging + Notifee: foreground display, token registration/refresh, `onNotificationOpenedApp`, `getInitialNotification` cold-start routing via `navigateFromNotification`), **iOS badge-count management** (inactivity badges), **force/optional update gating** (`ForceUpdateScreen` short-circuits the whole render tree when a `FORCE` update is required), **IAP init/reconnect**, a **`FloatingFeedbackButton`** shown on Home/owned-course screens, and **PostHog screen tracking** in `onStateChange` (emits `posthog.screen(routeName)` + per-screen `screen_time`).

**App stack (logged-in root) — `AppStackNavigator` inside `RootNavigator.tsx`.** `createNativeStackNavigator` (`AppStackParamList`), `headerShown:false`, `animation:'slide_from_right'`. Screens: `Preferences`, `GettingReady`, **`Main`** (the tab navigator), `ProfileSetup`, `HLSTest`, `NotificationDetail`, **`StreakScreen`** (`StreakScreen`/"Activity Streak" — a stack screen, **not** a tab), `TermsOfUse`, `PrivacyPolicy`, `Feedback` (slide-from-bottom). Initial route is `Preferences` when `auth.pendingPreferencesOnboarding` is set (post-signup wizard), else `Main`.

**Auth stack — `navigation/AuthNavigator.tsx`.** Login, Signup, OTP verification, forgot/reset password, profile setup (the screens backed by the `use*ViewModel` hooks in `features/auth/hooks/`).

**Main tabs — `navigation/MainTabNavigator.tsx`.** `createBottomTabNavigator` (`MainTabParamList`) with a **custom tab bar** (`shared/components/CustomBottomTabBar.tsx`) and **6 tabs**:

| Tab (route) | Component | Loading |
|---|---|---|
| `Home` (Dashboard) | `DashboardScreen` | **Eager** import |
| `MyCourses` | `MyCoursesNavigator` | nested stack |
| `Courses` | `CoursesNavigator` | nested stack |
| `Decks` | `DecksNavigator` | nested stack |
| `Chat` | `ChatScreen` | eager |
| `Profile` | `ProfileStackNavigator` | stack; **child screens `React.lazy`-loaded** |

> **Drift note:** the live tab set is `Home / MyCourses / Courses / Decks / Chat / Profile`. "Streak" is a pushed stack screen, not a tab.

Key behaviors:
- **Dashboard is eager, profile children are lazy.** `MainTabNavigator.tsx` explicitly comments: "Eager imports — lazy + Suspense caused occasional blank screens and module-resolution errors on Android." So `Dashboard`, `Chat`, `Profile` roots are eager; the profile detail screens (`AnalysisReport`, `BasicProfile`, `EditProfile`, `AccountSettings`, `RateApp`, `TermsOfUse`, `PrivacyPolicy`) are `React.lazy` behind `<Suspense fallback={null}>`.
- Tabs use `freezeOnBlur: true`, `lazy: true`, `tabBarHideOnKeyboard: true`, and a **transparent absolute-positioned bar** (`height` 88 iOS / 70 Android).
- **`HIDDEN_TAB_ROUTES`** (`TestInstruction`, `TestSeries`, `Study`, `ReviseAll`, `InstructionScreen`, `DeckStudy`, `PublicDeckPreview`) — the custom tab bar returns `null` when the focused nested route is one of these, giving full-screen study/test flows.
- **Tab-press `listeners`** reset nested stacks to their root (Chat → `RoomsList`, Profile → `ProfileHome`) and there's a safety net that force-initializes the Courses stack to `CoursesList` if it's never been visited (prevents blank tab).
- `useUnreadCounts(30000)` is mounted once here as the single polling source for chat unread badges across the whole tab navigator.

Route params/names are centralized in `navigation/types.ts` (`RootStackParamList`, `AppStackParamList`, `MainTabParamList`, `ProfileStackParamList`, and a `ScreenRoutes` enum). Nested navigation to a deep screen uses the `{ screen, params }` object form (see the `FeedbackScreen` deep-link reset logic in `RootNavigator.tsx`).

---

## 5. Provider hierarchy

Assembled in `src/app/Providers.tsx` and mounted by `src/app/App.tsx` (`<Providers><RootNavigator/></Providers>`). The **actual nesting (outermost → innermost)** is deeper than older summaries and ordered deliberately:

```
GlobalErrorBoundary
└─ GestureHandlerRootView
   └─ SafeAreaProvider
      └─ Redux <Provider store>
         └─ NetworkProvider
            └─ PersistGate (loading = <SplashScreen/>)
               └─ PostHogProvider
                  └─ PostHogSurveyProvider
                     └─ ThemeProvider
                        └─ KeyboardVisibilityProvider
                           └─ AlertProvider
                              └─ BottomSheetModalProvider
                                 └─ ChatSocketProvider
                                    └─ PostHogIdentify + {children}
```

Provider responsibilities:
- **`GlobalErrorBoundary`** (`shared/components/GlobalErrorBoundary.tsx`) — outermost catch for render crashes.
- **`GestureHandlerRootView`** — required root for `react-native-gesture-handler`.
- **`SafeAreaProvider`** — safe-area insets.
- **Redux `Provider`** — store access below it.
- **`NetworkProvider`** (`shared/providers/NetworkProvider.tsx`) — subscribes to `@react-native-community/netinfo` and dispatches `setNetworkStatus` into the `network` slice on every connectivity change. Placed **above** `PersistGate` so connectivity is known even during rehydration.
- **`PersistGate`** — blocks children behind `<SplashScreen/>` until redux-persist rehydrates.
- **`PostHogProvider` / `PostHogSurveyProvider`** — analytics + in-app surveys. Deliberately placed **below `PersistGate`** (comment in the file): they did heavy mount work (flush timers, AsyncStorage reads, identify probe) that used to stretch the visible splash; moving them inside the gate lets the splash paint immediately and PostHog boot in parallel. `autocapture={{ captureScreens: false }}` because screen tracking is done manually in `RootNavigator`.
- **`ThemeProvider`** (`shared/context/ThemeContext.tsx`) — bridges the Redux `theme` slice to context (`isDark`, `toggleTheme`, `setTheme`, `followSystem`). Listens to `Appearance.addChangeListener` and dispatches `updateSystemTheme`. Consumed via `useTheme()`.
- **`KeyboardVisibilityProvider`** (`shared/context/KeyboardVisibilityContext.tsx`) — simple `isKeyboardVisible` boolean context; consumed via `useKeyboardVisibility()`.
- **`AlertProvider`** (`shared/providers/AlertProvider.tsx`) — app-wide custom alert modal. Exposes `showAlert/showSuccess/showError/showWarning/showInfo/showConfirmation` via `useAlert()`, backed by `useCustomAlert` + `<CustomAlert>`.
- **`BottomSheetModalProvider`** (`@gorhom/bottom-sheet`) — enables bottom-sheet modals anywhere below it.
- **`ChatSocketProvider`** (`shared/providers/ChatSocketProvider.tsx`) — global Socket.IO connection (§7).
- **`PostHogIdentify`** — a headless component that identifies the user in PostHog (prefers normalized email as `distinct_id` to unify web + mobile identity) and `reset()`s on logout.

> **Drift note:** older shorthand ("ThemeProvider > ChatSocketProvider > AlertProvider > KeyboardVisibilityProvider") reverses several of these. The code order above is authoritative — in particular `ChatSocketProvider` is the **innermost**, and `KeyboardVisibilityProvider`/`AlertProvider` sit **above** it.

Side note: `App.tsx` fires `void initLocalDatabase()` as a **module-load side effect** (not in `useEffect`) so SQLite open + migrations run in parallel with bundle parse and rehydration, and `initSounds()` in a mount effect to pre-decode SFX.

---

## 6. Custom hooks

Hooks live in three places: `src/shared/hooks/` (feature-agnostic), `src/features/*/hooks/` (feature ViewModels + feature-specific), and the legacy `src/hooks/`.

**Shared hooks (`src/shared/hooks/`)**
- **`useNetworkStatus.ts`** — thin selector returning the `network` slice (`{ initialized, isConnected, isInternetReachable, type }`). The actual listening happens in `NetworkProvider`; this hook just reads it.
- **`useIsOffline.ts`** — derived boolean convenience over the network slice.
- **`useSound.ts`** — maps an SRS response rating (`0–3`: Forgot/Difficult/Okay/Perfect) to a named sound (`incorrect/difficult/okay/perfect`) and plays it via the central `soundService`. Returns `{ playSound(responseType, volume) }`. No per-component audio lifecycle (sounds pre-decoded at app start).
- **`useCustomAlert.ts`** — state machine behind `AlertProvider`.
- **`useAppTheme.ts`** — theme tokens/colors for styling.
- **`useSimulatedDownloadPercent.ts`** — fake progress animation for offline-download UX.

**Auth (`src/features/auth/hooks/`)**
- **`useAuth.ts`** — the primary auth facade. Exposes state (`isLoggedIn`, `token`, `user`, `loading`, `error`) and actions `signInWithEmail`, `signInWithGoogle`, `signInWithApple`, `signUp`, `signOut`, `clearAuthError`. `signOut` clears local DB + search/recent-course caches + baselines, dispatches `logout`, and fires `posthog.capture('user_logout')`. OAuth sign-ins additionally call `trackOAuthAuthenticated` which fires synthetic `onboarding_started/completed` events for brand-new OAuth users (who skip the Preferences wizard). Companion **`useLoginViewModel`, `useSignupViewModel`, `useOTPVerificationViewModel`, `useForgotPasswordViewModel`, `useResetPasswordViewModel`, `useProfileSetupViewModel`** are the screen ViewModels.

**Courses / IAP (`src/features/courses/explore/hooks/`)**
- **`useCoursePrice.ts`** — resolves a single course's **localized store price** (App Store / Google Play) from a `courseId`. Maps `courseId → productId` (`getProductIdFromCourseId`), calls `getProductDetails`, and — importantly — re-fetches when IAP becomes ready via `useSyncExternalStore(subscribeIapReady, getIapReadyTick)`. Treats "product not available / IAP not initialized / billing unavailable" as **non-errors** (expected during App Review / sandbox). No-ops on non-mobile platforms. Sibling `useCoursePrices` batch-fetches for lists (batch size 5, 100ms between batches, warning suppression for expected sandbox noise).

**Chat (`src/features/chat/hooks/`)** — `useChatSocket` (per-screen socket usage), `useUnreadCounts` (polling badge counts).

**Video (`src/hooks/`, legacy)**
- **`useVideoQualityMonitor.ts`** — subscribes to a singleton `videoQualityMonitor` and exposes live `{ currentQuality, currentMetrics, qualityHistory, averageBitrate, averageFrameRate, totalDroppedFrames }` plus `startMonitoring/stopMonitoring/updateMetrics/getQualityHistory/getAverageMetrics/resetMonitoring`. Configurable bitrate thresholds (low 500k / medium 1.5M / high 3M), 5s averaging window over the last 30s. Guards all `setState` with an `isMountedRef`.

**Other feature ViewModels** — `useDashboardViewModel`, `useMyCourseViewModel`, `useQuestionBankViewModel`, `useStudyViewModel`, `useReviseAllViewModel`, `useDeckStudyViewModel`, `useStreakViewModel`, `useWatermelonCourseObservers` (subscribes React state to WatermelonDB queries), `useIsCourseOfflineReady`, `useOfflineCourseContentIds`, etc. — one ViewModel per screen, per the enforced pattern.

---

## 7. Native modules &amp; platform

### New Architecture is disabled
Despite naming, the app runs on the **old bridge (Paper)**, not Fabric/TurboModules:
- `ios/Podfile`: `ENV['RCT_NEW_ARCH_ENABLED'] = '0'`, and `use_react_native!(… :new_arch_enabled => false, :fabric_enabled => false)`. The comment explains WatermelonDB needs the legacy bridge (`NativeModules.WMDatabaseBridge` is undefined under New Arch on RN 0.79+).
- `android/gradle.properties`: `newArchEnabled=false`, `hermesEnabled=true`.

So the two "RTN" modules below are **Paper native views / view-managers registered via `requireNativeComponent`**, not codegen'd Fabric components — the `rtn-*` naming reflects the intended future migration.

### `rtn-my-player` — Android HLS player (ExoPlayer)
Local package (`rtn-my-player/`). JS side is one file: `js/NativeRtnMyPlayer.ts` → `requireNativeComponent<{url?}>('RtnMyPlayer')`. Native side is Kotlin (`android/src/main/java/com/rtnmyplayer/`): `MyPlayer.kt` wraps **Media3 ExoPlayer** in a `LinearLayout` + `PlayerView`, ties into RN's `LifecycleEventListener` (pause/resume/release on host lifecycle), and implements HLS **quality selection** — `getAvailableQualities()` enumerates video track formats from `Tracks` (adds an "Auto" entry when >1 variant), and `setQuality("WIDTHxHEIGHT" | "auto")` drives the `DefaultTrackSelector` (`setMax/MinVideoSize` or `clearVideoSizeConstraints`). Registered in `MainApplication.kt` via `add(MyPlayerPackage())`. Podspec exists (`ios/**/*.{h,m,mm}`) but there's no iOS implementation shipped — this module is Android-focused.

### `rtn-my-video-picker` — cross-platform HLS player w/ quality UI
Local package (`rtn-my-video-picker/`) — despite the name it's an **HLS video player**, not a picker. Richer, cross-platform:
- **JS** (`js/`): `HLSVideoPlayer.tsx` (imperative ref API: `play/pause/stop`, `getAvailableQualities(): Promise<QualityOption[]>`, `setQuality(id): Promise<void>`), `EnhancedHLSVideoPlayer.tsx` (adds a quality button + AsyncStorage-persisted quality preference under `@hls_video_player_quality_preference`), `QualitySelector.tsx`, and `HLSVideoPlayerViewNativeComponent.tsx` → `requireNativeComponent<{source, autoPlay, controls, onLoad, onError}>('HLSVideoPlayerView')`. Barrel `index.ts` re-exports the public API + types.
- **iOS** (`ios/*.mm/.h`): `HLSVideoPlayerView` + `HLSVideoPlayerViewManager` on **AVKit/AVFoundation**; quality via `preferredPeakBitRate`, falling back to media selection groups.
- **Android** (`android/…kt`): `HLSVideoPlayerView` + `…Manager` + `…Package` on **ExoPlayer (Media3)** via track-selector constraints.
- Registered on Android via `add(HLSVideoPlayerPackage())` in `MainApplication.kt`; on iOS via `pod 'rtn-my-video-picker', :path => '../rtn-my-video-picker'` in the Podfile. Quality presets 240p→1440p + Auto (see `rtn-my-video-picker/README.md` / `IMPLEMENTATION_SUMMARY.md`).

There are also **JS-level video components** in `shared/components/` (`HLSPlayer.tsx`, `VideoComponent.tsx`, `VideoPlayerWithPlaylist.tsx`, `YouTubePlayer.tsx`) layered on top of `react-native-video` / `react-native-youtube-iframe` and the native modules. `src/app/screens/HLSTestScreen.tsx` is a dev harness (routed as `HLSTest`).

### `patches/` (patch-package + Podfile re-apply)
`patch-package` runs against `node_modules`; three patches are checked in:
- `lottie-react-native+6.7.2.patch` (16 KB) — library fix.
- `RCT-Folly+New.h.patch` and `fmt+base.h.patch` — these target **Pods** source (RCT-Folly / fmt), which `patch-package` can't reach, so the `Podfile` `post_install` **re-applies them** against `Pods/` after every `pod install` (idempotent `patch -N -r -`). Both are Xcode 26 / clang 21 compile fixes (folly `New.h` invoker indirection; fmt 11.0.2 `consteval` guard).

### iOS platform (`ios/`)
- `Podfile` (above) + `Podfile.lock` + `Pods/`. Firebase pods forced to `:modular_headers => true` (Swift/ObjC interop). WatermelonDB's `simdjson` linked via local podspec (`../node_modules/@nozbe/simdjson`). `post_install` also silences non-modular-include-in-framework warnings and adds aligned-new/delete flags for folly.
- `Gemfile` — pins CocoaPods (`>= 1.13`, excluding 1.15.0/1.15.1), plus Ruby 3.4+ stdlib gems (`tsort`, `bigdecimal`, `logger`, `mutex_m`, `cgi`, `nkf`, …) so bundler works on modern Ruby.
- App target `SpacedRevision`; also a `SpacedRevisionNotificationService` extension (rich push). `GoogleService-Info.plist` present. SFX assets (`.mp3`/`.caf`) live in `ios/`.

### Android platform (`android/`)
- `app/build.gradle` applies `com.android.application`, Kotlin, and `com.facebook.react`. Notably it hardcodes an **nvm node path** (`~/.nvm/versions/node/v22.19.0/bin/node`) with a PATH fallback for the RN bundling step — a known local-env gotcha.
- `MainApplication.kt` registers the two local packages manually alongside autolinked `PackageList`. Hermes + old arch per `gradle.properties`.
- CI builds a **release AAB** and publishes to Play Store internal track (§9).

### Chat socket integration (`ChatSocketProvider` + `chatSocketService`)
`shared/providers/ChatSocketProvider.tsx` connects **once** when a `user.id`/`username` becomes available, sets `chatSocketService.globalManaged = true` (so per-screen components don't attach duplicate listeners), normalizes every inbound room/direct message payload (a defensive `normMessage` that reconciles the many field-name variants and file/attachment shapes coming from the chat backend), and maintains unread counts in the `chat` slice. `chatSocketService.ts` targets `CHAT_SERVER` with Socket.IO **engine path `/chat`** (not the default) and tries two connection variants in order — **`path-only`** (`io(CHAT_SERVER, { path: '/chat' })`) then **`namespace`** (`io(`${CHAT_SERVER}/chat`, …)`) — whichever connects first wins. This mirrors the chat backend's non-default socket path requirement.

---

## 8. API / services layer

**Single Axios instance — `src/data/api/ApiService.ts`.**
```ts
const ApiService = axios.create({
  baseURL: API_BASE_URL, timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
});
```

**Base URLs / config — `src/shared/utils/constants.ts`** (git-ignored; `constants.example.ts` is the template). Keys: `API_BASE_URL` (main backend `/api`), `API_UPLOAD_BASE_URL` (profile/image upload), `CHAT_SERVER`, `ANSWER_SERVER` (answer-writing eval), `AI_SERVER`, the three Google OAuth client IDs, `APP_STORE_*` / `PLAY_STORE_*`, `APP_AUTH_ACCESS_KEY` (version-endpoint gate), and `POSTHOG_PROJECT_TOKEN`.
> **Gotcha:** the committed working copy in this checkout points several URLs at **local/ngrok dev endpoints** (`API_BASE_URL = http://localhost:8080/api`, `CHAT_SERVER = https://…ngrok-free.dev`) with the production URLs commented out. Because `constants.ts` is git-ignored, each dev keeps their own; **CI regenerates it** (§9). Never assume the values in a working tree are production.

**Interceptors (in order):**
1. **Offline guard** (request) — reads the `network` slice from the store; if the device is offline it rejects immediately with `new axios.Cancel('OFFLINE_REQUEST_BLOCKED')` so no request even leaves the app.
2. **Auth + headers** (request) — pulls `store.getState().auth.token` and sets `x-auth-token`. Strips `Content-Type` when the body is `FormData` (so RN sets the multipart boundary). For the app-version endpoint (`/app/version`) it additionally attaches `access-key` (= `APP_AUTH_ACCESS_KEY`) and `x-platform` (`ios`/`android`), which the backend middleware requires.
3. **Response** — passes offline cancellations through silently; classifies **network errors** (no `response`) vs **404** (PDF 404s stay silent — expected) vs **timeouts** vs **4xx client errors** (logged as `warn`) vs **5xx** (logged as `error`). Everything is re-rejected so callers still handle failures; the interceptor only controls log severity (deliberately avoiding `logger.error` for cases that would trigger RN's red LogBox).

**Auth token handling.** The token is the **source of truth in Redux** (`auth.token`), persisted via redux-persist, and injected per-request by the interceptor — there's no separate token store. Login thunks set it; `logout` clears it (and the global reducer reset wipes everything).

**Repository pattern.** Feature `services/*Repository.ts` files own endpoints and normalize response shapes. Example `getAllCourse()` in `features/courses/services/coursesRepository.ts` handles `Course[]`, `{courses}`, and `{data:{courses}}` shapes, fires image preloading, and enriches with IAP store prices — all try/caught with `logger.error` before re-throw (per `CLAUDE.md`: "Always try-catch API calls… never silently swallow"). Some services hit other backends directly using the non-`API_BASE_URL` constants (`ANSWER_SERVER`, `AI_SERVER`, `API_UPLOAD_BASE_URL`).

**Offline-first data.** Beyond HTTP, `src/data/local/` is a full **WatermelonDB/SQLite** cache (`database.ts` opens adapter `spacedrevision_wm` with 18 model classes; `queries/*LocalRepository.ts` are the read/write APIs; `sync/SyncManager.tsx` + `sync/sync.ts` reconcile local ↔ remote). This is why course catalogs left the Redux persist whitelist. `experimentalDecorators` + the Babel `data/local/watermelon` override exist for this layer.

---

## 9. Build &amp; CI

**GitHub Actions (`.github/workflows/`)** — the real CI:
- **`ci.yml`** (on PRs to `master`): pnpm + Node 22, `pnpm install --frozen-lockfile --ignore-scripts`, **stubs `constants.ts` from `constants.example.ts`** (real file is git-ignored), then runs **only affected tests**: `jest --changedSince=<baseSha> --ci --forceExit --passWithNoTests`. Comment explains `--onlyChanged` is intentionally omitted (conflicts with `--changedSince` and can hang).
- **`android-release.yml`** (on `v*` tags / manual): pnpm + Node 22 + JDK 17 + Gradle. **Generates `constants.ts` from GitHub Secrets** via an inline Node script that validates all 16 required keys are present, decodes the upload keystore from `ANDROID_KEYSTORE_BASE64`, runs `./gradlew bundleRelease` (passwords injected via `-P…`), uploads the AAB artifact, and **publishes to Play Store internal track** (`r0adkll/upload-google-play`, package `com.spacedrevision`). Cleans up the keystore `if: always()`.

**Jenkins (`Jenkinsfile`)** — a **stub/skeleton** only. Runs in a Docker `android-builder` agent, declares Android keystore credentials, but its `stages` just `echo` "Starting CI/CD" / print an env var. (Note: the block is misspelled `enviroment` and would not actually bind those credentials.) The GitHub Actions pipeline is the functional one.

**Xcode Cloud (`ci_scripts/ci_pre_xcodebuild.sh`)** — pre-build hook: installs Node deps (`npm ci`, falls back to yarn), Ruby deps (`bundle install`), then `bundle exec pod install --repo-update`. This is the iOS release path (Xcode Cloud), paralleling `android-release.yml`.

**No fastlane.** No `Fastfile` / `fastlane/` directory exists in either platform.

**Versioning (`scripts/version-manager.js`)** — `npm run version:*` keeps the marketing version + build numbers consistent across `package.json`, `app.json`, iOS, and Android. `__tests__/appVersion.test.ts` (run via `npm run test:version`) guards version consistency. Runtime update gating: `services/appVersion.service.ts` fetches `{min_build, latest_build, latest_version, message}`; `shared/utils/appVersion.ts::evaluateUpdate` returns `FORCE`/`OPTIONAL`/`OK`, consumed by `RootNavigator` (Force short-circuits the UI; Optional shows a dismissible modal tracked by `updatePromptTracker`).

**`app.json`** — carries an Expo config block (icons/splash/`bundleIdentifier com.spacedrevision`/`package com.spacedrevision`, plugins `expo-document-picker`, `expo-image-picker`) plus a top-level `version: 2.4.4`. **This is a bare React Native app, not Expo-managed** — the Expo block is largely vestigial config; builds go through native Gradle/Xcode toolchains, not EAS.

**Jest (`jest.config.js` + `jest.setup.js`, ~13 KB of mocks)** — `preset: react-native`, `maxWorkers: 2`, `watchman: false`, `testTimeout: 15000`, audio assets stubbed via `moduleNameMapper`, and a broad `transformIgnorePatterns` allowlist (redux, react-navigation, `.pnpm`, etc.). Tests co-locate in `__test__/` / `__tests__/` folders next to source.

---

## 10. Notable patterns &amp; gotchas

- **New Architecture OFF, on purpose.** `new_arch_enabled=false`/`fabric_enabled=false` (iOS) and `newArchEnabled=false` (Android) because WatermelonDB needs the legacy bridge. Don't flip these casually — you'll break the offline DB and the `rtn-*` native views.
- **`constants.ts` is git-ignored and per-environment.** Real credentials/URLs live only locally; CI synthesizes the file (from `.example` for tests, from Secrets for release). A working tree may show localhost/ngrok URLs — that's expected, not production.
- **Analytics fire inside thunks, not hooks.** PostHog `identify`/`capture` calls live in the auth thunks so they fire whether a screen dispatches directly or goes through `useAuth`. Follow this when adding tracked flows.
- **ViewModel + Repository conventions are enforced** (`CLAUDE.md`, `FolderPattern.md`): screens render only; logic in `use<Name>ViewModel`; HTTP in `services/*Repository.ts`; new files must land in the exact feature slot. Don't create new top-level folders or dump feature code in `shared/`.
- **Global logout = full store reset.** `rootReducer` returns `appReducer(undefined, logout)` — dispatching `logout` nukes all state at once. `useAuth.signOut` additionally clears WatermelonDB, search/recent caches, and SRS baselines (ordering matters — baselines cleared *after* `logout` to beat in-flight fetches).
- **Offline guard rejects requests app-side.** Any Axios call while offline throws an `axios.Cancel('OFFLINE_REQUEST_BLOCKED')`. Callers must treat `axios.isCancel(err)` as "offline," not a real failure.
- **Redux persist ≠ offline data.** Only `auth/theme/user/onboardingSelections` persist; course/study data is in SQLite via WatermelonDB. Bump `PERSIST_VERSION` on any persisted-shape change.
- **`RootNavigator.tsx` is a mega-file** owning splash, push notifications, badge counts, IAP init, update gating, floating feedback, and analytics — not just routing. Understand it before touching startup/notification behavior. `CLAUDE.md` explicitly forbids touching `SyncManager`/`NotificationBadge` wiring during UI-only refactors.
- **Dashboard eager, everything-lazy caused Android blank screens.** The tab roots are eagerly imported deliberately; only profile detail screens use `React.lazy`. Don't lazy-load tab roots.
- **Chat socket uses non-default path `/chat`** with a path-only → namespace fallback, and is globally managed (`globalManaged=true`) to prevent duplicate listeners. Per-screen chat code must not open its own connection.
- **Pod patches re-apply on every `pod install`.** RCT-Folly/fmt patches live in `patches/` but are applied to `Pods/` by the Podfile `post_install`, because `patch-package` only covers `node_modules`. If an iOS build fails on folly/fmt compile after a clean pod install, check these.
- **`postinstall` matters.** `scripts/prepare-nitro-modules-headers.js` must run (creates nitro headers) or Android CMake fails building `react-native-iap`. CI uses `--ignore-scripts`, so it relies on the checked-in state / separate steps — locally, don't skip postinstall.
- **Android hardcodes an nvm node path** in `app/build.gradle`. On a machine without that exact node version, it falls back to `node` on PATH — but a broken PATH during Gradle bundling is a classic failure here.
- **`react-native.config.js`** links fonts from `./src/assets/fonts` (`react-native-asset`), and `metro.config.js` swaps SVG handling to `react-native-svg-transformer` (SVGs are source modules, removed from `assetExts`).
- **Naming quirks to not "fix":** the run-Android script alias is `and` (not `android`); `rtn-my-video-picker` is a *player*, not a picker; the `testSeris` reducer key and `FlashCardArrtibuteModel`/`insrtuctor`-style misspellings are load-bearing (imports depend on them).

---

### Key file map (repo-relative)

| Concern | Path |
|---|---|
| Entry / registration | `index.js`, `src/app/App.tsx`, `src/app/Providers.tsx` |
| Store | `src/store/store.ts`, `rootReducer.ts`, `hooks.ts`, `migrations.ts`, `slices/` |
| Navigation | `src/navigation/RootNavigator.tsx`, `MainTabNavigator.tsx`, `AuthNavigator.tsx`, `types.ts` |
| HTTP client | `src/data/api/ApiService.ts`; config `src/shared/utils/constants.ts` (+ `.example.ts`) |
| Offline DB | `src/data/local/{database.ts,schema.ts,models/,queries/,sync/}` |
| Providers/contexts | `src/shared/providers/*`, `src/shared/context/*` |
| Native modules | `rtn-my-player/`, `rtn-my-video-picker/` |
| Native platform | `ios/Podfile`, `Gemfile`, `android/app/build.gradle`, `android/app/src/main/java/com/spacedrevision/MainApplication.kt` |
| CI | `.github/workflows/ci.yml`, `android-release.yml`; `ci_scripts/ci_pre_xcodebuild.sh`; `Jenkinsfile` (stub) |
| Conventions | `CLAUDE.md`, `docs/dev_documents/FolderPattern.md` |
| Build tweaks | `babel.config.js`, `metro.config.js`, `react-native.config.js`, `patches/`, `scripts/prepare-nitro-modules-headers.js` |
