# Mantes — Chat AI (iOS + Node.js)

> **Status:** 🚧 *Work-in-progress.* The foundation is solid (auth, model list, chat send, server‑side history, multi‑part rendering). It’s intentionally designed to be easy to **continue** and ship.

---

## What is this?

**Mantes** is a minimal yet production‑minded **Chat AI** stack inspired by the OpenAI app UX:

- **iOS (SwiftUI)**: sign up / log in (JWT in Keychain), pick a model, chat, render text + images (URL/base64), sidebar with local chat list.
- **Node.js backend** (Express + SQLite): auth, model catalog, chat send endpoint that **persists** history, and history retrieval endpoint.

> This repo focuses on clean architecture and safe defaults. Some “pro” features are not finished by design—so you can extend it your way.

---

## Features

### iOS (SwiftUI)
- Splash → fetch **models** from `/api/v2/model`.
- **Auth** with safe session persistence (JWT stored in **Keychain**).
- **Home** UI:
  - Model‑first flow (pick model before chatting).
  - Capsule composer (dominant vs the send button, auto‑grows with text).
  - Send chat → `POST /api/v2/chat/:model/aimodels`.
  - Automatic **server‑backed history** using `chat_id`.
  - Sidebar with search, new chat, and user/plan badge.
  - Multi‑part rendering (text, image URL/base64, file chip).
- Modern UI: subtle materials, rounded continuous shapes, spring animations.

### Backend (Node.js + Express + SQLite)
- **Auth**: `/auth/signup`, `/auth/login`, `/me` (JWT, bcrypt password hashing, helmet, CORS, basic rate‑limit).
- **Models**: `GET /api/v2/model` (OpenAI‑like `{ data: [...] }` shape).
- **Chat**:
  - Send & persist: `POST /api/v2/chat/:model/aimodels`.
  - Retrieve history: `GET /api/v2/chat/:chatId/:username/history`.
  - Flexible **ContentPart** schema (text/image/file).

---

## Architecture

```
ios/
 ├─ AppConfig.swift            # Backend base URL
 ├─ APIClient.swift            # HTTP client + DTO (models, chat, history)
 ├─ AuthStore.swift            # JWT + user (Keychain)
 ├─ AppState.swift             # availableModels from backend
 ├─ ModelItem.swift
 ├─ ChatMessage.swift          # ContentPart(text/image/file)
 ├─ ChatHistoryStore.swift     # sessions, local cache, remoteId(chat_id)
 ├─ ChatStore.swift            # UI state (input, messages, isSending)
 ├─ LoginView.swift
 ├─ HomeView.swift             # main UI + API integration
 ├─ SplashView.swift
 └─ MantesApp.swift            # root, Splash → Login/Home

server/
 ├─ server.js                  # Express API (auth, models, chat, history)
 └─ data/mantes.sqlite         # SQLite (auto-created)
```

---

## Getting Started

### Backend

1) Install & run:
```bash
cd server
npm install
# Optional: export JWT_SECRET and PORT
# export JWT_SECRET="change-me"
npm start
# server runs on http://localhost:3000
```

2) Quick endpoints:
- `GET /status` → `{ status: "ok", version }`
- `GET /api/v2/model` → model catalog
- `POST /auth/signup` / `POST /auth/login` → `{ token, user }`
- `GET /me` (Bearer) → `{ user }`
- `POST /api/v2/chat/:model/aimodels` (Bearer) → send chat and persist
- `GET /api/v2/chat/:chatId/:username/history` (Bearer) → read history

> Seed admin: `admin@example.com` / `admin123` (change for production).

### iOS (Xcode)
1) Open the iOS project (iOS 16+/Xcode 15+ recommended).
2) Update your backend URL if not localhost:
```swift
// AppConfig.swift
static let backendBaseURL = URL(string: "http://<your-ip-or-domain>:3000")!
```
3) Build & run.
   - Sign up / log in.
   - Pick a model, send a message.
   - Reopen a session with a `remoteId` → the app pulls history from server.

---

## API Contract (Summary)

### Auth
**POST** `/auth/signup`
```json
{ "email":"a@b.c", "password":"******", "name":"Juan" }
```
Response
```json
{ "token":"<JWT>", "user": { "id":1, "email":"a@b.c", "...": "..." } }
```

**POST** `/auth/login`
```json
{ "email":"a@b.c", "password":"******" }
```
Response
```json
{ "token":"<JWT>", "user": { "...": "..." } }
```

**GET** `/me` (Bearer) → `{ "user": { ... } }`

