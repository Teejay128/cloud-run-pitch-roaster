# Pitch Arena API — Live AI Pitch & Roast Arena

A lightweight, decoupled **Node.js (ES Modules)** backend service combining **Express REST endpoints**, a native **WebSocket server (`/ws`)**, and the official **Google Gen AI SDK (`@google/genai`)**, paired with a single-file static frontend for live audience workshops on **Google Cloud Run**.

## Why a Standalone Containerized API?

Tightly coupled full-stack page routes struggle when a single live experience needs:
1. **Stateful WebSocket fan-out** (`/ws`) with 30-second server heartbeats and live presence broadcasting across every connected audience browser.
2. **Multi-client ingestion** from the web dashboard (`POST /api/pitch`), `curl` / Postman scripts, and messaging bot webhooks (`POST /api/webhook/bot`) entering the exact same evaluation and broadcast pipeline.
3. **Zero-build portability** where Google Cloud Buildpacks detect `package.json`, install dependencies, and run `npm start` directly from source—no custom `Dockerfile` required.
4. **Live Cloud Run revision visibility (`K_REVISION`)** and configurable `AI_PERSONA` environment variables so presenters can demonstrate 50/50 canary traffic splitting in real time.

---

## Project Structure

```text
.
├── package.json          # Node.js >=20 ES Modules config + express, ws, @google/genai
├── index.js              # Express HTTP server, WebSocket server, & Gemini evaluation logic
├── public/
│   └── index.html        # Single-file UI (HTML + Tailwind CDN + Vanilla JS)
├── .env.example          # Environment variable template
├── .dockerignore         # Excludes node_modules, .env, .git during Cloud Run source builds
└── README.md             # Quickstart & Cloud Run deployment commands
```

---

## Local Quickstart

Requirements: **Node.js 20+**

```bash
npm install
cp .env.example .env
npm start
```

Open [http://localhost:8080](http://localhost:8080).

- **Zero-config mock mode:** If `GEMINI_API_KEY` is blank or unset, the service automatically evaluates pitches using deterministic mock responses (`mode: "mock"`) so first deployments never fail with `500` errors.
- **Live Gemini mode:** Add `GEMINI_API_KEY` to `.env` and start with:

```bash
node --env-file=.env index.js
```

---

## Cloud Shell Workshop Deployment (3 Steps)

Run these commands directly from the project root in **Google Cloud Shell**:

### 1. Initial Source Deployment (Zero-Config Mock Mode)

Deploy straight from source with Cloud Buildpacks (no `Dockerfile` needed):

```bash
gcloud run deploy pitch-arena \
  --source . \
  --region us-central1 \
  --allow-unauthenticated
```

### 2. WebSocket Production Tuning

Increase request timeout for long-lived WebSocket connections, enable session affinity, and raise concurrency per container:

```bash
gcloud run services update pitch-arena \
  --region us-central1 \
  --timeout=3600 \
  --session-affinity \
  --concurrency=500
```

### 3. Enable Live Gemini Evaluation (Vertex AI or Secret Manager)

**Option A — Vertex AI with GCP Credits (no API key required on Cloud Run):**

```bash
gcloud run services update pitch-arena \
  --region us-central1 \
  --set-env-vars="GOOGLE_GENAI_USE_VERTEXAI=true,GOOGLE_CLOUD_PROJECT=$(gcloud config get-value project),GOOGLE_CLOUD_LOCATION=global"
```

**Option B — Attach `GEMINI_API_KEY` via Secret Manager:**

Create the secret once (if it does not already exist) and mount it onto the service:

```bash
printf '%s' "YOUR_GEMINI_API_KEY" | gcloud secrets create gemini-key --data-file=-
```

```bash
gcloud run services update pitch-arena \
  --region us-central1 \
  --set-secrets="GEMINI_API_KEY=gemini-key:latest"
```

---

## Bonus Demo: 50/50 Canary Traffic Splitting Between AI Personas

Deploy a tagged revision with a different `AI_PERSONA`, then split traffic 50/50 and watch cards from both revisions appear side-by-side in the feed:

```bash
gcloud run deploy pitch-arena \
  --source . \
  --region us-central1 \
  --set-env-vars="AI_PERSONA=overly enthusiastic optimist mentor from YC" \
  --tag=kind \
  --no-traffic
```

```bash
gcloud run services update-traffic pitch-arena \
  --region us-central1 \
  --to-revisions=LATEST=50,kind=50
```

---

## REST & WebSocket API Reference

| Endpoint | Method / Protocol | Description |
| --- | --- | --- |
| `/health` | `GET` | Health check returning `{ status, revision, activeSockets, aiMode }` |
| `/api/pitches` | `GET` | Returns `{ revision, pitches }` (most recent 30 pitches, newest first, with community comments shuffled) |
| `/api/pitch` | `POST` | Evaluates `{ author, pitch, source? }` (handle required), broadcasts `NEW_PITCH`, returns `201` JSON |
| `/api/pitches/:id/comments` | `POST` | Adds a community roast comment `{ author, text }` under the pinned AI roast, broadcasts `NEW_COMMENT`, returns `201` JSON |
| `/api/webhook/bot` | `POST` | Native Telegram Bot webhook (`Update` payload), broadcasts `NEW_PITCH` as `source: "bot"`, replies in Telegram via `sendMessage` |
| `/ws` | `WebSocket` | Sends `INIT` on connect, broadcasts `PRESENCE`, `NEW_PITCH`, & `NEW_COMMENT`, runs 30s ping/pong heartbeat |

### Test REST Ingestion & Attach a Telegram Bot

```bash
SERVICE_URL="https://YOUR-SERVICE-URL.a.run.app"

# 1. Direct REST pitch submission
curl -X POST "$SERVICE_URL/api/pitch" \
  -H "Content-Type: application/json" \
  -d '{"author":"cloudshell_hacker","pitch":"AI agent that negotiates your rent while you sleep","source":"curl"}'

# 2. Register your Telegram @BotFather token with the webhook
curl "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=$SERVICE_URL/api/webhook/bot"
```
