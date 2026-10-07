# UI Rules — ArtisanLoft

## Layout
- Mobile-first, fully responsive. Every page must work at 360px width.
- Global layout: sticky Navbar (logo, search, wishlist, cart with count badge, account menu), page content, Footer.
- Product grid: 1 column (mobile), 2 (sm), 3 (lg), 4 (xl).

## Required states on EVERY data-driven screen
1. Loading: skeleton placeholders (not a blank page or lone spinner).
2. Error: clear message plus a "Try again" button.
3. Empty: friendly message and a next action (e.g. "Your cart is empty — Browse furniture").

## Interactions
- Buttons that trigger requests show a loading state and are disabled while pending (prevents double submit).
- Add to cart: instant feedback (toast + cart badge update). Optimistic update, roll back on error.
- Out-of-stock products: show an "Out of stock" badge and disable Add to cart. Low stock (<= 5): show "Only N left".
- Destructive actions (remove address, delete product) require confirmation.
- Toasts for success/failure of actions; inline messages for form validation.
- Forms: validate on blur and submit, show errors under the field, keep entered values on error.

## Auth UX
- Protected pages (cart checkout, orders, account, admin) redirect to /login and return the user to where they were after login.
- Show the user's name/avatar in the navbar when logged in; Logout clears session and cached data.
- Admin links are only visible to admins (but the server still enforces access).

## Pages
Home, Products (search + filters + pagination), Product Detail, Cart, Checkout (address -> payment), Order Confirmation, Orders list/detail, Wishlist, Login, Signup, Account, Admin (products, orders), 404.

## Content and accessibility
- Every image has alt text (product name). Use lazy loading for images below the fold.
- Buttons are real `<button>`, links are real `<a>`/`<Link>`. Visible keyboard focus.
- Color contrast must be readable; never rely on color alone for status (add text/icons).
- Money, dates and numbers formatted for en-IN.
