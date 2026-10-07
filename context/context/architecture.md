# Architecture — ArtisanLoft

## 1. System diagram
```
Browser (React on Vercel)
   |  supabase-js: signup / login (gets JWT)
   |  HTTPS + Authorization: Bearer <JWT>
   v
FastAPI on Render (stateless, N copies, gunicorn + uvicorn workers)
   |-- Supabase Postgres (via pooler)  : products, orders, users' profiles
   |-- Redis (Upstash)                 : carts, rate limits, idempotency keys, hot caches
   |-- Razorpay (test)                 : payments + webhook back to FastAPI
   |-- Supabase Storage                : product images
Browser also sends events to PostHog (analytics).
```

## 2. Folder structure (project root)
> If the React folder is not named `frontend`, change that one name here.
```
/ (project root)
├── AGENTS.md
├── context/                 <- these files
├── frontend/                <- React (CRA) + Tailwind
│   └── src/ (components/, pages/, hooks/, lib/, api/, context/)
├── backend-fastapi/
│   ├── app/
│   │   ├── main.py          <- app factory, middleware, router mounting
│   │   ├── core/            <- config.py (env), security.py (JWT verify), redis.py, logging.py
│   │   ├── db/              <- session.py (async engine), base.py
│   │   ├── models/          <- SQLAlchemy models
│   │   ├── schemas/         <- Pydantic request/response models
│   │   ├── routers/         <- auth/me, products, cart, orders, payments, admin, wishlist
│   │   ├── services/        <- business logic (cart_service, order_service, inventory_service)
│   │   └── tests/
│   ├── alembic/             <- migrations
│   ├── requirements.txt
│   └── .env.example
├── loadtest/                <- k6 scripts + results
└── backend-java-legacy/     <- OLD Spring Boot backend. Ignore. Do not edit.
```
Rule: routers are thin (validate, call a service, return). All business logic lives in `services/`.

## 3. Data model (Postgres, snake_case)
Money is stored as integer paise (`price_cents`, 100 paise = 1 INR). Never use floats for money. Use UUID primary keys for new tables. Keep the Phase 1 product table compatible; inspect existing models first and migrate with Alembic, never by deleting data.

- `profiles`: id (uuid, = Supabase auth user id), full_name, role ('customer' | 'admin', default 'customer'), created_at
- `categories`: id, name, slug (unique)
- `products`: id, name, slug (unique), description, price_cents (int, >0), currency (default 'INR'), category_id, image_url, stock (int, CHECK stock >= 0), is_active (bool), created_at, updated_at
- `addresses`: id, user_id, full_name, phone, line1, line2, city, state, postal_code, is_default
- `wishlist_items`: user_id, product_id (composite unique)
- `orders`: id (uuid), user_id, status ('pending' | 'paid' | 'shipped' | 'delivered' | 'cancelled' | 'failed'), total_cents, currency, shipping_address (jsonb snapshot), payment_provider, payment_ref, idempotency_key (unique), created_at, updated_at
- `order_items`: id, order_id, product_id, name_snapshot, unit_price_cents, quantity
- Indexes: products(slug), products(category_id), products(is_active), orders(user_id, created_at DESC), full-text/trigram index on products(name, description) for search.

Cart is NOT in Postgres. It lives in Redis (see below).

## 4. Redis usage
- Cart: hash `cart:{user_id}` -> field product_id, value quantity. TTL 30 days, refreshed on write. Guest cart kept in browser localStorage and merged on login.
- Rate limiting: key `rl:{route}:{user_or_ip}`, sliding or fixed window.
- Idempotency: key `idem:{user_id}:{key}` storing the created order id, TTL 24h.
- Hot cache: product list/detail cached 60s, invalidated on admin product update.
- If Redis is down: cart endpoints return 503 with a clear error; product browsing must still work (cache is optional, DB is the source of truth).

## 5. Key flows
**Auth**: React calls Supabase Auth (email+password, Google optional) -> gets JWT -> sends it as `Authorization: Bearer` -> FastAPI dependency `get_current_user` verifies signature, expiry and audience -> loads/creates the `profiles` row. FastAPI never stores or checks passwords. Admin routes use `require_admin` (role from `profiles`).

