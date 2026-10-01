# Cartiphy — Level 2: Admin Panel

Reads against `cartiphy-level-1-master-plan.md`, `cartiphy-level2-payments.md`, `cartiphy-level2-order-lifecycle.md`, and `cartiphy-level2-storefront-cart.md`. This is the contract for the admin panel's authentication, case system, roles, verification queue, settings editor, kill switches, and data-privacy handling.

---

## 1. Authentication and Security

- **2FA method: TOTP authenticator app** (Google Authenticator, Authy, or similar) — not SMS. SMS-based 2FA is vulnerable to SIM-swap attacks, and this is the highest-privilege surface in the whole product, so the small extra setup friction is worth it.
- Separate `admin.cartiphy.com` subdomain (per Level 1 §7).
- All permission checks run server-side — never trust a client-side role flag.
- **Session timeout: 30 minutes of inactivity**, auto-logout. Reasonable for a solo operator at launch without being disruptive.
- Every admin login (success and failure) is written to the audit log.

---

## 2. The Unified Case System

**One `cases` table, not four separate queues.** A `case_type` field distinguishes what kind of case it is: `refund`, `chargeback`, `dispute`, `report`, `risk_flag`, `appeal`. All share the same underlying structure: a timeline, linked records (order/vendor/buyer as relevant), notes, status, and resolution.

**Why unify rather than build separate systems per doc:** the Payments doc defines refund and chargeback cases, the Order Lifecycle doc defines dispute cases, and Level 1's trust/safety section defines reports and risk flags. Without unification, you'd end up checking four different inboxes for what is functionally the same workflow — open, review, respond, resolve. One `cases` table means one place to look, one timeline model to build once, and one audit pattern that covers everything.

**Case lifecycle (shared across all types):**
`Open → Under Review → Awaiting Response (if applicable) → Resolved / Rejected / Escalated`

- `case_notes` — internal comments, any admin role.
- Each case links back to its origin (a specific order, vendor, buyer, or flag source) and forward to its outcome (e.g., a `refund` case resolving links to the actual refund record from the Payments doc).
- A case list view can be filtered by `case_type`, so a refund-focused Finance-role admin and a moderation-focused Moderator-role admin can each see only what's relevant to them, without needing separate tables.
- **Image-scan hits** (OCR finding a phone number or off-platform instruction inside an uploaded image — see `cartiphy-level2-vendor-content-tools.md` §4) create `risk_flag` cases here, with the flagged image and matched pattern attached to the case. They flag for review rather than blocking the upload.

---

## 3. Roles and Permissions

Built from day one even with a single user, so adding staff later needs no rewrite.

| Role | Can do |
|---|---|
| **Super Admin** | Everything — all actions below, plus settings editor, role management, bans, kill switches. |
| **Finance** | Approve/reject refund and chargeback cases; view and manage the `refund_debts` ledger; view/edit subscriptions and billing (extend trial, comp a plan); generate reports. No ban/suspend authority, no settings editor access. |
| **Moderator** | Flag, warn, suspend (not ban) a store; unpublish listings; remove reviews; resolve dispute and report cases; per-store kill switch (pause a single store). No refund approval, no billing access, no global kill switch, no ban authority. |
| **Support** | View any record; add case notes; message vendors/buyers using templates; extend or reset a trial. No punitive actions, no money-related actions, no settings access. |

**Bans are Super Admin only** — the most severe, hardest-to-reverse action stays with the single highest trust level, even once other roles exist.

---

## 4. Verification Queue

- **Tier 1 (bank account name-match + phone OTP):** auto-approves the moment Flutterwave's name-match returns a clean result (per the Payments doc §1). Admin involvement is only triggered on a **partial or failed match**, which routes to the verification queue for manual review — where an admin can approve despite a reasonable name variation (married name, abbreviation) or reject with a clear reason sent to the vendor.
- **Tier 2 (CAC-verified):** always a manual review at launch. Vendor uploads their CAC document via the dashboard; an admin reviews it directly (no automated registry check at launch) and approves or rejects, awarding the "Verified Business" badge on approval.
- Every verification decision (auto or manual) is logged, including which match confidence triggered auto-approval, so the audit trail shows *why* a vendor was approved even when no human clicked anything.

---

## 5. What's Live-Editable vs. What Isn't (Yet)

Level 1 lists a long wishlist of things that could be admin-editable. Building a UI for all of it before launch is more scope than the platform needs on day one. The split:

