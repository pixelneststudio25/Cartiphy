# Cartiphy — Level 2: Vendor Content Tools

Reads against `cartiphy-level-1-master-plan.md`, `cartiphy-level2-storefront-cart.md`, `cartiphy-level2-template-library.md`, and `cartiphy-level2-admin-panel.md`. This is the contract for four vendor-facing content systems: link previews and SEO, AI-assisted copy, the Cartiphy Image Library, and image scanning.

Items marked **(proposed)** await confirmation; everything else is locked.

---

## 1. Link Previews (Open Graph), Favicons and SEO

### 1.1 OG image spec (locked)

- **1200×630px (1.91:1), hard cap 300KB.** This replaces the earlier 1900×1400 / 200KB idea. WhatsApp's own guidance uses 1200×630, and oversized images can cause WhatsApp to drop the preview entirely, so 300KB is the ceiling.
- Every OG image is served through Cloudinary at exactly this size, with automatic format and quality to stay under the cap.

### 1.2 Library plus dynamic compositing (no AI, no extra AI cost)

- **OG background library:** Cartiphy provides a set of on-brand OG backgrounds (correct size, each under 300KB). During store setup a vendor **picks one from the library or uploads their own**; uploads are validated for size, dimensions and format, and scanned per §4.
- **Dynamic product previews:** for a product page, Cloudinary's rule-based transformations layer the product's own photo, name, price and a small Cartiphy mark over the vendor's chosen background, output at 1200×630 and cached at the CDN edge. This is deterministic image compositing, billed under Cloudinary's ordinary transformation and bandwidth usage — it involves no AI and no credits.
- Store-level pages (homepage) use the chosen background with the store name and logo.
- Fallback when a vendor has chosen nothing: a default Cartiphy background.

### 1.3 Favicons

- Default for every store: the Cartiphy ring-mark favicon.
- If the vendor has uploaded a logo, a favicon is derived from it via Cloudinary transformation.
- `discover.`, `vendor.`, `admin.` and the marketing site always use the plain Cartiphy mark.

### 1.4 SEO

- **Structured data:** JSON-LD `Product` and `Offer` on every product page; `AggregateRating` only when the product or store has real verified reviews (never fabricated or empty).
- **Sitemap:** dynamically generated `sitemap.xml` covering every `live` store and its products; suspended, banned, closed and pre-live stores are excluded.
- **`robots.txt`:** generated per host; blocks admin and vendor dashboard surfaces.
- **Canonical rule:** the product page on the vendor's subdomain is the canonical URL. Discovery listings on `discover.cartiphy.com` are a lighter browsing layer and point their canonical at the vendor-subdomain page where full content duplicates exist.
- **Meta titles and descriptions (no AI needed):** templated defaults, e.g. "Buy {product name} from {store name} on Cartiphy. {first ~100 characters of description}." AI only improves the underlying description text (§2), not the template.

---

## 2. AI-Assisted Copy Generation

### 2.1 What it does

