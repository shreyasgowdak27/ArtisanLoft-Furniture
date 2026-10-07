# Build Plan — ArtisanLoft

Work ONE phase at a time. A phase is done only when ALL its "Done when" items are true. Update progress-tracker.md after each phase. Before each phase run /architect; after it run /review.

## Phase 1 — FastAPI backend + product API + frontend connected  [DONE]
Done when: homepage loads real products and correct images from FastAPI. (Completed: 30 products, Unsplash images matched.)

## Phase 2 — Move to Supabase Postgres + migrations
- Create Supabase project, set DATABASE_URL (pooler), add `.env.example`
- Replace SQLite with SQLAlchemy 2.0 async + asyncpg; add Alembic
- Create categories/products tables with indexes; write a seed script that loads the existing 30 products into Postgres
- Add /health and /health/ready
Done when: app runs only on Postgres, no SQLite code or files remain, seed script is repeatable, migrations apply from scratch on a fresh database.

## Phase 3 — Authentication
- Supabase Auth in the frontend: signup, login, logout, session persistence, protected routes
- FastAPI JWT verification dependency, `profiles` table auto-created on first login, GET/PATCH /me
- Admin role support (`require_admin`)
Done when: a new user can sign up, log in, refresh the page and stay logged in; protected endpoints return 401 without a token; user appears in Supabase dashboard; tests cover 401/403/200.

## Phase 4 — Cart (Redis)
- Upstash Redis connection, cart service and endpoints, guest cart in localStorage + merge on login
- Cart page and navbar cart count; price/stock enrichment from DB
Done when: add/update/remove works for guests and logged-in users, cart survives page refresh and login, quantities capped by stock, tests pass.

## Phase 5 — Checkout, orders, payments
- Addresses, checkout endpoint with Idempotency-Key and atomic stock decrement
- Razorpay test-mode order + client payment + verified webhook
- Orders list/detail pages; unpaid-order expiry job that restores stock
Done when: an order can be placed and paid end-to-end in test mode; two simultaneous buyers of the last item result in exactly one success (automated test); repeating the same request creates no duplicate order.

## Phase 6 — Discovery features
- Search (trigram/full-text), category/price filters, sorting, pagination, product detail page, wishlist
Done when: search and filters are fast (indexed), pagination works, wishlist persists per user.

## Phase 7 — Admin panel
- Admin-only pages: product CRUD with image upload to Supabase Storage, stock edit, order management
Done when: non-admins get 403 on every admin endpoint; admin changes appear on the storefront (cache invalidated).

## Phase 8 — Analytics
- PostHog: pageviews, signup, add_to_cart, begin_checkout, purchase events; identify user after login
Done when: a funnel (view -> add to cart -> checkout -> purchase) is visible in PostHog.

## Phase 9 — Hardening and load testing
- Rate limiting, input validation review, security headers, CORS locked to real origins, structured logging, global error handler
- k6 scripts: browse products, add-to-cart, checkout contention on a limited-stock item
- Document results (req/s, p95, error rate) and bottlenecks found/fixed
Done when: load test results are saved in `loadtest/RESULTS.md` and referenced in README.

## Phase 10 — Deploy and polish
- Deploy API to Render (multiple workers), frontend to Vercel, env vars set, CORS correct
- GitHub Actions: lint + tests on every push
- README: overview, architecture diagram, tech choices and reasons, setup, screenshots, load-test numbers, live links
Done when: public URL works end-to-end with real signup, a test order completes on production, CI is green, README is complete.
(Recommended: do a first deploy right after Phase 3 so deployment issues surface early, then redeploy after each phase.)
