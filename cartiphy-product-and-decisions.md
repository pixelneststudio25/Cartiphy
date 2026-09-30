# Cartiphy — Product Overview & Decision Log

A plain-language description of what Cartiphy is, the problems it solves, and every major decision made while planning it, with the options weighed and the reasons behind each choice. Companion files: `build-plan-v2.md` (technical plan), `design.md` (design rules), `admin-panel-plan.md` (admin panel).

---

# Part A — What Cartiphy is

## A1. In one paragraph

Cartiphy lets anyone in Nigeria open a professional online store in minutes. A seller picks a store style, adds products, and gets their own store link (for example `yourbrand.cartiphy.com`). Customers pay securely through Flutterwave, and the money goes straight to the seller's bank account. Cartiphy also runs a discovery page where shoppers can find verified, well-reviewed stores. Delivery stays in the seller's hands.

## A2. The problems it solves

| The problem today | How Cartiphy answers it |
|---|---|
| Most Nigerian online selling happens in WhatsApp and Instagram DMs. There are no records, no proof, and buyers pay first and hope. Scams make shoppers hesitant. | Every sale goes through a real checkout with a payment record, an order history and a dispute process. |
| Global store builders like Shopify price in dollars and assume foreign payment methods and shipping companies. They carry heavy features most local sellers never use. | Naira-first, Flutterwave-native, with deliberately simple features (flat products, seller-arranged delivery). |
| Getting a proper store usually means paying a developer or agency. | Sellers self-serve: answer a few questions, get a polished, animated store. |
| A seller has no way to prove they are trustworthy. | Verification tiers, a visible "Verified" badge, and reviews that only real buyers can leave. |
| Shoppers have nowhere to find trustworthy stores. | A discovery page that only surfaces verified sellers, ranked by ratings. |
| Order updates get lost because people rarely open email. | SMS (and later WhatsApp) for order events; email only for account and formal matters. |

## A3. Who uses it

- **Vendors:** sellers running a store. Both small informal sellers (WhatsApp/Instagram style, a few dozen products) and established small businesses with larger catalogs.
- **Customers:** shoppers. They can buy as a guest or use one Cartiphy account across every store.
- **Platform admin:** the Cartiphy team, who verify vendors, review disputes and reports, and enforce the rules.

## A4. How it works

**A vendor's journey**
1. Sign up and accept the marketplace rules.
2. Answer a few questions (store name, category, single or multiple products, style). Cartiphy picks a matching template.
3. Add products one by one or by spreadsheet upload.
4. Verify their bank account and phone number so they can accept payments.
5. Share the store link. Orders arrive in the dashboard and by SMS.
6. Arrange delivery themselves, mark the order out for delivery, and get paid directly through Flutterwave.

**A customer's journey**
1. Find a store by link or through discovery.
2. Browse, add to cart, and optionally tap "Ask a question" to chat with the vendor on WhatsApp.
3. Pay at checkout. The vendor's contact details appear after payment for delivery coordination.
4. Confirm delivery from a one-tap link in an SMS (or it auto-confirms after 72 hours).
5. Leave a review, or raise a dispute within 7 days.

## A5. What it costs vendors

| | Prime | Venture | Apex |
|---|---|---|---|
| Monthly price | Free | ₦12,000 | ₦28,000 |
| Cartiphy commission per sale | 4% | 2% | 1% |

Everyone starts with 14 days of Venture features. After that they subscribe or fall back to Prime with their data intact. Sellers also pay Flutterwave's processing fee (about 2% plus VAT). Paying yearly gives two months free.

## A6. What Cartiphy deliberately does not do (yet)

Shipping or courier integration, multiple currencies, product variants (size/colour systems), holding customer money in escrow, custom domains, staff accounts, automated cart recovery, service/booking stores, dark mode, and its own CMS.

---

# Part B — Decision log

Format: **Decision | Options weighed | Why**

## B1. Vision and scope

