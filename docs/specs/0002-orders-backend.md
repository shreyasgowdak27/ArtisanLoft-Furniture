# 0002 · Orders backend

**Date**: 2026-10-27
**Status**: Accepted

## Summary

This decision adds a persistent orders system to the FastAPI backend, letting an authenticated user convert their cart into a confirmed order in one atomic database operation. Stock is locked at the Postgres row level (a technique where the database holds an exclusive lock on each product row during the transaction, preventing another request from reading a stale stock count) so two simultaneous checkouts can never both succeed when only one item of stock remains. Prices, names, and quantities are snapshotted (copied at order time) onto each order item so the order record stays correct even if product details change later. No payment gateway is wired yet; every new order lands in `pending` status.

## Context

The cart system (spec 0001) stores quantities only and explicitly defers stock reservation to checkout. The spec note reads: "Checkout must recheck active status, price, and stock and use its own atomic stock decrement." This spec fulfils that obligation.

The existing `app/models/order.py` is a legacy stub from the Java Spring Boot era. It uses `Float` for money (a known source of rounding bugs), `Integer` primary keys (vulnerable to enumeration), and a foreign key to `users.id` (a table that does not exist in the current Supabase Postgres schema; the project uses `auth.users`). It must be replaced entirely; it cannot be extended.

The project enforces integer paise for all money values (`price_cents` on `products`). The same rule applies here: order totals and line amounts must be `BIGINT`, never `Float`.

There is no payment gateway in scope. The `pending` status is the only status created by this feature; a future `paid` transition will be designed separately. The order lifecycle must accommodate that transition without a schema change.

## Requirements

**User stories**:
- As a signed in shopper, I want to place an order from my cart so that I can purchase the items I have selected.
- As a signed in shopper, I want to view my order history so that I can track past purchases.
- As a signed in shopper, I want to view a single order's details so that I can see what I bought and at what price.

**Acceptance criteria**:
- **AC-1**: A signed in user can `POST /api/v1/orders` and, if their cart is non empty and all items are in stock, an order is created with status `pending`, all cart items are removed, product stock is decremented by each item's quantity, and the endpoint returns the new order with its items including snapshotted name, unit price, and line total.
- **AC-2**: If any cart item's requested quantity exceeds the product's current `stock` at checkout time, the entire checkout is rejected with `400`, no order is created, no stock is decremented, and the cart is unchanged.
- **AC-3**: Two simultaneous checkout requests from the same user (or two users both buying the last unit of the same product) cannot both succeed when combined demand exceeds stock. One succeeds; the other receives `400`.
- **AC-4**: The endpoint returns `400` if the cart is empty at checkout time.
- **AC-5**: `GET /api/v1/orders` returns a list of the signed in user's own orders, newest first, with items included.
- **AC-6**: `GET /api/v1/orders/{id}` returns the full order with items for the signed in user. It returns `404` if the order does not exist or belongs to a different user.
- **AC-7**: The API never accepts user id, price, stock, or product name from the client in order data. All those values are sourced from the database.
- **AC-8**: Deleting a Supabase Auth user cascades and removes that user's orders and order items.

## Options considered

### Option 1: Application layer lock (Redis distributed lock)

Acquire a per user Redis lock before checkout, read stock, decrement in Python, write back. No Postgres row level locking.

**Pros**:
- Familiar pattern for teams already using Redis.

**Cons**:
- Redis is declared as cache and rate limit only in this project; making it a correctness dependency for order integrity is a scope expansion and a new failure mode.
- A crashed worker that holds the lock can block checkout for all users until the TTL expires.
- Two users buying the same product still require a per product lock, not a per user lock, so this does not actually solve the stated concurrency problem without a more complex key scheme.

### Option 2: Postgres `SELECT ... FOR UPDATE` row locks, chosen

