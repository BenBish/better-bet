# Better Bet

Sports betting picks management system with agent API, real-time activity feed, and analytics dashboard.

**Stack:** Bun + Hono + SQLite (backend) · Vite + React + TanStack Router + Tailwind + shadcn/ui (frontend)

---

## Quick Start

```bash
# 1. Set up backend config
cp backend/.env.example backend/.env   # edit password + session secret

# 2. Install dependencies
bun install

# 3. Run both servers (backend :3000, frontend :5173)
bun run dev
```

Navigate to http://localhost:5173 and log in with the password from `backend/.env`.

---

## Commands

| Command | Description |
|---|---|
| `bun run dev` | Both servers (concurrently) |
| `bun run dev:backend` | Backend only (port 3000) |
| `bun run dev:frontend` | Frontend only (port 5173) |
| `bun run build` | Typecheck backend + build frontend |
| `bun run start:backend` | Production backend |
| `bun run start:frontend` | Production frontend preview |
| `bun run clean` | Remove SQLite database files |
| `cd backend && bun test` | Run backend tests |

---

## Architecture

```
backend/                    frontend/
├── src/                    ├── src/
│   ├── server.ts           │   ├── main.tsx
│   ├── index.ts            │   ├── router.ts
│   ├── lib/                │   ├── lib/api.ts
│   │   ├── db.ts           │   ├── components/
│   │   ├── picks-service.ts│   │   ├── ui/           # shadcn/ui primitives
│   │   ├── agent-service.ts│   │   ├── PicksPage.tsx
│   │   ├── analytics-svc.ts│   │   ├── AnalyticsPage.tsx
│   │   ├── activity-svc.ts │   │   ├── AgentsPage.tsx
│   │   ├── sse-emitter.ts  │   │   ├── LoginPage.tsx
│   │   ├── calculations.ts │   │   └── ActivityFeed.tsx
│   │   └── validations.ts  │   └── routes/
│   ├── middleware/
│   │   ├── auth.ts
│   │   └── rate-limit.ts
│   └── migrations/
└── .env                    └── vite.config.ts
```

- **Admin routes** — session-auth via HttpOnly cookie
- **Agent routes** — Bearer token auth with per-agent rate limiting (100 req/min)
- **SSE** — real-time activity feed at `GET /api/activity/stream`
- **Database** — SQLite via `bun:sqlite`, WAL mode, auto-migrated on startup

---

## How to Add a New Agent

### Step 1: Create the agent in the UI

1. Navigate to **Agents** page
2. Type the agent name and click **Create Agent**
3. **Copy the API key** — it's shown once and starts with `spk_`

> Alternatively, via curl:
> ```bash
> curl -X POST http://localhost:3000/api/admin/agents \
>   -H "Content-Type: application/json" \
>   -b "session=YOUR_SESSION_COOKIE" \
>   -d '{"name": "NewBot"}'
> ```
> Response: `{ "agent": { ... }, "key": "spk_abc123..." }`

### Step 2: Configure your agent

Give your agent the API key and point it at the picks endpoint. The agent should send picks via:

```
POST http://localhost:3000/api/agent/picks
Authorization: Bearer spk_abc123...
Content-Type: application/json
```

### Step 3: Agent prompt

Paste this into your agent's instructions:

---

