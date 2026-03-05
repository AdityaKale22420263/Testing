# Decisio — Architecture Diagram (Canva Reference)

Recreate this in Canva. Colours in [brackets]. Rounded rects for tiers, pills for nodes.

```
                              ┌──────────┐
                              │ Browser  │
                              └────┬─────┘
                                   │ HTTPS
                                   ▼
  ╔══════════════════════════════════════════════════════════════════╗
  ║  FRONTEND                                          [#DBEAFE]   ║
  ║                                                                ║
  ║   Next.js 16  ·  React 19  ·  TypeScript  ·  Tailwind v4      ║
  ║                                                                ║
  ║   ┌─────────┐ ┌────────┐ ┌──────────┐ ┌───────┐ ┌───────┐    ║
  ║   │Dashboard│ │Projects│ │Decisions │ │Search │ │ Admin │    ║
  ║   └─────────┘ └────────┘ └──────────┘ └───────┘ └───────┘    ║
  ║                                                                ║
  ║   Auth Context · Data Context · ShadCN UI · Recharts           ║
  ╚══════════════════════════════════════════════════════════════════╝
                                   │ REST / JSON + JWT
                                   ▼
  ╔══════════════════════════════════════════════════════════════════╗
  ║  API LAYER                                         [#DCFCE7]   ║
  ║                                                                ║
  ║   Flask 3.1  ·  Python 3.12                                    ║
  ║                                                                ║
  ║   ┌────────────────────────────────────────────────────────┐   ║
  ║   │ Blueprints: /auth /users /projects /decisions /tags    │   ║
  ║   └──────────────────────────┬─────────────────────────────┘   ║
  ║                              ▼                                  ║
  ║   ┌────────────────────────────────────────────────────────┐   ║
  ║   │ Marshmallow Validation · CORS · Error Handlers         │   ║
  ║   └────────────────────────────────────────────────────────┘   ║
  ╚══════════════════════════════════════════════════════════════════╝
                                   │
                        ┌──────────┴──────────┐
                        ▼                     ▼
  ╔═════════════════════════╗  ╔════════════════════════════════════╗
  ║ SECURITY    [#FED7AA]   ║  ║ BUSINESS LOGIC       [#E9D5FF]    ║
  ║                         ║  ║                                    ║
  ║  JWT Auth (HS256)       ║  ║  Decision Engine                   ║
  ║  Bcrypt Hashing         ║  ║  ┌──────────────────────────────┐  ║
  ║  Admin / Member Roles   ║  ║  │ Draft → Proposed → Decided   │  ║
  ║                         ║  ║  │              → Superseded     │  ║
  ╚═════════════════════════╝  ║  └──────────────────────────────┘  ║
                               ║  Immutability · Soft Deletes       ║
                               ║  Audit Trail · Supersede Workflow  ║
                               ╚════════════════════════════════════╝
                                          │ ORM Queries
                                          ▼
  ╔══════════════════════════════════════════════════════════════════╗
  ║  DATABASE                                          [#FEE2E2]   ║
  ║                                                                ║
  ║   PostgreSQL 18  ·  SQLAlchemy  ·  Flask-Migrate               ║
  ║                                                                ║
  ║   ┌─────┐ ┌────────┐ ┌─────────┐ ┌───────┐ ┌────┐            ║
  ║   │users│ │projects│ │decisions│ │options│ │tags│            ║
  ║   └─────┘ └────────┘ └────┬────┘ └───────┘ └────┘            ║
  ║                            ├─── decision_tags (M:N)            ║
  ║                            └─── audit_log (immutable)          ║
  ║                                                                ║
  ║   CHECK constraints · Partial unique indexes · FK cascades     ║
  ╚══════════════════════════════════════════════════════════════════╝

  ┌────────────────────────────────────────────────────────────────┐
  │  TESTING  [#F3F4F6]     pytest · 32 integration tests         │
  └────────────────────────────────────────────────────────────────┘


  ┌──────────────── THREE-LAYER VALIDATION ────────────────┐
  │                                                        │
  │  TypeScript    ──→   Marshmallow    ──→   PostgreSQL   │
  │  (compile)           (runtime)            (DB safety)  │
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

## Canva Colour Palette

| Tier | Hex | Role |
|---|---|---|
| Frontend | `#DBEAFE` | light blue |
| API | `#DCFCE7` | light green |
| Security | `#FED7AA` | light orange |
| Business Logic | `#E9D5FF` | light purple |
| Database | `#FEE2E2` | salmon |
| Testing | `#F3F4F6` | light grey |
| Borders / arrows | `#374151` | dark grey |
| Headings | `#111827` | near black |

