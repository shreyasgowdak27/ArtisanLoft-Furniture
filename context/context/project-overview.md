# Project Overview — ArtisanLoft

## What we are building
ArtisanLoft is a furniture e-commerce SaaS. Customers browse and search furniture, add to cart, check out with online payment (test mode), and track orders. Admins manage products, stock and orders.

It is not a commercial product, but it must be built, secured, tested and deployed to the standard of a real production system. It is the owner's flagship resume project for top software companies, so engineering quality, scalability design and measurable results matter more than feature count.

## Owner and working style
- Solo student developer, final-year B.E. (Computer Science and Business Systems).
- Building almost entirely with AI coding agents (Cursor). The owner learns concepts through the process.
- The owner must be able to explain every architectural decision in an interview. After each task, summarize what you did and why in plain English (3-5 lines).
- Prefer decisive choices over lists of options. Pick the industry-standard approach and say why.

## Users
1. Customer: sign up / log in, browse, search, filter, wishlist, cart, checkout, order history.
2. Admin: CRUD products, update stock, view and update order status.

## Tech stack (final — do not substitute)
| Layer | Choice |
|---|---|
| Frontend | React (Create React App) + Tailwind CSS + React Router + TanStack Query |
| Backend API | Python FastAPI (async), Pydantic v2 |
| Database | Supabase Postgres (via SQLAlchemy 2.0 async + Alembic migrations) |
| Auth | Supabase Auth (frontend uses supabase-js; FastAPI verifies the JWT) |
| File/image storage | Supabase Storage (product images; currently Unsplash URLs) |
| Cache / cart / rate limit | Redis (Upstash) |
| Payments | Razorpay in TEST mode |
| Analytics | PostHog |
| Frontend hosting | Vercel |
| API hosting | Render (or Railway) |
| CI | GitHub Actions (lint + tests) |
| Load testing | k6 (or Locust) |

## Origin / history
- First generated with Replit, later altered with Gemini. Original backend was Java Spring Boot + MySQL on Render — now LEGACY, replaced by FastAPI.
- Phase 1 of the FastAPI rebuild is DONE: `backend-fastapi` exists, product models/API built from the old seed data, the React frontend is connected to it, and the homepage loads 30 real products with correctly matched Unsplash images.

## Non-goals
- No mobile app, no multi-vendor marketplace, no real money, no microservices, no Kubernetes.
- Do not migrate the frontend away from CRA unless the owner asks.

## Definition of success
- Live public URL (frontend + API) with real signup/login.
- Core flows work reliably: auth, browse/search, cart, checkout + payment (test), orders, admin.
- Load test results documented (requests/sec, p95 latency, error rate) in README.
- README with architecture diagram, setup steps, and design decisions.
- CI passes; no secrets in the repo.