| Decision | Options weighed | Why |
|---|---|---|
| Build a focused Nigerian store builder, not a Shopify clone | Full Shopify competitor vs focused local product | A full clone is impossible on this budget. The real gap is a backend for stores plus local payments and trust. |
| Standalone brand and codebase, separate from the owner's AI website builder | Extend the existing builder vs start fresh | The existing builder has no backend, and the owner wants a distinct new brand. The template-selection idea is reused, not the code. |
| Nigeria first, NGN only | Multi-currency, pan-African at launch | Focus. Currency handling and country-specific payments multiply work. |
| No custom CMS | Build one vs Postgres schema plus dashboard | Commerce data is structured data, not "content". A CMS would consume most of the budget and compete with mature free tools. A small block editor can come later if vendors ask. |
| Realistic budget of about $150–600 for an MVP (not $20) | $20 proof of concept vs working MVP vs launch-ready product | $20 buys a barebones demo at best. Payments and webhooks need debugging cycles. Recurring costs were later added (see Part C). |
| Fully vibecoded build with checkpoints | Big-bang generation vs small tested slices | The owner can't debug code personally, so each phase ends with a passing test before the next begins. |
| Three levels of planning before building | Build first, plan as you go | A previous project ran into detail gaps that cost extra credits to fix. Locking decisions first is cheaper. |

## B2. People and accounts

| Decision | Options weighed | Why |
|---|---|---|
| Three roles: Vendor, Customer, Platform Admin | Adding staff members, multi-store owners, affiliates | Each extra role multiplies auth and permission work. Staff accounts are a later Apex feature. |
| Working names "Vendor" and "Customer" | "Merchants/Shoppers", "Sellers/Customers", more elegant names | Clarity beats elegance for now. Naming can be revisited during brand work. |
| Customer accounts are platform-wide | Store-specific accounts vs one account everywhere | Builds a real platform: returning customers, cross-store history, and a base for discovery. Costs a bit more setup (central auth). |
| Guest checkout silently creates a lightweight customer record | Fully anonymous guests vs auto-created record | Forcing signup kills conversion. A silent record lets order history build up. Later merge is via a one-time code on the same phone or email so nobody can claim someone else's orders. |

## B3. Store building and templates

| Decision | Options weighed | Why |
|---|---|---|
| Stores are picked from a template library, not generated per store | AI-generated code per store vs template selection | Per-store generation is expensive and unreliable. Selection costs almost nothing and templates are already premium and animated. |
| Templates are display layers fed by one shared backend | Backend baked into each template vs one shared API | One backend serves unlimited templates. Every template implements the same "slots" (product grid, product page, cart, checkout, confirmation). |
| Start with 3–5 polished templates | A large library at launch | Making a template commerce-ready is real work. Ship a few solid ones and expand later. |
| Questionnaire picks the template for now; AI selection later | AI selection from day one | With only 3–5 templates, AI adds cost with no benefit. Revisit when the library reaches roughly 15–20. |
| Templates written in plain HTML/CSS/JavaScript | React/Next.js templates | The owner knows HTML/CSS/JS well, so can read and sanity-check template code. The platform itself uses Next.js. |
| Server-rendered templates, not client-rendered | Client-side fetch (empty HTML shell) vs server rendering | Client rendering gives weak SEO and empty WhatsApp link previews, which matter a lot in Nigeria. Server rendering fixes both. |
| Server-side cart tied to a cookie | Browser-only cart | Browser-only carts vanish easily and can't feed abandoned-cart insight. |
| Vendors get limited customization | Full drag-and-drop editing | Full customization brings back per-store complexity. |
| Cartiphy owns checkout, order confirmation and the trust badge inside every template | Fully vendor-styled everything | Buyers should see identical trust cues everywhere. Templates style the rest. |
| Stores subdomain-based (`name.cartiphy.com`) with naming rules (lowercase, letters/numbers/hyphens, 3–30 characters, reserved words blocked) | Path-based URLs, freeform names | Subdomains feel like real stores. Rules prevent messy or impersonating names such as "admin" or "www". |

## B4. Catalog

