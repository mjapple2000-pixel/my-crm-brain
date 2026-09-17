# Business Rules

These rules are non-negotiable. Any AI agent generating database changes, code, or automations MUST follow these.

## Multi-Tenancy

- Every business-specific table must have a `business_id` column.
- Every query must filter by `business_id`.
- A business can never see or access another business's data.
- Foreign key joins must never allow a row to reference a parent/related row belonging to a different `business_id`. When adding a new FK, verify both sides resolve to the same business.
- Any code path using the `service_role` key (edge functions, Make scenarios, admin scripts) bypasses RLS entirely. These paths MUST manually filter by `business_id` in application code — RLS is not a safety net here.

## Row Level Security (RLS)

- RLS must be enabled on all tables containing business data.
- RLS policies must check that the requesting user's `business_id` matches the row's `business_id`.
- Superusers (VantageCareTech staff) may have separate elevated policies, but these must be explicit, limited, and documented.
- Any superuser access that bypasses normal `business_id` scoping should be logged (who, which business, when) where practical.
- Any new table or schema change must include working RLS policies AND explicit grants (`GRANT ALL TO authenticated`, sequence grants) — enabling RLS alone is not sufficient.
- `conversation_views` and `messages` RLS were previously tracked exceptions (disabled/pending) — confirmed via live Supabase advisor check as of 9/16 sync: both now have RLS enabled with working policies. No longer an open exception.

## IDs

- All primary keys are `bigint` (auto-incrementing) — not UUIDs.
- Never reuse or guess IDs — always let the database generate them automatically via `bigint` serial/identity columns.
- Exception found: quotes.id and invoices.id are NOT bigint — code casts them as String in three places (new_quote_screen.dart line 353, new_invoice_screen.dart line 429, quote_detail_screen.dart line 977), which only works if the underlying column is text/uuid. This contradicts the bigint-only rule above. Flagging for Mike to confirm intent rather than auto-correcting either the code or the rule.

## Deletes

- Never hard-delete records.
- Use soft deletes (`deleted_at` timestamp, nullable).
- Deleted records remain in the database but must be filtered out of normal views/queries (`WHERE deleted_at IS NULL`).

## Timestamps

- Every table must have `created_at` and `updated_at` columns.
- `updated_at` must auto-update via a `BEFORE UPDATE` trigger — never rely on application code to set it.

## Naming Conventions

- Table names: lowercase, plural, snake_case (e.g. `contacts`, `pipeline_stages`).
- Column names: lowercase, snake_case.
- Foreign keys: named `<table_singular>_id` (e.g. `business_id`, `contact_id`).

## Indexes

- Always add an index on `business_id` for any business-scoped table.
- For soft-deleted tables, use a partial index scoped to `WHERE deleted_at IS NULL` rather than a plain index, to keep frequent queries efficient as deleted rows accumulate.
- Add indexes on foreign keys and any columns used in frequent filters/sorts.

## AI Agents — General Rules

- Always read `crm_vision.md`, `tech_stack.md`, `database.md`, and this file before proposing changes.
- Never propose features that conflict with the scope decisions in `crm_vision.md`.
- Flag any schema change that affects RLS policies — these require extra care and explicit review.
- Never write directly to production. All changes go through review/approval first.

## Data Privacy

