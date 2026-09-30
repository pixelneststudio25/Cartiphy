# Cartiphy — Level 2: Order Lifecycle and Trust Mechanics

Reads against `cartiphy-level-1-master-plan.md` and `cartiphy-level2-payments.md`. This is the contract for Phase 4's status machine, confirmation flow, disputes, reviews, and vendor standing. Refunds themselves (the money-movement mechanics) are fully specified in the Payments doc — this document defines what *triggers* a refund from the order side and how disputes are tracked as cases.

---

## 1. Order Status Machine

**States:** `Placed → Confirmed → Out for Delivery → Delivered → Cancelled`

(Note: "Disputed" is **not** a status on the order itself — see §4. A dispute is tracked as a case while the order stays `Delivered`.)

| Status | Meaning | Who moves it, and how |
|---|---|---|
| **Placed** | Payment verified, order created (per Payments doc §3–4). This is the starting state — it is never skipped. | System, automatically, the moment the webhook confirms payment. |
| **Confirmed** | Vendor has actively acknowledged the order and will fulfill it. | **Vendor, manually** — an explicit "Accept order" action in the vendor dashboard. This is not automatic. |
| **Out for Delivery** | Vendor has dispatched the item (off-platform delivery — no courier integration at launch). | Vendor, manually. |
| **Delivered** | Item has arrived. This is also the terminal, "complete" state — orders that go through dispute resolution stay in this state; they don't move to a separate status. | Reached either by the buyer confirming (via the tokenized link or their account) or by the 72-hour auto-confirm timeout, both triggered by the vendor first marking the item delivered (see §2). |
| **Cancelled** | Order will not be fulfilled. | Buyer (before "Out for Delivery") or vendor (any time before shipping) — see §5. |

**Why "Confirmed" is a manual vendor step:** it's your first real signal that a vendor is actually paying attention to new orders, and it's what makes the fulfillment SLA in §3 meaningful — you can't measure "did the vendor respond" if acknowledgment is automatic.

---

## 2. The Delivery Confirmation Flow

This is the sequence from "vendor ships" to "order is truly complete."

1. Vendor marks **Out for Delivery**.
2. Vendor marks **Delivered** once the item has actually arrived (this is a separate, deliberate action — it is not triggered by marking "Out for Delivery").
3. The moment "Delivered" is marked:
   - A **tokenized, no-login confirmation link** is sent to the buyer by SMS and/or email.
   - A **72-hour auto-confirm timer** starts.
4. The buyer can confirm receipt two ways, both equally valid:
   - Tapping the tokenized link (works for guests and registered buyers alike).
   - Confirming directly from their order history if they're logged in.
5. **If the buyer confirms (either way):** order status is `Delivered` (already was), the 7-day dispute window starts from this moment, and the review prompt fires immediately (§6).
6. **If 72 hours pass with no buyer action:** the system auto-confirms on the buyer's behalf. Same downstream effects — dispute window starts, review prompt fires. This prevents an order from being permanently stuck if a buyer simply never checks the link.

**Why the clock starts at "vendor marks Delivered," not "Out for Delivery":** starting it at dispatch would risk auto-confirming an order that's still in transit, before the buyer has even received it.

---

## 3. Vendor Fulfillment SLA

Not previously defined at Level 1 — this closes that gap.

- If a vendor has not moved an order to **Out for Delivery** within **5 days** of confirming it, the order **auto-flags** into the admin queue for visibility.
- The buyer is notified at the same time and given a direct option to request a refund, rather than being left wondering what's happening.
- This SLA breach is one of the inputs to vendor standing (§7) — a vendor who repeatedly misses it should show up as lower-standing even before any dispute is ever raised.

---

## 4. Disputes (Tracked as Cases, Not an Order Status)

**Why disputes don't change order status:** the 7-day dispute window opens *after* delivery is confirmed — a dispute is something that happens to an order that has already completed its lifecycle, not a mid-flight state. Keeping the order at `Delivered` and layering a case on top of it (reusing the exact case structure defined in the Payments doc) means the order record stays a clean, permanent history of what actually happened, while the dispute's back-and-forth lives in its own case timeline.

**Flow:**
1. Buyer raises a dispute within 7 days of delivery confirmation (manual or auto), from their order history or the confirmation link page. Reason required (non-delivery, item not as described, damaged, wrong item, other).
2. A case opens, linking the order, buyer, vendor, and reason — identical structure to a payments case.
3. Vendor gets the same 48-hour response window as in the Payments doc (accept, contest with evidence, or stay silent).
4. Admin reviews and decides. **Outcome is refund or reject — there is no reship/replacement option at launch.** Reshipping would require tracking and proof-of-redelivery mechanisms that don't exist yet; keeping it to a binary refund/reject keeps the whole system consistent with the full-refund-only decision already locked.
5. If a dispute is not resolved within **5 days** of being raised, it auto-flags for priority admin attention (this was already specified in Level 1 — this document just confirms it plugs into the same case object, not a separate escalation path).
6. Once resolved (refund processed per the Payments doc, or rejected), the case closes. The order itself never leaves `Delivered`.

---

## 5. Cancellation

