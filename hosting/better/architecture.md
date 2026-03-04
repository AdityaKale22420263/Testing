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
