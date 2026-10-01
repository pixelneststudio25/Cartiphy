# Cartiphy — Level 2: Template Library and Design System

Reads against `cartiphy-level-1-master-plan.md`, `design.md`, and `cartiphy-level2-storefront-cart.md`. This is the contract for every vendor-facing storefront template: how they are built, what each must contain, how they are categorized, and how the library grows from 15 at launch toward 100.

Items marked **(proposed)** are recommendations awaiting confirmation; everything else is locked.

---

## 1. Scope and Ground Rules

- A template is **real code in the Next.js repo**, implementing the shared "slots" interface (product grid, product detail, cart, checkout shell, confirmation). The `templates` table holds only metadata (§9). Adding a template is always a coding task, never a content-only operation.
- **Vendors never touch code** (Storefront doc §1). They fill four fields (logo, images, About text, WhatsApp number) plus the product catalog. Template variety comes from Cartiphy's library, not per-vendor tweaking.
- **Cartiphy-owned components are fixed across every template:** checkout, order confirmation, the trust/verification badge, and the store policy page (Storefront doc §16). They always render in Clash Display / Inter and follow `design.md`, whatever template or font pairing the vendor chose.
- **Every template must be completely distinct** — different layout structure, hero treatment, grid behavior, header/footer, and motion personality. A palette or font swap of an existing template does not count as a new template.
- **Launch target: 15 templates. Long-term target: 100.** The jump from 15 to 100 is post-launch work (Level 1 Phase 10).

---

## 2. Font System

- Fonts come as **defined pairings** (one heading font + one body font bound together), never mixed freely. A template is assigned exactly one pairing.
- Pairings apply only to the vendor-styled parts of a template: header, hero, product names and copy, About section, footer. They never apply to Cartiphy-owned components (§1).
- All fonts are free Google Fonts, self-hosted and subset to Latin, `font-display: swap`, max two families per template (performance, §6).
- **(proposed) Library of 15 pairings** — confirm or swap before Level 3 task cards reference them:

| # | Heading + Body | Personality |
|---|---|---|
| F01 | Playfair Display + Source Sans 3 | Luxury / editorial |
| F02 | DM Serif Display + DM Sans | Editorial / warm |
| F03 | Cormorant Garamond + Montserrat | Luxury / refined (jewelry, beauty) |
| F04 | Space Grotesk + IBM Plex Sans | Tech / precise (electronics) |
| F05 | Sora + Nunito Sans | Modern minimal |
| F06 | Bebas Neue + Work Sans | Bold / streetwear |
| F07 | Archivo Black + Archivo | Bold / punchy |
| F08 | Poppins + Lato | Clean / friendly |
| F09 | Lora + Karla | Warm / food and home |
| F10 | Caveat Brush + Mulish | Handmade / personal (use sparingly for headings only) |
| F11 | Fredoka + Quicksand | Playful (kids and baby) |
| F12 | Bricolage Grotesque + Manrope | Contemporary / distinctive |
| F13 | Libre Baskerville + Open Sans | Classic / trustworthy |
| F14 | Syne + Outfit | Avant-garde fashion |
| F15 | Abril Fatface + Raleway | Dramatic editorial (beauty, fashion) |

Clash Display and Inter are deliberately excluded from this list so vendor storefronts never resemble Cartiphy's own surfaces.

---

## 3. Responsive Breakpoints (locked)

| Range | Target |
|---|---|
| ≤380px | Small phones |
| 381–480px | Standard phones |
| 481–768px | Large phones / small tablets |
| 769–1024px | Tablets / small laptops |
| 1025px+ | Desktop |

Rules:
- **Mobile-first.** Base styles target ≤380px; larger ranges add to them. Most Nigerian buyers arrive from a phone via a WhatsApp link, so the phone layout is the primary design, not a reduction of the desktop one.
- Every template spec (§8) must describe layout behavior at each of the five ranges: columns in the product grid, header behavior, hero treatment, cart presentation.
- No horizontal scrolling at any width. Minimum tap target 44px.
- Every template is tested at 360, 390, 430, 768, 1024 and 1440px wide.

---

## 4. Mandatory Template Anatomy

Every template, regardless of style, must implement all of the following.

**Header and navigation**
- Store logo (or store name fallback), cart entry point, and a way to reach the category filter and policies.
- Cart entry point shows item count.

**Hero / store introduction** — driven only by the store's banner image, About text and name.

**Multi-product stores**
- Product grid with the category filter (only if products span more than one category).
- **Sort and price filter:** sort by Featured (vendor order, default), Newest, Price low-to-high, Price high-to-low; optional min/max price filter. Sorting is client-assisted over server-rendered HTML; the default render must work without JavaScript.
- Empty-category and no-matching-filter states (warm-voiced, per design rule on empty states).

**Single-product stores** — no grid or filter; the product detail page is the homepage.