Within a single `AsyncSession` transaction, select each product row with `FOR UPDATE` (sorted by `product.id` to prevent deadlock), check stock, decrement, insert order and items, delete cart rows, commit. All in one round trip to the database.

**Pros**:
- Correctness is guaranteed by the database, not by application code or a second service.
- No new infrastructure; Postgres already handles this.
- A failed transaction auto rolls back; no compensating code needed.
- Lock scope is narrow: only the specific product rows in this order, for the duration of the transaction (typically under 100ms).

**Cons**:
- Under extreme contention (many users buying the same last unit simultaneously) lock wait time can spike. Acceptable for a furniture store with low concurrent checkout volume.
- The order of `FOR UPDATE` lock acquisition must be deterministic (sorted by `product_id`) or deadlocks are possible between two transactions locking the same rows in opposite order.

### Option 3: Optimistic concurrency with a `stock_version` column

Add a version counter to `products`. Read the version, write a conditional `UPDATE products SET stock = stock - n WHERE id = ? AND stock_version = ?`. If zero rows updated, retry or return `400`.

**Pros**:
- No lock contention under low conflict.

**Cons**:
- Requires a retry loop in application code; complexity grows with cart size (multiple products means multiple version checks).
- A hot product causes high retry rates and more database round trips than a simple row lock.
- Adds a migration to `products` just for this feature.

## Decision

**Chosen option**: Option 2: Postgres `SELECT ... FOR UPDATE` row locks.

Use a single `AsyncSession` transaction that acquires `FOR UPDATE` row locks on affected product rows (in `product_id` sort order), validates stock, decrements stock, inserts order and order items with price snapshots, and deletes the user's cart items. One commit or one rollback; no partial state possible.

**Implementation skills**: `architect` (`ArtisanLoft`, `.agents/skills/architect/`) · `develop` (`ArtisanLoft`, `.agents/skills/develop/`)

## Rationale

The project already runs Postgres via Supabase. Row level locking is a built in Postgres primitive with zero operational overhead: no extra service, no TTL tuning, no lock orphan risk. The cart spec explicitly called out that "checkout must use its own atomic stock decrement," making this the direct fulfilment of that stated invariant. The deadlock risk of `FOR UPDATE` is real but entirely preventable by sorting locks on `product_id` before acquiring them, a one line fix. Optimistic concurrency (Option 3) would work at very low contention but adds retry complexity that is unnecessary for a furniture store. Redis locking (Option 1) would expand Redis's role from cache to correctness primitive, a tradeoff not worth taking when Postgres solves the problem natively.

## Feature design

**Data model sketch**:

| Table | Field | Type | Nullable | Rules |
|---|---|---|---|---|
| `orders` | `id` | UUID | No | Primary key, `gen_random_uuid()` |
| `orders` | `user_id` | UUID | No | FK → `auth.users.id` `ON DELETE CASCADE` |
| `orders` | `status` | VARCHAR(20) | No | Default `pending`; future values: `paid`, `cancelled` |
| `orders` | `total_cents` | BIGINT | No | Sum of all `order_items.line_total_cents`; must be `> 0` |
| `orders` | `shipping_name` | VARCHAR(255) | No | Full name for delivery |
| `orders` | `shipping_line1` | VARCHAR(255) | No | Street address line 1 |
| `orders` | `shipping_line2` | VARCHAR(255) | Yes | Street address line 2 (apartment, floor, etc.) |
| `orders` | `shipping_city` | VARCHAR(100) | No | City |
| `orders` | `shipping_state` | VARCHAR(100) | No | State or province |
| `orders` | `shipping_pincode` | VARCHAR(20) | No | Postal or pin code |
| `orders` | `shipping_country` | VARCHAR(2) | No | ISO 3166-1 alpha-2 code; default `IN` |
| `orders` | `created_at` | TIMESTAMPTZ | No | Server default `now()` |
| `order_items` | `id` | UUID | No | Primary key, `gen_random_uuid()` |
| `order_items` | `order_id` | UUID | No | FK → `orders.id` `ON DELETE CASCADE` |
| `order_items` | `product_id` | UUID | No | FK → `products.id` `ON DELETE RESTRICT` |
| `order_items` | `product_name` | VARCHAR(255) | No | Snapshot of `products.name` at order time |
| `order_items` | `unit_price_cents` | BIGINT | No | Snapshot of `products.price_cents` at order time |
| `order_items` | `quantity` | INTEGER | No | Positive integer; `CHECK (quantity >= 1)` |
| `order_items` | `line_total_cents` | BIGINT | No | `unit_price_cents * quantity`; stored for fast reads |

