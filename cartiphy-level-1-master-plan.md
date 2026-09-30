# Cartiphy — Level 1 Master Plan

**Status:** Locked. This is the single source of truth for goals, stack, rules, and phase sequencing.
Level 2 documents go deeper into one system each (Payments, Order Lifecycle, Storefront, Admin, etc.).
Level 3 documents go deeper into one segment of a Level 2 system (task cards, prompts, fine detail).

**Rule for every coding agent (Codex included):** if something is not decided here, in `design.md`, or in the relevant Level 2/3 doc — ask before inventing it. Do not guess tokens, plans, states, copy, or schema.

---

## 1. Product Summary

Cartiphy is a Nigeria-first ecommerce store-builder platform. Vendors self-serve a store on their own subdomain from a curated template library; buyers pay through Flutterwave; shipping happens off-platform. Cartiphy is a standalone brand, separate from the owner's other projects.

**Roles:** Vendor · Customer (platform-wide account; guest checkout creates a lightweight record) · Platform Admin (internal).

---

## 2. Locked Decisions — Quick Reference

| Area | Decision |
|---|---|
| Name / domain | Cartiphy, cartiphy.com |
| Stack | Next.js + Supabase (Postgres, Auth, Storage) + Vercel (paid plan) + Cloudinary + Flutterwave + Termii (SMS/WhatsApp) + Resend (email) |
| Coding agent | Codex via ChatGPT Plus subscription (usage-limited, not metered API credits — see §10) |
| Storefront rendering | Server-rendered templates (vanilla HTML/CSS/JS with placeholders); data injected server-side; `platform.js` handles interactivity only |
| Cart | Server-side, keyed by httpOnly cookie ID |
| Single-product stores | Share the same cart infrastructure as multi-product stores. "Buy Now" silently creates a 1-item cart and skips straight to checkout — no separate checkout system |
| Subdomains | `store.cartiphy.com`, lowercase alphanumeric + hyphens, 3–30 chars, reserved-word blocklist |
| Products | Flat: name, price, stock, images, category. No variant system (vendors create separate products per variant) |
| Store type | Single-product or multi-product, vendor-chosen (`stores.store_type`) |
| Store status enum | `draft → pending_verification → live → suspended → banned → closed` |
| Money | Integer kobo everywhere. NGN only |
| Payments | Flutterwave sub-accounts (split payments) — see §4 for full settlement model. **No escrow at launch** |
| Vendor payouts / withdrawals | **No in-app withdrawal action and no Cartiphy-held balance.** Flutterwave settles the vendor's split directly to their own bank account on Flutterwave's own schedule. Vendor dashboard shows earnings **history**, not a wallet |
| Fee bearing | Seller bears Flutterwave's processing fee |
| Commission cap | None at launch. Flat percentage only (see §3). Revisit once transaction data exists |
| Bank account changes | Allowed, but treated as a security event: re-verification (name-match check) required, change logged (old/new masked), vendor notified by email + SMS immediately, and a 24–48h payout hold applied to the new account before it receives funds |
| Delivery | Off-platform. Vendor sets a flat delivery fee per store, added at checkout (₦0 = free) |
| Refunds | Full refunds only at launch. No partial refunds |
| Buyer confirmation | Tokenized no-login link, 72h auto-confirm. Unlocks verified reviews and closes the dispute window |
| Reviews | Verified purchase only (completed on-platform orders) |
| Discovery | Platform-wide; only Tier 1+ verified vendors surfaced; ranking = verified + rated, blended with newest as cold-start fill; no pay-to-rank |
| WhatsApp | Pre-purchase: "Ask a question" button (Add to Cart / Buy Now stays the primary CTA). Full vendor contact revealed to buyer after payment |
| Categories (v1) | Fashion & Apparel · Beauty & Personal Care · Food & Groceries · Electronics & Gadgets · Home & Living · Health & Wellness · Kids & Baby · Jewelry & Accessories · Arts, Crafts & Handmade · Other |
| Templates | 3–5 commerce-ready templates at launch, chosen by vendor via a style questionnaire (AI-assisted selection deferred until the library reaches ~15–20 templates) |
| Template design rules | Platform surfaces follow `design.md`. Checkout, order confirmation, and the trust badge are Cartiphy-owned components embedded inside every template, regardless of the vendor's chosen style |
| Vendor dashboard / buyer account visuals | Sleek.design is **dropped**. Visual references are produced with ChatGPT image generation and handed to Codex as design reference to build from directly in code |
| Compliance groundwork | Lawyer review, NDPA privacy policy, CAC considerations, etc. are addressed **after** the product is live, not before launch |

