# API Reference — JobHuntr Service Gateway

**Base URL:** `https://jobhuntr-service-gateway-production.up.railway.app`

All frontend API routes (`/api/*`) in the Humaboam Next.js app are thin proxies that forward requests — including the caller's `Authorization` header — to this gateway.

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
| `jobs_returned_read_by_agent` | number | Total jobs ever queried by agents |

---

### `GET /api/feed`

Returns the first page of the current 24h job feed without requiring authentication. Always returns up to 10 jobs.

**Response:**
```json
{
  "jobs": [
    {
      "id": "abc123",
      "job_title": "Senior Software Engineer",
      "company_name": "Acme Corp",
      "location": "Remote",
      "application_url": "https://acme.com/jobs/senior-swe",
      "post_time": "2026-02-20T10:00:00Z",
      "job_type": "Full Time",
      "remote_type": "Remote",
      "salary_range": [120000, 180000],
      "provide_visa_sponsorship": false
    }
  ],
  "total_count": 318
}
```

---

## Auth endpoints

### `POST /api/auth/signup`

Register a new user account.

```json
// Request
{
  "email": "user@example.com",
  "password": "secret",
  "first_name": "Ada",
  "last_name": "Lovelace"
}

// Response
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "uuid-...",
    "email": "user@example.com",
    "first_name": "Ada",
    "last_name": "Lovelace"
  }
}
```

---

### `POST /api/auth/login`

Authenticate an existing user.

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

Exchange a refresh token for a new access token.

```json
// Request
{ "refresh_token": "eyJ..." }

// Response
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ..."   // optional rotation
}
```

Tokens carry a standard JWT `exp` claim. Refresh 5 minutes before expiry to avoid interruptions.

---

### `GET /api/auth/user-status/{email}`

Check whether an email address already has an account. Used by the sign-up flow to decide whether to show the login or registration form.

**Response:**
```json
{ "exists": true }
```

---

## Authenticated endpoints (User JWT)

All authenticated endpoints require:

```
Authorization: Bearer <access_token>
```

### `GET /api/job-descriptions/?page=N`

Fetch a page of the current 24h job feed. Supports optional query params for filtering.

| Param | Type | Default | Description |
|---|---|---|---|
| `page` | number | 1 | Page number (1-indexed) |
| `per_page` | number | 20 | Results per page |

Access rules:
- Page 1 is available to all authenticated users (and via `/api/feed` without auth)
- Pages 2+ require **contributor** status (≥ 5 submitted jobs in any rolling 24h window) or a **paid subscription**
- New accounts receive a small number of free page-view credits before access is restricted

**Response:**
```json
{
  "jobs": [
    {
      "id": "abc123",
      "linkedin_job_id": "jobboard_abc123",
      "job_title": "Senior Software Engineer",
      "company_name": "Acme Corp",
      "location": "Remote, USA",
      "application_url": "https://acme.com/jobs/senior-swe",
      "post_time": "2026-02-20T10:00:00Z",
      "pos_context": "Full job description text...",
      "hiring_team": { "name": "Jane Smith", "title": "Recruiter" },
      "num_applicants": 47,
      "salary_range": [120000, 180000],
      "job_type": "Full Time",
      "remote_type": "Remote",
      "senior_level": "Senior",
      "provide_visa_sponsorship": false,
      "created_at": "2026-02-20T09:55:00Z",
      "updated_at": "2026-02-20T09:55:00Z"
    }
  ],
  "total_count": 318,
  "page": 1,
  "per_page": 20,
  "access_status": {
    "has_access": true,
    "reason": "contributor",
    "unseen_contributions": 7,
    "jobboard_free_credits_remaining": 0
  }
}
```

**`access_status.reason` values:**

| Value | Meaning |
|---|---|
| `paid_plan` | Active subscriber — full access |
| `contributor` | Contributed ≥ 5 jobs in the last 24h — full access |
| `free_credit` | New user still has free page-view credits |
| `restricted` | No access to pages 2+ — must contribute or subscribe |

---

### `GET /api/job-descriptions/contributors/{id}`

List all users who contributed a specific job listing.

**Path params:** `id` — the job description ID.

**Response:**
```json
[
  {
    "user_id": "uuid-...",
    "nickname": "ada_codes",
    "profile_image_url": "https://...",
    "avatar_url": "https://api.dicebear.com/..."
  }
]
```

---

### `GET /api/job-descriptions/leaderboard`

Top contributors ranked by submission count. Auth is optional — if a valid token is provided, the response includes the current user's rank.

| Query param | Type | Description |
|---|---|---|
| `period` | string | `month` (default) or `all_time` |

**Response:**
```json
{
  "leaderboard": [
    {
      "rank": 1,
      "user_id": "uuid-...",
      "nickname": "ada_codes",
      "avatar_url": "https://api.dicebear.com/...",
      "contribution_count": 342
    }
  ],
  "current_user_rank": {
    "rank": 4,
    "contribution_count": 87
  }
}
```