**Product detail page (all templates)**
- **Gallery:** every image the product has, swipeable on touch, thumbnails or dots, **tap/click-to-zoom** (pinch-zoom on phones). Works for 1 to 10 images.
- Name, price, stock state (in stock / low / sold out), description.
- **Product details block:** renders the structured `products.details` label/value pairs (materials, size, care, etc.) when present.
- **Delivery and total transparency:** the flat delivery fee is shown on the page next to the price (for example "+ ₦1,500 delivery"), so no cost appears for the first time at checkout.
- Primary CTA: Add to Cart (multi-product) or Buy Now (single-product). Secondary: "Ask a question" WhatsApp link (Storefront doc §6).
- Link to the store policy page.

**Cart (drawer or page, template's choice)**
- Line items with quantity controls and remove.
- **Subtotal, delivery fee and total shown before the buyer proceeds**, so the checkout total never changes from what the cart displayed.
- "No longer available" handling for out-of-stock items (Storefront doc §7).
- Single-product stores skip this view via Buy Now, but the checkout shell's first step still shows the same breakdown.

**Footer**
- Store name, policies link, contact-via-WhatsApp link (pre-purchase form only, per anti-circumvention rules).
- "Sold on Cartiphy" badge: shown on Prime, removable on Venture/Apex (entitlement-driven, never hardcoded).

**Cartiphy-owned components:** checkout shell, order confirmation, trust badge, policy page — embedded, never restyled (§1).

**Required states:** loading (skeletons or the ring-mark loader), empty cart, error, sold-out, store coming-soon (Cartiphy-owned, not per template).

---

## 5. Motion

- Snappy only, **100–200ms**, per `design.md`. Ease-out for entrances, no bouncing or looping decoration.
- Motion must have a purpose: state change, feedback, or transition between views.
- Respect `prefers-reduced-motion`: animations collapse to instant state changes.
- Each template spec (§8) lists its specific animations; none may exceed the performance budget.

---

## 6. Performance Budget (proposed targets)

Buyers often load these pages on mid-range Android phones over 4G, frequently from a WhatsApp preview tap. Target: a product page **usable in under 3 seconds** on that profile.

| Item | Target |
|---|---|
| Largest Contentful Paint (mid-range Android, 4G) | ≤ 2.5s |
| Initial HTML + CSS + JS (gzipped), per page | ≤ 150KB |
| Hero / banner image delivered | ≤ 150KB |
| Product grid thumbnail delivered | ≤ 60KB |
| Font files per template | ≤ 2 families, Latin subset only |
| Layout shift (CLS) | ≤ 0.1 |

Rules:
- All images served through Cloudinary with automatic format (WebP/AVIF) and quality, and explicit width/height to prevent layout shift.
- Images below the fold are lazy-loaded; the first product image / hero is prioritized.
- No template-specific JavaScript frameworks; `platform.js` plus minimal vanilla interactivity.
- A template that misses these targets in QA (§12) does not ship.

---

## 7. Distinctness Rule

Each template must differ from every other on **at least** these dimensions: overall layout structure, hero treatment, product-grid behavior, header pattern, footer pattern, cart presentation, and motion personality. Font pairing and palette are a supporting difference, never the only one.

The build pipeline (§11) enforces this: each new template's reference is compared against the existing library before a task card is written.

---

## 8. The Per-Template Spec Sheet

Every template has a one-page spec (written at Level 3 as a task-card input) defining:

1. Name, style tag, suggested categories, store-type compatibility (single / multi / both), plan tier.
2. Assigned font pairing (§2).
3. Layout at each breakpoint (§3): header, hero, grid columns, PDP arrangement, cart presentation.
4. Image slots the template exposes (banner, logo, any extra), with recommended sizes — needed for the template-switch content-mapping warning (Storefront doc §2).
5. Animations and transitions (§5), each with duration.
6. Footer design and the badge placement.
7. Empty, loading and sold-out state treatments.
8. Color application: how it uses `design.md`'s palette, plus any template-specific accent palette. Vendor-styled areas may use a template palette; Cartiphy-owned components never do.
9. Performance notes (heaviest asset, how it stays in budget).

---

## 9. Taxonomy, Filtering and Data Model

Two tag axes, both advisory (no lock-in):

- **Category affinity:** which of the 10 fixed categories the template suits best (one or more).
- **Style tag:** Minimal, Editorial, Bold, Handmade, Luxury, Playful, Tech, Classic, and others as the library grows.

Plus: store-type compatibility (single, multi, both) and plan tier access.

**Vendor-facing picker:** filters by category affinity and style tag; when the vendor's onboarding category is known, templates with matching affinity are shown first, but all compatible templates remain reachable. The style questionnaire (Level 1 open item) maps answers to style tags and suggests a short list.

`templates` table (metadata only): `name`, `slug`, `thumbnail_url`, `style_tags[]`, `category_affinity[]`, `store_type_support` (single / multi / both), `min_plan_tier`, `font_pairing_id`, `is_active`, `created_at`. `is_active` lets admin retire a template without deleting it; stores already using it keep working.

---

## 10. Launch Roster and Growth Path

**(proposed) Launch roster — 15 templates.** Each is a distinct layout family; font pairings shown are suggestions.

| ID | Layout family | Style | Stores | Suggested categories |
|---|---|---|---|---|
| T01 | Clean Grid (dense, minimal) | Minimal | Multi | Electronics, Home & Living |
| T02 | Clean Grid (wide cards, sidebar filter) | Tech | Multi | Electronics, Health & Wellness |
| T03 | Editorial (large imagery, asymmetric) | Editorial | Multi | Fashion, Beauty |
| T04 | Editorial (magazine rhythm) | Luxury | Multi | Jewelry, Beauty |
| T05 | Lifestyle (full-bleed hero, warm) | Editorial | Both | Food, Home & Living |
| T06 | Single-Product Spotlight (hero-first) | Bold | Single | Any |
| T07 | Single-Product Spotlight (story scroll) | Editorial | Single | Beauty, Health, Fashion |
| T08 | Handmade / Craft (textured, personal) | Handmade | Both | Arts & Crafts, Jewelry |
| T09 | Handmade / Craft (collage layout) | Handmade | Multi | Arts & Crafts, Home |
| T10 | Bold Street (oversized type, tight grid) | Bold | Multi | Fashion, Accessories |
| T11 | Playful (rounded, colorful blocks) | Playful | Both | Kids & Baby |
| T12 | Market (compact list-first, many products) | Classic | Multi | Food & Groceries, Other |
| T13 | Boutique (centered, airy, few products) | Luxury | Multi | Jewelry, Fashion |
| T14 | Showcase (horizontal scroll sections) | Minimal | Multi | Fashion, Home |
| T15 | Catalogue (table-like, spec-forward) | Tech | Multi | Electronics, Other |

**Build order (proposed):** ship the Phase 2 checkpoint on the first 3 templates (T01, T06, T03 — one grid, one single-product, one editorial, covering every store type and the slot interface). Build the remaining 12 in batches of 3–4 as a parallel template track before launch, each batch passing the template QA checklist (§13) before the next starts.

**Toward 100 (post-launch, Phase 10):** grow in batches, prioritizing categories with the most vendors and the least template coverage, guided by template-pick and template-switch data. Plan-tier access stays controlled by `min_plan_tier`. Level 1's deferred AI-assisted template selection becomes practical once the library is large enough to make the questionnaire feel limiting.

---

## 11. Build Pipeline and the Inspiration Rule

1. **Reference:** generate or gather a visual reference (ChatGPT image generation for originals; Pinterest boards for mood and pattern research).
2. **Spec sheet** (§8) written, including distinctness check against the existing library.
3. **Codex task card:** reference + spec + `design.md` + the slots interface.
4. **Build, register** in the `templates` table, **QA** against §13.

**Pinterest and other inspiration — the rule for every task card:**
- Inspiration boards are for mood, layout rhythm, color pairing and typographic pairing.
- Task-card wording must say: **"synthesize an original design inspired by the patterns across these references"** — never "replicate this screenshot."
- A close copy of one live store or paid theme is an IP risk and also fails the distinctness rule (§7).
- Reference images are never used as assets inside a template or the Cartiphy Image Library (see the Vendor Content Tools doc for permitted image sources).

---

## 12. Data Model Additions

- `templates` fields per §9.
- `stores.template_id` (already exists) plus the switch-cadence fields in Storefront doc §14.
- `store_template_history` (optional, proposed): store_id, template_id, switched_at — useful for template-popularity data.

---

## 13. Template QA Checklist (runs for every template before it ships)

1. Passes every anatomy item in §4, including gallery + zoom, sort/price filter (multi), and the delivery-fee and total display.
2. Renders correctly at all five breakpoint ranges and the six test widths in §3, with no horizontal scroll.
3. Meets the performance budget in §6 on a throttled mid-range Android profile.
4. Only uses its assigned font pairing in vendor areas; checkout, confirmation and trust badge render in Clash Display / Inter and look identical to the same components in every other template.
5. Every animation is 100–200ms and respects reduced-motion.
6. Works for its declared store types (single / multi / both) and handles 1 product, 1 image, and a long product name without breaking.
7. Content maps cleanly when a vendor switches to and from it (warning shown when a slot won't carry over).
8. Distinctness review against the existing library is documented.
9. Link preview (OG) renders correctly for a product page in WhatsApp.

---

## 14. Open Items

- Confirm or swap the 15 font pairings (§2) and the launch roster (§10).
- Which 3 templates are the "3 core" Prime templates (Level 1 §3)?
- Exact performance budget numbers (§6) — confirm after the first real template is measured.
- Whether templates may use their own cart icon or must use the ring/cycle motif from `design.md` anti-generic rule.
- Style questionnaire content and mapping to style tags (Level 1 open item, now sized for 15 templates).

---

*This document supersedes any earlier informal notes on templates. Read alongside `cartiphy-level-1-master-plan.md`, `design.md`, `cartiphy-level2-storefront-cart.md`, and `cartiphy-level2-vendor-content-tools.md`.*
