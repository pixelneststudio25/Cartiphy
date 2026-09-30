# Cartiphy — Level 2: Payments System

Reads against `cartiphy-level-1-master-plan.md`. This is the contract for all Phase 3–4 payment, webhook, settlement, refund, and chargeback code. If something here conflicts with Level 1, Level 1 wins and this doc needs correcting. If something isn't covered here, ask before inventing it — do not guess at API behavior, field names, or edge cases.

---

## 1. Sub-Account Lifecycle

**Trigger:** A vendor's Tier 1 verification (bank account name-match + phone OTP) passes.

**Flow:**
1. Vendor submits bank account number + bank in onboarding.
2. Cartiphy calls Flutterwave's account resolution/name-match to confirm the account name matches the vendor's registered name (allowing for reasonable variation — married names, abbreviations — flagged for manual admin review rather than auto-rejected on partial mismatch).
3. On match, Cartiphy creates a Flutterwave sub-account for the vendor, storing the sub-account reference on `vendors`.
4. `vendors.verification_tier` moves to Tier 1; the vendor can now accept live payments and appears in discovery.

**Failure handling:**
- Name mismatch below the acceptable threshold → vendor sees a clear in-app message to correct their details; does not silently retry.
- Flutterwave API error during sub-account creation → vendor stays at Tier 0, retried automatically once, then surfaced to admin-lite's queue if it fails twice.
- No sub-account, no live payments — this is enforced server-side at checkout time, never assumed from UI state alone.

---

## 2. Checkout → Charge Flow

