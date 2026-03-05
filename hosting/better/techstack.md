# Decisio — Tech Stack & Reasoning

Every technology choice with the reasoning behind it.

---

## Frontend

| Technology | Version | What It Does | Why We Chose It |
|---|---|---|---|
| **Next.js** | 16.1.6 | React meta-framework (routing, SSR, build tooling) | Industry standard for React apps. Gives us file-based routing, fast dev server, optimised production builds. We use it as a SPA (client-side only) — no SSR needed for this MVP, but the option is there for future SEO pages. |
| **React** | 19.2.4 | UI component library | The most widely adopted UI library. Component model makes it easy to build reusable pieces. React 19 brings improved server components and hooks. |
| **TypeScript** | 5.7.3 | Static type checking for JavaScript | Catches bugs at compile time instead of runtime. Every API response has a typed interface — if the backend changes shape, TypeScript flags it immediately. Essential for a team project. |
| **Tailwind CSS** | 4.2.0 | Utility-first CSS framework | No context-switching between CSS files and components. Styles live right in the markup. v4 uses CSS-native cascade layers and is significantly faster. No unused CSS in production. |
| **ShadCN / Radix UI** | Various | Headless UI component primitives (dialogs, dropdowns, tabs, etc.) | Pre-built accessible components that are unstyled by default — we control the look with Tailwind. Radix handles keyboard navigation, focus trapping, ARIA attributes. We get 20+ accessible components without writing any a11y code ourselves. |
| **Recharts** | 2.15.0 | Charting library (built on D3) | Dashboard needs charts for decision stats. Recharts is React-native (no DOM manipulation), declarative, and lightweight compared to raw D3. |
| **React Hook Form** | 7.54.1 | Form state management | Handles validation, dirty tracking, and submission without re-rendering the entire form on every keystroke. Performant for complex forms like the decision creation form with dynamic options. |
| **Zod** | 3.24.1 | Schema validation for TypeScript | Defines form validation schemas that integrate with React Hook Form via `@hookform/resolvers`. Gives us runtime validation on the frontend that mirrors our Marshmallow schemas on the backend. |
| **Lucide React** | 0.564.0 | Icon library | Clean, consistent SVG icons. Tree-shakeable — only the icons we import end up in the bundle. |
| **date-fns** | 4.1.0 | Date utility library | Lightweight date formatting and manipulation. We use it for displaying decision dates, "time ago" labels, etc. Much smaller than Moment.js and tree-shakeable. |
| **next-themes** | 0.4.6 | Theme switching (light/dark mode) | Handles system preference detection, localStorage persistence, and flash-free theme switching. One import, zero config. |
| **class-variance-authority** | 0.7.1 | Component variant management | Defines button/badge/card variants in a type-safe way. Works with Tailwind — e.g., `variant="destructive"` maps to red styles. Used by ShadCN components internally. |
| **tailwind-merge** | 3.3.1 | Tailwind class deduplication | Merges conflicting Tailwind classes intelligently — `"px-4 px-6"` becomes `"px-6"`. Prevents style bugs when composing components. |
| **clsx** | 2.1.1 | Conditional class joining | Cleaner than string concatenation for conditional CSS classes: `clsx("base", isActive && "active")`. |
| **cmdk** | 1.1.1 | Command palette component | Powers the search/command interface. Provides fuzzy search, keyboard navigation, and accessible list UI out of the box. |
| **sonner** | 1.7.1 | Toast notification library | Beautiful, animated toast notifications with zero config. Shows success/error messages after API operations. |
| **vaul** | 1.1.2 | Drawer component | Mobile-friendly drawer/sheet component used for the sidebar on smaller screens. |
| **embla-carousel-react** | 8.6.0 | Carousel component | Lightweight carousel used by the ShadCN carousel primitive. |
| **react-day-picker** | 9.13.2 | Calendar / date picker | Powers the date picker used in the decision form for `decision_date`. Accessible and customisable. |
| **react-resizable-panels** | 2.1.7 | Resizable split panels | Used for the sidebar + main content layout. Users can drag to resize. |
| **input-otp** | 1.4.2 | OTP input component | ShadCN dependency for one-time-password inputs. Included in the component library even though we don't use OTP auth — comes with the ShadCN install. |
| **PostCSS** | 8.5.x | CSS processing pipeline | Required by Tailwind CSS v4. Processes `@import`, autoprefixing, and Tailwind directives. |

---

## Backend