A **"Generate with AI"** button beside the About text, product description, product details, and tagline fields. The vendor supplies a few facts (or the system uses the product's existing name, category and price) and gets a first draft they can edit or regenerate. **Nothing is ever auto-published** — the vendor always sees and saves the text themselves.

### 2.2 Model and integration (locked)

- **DeepSeek V4 Flash via OpenRouter.** Chosen to keep cost near zero; short marketing copy does not need a frontier model.
- Set the OpenRouter routing mode explicitly (balanced price/speed) rather than relying on the default, since several providers serve this model at different prices.
- **Server-side only.** The OpenRouter key lives in server environment variables and is never exposed to the browser — same security posture as payment keys.
- Because the model is newer and less battle-tested, spot-check real output quality once wired up before trusting it unsupervised.
- Pricing and model availability shift; re-verify at implementation time.

### 2.3 Guardrails

- **Content-scanner runs on AI output too.** Generated text must pass the same scan (phone numbers, bank details, off-platform instructions) as vendor-typed text before it can be saved.
- The system prompt instructs the model never to include contact details, prices it was not given, health or performance claims, or guarantees. Output is always treated as a draft.
- Output language: English at launch.
- Per-vendor rate limiting (per minute and per day) independent of plan caps, to prevent abuse.
- Generation requests and outcomes are logged (`ai_generation_log`: store, field type, tokens, status, timestamp) for cost monitoring and abuse review.

### 2.4 Plan gating

Uses the existing entitlements pattern (`plan_entitlements`), never scattered plan checks. **(proposed)** A monthly generation allowance per plan, larger on higher tiers — Prime a small allowance, Venture generous, Apex generous or effectively unlimited. Exact numbers are an open item (§7); this becomes a listed benefit in the plan comparison. When a vendor reaches their allowance, the button shows a calm message with the reset date and the next plan's allowance, never a manipulative modal.

---

## 3. Cartiphy Image Library

### 3.1 Purpose and sources

A curated set of images vendors can pick from for banners, backgrounds, lifestyle shots and placeholders when they have no photos of their own.

- **Sources (locked): Unsplash and Pexels only.** The owner downloads, curates and sorts them.
- **Pinterest is not an image source** — pins are not licensed for redistribution. Pinterest remains fine for design inspiration only (Template Library §11).
- **Before loading anything, re-read the current Unsplash and Pexels license terms** for the specific use (images offered inside a product for vendors to use on their own storefronts) and keep a record of the terms as checked, with the date. Do not bulk-import images that depict identifiable people in ways that imply endorsement, or trademarked/branded products.
- Per-image source record: `source` (unsplash / pexels), `source_url`, `creator_name`, `license_checked_at`. Kept even where attribution is not required, as an audit trail.

### 3.2 Storage and structure

- Cloudinary folders: vendor uploads live in `vendors/{vendor_id}/products/`; the library lives separately in `cartiphy-library/{category}/`.
- Each library image is **tagged** (not only foldered) by category, use-case (banner, lifestyle, placeholder, og-background) and mood, so the vendor-facing picker can filter cleanly.
- Library images are pre-optimized; OG backgrounds additionally conform to §1.1.

### 3.3 Vendor experience

- The picker appears wherever an image slot exists (banner, OG background, placeholder). The vendor can filter by category and use-case.
- A library image chosen by a vendor is referenced, not copied into their folder, so a library image can be retired without breaking history only if the vendor is shown a replacement prompt; retiring an image in use is blocked unless replaced (proposed).
- Library images do not count toward a plan's per-product image cap (they are store-level design assets); proposed, to be confirmed.

---

## 4. Image Scanning (Anti-Circumvention)

Closes the gap where a vendor puts a phone number or "DM to buy" in an uploaded image instead of text.

- **Scope:** product images, store banner, logo, and uploaded OG backgrounds.
- **Method:** OCR on upload, looking for the same patterns the text content-scanner detects (phone numbers, bank details, off-platform instructions). Start lighter than the text scanner to limit false positives.
- **Behavior:** unlike text (where the save is blocked with a friendly fix), image hits **flag rather than block**, because OCR is noisier. A hit creates a `risk_flag` case in the admin queue (Admin doc §2) and counts as a content-scanner signal in the nightly risk job. Repeated or high-confidence hits can additionally trigger the friendly-block treatment; threshold is tuned with data.
- **Runs asynchronously** after upload so it never slows the vendor's flow.
- **Tooling** (Cloudinary add-on, self-hosted OCR, or a cloud vision API) is a Level 3 implementation choice, selected on cost per image at expected volumes.
- Hits and scan results are stored with the image record so admin can review the flagged image alongside the match.

---

## 5. Data Model Additions

- `stores.og_background_id` (or library image reference / uploaded asset), `stores.favicon_url`.
- `library_images`: id, cloudinary_public_id, category, use_case, tags[], source, source_url, creator_name, license_checked_at, is_active.
- `ai_generation_log`: store_id, field_type, model, tokens_in, tokens_out, status, created_at.
- `plan_entitlements` rows: monthly AI generation allowance per plan.
- `image_scan_results`: image reference, scan_status, matched_patterns[], confidence, flagged_case_id, scanned_at.
- `products.details`, described in Storefront doc §15, is one of the fields the AI tool can draft.

---

## 6. Checkpoint Test Script

1. A product link pasted into WhatsApp shows a 1200×630 preview with the product photo, name and price over the vendor's chosen background, loading under 300KB.
2. A store with no chosen OG background falls back to the default Cartiphy background.
3. A vendor logo produces a derived favicon; a store without one shows the Cartiphy mark.
4. A product page outputs valid `Product`/`Offer` JSON-LD; `AggregateRating` appears only once real reviews exist. A suspended store's pages disappear from the sitemap.
5. The canonical URL of a product is its vendor-subdomain page.
6. "Generate with AI" drafts text that the vendor can edit; nothing saves until they confirm. The key never appears in any browser-visible response.
7. AI output containing a phone number is blocked by the content-scanner exactly as typed text would be.
8. A vendor exceeding their plan's AI allowance sees the calm limit message; higher plans have a higher allowance driven purely by `plan_entitlements`.
9. An uploaded product image containing a phone number creates a risk-flag case without blocking the upload.
10. A library image picker filters correctly by category and use-case, and a chosen image does not count toward the product image cap (if confirmed).

---

## 7. Open Items

- Exact monthly AI allowances per plan.
- Whether library images count toward image caps.
- OCR tooling choice and initial confidence thresholds (Level 3).
- Policy when a library image in use needs to be retired.
- Initial OG background set: how many and which styles.

---

*This document supersedes any earlier informal notes on OG images, AI copy, the image library, and image scanning. Read alongside `cartiphy-level-1-master-plan.md` and the other Level 2 documents.*
