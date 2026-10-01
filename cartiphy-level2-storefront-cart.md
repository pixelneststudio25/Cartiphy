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

1. Request hits `*.cartiphy.com` (wildcard domain, DNS already covered in Level 1 §12's start-now tasks).
2. Next.js middleware reads the host header. **Reserved subdomains are checked first:** `vendor.` routes to the vendor dashboard/login route group, `admin.` routes to the admin panel, `discover.` routes to the buyer discovery/account route group. Any other subdomain is resolved as a vendor store lookup against the `stores` table.
3. **Store status check happens here (for store subdomains only), before anything else renders:**
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

Nothing else — no custom colors, no custom sections, no theme variables exposed to vendors at launch. Template variety comes from Cartiphy's own template library (15 at launch; see `cartiphy-level2-template-library.md`), not from per-vendor tweaking.

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
- A registered buyer's account, order history, saved addresses (if any), and delivery-confirmation actions live under **`discover.cartiphy.com`** (e.g., `discover.cartiphy.com/account`, `/orders`) — grouped with the buyer-facing discovery surface rather than scattered across individual store subdomains.
- The session cookie is set at the parent domain scope (`.cartiphy.com`, per Level 1's cross-subdomain sessions decision), so a buyer who logs in on `discover.cartiphy.com` is still recognized while browsing an individual store's subdomain — useful for a consistent "logged in" state even though guest checkout never requires it.

---

## 10. In-Store Browsing and Filtering

- **Multi-product stores** with products spanning more than one of the platform's fixed categories (Level 1 §2) get a simple **category filter** within their own storefront.
- **Single-product stores** get no filter and no grid at all — there is exactly one product to show, so the storefront is effectively a single product-detail page functioning as the store's homepage.
- **No full-text search within a single store** at launch — this is deliberately deferred to keep Phase 2 light. It's a safe thing to add later without restructuring the template engine, since it doesn't touch the data model or checkout flow at all.
- **Sort and price filter** for multi-product stores are in scope at launch — see §17.

---

## 11. SEO and Link Previews

- Because rendering is server-side (§3), each product and store page can carry accurate Open Graph tags (title, description, image) reflecting real content — not a generic placeholder.
- A shared-link preview (e.g., a buyer pasting a product link into WhatsApp) should show the product's actual name, price, and image — this only works because the HTML is complete on first response, which is the entire reason server-rendering was chosen over client-side rendering in Level 1.
- **The full spec for OG images (1200×630, 300KB cap, library plus dynamic compositing), favicons, structured data, sitemap and canonicals lives in `cartiphy-level2-vendor-content-tools.md` §1.** This section only establishes that server-rendering makes them possible.

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
- `products.details` — optional jsonb list of label/value pairs, max 8 (§15).
- `policy_versions` (already listed in Level 1 §6/§7) — versions the standard store policy page (§16).
- Template metadata additions (tags, affinity, font pairing) are defined in the Template Library doc §9.

---

## 15. Product Page Requirements and Structured Details

- **Gallery:** every image a product has (1 to 10, per plan) is shown in a swipeable gallery with **tap/click-to-zoom** (pinch-zoom on phones). Multiple angles are encouraged through the product form's copy, not forced.
- **Structured details:** `products.details` is an optional list of label/value pairs (materials, size or dimensions, care instructions, what's included). Maximum 8 pairs; label up to 30 characters, value up to 200; text only. Because products have no variant system, size guidance and similar information lives here. Details pass through the content-scanner like any other vendor text, are editable in the dashboard, and can be drafted with the AI tool (Vendor Content Tools doc §2).
- **Delivery fee shown on the product page** next to the price (see §17).
- Every template must implement all of this; the exact anatomy is in the Template Library doc §4.

---

## 16. Store Policy Page

- **A standard, Cartiphy-owned policy page on every store** (`/policies` on the store's subdomain), identical in content across all stores and linked from the footer, product pages, cart and checkout. Vendors cannot edit or add to it, which keeps the four-field customization rule intact.
- Content covers: how delivery works (vendor-arranged off-platform, flat fee set by the vendor, shown before checkout); returns and refunds (a buyer can raise a dispute within 7 days of confirming delivery, outcome is a full refund or a rejection, no partial refunds); how payment works (through Cartiphy's checkout only); and when the vendor's full contact details are revealed (after payment).
- **It must not promise "buyer protection" or guaranteed refunds** (Level 1 §5) — only describe the real process.
- Wording is versioned in `policy_versions`; the lawyer review happens in the post-launch compliance phase (Level 1 §12).
- Vendor-specific return terms are a possible Phase 10 addition, not launch scope.

---

## 17. Sorting, Price Filter and Cost Transparency

**Multi-product stores**
- **Sort:** Featured (vendor's order, then newest — the default), Newest, Price low-to-high, Price high-to-low.
- **Optional price filter** (min/max), combinable with the category filter in §10.
- The default server-rendered view works without JavaScript; sort and filter controls enhance it.

**Cost transparency**
- The flat delivery fee appears on the product page (e.g. "+ ₦1,500 delivery"), so no cost first appears at checkout.
- The cart shows **subtotal, delivery fee and total** before the buyer proceeds, and the checkout total always equals the cart total.
- Single-product stores skip the cart view via Buy Now, but the first step of checkout shows the same breakdown, and the product page already shows the fee.

---

## 18. Checkpoint Test Script

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
11. A product with 1 image and a product with 10 images both show a working gallery with tap-to-zoom on a phone-width screen.
12. Any two stores show identical content on `/policies`, linked from the footer, product page and checkout; the page contains no "buyer protection" or guaranteed-refund wording.
13. Sort and price filter work on a multi-product store, combine correctly with the category filter, and the default view still renders with JavaScript disabled.
14. The delivery fee is visible on the product page; the cart's total equals the checkout total; a single-product Buy Now shows the full breakdown on the first checkout step.
15. A product's `details` pairs render on the product page; a 9th pair is rejected; a phone number inside a detail value is caught by the content-scanner.

---

*This document supersedes any earlier informal notes on storefront and cart behavior. Read alongside `cartiphy-level-1-master-plan.md`, `cartiphy-level2-payments.md`, and `cartiphy-level2-order-lifecycle.md`. Next Level 2 document: Admin Panel detail. Updated to reference `cartiphy-level2-template-library.md` and `cartiphy-level2-vendor-content-tools.md`.*