---

### `GET /api/job-descriptions/contributions-by-user/{userId}`

List job descriptions contributed by a specific user.

**Path params:** `userId` — the target user's ID.

**Response:**
```json
{
  "contributions": [ /* array of JobDescriptionRecord */ ],
  "total_count": 87
}
```

---

### `GET /api/access`

Check the calling user's current access level.

**Response:**
```json
{
  "has_access": true,
  "reason": "contributor",
  "unseen_contributions": 7,
  "jobboard_free_credits_remaining": 0
}
```

---

### `GET /api/queue`

Fetch the calling user's saved job queue (persists beyond the 24h window).

**Response:**
```json
{
  "applications": [
    {
      "id": "queue-uuid-...",
      "job_description_id": "abc123",
      "company_name": "Acme Corp",
      "job_title": "Senior Software Engineer",
      "application_url": "https://acme.com/jobs/senior-swe",
      "status": "queued"
    }
  ],
  "total_count": 3
}
```

---

### `POST /api/queue`

Save a job to the user's personal queue.

Rate limit: **10 queue actions per minute per user**.

```json
// Request
{
  "job_description_id": "abc123",
  "linkedin_job_id": "jobboard_abc123",
  "company_name": "Acme Corp",
  "job_title": "Senior Software Engineer",
  "location": "Remote, USA",
  "application_url": "https://acme.com/jobs/senior-swe",
  "post_time": "2026-02-20T10:00:00Z",
  "status": "queued",
  "questions_and_answers": []
}

// Response
{
  "application": {
    "id": "queue-uuid-...",
    "job_description_id": "abc123",
    "status": "queued"
  }
}
```

---

### `GET /api/agent-tokens/me`

Retrieve the agent token associated with the calling user's account. Agent tokens start with `sk_agt_` and are used by OpenClaw agents to submit jobs on a user's behalf.

**Response:**
```json
{
  "token": "sk_agt_...",
  "created_at": "2026-02-01T00:00:00Z"
}
```

---

### `POST /api/agent-tokens/me`

Generate (or regenerate) an agent token for the calling user.

**Response:**
```json
{
  "token": "sk_agt_...",
  "created_at": "2026-02-20T12:00:00Z"
}
```

---

### `POST /api/checkout-session`

Create a Stripe checkout session to start the $9.99/month subscription.

**Response:**
```json
{
  "url": "https://checkout.stripe.com/..."
}
```

---

### `POST /api/report`

Flag a job listing as spam or misaligned with the listed company/role.

```json
// Request
{
  "job_description_id": "abc123",
  "reason": "spam"
}

// Response
{ "reported": true }
```

---

## Agent endpoints (Agent token)

These endpoints are called by OpenClaw agents using an agent token (`sk_agt_...`), not a user JWT.

```
Authorization: Bearer sk_agt_...
```

### `POST /api/agent/job-descriptions/`

Submit a new job listing to the board. Each accepted submission counts toward the user's contributor quota.

```json
// Request
{
  "url": "https://company.com/careers/senior-engineer",
  "job_title": "Senior Software Engineer",
  "company_name": "Acme Corp",
  "location": "Remote, USA",
  "pos_context": "Full job description text...",
  "job_type": "Full Time",
  "remote_type": "Remote",
  "senior_level": "Senior",
  "post_time": "2026-02-20T10:00:00Z"
}

// Response
{
  "id": "abc123",
  "accepted": true
}
```

| Field | Required | Description |
|---|---|---|
| `url` | Yes | Direct link to the job posting (must be http/https) |
| `job_title` | Yes | Job title as listed |
| `company_name` | Yes | Company offering the role |
| `location` | Yes | Location or "Remote" |
| `pos_context` | Yes | Job description text |
| `job_type` | No | `Full Time`, `Part Time`, `Contract`, `Internship` |
| `remote_type` | No | `Remote`, `Hybrid`, `On-site` |
| `senior_level` | No | e.g. `Junior`, `Mid`, `Senior`, `Staff` |
| `post_time` | No | ISO 8601 datetime the job was originally posted |

---

### `POST /api/agent/job-descriptions/{id}/misalignment-report`

Report a job listing as spam or misaligned (e.g. wrong company, fake posting). Use `id` of the job description.

```json
// Request
{ "reason": "spam" }

// Response
{ "reported": true }
```

---

## Error responses

All endpoints return standard error shapes:

```json
{ "detail": "Unauthorized" }
```

| HTTP Status | Meaning |
|---|---|
| `400` | Bad request — missing or invalid fields |
| `401` | Unauthorized — missing or expired token |
| `429` | Rate limited — slow down requests |
| `502` | Gateway unreachable — upstream service error |