`orders` has a `user_id` index for the list endpoint. `order_items` is indexed on `order_id` (the primary access pattern). There is no unique constraint on `(order_id, product_id)` because a future split shipment or correction flow might legitimately add the same product twice; that is a deliberate choice.

**State transitions**:

```
created → pending (by POST /api/v1/orders, this spec)
pending → paid    (future: payment confirmation webhook or mock endpoint)
pending → cancelled (future: cancellation feature)
```

No other transitions are valid. `status` is application enforced; there is no DB level constraint on the value set so adding future statuses does not require a migration.

**API surface**:

| Endpoint | Method | Key inputs | Key outputs | Auth | Key errors |
|---|---|---|---|---|---|
| `/api/v1/orders` | POST | `shipping_name`, `shipping_line1`, `shipping_line2` (opt), `shipping_city`, `shipping_state`, `shipping_pincode`, `shipping_country` (opt, default `IN`) all strings from request body | `OrderRead` (order + items) | Bearer JWT | `400` empty cart, `400` insufficient stock, `401`, `422` |
| `/api/v1/orders` | GET | None | `list[OrderRead]` newest first | Bearer JWT | `401` |
| `/api/v1/orders/{id}` | GET | `id: UUID` path param | `OrderRead` with items | Bearer JWT | `401`, `404` |

`OrderRead` response shape (the object returned by all three endpoints):

```json
{
  "id": "uuid",
  "status": "pending",
  "total_cents": 5999800,
  "shipping_name": "Shreyas Gowda",
  "shipping_line1": "12 MG Road",
  "shipping_line2": null,
  "shipping_city": "Bengaluru",
  "shipping_state": "Karnataka",
  "shipping_pincode": "560001",
  "shipping_country": "IN",
  "created_at": "2026-10-27T15:34:00Z",
  "items": [
    {
      "id": "uuid",
      "product_id": "uuid",
      "product_name": "Oak Dining Table",
      "unit_price_cents": 2999900,
      "quantity": 2,
      "line_total_cents": 5999800
    }
  ]
}
```

**Value sourcing**:

| Action | Value produced / displayed | Source |
|---|---|---|
| `POST /api/v1/orders` | `user_id` on order | `CurrentUser.id` from `get_current_user` dependency (JWT subject) |
| `POST /api/v1/orders` | Cart items to process | `cart_items` rows for the authenticated user, joined to `products` |
| `POST /api/v1/orders` | `product_name` on order item | Snapshot from `products.name` at transaction time |
| `POST /api/v1/orders` | `unit_price_cents` on order item | Snapshot from `products.price_cents` at transaction time |
| `POST /api/v1/orders` | `quantity` on order item | `cart_items.quantity` for that user and product |
| `POST /api/v1/orders` | `line_total_cents` on order item | `products.price_cents * cart_items.quantity`, computed in Python before insert |
| `POST /api/v1/orders` | `total_cents` on order | Sum of all `line_total_cents` values for this order, computed in Python before insert |
| `POST /api/v1/orders` | `status` on order | Hardcoded `pending`; not accepted from client |
| `POST /api/v1/orders` | Shipping fields on order | Request body fields validated by Pydantic; not derived |
| `POST /api/v1/orders` | Decremented `products.stock` | `products.stock - cart_items.quantity` per product, applied inside the locked transaction |
| `GET /api/v1/orders` | Orders returned | `orders` rows where `orders.user_id = CurrentUser.id`, ordered by `created_at DESC` |
| `GET /api/v1/orders` | Items on each order | `order_items` rows joined to `orders`, filtered transitively by `user_id` |
| `GET /api/v1/orders/{id}` | Order returned | `orders` row where `id = path param` AND `user_id = CurrentUser.id`; 404 if neither matches |
| `GET /api/v1/orders/{id}` | Items on order | `order_items` rows where `order_id = path param` |

