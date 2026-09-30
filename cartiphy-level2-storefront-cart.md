# Cartiphy — Level 2: Storefront and Cart

Reads against `cartiphy-level-1-master-plan.md`, `cartiphy-level2-payments.md`, and `cartiphy-level2-order-lifecycle.md`. This is the contract for Phase 2's template engine, cart, checkout shell, and buyer-facing storefront behavior.

---

## 1. Vendor Customization Model — No Custom Code

**This is the core boundary of the entire system:** vendors never touch code. Cartiphy's mandate is to provide world-class, elegant store templates — not a page builder or a code editor. A vendor's control is limited to:

- **Text fields** — About/description text, product names and descriptions, store name.
- **Image fields** — logo, store banner/cover, product images (via Cloudinary).
- **Link fields** — WhatsApp contact number (rendered as a `wa.me` link, see §5).
- **Product catalog** — full CRUD on products via the dashboard, not the storefront itself.

No custom HTML/CSS/JS, no custom colors, no custom layout sections at launch. This is what makes it possible to guarantee that the Cartiphy-owned components — checkout, order confirmation, the trust badge (per `design.md`'s anti-generic rule #12) — are genuinely tamper-proof and visually consistent across every store, regardless of which template a vendor picked.

**Consequence for the template engine:** every template is built entirely by Cartiphy (or from ChatGPT-generated visual references, built into code by Codex — no Sleek, per Level 1 §2). Vendors select and fill; they never construct.

---

## 2. Template Switching

- A vendor can switch templates through their dashboard at any time.
- **First 30 days of a store's life:** unlimited switches — no cadence limit. This is the settling-in period; there's no buyer history yet to disrupt, and vendors should be free to find the right fit.
- **After 30 days:** capped to **one switch per 30-day period.** This prevents template-thrashing, which would confuse returning buyers and repeatedly trigger content-remapping for no real benefit.
- **On every switch:** the system checks whether all of the vendor's current content (images, text fields) maps cleanly onto the new template's slots. If something won't carry over (e.g., a banner image the new template doesn't have a slot for), the vendor sees a clear warning before confirming — never a silent content loss.
- Switching is available regardless of store type (single-product or multi-product), as long as the new template supports that store type.

---

## 3. Rendering Flow (recap and detail)

Per Level 1 §4, storefronts are server-rendered for SEO and link-preview quality. In detail:

1. Request hits `store.cartiphy.com` (wildcard domain, DNS already covered in Level 1 §12's start-now tasks).
2. Next.js middleware reads the host header and resolves it to a `stores` row.
3. **Store status check happens here, before anything else renders:**
   - `draft` or `pending_verification` → render a public "coming soon" page (§4). Never a 404, never the real storefront.
   - `suspended` or `banned` → render a neutral "this store is currently unavailable" page — no reference to enforcement details (that stays internal).
   - `closed` → same neutral unavailable treatment.
   - `live` → proceed to normal rendering.
4. For a `live` store, the server loads the assigned template and the store's content/products, fills the template's placeholders server-side, and returns full HTML — this is what makes SEO tags and WhatsApp link previews actually work, since the content is present in the initial response, not injected client-side after load.
5. `platform.js` (the shared client SDK) takes over from there for interactivity: cart actions, checkout initiation, "Ask a question" links, and event tracking (§8).

---

## 4. Pre-Live Store Visibility

- A store in `draft` or `pending_verification` already has a subdomain reserved and reachable, per Level 1's subdomain rules — but visiting it shows a simple, on-brand **public "coming soon" page**, not the vendor's actual (incomplete or unpaid-enabled) storefront and not a broken 404.
- This keeps the subdomain feeling intentional the moment a vendor reserves it, which matters if they share the link early (e.g., on social media) before finishing setup.
- The "coming soon" page is a Cartiphy-owned template component, not something the vendor customizes.

---

## 5. Store-Level Customization Fields (Launch Scope)

Exactly four things beyond the product catalog, per your decision:
1. **Logo**
2. **Images** — store banner/cover image (and whatever image slots a given template defines beyond that)
3. **About / description text**
4. **WhatsApp contact number**

**WhatsApp number is a required onboarding field**, not optional — it's core to how buyers reach vendors pre-purchase (§6) and is used again post-payment for delivery coordination (per Level 1 §2).

Nothing else — no custom colors, no custom sections, no theme variables exposed to vendors at launch. Template variety comes from Cartiphy's own template library (3–5 at launch), not from per-vendor tweaking.

---

## 6. "Ask a Question" (Pre-Purchase WhatsApp)

- Renders as a `wa.me` link, pre-filled with a message that references the specific product the buyer was viewing (e.g., "Hi, I have a question about [Product Name]").
- This is a secondary CTA — **Add to Cart / Buy Now remains the primary CTA** on every product view, per Level 1's anti-circumvention stance (WhatsApp is for questions, not a checkout bypass).
- Full vendor contact (beyond this pre-filled question link) is only revealed to the buyer after payment, per Level 1 §2.

---

## 7. Cart Behavior

**Server-side, cookie-keyed** (per Level 1 §2 and §4), applying to both store types:

- **Multi-product stores:** standard add-to-cart, cart drawer/view, quantity adjustment, proceed to checkout.
- **Single-product stores:** "Buy Now" silently creates a one-item cart and skips straight to checkout — same underlying cart infrastructure, no separate checkout system (per Level 1's locked decision).

**Stock handling — no reservation system at launch.** Adding an item to a cart does not reserve stock. Stock is only authoritatively checked immediately before the charge is initiated (per the Payments doc §2, step 4) and again inside the atomic order-creation transaction (Payments doc §4). Consequence: rarely, a buyer can add something to their cart and find it's gone by the time they pay. This is accepted as a launch-scope tradeoff — a reservation-with-expiry system is real engineering scope for a problem that will be uncommon at initial volume. When it happens, checkout shows a clear "no longer available" message and removes the item from the cart, rather than failing silently.

**Cart lifecycle:**
- A cart with no checkout activity for **24 hours** counts as "abandoned" — this is the definition that feeds Phase 6's abandoned-cart analytics (count for Prime, list with click-to-chat WhatsApp link for Venture/Apex, per Level 1 §3).
- Cart records are purged after **30 days** of total inactivity — no indefinite storage of stale carts.

---

## 8. Guest Checkout — Required Fields

- **Phone number** — required. This is the primary contact channel, consistent with SMS being the main notification method throughout the product.
- **Email** — optional.
- **Delivery address** — a single freeform text field. No structured state/LGA/zone fields, since delivery fees are a flat per-store rate with no zone-based pricing logic at launch (zone-based delivery fees are explicitly in the Phase 10 backlog).

This creates the lightweight guest `customers` record referenced in Level 1 §1 and §2.

---

## 9. Registered Buyer Authentication

- **Phone-OTP login, passwordless.** No password field, no password reset flow.
- Consistent with how vendor authentication already treats phone verification as a first-class credential, and better suited to a mobile-first Nigerian buyer base than password-based accounts.
- A registered buyer's order history, saved addresses (if any), and delivery-confirmation actions all live behind this same phone-OTP session.

---

## 10. In-Store Browsing and Filtering

- **Multi-product stores** with products spanning more than one of the platform's fixed categories (Level 1 §2) get a simple **category filter** within their own storefront.
- **Single-product stores** get no filter and no grid at all — there is exactly one product to show, so the storefront is effectively a single product-detail page functioning as the store's homepage.
- **No full-text search within a single store** at launch — this is deliberately deferred to keep Phase 2 light. It's a safe thing to add later without restructuring the template engine, since it doesn't touch the data model or checkout flow at all.

---

## 11. SEO and Link Previews

- Because rendering is server-side (§3), each product and store page can carry accurate Open Graph tags (title, description, image) reflecting real content — not a generic placeholder.
- A shared-link preview (e.g., a buyer pasting a product link into WhatsApp) should show the product's actual name, price, and image — this only works because the HTML is complete on first response, which is the entire reason server-rendering was chosen over client-side rendering in Level 1.

---

## 12. `platform.js` SDK — Scope

The shared client-side SDK handles, and only handles:
- Cart actions (add, update quantity, remove) against the server-side cart.
- Checkout initiation (handing off to the inline Flutterwave flow per the Payments doc).
- Event tracking (§13).
- Rendering the "Ask a question" link with the correct pre-filled message.

It does **not** handle template layout or content — that's fully resolved server-side before the page reaches the browser.

---

## 13. Event Tracking

Per Level 1 §9 (Phase 2 scope), the following events are recorded per store, each tagged with a source (`discovery` or `direct`):
- `view` — a product or store page load.
- `cart_add` — an item added to cart (including the implicit add behind a single-product store's "Buy Now").
- `chat_tap` — the "Ask a question" link was clicked.

These feed: Phase 6's discovery ranking inputs, the chat-tap-to-order enforcement signal (Level 1 §5), and vendor analytics (Level 1 §3, gated by plan).

---

## 14. Data Model Additions / Clarifications

Beyond what's already listed in Level 1 §6 and the Order Lifecycle doc:
- `stores.customization` — a small structured object (or dedicated columns) for logo URL, banner image URL, About text, WhatsApp number — not freeform HTML.
- `stores.template_switched_at`, `stores.template_switch_count_30d` — needed to enforce the §2 cadence limit.
- `carts.last_activity_at` — needed to compute the 24-hour abandoned threshold and the 30-day purge.
- `customer_addresses` (already listed in Level 1 §6) — for guest checkout, this can simply be a freeform text field tied to the order rather than a separate structured table entry, per §8. A structured `customer_addresses` table is more relevant for registered buyers who might reuse an address across orders.

---

## 15. Checkpoint Test Script

1. A vendor with no coding ability can fully set up a store (logo, banner, About text, WhatsApp number, products) using only dashboard fields — no code entry point exists anywhere in the flow.
2. Visiting a `draft` store's subdomain shows the "coming soon" page; visiting a `suspended` store shows the neutral unavailable page; neither shows a 404 or leaks internal status.
3. A multi-product store's category filter works correctly when products span two or more categories; a single-product store shows no filter UI at all.
4. Add-to-cart on a multi-product store behaves normally; "Buy Now" on a single-product store skips straight to checkout with a 1-item cart created behind the scenes.
5. An item is removed from stock (simulated) after being added to a cart but before checkout — checkout correctly shows "no longer available" and removes it, rather than failing silently or overselling.
6. A cart untouched for 24+ hours is correctly counted in abandoned-cart analytics; a cart untouched for 30+ days is purged.
7. A vendor switches templates twice within their first 30 days (allowed) and then a third time in the same 30-day window after that period ends (blocked, with a clear message on when they can switch again).
8. A product page's shared link renders an accurate preview (name, price, image) when pasted into WhatsApp — confirming server-side rendering is genuinely working, not just configured.
9. Guest checkout completes with phone + freeform address only, no email; a registered buyer logs in via phone OTP and completes checkout with saved details.
10. An "Ask a question" tap opens WhatsApp with the correct product pre-filled, and is recorded as a `chat_tap` event with the correct source (`discovery` vs `direct`).

---

*This document supersedes any earlier informal notes on storefront and cart behavior. Read alongside `cartiphy-level-1-master-plan.md`, `cartiphy-level2-payments.md`, and `cartiphy-level2-order-lifecycle.md`. Next Level 2 document: Admin Panel detail.*
