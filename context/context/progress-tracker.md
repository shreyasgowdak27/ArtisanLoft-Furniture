# Progress Tracker — ArtisanLoft

HOW TO USE: Read this FIRST every session. UPDATE it at the END of every session (what was done, what is next, any problems). Keep it short and accurate. Stale context is worse than none.

## Current status
- Current phase: **Phase 4 — Cart** (Backend complete)
- Last updated: 2026-10-25
- Auth backend: complete (Supabase JWT verification via JWKS + PyJWT). Auth frontend: complete (Supabase client, AuthContext, Login, Signup, Navbar, Profile with GET /me, ProtectedRoute). Cart backend: complete with Postgres cart items, authenticated API routes, and migration `c3e8f941d2a0`. Cart frontend: not started. Checkout/payments: not started. Admin: not started.

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
- [x] Phase 3 (Backend): Replaced custom password/token authentication with Supabase Auth JWT verification:
  - Installed `pyjwt[crypto]` in `requirements.txt`
  - Implemented `verify_supabase_jwt` in `app/core/security.py` using PyJWT's `PyJWKClient` against `{SUPABASE_URL}/auth/v1/.well-known/jwks.json`, enforcing RS256/ES256 and audience `authenticated`
  - Removed custom password hashing (`passlib`, `bcrypt`) and custom token creation (`create_access_token`)
  - Removed `/auth/signup` and `/auth/login` custom endpoints
  - Added FastAPI dependency `get_current_user` in `app/core/deps.py` returning `CurrentUser(id, email)`
  - Kept protected `GET /me` (mounted at `/me`, `/api/v1/me`, `/api/auth/me`) using `get_current_user`
  - Mounted `auth_router` in `app/main.py`
  - Added comprehensive test suite in `tests/test_auth.py` covering 401s, 200s, RS256, ES256, expired tokens, and dependency overrides (all 8 tests passing)
- [x] Phase 3 (Frontend): Connected React frontend to Supabase Auth:
  - Created `src/lib/supabase.js` using `createClient` with `REACT_APP_SUPABASE_URL` and `REACT_APP_SUPABASE_ANON_KEY`
  - Refactored `AuthContext.js` to manage session/user state via `supabase.auth.signUp`, `signInWithPassword`, `signOut`, and `onAuthStateChange`
  - Updated `Login.jsx` and `Signup.jsx` for real email/password authentication with validation and error handling
  - Updated `Navbar.jsx` with responsive Login/Signup links or user email with Logout action
  - Configured Axios interceptor in `api.js` to attach Supabase access token as Bearer header on API requests
  - Updated `Profile.jsx` to call `GET /me` from the backend to verify the authenticated session
  - Added `ProtectedRoute.jsx` component to safeguard authenticated pages
  - Production build verified with zero errors

## In progress
- Phase 4 cart frontend and guest cart behavior are not started.

## Next up (in order)
1. Phase 4: Cart frontend, guest cart behavior, and authenticated cart integration.
2. First early deploy after the cart flow is complete.

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
- 2026-10-25: Built the cart backend using Supabase Postgres. Added the `cart_items` migration, atomic add route, owner scoped read and mutation routes, and tests. Migration `c3e8f941d2a0` is applied to Supabase.