> **You are a betting picks agent. Your job is to analyze sports matches and submit picks to a betting tracker API.**
>
> **API Endpoint:** `POST http://localhost:3000/api/agent/picks`
> **Auth Header:** `Authorization: Bearer spk_YOUR_KEY_HERE`
>
> **Request body (JSON):**
> ```json
> {
>   "match_date": "2026-07-05T15:00",
>   "competition": "Premier League",
>   "home_team": "Arsenal",
>   "away_team": "Chelsea",
>   "market": "Moneyline",
>   "selection": "Arsenal",
>   "recommended_odds": 2.10,
>   "stake": 10,
>   "notes": "Arsenal strong at home, Chelsea missing key defender"
> }
> ```
>
> **Field rules:**
> - `match_date` — ISO datetime string (e.g. `2026-07-05T15:00`)
> - `competition` — optional, e.g. "Premier League", "La Liga", "Champions League"
> - `home_team`, `away_team` — must not be the same team
> - `market` — bet type: "Moneyline", "Spread", "Over", "Under", "Total", etc.
> - `selection` — your specific pick: "Arsenal", "Over 2.5", "PSG -1.5", etc.
> - `recommended_odds` — decimal odds, must be > 1.0
> - `stake` — unit stake (default 1 if omitted)
> - `notes` — optional reasoning
>
> **Batch mode (up to 50 picks):**
> ```
> POST http://localhost:3000/api/agent/picks/batch
> ```
> ```json
> {
>   "picks": [
>     { "match_date": "2026-07-05T15:00", "home_team": "Arsenal", "away_team": "Chelsea", "market": "Moneyline", "selection": "Arsenal", "recommended_odds": 2.10, "stake": 10 },
>     { "match_date": "2026-07-05T17:30", "home_team": "Liverpool", "away_team": "Man City", "market": "Over", "selection": "Over 2.5", "recommended_odds": 1.85, "stake": 5 }
>   ]
> }
> ```
> Returns 207 Multi-Status with per-item results.
>
> **Update closing lines (batch):**
> ```
> POST http://localhost:3000/api/agent/picks/batch-closing-lines
> ```
> ```json
> { "updates": [ { "id": "pick-id-1", "closing_odds": 1.90 }, { "id": "pick-id-2", "closing_odds": 1.75 } ] }
> ```
>
> **Settle results (batch):**
> ```
> POST http://localhost:3000/api/agent/picks/batch-results
> ```
> ```json
> { "updates": [ { "id": "pick-id-1", "result": "won" }, { "id": "pick-id-2", "result": "lost" } ] }
> ```
> Valid results: `"won"`, `"lost"`, `"push"`, `"void"`
>
> **Rate limit:** 100 requests per minute. Use batch endpoints to stay under the limit.
>
> **Error handling:** If you receive a 400 error, check the response body for validation details. If you receive 429, wait before retrying.

---

### Step 4: Verify the agent is working

1. Check the **Activity** page for real-time entries as the agent submits picks
2. Check the **Picks** page — filter by the new agent name
3. Check the **Analytics** page — per-agent breakdown updates automatically

### Managing agents

| Action | How |
|---|---|
| **Rename** | Agents page → click ✏️ on agent card |
| **Deactivate** | Agents page → toggle "Active" switch off (soft-delete, preserves history) |
| **Reactivate** | Toggle "Active" back on |
| **Rotate key** | Agents page → click "Rotate Key" (old key immediately revoked) |
| **Delete** | Agents page → click "Delete" (soft-delete, can be reactivated) |

---

## API Endpoints

### Auth
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/login` | none | `{ "password": "..." }` → sets session cookie |
| POST | `/api/logout` | session | Clears session cookie |

### Admin (session-auth)
| Method | Path | Description |
|---|---|---|
| GET/POST | `/api/admin/agents` | List / create agents |
| GET/PUT/DELETE | `/api/admin/agents/:id` | Agent CRUD |
| POST | `/api/admin/agents/:id/rotate-key` | Rotate agent key |
| GET/POST | `/api/admin/picks` | List / create picks |
| GET/PUT/DELETE | `/api/admin/picks/:id` | Pick CRUD |
| PUT | `/api/admin/picks/:id/closing-line` | Set closing odds |
| PUT | `/api/admin/picks/:id/result` | Settle pick |
| GET | `/api/admin/analytics` | Overall stats |
| GET | `/api/admin/analytics/by-agent` | Per-agent breakdown |
| GET | `/api/admin/analytics/by-market` | Per-market breakdown |
| GET | `/api/admin/analytics/by-competition` | Per-competition breakdown |
| GET | `/api/admin/analytics/daily-pnl` | Daily P&L data |
| GET | `/api/admin/export/csv` | CSV export |
| GET | `/api/activity` | Paginated activity log |
| GET | `/api/activity/stream` | SSE real-time activity feed |

### Agent API (Bearer auth, rate limited)
| Method | Path | Description |
|---|---|---|
| POST | `/api/agent/picks` | Create single pick |
| POST | `/api/agent/picks/batch` | Batch create (207 Multi-Status) |
| POST | `/api/agent/picks/batch-closing-lines` | Batch closing lines (207) |
| POST | `/api/agent/picks/batch-results` | Batch settle results (207) |

### Public
| Method | Path | Description |
|---|---|---|
| GET | `/api/health` | Health check |

---

## Database

- **Engine:** `bun:sqlite`
- **File:** `backend/data/picks.db`
- **Tables:** `picks`, `agents`, `activity_log`, `schema_migrations`
- **Migrations:** Auto-run on first connection via `backend/src/migrations/`
- **Reset:** `bun run clean` removes all DB files

## Barge

Try [Barge](https://getbarge.io/?utm_source=github&utm_medium=referral&utm_campaign=barge&utm_content=BenBish-better-bet-readme-text), a cloud runtime environment for your agents.
