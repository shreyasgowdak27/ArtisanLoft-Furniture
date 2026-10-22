# Progress Tracker — ArtisanLoft

HOW TO USE: Read this FIRST every session. UPDATE it at the END of every session (what was done, what is next, any problems). Keep it short and accurate. Stale context is worse than none.

## Current status
- Current phase: **Phase 3 — Authentication** (Phase 2 complete)
- Last updated: 2026-10-22
- Auth: not started. Cart: not started. Checkout/payments: not started. Admin: not started.

## Completed
- [x] Phase 1: FastAPI backend (`backend-fastapi`) scaffolded; real product models/API built from old `data.sql`; React frontend connected to local FastAPI; homepage loads 30 real products end-to-end.
- [x] Fixed product image mismatch: Unsplash API used to fetch unique, correctly matched images for all 30 products; database re-seeded and verified.
- [x] Phase 2: Migrated from SQLite → Supabase Postgres via SQLAlchemy 2.0 async (asyncpg) + Alembic.
  - Installed: asyncpg, greenlet, sqlalchemy[asyncio], psycopg[binary], alembic
  - New structure: `app/db/session.py` (async engine), `app/db/base.py` (DeclarativeBase), `app/core/config.py` (pydantic-settings)
  - Product model: UUID PK, price_cents (BigInteger), slug (unique index), stock, is_active, created_at/updated_at
  - Alembic migration `a8d7615f99e9_create_products_table` applied to Supabase Postgres
  - All 30 products seeded with Unsplash image URLs. Seed is idempotent (run again → updates, not duplicates)
  - `/api/v1/products` and `/api/v1/products/{slug}` endpoints (async, SQLAlchemy 2.0 style)
  - `/health` (liveness) and `/health/ready` (DB check) added
  - SQLite file (`artisanloft.db`) deleted; all SQLite code removed
  - Frontend API URL updated: `/api/v1/products` from `http://127.0.0.1:8000`
  - `.env.example` and `.gitignore` updated; `requirements.txt` reflects async stack

## In progress
- Nothing yet.

## Next up (in order)
1. Phase 3: Authentication (Supabase Auth in frontend + FastAPI JWT verification + profiles table).
2. First early deploy right after Phase 3.

## Decisions made
- Backend rewritten in Python FastAPI; Java Spring Boot backend is legacy.
- Stack locked: Supabase (Postgres + Auth + Storage), Redis (Upstash), Razorpay test mode, PostHog, Vercel + Render, k6 for load tests.
- No custom password handling; Supabase Auth only.
- asyncpg chosen as the Postgres async driver (greenlet required for SQLAlchemy async mode on Python 3.14).
- Prepared statement caching disabled (`statement_cache_size=0`) — required by Supabase PgBouncer transaction pooler.
- price_cents (BigInteger) stores money in paise; `price` field on the API response is a derived float (price_cents / 100) for frontend compatibility.
- Auth router (legacy JWT/password) left untouched for Phase 3 rework; not mounted in main.py.

## Known issues / blockers
- `app/database.py`, `app/config.py`, `app/routers/auth.py`, `app/models/user.py` are legacy shims — will be cleaned up or replaced in Phase 3.
- `createOrder` removed from `frontend/src/api/api.js` (not used anywhere; Phase 5 will add it properly).

## Session log
- 2026-10-07: Defined final stack, wrote context files, installed JSM skills in Cursor.
- 2026-10-22: Phase 2 complete — SQLite → Supabase Postgres. Full async stack (asyncpg + SQLAlchemy 2.0 + Alembic). 30 products seeded. End-to-end verified.
