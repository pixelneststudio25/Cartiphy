# Cartiphy — Level 2: Notifications

Reads against `cartiphy-level-1-master-plan.md`, `cartiphy-level2-payments.md`, `cartiphy-level2-order-lifecycle.md`, `cartiphy-level2-storefront-cart.md`, and `cartiphy-level2-admin-panel.md`. This is the contract for Phase 5's notification system. Exact message copy/templates are a Level 3 task — this document defines channel, timing, recipient, and tone rules for every trigger already established across the other Level 2 documents.

---

## 1. Global Rules

- **Language: English only at launch.** Pidgin or other localization is a real content-production effort, deferred until there's evidence it's needed.
- **Channel: SMS only for now.** Termii is used for SMS; WhatsApp Business API approval is still an open external task (Level 1 §12). Once approved, transactional notifications can migrate to WhatsApp (cheaper, richer formatting), with SMS retained as the automatic fallback if a WhatsApp message fails to deliver.
- **Email is always the formal, detailed record.** SMS is always short, urgent-only, and links back to the dashboard or confirmation page for detail — SMS never carries a full explanation, a dispute reason, or sensitive detail.
- **Every SMS template targets under 160 characters** (one billing segment) — a concrete way to hold the line on SMS cost, which scales per segment.
- **Delivery handling:** every send attempt is logged to `notification_log` with its status. On failure, **one automatic retry**; if that also fails, it surfaces as a visible failure in the admin-lite queue (and later, the health dashboard) rather than failing silently.
- **No opt-out at launch.** Every notification defined below is transactional (order, security, or account-status related), not marketing — there's nothing here a user should be able to silence.
- **No batching or digests at launch.** Everything sends immediately on its trigger.

---

## 2. Authentication and Security

| Trigger | Recipient | Channel | Timing |
|---|---|---|---|
| Phone OTP (login or verification) | Vendor or buyer | SMS | Immediate |
| Vendor signup / welcome | Vendor | Email | Immediate |
| Bank account change initiated | Vendor | Email + SMS | Immediate (per Payments doc §7 — this is a security alert, not a routine confirmation) |

---

## 3. Orders (per the Order Lifecycle doc)

| Trigger | Recipient | Channel | Timing |
|---|---|---|---|
| Order placed (payment confirmed) | Buyer | Email (receipt) + SMS (short confirmation) | Immediate |
| Order placed | Vendor | Email + SMS ("You have a new order") | Immediate |
| Vendor accepts order (Confirmed) | Buyer | SMS (optional, light-touch) | Immediate |
| Vendor marks Out for Delivery | Buyer | SMS | Immediate |
| Vendor marks Delivered → confirmation link | Buyer | SMS + Email (the tokenized link itself) | Immediate |
| 72h auto-confirm fires | Buyer | SMS (informational) | At the 72h mark |
| 72h auto-confirm fires | Vendor | Email (order now finalized) | At the 72h mark |
| 5-day fulfillment SLA breach | Buyer | SMS + Email (refund-request option) | At breach |
| 5-day fulfillment SLA breach | Admin | Internal admin-queue flag only — no SMS/email to admin | At breach |
| Review prompt | Buyer | SMS + Email | Immediately on delivery confirmation (manual or auto) |
| Vendor responds to a review | Buyer (reviewer) | Email (light-touch) | Immediate |

---

## 4. Disputes and Cases (per the Order Lifecycle and Admin Panel docs)

| Trigger | Recipient | Channel | Timing |
|---|---|---|---|
| Dispute case opened | Vendor | Email + SMS (short alert, response window stated) | Immediate |
| Vendor response window closing (e.g., 12h left) | Vendor | SMS reminder | Before deadline |
| Dispute case resolved (refund) | Buyer | Email | On resolution |
| Dispute case resolved (rejected) | Buyer | Email (reason stated) | On resolution |
| Dispute unresolved after 5 days | Admin | Internal admin-queue priority flag only | At the 5-day mark |

---

## 5. Refunds and Chargebacks (per the Payments doc)

| Trigger | Recipient | Channel | Timing |
|---|---|---|---|
| Refund approved and processed | Buyer | Email (confirmation, expected timing) + SMS if the payment method involves a longer wait (e.g., bank transfer) | Immediate |
| Refund creates a vendor debt | Vendor | Email (amount, reason, recovery explanation — this is a formal record) | Immediate |
| Each recovery deduction from a future settlement | Vendor | Email (or a periodic summary if deductions are frequent) + visible line in the dashboard earnings view | On each deduction |
| Debt escalates to manual outreach (14-day/size threshold) | Vendor | Email (a direct, human-toned message, not templated boilerplate) | At escalation |
| Debt written off | — | Internal only (audit log), no notification needed | — |
| Chargeback opened | Vendor | Internal admin-queue visibility; no direct vendor SMS/email at this stage, since it's between Cartiphy and Flutterwave initially | On notice |
| Chargeback resolved (lost, debt created) | Vendor | Follows the same refund-creates-a-debt template above | On resolution |

