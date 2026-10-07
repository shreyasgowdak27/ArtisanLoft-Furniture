# UI Tokens — ArtisanLoft

RULE: For screens that already exist, FIRST read `frontend/tailwind.config.js` and the existing components, and KEEP the current look. The values below are the defaults for any NEW UI and for unifying inconsistencies. If the existing config already defines a palette, update this file to match it rather than the other way round.

## Brand feel
Warm, premium, crafted. Natural wood tones, soft neutrals, generous whitespace, large product photography.

## Colors (add to tailwind.config.js `theme.extend.colors`)
| Token | Hex | Use |
|---|---|---|
| `brand-900` | #3B2A1E | Headings, footer background |
| `brand-700` | #6B4A32 | Primary buttons, links |
| `brand-500` | #9C6B45 | Hover states, accents |
| `brand-100` | #F1E6DA | Subtle backgrounds, badges |
| `surface` | #FFFFFF | Cards, modals |
| `canvas` | #FAF7F3 | Page background |
| `ink` | #1F1A17 | Body text |
| `muted` | #6F665E | Secondary text |
| `line` | #E5DDD3 | Borders, dividers |
| `success` | #2F7D4F | In stock, paid |
| `warning` | #B7791F | Low stock, pending |
| `danger` | #B3322F | Errors, out of stock, delete |

## Typography
- Headings: a serif display font (e.g. "Playfair Display"), weights 600-700.
- Body/UI: a clean sans (e.g. "Inter"), weights 400-600.
- Scale (Tailwind): page title `text-3xl md:text-4xl`, section title `text-2xl`, card title `text-lg font-semibold`, body `text-base`, caption `text-sm text-muted`.

## Spacing and shape
- Spacing scale: Tailwind default (multiples of 4px). Section vertical padding `py-12 md:py-16`. Page container `max-w-7xl mx-auto px-4 sm:px-6 lg:px-8`.
- Radius: cards `rounded-xl`, buttons/inputs `rounded-lg`, badges `rounded-full`.
- Shadows: cards `shadow-sm`, hover `shadow-md`, modals `shadow-xl`.
- Breakpoints: Tailwind defaults (sm 640, md 768, lg 1024, xl 1280). Design mobile-first.

## Price display
Format with `Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' })`, converting from paise (divide by 100). One shared helper `formatPrice(paise)` — never format prices inline.
