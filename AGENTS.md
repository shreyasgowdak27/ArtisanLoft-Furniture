# AGENTS.md — ArtisanLoft

Read ALL files in the `/context` directory before starting any task, in this order:
1. context/progress-tracker.md  (where we left off)
2. context/project-overview.md
3. context/architecture.md
4. context/build-plan.md
5. context/code-standards.md
6. context/library-docs.md
7. context/ui-tokens.md, ui-rules.md, ui-registry.md (only for frontend work)

Hard rules (never break):
- Backend is Python FastAPI. The old Java Spring Boot backend is legacy: do not touch or extend it.
- Database is Supabase Postgres. Auth is Supabase Auth. Cart/cache/rate-limit is Redis (Upstash). Do NOT use SQLite or build custom password/login code.
- Never hardcode secrets. Use environment variables. Never commit `.env`.
- Money is stored as integer paise (price_cents), never floats.
- Stock changes must be atomic (see architecture.md invariants).
- Work one phase at a time from build-plan.md. Do not start the next phase unless asked.
- Before a big feature, run /architect. After writing it, run /review.
- At the end of EVERY session, update context/progress-tracker.md.
- The owner is a student who must be able to explain every decision in interviews: after each task, give a 3-5 line plain-English summary of what you did and why.
- If a library API differs from context/library-docs.md, check the library's current official docs and update library-docs.md.