---

## 6. Verification (per the Admin Panel doc)

| Trigger | Recipient | Channel | Timing |
|---|---|---|---|
| Tier 1 auto-approved | Vendor | Email + SMS ("You're verified — you can now sell") | Immediate |
| Tier 1 manual review — approved | Vendor | Email + SMS | On admin decision |
| Tier 1 manual review — rejected | Vendor | Email (what to fix) | On admin decision |
| Tier 2 (CAC) approved | Vendor | Email + SMS ("Verified Business badge unlocked") | On admin decision |
| Tier 2 rejected | Vendor | Email (what to fix) | On admin decision |

---

## 7. Billing and Trial (per Level 1 §3, Phase 7)

| Trigger | Recipient | Channel | Timing |
|---|---|---|---|
| Trial started | Vendor | Email (welcome + what's included + end date) | Immediate |
| Trial ending soon (e.g., 3 days out) | Vendor | Email + SMS reminder | Ahead of expiry |
| Trial converts to paid | Vendor | Email (subscription receipt) | On conversion |
| Trial lapses to Prime | Vendor | Email (calm, honest — what changes, data is intact) | On lapse |
| Subscription payment succeeds (recurring) | Vendor | Email (receipt) | Immediate |
| Subscription payment fails | Vendor | Email + SMS (action needed) | Immediate |
| Promo code applied | Vendor | Email (confirmation) | Immediate |

---

## 8. Enforcement (per Level 1 §5 and the Admin Panel doc)

Matches the action table already defined in Level 1's admin panel section — restated here purely as the channel contract:

| Action | Channel |
|---|---|
| Educational notice / warning | Email (formal record) |
| Removed from discovery (timed) | Email |
| Store suspended | Email + SMS (with appeal link) |
| Permanent ban | Email (formal record) |
| Reinstated / unbanned | Email + SMS |
| Appeal processed | Email |
| Trial extended / plan comped | Email |
| Review removed | Email to the reviewer, where appropriate |

All enforcement SMS is short and links to the dashboard for the full reason — never carries the detailed reason itself, consistent with Level 1's existing rule that SMS never contains detailed reasons.

---

## 9. Vendor Operational Alerts

| Trigger | Recipient | Channel | Timing |
|---|---|---|---|
| Low stock on a product | Vendor | Email (or SMS if urgent — e.g., stock hits zero) | When threshold crossed |
| Abandoned cart (Venture/Apex visibility) | Vendor | **Not an automatic notification** — this is a dashboard list feature with a manual click-to-chat WhatsApp link (per Level 1 §3 and the Storefront doc §7), not something Cartiphy sends on the vendor's behalf. |

---

## 10. Data Model Additions / Clarifications

Beyond what's already listed across the other Level 2 documents:
- `notification_log.channel` — enum: `sms, email` (no `whatsapp` value until that channel is actually enabled post-launch).
- `notification_log.retry_count` — 0 or 1, enforcing the single-retry rule.
- `notification_log.status` — `sent, failed, retried_failed`.
- `message_templates` (already listed in the Admin Panel doc) — one row per trigger in the tables above, each tagged with its channel and required variables, editable via the admin settings editor once built.

---

## 11. Checkpoint Test Script

1. Every trigger table above fires exactly once per real event during a full Phase 3–5 simulated order (payment → confirm → ship → deliver → review) — cross-check against the Payments and Order Lifecycle checkpoint scripts, which already exercise these events.
2. A deliberately-failed SMS send retries exactly once, then surfaces as a visible failure — never retries indefinitely, never fails silently.
3. Every SMS template drafted at Level 3 is checked against the 160-character limit before being finalized.
4. A bank account change fires both email and SMS immediately, matching the Payments doc's security-alert requirement.
5. An enforcement suspension sends email + SMS with an appeal link, and the SMS itself contains no detailed reason — only a link to view it.
6. A vendor's subscription payment failure triggers both channels, distinct from a successful recurring payment which triggers email only.

---

*This document supersedes any earlier informal notes on notification behavior. Read alongside `cartiphy-level-1-master-plan.md` and the other Level 2 documents. This completes the current Level 2 set. Actual message copy for each row above is a Level 3 task, along with task-card drafting for Phase 0–1.*
