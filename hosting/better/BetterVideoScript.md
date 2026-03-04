# Decisio — Video Walkthrough Script (10–15 min)

Use this as your talking-point guide. Read through it, internalise the reasoning, then speak naturally on camera.

---

## 0. THE PROBLEM & THE SOLUTION (≈ 2 min)

### The Problem

> "Every engineering team makes dozens of technical decisions — which database to use, how to structure the API, when to refactor. But these decisions live in Slack threads, meeting notes, or just someone's head. Six months later, nobody remembers *why* we chose Postgres over MongoDB, or *who* approved the decision, or *what alternatives we considered*. New team members join and re-fight the same battles because there's no record."

**Pain points to mention:**
- Decisions get lost in chat history and scattered docs
- No clear lifecycle — is it just an idea, or did the team actually commit?
- No accountability — who decided, when, and why?
- No traceability — if a decision turns out wrong, there's no link to what replaced it
- Re-visiting old debates wastes time because there's no record of what was already evaluated

### The Solution: Decisio

> "Decisio is a decision tracking platform that gives every technical decision a structured lifecycle, a permanent record, and a complete audit trail."

**Key points:**
- Every decision has a **status lifecycle**: Draft → Proposed → Decided → Superseded
- Each decision records the **problem context**, the **options considered** (with pros/cons), and the **final outcome**
- Once decided, the record is **immutable** — you can't silently rewrite history
- If a decision needs to change, you **supersede** it — a new decision replaces the old one with a clear link back
- Every action is **audited** — who did what, when, and why
- Decisions are **tagged** and organised by **project** for easy search

### Real-Life Use Case: Bob & The API Gateway Decision

> "Let me walk you through how this would actually be used."

1. **Bob is a backend engineer** on the Platform Team. The team needs to decide whether to use an API gateway or keep direct service-to-service calls.

2. **Bob creates a new decision** in Decisio under the "Platform Infrastructure" project. He writes the context: *"Our microservices currently call each other directly. As we scale to 15+ services, we're seeing retry storms and inconsistent auth checks."*

3. **Bob adds three options:**
   - **Option A: Kong API Gateway** — Pros: mature, plugin ecosystem. Cons: operational overhead, new infra.
   - **Option B: AWS API Gateway** — Pros: managed service, no ops. Cons: vendor lock-in, cold starts.
   - **Option C: Keep direct calls** — Pros: no change needed. Cons: problems will get worse.

4. **Bob tags the decision** with `infrastructure`, `api-design`, and `scalability`, then **transitions it to Proposed** — the team can now review it.

5. **In the next team meeting**, they discuss. Alice and Carol review the options in Decisio. They agree on Kong. The lead **transitions the decision to Decided** and marks Option A as chosen, with a final summary: *"Going with Kong for centralized auth, rate limiting, and retry policies."*

6. **The decision is now frozen.** Nobody can silently edit it. The audit trail shows who proposed it, when it was decided, and who approved it.

7. **Three months later**, the team realises Kong's operational overhead is too high. Instead of editing the old decision, David **creates a new decision that supersedes it**: *"Migrating from Kong to AWS API Gateway — the ops burden of self-hosted Kong isn't justified at our scale."* The old decision is automatically marked Superseded with a pointer to the new one.

8. **A year later, a new hire** joins and wonders "why are we on AWS API Gateway?" They search Decisio, find the current decision, click through to the superseded one, and read the full history — problem, options, reasoning, and what changed. **No Slack archaeology needed.**

> "That's the core workflow. Let me show you how I built it."

---

## 1. ARCHITECTURE (≈ 2.5 min)

### High-Level
- **Three-tier architecture**: React frontend → Flask REST API → PostgreSQL database
- Frontend: Next.js 16 + React 19 + TypeScript + Tailwind CSS v4
- Backend: Python 3.12, Flask 3.1, SQLAlchemy ORM, Marshmallow validation
- Database: PostgreSQL 18 with full relational integrity

### Why this split?
- **Separation of concerns** — each layer does one job. Frontend handles UI state, backend owns business logic, database enforces data integrity.
- API is stateless (JWT tokens), so it's easy to scale horizontally later.
- Could swap the frontend to a mobile app without touching the backend.

### Application Factory Pattern
- Flask uses the `create_app()` factory pattern — the app isn't a global variable.
- This allows us to spin up different instances for dev, test, and production with different configs.
- Tests create a fresh app with a test database on every run — full isolation.

### Key Architectural Layers
```
Browser (React SPA)
   ↓  HTTP/JSON + JWT Bearer token
Flask API  (routes → services → models)
   ↓  SQLAlchemy ORM
PostgreSQL  (constraints, indexes, CHECK rules)
```

---

## 2. PROJECT STRUCTURE (≈ 2 min)

