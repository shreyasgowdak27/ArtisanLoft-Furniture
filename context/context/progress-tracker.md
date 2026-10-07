# Progress Tracker — ArtisanLoft

HOW TO USE: Read this FIRST every session. UPDATE it at the END of every session (what was done, what is next, any problems). Keep it short and accurate. Stale context is worse than none.

## Current status
- Current phase: **Phase 2 — Move to Supabase Postgres + migrations** (not started)
- Last updated: 2026-10-07
- Auth: not started. Cart: not started. Checkout/payments: not started. Admin: not started.

## Completed
- [x] Phase 1: FastAPI backend (`backend-fastapi`) scaffolded; real product models/API built from old `data.sql`; React frontend connected to local FastAPI; homepage loads 30 real products end-to-end.
- [x] Fixed product image mismatch: Unsplash API used to fetch unique, correctly matched images for all 30 products; database re-seeded and verified.

## In progress
- Nothing yet.

## Next up (in order)
1. Phase 2: Supabase project -> Postgres via pooler -> SQLAlchemy async + Alembic -> seed script -> remove SQLite.
2. Phase 3: Authentication.
3. First early deploy right after Phase 3.

## Decisions made
- Backend rewritten in Python FastAPI; Java Spring Boot backend is legacy.
- Stack locked: Supabase (Postgres + Auth + Storage), Redis (Upstash), Razorpay test mode, PostHog, Vercel + Render, k6 for load tests.
- No custom password handling; Supabase Auth only.

## Known issues / blockers
- None recorded.

## Session log
- 2026-10-07: Defined final stack, wrote context files, installed JSM skills in Cursor.