**Cart**: add/update/remove -> Redis only. `GET /cart` reads Redis, then enriches with current price/stock from Postgres (prices always come from DB, never trust client prices).

**Checkout** (`POST /orders/checkout`, requires `Idempotency-Key` header):
1. Check idempotency key in Redis; if seen, return the existing order.
2. Load cart, validate every item exists, is active, quantity > 0.
3. Open ONE DB transaction:
   - for each item run `UPDATE products SET stock = stock - :qty WHERE id = :id AND stock >= :qty` and require rowcount == 1 (else rollback, return 409 "out of stock" naming the item)
   - insert order (status 'pending') and order_items with price snapshots
4. Create Razorpay order for total; return order + payment details to the client.
5. After the client pays, Razorpay calls our webhook `POST /payments/webhook`: verify signature, mark order 'paid' (idempotent), clear cart.
6. Pending orders unpaid after 15 minutes: background job cancels them and restores stock.

**Admin**: product CRUD (invalidates cache), stock edit, order list with status update.

## 6. Invariants (must always be true)
1. Stock never goes below zero; decrement is atomic in the DB, never read-then-write in Python.
2. Prices used for charging come from the DB at checkout time; order_items store a snapshot.
3. Retrying the same checkout request (same Idempotency-Key) never creates a second order.
4. Webhook handling is idempotent and signature-verified.
5. Every non-public endpoint requires a verified JWT; admin endpoints check role server-side.
6. A user can only read/modify their own cart, orders, addresses and wishlist.
7. Secrets only in environment variables.
8. Supabase Row Level Security is ON for all tables (defense in depth), even though FastAPI is the main gatekeeper.

## 7. Scalability design (what to say in interviews)
- API is stateless -> scale horizontally by adding instances behind Render's load balancer.
- DB connections go through Supabase's connection pooler (transaction mode); keep SQLAlchemy pool small per instance; disable prepared statement caching for asyncpg with the pooler.
- Add-to-cart touches only Redis, so cart traffic never hits Postgres.
- The real contention point is stock at checkout -> handled by atomic conditional UPDATE (no oversell) and short transactions.
- Idempotency keys protect against double-clicks and retries.
- Rate limiting protects the API from abuse.
- Images served from CDN-backed storage; frontend served from Vercel CDN.
- Proof, not claims: k6 load test in `loadtest/` with documented numbers.

## 8. API surface (`/api/v1`)
- GET /products (search, category, price range, sort, page, page_size), GET /products/{slug}, GET /categories
- GET /me, PATCH /me
- GET/POST/PATCH/DELETE /cart..., POST /cart/merge
- GET/POST/DELETE /wishlist
- POST /orders/checkout, GET /orders, GET /orders/{id}
- POST /payments/webhook
- Admin: POST/PATCH/DELETE /admin/products, GET /admin/orders, PATCH /admin/orders/{id}/status
- GET /health (liveness) and GET /health/ready (DB + Redis check)
Consistent error format: `{ "error": { "code": "OUT_OF_STOCK", "message": "...", "details": {} } }`

## 9. Environment variables (names only — values go in .env, never committed)
Backend: `DATABASE_URL`, `SUPABASE_URL`, `SUPABASE_JWKS_URL` (or `SUPABASE_JWT_SECRET`), `SUPABASE_SERVICE_ROLE_KEY`, `REDIS_URL`, `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET`, `ALLOWED_ORIGINS`, `ENV`.
Frontend (CRA needs the REACT_APP_ prefix): `REACT_APP_API_URL`, `REACT_APP_SUPABASE_URL`, `REACT_APP_SUPABASE_ANON_KEY`, `REACT_APP_POSTHOG_KEY`, `REACT_APP_POSTHOG_HOST`, `REACT_APP_RAZORPAY_KEY_ID`.
The Supabase service-role key and Razorpay secrets are backend-only. Never expose them in frontend code.