**Key invariants**:

1. An order row and its items are inserted in the same transaction as the stock decrement and cart clear. No partial order is possible.
2. `total_cents` equals the sum of `order_items.line_total_cents` for that order. Both are computed in the service layer before any insert; neither is accepted from the client.
3. `order_items.line_total_cents` equals `unit_price_cents * quantity`. Stored for read performance; recomputable from the other two columns.
4. `products.stock` is never decremented below zero. The `FOR UPDATE` lock check reads the committed stock and rejects the checkout if `stock < requested_quantity` before any write.
5. Product rows are locked in ascending `product_id` order in every transaction. This is mandatory to prevent deadlocks when two transactions lock the same rows.
6. Cart items are deleted within the same transaction as the order insert. The cart is never cleared before the order is confirmed.
7. An order belongs to exactly one `auth.users` user. Users cannot read, list, or modify another user's orders.
8. Money values are BIGINT integer paise, never Float.

**Security model**:

Every endpoint requires the `get_current_user` Bearer JWT dependency. The router never accepts a `user_id` in path, query, or body: it is always derived from `CurrentUser.id`. The list and detail endpoints filter by `user_id` in the SQL predicate, so a user cannot enumerate or read another user's orders. A missing or wrong order id returns `404` (not `403`) to avoid leaking the existence of orders belonging to other users. Row level security policies on `orders` mirror the application layer rule as defense in depth (same pattern as `cart_items`). This feature handles order identifiers and shipping addresses (PII); no payment card data is stored.

**Configuration required**:

No new environment variables. This feature uses the existing `DATABASE_URL` (async SQLAlchemy session) and `SUPABASE_URL` / `SUPABASE_JWT_SECRET` (auth dependency). No new third party credentials.

**Critical test scenarios**:

- Happy path: a signed in user with a non empty cart posts valid shipping data; the response contains the new order with snapshotted items and `status: pending`; product stock is reduced by the ordered quantities; the user's cart is empty afterward; verifies **AC-1**, **AC-5**, **AC-7**.
- Insufficient stock: one item in the cart has `quantity` exceeding current `products.stock`; the endpoint returns `400`; no order row is created; stock is unchanged; the cart still contains all items; verifies **AC-2**.
- Concurrent checkout: two simultaneous requests both attempt to buy the last unit of the same product; exactly one succeeds with `200` and the other receives `400`; final `products.stock` for that product is zero, not negative; verifies **AC-3**.
- Empty cart: user posts to `/api/v1/orders` with an empty cart; endpoint returns `400`; verifies **AC-4**.
- Order list: user with two orders calls `GET /api/v1/orders`; both are returned newest first with items; verifies **AC-5**.
- Order detail, own order: `GET /api/v1/orders/{id}` with a valid order id owned by the caller returns the full order; verifies **AC-6**.
- Order detail, another user's order: `GET /api/v1/orders/{id}` with an order id owned by a different user returns `404`; verifies **AC-6**, **AC-7**.
- Auth: unauthenticated request to any endpoint returns `401`; verifies **AC-7**.
- Cascade delete: deleting the Supabase Auth user removes their `orders` and `order_items` rows; verifies **AC-8**.

## Build plan