---

## 3. Plans, Pricing and Entitlements

Plans: **Prime** (free) · **Venture** · **Apex**. Annual billing = 10 months' price for 12.

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
| Abandoned carts | Count only | List + click-to-chat WhatsApp link | + automated recovery (later) |
| Analytics | This month | 90 days: trends, top products, traffic | Full history, repeat customers, CSV export |
| Order export | No | Yes | Yes |
| Vendor SMS/WhatsApp broadcasts | No | Monthly allowance (TBD) | Larger allowance (TBD) |
| Staff accounts | No | No | Yes (later) |
| Custom domain | No | No | Yes (later) |
| Support | Help center + email | WhatsApp, business hours | Priority queue |

Never gated by plan: verification, reviews, dispute handling, buyer confirmation, buyer-facing SMS/email, low-stock alerts.

**Trial and lapse rules**
- Everyone gets 14 days of Venture features. Trial vendors pay Venture's 2% commission.
- Trial starts only after Tier 1 verification (bank + phone). One trial per unique phone and per unique bank account.
- After the trial (or a lapsed subscription), the vendor falls back to Prime. Data stays intact. Products over the Prime cap stay live, but new ones cannot be added.
- Upgrade prompts must be honest and calm: live commission-savings meter (only when the saving is real), moment-of-need cards (e.g. 20/25 products used), dimmed locked features with a plan tag, end-of-trial usage summary. No manipulative modals.

**Implementation rule:** all gating reads from a `plans` + `plan_entitlements` config table. Features flip by data, never by scattered `if plan === ...` checks in code.

**Still open (decide with real data, not before):** monthly SMS/WhatsApp allowances for Venture/Apex; exact analytics metric definitions; support-response targets; Apex per-transaction commission cap (deferred — none at launch, see §2).

---

## 4. Architecture

**Rendering flow (storefront)**
1. Request hits `store.cartiphy.com` (wildcard domain).
2. Next.js middleware reads the host header, resolves the store, loads its template and data.
3. Server fills template placeholders (Liquid/Handlebars-style engine) and returns full HTML (so SEO and WhatsApp link previews work correctly).
4. `platform.js` (shared SDK) handles cart actions, checkout initiation, and event tracking against the API.

**Payments and settlement flow (no escrow)**
1. Checkout total = items + vendor delivery fee.
2. Flutterwave charge is created with a **split**: Cartiphy's commission routes to Cartiphy's main account; the vendor's share routes to their own Flutterwave sub-account. The seller bears Flutterwave's processing fee.
3. Order and `order_items` are created only on a **verified webhook** (payment confirmed), never on client-side confirmation.
4. Flutterwave settles the vendor's sub-account to their linked bank account on **Flutterwave's own settlement schedule** — Cartiphy does not hold, release, or gate this money, and the vendor does not request a withdrawal. This is a direct consequence of the no-escrow decision.
5. The vendor dashboard displays **earnings history** (derived from `orders`/`payments`) — never a "balance" or "available to withdraw" figure, since no such balance exists in this model.

**Security baseline (Phase 0, non-negotiable)**
- Row Level Security on every table; role model: vendor, customer, admin.
- Payment and webhook logic runs server-side only. Service-role key never reaches the browser.
- Flutterwave webhook signature verification; idempotent handling via `webhook_events`.
- Rate limiting on auth, checkout, and confirmation-link endpoints.
- Automated cross-tenant test: vendor A must fail to read vendor B's orders. Runs at every checkpoint.
- Stock decrement is atomic (single transaction; reject if insufficient stock).

---

## 5. Trust, Safety and Enforcement

**Marketplace Rules** (plain-language page + versioned ToS acceptance at signup)
- Prohibited: phone numbers, bank details, or "DM to buy" instructions in listings/store text; directing Cartiphy-originated buyers to pay outside checkout; discouraging checkout use.
- Allowed: answering questions on WhatsApp, coordinating delivery after payment, selling to a vendor's own existing customers by chat.

**Prevent first:** saving a listing with contact/bank details or off-platform phrases shows a friendly block with a fix. Only repeated attempts count against the vendor.

**Automatic flags** (probabilistic, never proof) → nightly job → admin queue:
content-scanner hits · buyer reports · chat-tap-to-order ratio vs. platform median (minimum sample size) · sudden order drop with steady traffic/taps.

