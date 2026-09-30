# Cartiphy — Design System & Rules

Reference document for coding agents and designers. All screens must comply with these rules — this exists specifically to prevent generic/templated output.

---

## Brand Identity

- **Name:** Cartiphy (domain: cartiphy.com)
- **Logo:** Two-tone concentric broken-ring icon (outer near-black ring, inner terracotta ring) + "Cartiphy" wordmark
- **Voice:** Warm & Encouraging — never corporate, never cold/transactional
- **Positioning:** Premium, trustworthy, distinctly Nigerian — not generic global SaaS

## Color Palette (locked)

| Role | Color | Hex |
|---|---|---|
| Primary background | Cream | `#F7F4EF` |
| Text / dark surfaces / dark mode background | Near-black | `#1C1B1A` |
| Accent (sparing use only) | Terracotta | `#C1592B` |

**Status colors (supporting, to stay within palette family — no unrelated blues):**
- Success / Delivered: muted green, e.g. `#3F6B4A` on `#E1EBE3` tint
- Pending / Awaiting: muted gold, e.g. `#B08427` on `#F3EBD9` tint
- Dispatch / In-progress: terracotta tint, `#C1592B` on `#F3E3DA` tint
- Disputed/Error: a muted rust-red distinct from the accent terracotta — define exact hex before use, must not clash with brand accent

**Usage rule:** Terracotta accent is reserved for primary CTAs, badges, and key highlight moments only — never large background fills, never more than one dominant accent use per screen.

## Typography

- **Headings/display:** Clash Display (free, Fontshare)
- **Body/UI:** Inter (free)
- Fixed type scale — no page invents its own sizes/weights outside the defined scale
- No accenting a single word/phrase in a headline (bold/italic/color) — a generic AI-design tell
- No unnecessary ALL-CAPS labels or tracked-out eyebrow text above headings

## Motion

- Snappy and quick only: 100–200ms for transitions
- No slow fades, no long crossfades, no bouncy/decorative animation
- Motion responds to user action (opening, confirming, expanding) — not scattered ambient animation
- Signature loader: the ring-mark icon rotating, using its natural gap as the spinner break, with a subtle pulse on the terracotta segment

---

## Anti-Generic Rules (mandatory, all approved)

1. No default component-library look. Every component (even if built on shadcn/Material/Bootstrap) must be re-skinned with Cartiphy tokens before shipping.
2. No generic SaaS blue/purple gradients, anywhere, ever.
3. Terracotta accent reserved for primary CTAs, badges, small highlights — never large fills.
4. No emoji as UI elements/icons/decoration. Warmth comes from copy, not emoji. (Casual SMS/WhatsApp copy is the one exception where emoji may appear.)
5. No generic "four metric cards in a row" dashboard opener. Lead with what the user needs to act on.
6. No heavy drop-shadow-on-everything. Define exact shadow tokens, use sparingly and consistently.
7. Empty states must use Cartiphy's warm voice — never flat "No data available."
8. No literal shopping cart/shopping bag icons anywhere in the brand system. Use the ring/cycle motif instead.
9. All motion is snappy (100–200ms) — no decorative slow motion.
10. Every screen must pass the "could this be any SaaS product" test — if a screen is swappable with another brand's, it fails and needs a Cartiphy-specific detail restored.
11. Typography hierarchy is fixed (Clash Display / Inter) — no page deviates or invents its own.
12. Trust/verification badge visuals (shape, color, placement) must be identical everywhere they appear (discovery, dashboard, store profile).

**Additional general design-quality bar (from studio-level design principles):**
- Ground every screen in Cartiphy's real subject matter and content — no filler/lorem-ipsum-feeling copy
- Spend visual boldness in one place per screen; keep everything else disciplined and quiet
- Responsive down to mobile, visible keyboard focus, reduced-motion respected, accessible contrast
- Numbered markers (01/02/03) only where content is a genuine sequence
- No middle-dot-joined meta strings, no spaced-em-dash labels, no monospace data labels, no arrow (→) appended to every button/link by default — these are template-chrome tells

---

## Full Page/Surface Inventory

**A. Marketing Website:** Homepage, Pricing, About/Trust, How it works, Vendor signup landing, Login, 404/error

**B. Discovery (platform-wide, buyer-facing):** Discovery homepage, Category pages, Search results, Store profile page

**C. Storefront (per-vendor template, not fixed pages):** Product grid, Product detail, Cart, Checkout, Order confirmation, Store About — each template implements these per its own style, bound to the shared commerce-schema slots (see Technical Plan)

**D. Vendor Dashboard:** Onboarding wizard, Dashboard home, Products list, Add/edit product, Orders list, Order detail, Payouts/earnings, Store settings, Verification/KYC flow, Subscription/billing, Reviews management

**E. Buyer Account:** Order history (cross-store), Tokenized delivery-confirmation page (no login), Profile/settings

**F. Admin (internal):** Admin dashboard, Vendor verification queue, Dispute resolution queue, Flagged content queue, Platform analytics

**G. System-wide / cross-cutting:** Signature loader, Empty states, Error states, Toast/notification system, Trust badge system, Email templates (Resend), SMS/WhatsApp templates (Termii)

## Design Tool Allocation

- **Sleek.design** (mobile-screen generation): Vendor Dashboard (D), Buyer Account (E)
- **Figma / hand-designed:** Marketing website (A), Discovery (B), Admin (F), tokenized confirmation page
- **Specified directly in this document, coded straight from spec:** System-wide elements (G)
- **Not fixed pages — governed by shared schema rules, styled per template:** Storefront (C)

## Subdomain Rules

- Lowercase only
- Alphanumeric characters and hyphens only
- Length: 3–30 characters
- Reserved-word blocklist (e.g. admin, api, www, app, dashboard, support)

---

*This document should be read alongside the Cartiphy Build Plan (build-plan.md) for technical/phase sequencing.*
