# Cartiphy — Admin Panel Plan (v1)

Read with `build-plan-v2.md` and `design.md`. Status: v1, recommendations incorporated. Defaults are assumed for the open questions in section 10 unless changed.

---

## 1. Purpose

The admin panel is where Cartiphy is run day to day: verifying vendors, watching for problems, enforcing the Marketplace Rules, resolving disputes, and tuning how the platform works without a code change. It is the highest-privilege surface in the product, so it is built with the strictest security.

## 2. What will need updating as Cartiphy grows

Many things that feel like "code" will change often. Build them as editable settings with history, not hard-coded values.

| Changes over time | Why it will change | Admin control needed |
|---|---|---|
| Plans, prices, commission rates, product caps and feature entitlements | Pricing will be tested and adjusted | Plan editor (drives the `plans` / `plan_entitlements` config) |
| Promo campaigns and codes | Promo seasons, trial extensions | Campaign and code manager |
| Category list | New categories as sellers demand them | Category manager |
| Reserved subdomain words | Impersonation and abuse patterns appear | Blocklist editor |
| Content-scanner rules (phrases, patterns) | Vendors will find new ways to share contact details | Rule editor with test box |
| Risk-flag thresholds (chat-tap ratio, minimum sample size) | Need tuning against real data | Threshold settings |
| Marketplace Rules and Terms of Service versions | Legal review, new situations | Policy version publisher with forced re-acceptance |
| Email and SMS message templates | Wording, legal tone, new events | Template editor with preview and variables |
| Template library | New templates, retiring weak ones | Enable/disable templates, tag by category, assign to plans |
| Operational timings | Trial length (14 days), auto-confirm (72 hours), dispute window (7 days), flag at 5 days | Platform settings, with change history |
| SMS and WhatsApp allowances per plan | Costs and usage will show what is sustainable | Entitlement settings |
| Discovery ranking weights and eligibility | Cold-start fill matters less as data grows | Ranking settings |
| Feature flags and kill switches | Flutterwave outages, buggy releases | Global and per-store switches (e.g. pause checkout) |
| Announcements and maintenance banners | Planned downtime, policy notices | Banner manager |
| Blocklist (banned bank accounts, phones) | Grows with enforcement | Blocklist manager with reason |

Rule for all of the above: every change is logged (who, when, old value, new value), and money-related settings need a confirmation step.

## 3. Accounts and records the admin needs access to

| Record | Access level | Notes |
|---|---|---|
| **Vendors** | Full view and action | Profile, verification status and documents, plan, trial, standing, risk flags, enforcement history, messages sent, notes |
| **Stores** | Full view and action | Template, subdomain, status, discovery eligibility, events summary |
| **Products/listings** | View, unpublish, flag | For moderation |
| **Orders and payments** | View | Status history, payment reference, Flutterwave reconciliation |
| **Disputes and reports** | View and resolve | Linked to orders, stores and cases |
| **Reviews** | View, remove | Removal only with a reason (fake/abusive) |
| **Customers (registered and guest)** | Limited view | Contact details masked by default, reveal on click (logged); support and data-request handling |
| **Subscriptions and billing** | View, limited edit | Extend/reset trial, comp a plan, with reason |
| **Admin/staff accounts** | Super admin only | Roles and permissions |
| **System records** | View | Webhook events (with manual retry), notification log, audit log |

Roles built in from day one even if only one person uses them at first: **Super Admin** (you), **Support** (view, notes, message), **Moderator** (flags, listings, reviews, warnings), **Finance** (payments, billing, reports). Adding a team member later then needs no rewrite.

A read-only **"view as vendor"** mode so you can see exactly what a vendor sees when they report a problem. It never allows edits, money actions or password access, and every use is logged.

## 4. Actions

| Action | What it does | Message sent |
|---|---|---|
| View / inspect | Open any vendor, store, order, listing | None |
| Monitor | Dashboards, risk queue, watchlist | None |
| Flag | Mark for review with a reason; can also be raised automatically | None (internal) |
| Add note | Internal comment on any record | None |
| Verify (Tier 1 / Tier 2) | Approve verification | Email + SMS: "You're verified" |
| Reject verification | With reason | Email with what to fix |
| Unverify | Remove verification with reason | Email (formal record) |
| Warn | Formal warning on the enforcement ladder | Email (formal record) |
| Remove from discovery | Timed, 30 days by default, auto-reinstates | Email |
| Unpublish listing | Remove one product | Email |
| Suspend store | Stop new orders; existing orders continue | Email + SMS, with appeal link |
| Ban | Permanent; blocklists bank account and phone | Email (formal record) |
| Reinstate / unban | With reason | Email + SMS |
| Process appeal | Accept or decline | Email |
| Extend or reset trial, comp a plan | Support goodwill | Email |
| Remove review | For fake or abusive content | Email to the reviewer where appropriate |
| Retry webhook | Re-process a failed payment event | None |

