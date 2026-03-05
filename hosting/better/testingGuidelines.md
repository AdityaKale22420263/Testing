# Decisio — Testing Guidelines

## Overview

Decisio uses **pytest** as its testing framework. All tests are automated integration tests that exercise the REST API endpoints against a real PostgreSQL database, ensuring the full request lifecycle (routing → middleware → service → database → response) is verified.

| Metric          | Value          |
|-----------------|----------------|
| **Framework**   | pytest 9.0.2   |
| **Total Tests** | 36             |
| **Test Files**  | 5              |
| **Status**      | All Passing ✅ |

---

## Prerequisites

| Requirement    | Version  | Notes                                  |
|----------------|----------|----------------------------------------|
| Python         | 3.12+    | Installed and on PATH                  |
| PostgreSQL     | 16+      | Running on `localhost:5432`            |
| Test Database  | —        | A database named `decisio_test` must exist |

### Create the test database (one-time setup)

```sql
CREATE DATABASE decisio_test;
```

> The test suite uses a **separate database** (`decisio_test`) so your development data in `decisio_dev` is never affected.

---

## How to Run Tests

### 1. Activate the virtual environment

```powershell
cd backend
.\venv\Scripts\Activate.ps1        # Windows PowerShell
# source venv/bin/activate          # macOS / Linux
```

### 2. Run the full test suite

```bash
python -m pytest -v
```

**Expected output:**

```
============================= test session starts =============================
collected 36 items

tests/test_admin.py    ...   5 passed
tests/test_auth.py     ...   8 passed
tests/test_decisions.py ...  16 passed
tests/test_projects.py  ...  7 passed

============================= 36 passed =============================
```

### 3. Run a specific test file

```bash
python -m pytest tests/test_decisions.py -v
```

### 4. Run a specific test class or method

```bash
python -m pytest tests/test_decisions.py::TestSoftDelete -v
python -m pytest tests/test_decisions.py::TestSoftDelete::test_admin_can_soft_delete -v
```

---

## Test Architecture

```
backend/tests/
├── conftest.py          # Shared fixtures & helpers
├── test_auth.py         # Authentication (login, JWT, /me)
├── test_admin.py        # Admin-only access control
├── test_decisions.py    # Decision lifecycle (CRUD, workflow, delete)
├── test_projects.py     # Project CRUD & constraints
└── test_audit.py        # (Reserved for future audit-specific tests)
```

### Database Isolation

Each test runs in full isolation:

1. **Session-scoped** — tables are created once when the test session starts.
2. **Per-test rollback** — after every test, all database changes are rolled back and all tables are truncated.
3. No test can see data created by another test.

### Shared Fixtures (`conftest.py`)

| Fixture        | Scope    | Description                                      |
|----------------|----------|--------------------------------------------------|
| `app`          | session  | Creates the Flask app with `testing` config       |
| `setup_db`     | session  | Creates all tables at start, drops at end         |
| `db_session`   | function | Provides a clean DB for each test (auto-rollback) |
| `client`       | function | Flask test client for making HTTP requests        |
| `admin_user`   | function | Creates an admin user (`testadmin@decisio.local` / `admin123`) |
| `member_user`  | function | Creates a regular user (`testmember@decisio.local` / `member123`) |

**Helper functions:**

| Function           | Description                                 |
|--------------------|---------------------------------------------|
| `_login(client, email, password)` | Logs in and returns the JWT token |
| `_auth_header(token)` | Returns `{"Authorization": "Bearer <token>"}` |

---

## Test Inventory

### `test_auth.py` — Authentication (8 tests)

| # | Test | What It Verifies |
|---|------|------------------|
| 1 | `TestLogin::test_login_success` | Valid credentials return 200 with JWT token and user object |
| 2 | `TestLogin::test_login_wrong_password` | Wrong password returns 401 with "Invalid" error |
| 3 | `TestLogin::test_login_nonexistent_user` | Non-existent email returns 401 |
| 4 | `TestLogin::test_login_deactivated_user` | Deactivated user returns 401 with "deactivated" message |
| 5 | `TestLogin::test_login_validation_missing_fields` | Empty JSON body returns 400 (validation error) |
| 6 | `TestMe::test_me_authenticated` | `GET /api/auth/me` with valid JWT returns user info |
| 7 | `TestMe::test_me_no_token` | `GET /api/auth/me` without token returns 401 |
| 8 | `TestMe::test_me_invalid_token` | `GET /api/auth/me` with garbage token returns 401 |

### `test_admin.py` — Admin Access Control (5 tests)

| # | Test | What It Verifies |
|---|------|------------------|
| 1 | `test_member_cannot_access_users` | Regular member gets 403 on `GET /api/users` |
| 2 | `test_admin_can_list_users` | Admin gets 200 on `GET /api/users` |
| 3 | `test_admin_can_create_user` | Admin can `POST /api/users` and gets 201 |
| 4 | `test_member_cannot_create_user` | Regular member gets 403 on `POST /api/users` |
| 5 | `test_system_audit_admin_only` | `GET /api/audit` returns 403 for member, 200 for admin |

