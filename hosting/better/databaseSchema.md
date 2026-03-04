# Decisio — Database Schema Explained

A deep-dive into every table, column, constraint, index, and relationship in the
Decisio database, with the **reasoning** behind each design choice.

---

## At a Glance

| Table | Purpose | Rows (seed) |
|---|---|---|
| `users` | Everyone who can log in | 6 |
| `projects` | Groups related decisions together | 4 |
| `decisions` | Core entity — a recorded technical decision | 9 |
| `options` | Alternatives considered for a decision | ~18 |
| `tags` | Labels for cross-cutting categorisation | 9 |
| `decision_tags` | Many-to-many join table | ~21 |
| `audit_log` | Immutable history of every mutation | ~21 |

---

## 1. `users`

```
id            INTEGER  PK
name          VARCHAR(100)  NOT NULL
email         VARCHAR(255)  NOT NULL  UNIQUE
password      VARCHAR(255)  NOT NULL
is_admin      BOOLEAN  NOT NULL  DEFAULT false
is_active     BOOLEAN  NOT NULL  DEFAULT true
created_at    TIMESTAMP  NOT NULL  DEFAULT now()
updated_at    TIMESTAMP  NOT NULL  DEFAULT now()  ON UPDATE now()
```

### Why this design?

| Column | Reasoning |
|---|---|
| `email UNIQUE` | Email is the login identifier. Uniqueness is enforced at the DB level so even a race condition between two concurrent signup requests can't create duplicates. |
| `password VARCHAR(255)` | Stores a **Bcrypt hash**, not the raw password. Bcrypt hashes are 60 chars but we use 255 to leave room if we ever switch to Argon2 (longer output). |
| `is_admin` | Simple two-role model — admin vs member. Admins can create users, deactivate accounts, and see all audit logs. A boolean is the simplest RBAC for an MVP; a full `roles` table would be over-engineering at this stage. |
| `is_active` | **Soft deactivation** instead of deleting users. If you delete a user row, every foreign key pointing to them (decisions, audit logs, projects) would need `ON DELETE SET NULL` and you'd lose the "who created this" history. Deactivating keeps referential integrity intact. |
| `updated_at` | Auto-updates on every write via SQLAlchemy's `onupdate` hook. Useful for cache invalidation and "last active" displays. |

### Why NOT do…

- **Email verification?** — Out of scope for a 48-hour MVP. Would need SMTP config, token table, and a confirmation flow.
- **Separate `roles` table?** — Only two roles exist. A join table for roles adds 3 tables (`roles`, `user_roles`, `permissions`) with no MVP benefit.

---

## 2. `projects`

```
id            INTEGER  PK
name          VARCHAR(150)  NOT NULL  UNIQUE
description   TEXT  (nullable)
created_by    INTEGER  FK → users.id  NOT NULL
is_archived   BOOLEAN  NOT NULL  DEFAULT false
created_at    TIMESTAMP  NOT NULL  DEFAULT now()
updated_at    TIMESTAMP  NOT NULL  DEFAULT now()  ON UPDATE now()
```

### Why this design?

| Column | Reasoning |
|---|---|
| `name UNIQUE` | Projects are the top-level organising unit. Two projects called "Backend Rewrite" would cause confusion in dropdowns and search results. |
| `description TEXT` | Nullable on purpose — not every project needs a description. Using `TEXT` instead of `VARCHAR` because project descriptions can be lengthy. |
| `created_by FK → users` | Tracks who created the project. Doesn't mean "owner" in a permissions sense (any logged-in user can add decisions to any project), but it's useful for accountability. |
| `is_archived` | Soft archive instead of delete. Archived projects stop appearing in active lists but their decisions and audit history remain intact. Deleting a project would cascade-delete all its decisions and destroy historical data. |

### Relationship: `projects → decisions`

```python
decisions = relationship("Decision", cascade="all, delete-orphan")
```

This means if a project **is** actually deleted (hard delete), all its decisions go with it. But because we prefer archiving, this cascade is a safety net, not the primary strategy.

---

## 3. `decisions` ⭐ (Core Table)

```
id              INTEGER  PK
project_id      INTEGER  FK → projects.id  NOT NULL
title           VARCHAR(255)  NOT NULL
context         TEXT  NOT NULL
final_summary   TEXT  NOT NULL  DEFAULT ''
status          VARCHAR(20)  NOT NULL  DEFAULT 'Draft'
created_by      INTEGER  FK → users.id  NOT NULL
decision_date   DATE  NOT NULL  DEFAULT today()
superseded_by   INTEGER  FK → decisions.id  (nullable, self-reference)
is_deleted      BOOLEAN  NOT NULL  DEFAULT false
created_at      TIMESTAMP  NOT NULL  DEFAULT now()
updated_at      TIMESTAMP  NOT NULL  DEFAULT now()  ON UPDATE now()
```