| Decision | Options weighed | Why |
|---|---|---|
| Flat products: name, price, stock, images | Built-in variants (size/colour) | Variants add heavy complexity. Vendors create separate products for each variant in v1. |
| Both single upload and CSV bulk upload | UI only | Established sellers with big catalogs need spreadsheet import. Bulk upload is gated to Venture and above. |
| Vendors choose single-product or multi-product store | One store type only | A store type field, plus template constraints. Not a separate schema. |
| Ten launch categories; Services deferred | Include Services (bookings) | Services need time slots instead of stock, which is a different data model. Ten product categories cover what Nigerians sell online. |
| Prices are vendor-set final prices, no platform tax logic | Building VAT handling | Punted for v1 to limit scope. Vendors handle their own tax obligations. |

## B5. Payments and funds

| Decision | Options weighed | Why |
|---|---|---|
| Flutterwave as the payment provider | Paystack, Stripe | The owner chose it. Naira-native and supports local payment methods. |
| Sub-accounts (split payments) sending money directly to vendors | Pooled account paying vendors out later | Holding vendor money creates cash-flow risk and possible regulatory exposure (CBN rules). Sub-accounts settle straight to vendors' banks. |
| **No escrow at launch** | Real escrow through pooled collection vs sub-accounts | Discovered mid-planning that sub-account settlement can't be held until buyer confirmation. Escrow would require holding customer funds, which needs legal and Flutterwave clarity first. Trust instead comes from verification, verified reviews, disputes and enforcement. Escrow stays on the later backlog. |
| Dropped WhatsApp-based checkout | WhatsApp payments/ordering | It defeats the point of the platform: no payment record, no dispute evidence, easy to scam. WhatsApp is for questions and notifications. |
| Seller bears Flutterwave's processing fee | Buyer pays, shared | Simple and standard for seller platforms. The owner chose it. |
| Delivery stays off-platform; vendor sets a flat delivery fee added at checkout | No fee at checkout, arranged on WhatsApp; zone-based fees; free delivery only | If delivery is paid separately, buyers lose protection and records. Flat fee (0 for free delivery) keeps the whole payment on-platform. Zone-based fees come later. |
| Commission applies to item price only, not delivery | Include delivery | Cleaner and fairer. |
| All money stored as integer kobo | Decimals | Avoids rounding errors. |

## B6. Trust, verification and enforcement

| Decision | Options weighed | Why |
|---|---|---|
| Three verification tiers: Tier 0 build only; Tier 1 bank and phone verified (can sell, appears in discovery, eligible for trial); Tier 2 CAC-verified business badge | Full KYC for everyone, no gating | Building is free of friction; taking money requires a real bank account. CAC is optional to earn a badge. |
| Reviews only from verified on-platform purchases | Open reviews | Prevents fake reviews and rewards on-platform selling. |
| Buyer confirms delivery via tokenized link; auto-confirms after 72 hours | Vendor self-reports delivery | Vendor self-reporting is easy to abuse. With no escrow, confirmation now unlocks reviews and closes the dispute window. |
| Disputes within 7 days of delivery confirmation; auto-flag at 5 days unresolved | Longer or shorter windows | A balance between buyer safety and vendor certainty. |
| Refunds are vendor-initiated through Flutterwave; Cartiphy tracks and enforces | Platform absorbs refunds | Cartiphy is not the merchant of record. |
| Trial abuse prevented by one trial per unique phone and bank account | Email-only, device fingerprinting | Emails are free to create; bank accounts and phone numbers are real friction. Fingerprinting is a weak secondary signal, skipped. |
| WhatsApp visibility: "Ask a question" button before purchase, full contact after payment | Always public, hidden, masked numbers | Nigerian buyers expect to chat before paying; hiding numbers hurts conversion. Masking is costly and easy to bypass. Add-to-cart stays primary. |
| Anti-circumvention rule kept narrow | Ban all off-platform contact | Vendors' own audiences and post-order delivery chats can't and shouldn't be policed. |
| Prevent first: a friendly block when listing text contains phone numbers, bank details or "DM to buy" | Punish after the fact | Most vendors comply when told at the moment. |
| Automatic risk flags feed an admin queue; a human decides | Automatic bans | Signals like chat-tap-to-order ratio are probabilistic, so wrongful bans are a real risk. |
| Graduated ladder for circumvention plus a fast track for fraud; appeals; no money penalties | Fines, payout holds | Funds settle directly to vendors, so payout holds are impossible. Bans blocklist the bank account and phone. |
| Marketing must not promise guaranteed "buyer protection" | Claiming protection | Without escrow it would overpromise. Claim only verified sellers, payment records, verified reviews and a dispute process. |

