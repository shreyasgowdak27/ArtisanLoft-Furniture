# Library Docs — key patterns for this project

IMPORTANT: Library APIs change. These are the PATTERNS we use. Before writing code with a library, check its current official documentation (or Context7 if available). If the docs differ from this file, follow the docs and update this file.

## SQLAlchemy 2.0 async + asyncpg + Supabase pooler
```python
# db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(
    settings.database_url,           # postgresql+asyncpg://...  (Supabase pooler URL)
    pool_size=5, max_overflow=5, pool_pre_ping=True,
    connect_args={"statement_cache_size": 0},   # needed with pgbouncer transaction mode
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db():
    async with SessionLocal() as session:
        yield session
```
Atomic stock decrement (never read-then-write):
```python
result = await db.execute(
    update(Product)
    .where(Product.id == pid, Product.stock >= qty)
    .values(stock=Product.stock - qty)
)
if result.rowcount != 1:
    raise OutOfStock(pid)   # rollback the whole transaction
```

## Alembic
`alembic revision --autogenerate -m "message"` then READ the generated file before `alembic upgrade head`. Use the same sync/async URL strategy as in `alembic/env.py`.

## Supabase Auth (frontend, supabase-js)
```js
// lib/supabase.js
import { createClient } from '@supabase/supabase-js';
export const supabase = createClient(process.env.REACT_APP_SUPABASE_URL, process.env.REACT_APP_SUPABASE_ANON_KEY);
// signup: supabase.auth.signUp({ email, password })
// login:  supabase.auth.signInWithPassword({ email, password })
// token:  const { data } = await supabase.auth.getSession(); data.session.access_token
// listen: supabase.auth.onAuthStateChange((event, session) => ...)
```
## Supabase JWT verification (backend)
Verify the Bearer token on every protected request: check signature (via the project's JWKS endpoint, or the JWT secret on older projects), expiry, and audience `authenticated`. The user id is the `sub` claim. Use `PyJWT`. Check current Supabase docs for the signing-key method your project uses. Cache the JWKS.
```python
async def get_current_user(creds=Depends(HTTPBearer()), db=Depends(get_db)) -> CurrentUser:
    claims = verify_supabase_jwt(creds.credentials)   # raises 401 on any failure
    return await load_or_create_profile(db, user_id=claims["sub"], email=claims.get("email"))
```

## Redis (Upstash) with redis-py asyncio
```python
import redis.asyncio as redis
r = redis.from_url(settings.redis_url, decode_responses=True)
await r.hincrby(f"cart:{user_id}", str(product_id), qty)
await r.expire(f"cart:{user_id}", 60*60*24*30)
cart = await r.hgetall(f"cart:{user_id}")
```
Rate limit (fixed window): `INCR rl:{route}:{id}` then `EXPIRE` on first hit; return 429 when over the limit. Idempotency: `SET idem:{user}:{key} {order_id} NX EX 86400`.
Upstash uses TLS: URL starts with `rediss://`.

## Razorpay (test mode)
Backend creates an order (`client.order.create({amount, currency: "INR", receipt, ...})`, amount in paise). Frontend opens Razorpay Checkout with the returned order id and the public key id. Backend verifies the payment signature and the webhook signature with the webhook secret (`utility.verify_webhook_signature`). Never trust the frontend's "payment success"; only the verified webhook or signature check marks an order paid.

## TanStack Query (React)
```js
const { data, isLoading, error } = useQuery({ queryKey: ['products', filters], queryFn: () => api.getProducts(filters) });
const mutation = useMutation({ mutationFn: api.addToCart, onSuccess: () => queryClient.invalidateQueries({ queryKey: ['cart'] }) });
```

## PostHog (React)
```js
import posthog from 'posthog-js';
posthog.init(process.env.REACT_APP_POSTHOG_KEY, { api_host: process.env.REACT_APP_POSTHOG_HOST });
posthog.identify(user.id);                 // after login
posthog.capture('add_to_cart', { product_id, price });
```
Event names (snake_case): `product_viewed`, `search_performed`, `add_to_cart`, `begin_checkout`, `purchase`, `signup`. Never send passwords or payment details.

## k6 (load testing)
Scripts in `loadtest/`. Scenarios: (1) browse products at high rate, (2) add-to-cart with auth token, (3) checkout contention: N virtual users buying a product with stock = 5; expect exactly 5 successes and 0 oversells. Record req/s, p95 latency, error rate in `loadtest/RESULTS.md`.

## Deployment notes
- Render: start command `gunicorn app.main:app -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:$PORT`. Health check path `/health`. Free instances sleep when idle: mention cold start in README.
- Vercel: CRA build `npm run build`, output `build`, set `REACT_APP_*` env vars in the dashboard. Add a rewrite so React Router deep links work.
- Supabase: use the pooler connection string (port 6543, transaction mode) for the app and the direct connection only for migrations if needed.
