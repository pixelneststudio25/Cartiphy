# Cartiphy — Build Plan v2 (Level 1 Master Plan)

Supersedes `build-plan.md`. Read together with `design.md` (design system and rules), `admin-panel-plan.md` (admin panel) and `cartiphy-product-and-decisions.md` (overview and decision log).
Purpose: one source of truth so coding agents never guess. If something is not here or in design.md, ask before inventing it.

---

## 1. Product summary

Cartiphy is a Nigeria-first ecommerce store-builder platform. Vendors self-serve a store on their own subdomain from a curated template library; buyers pay through Flutterwave; shipping happens off-platform. It is a standalone brand, separate from the owner's AI website builder.

Roles: **Vendor**, **Customer** (platform-wide account, guest checkout creates a lightweight record), **Platform Admin** (internal).

## 2. Locked decisions (quick reference)

| Area | Decision |
|---|---|
| Name / domain | Cartiphy, cartiphy.com |
| Stack | Next.js + Supabase (Postgres, Auth, Storage) + Vercel (paid plan) + Cloudinary + Flutterwave + Termii (SMS/WhatsApp) + Resend (email) |
| Storefront rendering | Server-rendered templates (vanilla HTML/CSS/JS with placeholders); data injected on the server; `platform.js` handles interactivity only |
| Cart | Server-side, keyed by httpOnly cookie ID |
| Subdomains | `store.cartiphy.com`, lowercase alphanumeric + hyphens, 3–30 chars, reserved-word blocklist |
| Products | Flat: name, price, stock, images, category. No variant system (vendors create separate products) |
| Store type | Single-product or multi-product, chosen by vendor (`stores.store_type`) |
| Money | Stored as integer kobo everywhere. NGN only |
| Payments | Flutterwave sub-accounts (split payments), direct settlement to vendor. **No escrow at launch** |
| Fee bearing | Seller bears Flutterwave's processing fee |
| Delivery | Off-platform. Vendor sets a flat delivery fee, added at checkout (0 = free delivery) |
| Buyer confirmation | Tokenized no-login link, 72h auto-confirm. Unlocks verified reviews and closes dispute window |
| Reviews | Verified purchase only (completed on-platform orders) |
| Discovery | Platform-wide; only Tier 1+ verified vendors surfaced; ranking = verified + rated, blended with newest as cold-start fill; no pay-to-rank |
| WhatsApp | Pre-purchase: "Ask a question" button (add-to-cart stays primary CTA). Full vendor contact revealed after payment |
| Categories (v1) | Fashion & Apparel; Beauty & Personal Care; Food & Groceries; Electronics & Gadgets; Home & Living; Health & Wellness; Kids & Baby; Jewelry & Accessories; Arts, Crafts & Handmade; Other |
| Templates | 3–5 commerce-ready templates at launch, chosen by questionnaire (AI selection deferred until library is ~15–20) |
| Template design rules | Platform surfaces follow design.md. Checkout, order confirmation and trust badge are Cartiphy-owned components inside every template |

## 3. Plans, pricing and entitlements

Plans: **Prime** (free), **Venture**, **Apex**. Annual billing = 10 months' price for 12.

| | Prime | Venture | Apex |
|---|---|---|---|
| Monthly price | Free | ₦12,000 | ₦28,000 |
| Cartiphy commission | 4% | 2% | 1% |
| Products | 25 | 250 | Unlimited |
| Images per product | 3 | 6 | 10 |
| Bulk CSV upload | No | Yes | Yes |
| Templates | 3 core | All | All + early access |
| "Sold on Cartiphy" footer badge | Shown | Removable | Removable |
| Buyer discount codes | No | Unlimited | Unlimited + automatic discounts |
| Abandoned carts | Count only | List with click-to-chat WhatsApp link | + automated recovery (later) |
| Analytics | This month | 90 days: trends, top products, traffic | Full history, repeat customers, CSV export |
| Order export | No | Yes | Yes |
| Vendor SMS/WhatsApp broadcasts | No | Monthly allowance (TBD) | Larger allowance (TBD) |
| Staff accounts | No | No | Yes (later) |
| Custom domain | No | No | Yes (later) |
| Support | Help center + email | WhatsApp, business hours | Priority queue |

