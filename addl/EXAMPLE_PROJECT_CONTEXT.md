# TaskFlow — Project Intelligence Document

> This is the living SDLC record of this project. It captures not just
> what was built, but why every decision was made, what was explored and
> abandoned, what failed and what was learned. Written so that any future
> engineer or AI model can achieve full context from this document alone.

**Started**: 2026-05-10
**Last Updated**: 2026-06-03
**Current Phase**: Development
**Status**: MVP auth complete. Task board in progress. Slack OAuth blocked on callback issue (now resolved).

---

## 1. Why This Exists — The Problem

**Problem Statement**:
Teams using Slack + Notion for task management lose the reasoning behind
tasks constantly. A task gets created from a Slack thread, documented in
Notion, but the conversation that spawned it — the why — is severed. By
the time someone picks up the task, the context is gone.

**Who has this problem**:
Small engineering teams (3-12 people) working async. Technically literate
but unwilling to adopt heavyweight PM tools. They live in Slack and need
tasks to feel native to where they already work.

**Current state / workarounds**:
- Pinning Slack messages (breaks when accidentally unpinned)
- Notion databases (disconnected from where decisions happen)
- GitHub Issues (excludes non-technical stakeholders)

**Why now**:
Slack deprecated its own task features in 2025. Teams are actively
looking. Window is real and closing as competitors move in.

---

## 2. What We're Building

**Vision**:
A task manager that lives where teams already talk — carrying context from
conversations, not severing it.

**Core value proposition**:
Every task carries the thread that created it. You always know why it
exists and who asked for it.

**In scope**:
- Slack integration (create tasks from messages)
- Kanban board (web)
- Thread-linked task context
- Team workspaces with multi-user access

**Explicit Non-Goals** (out of scope — and why):
- **Time tracking** — Adds complexity without serving the core value. Teams
  that need time tracking have already bought Harvest or Toggl. Not our fight.
- **Gantt / timeline views** — Fundamentally different audience (PMs, not
  dev teams). Would pull product direction away from the core user.
- **Mobile native app** — Phase 2 at earliest. Web-first validates the model
  before native investment.
- **AI summarization of threads** — Interesting but distracts from getting
  the core flow working. Explicitly deferred.

**Success looks like**:
A team of 5 replaces their Notion task board with TaskFlow within 2 weeks
and does not go back. Retention at 30 days is the signal.

---

## 3. Who It's For

**Primary users**:
Engineering leads and developers at seed-stage startups. Comfortable with
CLI tools. Will read short docs. Won't read long docs. Opinionated about
tooling and fast to abandon things that feel clunky.

**Secondary users / stakeholders**:
Non-technical co-founders who need visibility into engineering work without
needing to learn Jira.

**User assumptions**:
- Has an existing Slack workspace
- Comfortable with OAuth login flows
- Expects fast load times — will abandon if > 2s initial load
- Not willing to reconfigure how they use Slack for us

---

## 4. Non-Functional Requirements

| Requirement | Target | Rationale |
|---|---|---|
| Page load time | < 2s | User assumption — will abandon if slower |
| API response time | < 300ms p95 | Board must feel instant |
| Uptime | 99.5% | Startup tolerance; not mission-critical yet |
| Concurrent users | 500 at launch | Conservative — revisit at 200 active teams |
| Data retention | Indefinite | Tasks are permanent records even if threads expire |
| Security classification | Low-medium | No PII beyond email; no financial data |

---

## 5. Architecture

**System overview**:
Next.js frontend on Vercel. FastAPI backend on Railway. PostgreSQL database.
Slack webhooks received by backend, processed async, stored, surfaced via
REST API to frontend. Auth is stateless JWT.

**Component breakdown**:
- `web/` — Next.js app (task board, settings, auth pages)
- `api/` — FastAPI (REST API, Slack event handler, auth endpoints)
- `db/` — PostgreSQL via SQLAlchemy ORM

**Data flow**:
Slack event → FastAPI webhook handler (ack immediately) → background task
processes event → Task record written to PostgreSQL → Next.js polls REST
API on 5s interval → renders board.

> Note: Originally designed with WebSocket for real-time updates. Reverted
> to polling on 2026-05-28 after Railway's 30s WS timeout made it
> unworkable. See § Exploration Journal.

