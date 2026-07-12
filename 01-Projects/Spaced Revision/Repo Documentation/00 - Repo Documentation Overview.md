---
Date: 2026-07-12
tags:
  - spaced-revision
  - architecture
  - documentation
Status: living
---

# Spaced Revision — Repo Documentation

In-depth working documentation for the four client/server repos that make up the core Spaced Revision platform. Generated from full source inspection (July 2026). Each note is a standalone deep-dive; this page is the map.

## Notes in this folder

| Repo | Stack | Deep-dive note |
|------|-------|----------------|
| `spaced-revision-sern-backend` | Node.js/Express (CommonJS), MySQL 8, Redis, ClickHouse, BullMQ | [[Backend - Main REST API]] |
| `spaced-revision-sern-frontend` | React 18 / Vite 7, Redux (classic thunks), Tailwind + DaisyUI, MUI 5 | [[Web Frontend - React]] |
| `react-native-app` | React Native 0.79 / React 19, TypeScript, Redux Toolkit, WatermelonDB | [[React Native App]] |
| `spaced-revision-chat-backend` | Node.js/Express (ESM), Socket.IO 4, MongoDB (Mongoose 8) | [[Chat Backend]] |

## System data flow

```
                 ┌─────────────────┐        ┌─────────────────┐
                 │  Web Frontend   │        │  React Native   │
                 │  (React/Vite)   │        │   (iOS+Android) │
                 └────────┬────────┘        └────────┬────────┘
                          │  x-auth-token (JWT)       │
                          ▼                           ▼
                 ┌────────────────────────────────────────────┐
                 │      Backend — Main REST API (Express)      │
                 │  MySQL 8 (primary) · Redis (cache+BullMQ)   │
                 │  ClickHouse (OLAP) · Firebase FCM · S3      │
                 └───┬───────────────┬───────────────┬────────┘
                     │               │               │
        Socket.IO /chat        push (FCM)      Razorpay orders
                     │               ▲               │
                     ▼               │               ▼
            ┌──────────────┐   (fire-and-      razorpay-webhook-server
            │ Chat Backend │    forget)         (separate process)
            │ Socket.IO +  │──────┘
            │   MongoDB    │
            └──────────────┘
```

- Both frontends authenticate to the **main backend** with a JWT sent in the **`x-auth-token`** header.
- **Chat** is a separate Socket.IO service on a **non-default path `/chat`** (both web + RN clients must set `{ path: '/chat' }`). It calls back into the main backend (`MAIN_API`) for push-notification delivery.
- **Push** is centralized: only the main backend holds FCM credentials; the chat backend fires fire-and-forget push requests to it.
- **Payments** create orders in the main backend; a separate `razorpay-webhook-server` process handles the webhooks (not documented here).

## Ports (dev)

| Service | Port | Path notes |
|---------|------|-----------|
| Main backend | 8080 | REST `/api`, `/metrics`, `/health`, `/api-docs`, `/admin/queues` |
| Web frontend | 3000 | Vite dev, `--host` + ngrok hosts allowed |
| Chat backend | 9000 | Socket.IO path **`/chat`** (not `/socket.io`); `/iap` namespace |
| RN Metro | 8081 | `npm start --reset-cache` |

## Cross-cutting facts worth remembering

- **Package manager drift:** both the web frontend and the RN app have migrated to **pnpm** (with `--frozen-lockfile` CI), even though Volta/CLAUDE.md still mention Yarn. Install with pnpm.
- **Auth model:** main backend = real JWT (365-day app tokens, `x-auth-token`). Chat backend = **trust-based** socket auth (client asserts `id`/`username`/`is_admin`, no token check) — a known security gap.
- **The `sevices/` (misspelled) directory in the backend is real and load-bearing** — do not "fix" the spelling.
- **WatermelonDB (SQLite)** is the RN app's offline store; course catalogs no longer live in redux-persist.
- **Brand teal `#1F4445` / `#2F6767`** is only in hand-written CSS on web — the DaisyUI `primary` token is actually blue `#033A84`.

> Related task board: [[Things to do]] · these notes complement each repo's in-tree `CLAUDE.md` + `docs/`.
