---
Date: 2026-07-12
tags:
  - spaced-revision
  - chat
  - socket-io
  - documentation
Status: living
---

> Part of [[00 - Repo Documentation Overview]]

# Spaced Revision — Chat Backend (`spaced-revision-chat-backend`)

Real-time chat microservice for the Spaced Revision platform. Node.js + Express 5 + Socket.IO 4 + MongoDB (Mongoose 8). Handles course/room chat, 1:1 direct messages, feedback/bug-report threads, file attachments via S3, push-notification fan-out to the main API, a maintenance-mode flag, and an In-App-Purchase (IAP) notification side-channel.

> Repo root: `/Users/stardust/Code/spaced-revision-chat-backend`
> Everything of substance lives under `src/`. There is **no test suite, no lint, no build step** — only `npm run dev` (nodemon).

---

## 1. Overview

| Fact | Detail |
|------|--------|
| Language / module system | **ESM** (`"type": "module"` in `package.json`), Node 24 in Docker (Node 20 in CI) |
| HTTP + WS port | **9000** (`process.env.PORT || 9000`) |
| Datastore | MongoDB Atlas via Mongoose (`chat_messages`, `chat_room_reads`, `bug_reports`, `system_config` collections) |
| Realtime | Socket.IO server mounted on a **custom path `/chat`** (not the default `/socket.io`) |
| Entry point | `src/index.js` |
| Run (dev) | `npm run dev` → `nodemon src/index.js` |
| Run (prod) | `node src/index.js`, or PM2 via `environment.config.mjs`, or Docker |
| Docker | `docker compose up --build` (`compose.yaml` builds `Dockerfile`, maps `9000:9000`, `env_file: .env`) |

### Top-level-await + hard startup dependency on Mongo

`src/index.js` uses ESM top-level `await`, so the process **cannot finish booting without a working Mongo connection**:

```js
const PORT = process.env.PORT || 9000;
await connectMongo();          // top-level await — blocks module eval
const app = express();
app.use(bodyParser.json());
app.use(cors({ origin: '*' }));
const server = http.createServer(app);
await initSocket(server, { corsOrigin: '*' });   // also awaits connectMongo() again (idempotent)
initIapSocket();
```

`connectMongo()` lives in `src/services/chatServices.js` and **throws at import time** if `MONGO_URI` is unset — this happens as the module is first evaluated, before any function runs:

```js
const MONGO_URI = process.env.MONGO_URI;
if (!MONGO_URI) {
  throw new Error('The MONGO_URI environment variable is not set. Please define it to connect to MongoDB.');
}
export async function connectMongo() {
  if (mongoose.connection.readyState === 1) return;   // idempotent guard
  await mongoose.connect(MONGO_URI, { minPoolSize: 2, maxPoolSize: 10 });
  console.log('✅  MongoDB connected');
}
```

Connection pool is fixed at min 2 / max 10.

### Startup order

1. `dotenv/config` loads `.env`.
2. `await connectMongo()` — connect to Mongo (throws if URI missing).
3. Express app + `bodyParser.json()` + `cors({ origin: '*' })`.
4. `await initSocket(server, …)` — create the Socket.IO server on path `/chat`, wire the admin UI, register the singleton IO instance, attach all connection handlers.
5. `initIapSocket()` — attach the `/iap` namespace onto the same server.
6. Mount REST routers, then `server.listen(9000)`.

### Dependencies (and notably-unused ones)

Used: `express`, `socket.io`, `@socket.io/admin-ui`, `mongoose`, `cors`, `dotenv`, `axios`, `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`.

**Declared but never imported anywhere in `src/`:** `jsonwebtoken`, `redis`, `@socket.io/redis-adapter`. Consequences:
- `JWT_SECRET` is set in every env file but **no code reads it** — there is no JWT verification in this service.
- No Redis adapter is wired, so Socket.IO runs **single-node only**. The `UserConnectionManager` is an in-memory `Map` and rooms are in-process — horizontal scaling would break DM routing and admin/room fan-out. Consistent with the PM2 config (`instances: 1`, `exec_mode: "fork"`).