**Key boundaries**:
- Slack webhook handling isolated in its own router (`/slack/*`). Never
  touches auth logic directly. This is intentional — Slack events can
  arrive unauthenticated and must be verified separately.
- Frontend never queries database. Always through API. No exceptions.

---

## 6. Tech Stack

| Layer | Technology | Why this choice | Risk |
|---|---|---|---|
| Frontend | Next.js 14 | App Router + RSC, fast deploys via Vercel, familiar | Rapid Next.js changes may require migration effort |
| Backend | FastAPI | Async, fast, strong Python ecosystem, team familiarity | Less opinionated than Django — we own more decisions |
| Database | PostgreSQL | Relational, battle-tested, Supabase-compatible for future | None significant |
| Auth | JWT (access + refresh) | Stateless — works for future mobile, scales horizontally | Cannot revoke access token before expiry; mitigated by short TTL |
| Infrastructure | Vercel + Railway | Fast deploys, low ops overhead for current team size | Railway is a smaller vendor — revisit if they change pricing |
| Testing | Pytest + Playwright | API unit tests + E2E for core journeys | Playwright tests are slow — keep E2E set small |

---

## 7. Data Model

**Key entities**:
- `users` — account holders
- `workspaces` — team containers, users belong to many
- `tasks` — core unit of work, belongs to one workspace
- `slack_threads` — source conversation linked to a task
- `refresh_tokens` — for token rotation, scoped per user (not yet per device)

**Relationships**:
- User ↔ Workspaces (many-to-many via `workspace_members`)
- Task → Workspace (one workspace, non-nullable)
- Task → SlackThread (optional, immutable once set)
- RefreshToken → User (many tokens per user, one per issued session)

**Constraints**:
- Task cannot exist without a workspace
- Slack thread link is optional but immutable once set (data integrity
  with external system)
- Soft deletes only — tasks get `archived_at`, never hard deleted

---

## 8. API & Interfaces

**External APIs consumed**:
- Slack Events API (webhook inbound)
- Slack Web API (message fetching, OAuth token exchange)

**Internal API contracts**:
- `POST /auth/token` — login, returns access + refresh tokens
- `POST /auth/refresh` — rotates refresh token, invalidates old
- `POST /auth/slack` — Slack OAuth callback handler
- `GET /workspaces/{id}/tasks` — paginated task list with filters
- `POST /workspaces/{id}/tasks` — create task
- `PATCH /tasks/{id}` — update status, assignee, title

**Integration points**:
- Slack OAuth app credentials: `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_SIGNING_SECRET`

---

## 9. Security Model

**Sensitive data**:
- User email addresses (low sensitivity)
- Slack OAuth tokens (high — stored encrypted at rest)
- Refresh tokens (high — stored hashed, never plaintext)
- Task content (medium — private team data)

**Threat surface**:
- Slack webhook endpoint (public, unauthenticated inbound)
- OAuth callback endpoint (must validate state param to prevent CSRF)
- Refresh token endpoint (must validate token and rotate atomically)

**Auth & authorization design**:
- Access token: JWT, 15-minute TTL, signed with HS256
- Refresh token: opaque random token, hashed in DB, 30-day TTL
- Workspace authorization: checked on every API route via middleware
- Slack webhook: verified via HMAC-SHA256 signature on every request

**Security constraints that must never be violated**:
- Slack credentials never logged, even in debug mode
- Refresh tokens stored hashed only (bcrypt). Never log or return plaintext
- Slack webhook signature must be verified before any processing
- OAuth state param must be validated to prevent CSRF

---

## 10. Design Decisions

## Stateless JWT Authentication

**Date**: 2026-05-12
**Status**: Decided

**Context**:
Needed auth strategy before building protected routes. App is web-only
today but mobile is on the roadmap.

**Decision**:
JWT with 15-minute access tokens and rotating refresh tokens (30-day TTL).

**Reasoning**:
Stateless auth means any server instance can verify a token without a
shared session store. Horizontal scaling is clean. Mobile clients work
without sticky sessions. Short access token TTL limits the blast radius
of a stolen token.

