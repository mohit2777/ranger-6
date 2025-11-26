---
title: WhatsApp Multi-Automation
emoji: 💬
colorFrom: green
colorTo: blue
sdk: docker
app_port: 7860
---

# WhatsApp Multi-Automation Platform & n8n Integration Guide

A production-ready, multi-account WhatsApp automation backend with a real-time dashboard, Supabase persistence, a durable webhook delivery queue, and first-class n8n workflows for bi-directional automation.

---

## Why this repo exists
- Multi-account WhatsApp automation (QR onboarding, session persistence, multi-device support).
- Durable webhook delivery with retries, exponential backoff, and delivery metrics stored in Supabase.
- Secure dashboard (session auth, rate limiting, Helmet, compression) served by the same Express app.
- Drop-in n8n support: optimized outbound payloads plus a hardened `/api/webhook-reply` endpoint for replies.
- Deployment-ready configuration for Render/Railway/VPS, including health endpoints and graceful workers.

---

## Architecture at a glance
| Layer | Technology | Notes |
| --- | --- | --- |
| HTTP API & Dashboard | Node.js 18+ / Express / Socket.IO | Serves REST API, Socket-driven dashboard, QR auth, media uploads |
| WhatsApp clients | `whatsapp-web.js` (RemoteAuth/LocalAuth) | Manages multi-session instances and message dispatch |
| Data store | Supabase (Postgres) | Tables in `supabase-schema.sql`: accounts, webhooks, message_logs, `webhook_delivery_queue`, helper views |
| Webhook reliability | `utils/webhookDeliveryService.js` + `config/database.js` | Enqueues webhook deliveries, retries with exponential backoff, dead-lettering |
| Workflow engine | n8n | Receives webhook events and sends replies via `/api/webhook-reply` |
| Observability | Winston logs + `/api/health` | Queue depth, worker status, uptime, Supabase latency |

---

## Prerequisites
- Node.js **16+** (18+ recommended) and npm.
- Supabase project (free tier is fine) with access to SQL editor.
- WhatsApp account(s) for authentication.
- Optional: Docker (for n8n), Render/Railway/PM2 for deployment.

---

## 1. Local project setup
1. **Clone and install**
   ```bash
   git clone <your-repo-url>
   cd admin123
   npm install
   ```
2. **Prepare Supabase**
   - Open the Supabase SQL editor and run `supabase-schema.sql` (creates base tables + `webhook_delivery_queue`).
   - If migrating from an older schema, run `supabase-session-migration.sql` after the main script.