**Latent packaging gotcha:** `src/index.js` does `import bodyParser from 'body-parser'`, but `body-parser` is **not** in `package.json` dependencies. It resolves only because Express 5 bundles it transitively. `npm ci --omit=dev` in the Dockerfile still works today only by that accident.

---

## 2. Socket.IO setup — the `/chat` path is mandatory

The single most important integration fact. In `src/ws/chatSocket.js`:

```js
const io = new Server(httpServer, {
  path: '/chat',
  cors: { origin: opts.corsOrigin || '*' },
});
```

The Socket.IO **path is overridden to `/chat`**. Clients that connect with the default configuration (`/socket.io`) will **never establish a connection**. Every client MUST pass `{ path: '/chat' }`:

```js
const socket = io('http://localhost:9000', { path: '/chat' });
```

### Two namespaces, both on path `/chat`

- **Main namespace** (`/`) — all chat, DM, file, feedback, bug-report traffic (`src/ws/chatSocket.js`).
- **`/iap` namespace** — IAP notification channel (`src/ws/iap.socket.js`).

The transport **path** and the Socket.IO **namespace** are independent concepts. The `/iap` namespace is still served over path `/chat`. An IAP client connects like this (from the doc comment in `iap.socket.js`):

```js
io('http://localhost:9000/iap', {
  path: '/chat',
  extraHeaders: { authorization: 'YOUR_KEY' }
});
```

### Shared IO singleton

`src/ws/socketInstance.js` holds the one `Server` instance so non-socket code (routes/services) can emit:

- `setIO(io)` — called once by `initSocket()`. **Throws `'Socket.IO instance already set'` if called twice.**
- `getIO()` — throws `'Socket.IO not initialized…'` if called before `initSocket()`. This is why `initIapSocket()` must run *after* `initSocket()` — it calls `getIO().of('/iap')`.

### Connection handshake (main namespace)

Connecting establishes a socket but grants **nothing** until the client emits an `auth` event (see §3). Until authenticated, the handler-local `userId`/`username` are `null` and every gated event (`subscribe`, `publish`, etc.) rejects with an error event.

---

## 3. Socket auth model — trust-based, client-asserted `is_admin`

There is **no token verification on the socket layer.** Authentication is a client-supplied `auth` event whose fields are taken at face value (`src/ws/chatSocket.js`):

```js
socket.on('auth', ({ id, username: userUsername, is_admin: isAdmin }) => {
  if (!isValidUserId(id) || !userUsername) {
    socket.emit('auth-error', { message: 'Invalid user ID or username' });
    return;
  }
  userId = id;
  username = userUsername;
  socket.username = username;
  if (userManager.addUser(userId, username, socket.id)) {
    socket.join(`user:${userId}`);       // personal room for DMs
    if (isAdmin) {
      socket.join('room:admins');        // ← admin room, gated only by a client boolean
    }
    socket.emit('auth-success', { userId, username });
  } else {
    socket.emit('auth-error', { message: 'Failed to authenticate user' });
  }
});
```

`isValidUserId` (in `directMessageService.js`) merely checks the value is a non-empty string:

```js
export function isValidUserId(userId) {
  return userId && typeof userId === 'string' && userId.trim().length > 0;
}
```

**Security implications (flag for the KB):**
- Any client can claim any `id`/`username` — there is no proof of identity. A malicious client can impersonate another user, send DMs as them, and mark rooms read on their behalf.
- **`is_admin` is entirely client-controlled.** Anyone who sends `auth` with `is_admin: true` joins `room:admins` and thereby receives every `new-bug-report`, `new-feedback-message`, and admin-targeted event in real time. This is a genuine information-disclosure hole.
- Note the distinction between two "admin" concepts: the *socket* `room:admins` (client-asserted) vs. the server-side `CHAT_ADMIN_USER_IDS` env list, which is the trusted set used to decide **push-notification recipients** for feedback/bug rooms. The env list is authoritative for who gets pinged; the socket boolean decides who gets the live socket event.

