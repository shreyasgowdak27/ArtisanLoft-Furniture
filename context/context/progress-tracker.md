# Progress Tracker — ArtisanLoft

HOW TO USE: Read this FIRST every session. UPDATE it at the END of every session (what was done, what is next, any problems). Keep it short and accurate. Stale context is worse than none.

## Current status
- Current phase: **Phase 4 — Cart** (Backend & Frontend complete)
- Last updated: 2026-10-28
- Auth backend: complete (Supabase JWT verification via JWKS + PyJWT). Auth frontend: complete (Supabase client, AuthContext, Login, Signup, Navbar, Profile with GET /me, ProtectedRoute). Cart backend: complete with Postgres cart items, authenticated API routes, and migration `c3e8f941d2a0`. Cart frontend: complete with API integration, ProductCard and ProductDetail Add-to-cart with auth redirection, Cart page (items, image, price, quantity controls, remove, subtotal), loading/empty/error states with over-stock error display. Checkout/payments: not started. Admin: not started.

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
- [x] Phase 4 (Cart): Backend & Frontend complete:
  - Postgres `cart_items` table with atomic upsert and correlated product stock checks.
  - Cart frontend wired to FastAPI backend (`/api/v1/cart`, `/api/v1/cart/items`) using Supabase JWT.
  - ProductCard and ProductDetail "Add to Cart" buttons with login redirect and stock capping.
  - Cart page with item images, price, quantity controls, remove buttons, order summary, and loading/empty/error states.
  - Navbar cart badge reflecting real cart count.
- [x] Phase 5 (Orders Backend): Spec 0002 implemented:
  - Replaced legacy `order.py` with Postgres-native `Order` and `OrderItem` models with UUID PKs, BigInteger money, price/name snapshots.
  - Added Alembic migration `e8d9a0b1c2d3` creating `orders` and `order_items` with check constraints, foreign keys, and RLS policies.
  - Implemented `order_service.py` with atomic checkout (locks sorted by `product_id` ASC with `FOR UPDATE`, stock decrement, cart clearing, rollback on failure).
  - Implemented `orders.py` router with `POST /api/v1/orders`, `GET /api/v1/orders`, and `GET /api/v1/orders/{id}` with owner isolation (404 on unowned).
  - Full test suite in `tests/test_orders.py` verifying concurrency, locking, rollback, empty cart, stock validation, and ownership.
- [x] Phase 5 (Checkout Frontend):
  - Created `/checkout` page with full shipping address form, client-side validation (including 6-digit pincode), order summary with paise-to-rupee conversion, and atomic order placement via `POST /api/v1/orders` using Supabase JWT.
  - Created `/order-success` and `/order-success/:orderId` page displaying order ID, snapshotted items, total in rupees, status badge, and immediate cart context/badge clearing.
  - Connected "Buy Now" flow on `ProductDetail`: adds product to cart, navigates to `/checkout`, and seamlessly prompts unauthenticated users to login before returning to checkout.
  - Wired "Proceed to Checkout" from Cart page.
  - Robust error and loading states handling backend 400 over-stock errors with alert banners and disabled button state to prevent double submits.

## In progress
- Phase 5: Checkout, orders, payments.

## Next up (in order)
1. Phase 5: Checkout endpoint with Idempotency-Key, atomic stock decrement, Razorpay test mode, orders list/detail.
2. First early deploy.

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
- 2026-10-27: Built the orders backend (spec 0002). Replaced legacy order.py with UUID PK + BigInteger money models (orders, order_items), Alembic migration e8d9a0b1c2d3, atomic checkout with SELECT ... FOR UPDATE locked by product_id ASC, owner-scoped queries with 404 on unowned orders, and comprehensive test suite with 30 passing tests.
- 2026-10-27: Built the checkout and order success frontend. Integrated POST /api/v1/orders with Supabase JWT, client form validation (6-digit pincode), money conversion from paise to INR, over-stock error display, Buy Now redirect flow, and cart clearing.
- 2026-10-27: Fixed orders endpoint 500 / Network error. Root cause: Order model mapped user_id with ForeignKey("auth.users.id"); because auth.users is external to SQLAlchemy's metadata, ORM flush failed with NoReferencedTableError, causing an unhandled 500 without CORS headers (blocked by browser as "Network error"), while migration e8d9a0b1c2d3 had not yet been applied. Fix: applied migration e8d9a0b1c2d3 and removed external ForeignKey from Order.user_id. Added test verifying create_order and serialization.
- 2026-10-27: Fixed homepage bedroom category image. Replaced deleted Unsplash URL (404) with active bedroom image (photo-1616594039964-ae9021a400a0) and added onError fallback to suppress broken image icons in favor of neutral card placeholder backgrounds.
- 2026-10-27: Secured WhatsApp contact. Replaced hardcoded personal phone number in Contact.jsx with REACT_APP_WHATSAPP_NUMBER environment variable, created frontend/.env.example, verified .env is gitignored, updated Contact Us button to "Chat on WhatsApp" with prefilled message opening in a new tab with rel="noopener noreferrer", and audited git history for the committed number.
- 2026-10-28: Replaced WhatsApp contact with full-stack contact form. Added contact_messages table (migration 981b0404bdba), POST /api/v1/contact with Pydantic validation, honeypot bot trap, and in-memory IP rate limiting. Updated Contact.jsx with loading/success/error states, removed all wa.me links, WhatsApp buttons, and company phone numbers in favor of REACT_APP_CONTACT_EMAIL.