Never gated on any plan: verification, reviews, dispute handling, buyer confirmation, buyer-facing SMS/email, low-stock alerts.

**Trial and lapse rules**
- Everyone gets 14 days of Venture features. Trial vendors pay Venture's 2% commission.
- Trial starts only after Tier 1 verification (bank + phone). One trial per unique phone and per unique bank account.
- After the trial (or a lapsed subscription) the vendor falls back to Prime. Data stays intact. Products over the Prime cap stay live, but new ones cannot be added.
- Upgrade prompts must be honest and calm: live commission-savings meter (only when the saving is real), moment-of-need cards (e.g. 20/25 products), dimmed locked features with plan tag, end-of-trial summary of what was used. No manipulative modals.

**Implementation:** all gating reads from a `plans` + `plan_entitlements` config in the database. Features flip by data, never by scattered `if plan ===` checks.

## 4. Architecture

**Rendering flow (storefront)**
1. Request hits `store.cartiphy.com` (wildcard domain).
2. Next.js middleware reads the host header, resolves the store, loads its template and data.
3. Server fills template placeholders (Liquid/Handlebars-style engine) and returns full HTML (SEO and WhatsApp link previews work).
4. `platform.js` (shared SDK) handles cart actions, checkout initiation and event tracking against the API.

**Security baseline (Phase 0, non-negotiable)**
- Row Level Security on every table; role model: vendor, customer, admin.
- Payment and webhook logic runs server-side only. Service-role key never reaches the browser.
- Flutterwave webhook signature verification; idempotent handling via `webhook_events`.
- Rate limiting on auth, checkout and confirmation-link endpoints.
- Automated cross-tenant test: vendor A must fail to read vendor B's orders. Runs at every checkpoint.
- Stock decrement is atomic (single transaction; reject if insufficient stock).

**Payments flow**
Checkout total = items + vendor delivery fee. Flutterwave charge with split to vendor sub-account; Cartiphy commission taken via the split; seller bears processing fee. Order created only on verified webhook.

## 5. Trust, safety and enforcement

**Marketplace Rules (plain-language page + versioned ToS acceptance at signup)**
- Prohibited: phone numbers, bank details or "DM to buy" instructions in listings/store text; directing Cartiphy-originated buyers to pay outside checkout; discouraging checkout use.
- Allowed: answering questions on WhatsApp, delivery coordination after payment, selling to own existing customers by chat.

**Prevent first:** saving a listing with contact/bank details or off-platform phrases shows a friendly block with a fix. Only repeated attempts count.

**Automatic flags (probabilistic, never proof) → nightly job → admin queue:**
content-scanner hits; buyer reports; chat-tap-to-order ratio vs platform median (minimum sample size); sudden order drop with steady traffic/taps.

**Two enforcement tracks**
- *Circumvention (graduated):* educational notice → formal warning email → 30 days removed from discovery → store suspended (no new orders; existing orders fulfilled) → permanent ban + blocklist (bank account, phone).
- *Fraud (fast track):* non-delivery, fake goods, scam reports → suspension pending review.
- Every action notifies the vendor with reason and an appeal window. Owner reviews the queue weekly.
- No money-based penalties (funds settle directly to vendors; ToS must not claim payout holds).

**Vendor verification tiers**
- Tier 0: can build a store; cannot accept live payments.
- Tier 1 (bank account name-match + phone OTP): can sell, appears in discovery, eligible for trial.
- Tier 2 (CAC-verified): "Verified Business" badge.

**Disputes and refunds:** buyer may dispute within 7 days of delivery confirmation; unresolved after 5 days → auto-flag vendor. Refunds are vendor-initiated through Flutterwave; the platform tracks status and enforces via the ladder above. Marketing must not promise "buyer protection" or guaranteed refunds — only verified sellers, payment record, verified reviews and a dispute process.