This is the most complex table. Let's break down every choice.

### 3a. Status as VARCHAR, not a PostgreSQL ENUM

```python
VALID_STATUSES = ("Draft", "Proposed", "Decided", "Superseded")
```

**Why VARCHAR over ENUM?**
PostgreSQL `ENUM` types require `ALTER TYPE` DDL to add new values, and you
can't remove or reorder values at all. A `VARCHAR(20)` with application-level
validation is far easier to evolve. The trade-off is slightly weaker DB-level
enforcement, which we compensate for with Marshmallow schema validation and
service-layer checks.

### 3b. The Status State Machine

```
Draft → Proposed → Decided → Superseded
```

**Why a strict, forward-only state machine?**

This mirrors real-world decision governance:

1. **Draft** — Someone is still writing it up. Editable freely.
2. **Proposed** — Put forward for team review. The fact that it moved here is
   logged in the audit trail so you know when deliberation started.
3. **Decided** — The team committed to this approach. After this point,
   **the decision becomes immutable** — you can't edit the title, context,
   or options anymore. This is the key design philosophy: *a decided decision
   is a historical record*. If it could be silently edited, the audit trail
   becomes meaningless.
4. **Superseded** — A newer decision replaced this one. The old decision
   isn't deleted; it stays visible with a pointer to what replaced it.

**No skipping steps.** You can't go from Draft → Decided because that bypasses
the proposal/review phase. You can't go backwards (Decided → Proposed) because
that would let someone un-commit a decision the team already agreed on.

**Why not allow backwards?** In decision management, going backwards is a
sign of process failure. If a decided decision turns out to be wrong, the
correct action is to supersede it with a new decision that explains *why*
the old approach failed — not to silently revert it and pretend it never
happened.

### 3c. The `superseded_by` Self-Reference

```sql
superseded_by  INTEGER  FK → decisions.id  (nullable)
```

This is a **self-referential foreign key** — a decision points to another
decision in the same table.

**Why self-reference instead of a separate `supersessions` table?**

A separate table (`old_decision_id`, `new_decision_id`) would allow:
- Many-to-many supersession (decision A superseded by both B and C)
- Chains of supersession with different semantics

But in reality, supersession is **1:1** — one old decision is replaced by
exactly one new decision. A self-referential FK is the simplest way to model
this. It also means a single query can show "this decision was superseded by
Decision #42" without a join.

**The flow:**
1. Decision #1 exists in status "Decided"
2. User creates Decision #2 and says "this supersedes #1"
3. The service layer atomically:
   - Sets `Decision #1.status = 'Superseded'`
   - Sets `Decision #1.superseded_by = 2`
   - Creates an audit log entry

Decision #2 is the new reality. Decision #1 is preserved forever as historical
record, with a clear pointer to its replacement.

### 3d. Constraints on `superseded_by`

Three constraints work together to enforce correctness:

#### CHECK: `no_self_supersede`
```sql
CHECK (superseded_by != id)
```
**Why?** A decision superseding itself is nonsensical — it would create a
self-referencing loop. This constraint catches bugs at the database level
even if the application code has a logic error.

#### CHECK: `supersede_status_sync`
```sql
CHECK ((superseded_by IS NULL) OR (status = 'Superseded'))
```
**Why?** These two columns must stay in sync:
- If `superseded_by` is set, the status **must** be 'Superseded'
- If status is anything else, `superseded_by` must be NULL

Without this constraint, you could have a decision in "Draft" status that
claims to be superseded by another decision — a contradictory state. The
CHECK constraint makes this **impossible at the database level**.

#### Partial Unique Index: `idx_decisions_superseded_by`
```sql
CREATE UNIQUE INDEX idx_decisions_superseded_by
  ON decisions (superseded_by)
  WHERE superseded_by IS NOT NULL;
```
**Why partial?** A normal `UNIQUE` on `superseded_by` would fail because many
decisions have `superseded_by = NULL`, and NULL doesn't equal NULL in SQL.
The `WHERE superseded_by IS NOT NULL` filter means:
- Multiple decisions can have NULL (not superseded) — that's fine
- But if `superseded_by` has a value, it must be unique across the table

This enforces **1:1 supersession** — two different decisions can't both point
to the same replacement decision. If Decision #5 supersedes #1, no other
decision can also claim to be superseded by #5.

### 3e. `is_deleted` — Soft Deletes

**Why not hard delete?**