The **only secret-gated socket surface** is the `/iap` namespace, which enforces a shared key (§8). Push-notification and friendship-gate calls to the main API do pass an `authToken`/internal key, but those are forwarded to `MAIN_API`, not validated here.

---

## 4. Events & rooms

### Client → Server events (main namespace)

| Event | Payload | Effect |
|-------|---------|--------|
| `auth` | `{ id, username, is_admin }` | Authenticate; join `user:{id}` (and `room:admins` if `is_admin`). |
| `subscribe` | `{ roomId }` | Join room; emits `history` (last 50 msgs). Requires prior `auth`. |
| `subscribe-direct` | `{ otherUserId }` | Emits `direct-history` for the DM pair. |
| `unsubscribe` | `{ roomId }` | Leave a room. |
| `publish` | `{ roomId, user, username, text, authToken }` | Persist + broadcast a room message; fire push notifications. |
| `publish-direct` | `{ recipientId, text, authToken }` + ack callback | Friendship-gated DM; support-user 8234 side-effects. |
| `file-uploaded` | `{ roomId, key, filename, contentType, size, senderId }` | Attach an already-uploaded S3 object to a room; emits `file`. |
| `file-uploaded-direct` | `{ recipientId, key, filename, contentType, size }` | DM file attachment; emits `direct-file`. |
| `read` | `{ roomId, userId }` | Upsert read marker; emits `room_read_ack`. |
| `read-direct` | `{ userId, otherUserId }` | Read marker on the sorted DM key; emits `direct_read_ack`. |
| `bug-report` | `{ text, type, email, authToken }` | Create `BugReport` + `bug-{id}` room; notify admins. |
| `feedback-message` | `{ text, type, email, authToken }` | Post to `feedback-{userId}` room; notify admins. |
| `disconnect` / `error` | — | Cleanup / error surface. |

### Server → Client events

`auth-success`, `auth-error`, `subscribe-error`, `history` `{ roomId, messages }`, `direct-history` `{ messages, otherUserId, otherUsername, conversationId }`, `message`, `publish-error`, `direct-message`, `direct-error`, `direct-file`, `file`, `file-error`, `room_read_ack`, `direct_read_ack`, `new-bug-report` `{ reportId, roomId, type? }`, `bug-report-ack`, `bug-report-error`, `new-feedback-message` `{ roomId, userId }`, `feedback-message-ack`, `connection-error`.

### Room-name conventions (load-bearing)

| Room name | Meaning | Who joins / how used |
|-----------|---------|----------------------|
| `user:{userId}` | A user's personal room | Joined on `auth`. **DMs are delivered here**, not via a shared room. |
| `room:admins` | Admin broadcast channel | Joined on `auth` if `is_admin` truthy. Receives `new-bug-report`, `new-feedback-message`. |
| `{roomId}` (arbitrary) | Course / group chat room | Joined via `subscribe`. Usually a course id. |
| `feedback-{userId}` | Per-user support thread | One room per user; created lazily on first `feedback-message`. |
| `bug-{reportId}` | One thread per bug report | `reportId` is the Mongo `_id` of the `BugReport`. |

**DM delivery is via personal rooms, not a joint room.** A DM is emitted to *both* participants' `user:` rooms:

```js
io.to(`user:${userId}`).to(`user:${recipientId}`).emit('direct-message', messageData);
```

The **sorted DM conversation key** — `[a, b].sort().join('-')` — is used only for **read-state and unread counting**, never as a socket room. It appears:
- In `subscribe-direct`'s response as `conversationId: ${[userId, otherUserId].sort().join('-')}`.
- In `markDirectChatRead` as the `RoomRead.roomId`.
- In the DM unread pipeline, keyed as `dm:<minId>-<maxId>`.

### Magic recipient `8234` = support