**Alternatives Considered**:
- Server sessions — rejected: requires shared Redis or sticky sessions;
  complicates mobile
- Opaque tokens with a token store — rejected: effectively sessions by
  another name with the same scaling problems

**Explicit Non-Decision**:
We are NOT storing sessions. Even for simplicity in MVP. The cost of
refactoring auth later when mobile is added is too high.

**Trade-offs**:
Access tokens cannot be truly revoked before expiry. We accept this in
exchange for simplicity and scalability. Mitigated by 15-minute TTL.

**Consequences**:
- Logout must call the API (can't just clear a cookie)
- Client must silently refresh access tokens
- `refresh_tokens` table required in DB

---

## Soft Deletes Only

**Date**: 2026-05-18
**Status**: Decided

**Context**:
Early feedback showed users felt accidental deletion in similar tools was
catastrophic. Slack threads persist even when tasks are "deleted" —
hard deleting a task while the thread still exists creates inconsistency.

**Decision**:
All deletions are soft. Tasks get `archived_at`. No hard deletes in MVP.

**Reasoning**:
Maintains integrity with linked Slack threads. Enables undo without extra
infrastructure. Preserves audit trail automatically.

**Explicit Non-Decision**:
No recycle bin UI in MVP. Archived tasks are simply hidden. A restore
flow can come later if users ask for it.

**Trade-offs**:
Database grows unboundedly. Needs a purge policy in Phase 2.

---

## Polling Over WebSockets for Real-Time Updates

**Date**: 2026-05-30
**Status**: Decided — supersedes original WebSocket design

**Context**:
Originally intended to use WebSockets for live board updates. After the
spike (see § Exploration Journal), Railway's infrastructure made this
unworkable.

**Decision**:
5-second polling via standard REST API calls.

**Reasoning**:
Polling is simple, debuggable, and costs nothing. For a kanban board
where updates are infrequent, 5s lag is imperceptible. WebSockets add
significant operational complexity for a marginal UX gain at this scale.

**Alternatives Considered**:
- WebSockets on Railway — rejected: 30s connection timeout, non-negotiable
- Server-Sent Events (SSE) — considered but Railway has same timeout issue;
  deferred to Phase 2 on different infra if needed
- Long polling — rejected: more complex than regular polling, less upside
  than true WebSockets

**Explicit Non-Decision**:
Not switching infrastructure providers just to support WebSockets. The
core product value doesn't require real-time — it requires reliability.

**Trade-offs**:
5s latency on board updates. Acceptable. More API calls than WebSocket
would generate — negligible at current scale.

---
**⚠ Original design superseded**: 2026-05-28

**Why reversed**:
FastAPI WebSocket support was implemented and working locally, but
Railway's free tier enforces a 30-second connection timeout on all TCP
connections. WebSocket clients were being dropped silently. The error was
not immediately obvious because it looked like a client-side disconnect.

**What we learned from the WebSocket direction**:
Infrastructure constraints must be validated before building features
that depend on persistent connections. We spent 2 days on an
implementation that the hosting layer made impossible.

**What's preserved**:
The FastAPI WebSocket router code is kept in `api/ws/` (not deleted) in
case we revisit this on different infrastructure.

**See**: "Polling Over WebSockets for Real-Time Updates" (above)

---

## 11. Engineering Standards

**Coding conventions**:
- Python: Black formatter, isort, strict mypy typing
- TypeScript: ESLint + Prettier, strict mode, no implicit `any`
- No `any` type without an explanatory comment

**Patterns in use**:
- Repository pattern for all DB access — no raw queries in route handlers
- All API errors return `{ error: string, code: string }` consistently
- Feature flags via environment variables only — no library for now

**Testing approach**:
- All API routes: at least one happy-path Pytest test
- Auth flows: full negative-case coverage (invalid tokens, expired tokens,
  missing fields)
- E2E (Playwright): three core journeys only — signup, create task from
  Slack, move task on board
- WebSocket code in `api/ws/` is not tested — it's preserved but inactive

**Things that must never be violated**:
- Frontend never queries DB directly
- Slack credentials never logged
- Refresh tokens stored hashed, never plaintext
- Slack webhook signature verified before processing

---

## 12. Assumption Log

| Assumption | Type | Status | Evidence |
|---|---|---|---|
| Users won't tolerate > 2s load time | User | Unvalidated | Sourced from general SaaS benchmarks, not user research |
| 5s polling lag is imperceptible on a kanban board | Technical | Unvalidated | Intuitive — needs user testing |
| Small teams prefer lightweight tooling over features | User | Partially validated | Consistent in early conversations; not formally tested |
| Railway pricing will remain stable | Business | Unvalidated | Risk — see Risk Register |
| Slack thread links remain accessible indefinitely | Technical | INVALIDATED 2026-05-25 | Slack free tier expires thread history at 90 days. See Open Questions. |

---

## 13. Risk Register

| Risk | Likelihood | Impact | Mitigation | Status |
|---|---|---|---|---|
| Railway changes pricing or limits | Medium | High | Abstract infra layer; avoid Railway-specific APIs | Monitoring |
| Slack API deprecates Events API v2 | Low | Critical | No mitigation yet — would require full rebuild | Watching |
| Slack thread history expires (free tier = 90 days) | High | Medium | Store thread content snapshot at link time | Planned — not built |
| Refresh token not rotated — stolen token reuse | Medium | High | Token rotation is on the roadmap — not in MVP | In progress |
| Webhook signature not verified — spoofed events | Medium | High | Must fix before launch | Planned |

---

## 14. Conversations & Thinking

## Slack Thread Expiry Problem — 2026-05-25

**User's intent / thinking**:
User noticed during testing that a thread they linked a task to was no
longer accessible. Realized Slack free tier expires message history.
Raised concern that the core value proposition — tasks carrying their
source thread — breaks for free Slack users.

**Model's analysis**:
This is a real product risk, not just a bug. If the linked thread expires,
the task loses its context — exactly the problem TaskFlow exists to solve.
The mitigation is to snapshot the thread content (or at minimum the
permalink and first few messages) at the moment of linking, storing it
in our database. This decouples the task's context from Slack's retention
policy.

**What was explored**:
- Ignoring the problem for MVP (rejected — undermines core value)
- Warning users when threads are at risk of expiry (band-aid)
- Storing full thread snapshot at link time (chosen direction)

**Outcome**:
Invalidated the assumption that thread links remain accessible. Added to
Risk Register. Thread content snapshotting added to roadmap. Open question
raised about how much to store.

---

## Initial Architecture Discussion — 2026-05-10

**User's intent / thinking**:
Wanted a stack maintainable by a solo developer but professional enough to
hand off to a small team without a rewrite. Python backend was non-negotiable.
React experience made Next.js the natural frontend choice. Did not want to
manage infrastructure heavily.

**Model's analysis**:
FastAPI + Next.js is well-understood and a solid choice. Pushed back on
SQLite for the database — Slack webhook processing means concurrent writes
from background tasks, which SQLite handles poorly under contention.
Railway recommended over Heroku (pricing) and AWS (ops overhead for MVP).

**What was explored**:
- SQLite: rejected (concurrent webhook writes)
- Supabase as full backend: rejected (too opinionated, wanted to own API)
- Django: rejected (more than needed, slower to iterate)

**Outcome**:
FastAPI + PostgreSQL + Next.js on Vercel/Railway. User confirmed comfort
and prior knowledge with this stack.

---

## 15. Exploration Journal

## WebSocket Real-Time Updates — 2026-05-26 to 2026-05-28

**Why we tried it**:
Real-time board updates would feel more collaborative — teammates moving
tasks would appear instantly for other users on the same board. Seemed
like a meaningful UX improvement and FastAPI has native WebSocket support.

**What we did**:
Implemented a WebSocket endpoint in FastAPI at `/ws/board/{workspace_id}`.
Connected Next.js client via native WebSocket API. Broadcast task update
events on state change. Worked correctly in local development.

**What happened**:
Deployed to Railway staging. Connections dropped silently after ~30 seconds.
Initially misdiagnosed as a client-side disconnect bug. Spent ~4 hours
ruling out the Next.js implementation. Eventually traced to Railway's
infrastructure — all TCP connections (including WebSockets) are subject
to a 30-second idle timeout on the free tier.

**Why it didn't work**:
Railway's load balancer enforces a 30-second connection timeout. WebSocket
keepalives were not sufficient to prevent this. Not a code bug — a hosting
constraint that cannot be worked around on the current plan.

**What we learned**:
Always validate infrastructure constraints (connection limits, timeout
policies, egress costs) before building features that depend on persistent
connections. The local dev environment masked the constraint entirely.

**Impact on direction**:
Reverted to 5-second polling. Code kept in `api/ws/` but inactive.
Design Decision updated to reflect the reversal.

**Revisit?**:
Yes — if we move to a hosting provider that supports long-lived connections
(e.g., Fly.io, self-managed), or upgrade Railway plan. SSE (Server-Sent
Events) is also worth evaluating as a simpler alternative to WebSockets.

---

## 16. Known Issues & Technical Debt

| Issue | Severity | Status | Workaround | Resolution |
|---|---|---|---|---|
| Slack webhook signature not verified | High | Planned (pre-launch blocker) | None — don't expose to public yet | — |
| Refresh token rotation not implemented | High | In progress | Tokens don't rotate — acceptable for closed beta only | — |
| Task list API does full table scan | Medium | Backlog | Acceptable at current data volume | Index on `workspace_id + status` needed |
| Slack thread content not snapshotted | Medium | Planned | Linked threads may expire on free Slack | Store snapshot at link time |
| No error boundary in Next.js app | Low | Backlog | Unhandled errors crash full page | Add React error boundary |
| WebSocket code untested | Low | Accepted | Code is inactive — no active tests | Will test if/when reactivated |

---

## 17. Lessons Learned

**Mistakes made**:
- Spent ~4 hours debugging the WebSocket drop before checking Railway's
  connection timeout policy. Always verify infrastructure constraints
  before building features that need persistent connections.
- Spent time debugging Slack OAuth before checking the Slack App Dashboard.
  External service configuration errors almost always live in the dashboard,
  not in the code.
- Did not validate the Slack thread expiry assumption early enough. Core
  product assumption was wrong and we discovered it late.

**Surprising discoveries**:
- Slack's Events API sends duplicate events if your endpoint doesn't return
  200 fast enough. Must acknowledge immediately and process asynchronously.
- Next.js App Router + FastAPI CORS interact unexpectedly with credentials.
  `credentials: 'include'` must be set on EVERY fetch AND in the CORS config.
- Railway's timeout applies to SSE as well — not just WebSockets.

**What we'd do differently**:
- Validate all infrastructure constraints in a spike before building
  features that depend on them.
- Set up E2E tests earlier. First Playwright pass found 3 auth bugs
  that unit tests completely missed.
- Explicitly document non-goals from day one. "We're not building time
  tracking" prevents an hour of discussion every sprint.

---

## 18. Open Questions

- Refresh tokens: should they be device-scoped? Currently one per user —
  logging out on one device logs out everywhere. Acceptable for now?
- Slack thread snapshot: how much content do we store? Just the first
  message? The whole thread? What's the storage implication at scale?
- Task activity feed: do users need to see who moved a task and when?
  Not in MVP but stakeholders keep asking.

---

## 19. Roadmap

**Next up** (committed):
- Thread content snapshotting at link time
- Slack webhook signature verification (pre-launch blocker)
- Refresh token rotation

**Backlog** (likely):
- Task comments
- Email notifications on assignment
- Pagination on task list API
- Per-device refresh token scoping

**Abandoned** (tried or planned, then cut — with reason):
- WebSocket real-time updates: infrastructure constraint on Railway.
  Reverted to polling. Revisit if hosting changes.

**Ideas** (maybe someday):
- GitHub PR linking
- Mobile native app
- AI summarization of source Slack thread
- SSE as WebSocket alternative

---

## 20. References

- [Slack Events API docs](https://api.slack.com/events)
- [FastAPI security / JWT pattern](https://fastapi.tiangolo.com/tutorial/security/)
- [Refresh token rotation — Auth0](https://auth0.com/blog/refresh-tokens-what-are-they-and-when-to-use-them/)
- [Railway TCP timeout documentation](https://docs.railway.app/reference/tcp-proxy)
- Inspiration: Linear's thread-linked issues, Notion's simplicity