| Technology | Version | What It Does | Why We Chose It |
|---|---|---|---|
| **Python** | 3.12 | Programming language | Clean syntax, massive ecosystem, fast enough for a web API. Python 3.12 has significant performance improvements and better error messages. |
| **Flask** | 3.1.0 | Web framework (micro-framework) | Lightweight and unopinionated — lets us structure the code exactly how we want (factory pattern, blueprints, service layer). Django would give us an admin panel and ORM for free, but also forces a project structure and adds weight we don't need. Flask gives us the flexibility to build a clean layered architecture. |
| **Flask-SQLAlchemy** | 3.1.1 | SQLAlchemy integration for Flask | SQLAlchemy is the most powerful Python ORM. Flask-SQLAlchemy adds session management and app integration. We get relationship mapping, eager/lazy loading, and full control over queries without writing raw SQL. |
| **Flask-Migrate** | 4.1.0 | Database migrations (Alembic wrapper) | Auto-generates migration scripts from model changes. Run `flask db migrate` → `flask db upgrade` and the database schema updates. No manual SQL needed. Essential for evolving the schema over time. |
| **Flask-Bcrypt** | 1.0.1 | Password hashing | Bcrypt is a slow-by-design hashing algorithm — resistant to brute force. One-line API: `bcrypt.generate_password_hash(password)`. Stores a 60-char hash that includes the salt, so no separate salt column needed. |
| **Flask-CORS** | 5.0.1 | Cross-Origin Resource Sharing | Frontend runs on `localhost:3000`, backend on `localhost:5000`. Without CORS headers, the browser blocks cross-origin requests. Flask-CORS adds the right headers automatically. |
| **PyJWT** | 2.10.1 | JSON Web Token encoding/decoding | Generates and validates JWT tokens for stateless authentication. HS256 algorithm with a server-side secret. Tokens contain `user_id` and `exp` (expiry) claims. |
| **Marshmallow** | 3.25.1 | Request validation & serialisation | Validates every API input against a schema — required fields, type checks, length limits, enum values. Returns structured error messages (e.g., `{"title": ["Length must be between 1 and 255."]}`). Independent of Flask — can be used in services or tests. |
| **psycopg2-binary** | 2.9.10 | PostgreSQL database adapter | The standard Python driver for PostgreSQL. The `-binary` version includes pre-compiled C extensions — no system-level build tools needed on install. |
| **python-dotenv** | 1.1.0 | Environment variable loading | Reads `.env` files and loads them into `os.environ`. Keeps secrets (DB password, JWT secret) out of source code. |

---

## Database

| Technology | Version | What It Does | Why We Chose It |
|---|---|---|---|
| **PostgreSQL** | 18 | Relational database | The most advanced open-source RDBMS. We chose it over SQLite because we need: CHECK constraints, partial unique indexes, proper concurrent access, and JSONB support for future needs. It also enforces foreign keys strictly (unlike MySQL in some modes). |

### Why not SQLite?
SQLite lacks partial unique indexes and CHECK constraint enforcement is limited. We have two partial unique indexes (`idx_one_chosen_option_per_decision`, `idx_decisions_superseded_by`) that are critical to data integrity — they can't be expressed in SQLite.

### Why not MongoDB?
Our data is fundamentally relational — decisions belong to projects, have options, reference other decisions via supersession. A document database would denormalise this and make cross-references messy. The audit trail needs transactional consistency (write audit log + update decision atomically) — relational databases handle this natively.

---

## Testing

| Technology | Version | What It Does | Why We Chose It |
|---|---|---|---|
| **pytest** | 9.0.2 | Test framework | Python's most popular test framework. Fixtures for setup/teardown, parameterised tests, clear assertion output. Our `conftest.py` creates a fresh app + test database per session. |

---

## Dev Tools

| Technology | What It Does | Why |
|---|---|---|
| **pnpm** | Package manager (frontend) | Faster and more disk-efficient than npm. Uses symlinks instead of copying — installs are near-instant after first run. |
| **ESLint** | JavaScript/TypeScript linter | Catches code quality issues and enforces style consistency across the frontend codebase. |
| **Alembic** (via Flask-Migrate) | Migration version control | Each migration is a versioned script. Can upgrade and downgrade the database schema. History is tracked in a `migrations/` folder. |
| **GitHub Copilot (Claude)** | AI pair programming | Used throughout development for scaffolding, implementation, and iteration. AI guidance files (`CLAUDE.md`, `architecture-decisions.md`, `coding-standards.md`) kept output consistent. |

---

## Summary by Category

```
FRONTEND (11 core libs)
  Framework:    Next.js 16 + React 19
  Language:     TypeScript 5.7
  Styling:      Tailwind CSS v4 + ShadCN/Radix UI
  Forms:        React Hook Form + Zod
  Charts:       Recharts
  Icons:        Lucide React
  Dates:        date-fns
  UX:           sonner (toasts), cmdk (search), next-themes (dark mode)

BACKEND (9 packages)
  Framework:    Flask 3.1
  Language:     Python 3.12
  ORM:          SQLAlchemy (via Flask-SQLAlchemy)
  Migrations:   Alembic (via Flask-Migrate)
  Auth:         PyJWT + Flask-Bcrypt
  Validation:   Marshmallow
  Database:     psycopg2-binary → PostgreSQL 18
  Config:       python-dotenv
  CORS:         Flask-CORS

DATABASE
  PostgreSQL 18 — relational integrity, CHECK constraints, partial unique indexes

TESTING
  pytest 9.0 — 32 integration tests, separate test database
```