Build approach: no approach is recorded in `AGENTS.md` or `docs/scope/`. Defaulting to end to end tracer bullet slices (thin vertical thread through every layer, thickened per slice) as the production default.

1. Replace `app/models/order.py` with the correct `Order` and `OrderItem` SQLAlchemy models (UUID PKs, BIGINT money, correct FK targets including `auth.users.id`), add `app/schemas/order.py` with `OrderItemRead`, `OrderRead`, and `OrderCreate` Pydantic schemas, and write one Alembic migration that creates `orders` and `order_items` with all constraints, indexes, FK rules, and row level security policies. Satisfies **AC-1**, **AC-7**, **AC-8**.
2. Write `app/services/order_service.py` with the checkout function: open a transaction, load cart items (raising `400` if empty), lock product rows with `SELECT ... FOR UPDATE` in `product_id` sort order, validate stock for every item (raising `400` on first failure, rolling back), compute snapshots and totals, insert `orders` and `order_items` rows, decrement `products.stock`, delete the user's `cart_items`, commit, and return the assembled `OrderRead`. Satisfies **AC-1**, **AC-2**, **AC-3**, **AC-4**, **AC-7**.
3. Write `app/services/order_service.py` list and detail query functions: `list_orders` selects orders by `user_id` newest first with joined items; `get_order` selects by `id` and `user_id`, returning `None` (mapped to `404` in the router) when not found. Satisfies **AC-5**, **AC-6**, **AC-7**.
4. Write `app/routers/orders.py` with the three routes (`POST`, `GET /`, `GET /{id}`), wire `get_current_user` and `get_db` dependencies, add `include_router` for the orders router in `app/main.py`. Satisfies **AC-1**, **AC-5**, **AC-6**.
5. Write tests in `backend-fastapi/tests/test_orders.py` covering the critical test scenarios above. Satisfies **AC-1** through **AC-8**.

## Consequences

**Positive**:
- Order integrity is database guaranteed: a Postgres transaction is the only tool that can make "insert order, decrement stock, clear cart" atomic. No application memory state, no compensating transaction, no saga pattern needed at this scale.
- Price snapshots on `order_items` make the order record self contained: a product can be renamed, repriced, or deactivated without corrupting historical orders.
- The `pending` only status model is intentional minimal design; adding `paid` or `cancelled` later requires no schema migration, only a new service function and a status string check.

**Negative or tradeoffs**:
- `FOR UPDATE` row locks will cause lock contention if many users simultaneously check out items that share a popular product. For a furniture store this is an acceptable tradeoff; for a flash sale at scale it would need a queue based approach.
- Locking in `product_id` sort order is a required invariant. Any future checkout code path that locks products in a different order risks deadlock. This must be documented and enforced in code review.
- `line_total_cents` is a stored derived value (see the "never compute and store derived values" rule in the feature mode guide). It is an intentional exception: order items are immutable after creation and the value is read frequently. The tradeoff is explicit.

**Neutral**:
- The legacy `order.py` model file is replaced wholesale. If any other code imports from it (the current codebase does not; confirmed by inspection), those imports will break and must be updated.
- No new environment variables or external credentials are required.

## Follow-up

- [ ] Design the `paid` transition: a mock "mark as paid" endpoint (`PATCH /api/v1/orders/{id}/pay`) so the frontend can simulate a completed purchase. This should be a separate spec before implementation.
- [ ] Update `context/architecture.md` and `AGENTS.md` after this feature ships to reflect that orders use Postgres (not a legacy Java service) and that the old `order.py` stub is gone.
- [ ] Consider adding `GET /api/v1/orders` pagination (cursor or offset) before the list endpoint is exposed to end users. A single user will not accumulate thousands of orders quickly, but the pattern should be consistent with the products endpoint.
- [ ] The shipping address is stored flat on `orders`. If a future version wants reusable saved addresses, extract them to a `shipping_addresses` table. For now, flat is correct and simpler.