- Customer data (a business's contacts, leads, conversations, etc.) must never be used to train models or shared across businesses.
- AI agents accessing `knowledge_base` for a business must only access that business's own records.

## Plan Gating

Feature access is gated by plan tier. This is non-negotiable — without gating there is no reason for any customer to upgrade.

- The three confirmed tiers are: **Starter ($97/mo)**, **Growth ($297/mo)**, **Pro ($497/mo)**.
- All paid plans include a **15-day free trial** — the customer selects a plan at signup, gets 15 days free, then is charged that plan's rate automatically.
- **Beta users are completely separate.** Beta testers are personally selected family and friends who have permanent free access in exchange for testing and feedback. They are not trials, not paying customers, and not part of the public product. Beta access is controlled via the `is_beta` flag and `beta_testers` table — never remove or modify this system when implementing plan gating.
- **Beta access now has a usage cap, per `receive-email/index.ts` lines 774–806 (mirrored in `microsoft-graph-webhook/index.ts`).** A beta business is capped at the Pro-equivalent AI-message allowance (2,500/mo, read from `business_usage_live`) unless `businesses.beta_card_added` is true. Past the cap: AI pauses on that conversation (`ai_enabled = false`, `flagged_for_beta_cap = true`), the customer gets a "team member will follow up" message, and `notify-beta-cap-reached` emails the owner once per billing period (deduped via `business_usage.beta_cap_notified_at`). A business can add a card via the new `create-beta-card-setup` Stripe Checkout flow (Settings → Billing, `_BillingSectionState`, `settings_screen.dart` line 5168) to cover overage and lift the cap. **Flagging for Mike to confirm this is intended** — it narrows the "permanent free access" language above from unconditional to capped-with-optional-card, and I don't want to silently rewrite that line without you signing off.
- **New abuse circuit breaker, undocumented until now** — `receive-email/index.ts` lines 760–770 (mirrored in `microsoft-graph-webhook/index.ts`): if a single conversation gets 45 AI replies within a rolling 24-hour window, the AI hands off with a canned "someone will follow up" message and sets `conversations.ai_enabled = false`, `flagged_for_abuse = true`. Tracked via `ai_reply_count_24h`/`ai_reply_window_reset_at`. This is independent of and separate from the beta cap above.
- The businesses table has two separate columns that must never be conflated: plan (tier value — starter|growth|pro, derived from the Stripe Price ID and intentionally kept even after cancellation for analytics/win-back) and subscription_status (Stripe's own lifecycle state — active|trialing|past_due|cancelled). The Stripe webhook (stripe-webhook edge function) writes plan on customer.subscription.created/updated and subscription_status on both updated and deleted events, but never clears plan on cancellation. As of this sync, check-plan-feature/index.ts (185 lines total) no longer contains the subscription_status/beta logic itself — a code comment states that logic now lives entirely inside a check_plan_feature() Postgres function, which is the single source of truth (also used directly by other functions and RLS policies). That Postgres function's definition still isn't visible in the repo — the 7 migration files that now exist don't include it — so you'd still need the Supabase dashboard to confirm the subscription_status-then-plan check order.
- **Starter** unlocks: SMS, unified inbox, pipeline, automations (limited), missed call text back, AI receptionist, review requests, contact timeline, appointment reminders.
- **Growth** unlocks: full AI suite (sales coach, lead responder, appointment assistant, review responses), custom workflows, multiple pipelines, API access, advanced automations.
- **Pro** unlocks: priority support, unlimited SMS, advanced analytics, white label (future), dedicated onboarding.
- Now called by at least 24 edge functions: log-job-expense, get-job-costing-report, and get-expense-report (feature: job_costing), optimize-route (feature: route_optimization), update-team-location (feature: gps_tracking), send-on-my-way-sms and employee-hub-action (feature: on_my_way_sms — employee-hub-action also separately gates job_costing), clock-in-out, edit-timesheet-entry, force-clock-out, get-timesheets, and export-timesheets-pdf (feature: time_tracking), get-timesheets and export-timesheets-pdf ALSO gate overtime_tracking, generate-weekly-insight (feature: ai_dashboard_insights), and gmail-oauth-connect (feature: gmail_sync — Growth+ only, confirmed at gmail-oauth-connect/index.ts lines 346–349). Also newly confirmed: decide-timesheet, plus parts of employee-hub-action and get-employee-hub-data (feature: timesheet_approval_workflow — undocumented until now); receive-email, gmail-inbound-webhook, and microsoft-graph-webhook (feature: email_trust_filtering — gates the new sender allow/block rules, see database.md); quickbooks-sync-hours (feature: quickbooks_payroll_sync); send-campaign (feature: campaigns); microsoft-oauth-callback (feature: outlook_sync, Pro only); and three cron-triggered reminder functions with feature keys that appear in no other doc: send-appointment-reminders (feature: appointment_reminders), send-invoice-overdue-reminders (feature: invoice_overdue_reminders), and send-quote-followups (feature: quote_followups). ⚠️ AI Form Recreation, the Template Library, and the PTO system (pto_policy_screen.dart, employee_pto_screen.dart — feature key pto_tracking) are all gated client-side only, with NO server-side check_plan_feature call in any pto_requests-writing edge function (notify-pto-request, get-timesheets, export-timesheets-pdf, quickbooks-sync-hours) — see Open Questions.
- When a customer hits a gated feature they don't have access to, they see an upgrade prompt, not an error. Confirmed in 4 screens: reporting_screen.dart (lines 299 and 330), pipelines_screen.dart (line 2585), appointments_screen.dart (lines 6154 and 7505), and routes_screen.dart (line 364) all check for {error: \"upgrade_required\"} from the server.
- Usage limits also apply by tier (AI message usage, tracked monthly). This is now built: see database.md's \business_usage`/`business_usage_live` tables and the `get_business_usage_summary` RPC, surfaced in Settings → Billing. The old `businesses.minutes_used_this_month`/`included_minutes` scaffolding referenced here previously has been fully removed from the codebase — zero references remain.
- We are not racing to the bottom on pricing. Never propose reducing tier prices or collapsing tiers without explicit instruction.

## Payment Processing Fees

- NexaFlow takes a platform fee on every card payment a customer makes through a business's connected Stripe account. `create-invoice-payment` applies `application_fee_amount` (a percentage read from the `PLATFORM_FEE_PERCENT` environment variable) on top of the invoice or milestone amount when creating the Stripe Checkout session. Any new or replacement payment-collection path must apply this same fee — do not build one that skips it.