**Integration method:** Inline (Flutterwave's embedded/in-page checkout), so the buyer never leaves the Cartiphy-hosted storefront. This matters for brand trust and for keeping the buyer inside the store's own visual identity through payment.

**Payment methods enabled:** Card and bank transfer. (USSD deferred — can be enabled later via the same integration with no architecture change.)

**Sequence:**
1. Buyer reaches checkout with a server-side cart (per Level 1 §4).
2. Server calculates: item subtotal + vendor delivery fee = checkout total.
3. Server calculates the commission split **from `plan_entitlements`, read live at charge time** — never from a cached or hardcoded rate. Commission is computed on **item subtotal only**, excluding the delivery fee.
4. Server initiates a Flutterwave charge with a **dynamic transaction-level split** (not a static ratio baked into the sub-account): X% (or the vendor's current plan rate) to Cartiphy's main account, remainder to the vendor's sub-account, from the item subtotal; the delivery fee routes to the vendor in full.
5. Buyer completes payment inline (card or transfer).
6. Server does **not** create the order yet — it waits for the verified webhook (§3). The inline redirect/success screen is a "processing" state, not a confirmation.

**Why dynamic splits over static:** a vendor's commission rate can change (trial → paid, plan upgrade/downgrade, a future per-vendor override) without ever touching their sub-account configuration. The split is computed correctly for every single charge regardless of when their plan last changed.

---

## 3. Webhook Handling

**Event types tracked:**
- Charge completed (successful)
- Charge failed
- Transfer/settlement events (if exposed by Flutterwave for sub-accounts — used for §6)
- Chargeback notice (see §8)

**Verification:** every incoming webhook is checked against Flutterwave's signature header before any processing. Unsigned or invalid-signature payloads are logged and discarded, never processed.

**Idempotency:** every webhook is written to `webhook_events` keyed by Flutterwave's event/transaction reference before any side effect runs. If an event with the same reference already exists and was already processed, the handler exits without repeating order creation, stock decrement, or notifications. This is what makes duplicate webhook delivery (Flutterwave's own retry behavior) harmless.

**On a verified "charge completed" event (first time seen):**
1. Look up the pending cart/checkout session by reference.
2. Re-verify stock is still available (see §4's concurrency handling).
3. Create `orders` + `order_items` + decrement stock, atomically.
4. Record the `payments` row with the split breakdown.
5. Fire the "order confirmed" notification (Phase 5 scope, referenced here as a dependency).

**On a verified "charge failed" event:** mark the checkout session failed, release any soft-held stock, no order created, no debt logic involved.

**Timeout / no webhook arrives:** if a checkout session has been "processing" for more than ~5 minutes with no webhook received, the system automatically calls Flutterwave's verify-transaction endpoint to check the real status and reconciles based on the result. Only if that verification call **also** fails or is inconclusive does it surface to the admin-lite queue for manual review. The buyer should never be left in "processing" indefinitely without an automatic resolution attempt.

---

## 4. Order Creation and Stock Decrement

**Concurrency (last-item race):** stock availability is checked immediately before initiating the charge (§2, step 4) — this is the first line of defense and prevents most collisions. As a second line of defense, the atomic transaction in the webhook handler (§3) re-checks stock at the moment of order creation and rejects if insufficient, inside the same transaction as the decrement, so two simultaneous "successful" payments can never both create an order against one remaining unit.

**If a payment succeeds but stock is gone by confirmation time** (the buyer who "lost" the race): the system automatically triggers a full refund for that transaction and sends the buyer an apologetic notification explaining what happened. A buyer must never end up charged with no order and no explanation.

**Transaction shape:** `orders` insert, `order_items` insert, `products.stock` decrement, and `payments` insert happen in a single database transaction. Partial failure (e.g., stock decrement fails) rolls back the whole thing — an order is never created without a matching stock decrement, and vice versa.

---

## 5. Error and Edge Cases

| Case | Handling |
|---|---|
| Buyer closes tab mid-payment | Webhook still arrives independently; order is created on webhook receipt regardless of whether the buyer is still on the page. Buyer sees status via order confirmation email/SMS, not just the browser session. |
| Webhook arrives before the browser redirect | Handled — order creation is driven entirely by the webhook, never by the redirect/return URL. The return URL only triggers a status *check*, never a status *write*. |
| Declined payment | No order, no retry loop stored server-side; buyer can simply try again from checkout. |
| Expired checkout session | Cart persists (server-side, per Level 1); buyer restarts checkout, a fresh charge is initiated. |
| Duplicate order from a resubmitted form | Prevented by keying checkout sessions to a single-use reference; a second submission from the same session is rejected, not re-processed. |

---

## 6. Settlement and Dashboard Display

Per Level 1 §4: **there is no vendor withdrawal action and no Cartiphy-held balance.** This section defines exactly what the vendor sees instead.

- **Earnings history view:** derived entirely from `orders` + `payments` (amount, date, order reference, split breakdown). This is a read-only ledger of what has been earned and paid out to the vendor's bank by Flutterwave — not a wallet Cartiphy controls.
- **Settlement status per order:** if Flutterwave's API exposes settlement state for a sub-account transaction, display it (e.g., "Settled" / "Pending settlement"). If that data isn't reliably available, default to showing orders as "Payment confirmed" without claiming a settlement status Cartiphy can't actually verify — never guess or fabricate a settlement timestamp.
- **Recovery line (when relevant):** if a vendor has an active entry in `refund_debts` (§8), their earnings view shows a transparent "amount being recovered — refund case #X" line each time a deduction happens. This is never a silent deduction.

---

## 7. Bank Account Change Flow

Per Level 1 §2, this is treated as a security event, not a routine settings edit.

1. Vendor submits a new bank account in their dashboard.
2. System re-runs the same name-match verification used at initial Tier 1 verification.
3. On match: the change is logged in an append-only fashion (old account masked, new account, timestamp, who initiated it) — never a silent overwrite of the existing record.
4. Vendor is notified immediately by **email and SMS**: "Your payout account was changed. If this wasn't you, contact support immediately."
5. A **24–48 hour hold** is applied: any settlement that would otherwise go to the new account during that window is held, giving the vendor a real window to notice and report an unauthorized change before money moves.
6. After the hold expires with no dispute raised, future settlements route to the new account normally.

---

## 8. Refunds and Chargebacks

### 8.1 What triggers a refund

| Scenario | Who initiates | Notes |
|---|---|---|
| Buyer reports non-delivery | Buyer, via order history or the tokenized confirmation page | Must fall within the 7-day dispute window from delivery confirmation (or from expected delivery, if never confirmed) |
| Buyer reports item not as described / damaged / wrong item | Buyer | Same 7-day window |
| Vendor voluntarily agrees to refund (own error, wants to cancel before shipment) | Vendor, via vendor dashboard | Still routes through admin approval — never auto-executes, even when the vendor initiates it |
| Order cancelled before the vendor confirms/ships | Vendor or buyer | Lower-risk case (very unlikely to have settled yet) but still passes through the same admin-approval step — no automated exception is carved out, per your explicit decision |
| Admin-initiated refund following an enforcement action | Admin | E.g., a vendor is suspended for fraud and outstanding orders are refunded directly by Cartiphy |
| External chargeback from the buyer's bank/card network | Buyer's bank, via Flutterwave | Different path — see §8.4 |

### 8.2 The end-to-end refund flow

**Step 1 — Case opens.** Any trigger above creates a case linking the order, buyer, vendor, and the reason given. This reuses the `cases` construct from the admin panel plan rather than inventing a separate refund-specific object.

**Step 2 — Vendor response window.** For buyer-initiated cases, the vendor gets a short window (recommend 48 hours) to respond in their dashboard — accept the refund, contest it with evidence (e.g., proof of delivery), or stay silent. Silence is treated as no objection once the window closes.

**Step 3 — Admin review.** The admin panel shows the full case: order and payment details, buyer's reported reason and any evidence, vendor's response (if any), and prior history for that vendor if relevant. This is a manual review, not an automated approval, matching your explicit decision.

**Step 4 — Admin decision (two-step confirmation, as with any money-related admin action).**
- **Denied:** buyer is notified by email with the reason; case closes; buyer retains the right to raise it again only if new evidence exists (no repeated re-litigation of the same claim).
- **Approved:** proceed to Step 5.

**Step 5 — Settlement-status check (this is the core of making it flawless).** The system checks whether the vendor's share of this specific order has already settled to their bank.
- **Not yet settled:** Flutterwave reverses the full transaction — buyer's payment, vendor's share, and Cartiphy's commission all unwind together. No debt is created. This is the preferred, clean outcome, and it's *why* keeping the refund window tight (bounded by the 7-day dispute window) matters — it maximizes how often this branch applies.
- **Already settled:** Cartiphy refunds the buyer in full from its own balance. A `refund_debts` entry is created for the vendor's share only (the commission was never disbursed, so it isn't part of the debt).

**Step 6 — Notifications fire immediately.**
- **Buyer:** email confirming the refund was processed and roughly when to expect it back on their payment method, plus a status update visible in their order history. SMS only if it's a payment-method-specific delay worth flagging (e.g., bank transfer refunds can take longer than card refunds).
- **Vendor:** email explaining the case outcome and, if a debt was created, explicitly stating the amount, the reason, and that it will be recovered from future settlements — never a surprise deduction with no explanation.

**Step 7 — Recovery (only if a debt exists).**
- On each future settlement for that vendor, the system automatically deducts up to a **capped percentage (recommend 50%)** of that settlement toward the debt — never the full settlement, so one unrelated sale isn't wiped out.
- Each deduction is visible in the vendor's earnings view (§6) with the case reference.
- If the debt remains outstanding past **14 days** or exceeds a defined size threshold, it automatically escalates to a case for manual outreach (the "conversation" fallback) handled by Support or Finance roles from the admin panel.
- If a debt is ultimately unrecoverable (vendor inactive, disputes it, etc.), an admin can write it off with a required reason — logged permanently, and the unresolved pattern feeds into the vendor's risk flags rather than disappearing silently.

**Step 8 — Case closes.** Once the buyer's refund is complete and any debt is either recovered or formally written off, the case is marked resolved with a full audit trail.

### 8.3 Platforms involved

- **Cartiphy admin panel** — case creation, review, approval, recovery tracking. This is where the process is orchestrated.
- **Flutterwave** — executes the actual refund/reversal and reports settlement status.
- **Resend (email)** — the formal record at every stage, for both buyer and vendor.
- **Termii (SMS)** — urgent, short status pings only (no case detail), consistent with how SMS is used everywhere else in the product.
- **Vendor and buyer dashboards** — ongoing visibility into case status and, for vendors, any active recovery.
- **WhatsApp is explicitly not part of this official flow.** Vendor contact is revealed to the buyer after payment for delivery coordination, and the two may separately message each other there, but nothing about the refund process itself runs through WhatsApp — every refund action must be traceable in Cartiphy's own system, not dependent on an off-platform conversation.

### 8.4 Chargebacks

A chargeback is a buyer disputing a **card** charge directly with their bank or card network, bypassing Cartiphy's own dispute system entirely, and arrives via a Flutterwave notification/webhook with a hard response deadline set by the card network.

- At launch, evidence submission to contest a chargeback happens directly on Flutterwave's own merchant dashboard (no custom Cartiphy UI for this yet) — but the system still opens an internal case to track it, so it isn't invisible.
- **If Cartiphy contests and wins:** no refund occurs; case closed.
- **If lost or not contested in time:** the disputed principal goes through the **same settlement-check and `refund_debts` recovery process as §8.2, Step 5 onward.**
- **The chargeback processing fee** Flutterwave charges (regardless of outcome) is absorbed by Cartiphy as a cost of doing business — not passed to the vendor, since it's a small, fixed cost and chasing it would damage vendor trust more than it's worth.
- **An unusually high chargeback rate on a specific vendor is itself a risk-flag trigger** — feeding into the same nightly risk-scoring job as other enforcement signals, since repeated chargebacks threaten Cartiphy's own standing with its payment processor, not just that vendor's account.

---

## 9. Admin-Lite Hooks (Phase 3–4, Stage A0)

The admin-lite screen introduced in Phase 3 needs to expose, at minimum:
- Read-only orders list with payment status and split breakdown.
- Read-only `webhook_events` viewer with manual retry capability.
- Read-only `refund_debts` list (once Phase 4 introduces refunds), showing outstanding amounts and recovery status per vendor.
- Case list filtered to payment-related cases (refunds, chargebacks), even before the full case UI (A1) is built — a simple list view is enough for Phase 3–4.

---

## 10. Checkpoint Test Script

A concrete, repeatable list to confirm this system actually works before moving past Phase 3–4:

1. Sandbox payment completes → webhook fires once → order created → stock decremented → vendor sees order.
2. The same webhook is delivered twice (simulate Flutterwave's retry) → no duplicate order, no double stock decrement.
3. Two simulated concurrent purchases of the last unit → exactly one order is created; the losing payment is automatically refunded with a notification sent.
4. A webhook is deliberately delayed past 5 minutes → the system auto-polls and resolves the order correctly without manual intervention.
5. A declined payment → no order, no stock change, buyer can retry cleanly.
6. A bank account change → re-verification runs, change is logged, notifications sent, and a settlement attempted inside the 24–48h hold window is correctly held.
7. A refund approved on an unsettled order → full clean reversal, no debt entry created.
8. A refund approved on an already-settled order → buyer refunded, `refund_debts` entry created correctly for the vendor's share only, vendor notified.
9. A simulated chargeback, lost → follows the same debt path as test 8; chargeback fee logged as absorbed by Cartiphy, not charged to vendor.
10. A vendor debt recovery deduction on a subsequent settlement → capped correctly, visible in the vendor's earnings view, case updates.

---

## 11. Carried Forward to Next Version (Post-Launch)

- **Guest checkout OTP verification.** Deferred for launch (friction-free checkout), but flagged as a near-term priority for the version after launch, not the indefinite backlog — revisit as soon as real fraud/chargeback data justifies the added friction.
- USSD as a third payment method.
- Custom Cartiphy UI for chargeback evidence submission (currently handled directly on Flutterwave's dashboard).

---

*This document supersedes any earlier informal notes on payments. Read alongside `cartiphy-level-1-master-plan.md` and, once written, the Level 2 Order Lifecycle and Admin documents, which both depend on the definitions here (order status machine, case structure, admin-lite scope).*
