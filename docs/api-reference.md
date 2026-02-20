# API Reference — JobHuntr Service Gateway

**Base URL:** `https://jobhuntr-service-gateway-production.up.railway.app`

---

## Public endpoints (no auth)

### `GET /api/health`

Health check for load balancers.

**Response:**
```json
{ "status": "ok" }
```

---

### `GET /api/stats`

Public stats for the landing page.

**Response:**
```json
{
  "contributors": 42,
  "jobs_last_24h": 318,
  "jobs_returned_read_by_agent": 1200
}
```

| Field | Type | Description |
|---|---|---|
| `contributors` | number | Unique contributors in the system |
| `jobs_last_24h` | number | Jobs in the current 24h sliding window |
| `jobs_returned_read_by_agent` | number | Total jobs ever queried by agents (optional) |

---

## Auth endpoints

### `POST /api/auth/login`

```json
// Request
{ "email": "user@example.com", "password": "secret" }

// Response
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

### `POST /api/auth/refresh`

```json
// Request
{ "refresh_token": "eyJ..." }

// Response
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ..."   // optional rotation
}
```

Tokens carry a standard JWT `exp` claim. Refresh 5 minutes before expiry.

---

## Authenticated endpoints

All authenticated endpoints require:

```
Authorization: Bearer <access_token>
```

### `GET /api/jobs?page=1`

Fetch the current 24h job feed.

| Param | Type | Default | Description |
|---|---|---|---|
| `page` | number | 1 | Page number (1-indexed) |

Page 1 is publicly viewable. Pages 2+ require contributor or subscriber access.

**Response:**
```json
{
  "jobs": [
    {
      "id": "abc123",
      "title": "Senior Software Engineer",
      "company": "Acme Corp",
      "location": "Remote",
      "url": "https://acme.com/jobs/senior-swe",
      "posted_at": "2026-02-20T10:00:00Z"
    }
  ],
  "total": 318,
  "page": 1,
  "per_page": 20
}
```

---

### `POST /api/jobs/contribute`

Submit job URLs to the board. Contribute ≥ 5 unique URLs in any 24h window to unlock full feed access.

```json
// Request
{
  "urls": [
    "https://company.com/careers/role-1",
    "https://company.com/careers/role-2"
  ]
}

// Response
{
  "accepted": 2,
  "rejected": 0,
  "contributor_count_24h": 3
}
```

---

### `POST /api/queue`

Save a job to your personal queue (persists beyond 24h window).

```json
// Request
{ "job_id": "abc123" }

// Response
{ "queued": true }
```

Rate limit: **10 queue actions per minute per user**.

---

### `GET /api/queue`

Fetch your saved job queue.

```json
{
  "queued_jobs": [ ... ]
}
```
