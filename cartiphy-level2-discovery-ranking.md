# Cartiphy — Level 2: Discovery and Ranking

Reads against `cartiphy-level-1-master-plan.md` and the other Level 2 documents (Payments, Order Lifecycle, Storefront/Cart, Admin Panel). This is the contract for Phase 6's discovery homepage, category pages, search, and store profile pages.

---

## 1. Ranking Formula

Level 1 specified "verified + rated, blended with newest as cold-start fill" directionally. This is the concrete version.

**Composite score per store (or per product, depending on the listing context — see §3):**

```
score = bayesian_rating × vendor_standing_factor × recency_boost
```

- **`bayesian_rating`** — the store's average rating adjusted using a Bayesian estimate (pulling the average toward the platform-wide mean when a store has few reviews). This prevents a store with a single 5-star review from outranking a store with fifty genuine 4.5-star reviews — a common ranking failure in early marketplaces.
- **`vendor_standing_factor`** — pulled directly from the standing computation defined in the Order Lifecycle doc §7 (dispute rate, refund rate, on-time fulfillment rate). A store with poor standing is pushed down even if its rating happens to be decent.
- **`recency_boost`** — a multiplier that starts elevated for a newly-verified store and **decays as it accumulates reviews**. This is what actually blends "newest" into the score itself, rather than running two separate ranking passes (one for rated stores, one for new ones) and trying to interleave the results.

**Why one formula instead of two lists:** interleaving a "top rated" list and a "newest" list is a common shortcut, but it produces an arbitrary, hard-to-explain mix. A single decaying-boost formula gives every store one score, one position, and a ranking that naturally shifts from "boosted for being new" to "earned by real reviews" as data accumulates — without needing a hard cutoff or manual rule about when a store "graduates" out of the new-store boost.

## 2. Guaranteed New-Store Visibility

The formula in §1 still risks a genuinely new store (zero reviews yet, boost not yet proven out) getting buried under established ones on a busy day. Recommendation, layered on top of the formula rather than replacing it: a **rotating "New on Cartiphy" slot** on the discovery homepage, populated by recently Tier 1+ verified stores regardless of their computed score. This is the actual cold-start guarantee — the formula's recency boost handles the gradual transition, this slot handles the "day one, zero data" problem directly.

## 3. What Discovery Actually Shows

All of Discovery — homepage, category pages, search, and store profile pages — lives at `discover.cartiphy.com`, per Level 1's app architecture.

- **Products first, marketplace-style** — the discovery homepage, category pages, and search results are primarily grids of individual products, each showing its store's name and verification badge alongside it. This matches how Nigerian buyers already expect to shop (closer to Jumia/Konga than a directory of shopfronts).
- **Store profile pages exist separately**, for a buyer who wants to see everything one vendor sells (§4).
- Ranking (§1) is computed at the **store** level (standing, rating) but applied to surface that store's **products** in listings — a well-standing store's products rank higher than an equivalent product from a poorly-standing store.

## 4. Store Profile Pages