**Two enforcement tracks**
- *Circumvention (graduated):* educational notice → formal warning email → 30 days removed from discovery → store suspended (no new orders; existing orders still fulfilled) → permanent ban + blocklist (bank account, phone).
- *Fraud (fast track):* non-delivery, fake goods, scam reports → suspension pending review.
- Every action notifies the vendor with a reason and an appeal window. Owner reviews the queue weekly.
- No money-based penalties (funds settle directly to vendors; ToS must not claim payout holds as a penalty tool — the only sanctioned hold is the 24–48h bank-change security hold in §2).

**Vendor verification tiers**
- Tier 0: can build a store; cannot accept live payments.
- Tier 1 (bank account name-match + phone OTP): can sell, appears in discovery, eligible for trial.
- Tier 2 (CAC-verified): "Verified Business" badge.

**Disputes and refunds:** buyer may dispute within 7 days of delivery confirmation; unresolved after 5 days → auto-flag vendor. Refunds are vendor-initiated through Flutterwave, full amount only (no partials at launch); the platform tracks status and enforces via the ladder above. Marketing must not promise "buyer protection" or guaranteed refunds — only verified sellers, a payment record, verified reviews, and a dispute process.

---

## 6. Data Model (by domain — DDL is a Level 2 task)

- **Identity:** `vendors`, `customers` (is_guest flag), `admin_users`, `tos_acceptances`, `blocklist` (bank account hash, phone)
- **Stores and catalog:** `stores` (store_type, subdomain, category, template_id, status, standing), `store_settings`, `products`, `product_images`, `templates`
- **Commerce:** `carts`, `cart_items`, `orders`, `order_items`, `order_status_history`, `customer_addresses`, `discount_codes`
- **Payments:** `payments`, `webhook_events`, `refunds`
- **Trust:** `reviews`, `disputes`, `reports`, `risk_flags`, `enforcement_actions`, `appeals`
- **Billing:** `plans`, `plan_entitlements`, `subscriptions`, `promo_codes`, `trials`
- **Analytics and ops:** `store_events` (view, cart_add, chat_tap, source: discovery/direct), `notification_log`, `audit_log`
- **Admin:** `admin_roles`, `cases`, `case_notes`, `message_templates`, `template_versions`, `platform_settings`, `settings_history`, `feature_flags`, `content_rules`, `policy_versions`, `verification_reviews`, `announcements`, `data_requests`

**Rules:** money = integer kobo; every table has `created_at`; every vendor-scoped table has `store_id` and an RLS policy; bank account changes on `vendors` are append-logged, never overwritten silently.

---

## 7. Admin Panel Summary

*(Full detail lives in the Level 2 Admin document; this is the Level 1 summary.)*

- **Roles from day one** even with a single user: Super Admin, Support, Moderator, Finance — so adding staff later needs no rewrite.
- **Read-only "view as vendor" mode**, fully logged, no edits/money actions/password access.
- **Immutable audit log** on every action: who, what, when, old/new value.
- **Cases, not just flags** — one case groups the flags, reports, disputes, and messages for a single issue.
- **Reversibility** — every ban/suspension is reversible with a reason; timed actions auto-expire.
- **Two-step confirmation** on destructive actions (ban, unverify, bulk actions).
- **Universal search** — phone, email, order ID, subdomain, bank account name.
- **Reconciliation tools** — payments view against Flutterwave records, webhook-events viewer with manual retry.
- **Health dashboard** — vendors by tier/plan, MRR, commission collected, trial conversion, dispute rate, failed payments/webhooks, queue age.
- **Kill switches** — pause all checkouts platform-wide, or pause a single store instantly.
- **Admin-lite is built early** (Phase 3–4), not deferred to Phase 8: 2FA login, read-only orders/payments/webhook-events lookup, audit log foundation.
- **Everything editable, not hardcoded:** plans/pricing/commission, promo codes, category list, reserved subdomains, content-scanner rules, risk thresholds, ToS/Marketplace Rules versions (with forced re-acceptance), message templates, template library, operational timings (trial length, auto-confirm window, dispute window), ranking weights, feature flags, announcements, blocklist.

**Admin build stages**
| Stage | Contents | When |
|---|---|---|
| A0 Admin-lite | 2FA login, read-only orders/payments/webhook-events, audit log foundation | Phase 3–4 |
| A1 Launch-critical | Vendor/store search, verification queue, flag/warn/suspend/ban with templated messaging, disputes/reports queue, content-scanner rules, plan/settings editor, audit log UI | Phase 8 |
| A2 Operations | Cases, appeals, health dashboard, reconciliation, timed actions, bulk actions, extra staff roles | Post-launch |
| A3 Scale | Finance exports, data-request tooling, announcements, advanced analytics, view-as-vendor refinements | Later |

---

## 8. Design System Summary

*(Full detail lives in `design.md`; this is the Level 1 pointer.)*