**Every punitive action requires a reason from a fixed list plus optional free text, and is written to the audit log.**

## 5. Automated messaging

- Each action above maps to a template. The admin picks the action, sees a **preview**, can add a short custom note, and confirms.
- **Email is the formal record.** SMS is used only for urgent status changes (verified, suspended, reinstated) and never contains detailed reasons; it links to the vendor dashboard instead.
- Templates use variables (vendor name, store name, reason, appeal link, dates), are versioned, and should be reviewed by a lawyer for suspension and ban wording.
- Tone follows the brand voice: warm but firm. Never cold or accusatory.
- Every message is stored in `notification_log`, linked to the action that triggered it.
- Confirm SMS sender ID and messaging-consent rules with Termii and Nigerian regulations before sending enforcement SMS.

## 6. What I think you're missing (recommendations)

1. **Admin security first.** Mandatory two-factor authentication, a separate `admin.cartiphy.com` subdomain, server-only permission checks, and session timeouts. A compromised admin account can suspend or expose everything.
2. **An immutable audit log.** Who did what, to which record, when, with what reason. This protects you in disputes with vendors and is essential once anyone else has access.
3. **Cases, not just flags.** A "case" groups the flags, buyer reports, disputes, messages and decision for one issue, so nothing is scattered across screens. It also gives appeals a home.
4. **Reversibility.** Timed actions expire automatically; every ban and suspension can be reversed with a reason. Wrongful bans are the biggest reputational risk.
5. **Two-step confirmation for destructive actions** (ban, unverify, bulk actions), with the consequences spelled out before you confirm.
6. **Search everything.** One search box that finds a phone number, email, order ID, subdomain or bank account name. Support work is mostly lookup.
7. **Reconciliation and debugging tools.** A payments view against Flutterwave records and a webhook-events viewer with manual retry. This is the most valuable tool for someone who can't read the code when a payment goes wrong.
8. **Platform health dashboard.** Vendors by tier and plan, monthly recurring revenue, commission collected, trial conversion, dispute rate, failed payments and webhooks, SMS spend, queue age.
9. **Data protection tooling.** Masked personal data by default, logged reveals, and handling for customer data-access and deletion requests under the Nigeria Data Protection Act.
10. **Finance exports.** Commission and subscription income reports your accountant can use, including VAT.
11. **Data retention rules.** How long to keep resolved cases, messages and logs, decided with your lawyer.
12. **Kill switches.** Pause all checkouts if Flutterwave has an outage; pause a single store instantly.
13. **Admin-lite early.** Do not wait for Phase 8. From Phase 3 you need a read-only page for orders and webhook events, or you'll be blind while debugging payments.

## 7. Admin data model additions

`admin_users`, `admin_roles`, `audit_log`, `cases`, `case_notes`, `message_templates`, `template_versions`, `platform_settings`, `settings_history`, `feature_flags`, `content_rules`, `policy_versions`, `verification_reviews`, `announcements`, `data_requests`. Existing tables from build-plan-v2 (`risk_flags`, `enforcement_actions`, `appeals`, `blocklist`, `notification_log`, `webhook_events`) are reused.

## 8. Build phases

| Stage | Contents | When |
|---|---|---|
| **A0 Admin-lite** | Admin login with 2FA, read-only lookup for orders, payments and webhook events, audit log foundation | Phase 3–4 |
| **A1 Launch-critical** | Vendor and store search and detail, verification queue, flag / warn / suspend / ban with templated email and SMS, disputes and reports queue, content-scanner rules, plan and settings editor, audit log UI | Phase 8 |
| **A2 Operations** | Cases and appeals, health dashboard, reconciliation, timed actions and reinstatement, bulk actions with safeguards, roles for extra staff | Post-launch |
| **A3 Scale** | Finance exports, data-request tooling, announcements, advanced analytics, view-as-vendor refinements | Later |

## 9. Design notes

- Desktop-first (unlike the mobile-first vendor dashboard), data-dense tables, filters and side panels.
- Uses the same design tokens and anti-generic rules as the rest of Cartiphy, though it can be plainer since it is an internal tool. Built from `design.md` and code, not Sleek (which is mobile-focused).
- Clear visual weight for destructive actions; status colors must be defined before building.

## 10. Decisions (defaults assumed)

1. Admin users: only the owner at first. Roles (Super Admin, Support, Moderator, Finance) are built from day one so staff can be added without a rewrite.
2. Read-only "view as vendor" is included, fully logged, with no edits, money actions or password access.
3. Enforcement reason list: fraud, circumvention, fake listing, abusive content, failed verification, other. Extendable later.
4. Data retention and deletion periods are set with a lawyer before launch.
5. Enforcement SMS is sent only after confirming sender ID and messaging-consent rules with Termii and Nigerian regulations.
6. Admin-lite (2FA login, read-only orders, payments and webhook events, audit log foundation) is built during Phases 3–4, not deferred to Phase 8.