### Models
**GET** `/api/v2/model`
```json
{ "data": [ { "id":"sakra-small", "name":"Sakra Small", "hint":"Fast • Low cost" }, ... ] }
```

### Chat Send
**POST** `/api/v2/chat/:model/aimodels` (Bearer)
```json
{
  "username": "juan",
  "chat_id": "c_abc123",        // optional (if missing, server creates one)
  "message": { "content": [
    { "type":"text", "text":"hello" }
    // or images/files:
    // { "type":"image", "source":"url", "url":"https://..." }
  ]},
  "system": "You are Mantes...", // optional; only when chat is empty
  "metadata": { "any": "thing" } // optional
}
```
Response
```json
{
  "chat_id":"c_abc123",
  "choices":[{
    "message": { "role":"assistant", "content":[ { "type":"text","text":"You said: hello" } ] }
  }]
}
```

### Chat History
**GET** `/api/v2/chat/:chatId/:username/history?limit=100&order=asc` (Bearer)
```json
{
  "chat": { "id":"c_abc123", "username":"juan", "title":"hello", "...":"..." },
  "data": [
    { "role":"user", "content":[{ "type":"text","text":"hello" }], "created_at":"..." },
    { "role":"assistant", "content":[{ "type":"text","text":"You said: hello" }], "created_at":"..." }
  ]
}
```

#### `ContentPart` schema
```json
{
  "type": "text | image | file",
  "text": "string",
  "source": "url | base64",
  "url": "string",
  "data": "base64",
  "mime": "image/png",
  "alt": "string",
  "width": 640,
  "height": 360,
  "file_url": "string",
  "file_name": "string",
  "file_mime": "string"
}
```

---

## App Flow

1) **Splash** fetches model catalog.  
2) **Auth** creates a JWT session (stored in **Keychain**).  
3) **Home**:
   - Pick a **model** → session becomes model‑bound.
   - Send your first message → app posts to backend.
   - Backend creates a `chat_id` (if needed), persists messages, returns assistant content.
   - App stores `remoteId` (the `chat_id`) for future syncing.
   - Opening that session later → app pulls full **history** from server.

---

## Security & Privacy

- JWT is stored in **Keychain** (never in UserDefaults).
- Passwords are salted & hashed with **bcrypt**.
- **helmet**, **CORS**, and **basic rate‑limit** enabled on server.
- Base64 image decoding limited on client (8MB guard) to avoid memory spikes.
- Token lifetime: **7 days** (tweak as needed).

> **Production tips:** consider refresh tokens, TLS, token revocation, per‑user rate limiting, and moving file/image uploads to object storage (S3/GCS) with signed URLs.

---

## What’s Not Done Yet (Roadmap)

- [ ] Streaming responses (SSE/WebSocket) for real‑time typing.  
- [ ] List/delete chats from backend (`GET /api/v2/chat/:username/list`, `DELETE /api/v2/chat/:chatId`).  
- [ ] Uploads to object storage (replace base64).  
- [ ] Plan enforcement (restrict models by plan).  
- [ ] Consistent error codes + i18n messages.  
- [ ] Basic analytics (latency, token usage).  
- [ ] Real model providers wired into `generateAssistantReply` (OpenAI/Gemini/Llama/etc.).  
- [ ] Markdown rendering (code blocks, links) + copy buttons.  
- [ ] Unit/UI tests & CI.  
- [ ] Settings (theme, per‑model system prompt).  

---

## Customization

- Backend URL: `AppConfig.backendBaseURL`.  
- Model catalog: edit `models[]` in `server.js`.  
- Branding: tweak colors/gradients/fonts in `HomeView` & `SplashView`.

---

## Troubleshooting

- **400/401/403** when sending chat → verify **JWT** in `Authorization: Bearer <token>`; log in again.  
- **Missing history** → ensure session has `remoteId` (saved after the first send).  
- **Images not showing** → check `source/url/data` per `ContentPart` rules.  
- **Simulator can’t hit `localhost`** → use your LAN IP in `AppConfig` (e.g., `http://192.168.1.5:3000`).

---

## License

Use freely for internal dev, education, or as the base for your product. Add a formal license that suits your project needs.

---

## Closing Notes

This isn’t 100% complete on purpose. The **hard parts**—auth, stable API contracts, state + server‑side history, solid iOS architecture, safe persistence—are already **done** so you can move fast on the rest. If you want help with the next steps (streaming, chat lists, real model providers, or deployment), the code is ready for it.