- **Brand:** Cartiphy, concentric broken-ring icon (near-black outer, terracotta inner), warm and encouraging voice, premium and distinctly Nigerian — never generic global SaaS.
- **Palette (locked):** Cream `#F7F4EF` background, near-black `#1C1B1A` text/dark mode, terracotta `#C1592B` accent (sparing use only — CTAs, badges, one highlight per screen, never large fills).
- **Typography:** Clash Display (headings), Inter (body/UI), fixed type scale — no page invents its own sizes.
- **Motion:** Snappy only, 100–200ms; signature ring-mark loader.
- **Anti-generic rules:** no component-library default look, no SaaS blue/purple gradients, no emoji as UI elements, no generic "four metric cards" dashboard opener, no literal shopping cart/bag icons (use the ring/cycle motif), consistent trust badge everywhere it appears.
- **Vendor Dashboard and Buyer Account visuals:** produced via ChatGPT image generation as reference, then built directly in code by Codex (Sleek.design is not used).
- **Still open, to define in Level 2:** exact type scale values, spacing/radius/shadow tokens, status colors, error-red hex.

---

## 9. Phases

Each phase ends with a **scripted checkpoint** that must pass before the next phase begins.

### Phase 0 — Foundations and security baseline
Repo, Vercel (paid), separate staging/production Supabase projects, env var map (Flutterwave test/live keys), migrations via Supabase CLI in git, Sentry, backups confirmed. Core schema with RLS, roles, `plans`/`plan_entitlements` seed, `store_events` table. Sandbox accounts: Flutterwave, Termii, Resend, Cloudinary.
**Checkpoint:** deployed skeleton; cross-tenant RLS test passes; keys verified.