| Who | Window | Result |
|---|---|---|
| Buyer | Any time before the vendor marks **Out for Delivery** | Order moves to `Cancelled`. Since this happens early, it will almost always fall into the Payments doc's clean "not yet settled" refund path — no debt ledger involvement expected in the normal case. |
| Vendor | Any time before marking **Out for Delivery** | Same outcome — `Cancelled`, refund triggered. |

Once an order is `Out for Delivery`, cancellation is no longer available to either party — from that point, a problem is handled as a dispute (§4) after delivery, not a cancellation.

---

## 6. Reviews

- **One review per order** (not per product), even if the order contains multiple items — keeps the model simple and matches the fact that products don't have variants either.
- **Only eligible on verified purchases** — an order that has reached `Delivered` (manually or auto-confirmed) via an on-platform payment. An order that was refunded or cancelled is not eligible.
- **Prompt timing:** sent immediately when the order reaches confirmed-`Delivered` status (§2, step 5 or 6) — not delayed, so the experience is still fresh for the buyer.
- **Vendor response:** vendors can publicly respond to a review once, visible alongside it on their store profile — gives them a voice without opening a back-and-forth thread.
- **Removal:** per the admin panel plan, a review can only be removed by an admin, with a required reason (fake or abusive content), and the action is logged.
- **Rating aggregation:** store-level rating is the average of all its order reviews; this feeds directly into the discovery ranking already defined at Level 1 (verified + rated, blended with newest as cold-start fill).

---

## 7. Vendor Standing

Not previously defined at Level 1 beyond being mentioned — this section defines its inputs.

**Standing is a blend of:**
- Dispute rate (disputes raised ÷ completed orders)
- Refund rate (refunds issued ÷ completed orders)
- On-time fulfillment rate (orders shipped within the 5-day SLA ÷ total confirmed orders)

**How it's used:**
- Visible to **admins only** at launch — not shown publicly to buyers as a score. Showing an explicit number to buyers before you have enough data to make it meaningful risks being both inaccurate and reputationally risky for vendors.
- Factored lightly into **discovery ranking** alongside the existing verified+rated+newest blend — a vendor with a rough patch shouldn't vanish from discovery overnight, but a consistently poor standing should push them down.
- A low standing is one of the inputs the nightly risk-scoring job (from Level 1 §5) can use alongside the existing signals (content-scanner hits, buyer reports, chat-tap ratio).

**Not decided here, left for tuning once real data exists:** the exact weighting between the three inputs, and the specific threshold at which standing meaningfully affects ranking. Flagging this explicitly rather than inventing numbers now.

---

## 8. Data Model Additions / Clarifications

Beyond what's already listed in Level 1 §6:
- `orders.status` — enum: `placed, confirmed, out_for_delivery, delivered, cancelled` (no `disputed` value — disputes live in `cases`).
- `orders.confirmed_at`, `orders.shipped_at`, `orders.delivered_marked_at`, `orders.delivery_confirmed_at` — explicit timestamps for each transition, needed to compute the SLA (§3) and the dispute/auto-confirm windows (§2, §4).
- `orders.delivery_confirmation_method` — `buyer_link`, `buyer_account`, or `auto_timeout`, so it's always clear how an order reached confirmed-delivered.
- `cases` (already defined in Payments doc) — used for disputes exactly as it's used for refunds; no new table needed.
- `reviews.vendor_response` — nullable text field, single response only.
- `vendor_standing_snapshots` (new) — periodic computed snapshot per vendor (dispute rate, refund rate, on-time rate, computed score, computed_at) so standing has a history rather than only a live number, useful for both admin review and future tuning.

---

## 9. Notification Triggers (Phase 5 Dependency)

This document defines *when* an event happens; Phase 5 defines the actual message content. For reference, the triggers introduced or clarified here:
- Vendor accepts order (Confirmed) — optional buyer notice
- Vendor marks Out for Delivery
- Vendor marks Delivered → buyer confirmation link (SMS + email)
- 72h auto-confirm fires
- 5-day fulfillment SLA breach → buyer notified with refund-request option; admin flagged
- Dispute case opened / vendor response window closing / case resolved
- Review prompt sent

---

## 10. Checkpoint Test Script

1. Order is Placed automatically on payment confirmation; vendor sees it and manually confirms it.
2. Vendor marks Out for Delivery, then Delivered — confirmation link fires, 72h timer starts.
3. Buyer confirms via the tokenized link — order stays Delivered, dispute window opens, review prompt fires.
4. A second simulated order — buyer takes no action for 72 hours — auto-confirms correctly with the same downstream effects.
5. A vendor fails to ship within 5 days of confirming — order auto-flags, buyer gets a refund-request option.
6. A buyer cancels before "Out for Delivery" — order moves to Cancelled, refund triggers via the Payments doc's clean path.
7. A dispute is raised within the 7-day window — case opens, vendor responds within 48h, admin resolves as a refund — order itself remains `Delivered` throughout.
8. A dispute is left unresolved for 5 days — auto-flags for priority admin attention.
9. A completed order is reviewed once; a second review attempt on the same order is rejected; vendor posts one public response.
10. Vendor standing snapshot computes correctly after a mix of on-time, late, disputed, and clean orders.

---

*This document supersedes any earlier informal notes on order status and disputes. Read alongside `cartiphy-level-1-master-plan.md` and `cartiphy-level2-payments.md`. Next Level 2 documents: Storefront and Cart, then Admin Panel detail.*
