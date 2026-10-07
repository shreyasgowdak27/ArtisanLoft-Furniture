# Code Standards — ArtisanLoft

## General
- Small, focused functions and files. No dead code, no commented-out code, no leftover console.log / print debugging.
- Descriptive names. Comments explain WHY, not what.
- Never hardcode secrets, URLs or keys; read from environment/config.
- Never silently swallow errors. Handle them and return a clear message.
- Every new feature ships with tests for its main path and its key failure cases.
- Do not add a new dependency without saying why. Prefer what is already in the stack.
- Make the smallest change that completes the task. Do not refactor unrelated code.

## Python / FastAPI
- Python 3.11+, full type hints, async endpoints and async DB access.
- Pydantic v2 models for ALL request and response bodies. Never return raw ORM objects.
- Layering: router (thin) -> service (business logic) -> models/db. Routers never contain SQL.
- SQLAlchemy 2.0 style (`select()`, `AsyncSession`). Never string-concatenate SQL; use parameters.
- Schema changes only through Alembic migrations (one migration per change). Never edit the DB by hand.
- Dependencies via FastAPI `Depends` (db session, current user, admin check, redis).
- Config via `pydantic-settings` in `core/config.py`; fail fast at startup if a required variable is missing.
- Money: integer paise. Time: timezone-aware UTC.
- API under `/api/v1`. Correct status codes (201 create, 204 delete, 401 unauthenticated, 403 forbidden, 404, 409 conflict, 422 validation, 429 rate-limited).
- Errors use the shape in architecture.md section 8.
- Pagination on every list endpoint; never return unbounded lists.
- Tests: pytest + httpx AsyncClient, tests live in `app/tests/`. Use a separate test database or transactional rollback; never test against production data.
- Format and lint with `ruff` (format + lint). Code must pass before commit.

## React / Frontend
- Functional components and hooks only. One component per file, PascalCase file names.
- Server state (API data) with TanStack Query; local UI state with useState/useReducer; auth/session in a single AuthContext. No Redux.
- All API calls go through `src/api/` (one typed client with the base URL, auth header and error handling). Components never call fetch directly.
- Every data-driven screen handles three states: loading (skeleton), error (message + retry), empty (helpful message).
- Forms: controlled inputs, client-side validation plus show server errors.
- Never put secrets in frontend code. Only `REACT_APP_*` public values (Supabase anon key is public by design).
- Accessibility: semantic HTML, alt text on images, labels on inputs, keyboard focusable buttons.
- Use Tailwind utility classes; no inline styles; no new CSS files unless needed.
- Format with Prettier, lint with ESLint.

## Git
- Small commits, message format: `type: short description` (feat, fix, refactor, test, docs, chore).
- Never commit `.env`, `node_modules`, `__pycache__`, `*.db`. Keep `.gitignore` correct.
- `.env.example` files stay up to date with every new variable (names only, no values).

## Security checklist (apply to every endpoint)
- Auth required unless explicitly public. Ownership checked (user can only touch their own data).
- Validate and bound all input (lengths, ranges, enums).
- Rate limit write endpoints and auth-related endpoints.
- CORS only for known origins. No wildcard in production.
- Do not leak internal errors or stack traces to the client.