### Phase 1 — Vendor and store core
Vendor auth (email + phone), ToS/Marketplace Rules acceptance gate. Onboarding wizard: store name, category, single/multi-product, style questionnaire → template pick. Product CRUD with entitlement caps; images via Cloudinary; content-scanner friendly block on listing text. CSV bulk upload (entitlement-gated) with row-level error reporting. Subdomain validation, reservation, routing. A minimal vendor dashboard shell (needed by Phase 3's checkpoint).
**Checkpoint:** vendor signs up, creates a store, adds products, sees the live subdomain.

### Phase 2 — Storefront rendering and cart
Template engine (server-side placeholder filling); commerce slots: product grid, product detail, cart, checkout shell, confirmation. `platform.js` SDK; server-side cart via cookie. Single-product "Buy Now" path reusing the same cart infrastructure. 3–5 commerce-ready templates; SEO and OG tags per store. Event tracking (view, cart_add, chat_tap, source); "Ask a question" WhatsApp button; footer badge gating. Guest capture (email/phone → lightweight customer record).
**Checkpoint:** buyer browses a real store, cart survives refresh, events recorded, link preview renders correctly for both store types.

### Phase 3 — Payments (highest risk; use the strongest model here)
Tier 1 verification: bank account name-match and phone OTP; sub-account creation. Flutterwave checkout with split (commission), delivery fee, seller-borne fee. Verified webhook handler, idempotent via `webhook_events`. Atomic stock decrement; order and `order_items` on confirmed payment only. Admin-lite (A0): 2FA login, read-only orders/payments/webhook-events viewer, audit log foundation.
**Checkpoint:** sandbox payment → webhook → order → stock decrement → vendor sees order in dashboard; duplicate webhook is harmless; concurrent last-item purchase test passes.

### Phase 4 — Order lifecycle and trust mechanics
Status machine with history: Placed → Confirmed → Out for Delivery → Awaiting Confirmation → Delivered / Disputed / Cancelled. Tokenized buyer confirmation link, 72h auto-confirm; vendor contact reveal after payment. Disputes (7-day window, 5-day auto-flag), vendor-initiated full refunds via Flutterwave. Verified-purchase reviews and rating aggregation; vendor standing indicator. Bank-account-change flow (re-verification, logging, 24–48h hold).
**Checkpoint:** full happy path, disputed path, auto-confirm path, and a bank-change event all run correctly end to end.

### Phase 5 — Notifications
Termii SMS for all buyer-facing order events (always included, not plan-gated); Resend for auth, receipts, formal notices; `notification_log`.
**Checkpoint:** every Phase 4 transition sends the right message exactly once.

### Phase 6 — Discovery and insights
Global index, category pages, search (Postgres full-text), store profiles. Ranking: verified + rated, blended with newest; Tier 1+ eligibility only. Vendor analytics by plan; abandoned-cart visibility by plan (count vs. list with click-to-chat).
**Checkpoint:** discovery homepage shows real, eligible stores with sensible ranking; analytics respect entitlements.

### Phase 7 — Monetization and growth tools
Flutterwave subscription billing (monthly and annual), Venture trial rules, fallback to Prime, gentle downgrade. Subscription promo codes; commission-savings meter and upgrade prompts. Buyer discount codes (Venture/Apex); order export.
**Checkpoint:** trial → paid, trial → Prime fallback, promo applied, commission correct at each plan.

### Phase 8 — Moderation, enforcement and admin
Full admin panel (A1): vendor/store search, verification queue, disputes, flagged content, risk queue, templated email/SMS with actions, settings editor, audit log UI. Report button (listings/stores/orders); nightly risk-scoring job; enforcement ladder with notifications, appeals, blocklist.
**Checkpoint:** flagged store reviewed, warned, restricted from discovery, suspended; blocked bank/phone cannot re-register.

### Phase 9 — Launch readiness
Lawyer review of vendor terms and Marketplace Rules; NDPA privacy policy; confirm no raw card data touches Cartiphy. Security review pass, backup restore test, monitoring alerts, full end-to-end run, soft launch with a small vendor cohort.
*(Per §2, the compliance/legal groundwork itself is scheduled to begin after the product is functionally live — this phase's legal items should be read as "start once live," not "block launch on.")*

### Phase 10 — Post-launch backlog
Custom domains, staff accounts, automated cart recovery, zone-based delivery fees, AI template selection, dark mode, Services category, multi-store owners, escrow (only after Flutterwave/legal clarity), expanded template library, Apex commission cap (if data supports it), partial refunds (if needed).

---

## 10. Working Rules for Coding Agents

1. Read this file, `design.md`, and the relevant Level 2 doc before any task; never invent tokens, plans, states, or copy.
2. Build one vertical slice at a time; finish and pass the checkpoint before starting the next phase.
3. Money is integer kobo; never floats.
4. Every table gets RLS at creation; never ship a table without a policy.
5. Payment and webhook code gets extra scrutiny and tests — this is the highest-risk surface in the product.
6. Use the entitlements config for any plan-based behavior; never hardcode `if plan === ...`.
7. Do not add features not listed here; add proposals to the Phase 10 backlog instead.
8. When stuck (especially in Phase 3–4 logic), **stop and escalate for a second opinion rather than guessing or brute-forcing a fix.**

---

## 11. Model, Tooling and Session Guidance

- **Agent:** Codex via ChatGPT Plus — a usage-limited plan, not pay-per-token credits. Budgeting is therefore about **sessions and time**, not dollars.
- Because session rhythm on this plan hasn't been tested yet, Level 3 task cards should start **modest and self-contained** (one clear task, one checkpoint, minimal cross-file sprawl) so a single session can realistically finish one without running into a quota reset mid-task.
- Recalibrate task-card size after the first few real Codex sessions show how much ground one session actually covers.
- Reserve the most careful, well-specified task cards for Phases 0, 3, 4, and 8 (security, payments, state machine, enforcement). Lighter-weight cards are fine for CRUD/UI-heavy phases (1, 2, 6).
- Recurring costs beyond the Codex subscription: Vercel paid plan, Supabase paid tier (for backups), SMS spend, domain — track these separately from build effort.

---

## 12. Start-Now External Tasks (long lead time)

- Flutterwave: confirm split payments/sub-accounts are approved for your business account as a marketplace; confirm KYB documents required.
- Termii: sender ID approval; WhatsApp Business API application.
- Resend: domain DNS (SPF/DKIM).
- Vercel: paid plan; confirm wildcard-subdomain setup.
- Supabase: confirm backup coverage on the chosen plan.
- Legal (scheduled to begin once the product is live, per §2): Nigerian lawyer for vendor terms, Marketplace Rules, and funds-model review; NDPA-aligned privacy policy; accountant on VAT for Cartiphy's subscription and commission income.

---

## 13. Open Items Carried Into Level 2

- Exact design tokens: type scale, spacing, radius, shadow values, status colors, error-red hex.
- Monthly SMS/WhatsApp allowances for Venture and Apex; exact analytics metric definitions; support-response targets.
- Enforcement thresholds (chat-tap ratio, minimum sample sizes) — tune after real data exists.
- Detailed DDL and RLS policies per table; per-phase task card breakdown and prompts.
- Style questionnaire content (what it asks, how it maps to template choice).
- Data retention and deletion periods (set with a lawyer once legal groundwork begins).

---

*This document supersedes all earlier build-plan drafts. Read alongside `design.md` (visual system) and the Level 2 documents as they're written, starting with Payments.*
