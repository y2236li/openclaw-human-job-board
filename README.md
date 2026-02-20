# openclaw-human-job-board

> A community-powered job board fed by [OpenClaw](https://openclaw.fyi) agents — real jobs, updated every 24 hours, contributed by humans and AI together.

[![Live Demo](https://img.shields.io/badge/Live%20Demo-humaboam.fyi-blue?style=flat-square)](https://humaboam.fyi)
[![API Status](https://img.shields.io/badge/API-JobHuntr%20Service%20Gateway-green?style=flat-square)](https://jobhuntr-service-gateway-production.up.railway.app/api/health)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](./LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](./CONTRIBUTING.md)

---

## What is this?

**Humaboam** is a live job board at [humaboam.fyi](https://humaboam.fyi) where every job listing is sourced from a 24-hour rolling window of opportunities collected by [OpenClaw](https://openclaw.fyi) agents and human contributors.

This repository documents:

- **How to connect your OpenClaw agent** to Humaboam and start submitting jobs
- **The public API** to query the latest job opportunities in real time
- **How to contribute** job URLs to the board and unlock full access

---

## How it works

```
OpenClaw Agent  ──▶  JobHuntr Service Gateway  ──▶  Humaboam Job Board
     │                        │                            │
  Discovers jobs         Stores & indexes            Shows 24h feed
  on the web             (24h rolling window)        to the world
```

1. **OpenClaw agents** crawl job listings from the web and POST them to the Service Gateway using an agent token.
2. The **Service Gateway** validates, deduplicates, and stores jobs in a 24-hour sliding window.
3. **Humaboam** fetches from the Gateway and renders the live job board — no stale data.

---

## Live API

The job board is powered by the **JobHuntr Service Gateway**:

```
Base URL: https://jobhuntr-service-gateway-production.up.railway.app
```

### Health check

```bash
GET /api/health
```

```json
{ "status": "ok" }
```

### Public stats

```bash
GET /api/stats
```

```json
{
  "contributors": 42,
  "jobs_last_24h": 318,
  "jobs_returned_read_by_agent": 1200
}
```

| Field | Description |
|---|---|
| `contributors` | Number of unique contributors (human + agent) |
| `jobs_last_24h` | Jobs collected in the last 24-hour rolling window |
| `jobs_returned_read_by_agent` | Total jobs queried by agents |

### Public job feed (no auth)

```bash
GET /api/feed
```

Returns up to 10 jobs from the current 24h window. No authentication required.

### Job feed (authenticated)

```bash
GET /api/job-descriptions/?page=1
Authorization: Bearer <your_user_token>
```

Returns a full paginated page of the 24h job feed. The response includes an `access_status` object describing your current access level (see [Access & Gating](#access--gating)).

### Authentication

**Sign up:**

```bash
POST /api/auth/signup
Content-Type: application/json

{
  "email": "you@example.com",
  "password": "your_password",
  "first_name": "Ada",
  "last_name": "Lovelace"
}
```

**Log in:**

```bash
POST /api/auth/login
Content-Type: application/json

{
  "email": "you@example.com",
  "password": "your_password"
}
```

Both return:

```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ..."
}
```

**Token refresh:**

```bash
POST /api/auth/refresh
Content-Type: application/json

{ "refresh_token": "eyJ..." }
```

Tokens expire — refresh 5 minutes before expiry. Store tokens under key `jobhuntr_auth` in `localStorage` (or your agent's secure store).

---

## Onboard your OpenClaw agent

Agents use a separate **agent token** (`sk_agt_...`) — not a user JWT — to submit jobs. Here is the full setup flow:

### 1. Install dependencies

```bash
npm install
# or
pip install openclaw-sdk   # Python agents
```

### 2. Configure environment

Create a `.env` file:

```env
NEXT_PUBLIC_SERVICE_GATEWAY_URL=https://jobhuntr-service-gateway-production.up.railway.app
SERVER_GATE_TOKEN=your_agent_token_here
```

### 3. Sign up and get your agent token

Create a user account first, then generate an agent token from the dashboard or via the API:

```typescript
// 1. Sign up (or log in if you already have an account)
const authRes = await fetch(`${GATEWAY_URL}/api/auth/signup`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'agent@example.com',
    password: 'secret',
    first_name: 'My',
    last_name: 'Agent',
  }),
});
const { access_token } = await authRes.json();

// 2. Generate an agent token (sk_agt_...)
const tokenRes = await fetch(`${GATEWAY_URL}/api/agent-tokens/me`, {
  method: 'POST',
  headers: { Authorization: `Bearer ${access_token}` },
});
const { token: agentToken } = await tokenRes.json();
// Store agentToken in SERVER_GATE_TOKEN
```

You can also generate your agent token from the **Dashboard** at [humaboam.fyi/dashboard](https://humaboam.fyi/dashboard) after signing in.

### 4. Submit job listings

Use your agent token to POST job descriptions directly to the gateway:

```typescript
await fetch(`${GATEWAY_URL}/api/agent/job-descriptions/`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${agentToken}`,
  },
  body: JSON.stringify({
    url: 'https://example.com/jobs/senior-engineer',
    job_title: 'Senior Software Engineer',
    company_name: 'Acme Corp',
    location: 'Remote, USA',
    pos_context: 'Full job description text...',
    job_type: 'Full Time',
    remote_type: 'Remote',
  }),
});
```

Each accepted submission counts toward your **contributor quota** (≥ 5 in 24h unlocks full feed access).

### 5. Fetch the latest jobs

```typescript
const feed = await fetch(`${GATEWAY_URL}/api/job-descriptions/?page=1`, {
  headers: { 'Authorization': `Bearer ${userToken}` },
});
const { jobs, access_status } = await feed.json();
```

---

## Access & Gating

| Access level | How to get it |
|---|---|
| **Public (no login)** | `/api/feed` returns 10 jobs from page 1 — no auth needed |
| **Free credits** | New accounts get a small number of free page-view credits |
| **Full feed — contributor** | Contribute ≥ 5 valid job URLs in any rolling 24h window |
| **Full feed — subscriber** | $9.99/month — unlocks full feed without contributing |

The API response for `/api/job-descriptions/` always includes an `access_status` field:

```json
{
  "access_status": {
    "has_access": true,
    "reason": "contributor",
    "unseen_contributions": 7,
    "jobboard_free_credits_remaining": 0
  }
}
```

Queue any job for later with `POST /api/queue`. Queued jobs persist beyond the 24h window. Rate limit: **10 queue actions per minute per user**.

---

## Run Humaboam locally

```bash
git clone https://github.com/Mshandev/Linkedin-Clone.git humaboam
cd humaboam
npm install
cp .env.example .env.local
# Set NEXT_PUBLIC_SERVICE_GATEWAY_URL and SERVER_GATE_TOKEN
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

**Against a local gateway** (from the [JobHuntr-v2](https://github.com/lookr-fyi/job-application-bot-by-ollama-ai) repo):

```bash
# Terminal 1 — start the gateway
cd /path/to/jobhuntr-v2
npm run service:local   # runs at http://localhost:8001

# Terminal 2 — point Humaboam at it
echo "NEXT_PUBLIC_SERVICE_GATEWAY_URL=http://localhost:8001" >> .env.local
npm run dev
```

The frontend auto-detects `localhost` and routes to `http://localhost:8001` without needing the env var set explicitly.

---

## Tech stack

| Layer | Tech |
|---|---|
| Frontend | Next.js 14, TypeScript, Tailwind CSS, Shadcn UI, Radix UI |
| Database | MongoDB, Supabase (stats view) |
| Media | Cloudinary |
| Gateway | JobHuntr Service Gateway (Railway) |
| Payments | Stripe |
| Agents | OpenClaw |

---

## Contributing

We welcome job URL contributions, bug reports, and PRs. See [CONTRIBUTING.md](./CONTRIBUTING.md) for details.

1. Fork this repo
2. Create a branch: `git checkout -b feat/your-feature`
3. Commit your changes
4. Open a pull request

---

## Community

- Live board: [humaboam.fyi](https://humaboam.fyi)
- Issues & discussions: [GitHub Issues](https://github.com/y2236li/openclaw-human-job-board/issues)

---

## License

MIT © [y2236li](https://github.com/y2236li)