## Canva Tips

- Dashed borders for tier containers, solid for inner nodes
- Pill shapes for the five pages row
- Small text near arrows for labels ("REST / JSON", "ORM Queries")
- State machine looks good as a horizontal flow inside one box
- Font: ~18px headings, ~12px labels

---

## Block-by-Block Explanation (for video narration)

### 1. Browser
The user opens Decisio in any modern browser. This is the entry point — everything the user sees and interacts with originates here.

### 2. FRONTEND (light blue)
This is the **client-side application** running in the browser.

- **Next.js 16 · React 19 · TypeScript · Tailwind v4** — the core stack. Next.js handles routing and rendering, React manages the UI components, TypeScript catches type errors at compile time, and Tailwind provides utility-first CSS styling.
- **Dashboard, Projects, Decisions, Search, Admin** — these are the five main pages/views. Each is a self-contained React component.
- **Auth Context** — a React context that manages the logged-in user's state and JWT token across the entire app. Every page knows who you are through this.
- **Data Context** — a React context that holds all decisions, projects, and tags in memory and provides functions like `createDecision()`, `transitionStatus()`, `deleteDecision()`. Components call these instead of making raw API calls.
- **ShadCN UI** — the component library (buttons, modals, dialogs, dropdowns) built on Radix UI primitives. This gives us accessible, theme-consistent components.
- **Recharts** — the charting library powering the dashboard's bar and pie charts.

### 3. Arrow: HTTPS
The browser loads the frontend over HTTPS. All static assets (JS bundles, CSS) are served by Next.js.

### 4. Arrow: REST / JSON + JWT
This is the communication channel between frontend and backend. Every API call:
- Uses standard HTTP methods (GET, POST, PATCH, DELETE)
- Sends and receives JSON payloads
- Includes a `Authorization: Bearer <JWT>` header for authentication

This is a **stateless** protocol — the server doesn't remember sessions. Every request must carry its own credentials.

### 5. API LAYER (light green)
This is the **backend web server** that receives and responds to HTTP requests.

- **Flask 3.1 · Python 3.12** — Flask is a lightweight WSGI framework. It's intentionally minimal, which gives us full control over the architecture.
- **Blueprints: /auth /users /projects /decisions /tags** — Flask Blueprints organize routes by resource. Each blueprint is a separate Python module handling all endpoints for that resource (e.g., `/api/decisions` has its own blueprint with GET, POST, PATCH, DELETE routes).
- **Marshmallow Validation** — every incoming JSON body is validated against a Marshmallow schema before it reaches business logic. Invalid data is rejected with a 400 error and detailed field-level messages.
- **CORS** — Cross-Origin Resource Sharing headers allow the frontend (running on `localhost:3000`) to call the backend (running on `localhost:5000`).
- **Error Handlers** — centralized error handling ensures every failure returns a consistent JSON error response, never a raw stack trace.

### 6. Arrow: from API Layer splitting into Security and Business Logic
After a request passes the API layer (routing + validation), it flows into two concerns:
- **Security** — is this user allowed to do this?
- **Business Logic** — what actually happens?

These run sequentially (auth first, then logic), but they're shown side-by-side to emphasize they're separate concerns.

### 7. SECURITY (light orange)
This block handles **authentication** (who are you?) and **authorization** (what can you do?).