When a `publish-direct` targets recipient **`8234`** (the support account), the handler auto-creates a `BugReport` and a `bug-{id}` room, in addition to saving the DM. The **message-text prefix decides the report type**:

```js
const isSupportMessage = String(recipientId) === '8234';
if (isSupportMessage) {
  const featureTagMatch = text.match(/^\[Request a feature\]/i);
  const messageTagMatch = text.match(/^\[Send us a message\]/i);
  if (featureTagMatch)      bugReportType = 'feature';
  else if (messageTagMatch) bugReportType = 'message';
  else                      bugReportType = 'message';   // default
  // → saveBugReport({ senderId, username, type, text, recipientId: 'admins' })
  //   roomId = `bug-${bugReportDoc._id}`; saveMessage(...) into that room;
  //   io.to('room:admins').emit('new-bug-report', { reportId, roomId, type })
}
```

Recipient `8234` is also **exempt from the friendship gate** and is **excluded from DM push notifications** (support messages don't push to 8234). The DM itself is still saved and delivered to both personal rooms as normal.

---

## 5. Data model (MongoDB / Mongoose)

Four models, all in `src/models/`.

### `Message` → collection `chat_messages` (`src/models/Message.js`)

**Dual-mode schema** — the same collection stores both room messages and direct messages, discriminated by the `isDirect` flag. Timestamps map `createdAt → ts`; `updatedAt` disabled; `versionKey` off.

| Field | Type | Notes |
|-------|------|-------|
| `roomId` | String | indexed, sparse. Room messages only. |
| `user` | String | default `'anon'`. Legacy sender id (kept for backward compat). |
| `sender` | String | indexed. **Required when `isDirect`**. |
| `recipient` | String | indexed, sparse. DM recipient. |
| `username` | String | **required**, max 50. Sender display name. |
| `recipientUsername` | String | max 50. |
| `text` | String | **max 1000**, trimmed. |
| `file` | object | `{ key (indexed, sparse), filename ≤255, contentType ≤100, size ≥0 }`. |
| `isDirect` | Boolean | default `false`, indexed. Mode switch. |
| `ts` | Date | = `createdAt`. |

Compound indexes: `{ isDirect, sender, recipient }`, `{ isDirect, ts:-1 }`, `{ roomId, ts:-1 }`.

The dual-mode invariant is enforced by a `pre('validate')` hook:

```js
messageSchema.pre('validate', function(next) {
  if (this.isDirect) {
    if (!this.sender || !this.recipient)
      return next(new Error('Direct messages must have both sender and recipient'));
    if (this.roomId)
      return next(new Error('Direct messages cannot have a roomId'));
  } else {
    if (!this.roomId)
      return next(new Error('Room messages must have a roomId'));
  }
  if (!this.text && !this.file?.key)
    return next(new Error('Messages must have either text content or a file attachment'));
  next();
});
```

Helper methods/statics: `isFromUser`, `getOtherUserId`, and a static `findDirectMessages(u1, u2, {limit, beforeTs})` (caps limit at 100).

### `RoomRead` → collection `chat_room_reads` (`src/models/RoomRead.js`)

Read-position tracking, reused for both rooms and DMs.

| Field | Type | Notes |
|-------|------|-------|
| `roomId` | String | **required**, indexed. For DMs this holds the *sorted conversation key*. |
| `userId` | String | **required**, indexed. |
| `lastReadAt` | Date | default epoch `new Date(0)`. |

Compound index `{ roomId, userId }` — the comment explains it directly serves the `$lookup` in both unread pipelines (matches `roomId` AND `userId` per message doc; without it Mongo filters in memory, the dominant unread-query cost).

### `BugReport` → collection `bug_reports` (`src/models/BugReport.js`)

| Field | Type | Notes |
|-------|------|-------|
| `senderId` | String | **required**, indexed. |
| `username` | String | **required**, max 100. |
| `recipientId` | String | **required**, indexed. In practice `'admins'`. |
| `type` | String | **enum `['issue','feature','message']`**, default `'message'`. |
| `text` | String | **required**. |
| `email` | String | max 255, default `null`. |
| `ts` | Date | = `createdAt`. |

Indexes: `{ recipientId, ts:-1 }`, `{ senderId, ts:-1 }`. (Note: the socket `bug-report` handler passes `type: type || 'message'`, but the enum has no `'issue'` producer in the socket path — only `feature`/`message` are set from tags; `issue` is valid but only reachable if a caller supplies it.)

### `SystemConfig` → collection `system_config` (`src/models/SystemConfig.js`)

Generic key/value flags; used today only for `maintenance_mode`.

| Field | Type | Notes |
|-------|------|-------|
| `key` | String | **required, unique, indexed**, max 100. |
| `value` | Mixed | **required**. For maintenance this is a boolean. |
| `message` | String | max 500, default `null`. |
| `maintenanceDurationMinutes` | Number | default `null`, min 0. |
| `createdAt`/`updatedAt` | Date | both timestamps enabled here (unlike the other models). |

---

## 6. Push notifications — fire-and-forget to the main API

All push logic is in `src/services/pushNotificationService.js`. This service **holds no FCM credentials**; it calls the **main SERN backend** (`MAIN_API`) which owns the tokens and actually delivers via FCM. Every push call from the socket handlers is **fire-and-forget** — `.catch()`-ed and never awaited on the send path, so a notification failure never blocks or fails the already-delivered socket message.

Three exported senders:

1. **`sendPushNotificationsToSubscribers(roomId, username, authToken, text, senderId)`** — the default path for a `publish` to a normal room:
   - `GET {MAIN_API}/community/subscribers/{roomId}` → subscriber list.
   - `GET {MAIN_API}/push-tokens/list` → users with FCM tokens (array is nested at `response.data.data`).
   - Intersect the two by `user_id`, **exclude the sender**, then `GET {MAIN_API}/course/details/{roomId}` for the course name.
   - For each remaining user, `POST {MAIN_API}/chat-push-notification/send` (because `use_chat_push_route: true`) with title `"{courseName}: "`, body `"{username}: {text}"`, and `routingData` `{ route:'Chat', screen:'RoomChat', room_id, room_name }`.

2. **`sendRoomPushNotificationToUsers({ recipientUserIds, roomId, roomName, senderUsername, text, authToken, title })`** — used for **feedback/bug rooms**, where recipients must be explicit (admins + room owner) rather than course subscribers. Dedupes, truncates body at 100 chars, `POST {MAIN_API}/push-notification/send` per recipient.

3. **`sendDirectMessagePushNotification(recipientId, senderId, senderUsername, messagePreview, authToken)`** — for DMs. No-op if `authToken` missing. `routingData` opens `DirectChat` with the sender. Skipped entirely for recipient `8234`.

**How each socket path decides recipients** (in the `publish` handler):

```js
const isFeedbackRoom = String(roomId).startsWith('feedback-');
const isBugRoom      = String(roomId).startsWith('bug-');
if ((isFeedbackRoom || isBugRoom) && authToken) {
  // recipients = CHAT_ADMIN_USER_IDS ∪ {room owner}, minus the sender
  // feedback- → owner is the userId suffix; bug- → owner is BugReport.senderId
  await sendRoomPushNotificationToUsers({ ... });
} else {
  // normal room → notify course subscribers with FCM tokens (excluding sender)
  sendPushNotificationsToSubscribers(roomId, senderDisplayName, authToken, text, senderUserId).catch(...);
}
```

The `bug-report` and `feedback-message` handlers additionally push to `CHAT_ADMIN_USER_IDS` (minus the sender) via `sendRoomPushNotificationToUsers` when `authToken` is present and the admin list is non-empty.

The generic `sendNotificationToUser(...)` helper chooses `/chat-push-notification/send` vs `/push-notification/send` based on `options.use_chat_push_route`, and forwards `authToken` as an `x-auth-token` header on every call.

---

## 7. File uploads / S3 (`src/services/storageService.js`)

The service never proxies file bytes. Clients get a **presigned S3 PUT URL**, upload directly to S3, then tell the socket the object exists via `file-uploaded` / `file-uploaded-direct`.

Config (constants at top of file):
- `MAX_FILE_SIZE` = 50 MB.
- `ALLOWED_TYPES` = `application/pdf`, `image/png|jpeg|gif|webp` only.
- Upload URL TTL = **60 s**; download URL TTL = **5 min**.
- Key format: `uploads/{roomId | 'direct-messages'}/{uuid}.{ext}`.

Exports: `presignPut({roomId, filename, contentType, sizeBytes})`, `presignPutDirect(...)` (roomId → `direct-messages` path), `presignGet({key})`, `getStorageConfig()`. Upload params are validated (`validateUploadParams`) before signing.

### Two deliberate S3 workarounds (KB-worthy)

**(a) `presignGet` omits `ResponseContentType` / `ResponseContentDisposition`.** The code comment is explicit:

```js
// Note: We don't include ResponseContentType or ResponseContentDisposition ...
// because they can cause signature mismatches in presigned URLs (400 Bad Request errors).
// S3 will serve the file with the Content-Type that was set during upload (PUT operation).
const getCommand = new GetObjectCommand({ Bucket: process.env.S3_BUCKET, Key: key });
```

So the download content type is whatever was set on the PUT, not overridable at GET time — a deliberate trade to avoid intermittent 400s from signature mismatches.

**(b) Eventual-consistency delay + retry** in both file socket handlers (`chatSocket.js`). After saving the message, before presigning the GET URL, there is a **300 ms sleep**, then up to **3 retries** of `presignGet` with **200 ms** backoff. If all retries fail the message is still emitted with `url: null` (frontend can re-request via `POST /api/file-url`):

```js
await new Promise(r => setTimeout(r, 300));   // let S3 settle
let url, retries = 3;
while (retries > 0) {
  try { url = await presignGet({ key, contentType, filename }); break; }
  catch (err) { if (--retries === 0) url = null; else await new Promise(r => setTimeout(r, 200)); }
}
```

Purpose: prevent "Preview unavailable" errors when a freshly-uploaded object is fetched immediately.

REST endpoints for the upload flow live in `src/routes/uploadRoutes.js`, mounted at `/api`: `POST /api/upload-url`, `POST /api/upload-url-direct`, `POST /api/file-url`, `GET /api/upload-config`. Several read routers (`chatRoutes`, `directMessageRoutes`) also hydrate `file.key` into short-lived `file.url` via `presignGet` on the way out.

---

## 8. IAP namespace (`src/ws/iap.socket.js`, `src/controllers/iap.controler.js`, `src/services/iap.service.js`, `src/routes/iap.routes.js`)

A separate real-time channel used to push In-App-Purchase status updates to a specific user's app instance (e.g. after a webhook confirms a purchase).

**Namespace `/iap`, still on transport path `/chat`.** The connection is **secret-gated** — the only such surface in the service — via a handshake middleware:

```js
iapIo.use((socket, next) => {
  const authToken = socket.handshake.headers.authorization;
  const expected  = process.env.IAP_AUTHORIZATION_KEY;
  if (!authToken || !expected || authToken !== expected) {
    return next(new Error('Authorization Denied'));
  }
  next();
});
```

After connecting, a client emits `register` with `{ userId }` to join its `user:{userId}` room. Server-side, `notifyIap(userId, payload)` (in `iap.service.js`) emits:

```js
getIapNamespace().to(`user:${userId}`).emit('iap:notification', payload);
```

`getIapNamespace()` returns `getIO().of('/iap')` — so it only works after both `initSocket()` and `initIapSocket()` have run.

**REST trigger:** `POST /iap/notification` (router `src/routes/iap.routes.js`) is protected by `checkAuth` (`src/middlewares/iap.auth.js`), which checks the same `IAP_AUTHORIZATION_KEY` against the `authorization` header (401 otherwise). The controller reads `{ status, userId }` from the body and calls `notifyIap(userId, { status })`, returning 200. This is how the payment/webhook side of the platform pushes a live purchase-status update to the app over the IAP socket.

---

## 9. Admin UI — hardcoded bcrypt credential (security concern)

`initSocket()` wires the official Socket.IO Admin UI (`@socket.io/admin-ui`) with a **credential baked into source** (`src/ws/chatSocket.js`):

```js
instrument(io, {
  auth: {
    type: 'basic',
    username: 'admin',
    password: '$2b$10$heqvAkYMez.Va6Et2uXInOnkCT6/uQj1brkrbyG3LpopDklcq7ZOS',  // bcrypt hash, in-repo
  },
});
```

This exposes the admin instrumentation namespace (`/admin`) so an operator can watch live sockets/rooms at `https://admin.socket.io`. **Flag:** the bcrypt hash is committed to the repository; the plaintext for username `admin` is whatever hashes to this value, and the hash being public makes offline cracking possible. There is no env-var override — rotating it requires a code change. Treat this as a hardcoded secret to migrate to `process.env`.

---

## 10. Environment variables & gotchas

### Environment variables

| Var | Required? | Used by | Notes |
|-----|-----------|---------|-------|
| `MONGO_URI` | **Yes** | `chatServices.js` | **Throws at module import** if unset (top-level check). |
| `PORT` | No | `index.js` | Default 9000. |
| `MAIN_API` | Yes (for push + DM gate) | `pushNotificationService.js`, `friendshipGate.js` | Base URL of the SERN backend. **Inconsistent default handling**: friendshipGate falls back to `http://localhost:8080`; push service reads `process.env.MAIN_API` with no fallback (undefined → broken URLs). In `.env` it's `http://localhost:8080/api`. |
| `IAP_AUTHORIZATION_KEY` | Yes (for IAP) | `iap.socket.js`, `iap.auth.js` | Gates both the `/iap` socket and `POST /iap/notification`. If unset, all IAP auth **fails closed** (the `!expected` check denies). |
| `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `S3_BUCKET` | Yes (for files) | `storageService.js` | S3 presigning. Region `ap-south-1`. |
| `CHAT_ADMIN_USER_IDS` | No | `chatSocket.js` | Comma list (e.g. `3,11,6399`) parsed into a `Set`. **Server-trusted** admin identity for push-recipient decisions (distinct from the client-asserted `room:admins`). |
| `INTERNAL_SERVICE_KEY` | Yes (for DM gate) | `friendshipGate.js` | Sent as `x-internal-key` to `MAIN_API/peers/can-dm`. **If unset, the gate denies all non-support DMs** (fails closed). |
| `JWT_SECRET` | No | — | Present in every env file but **never referenced in code**. Dead config. |
| `NODE_ENV` | No | routes | Only gates whether error `stack`/`message` details are returned in responses. |

### Committed-secret hazards
- **`.env` is present in the working tree** with live-looking MongoDB Atlas and AWS credentials (it's `.gitignore`d, but exists on disk).
- **`environment.config.mjs` is a PM2 ecosystem file with hardcoded PROD secrets** (`MONGO_URI` with password, AWS keys, `JWT_SECRET`) and is **not** gitignored. Despite the `.mjs` name it is not an app config module — it's the PM2 `apps[]` deployment descriptor pointing at `/home/ubuntu/projects/spaced-revision-chat-backend/src/index.js`.
- The admin-UI bcrypt hash (§9) is in `chatSocket.js`.

### Behavioral gotchas
- **No authentication on any REST route.** `/rooms/*`, `/direct/*`, `/bug-reports/*`, `/feedback-rooms/*`, `/system/*`, `/api/*` accept a `userId`/`currentUser` query param and trust it. Notably **`POST /system/maintenance` is unauthenticated** — anyone who can reach the port can toggle platform-wide maintenance mode (`SystemConfig` `maintenance_mode`), which the apps read via `GET /system/status`.
- **CORS is `origin: '*'`** on both Express and Socket.IO.
- **Single-node only.** In-memory `UserConnectionManager` + in-process rooms + no Redis adapter mean DM routing and admin fan-out break if you run more than one instance. `getUserUsername(recipientId)` returns the placeholder `User_{id}` when the recipient isn't currently connected — and that placeholder gets **persisted** as `recipientUsername` on the message doc.
- **`setIO` is one-shot** — a second `initSocket()` (e.g. an accidental double-init) throws.
- **Friendship gate** (`evaluateDmGate` / `checkCanDm`) enforces a 1-free-intro-message quota for non-peers by calling `MAIN_API/peers/can-dm`; peers/educators bypass; a reply from the recipient unlocks free chatting; support (`8234`) is exempt. It **fails closed** (denies) on timeout (5 s), non-200, or missing internal key.
- **REST read routes rehydrate file URLs** every call (short-lived presigned GETs), so responses aren't cacheable and add S3 signing latency per attachment.
- **`fetchMessages` returns oldest→newest** (queries newest-first with a limit, then `.reverse()`), which matters for pagination via `before`/`beforeTs` cursors.
- **Unread counts** (`unreadCountsFor`) return a merged map keyed `room:{roomId}` and `dm:{sortedKey}`, computed by two Mongo aggregation pipelines that `$lookup` into `chat_room_reads`.
- **CI does effectively nothing** (`.github/workflows/ci.yml`): `npm ci` then `npm test --if-present` — and there is no `test` script, so it always passes.
- **No `/health` endpoint** exists despite the platform-wide monitoring conventions; liveness is inferred from the listen log line.

---

## Appendix — file map

```
src/index.js                         entry: mongo connect → express → sockets → REST → listen(9000)
src/ws/chatSocket.js                 main namespace: all chat/DM/file/feedback/bug handlers + admin UI
src/ws/iap.socket.js                 /iap namespace (key-gated) + getIapNamespace()
src/ws/socketInstance.js             setIO/getIO singleton
src/models/Message.js                chat_messages (dual-mode room/DM)
src/models/RoomRead.js               chat_room_reads (read markers, rooms + DMs)
src/models/BugReport.js              bug_reports
src/models/SystemConfig.js           system_config (maintenance_mode)
src/services/chatServices.js         connectMongo, saveMessage, fetch/sorted/feedback/unread
src/services/directMessageService.js DM persistence, conversation queries, quota count, isValidUserId
src/services/bugReportService.js     bug report CRUD
src/services/friendshipGate.js       DM permission gate → MAIN_API/peers/can-dm
src/services/pushNotificationService.js  3 push senders → MAIN_API (FCM owned by main backend)
src/services/storageService.js       S3 presign PUT/GET, validation, config
src/services/iap.service.js          notifyIap → /iap emit
src/controllers/iap.controler.js     POST /iap/notification handler (filename misspelled)
src/middlewares/iap.auth.js          checkAuth: IAP_AUTHORIZATION_KEY
src/routes/chatRoutes.js             /rooms
src/routes/uploadRoutes.js           /api (presign)
src/routes/directMessageRoutes.js    /direct
src/routes/bugReportRoutes.js        /bug-reports
src/routes/feedbackRoomsRoutes.js    /feedback-rooms
src/routes/iap.routes.js             /iap
src/routes/systemRoutes.js           /system (maintenance mode — unauthenticated)
```

---

**Key non-obvious findings:** the Socket.IO path override to `/chat`; fully trust-based socket auth with client-supplied `is_admin` (anyone can join `room:admins`); completely unauthenticated REST surface including `POST /system/maintenance`; three declared-but-unused deps (`jsonwebtoken`, `redis`, `@socket.io/redis-adapter`) implying single-node-only operation; `body-parser` imported without being a declared dependency; and three committed-secret hazards (`.env` on disk, `environment.config.mjs` PM2 file with prod secrets not gitignored, and the hardcoded admin-UI bcrypt hash in `src/ws/chatSocket.js`).