3. **Create environment file**
   ```bash
   cp .env.example .env
   ```
   Update the values you copied from Supabase plus dashboard credentials (see [Environment reference](#3-environment-reference)).
4. **Ensure session directories exist** (already committed but keep them writable): `sessions/`, `wa-sessions-temp/`, `logs/`.
5. **Run the app**
   ```bash
   npm run dev          # nodemon with live reload
   # or
   npm start            # production mode
   ```
6. **Verify health & dashboard**
   - Health check: `http://localhost:3000/api/health`
   - Login: `http://localhost:3000/login` → use the credentials from `.env`
   - Dashboard: `http://localhost:3000/dashboard`

> **Tip:** The first time a WhatsApp client spins up it downloads a Chromium build (~100 MB). Keep the session folders persistent so you do not have to re-scan QR codes every restart.

---

## 2. Database & webhook queue essentials
- `supabase-schema.sql` creates:
  - `whatsapp_accounts`, `webhooks`, `message_logs`, support views, and helper SQL functions.
  - `webhook_delivery_queue` table + indexes + triggers needed by `WebhookDeliveryService`.
- `config/database.js` supplies helper methods such as `enqueueWebhookDelivery`, `getDueWebhookDeliveries`, and `resetStuckWebhookDeliveries` (called at boot).
- `utils/webhookDeliveryService.js`:
  - Polls the queue (`WEBHOOK_WORKER_INTERVAL_MS`, default 3000 ms).
  - Claims jobs, posts via Axios with payload optimizations for n8n, writes success/failure logs back to Supabase.
  - Automatically disables itself if the queue table is missing—check server logs for the "Apply the new webhook queue schema" hint.

**After applying the SQL migration, restart the server** so the worker can reset stuck jobs and start polling.

---

## 3. Environment reference
| Variable | Purpose | Required | Default |
| --- | --- | --- | --- |
| `PORT` | HTTP port (Render supplies one via `$PORT`) | Optional | 3000 |
| `NODE_ENV` | `development` \| `production` | Optional | development |
| `DASHBOARD_USERNAME`, `DASHBOARD_PASSWORD` | Dashboard auth | ✅ | – |
| `SESSION_SECRET` | Express-session secret | ✅ | – |
| `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` | Supabase credentials (service key is used server-side only) | ✅ | – |
| `DEFAULT_WEBHOOK_URL` | Fallback when an account has no explicit webhook | Optional | – |
| `LOG_LEVEL`, `LOG_FILE` | Winston logging | Optional | `info`, `logs/app.log` |
| `RATE_LIMIT_WINDOW_MS`, `RATE_LIMIT_MAX_REQUESTS` | Express rate limiter | Optional | `900000`, `100` |
| `WA_MESSAGE_QUEUE_SIZE`, `WA_MESSAGE_BATCH_SIZE`, `WA_MESSAGE_BATCH_INTERVAL` | WhatsApp batching | Optional | `20`, `10`, `5000` |
| `CACHE_TTL`, `QUERY_CACHE_SIZE` | In-memory LRU cache | Optional | `60000`, `1000` |
| `WEBHOOK_WORKER_INTERVAL_MS`, `WEBHOOK_WORKER_BATCH_SIZE` | Queue polling cadence/batch | Optional | `3000`, `10` |
| `WEBHOOK_MAX_RETRIES`, `WEBHOOK_BACKOFF_MS`, `WEBHOOK_MAX_BACKOFF_MS` | Retry + exponential backoff controls | Optional | `5`, `2000`, `60000` |
| `KEEPALIVE_URL`, `KEEPALIVE_INTERVAL_MINUTES` | Optional uptime ping (Render/Railway) | Optional | unset, `14` |

Add any additional secrets (e.g., `N8N_WEBHOOK_SECRET`) if you plan to validate headers inside workflows. Set `KEEPALIVE_URL` to your deployed HTTPS domain if you want the server to self-ping every ~14 minutes (or adjust `KEEPALIVE_INTERVAL_MINUTES`) to prevent Render/Railway from idling the process.

---

## 4. Operating the platform
1. **Log in** at `/login` using the credentials from `.env`.
2. **Add an account** → scan the QR with WhatsApp → wait until the status flips to `ready`.
3. **Attach webhooks** per account; if the URL contains `n8n` or `nodemation` the payload is auto-optimized and the timeout drops from 10 s to 5 s.
4. **Send messages** via the dashboard (`/api/send`) or programmatically using REST endpoints (`/api/send`, `/api/send-media`).
5. **Monitor health** using `/api/health` (exposes worker state, queue depth, memory, Supabase latency) and by tailing `logs/combined.log` / `logs/error.log`.

---

## 5. n8n integration (receive + reply)
### 5.1 Receive WhatsApp events in n8n
1. Create a **Webhook** node → choose `Production URL`.
2. Copy the URL and add it under the account’s Webhooks tab in the dashboard. Optionally set a secret.
3. Deploy the n8n workflow.
4. Every incoming/outgoing WhatsApp event triggers the webhook queue, which delivers payloads like:
   ```json
   {
     "account_id": "a4f1...",
     "direction": "incoming",
     "sender": "447700900000",
     "recipient": "558199999999",
     "message": "Hello",
     "timestamp": 1731200000,
     "type": "chat",
     "chat_id": "447700900000-123@g.us",
     "is_group": false,
     "media": null,
     "optimized": true
   }
   ```
   Headers: `Content-Type: application/json`, `X-Webhook-Secret`, `X-Account-ID`, `User-Agent: WhatsApp-Multi-Automation/3.0`.

### 5.2 Reply to WhatsApp from n8n
Use the **HTTP Request** node pointing at `/api/webhook-reply`:
- **Method:** `POST`
- **URL:** (see networking notes below)
- **Headers:** `Content-Type: application/json`, plus `X-API-Key` or similar if you secured the endpoint.
- `/api/send` and `/api/send-media` remain available if you prefer those routes.

#### Step-by-step node setup (text replies)
1. Drag an **HTTP Request** node into your workflow.
2. Set **HTTP Method** to `POST` and paste the URL of your backend `/api/webhook-reply`.
3. Under **Authentication**, select the credential type you use for this API (or leave as "None" if unsecured).
4. Enable **Send Headers** and add:
   - `Content-Type: application/json`
   - Any custom secret header (`X-Webhook-Secret`, `X-API-Key`, etc.).
5. Switch **Body Content Type** to `JSON` and use the following structure:
   ```json
   {
     "sessionId": "={{$json.account_id}}",
     "phone": "={{$json.sender}}",
     "message": "Thanks for reaching out!"
   }
   ```
6. (Optional) Map additional keys like `options.replyToMessageId` or template variables depending on your automation.

#### Sending media via `/api/webhook-reply`
- Keep the same node but add an `options` object describing the media. Supported shortcuts:
  ```json
  {
    "sessionId": "={{$json.account_id}}",
    "phone": "={{$json.sender}}",
    "message": "Here is the file you requested",
    "options": {
      "mediaUrl": "https://example.com/path/invoice.pdf",
      "mediaMimeType": "application/pdf",
      "caption": "Invoice #{{$json.invoiceNumber}}"
    }
  }
  ```
- If you need raw base64 uploads, supply `options.mediaBase64` and `options.mediaFilename` instead of `mediaUrl`.
- The backend inspects `mediaMimeType` to decide whether to send image/video/audio/document correctly; make sure the type matches the actual file.

#### Using the legacy endpoints for specialized flows
- **`/api/send`** (text only):
  ```json
  {
    "account_id": "={{$json.account_id}}",
    "number": "={{$json.sender}}",
    "message": "Direct text via /api/send"
  }
  ```
- **`/api/send-media`** (advanced media control):
  ```json
  {
    "account_id": "={{$json.account_id}}",
    "number": "={{$json.sender}}",
    "media": {
      "url": "https://example.com/cat.mp4",
      "mimetype": "video/mp4",
      "filename": "cat.mp4"
    },
    "caption": "Video via /api/send-media"
  }
  ```
Use these endpoints when you need maximum control over MIME metadata or want to bypass the webhook secret workflow entirely.

### 5.3 n8n running inside Docker
| Scenario | URL to use in HTTP Request node |
| --- | --- |
| Backend also runs on your machine (Windows/macOS) | `http://host.docker.internal:3000/api/webhook-reply` |
| Backend on Linux host | Start n8n with `docker run --add-host=host.docker.internal:host-gateway ...` then use `http://host.docker.internal:3000/...` |
| Backend running in another container on the same Docker network | Reference the service name: `http://whatsapp-api:3000/api/webhook-reply` |
| Backend deployed on Render/Railway/VPS | Use the public HTTPS URL Render provides, e.g., `https://your-service.onrender.com/api/webhook-reply` |

Test connectivity from inside the container before running the workflow:
```bash
docker exec -it <n8n_container> curl -i http://host.docker.internal:3000/api/health
```
A 200/401 response proves the container can reach your backend; otherwise fix networking/tunnels first.

### 5.4 n8n hosted on Render while backend is local
Render cannot reach `localhost:3000` on your laptop. Either deploy this app to Render (binding to `$PORT`) or expose your local server via a tunnel (ngrok/Cloudflared) and use that HTTPS URL inside n8n.

---

## 6. Deployment checklist
- **Render / Railway / FlyIO**
  1. Connect the repo.
  2. Set the start command to `npm start` and add all environment variables (remember Supabase keys and worker tuning vars).
  3. Respect the platform’s injected `PORT` (the app already reads `process.env.PORT`).
  4. Ensure `sessions/`, `wa-sessions-temp/`, and `logs/` are persisted (Render persistent disk, Railway volume, S3, etc.) if you need long-lived WhatsApp sessions.
  5. Optional: set `KEEPALIVE_URL` (and optionally `KEEPALIVE_INTERVAL_MINUTES`) so the app pings itself every 14 minutes to avoid platform sleep.
- **VPS / PM2**
  ```bash
  npm install --production
  pm2 start index.js --name whatsapp-automation
  pm2 save
  pm2 startup
  ```
- **Health probes**
  - HTTP GET `/api/health` → 200 when healthy, 500 when worker disabled/misconfigured.
  - Optional: add uptime monitors hitting `/api/health` and `/api/stats`.

---

## 7. Troubleshooting quick reference
| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `WebhookDeliveryService disabled: Missing database table` | SQL migration not applied | Run `supabase-schema.sql`, redeploy, confirm table exists |
| n8n HTTP Request returns 502 | Backend unreachable (wrong URL, not deployed, Docker cannot reach host) | Use `host.docker.internal`, deploy the API, or expose via tunnel |
| WhatsApp session keeps expiring | Sessions folder not persisted or multiple instances share same session ID | Map `sessions/` to persistent storage per instance |
| `/api/webhook-reply` returns 401/403 | Missing/incorrect secret header | Align n8n headers with backend validation logic |
| Render app crashes with `EADDRINUSE` | Another process bound to `$PORT` | Ensure only this process listens on the provided port |

Logs live in `logs/combined.log` and `logs/error.log`. Tail them or stream to your logging stack of choice.

---

## 8. Useful npm scripts
| Script | Description |
| --- | --- |
| `npm run dev` | Start the API with `nodemon` (auto-restart on file changes) |
| `npm start` | Production start (plain Node) |
| `npm test` | Runs `test.js` harness (extend with your own diagnostics) |

Feel free to add linting/tests to suit your workflow.

---

## 9. License
MIT License. Use responsibly; WhatsApp Web automation is not officially supported by Meta, so comply with their Terms of Service.
