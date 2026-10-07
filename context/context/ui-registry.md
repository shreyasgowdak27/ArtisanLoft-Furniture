# UI Registry — reusable components

AGENT: this list is the intended set. Before building anything, scan `frontend/src/components` and `frontend/src/pages`. If a component below already exists, REUSE it and correct its row here. If you create a new reusable component, add a row. Never create a duplicate of an existing component.

| Component | Path | Purpose / usage | Status |
|---|---|---|---|
| Navbar | components/layout/Navbar | Logo, search, cart badge, account menu | verify |
| Footer | components/layout/Footer | Links, brand | verify |
| ProductCard | components/product/ProductCard | Image, name, price, stock badge, add-to-cart, wishlist heart | verify |
| ProductGrid | components/product/ProductGrid | Responsive grid of ProductCard with loading/empty/error states | verify |
| ProductFilters | components/product/ProductFilters | Category, price range, sort | to build |
| SearchBar | components/common/SearchBar | Debounced search input | to build |
| Button | components/ui/Button | Variants: primary, secondary, ghost, danger; `loading` prop | to build |
| Input | components/ui/Input | Label, error text, consistent styling | to build |
| Modal | components/ui/Modal | Accessible dialog | to build |
| Toast | components/ui/Toast | Success/error notifications | to build |
| Skeleton | components/ui/Skeleton | Loading placeholders | to build |
| Badge | components/ui/Badge | Status (in stock, pending, paid...) | to build |
| Pagination | components/ui/Pagination | Page controls | to build |
| QuantityStepper | components/cart/QuantityStepper | +/- quantity with stock cap | to build |
| CartItemRow | components/cart/CartItemRow | One cart line | to build |
| OrderSummary | components/cart/OrderSummary | Subtotal, shipping, total | to build |
| AddressForm | components/checkout/AddressForm | Shipping address | to build |
| ProtectedRoute | components/auth/ProtectedRoute | Redirects unauthenticated users | to build |
| AdminRoute | components/auth/AdminRoute | Admin-only wrapper | to build |
| EmptyState | components/common/EmptyState | Icon, message, action | to build |
| ErrorState | components/common/ErrorState | Message + retry | to build |

Shared helpers: `lib/formatPrice.js`, `lib/supabase.js`, `api/client.js` (single API client), `context/AuthContext.jsx`.