- **JWT Auth (HS256)** — on login, the server issues a JSON Web Token signed with HMAC-SHA256. The token contains the user ID and expiry (24 hours). Every subsequent request includes this token, and the server verifies it without a database lookup.
- **Bcrypt Hashing** — passwords are never stored in plain text. They're hashed with bcrypt (a deliberately slow hashing algorithm that makes brute-force attacks impractical).
- **Admin / Member Roles** — two roles. Members can create decisions, submit for review, and browse. Admins can additionally approve decisions (Proposed → Decided), manage users, soft-delete decisions, and view the system audit log. Enforced via `@require_login` and `@require_admin` decorators on each route.

### 8. BUSINESS LOGIC (light purple)
This is where the **domain rules** live — the things that make Decisio more than a basic CRUD app.

- **Decision Engine** — manages the four-state lifecycle:
  - **Draft** — initial state, editable by creator
  - **Proposed** — submitted for review, still editable
  - **Decided** — approved by admin, now **immutable** (frozen forever)
  - **Superseded** — replaced by a newer decision, which auto-creates a new Draft
- **Immutability** — once a decision reaches "Decided", no fields can be changed. This is a core integrity guarantee — the historical record is preserved.
- **Soft Deletes** — decisions are never physically removed from the database. Admin "delete" sets an `is_deleted` flag. The decision disappears from all views but remains in the database for auditing.
- **Audit Trail** — every mutation (create, edit, status change, supersede, delete) writes an immutable `AuditLog` row recording who did what, when, and the old/new status. This provides a complete forensic history.
- **Supersede Workflow** — when requirements change after a decision is finalized, instead of editing it (which is blocked), users supersede it. This creates a new Draft pre-linked to the old decision, and marks the old one as Superseded. The chain is traceable.

### 9. Arrow: ORM Queries
Business logic communicates with the database through **SQLAlchemy ORM**. Instead of writing raw SQL, service functions use Python objects (e.g., `decision.status = "Decided"`), and SQLAlchemy translates these to optimized SQL queries. This provides safety (parameterized queries, no SQL injection) and portability.

### 10. DATABASE (salmon)
This is the **persistence layer** — where all data lives permanently.

- **PostgreSQL 18** — a production-grade relational database chosen for its strong constraint enforcement, JSON support, and partial unique indexes.
- **SQLAlchemy** — the Python ORM that maps Python classes to database tables.
- **Flask-Migrate** — manages schema evolution through Alembic migration files. When the schema changes, migrations are versioned and can be applied/rolled back.
- **Seven tables:**
  - `users` — accounts with bcrypt-hashed passwords and admin flags
  - `projects` — containers that group related decisions
  - `decisions` — the core entity with status, context, final summary, and soft-delete flag
  - `options` — the alternatives considered for each decision (with pros/cons)
  - `tags` — labels for categorizing decisions
  - `decision_tags` — a many-to-many junction table linking decisions ↔ tags
  - `audit_log` — an append-only (immutable) history of every action
- **CHECK constraints** — enforce valid status values at the database level (not just in code)
- **Partial unique indexes** — e.g., only one non-deleted project per name
- **FK cascades** — deleting a decision cascades to its options and audit entries

### 11. TESTING (light grey)
A horizontal bar at the bottom representing the **test suite**.

- **pytest** — the test framework
- **36 integration tests** — they test the full stack from HTTP request down to database and back. They cover authentication, authorization, CRUD, status workflow, immutability, supersede, audit trail, and soft delete. Each test runs against a real PostgreSQL database (`decisio_test`) with full isolation.

### 12. THREE-LAYER VALIDATION (bottom box)
This shows Decisio's **defense-in-depth** validation strategy. Data is validated at three separate layers:

1. **TypeScript (compile time)** — catches type mismatches in the frontend before code even runs. If a component tries to pass a string where a number is expected, the build fails.
2. **Marshmallow (runtime)** — validates every incoming API request against a schema. Rejects missing fields, wrong types, or invalid values with clear error messages.
3. **PostgreSQL (database safety)** — CHECK constraints, NOT NULL, unique indexes, and foreign keys enforce data integrity at the lowest level. Even if code has a bug, the database won't let bad data in.

If any one layer fails, the others still catch the problem. This is why no invalid data can reach the database.