### Backend (`backend/`)
```
backend/
├── app.py            # Application factory (create_app)
├── config.py         # Dev / Test / Prod config classes
├── extensions.py     # SQLAlchemy, Migrate, Bcrypt singletons
├── models/           # 6 SQLAlchemy models (User, Project, Decision, Option, Tag, AuditLog)
├── schemas/          # 5 Marshmallow schemas for input validation
├── services/         # Business logic layer (no Flask request context leakage)
├── routes/           # 6 route blueprints (thin controllers — validate → call service → respond)
├── middleware/       # @require_login, @require_admin decorators
├── tests/            # 32 pytest tests (auth, projects, decisions, audit, admin)
├── migrations/       # Alembic auto-generated migration
├── seed_demo.py      # Demo data seeder (4 projects, 9 decisions, 9 tags, 21 audit entries)
└── requirements.txt  # Pinned dependencies
```

**Why this matters**: Routes are thin, services are thick. A route does: parse input → validate with schema → call service → return JSON. All business rules live in the service layer, making them testable without HTTP.

### Frontend (`frontend/`)
```
frontend/
├── app/              # Next.js app router (layout, page, globals.css)
├── components/
│   ├── pages/        # 8 page-level smart components
│   ├── ui/           # 50+ reusable ShadCN/Radix UI primitives
│   ├── app-shell.tsx # Main layout + routing controller
│   └── app-sidebar.tsx, top-bar.tsx, etc.
├── lib/
│   ├── types.ts      # TypeScript interfaces matching API response shapes
│   ├── api.ts        # API client singleton (all HTTP calls in one place)
│   ├── auth-context.tsx  # JWT auth provider (login/logout, token persistence)
│   ├── data-context.tsx  # App-wide data layer (loads projects, decisions, tags on auth)
│   └── utils.ts
└── hooks/            # Custom React hooks
```

**Key design choice**: Single `api.ts` file is the only place that talks to the backend. Every page component calls `useData()` from context or calls `api.xxx()` methods. If the API shape changes, you fix ONE file.

---

## 3. TECHNICAL DECISIONS (≈ 3 min)

### Decision 1: Stateless JWT Auth (not session-based)
- **Why**: No server-side session store needed. Token is self-contained — contains user_id, expiry.
- **How**: Login → backend generates JWT (HS256, 24hr expiry) → frontend stores in localStorage → sends as `Authorization: Bearer <token>` on every request.
- **Middleware**: `@require_login` decorator decodes JWT, loads user, sets `g.current_user`. `@require_admin` chains on top — checks `is_admin` flag.

### Decision 2: Three-Layer Validation (Interface Safety)
This is the big one. Data is validated at three levels:

| Layer | Technology | What it catches |
|-------|-----------|----------------|
| **Frontend** | TypeScript interfaces | Compile-time type mismatches. You can't pass a number where a string is expected. |
| **Backend API** | Marshmallow schemas | Runtime validation — required fields, string length limits, email format, OneOf enumerations. Returns 400 with specific error messages. |
| **Database** | PostgreSQL constraints | Last line of defense — NOT NULL, UNIQUE, CHECK constraints, FOREIGN KEYS. Even if API validation is bypassed, the DB rejects bad data. |

**Example flow**: Creating a decision → TypeScript ensures the form sends the right shape → Marshmallow's `CreateDecisionSchema` validates title (1-255 chars), context (required), project_id (integer), status (OneOf) → PostgreSQL enforces NOT NULL, FK to projects table, CHECK constraint that `superseded_by != id`.

### Decision 3: Status State Machine (Correctness)
Decisions follow a strict lifecycle: **Draft → Proposed → Decided → Superseded**
- Transitions are controlled by a `STATUS_TRANSITIONS` dictionary in the model.
- The service layer (`transition_status`) checks current status against allowed next states.
- You **cannot skip steps** (e.g., Draft → Decided is blocked).
- You **cannot go backwards** (e.g., Decided → Proposed is blocked).
- You **cannot manually set status to Draft** on transition (blocked in Marshmallow schema).
- Superseded is only set via the `supersede_decision` flow — it atomically creates a new decision + marks the old one.

### Decision 4: Immutability of Decided Records
- Once a decision reaches "Decided" status, its content is frozen.
- The only action allowed is to supersede it, which creates a new Draft linked back to the original.
- This preserves the historical record — you can always trace why a decision was changed.

### Decision 5: Audit Trail
- Every mutation (create, edit, status change, supersede, soft-delete) writes an `AuditLog` entry.
- Captures: who did it (`actor_id`), what they did (`action`), old/new status, optional note.
- This is append-only — audit entries are never edited or deleted.

### Decision 6: Soft Deletes
- Decisions are never actually removed from the database. `is_deleted = True` hides them from the default query.
- This preserves referential integrity and audit history.

---

## 4. AI USAGE (≈ 2 min)

### How AI Was Used
- **GitHub Copilot (Claude)** was used as a pair-programming assistant throughout the entire build.
- AI codegen in a VS Code chat environment — described what I wanted, reviewed the output, iterated.