## B7. Notifications

| Decision | Options weighed | Why |
|---|---|---|
| SMS via Termii for order events; WhatsApp Business API added later | SMS only, WhatsApp only | SMS works immediately. WhatsApp needs Meta approval with a long lead time. |
| Email via Resend for account, receipts and formal notices | SMS for everything | Email is best for records and paper trails. |
| Buyer-facing order messages are always included on every plan | Gating them | They are part of the trust flow. Gating them would break it. |

## B8. Discovery

| Decision | Options weighed | Why |
|---|---|---|
| Add a platform-wide discovery system | Subdomain-only access | Turns Cartiphy from a hosting tool into a platform with its own demand. Modest extra work. |
| Only Tier 1+ verified vendors are surfaced | Everyone listed | Discovery is an implicit recommendation, so quality matters. |
| Ranking: verified and rated first, blended with newest stores as cold-start fill | Rating only, newest only | Avoids an empty-looking or stale homepage early on. |
| No pay-to-rank | Featured placement for higher plans | Paid boosts would undermine the trust promise. Labeled featured slots may come later. |

## B9. Money model

| Decision | Options weighed | Why |
|---|---|---|
| Subscription plus commission | Commission only, subscription only | Subscription revenue can't leak off-platform; commission scales with sales. |
| Plan names Prime, Venture, Apex | Starter/Growth/Business | The owner wanted non-generic names. |
| Prime is a permanent free tier; everyone gets 14 days of Venture, then subscribes or falls back to Prime | Paid-only with a trial | Lowers the barrier for informal sellers, gives a graceful lapse behaviour, and largely removes trial abuse. |
| Commission stays during the trial (at Venture's 2%) | Waive commission during trial | Avoids a never-monetized period and makes the post-trial jump visible. |
| Prices: Prime free/4%, Venture ₦12,000/2%, Apex ₦28,000/1% | Prime at 5%; other price points | Break-evens fall at roughly ₦600k and ₦1.6M monthly sales. Prime lowered from 5% to 4% to stay competitive and reduce the incentive to sell off-platform. |
| Annual billing gets 2 months free | Monthly only | Helps cash flow. |
| Never gate trust or safety features | Gate everything | Verification, reviews, disputes, low-stock alerts and buyer messages stay free on all plans. |
| Upgrade design: live savings meter, moment-of-need cards, visible locked features, end-of-trial summary, gentle downgrade | Pop-ups, hard limits | Honest prompts fit the warm brand voice. Over-cap products stay live. |
| Buyer discount codes included as a Venture/Apex feature | Skip | Requested by the owner. Added to the plan. |
| Lapse handling and promo-season mechanics are deferred | Decide now | Build first; schema stays flexible. |

## B10. Technology and operations

| Decision | Options weighed | Why |
|---|---|---|
| Next.js + Supabase + Vercel (paid) + Cloudinary | Less common stacks | The most AI-friendly combination, meaning fewer failed generations and a lower credit burn. |
| Supabase Auth, not hand-rolled auth | Custom auth | Auth bugs are subtle and security-critical. |
| Row Level Security everywhere; server-only payment logic; webhook signature checks; rate limits; cross-tenant test | Trusting generated code | The classic vibecoding failure is one vendor reading another's data. |
| Separate staging and production; Sentry; backups; migrations in git; one scripted end-to-end test per checkpoint | Ad hoc | Protects a builder who can't debug by hand. |
| Stronger AI models only for payments, RLS, state machines and enforcement; cheaper models for CRUD and UI | One model for everything | Controls credit spend where it matters. |

## B11. Brand and design

| Decision | Options weighed | Why |
|---|---|---|
| Name: Cartiphy (cartiphy.com) | Other names | Short, invented-sounding, easy to say and spell over a phone call. |
| Palette: cream `#F7F4EF`, near-black `#1C1B1A`, terracotta `#C1592B` | Green + gold, navy + terracotta, charcoal + emerald, plum + amber | Started on green/gold, then pivoted. The final palette is warmer, more distinctive, avoids corporate blue and Jumia orange, and suits the warm voice. |
| Light theme first; dark mode later with `#1C1B1A` as the dark base | Both at launch | Dark mode doubles design and QA work. |
| Fonts: Clash Display for headings, Inter for UI | Cabinet Grotesk, General Sans, Satoshi, IBM Plex Sans | Both free. Clash suits the wordmark; Inter is highly legible. |
| Logo: two-tone concentric broken-ring mark | Original single ring, an AI-generated blob, a three-ring version | The AI-generated concept was generic and unreadable at small sizes. The three-ring version was busy. Two colours give clear separation at favicon size. The ring doubles as the loader. The abstract mark was accepted deliberately. |
| Voice: warm and encouraging | Sharp/confident, plain/no-nonsense | Fits the trust-building positioning. |
| Motion: snappy 100–200ms; signature loader is the rotating ring mark | Smooth/elegant | Efficient feel; the loader ties directly to the logo. |
| Imagery: real photography, not illustration | Illustrated/abstract | Real people and products signal trust. |
| Twelve anti-generic design rules (see design.md) | Unconstrained generation | Coding agents drift toward generic patterns unless constrained. |

## B12. Design workflow

| Decision | Options weighed | Why |
|---|---|---|
| Sleek.design for vendor dashboard and buyer account; Figma for marketing, discovery and admin; system elements specified in design.md | One tool for everything | Sleek is mobile-first and unsuited to marketing sites or storefront templates. |
| Review each generated screen against the design rules before using it | Use generations as-is | The first dashboard concept broke three rules (emoji, shopping-bag icon, blue status badge). Reviews catch drift. |
| Regenerate via a corrected prompt rather than use the HTML reference I built | Keep my HTML | The owner didn't like it. Approved screens will be converted to reference HTML later. |

---

# Part C — Course corrections (honest record)

| Item | What was first said | What changed and why |
|---|---|---|
| Budget | $20 could work | Realistic range $150–600 plus recurring costs (paid Vercel plan, Supabase paid tier for backups, SMS). |
| Escrow | Sub-accounts and escrow-delay payouts both approved | They contradict each other. Escrow dropped at launch. |
| Payout holds as enforcement | Hold vendor payouts | Impossible with direct settlement; enforcement is non-monetary. |
| "Buyer protection" wording | Listed as a Cartiphy counter to off-platform selling | Overpromises without escrow; marketing restricted. |
| Storefront rendering | Client-side `platform.js` fetching | Server-rendered for SEO and link previews. |
| Cart | Session-based, no local storage | Vague; replaced by a server-side cart with a cookie. |
| Data protection law | NDPR | The current law is the Nigeria Data Protection Act 2023. |
| Vercel | Free tier initially | The free plan isn't for commercial use; a paid plan is budgeted. |
| Colours | Green and gold | Terracotta/cream/near-black. |
| Trial | 14 days, commission still applies | Refined: 14 days of Venture with Venture's commission, then fall back to Prime. |
| Prime commission | 5% | 4%. |
| AI template selection | AI picks templates at launch | Questionnaire first; AI once the library is large. |

---

# Part D — Still open

- Design tokens: type scale, spacing, radius, shadow values, status colors, error-red hex.
- Monthly SMS/WhatsApp allowances for Venture and Apex.
- Enforcement thresholds (tune after real data).
- Support response targets.
- Optional per-transaction commission cap on Apex.
- Detailed table definitions, security policies and per-phase task breakdowns.
- Start-now tasks: Flutterwave marketplace approval, Termii sender ID and WhatsApp approval, Resend DNS, Vercel and Supabase plans, lawyer and accountant review.