## 6. Data model (by domain; DDL is a Level 2 task)

- **Identity:** `vendors`, `customers` (is_guest flag), `admin_users`, `tos_acceptances`, `blocklist` (bank account hash, phone)
- **Stores and catalog:** `stores` (store_type, subdomain, category, template_id, status, standing), `store_settings`, `products`, `product_images`, `templates`
- **Commerce:** `carts`, `cart_items`, `orders`, `order_items`, `order_status_history`, `customer_addresses`, `discount_codes`
- **Payments:** `payments`, `webhook_events`, `refunds`
- **Trust:** `reviews`, `disputes`, `reports`, `risk_flags`, `enforcement_actions`, `appeals`
- **Billing:** `plans`, `plan_entitlements`, `subscriptions`, `promo_codes`, `trials`
- **Analytics and ops:** `store_events` (view, cart_add, chat_tap, source: discovery/direct), `notification_log`, `audit_log`

Rules: money = integer kobo; every table has created_at; every vendor-scoped table has store_id and an RLS policy.

## 7. Phases

Each phase ends with a **checkpoint** (a scripted test that must pass) before the next begins.

### Phase 0 — Foundations and security baseline
- Repo, Vercel (paid), separate staging and production Supabase projects, env var map (Flutterwave test/live keys).
- Migrations via Supabase CLI committed to git; Sentry; backups confirmed.
- Core schema with RLS, roles, `plans`/`plan_entitlements` seed, `store_events` table.
- Sandbox accounts: Flutterwave, Termii, Resend, Cloudinary.
- **Checkpoint:** deployed skeleton; cross-tenant RLS test passes; keys verified.

### Phase 1 — Vendor and store core
- Vendor auth (email + phone), ToS/Marketplace Rules acceptance gate.
- Onboarding wizard: store name, category, single/multi-product, style questionnaire → template pick.
- Product CRUD with entitlement caps; images via Cloudinary; content-scanner friendly block on listing text.
- CSV bulk upload (entitlement-gated) with row-level error reporting.
- Subdomain validation, reservation and routing.
- **Checkpoint:** vendor signs up, creates a store, adds products, sees the live subdomain.

### Phase 2 — Storefront rendering and cart
- Template engine (server-side placeholder filling); commerce slots: product grid, product detail, cart, checkout shell, confirmation.
- `platform.js` SDK; server-side cart via cookie.
- 3–5 commerce-ready templates; SEO and OG tags per store.
- Event tracking (view, cart_add, chat_tap, source); "Ask a question" WhatsApp button; footer badge gating.
- Guest capture (email/phone → lightweight customer record).
- **Checkpoint:** buyer browses a real store, carts survive refresh, events recorded, link preview renders.

### Phase 3 — Payments (highest risk; budget the strongest model here)
- Tier 1 verification: bank account name-match and phone OTP; sub-account creation.
- Flutterwave checkout with split (commission), delivery fee, seller-borne fee.
- Verified webhook handler, idempotent via `webhook_events`.
- Atomic stock decrement; order and order_items on confirmed payment.
- Admin-lite (see `admin-panel-plan.md`, stage A0): admin login with 2FA, read-only orders, payments and webhook events viewer, audit log foundation.
- **Checkpoint:** sandbox payment → webhook → order → stock decrement → vendor sees order; duplicate webhook is harmless; concurrent last-item test passes.

### Phase 4 — Order lifecycle and trust mechanics
- Status machine with history: Placed → Confirmed → Out for Delivery → Awaiting Confirmation → Delivered / Disputed / Cancelled.
- Tokenized buyer confirmation link, 72h auto-confirm; vendor contact reveal after payment.
- Disputes (7-day window, 5-day auto-flag), vendor-initiated refunds via Flutterwave.
- Verified-purchase reviews and rating aggregation; vendor standing indicator.
- **Checkpoint:** full happy path, disputed path, auto-confirm path.