### `test_projects.py` — Project CRUD (7 tests)

| # | Test | What It Verifies |
|---|------|------------------|
| 1 | `test_create_project` | Creates a project, verifies name and creator |
| 2 | `test_create_duplicate_project` | Duplicate project name returns 409 Conflict |
| 3 | `test_list_projects` | `GET /api/projects` returns at least 1 project |
| 4 | `test_get_project` | `GET /api/projects/:id` returns correct project |
| 5 | `test_update_project` | `PATCH /api/projects/:id` updates description |
| 6 | `test_archive_project` | Setting `is_archived: true` archives the project |
| 7 | `test_unauthenticated_access` | Request without JWT token returns 401 |

### `test_decisions.py` — Decision Lifecycle (16 tests)

#### CRUD (3 tests)

| # | Test | What It Verifies |
|---|------|------------------|
| 1 | `test_create_decision` | Creates a decision with 2 options, status = "Draft" |
| 2 | `test_update_decision` | PATCH updates the title successfully |
| 3 | `test_list_decisions_by_project` | Filtering by `project_id` returns correct count |

#### Status Transitions (4 tests)

| # | Test | What It Verifies |
|---|------|------------------|
| 4 | `test_draft_to_proposed` | Member can advance Draft → Proposed |
| 5 | `test_proposed_to_decided_requires_admin` | Member gets 403 on Proposed → Decided; admin gets 200 |
| 6 | `test_cannot_skip_status` | Draft → Decided (skipping Proposed) returns 400 |
| 7 | `test_cannot_reverse_status` | Proposed → Draft (backward) returns 400 |

#### Immutability (1 test)

| # | Test | What It Verifies |
|---|------|------------------|
| 8 | `test_decided_cannot_be_edited` | PATCH on a Decided decision returns 400 "immutable" |

#### Supersede (2 tests)

| # | Test | What It Verifies |
|---|------|------------------|
| 9  | `test_supersede_decided_decision` | Creates a new Draft, marks old as Superseded with link |
| 10 | `test_cannot_supersede_draft` | Superseding a Draft returns 400 |

#### Audit Trail (2 tests)

| # | Test | What It Verifies |
|---|------|------------------|
| 11 | `test_audit_created_on_decision_create` | Creating a decision produces 1 audit entry (action: "created") |
| 12 | `test_audit_tracks_status_changes` | Two status changes produce 3 total audit entries |

#### Soft Delete (4 tests)

| # | Test | What It Verifies |
|---|------|------------------|
| 13 | `test_admin_can_soft_delete` | Admin DELETE returns 200; decision removed from list |
| 14 | `test_deleted_decision_returns_404` | GET on a deleted decision returns 404 |
| 15 | `test_non_admin_cannot_delete` | Regular member DELETE returns 403 |
| 16 | `test_cannot_delete_already_deleted` | Double-delete returns 400 "already deleted" |

---

## Test Coverage by Feature

| Feature | Tests | Key Assertions |
|---------|-------|----------------|
| **Authentication** | 8 | JWT issuance, invalid credentials, deactivated accounts, token validation |
| **Authorization** | 7 | Admin-only routes return 403 for members; all routes return 401 without token |
| **Project CRUD** | 7 | Create, read, update, archive; duplicate name constraint; unauthenticated access |
| **Decision CRUD** | 3 | Create with options, update fields, list with filters |
| **Status Workflow** | 4 | Forward-only transitions, admin-only approval, no skipping/reversing |
| **Immutability** | 1 | Decided decisions reject edits |
| **Supersede** | 2 | Creates replacement, links old → new, blocks non-Decided |
| **Audit Trail** | 2 | Automatic logging of create and status change events |
| **Soft Delete** | 4 | Admin-only delete, 404 after delete, no double-delete |

---

## Configuration

Tests use the `testing` configuration profile defined in `backend/config.py`:

```python
class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = os.getenv(
        "TEST_DATABASE_URL",
        "postgresql://postgres:postgres@localhost:5432/decisio_test",
    )
```

To use a custom database URL, set the `TEST_DATABASE_URL` environment variable:

```powershell
$env:TEST_DATABASE_URL = "postgresql://user:pass@host:5432/decisio_test"
python -m pytest -v
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `FATAL: database "decisio_test" does not exist` | Run `CREATE DATABASE decisio_test;` in psql |
| `connection refused` | Ensure PostgreSQL is running on port 5432 |
| `role "postgres" does not exist` | Set `TEST_DATABASE_URL` with your actual DB credentials |
| Tests pass individually but fail together | Should not happen — each test has full isolation. Check for fixture issues. |