### What AI helped with:
1. **Scaffolding** — Initial project structure, boilerplate (app factory, config, extensions).
2. **Models & Schemas** — Defining SQLAlchemy models with proper constraints, Marshmallow validators. I described the data model; AI generated the code; I reviewed and refined.
3. **Service Layer Logic** — Status transition logic, supersede workflow, tag/option syncing. I described the business rules; AI implemented them.
4. **Tests** — 32 integration tests covering auth flows, CRUD operations, permission checks, edge cases. AI generated test scaffolds; I specified what scenarios to cover.
5. **Frontend Wiring** — Connecting React components to the API, building the data context, auth flows.
6. **Diagrams** — ER diagrams and architecture visuals for this presentation.

### What I did (human decisions):
- **Architecture choices** — chose Flask + PostgreSQL + Next.js stack, decided on the three-layer validation approach, designed the state machine.
- **Data model design** — defined which tables, columns, constraints, and relationships were needed.
- **Business rules** — specified the decision lifecycle, immutability rules, audit requirements.
- **Code review** — read through all generated code, caught edge cases, requested fixes.
- **Testing strategy** — decided what to test, what edge cases matter.

### AI Guidance Files
We maintained three AI guidance documents (`ai-guidance/` folder):
- `CLAUDE.md` — Project context, conventions, and rules for the AI assistant.
- `architecture-decisions.md` — Logged all architecture choices with rationale.
- `coding-standards.md` — Code style rules, naming conventions, patterns to follow.

These ensured **consistency** — the AI followed the same patterns throughout because it had documented rules to reference.

---

## 5. RISKS & MITIGATIONS (≈ 2 min)

### Risk 1: JWT in localStorage — XSS vulnerability
- **Risk**: If an attacker can run JavaScript on the page, they can steal the JWT from localStorage.
- **Mitigation (if extended)**: Move to HttpOnly cookies with CSRF protection. For this MVP scope, it's an acceptable trade-off.
- **Current safeguard**: React's JSX auto-escapes output, reducing XSS surface area significantly.

### Risk 2: No Rate Limiting
- **Risk**: API endpoints could be hammered (brute-force login, spam creation).
- **Mitigation (if extended)**: Add Flask-Limiter with per-IP and per-user rate limits.
- **Why not now**: MVP scope — adding it is a 30-minute task, doesn't affect core architecture.

### Risk 3: No Pagination
- **Risk**: Large datasets (hundreds of decisions) could cause slow responses.
- **Mitigation (if extended)**: Add `page` and `per_page` query params, return paginated JSON with metadata.
- **Current state**: Fine for MVP scale. The queries use indexes on project_id, status, and created_by.

### Risk 4: Single-Server Deployment
- **Risk**: No horizontal scaling.
- **Mitigation**: JWT is stateless, so you can horizontally scale the API behind a load balancer. DB is the bottleneck — could add read replicas.

### Risk 5: No Email Verification / Password Reset
- **Risk**: No way to recover accounts.
- **Mitigation (if extended)**: Add email service integration, password reset tokens.

---

## 6. EXTENSION APPROACH (≈ 2 min)

### How would you extend this?

**Feature: Real-time Notifications**
- Add WebSocket support (Flask-SocketIO) to push audit events to connected users.
- Frontend subscribes to decision changes — live updates when someone changes a status.

**Feature: File Attachments**
- Add an `attachments` table (decision_id FK, file_url, mime_type, uploaded_by).
- Use presigned S3 URLs for upload/download.
- No change to existing models — purely additive.

**Feature: Decision Templates**
- Add a `templates` table with preset options and tag sets.
- "Create from template" in the UI pre-fills the form.

**Feature: Role-Based Access Control (RBAC)**
- Currently binary (Admin vs Member). Could extend to per-project roles (Viewer, Editor, Approver).
- Add a `project_memberships` join table with a `role` column.

**Feature: Advanced Search**
- Add PostgreSQL full-text search on decision titles, context, and option text.
- Use `tsvector` columns and GIN indexes — no external search engine needed.

**Feature: Export & Reporting**
- PDF/CSV export of decisions per project.
- Dashboard analytics with Recharts (already installed).

### The key point:
The current architecture is designed for extensibility. The service layer pattern means new features follow the same flow: add model → add schema → add service → add route → add frontend page. No refactoring needed.

---

## QUICK DEMO FLOW (what to show on screen)

1. **Login** as admin → show the dashboard with stats
2. **Create a project** → show validation (try empty name)
3. **Create a decision** with options and tags → show the form
4. **Transition status** Draft → Proposed → Decided → show the state machine working
5. **Supersede** a decided decision → show the link back to original
6. **Audit trail** → show the full history of changes
7. **Admin panel** → show user management (only visible to admin)
8. **Search** → find decisions by keyword/tag
9. **Show tests passing** → run `pytest` in terminal, all 32 green

---

## KEY NUMBERS TO MENTION

- **7 database tables** with full relational integrity
- **2 CHECK constraints** + **2 partial unique indexes** at the DB level
- **5 Marshmallow schemas** validating every API input
- **6 API route blueprints** with 20+ endpoints
- **32 integration tests** — all passing
- **3-layer validation**: TypeScript → Marshmallow → PostgreSQL
- **4-state lifecycle**: Draft → Proposed → Decided → Superseded
- **Full audit trail** on every mutation