If you `DELETE FROM decisions WHERE id = 5`, you lose:
- The audit trail (all audit_log entries reference decision_id=5 — FK violation)
- The supersession chain (if decision #3 is superseded by #5, deleting #5 breaks the pointer)
- Historical data for reporting

Soft delete (`is_deleted = true`) keeps the row in the database but hides it
from normal queries. The decision's audit history, supersession chain, and
project association all remain intact.

**Trade-off:** You need to add `WHERE is_deleted = false` to every query.
Our service layer handles this centrally.

### 3f. `context` vs `final_summary`

| Field | When filled | Purpose |
|---|---|---|
| `context` | At creation time | *Why* are we making this decision? What's the problem? What constraints exist? |
| `final_summary` | When decided | *What* did we decide and *why*? The executive summary of the outcome. |

Separating these lets you capture the journey: the problem statement stays in
`context`, the conclusion goes in `final_summary`. Someone reading the decision
6 months later sees both.

### 3g. Indexes

```
idx_decisions_project_id   → project_id     (filter by project)
idx_decisions_status       → status         (filter by status)
idx_decisions_created_by   → created_by     (my decisions)
```

These cover the three most common query patterns:
1. "Show me all decisions in Project X"
2. "Show me all Draft decisions"
3. "Show me all decisions I created"

---

## 4. `options`

```
id            INTEGER  PK
decision_id   INTEGER  FK → decisions.id  ON DELETE CASCADE  NOT NULL
title         VARCHAR(255)  NOT NULL
pros          TEXT  DEFAULT ''
cons          TEXT  DEFAULT ''
is_chosen     BOOLEAN  NOT NULL  DEFAULT false
position      SMALLINT  NOT NULL  DEFAULT 0
created_at    TIMESTAMP  NOT NULL  DEFAULT now()
```

### Why this design?

| Column | Reasoning |
|---|---|
| `decision_id ON DELETE CASCADE` | If a decision is hard-deleted (rare, but possible), its options have no reason to exist independently. Cascade means they disappear automatically — no orphaned rows. |
| `pros` / `cons` as `TEXT` | Free-form text lets users list as many points as they want. An alternative design would be a separate `pros_cons` table with one row per point, but that's over-engineering for an MVP. |
| `is_chosen` | Boolean flag for which option was selected. Only relevant when the decision reaches "Decided" status. |
| `position` | Display order. Allows users to reorder options via drag-and-drop without renaming. `SMALLINT` because you'll never have 32,000 options on a decision. |

### Partial Unique Index: One Chosen Option Per Decision

```sql
CREATE UNIQUE INDEX idx_one_chosen_option_per_decision
  ON options (decision_id)
  WHERE is_chosen = true;
```

**Why?** A decision can have many options, but only **one** can be the chosen
outcome. Without this index, a bug could mark two options as `is_chosen = true`
for the same decision — the database would happily allow it.

The partial unique index says: "among all rows where `is_chosen = true`,
`decision_id` must be unique." This guarantees exactly zero or one chosen
option per decision, enforced at the database level.

**Why not enforce this only in application code?** Because concurrent requests
could both try to mark different options as chosen at the same time. The
database index makes this a serialized constraint — only one can win.

---

## 5. `tags`

```
id            INTEGER  PK
name          VARCHAR(50)  NOT NULL  UNIQUE
created_at    TIMESTAMP  NOT NULL  DEFAULT now()
```

### Why this design?

Tags are intentionally simple — just a name. They're a flat taxonomy, not
hierarchical. Examples: `"security"`, `"performance"`, `"api-design"`.

| Column | Reasoning |
|---|---|
| `name UNIQUE` | Prevents duplicate tags. Without this, you'd end up with both "Security" and "security" (capitalised differently). The uniqueness constraint plus application-level normalisation keeps the tag list clean. |
| `VARCHAR(50)` | Long enough for descriptive tags like `"database-migration-strategy"` but short enough to discourage using tags as descriptions. |

**Why not a `created_by` FK?** Tags are a shared global resource. It doesn't
matter who created the tag — anyone can use any tag. Adding a creator would
suggest ownership, which isn't the intended model.

---

## 6. `decision_tags` (Join Table)

```
decision_id   INTEGER  FK → decisions.id  ON DELETE CASCADE  PK
tag_id        INTEGER  FK → tags.id  ON DELETE CASCADE       PK
```

### Why this design?

This is a standard many-to-many join table. A decision can have multiple tags,
and a tag can be applied to multiple decisions.

**Composite primary key** (`decision_id`, `tag_id`) serves double duty:
1. **Primary key** — uniquely identifies each row
2. **Unique constraint** — prevents the same tag from being applied to the same
   decision twice

**`ON DELETE CASCADE` on both sides:**
- Decision deleted → its tag associations disappear
- Tag deleted → it's removed from all decisions

No `created_at` on the join table — we don't care *when* a tag was applied,
only *that* it's applied.

---

## 7. `audit_log`

```
id            INTEGER  PK
decision_id   INTEGER  FK → decisions.id  NOT NULL
actor_id      INTEGER  FK → users.id  NOT NULL
action        VARCHAR(100)  NOT NULL
old_status    VARCHAR(20)  (nullable)
new_status    VARCHAR(20)  (nullable)
note          TEXT  (nullable)
created_at    TIMESTAMP  NOT NULL  DEFAULT now()
```

### Why this design?

The audit log is the **single source of truth** for what happened to a decision.
The assessment spec calls for tracking changes — this table does it.

| Column | Reasoning |
|---|---|
| `decision_id` | Which decision was affected. Every audit entry is tied to exactly one decision. |
| `actor_id` | Who did it. Foreign key to `users` ensures we always know the responsible person. |
| `action` | What happened: `"created"`, `"updated"`, `"status_changed"`, `"superseded"`. Free-form `VARCHAR(100)` instead of an enum so we can add new action types without schema changes. |
| `old_status` / `new_status` | Nullable — only populated for status changes. Captures the before/after so you can reconstruct the full timeline: Draft → Proposed → Decided → Superseded. |
| `note` | Optional context. E.g., "Superseded because the legal team required a different approach." |
| `created_at` | When it happened. No `updated_at` — **audit entries are immutable**. Once written, they should never be modified. |

### Why immutable?

If you could edit audit entries, the entire audit trail becomes untrustworthy.
The model intentionally has **no** `updated_at` column and the service layer
only ever calls `db.session.add()` — never updates existing entries.

### Indexes

```
idx_audit_log_decision_id  → decision_id   (get all history for decision X)
idx_audit_log_actor_id     → actor_id      (what did user Y do?)
```

The `decision_id` index is critical because the most common query is "show me
the full audit trail for this decision" — the detail page loads it eagerly.

---

## Relationships Summary

```
users ──┬── 1:N ──→ projects        (creator)
        ├── 1:N ──→ decisions       (creator)
        └── 1:N ──→ audit_log       (actor)

projects ── 1:N ──→ decisions

decisions ──┬── 1:N ──→ options
            ├── M:N ──→ tags          (via decision_tags)
            ├── 1:N ──→ audit_log
            └── 1:1 ──→ decisions     (superseded_by — self-reference)
```

---

## The Three-Layer Validation Strategy

One of the key architectural decisions: validation happens at **three** levels,
each catching different classes of errors.

```
Layer 1: TypeScript Types (frontend)
  ↓ catches typos, wrong types before HTTP request is even sent
Layer 2: Marshmallow Schemas (backend)
  ↓ catches invalid input structure, missing fields, bad values
Layer 3: PostgreSQL Constraints (database)
  ↓ catches concurrent race conditions, data corruption, bugs
```

### Why all three?

| Scenario | Which layer catches it? |
|---|---|
| User types a string into a number field | TypeScript (compile time) |
| Status set to `"Approved"` (not a valid status) | Marshmallow (runtime, returns 400) |
| Two concurrent requests both try to mark different options as chosen | PostgreSQL (partial unique index violation) |
| A bug in the service layer sets `superseded_by` to the same decision's ID | PostgreSQL (`no_self_supersede` CHECK) |
| A request bypasses the UI entirely (curl / Postman) | Marshmallow + PostgreSQL (UI validation is irrelevant) |

No single layer catches everything. The database is the **last line of defence**
— if application code has a bug, the constraints prevent data corruption.

---

## Design Decisions I'd Change in Production

| Current MVP | Production Alternative | Why I didn't do it in 48 hours |
|---|---|---|
| `is_admin` boolean | Full RBAC (roles table + permissions table) | Only two roles exist in the assignment |
| `VARCHAR` statuses | PostgreSQL ENUM or a `statuses` lookup table | ENUMs are painful to migrate; VARCHAR is good enough |
| No pagination | Cursor-based pagination on all list endpoints | Only ~10 decisions in seed data |
| No full-text search | PostgreSQL `tsvector` + GIN index on title/context | Would need search indexes and ranking logic |
| Single `superseded_by` FK | A `decision_history` table tracking chains | 1:1 supersession covers the spec; chains are future scope |
| No file attachments | S3 + `attachments` table | File upload is a whole subsystem |
| Bcrypt password hashing | Argon2id | Bcrypt is still secure; Argon2 is the modern standard |