**These live on `discover.cartiphy.com` — not on the vendor's subdomain, and not on the marketing site.** A store profile page is a Cartiphy-branded summary: logo, About text, verification badge, aggregate rating, and a grid of the vendor's top/all products. It is part of the Discovery surface (per Level 1's page inventory, section B) and the buyer discovery/account route group defined in Level 1's app architecture, styled consistently across every vendor regardless of which storefront template they picked.

**Why not just link to the vendor's subdomain storefront:** the subdomain storefront is the vendor's own templated experience (Storefront doc §1–§2), which varies in layout by template choice. Discovery needs a single, predictable, Cartiphy-owned page to browse a vendor's full catalog with consistent design — exactly the same logic that keeps checkout and the trust badge Cartiphy-owned inside every template.

**The actual purchase still happens on the vendor's own subdomain:** tapping a product on the store profile page takes the buyer through to that product's page on `vendorname.cartiphy.com`, where cart and checkout (Storefront doc) take over. Discovery never duplicates checkout — it's purely a browsing and discovery layer sitting in front of it.

## 5. Search

- **Postgres full-text search** across three fields with descending weight: product name (highest), product description, store name.
- **Relevance-first.** A search for "leather bag" should surface the most textually relevant results first; the rating/standing score from §1 only acts as a **tiebreaker** among results of similar relevance — it never overrides what the buyer actually typed.
- Category pages use the same underlying ranking as the homepage (§1), scoped to that category.
- **Typo tolerance:** exact full-text matches always rank first. When a query returns few or no full-text results (below a small threshold tuned at build time), the search falls back to trigram similarity (Postgres `pg_trgm`) on product name and store name, so a misspelling like "lether bag" still finds leather bags. A short synonym list (e.g. "sneakers"/"trainers", "phone"/"mobile") can be maintained as a settings-editable table, added when real search data shows which ones matter.
- **No dead-end "no results" page:** when nothing matches even after fuzzy fallback, show a warm-voiced message plus the category list and the "New on Cartiphy" products, so the buyer always has somewhere to go.
- Fuzzy fallback never overrides relevance: a fuzzy match ranks below any exact match.

## 6. Eligibility

- **No minimum product count.** Any Tier 1+ verified, `live`-status store is eligible for discovery the moment it meets those two conditions — even with a single product listed. This matches how low the bar already is for a single-product store to exist at all (per the Storefront doc).
- A `suspended` or `banned` store is immediately excluded from discovery and search, consistent with the enforcement ladder in Level 1 §5.

## 7. Personalization

**None at launch.** Every buyer sees the same ranking for the same query or homepage load — no recently-viewed feed, no "recommended for you" logic. This isn't a permanent limitation; it's explicitly deferred because personalization needs real behavioral data (view/cart_add/chat_tap events, already being collected per the Storefront doc §13) to be worth building, and a Phase 10 candidate once that data exists in volume.

---

## 8. Data Model Additions / Clarifications

Beyond what's already listed in Level 1 §6:
- `store_ranking_snapshots` (new) — periodic computed score per store (bayesian_rating, vendor_standing_factor, recency_boost, final score, computed_at), so ranking has an explainable history rather than only a live, opaque number — useful for admin review if a vendor ever disputes their ranking.
- `stores.new_store_boost_expires_at` — marks when a store's `recency_boost` will have fully decayed, used to compute §1's multiplier.
- Search relies on Postgres's built-in full-text search (`tsvector`/`tsquery`) on `products.name`, `products.description`, and `stores.name` — no separate search index/service needed at this scale.
- `pg_trgm` extension enabled, with trigram indexes on `products.name` and `stores.name`, for the typo-tolerant fallback in §5.
- `search_synonyms` (optional, added when needed) — term, equivalent terms, editable in the settings editor.

## 9. Checkpoint Test Script

1. A store with 50 genuine 4.5-star reviews outranks a store with a single 5-star review, confirming the Bayesian adjustment is actually working.
2. A store with poor standing (per the Order Lifecycle doc) ranks lower than a similarly-rated store with good standing.
3. A newly-verified store with zero reviews appears in the "New on Cartiphy" slot regardless of its (currently low/undefined) computed score.
4. A search for a specific product term returns that product as the top result even when a higher-rated but less textually relevant product exists — confirming relevance beats rating.
5. A single-product store with no reviews yet is still visible in its category page once Tier 1+ verified and live.
6. A `suspended` store's products and store profile page immediately disappear from discovery, search, and category pages.
7. Tapping a product on a store's Discovery profile page correctly routes to that product's real page on the vendor's own subdomain, where checkout takes over.
8. A search for a deliberately misspelled product name ("lether bag") returns the correct product through the fuzzy fallback, and an exact-spelling search still ranks exact matches first.
9. A search with no possible match shows the warm "no results" page with categories and "New on Cartiphy" products, not an empty page.
10. A fuzzy-matched result never outranks an exact match for the same query.

---

*This document supersedes any earlier informal notes on discovery and ranking. Read alongside `cartiphy-level-1-master-plan.md` and the other Level 2 documents.*