### Phase 5 — Notifications
- Termii SMS for all buyer-facing order events (always included); Resend for auth, receipts, formal notices; `notification_log`.
- **Checkpoint:** every Phase 4 transition sends the right message once.

### Phase 6 — Discovery and insights
- Global index, category pages, search (Postgres full-text), store profiles.
- Ranking: verified + rated, blended with newest; Tier 1+ eligibility only.
- Vendor analytics by plan; abandoned-cart visibility by plan (count vs list with click-to-chat).
- **Checkpoint:** discovery homepage shows real, eligible stores with sensible ranking; analytics respect entitlements.

### Phase 7 — Monetization and growth tools
- Flutterwave subscription billing (monthly and annual), Venture trial rules, fallback to Prime, gentle downgrade.
- Subscription promo codes; commission-savings meter and upgrade prompts.
- Buyer discount codes (Venture/Apex); order export.
- **Checkpoint:** trial → paid, trial → Prime fallback, promo applied, commission correct at each plan.

### Phase 8 — Moderation, enforcement and admin
- Full admin panel per `admin-panel-plan.md` (stage A1): vendor and store search, verification queue, disputes, flagged content, risk queue, templated email/SMS with actions, settings editor, audit log UI.
- Report button (listings/stores/orders); nightly risk-scoring job; enforcement ladder with notifications, appeals, blocklist.
- **Checkpoint:** flagged store reviewed, warned, restricted from discovery, suspended; blocked bank/phone cannot re-register.

### Phase 9 — Launch readiness
- Lawyer review of vendor terms and Marketplace Rules; NDPA privacy policy; confirm no raw card data touches Cartiphy.
- Security review pass, backup restore test, monitoring alerts, full end-to-end run, soft launch with a small vendor cohort.

### Phase 10 — Post-launch backlog
Custom domains, staff accounts, automated cart recovery, zone-based delivery fees, AI template selection, dark mode, Services category, multi-store owners, escrow (after Flutterwave/legal clarity), expanded template library.

## 8. Start-now external tasks (long lead time)

- Flutterwave: confirm split payments/subaccounts are approved for your business account as a marketplace; confirm KYB documents.
- Termii: sender ID approval; WhatsApp Business API application.
- Resend: domain DNS (SPF/DKIM).
- Vercel: paid plan; confirm wildcard-subdomain setup.
- Supabase: confirm backup coverage on the chosen plan.
- Legal: Nigerian lawyer for vendor terms, Marketplace Rules, funds-model review; NDPA-aligned privacy policy; accountant on VAT for Cartiphy's own subscription and commission income.

## 9. Working rules for coding agents

1. Read this file and `design.md` before any task; do not invent tokens, plans, states or copy.
2. Build one vertical slice at a time; finish and pass the checkpoint before starting the next phase.
3. Money is integer kobo; never floats.
4. Every table gets RLS at creation; never ship a table without a policy.
5. Payment and webhook code is written and reviewed with the strongest model, with tests.
6. Use the entitlements config for any plan-based behavior.
7. Do not add features not listed here; add them to the backlog instead.

## 10. Model and budget guidance

- Cheaper models: CRUD, forms, templates, dashboard UI (Phases 1, 2, 6).
- Strongest model: payments, webhooks, RLS, order state machine, enforcement logic (Phases 0, 3, 4, 8).
- Track credit spend per phase. If an "easy" phase overruns badly, stop and reassess.
- Recurring costs beyond credits: Vercel paid plan, Supabase paid tier (backups), SMS, domain. Earlier estimates omitted these.

## 11. Still open / to be defined at Level 2

- Design tokens: type scale, spacing, radius, shadow values, status colors, error-red hex (design.md).
- Monthly SMS/WhatsApp allowances for Venture and Apex; exact analytics metric definitions; support-response targets.
- Enforcement thresholds (chat-tap ratio, minimum sample sizes) — tune after real data exists.
- Apex per-transaction commission cap (optional, later).
- Detailed DDL and RLS policies per table; per-phase task breakdown and prompts.