**Gets a built editor before launch** (things you'll realistically tune often, pre- and post-launch):
- Plans, pricing, commission rates, product/image caps (`plans` / `plan_entitlements`)
- Trial length
- Dispute window, auto-confirm window, fulfillment SLA window (the timing values from the Order Lifecycle doc)
- Category list
- Reserved subdomain blocklist
- AI generation allowances per plan (`plan_entitlements` rows)
- Template library metadata (active flag, plan tier, tags) — templates themselves are code; only their metadata is editable here
- Cartiphy Image Library (add/retire/tag images, OG backgrounds)

**Edited directly via Supabase's own table editor for now** (rarer, more technical, or not yet well-understood enough to build a polished UI around):
- Content-scanner rule patterns/phrases
- Risk-flag thresholds (chat-tap ratio, minimum sample size)
- Discovery ranking weight formula

This isn't a permanent state — once real usage shows which of these actually need frequent tuning, they graduate into the proper settings editor (Admin build stage A2/A3, per Level 1 §7). Building a full UI for something you might touch twice a year isn't worth the effort now.

**All of it, regardless of editing method, is logged in `settings_history`** — who changed what, old value, new value — so "edited directly in Supabase" doesn't mean "untracked."

---

## 6. Kill Switches

| Switch | Who can pull it | Scope |
|---|---|---|
| **Global "pause all checkouts"** | Super Admin only | Every store on the platform stops accepting new payments — reserved for a severe event like a Flutterwave outage or a critical bug. |
| **Per-store pause** | Moderator and above | A single store stops accepting new orders — this is a normal enforcement tool, functionally similar to a suspension but instantly reversible and not tied to the formal enforcement ladder's notification requirements. |

Both are logged with who triggered them, when, and (for the global switch) a required reason, given its severity.

---

## 7. Data Privacy and Masking

Per Level 1 §2, formal compliance work (NDPA privacy policy, lawyer review, data-request tooling) begins after the product is live. That doesn't mean privacy hygiene waits too — some of it is cheap and worth building now regardless of legal timing:

**Built now:**
- Customer phone and email are **masked by default** in the admin panel (e.g., `080****1234`).
- **Reveal-on-click** shows the real value, but every reveal is written to the audit log — who revealed what, when, and ideally in the context of which case or lookup.
- This alone gives you a defensible, good-practice baseline even before the formal privacy policy exists.

**Deferred to the post-launch legal/compliance phase (Level 1 §12):**
- Formal data-access and deletion request handling (`data_requests` table exists in the schema per Level 1 §6, but the actual workflow and legal review of retention periods wait until the lawyer engagement begins).
- Retention period policy itself.

---

## 8. Data Model Additions / Clarifications

Beyond what's already listed in Level 1 §6 and §7:
- `cases.case_type` — enum: `refund, chargeback, dispute, report, risk_flag, appeal`.
- `admin_roles` — enum: `super_admin, finance, moderator, support`, with a straightforward permission-check function used server-side everywhere, not scattered `if role === ...` checks (mirrors the plan-entitlements pattern from Level 1 §3).
- `verification_reviews.match_confidence` — records what triggered auto-approval vs. manual review for Tier 1.
- `settings_history` — already listed in Level 1 §7; applies uniformly whether a setting was changed via the built editor or directly in Supabase.
- `data_reveal_log` (new, or a `reveal` action type within `audit_log`) — records every masked-field reveal.
- `image_scan_results` and `ai_generation_log` (defined in the Vendor Content Tools doc §5) — readable by Moderator and above for flagged-image review and AI abuse review respectively.
- `library_images` and `templates` (Template Library doc §9) — metadata editable via the settings editor.

---

## 9. Checkpoint Test Script

1. Admin login requires TOTP; a session left idle for 30+ minutes is auto-logged out.
2. A vendor with a clean Flutterwave name-match on Tier 1 auto-approves with zero admin action; a partial-match vendor correctly routes to the verification queue.
3. A Tier 2 CAC document upload appears in the manual review queue and can be approved/rejected with a logged reason.
4. A refund case, a dispute case, and a report case all appear correctly in the same case list, filterable by type — confirming the unified structure actually works rather than needing separate views.
5. A Support-role admin can view a case and add a note but cannot approve a refund or suspend a store; a Finance-role admin can approve a refund but cannot suspend a store; a Moderator can suspend a store but cannot approve a refund; only Super Admin can ban.
6. The global kill switch, pulled by Super Admin, stops checkout on every store; a per-store pause, pulled by a Moderator, stops only that one store.
7. A customer's phone number is masked by default in the admin panel; clicking reveal shows the real number and logs the reveal with admin identity and timestamp.
8. A setting changed via the built editor (e.g., trial length) and a value changed directly in Supabase (e.g., a risk threshold) both correctly appear in `settings_history`.
9. An uploaded image containing a phone number creates a `risk_flag` case with the image and matched text attached; a Moderator can review and resolve it from the normal case list.
10. Retiring a template (`is_active = false`) removes it from the vendor picker without breaking stores already using it; a change to a plan's AI allowance takes effect via `plan_entitlements` and appears in `settings_history`.

---

*This document supersedes any earlier informal notes on the admin panel. Read alongside `cartiphy-level-1-master-plan.md` and the other Level 2 documents. This completes the core Level 2 set (Payments, Order Lifecycle, Storefront/Cart, Admin Panel) — remaining Level 2 candidates from Level 1's open items include Discovery/Ranking detail and Notifications content, or the set can move to Level 3 task-card drafting for Phase 0–1.*
