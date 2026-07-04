# Database (Supabase)

This is a high-level list of tables. For exact columns, check Supabase directly — this file tracks purpose, multi-tenancy status, and known issues, not full schema details.

Project ref: `rllriopqojaraceytdno` (us-east-1)

---

## Core / Multi-Tenant

- **businesses** — each customer account using the CRM. The root tenant record; every business-scoped table should ultimately resolve to a row here via business_id. Also confirmed present: default_tax_rate(used when creating a new quote/invoice),stripe_connect_id, stripe_connect_ready, stripe_connect_onboarded (Stripe Connect status, used by Invoicing to gate payment collection).
- **profiles** — user profiles, linked to businesses. This is the source of truth for a user's `business_id` association (join here for tenant checks, not `users`).
- **users** — Supabase auth records. Do not assume `business_id` lives here — always resolve via `profiles`.
- **superusers** — staff admin accounts (elevated access per Business Rules RLS section).
- **beta_testers** — beta program tracking (status, invite tokens, activation).

---

## CRM Core

- **contacts** — a business's customers/contacts. `business_id` required.
- **leads** — incoming leads. Uses lead_name/lead_email/lead_phone/lead_statusnaming (not generic columns).business_idrequired. Also confirmed present:lead_address(used acrosscontacts_screen.dartandcontact_detail_screen.dart, pre-dates this sync) and client_access_token(random per-lead token powering the/client/:token portal route — generated on first quote/invoice send if the lead doesn't already have one).
- **deals** — sales pipeline deals. Uses `lead_id` FK referencing `leads` (UI labels this field "Contact" — not a `contact_id`). `business_id` required.
- **pipelines** — pipeline definitions. `business_id` required.
- **pipeline_stages** — stages within a pipeline. Scoped via `pipeline_id` → `pipelines.business_id`; verify FK join doesn't allow cross-business stage assignment.
- **custom_values** — custom fields/values for contacts or deals. `business_id` required.
- **tasks** — to-dos/follow-ups. `business_id` required.

---

## Conversations / Messaging

- **conversations** — message threads. `business_id` required.
- **conversation_views** — saved filter views for the Conversations screen. ⚠️ **RLS currently disabled** — tracked exception, pending fix. Should have `business_id` (and likely `user_id` for per-user saved views).
- **messages** — individual messages. `business_id` required. ⚠️ **RLS pending re-enable** — tracked exception in Business Rules.
- **support_chats** — support conversations with staff. Scoped to business + superuser visibility.
- **support_tickets** — support ticket records. Scoped to business + superuser visibility.
- **snippets** — saved canned-reply text. `business_id` confirmed present in insert code (`snippets_screen.dart`). RLS policy status unverified (no migration files in repo) — check Supabase dashboard.

---

## AI / Knowledge Base

- **nexaflow_kb** — internal knowledge base powering the support chatbot for the CRM product itself. **No `business_id` — internal/global only.** AI agents must never mix this with `knowledge_base`.
- **knowledge_base** — per-business knowledge base used by AI to answer that business's customers and book appointments. **Must have `business_id`**, and AI agents must only query a business's own rows. Do not confuse with `nexaflow_kb`.
- **ai_usage_logs** — tracking AI usage/costs. `business_id` required for per-business cost attribution.

---

## Calendar / Appointments

- **appointments** — scheduled appointments. `business_id` required.
- **calendars** — calendar definitions. `business_id` required.
- **calendar_rooms** — bookable rooms. Scoped via `calendar_id` → business; verify join.
- **calendar_equipment** — bookable equipment. Scoped via `calendar_id` → business; verify join.
- **calendar_groups** — grouping of calendars. `business_id` required.

---

## Automations

- **automations** — automation definitions. `business_id` required.
- **automation_enrollments** — contacts enrolled in automations. ⚠️ Marked "unrestricted" — likely relies on `automation_id` / `contact_id` joins for business scoping rather than a direct `business_id` column. This is a potential cross-tenant leak vector per the FK-join rule in Business Rules; needs review — consider adding a direct `business_id` column.
- **automation_logs** — automation run history. `business_id` required (or resolvable via `automation_id` join — verify).

---

## Job Costing 

- **job_expenses** — cost entries logged against a job (an appointment or a deal). `business_id` required. Confirmed columns from insert code (`log-job-expense` edge function): `business_id`, `appointment_id` (nullable), `deal_id` (nullable), `expense_type` (labor|material|subcontractor|other), `amount_cents`, `description`, `logged_by_profile_id`, `logged_at`, `deleted_at` (referenced by `compute-job-cost-snapshot` as a soft-delete filter). Gated behind Growth plan via `check_plan_feature` RPC (`job_costing` feature).
- **job_revenue_snapshots** — computed profit/loss snapshot per job (appointment or deal), upserted by the `compute-job-cost-snapshot` edge function on conflict of `appointment_id` or `deal_id`. Confirmed columns: `business_id`, `appointment_id`, `deal_id`, `total_expenses_cents`, `total_revenue_cents`, `gross_profit_cents`, `profit_margin_pct`, `job_type`, `snapshotted_at`. Revenue side is pulled from the `invoices` table (see Open Questions — `invoices` itself is not yet documented here).

## Jobs / Quotes / Invoicing

  - **quotes** — customer-facing price quotes. `business_id` required. ⚠️ Primary key is NOT `bigint` — see IDs exception in Business Rules. Confirmed columns: `business_id`, `contact_id` (FK → `leads`, same "labeled Contact, actually a leads FK" pattern as `deals`), `quote_number` (format `Q-###`), `status` (draft|sent|approved|declined), `job_title`, `expires_at`, `notes`, `tax_rate`, `subtotal`, `tax_amount`, `total`, `sent_at`, `approved_at`, `approved_via`, `updated_at`, `created_at`, `deleted_at`.
  - **invoices** — customer-facing invoices, created fresh or converted from a `quotes` row. `business_id` required. ⚠️ Primary key is NOT `bigint` — see IDs exception in Business Rules. Confirmed columns: `business_id`, `contact_id` (FK → `leads`), `quote_id` (nullable FK → `quotes`, set when converted from a quote), `invoice_number` (format `INV-###`), `status` (approved|paid, at minimum), `amount_due`, `subtotal`, `tax_amount`, `tax_rate`, `job_title`, `notes`, `due_date`, `payment_link_id` (FK → `payment_links`), `paid_at`, `updated_at`, `created_at`, `deleted_at`.
  - **line_items** — shared line-item table for both quotes and invoices, distinguished polymorphically. `business_id` required. Confirmed columns: `business_id`, `parent_type` (`'quote'` | `'invoice'`), `parent_id`, `service_item_id` (nullable FK → `service_library`), `description`, `quantity`, `unit_price`, `discount_type`, `discount_value`, `total`, `sort_order`, `updated_at`, `deleted_at`.
  - **service_library** — a business's preset services/prices, managed from Settings → Service Library, used to prefill quote/invoice line items. `business_id` required. Confirmed columns: `business_id`, `name`, `description`, `default_price`, `unit`, `is_active`, `updated_at`, `deleted_at`.
  - **payment_links** — Stripe payment tracking, one per invoice (also used independently of invoices by the pre-existing `generate-payment-link` function). `business_id` required. Confirmed columns: `business_id`, `invoice_id`, `stripe_payment_intent_id`, `stripe_payment_link_url`, `amount_cents`, `currency`, `status` (pending|paid), `paid_at`, `deleted_at`.
  - **client_service_requests** — service requests submitted by a customer through the client portal. `business_id` required, uses a normal `bigint` id (follows the IDs rule, unlike quotes/invoices above). Confirmed columns: `business_id`, `lead_id`, `description`, `preferred_date`, `status` (new|reviewed|scheduled|declined), `internal_notes`, `created_at`, `deleted_at`.

---

## Marketing

- **campaigns** — marketing campaigns. `business_id` required.
- **campaign_contacts** — contacts targeted in campaigns. Scoped via `campaign_id` → business; verify join doesn't allow cross-business contact targeting.
- **forms** — lead capture forms. `business_id` required.
- **form_submissions** — form submission records. Scoped via `form_id` → business; verify join.
- **smart_lists** — saved contact filters/segments. `business_id` required.

---

## Misc

- **trigger_links** — trackable links created per-business for campaigns. `business_id` confirmed present in insert code (`conversations_screen.dart`, `handle-trigger-link` edge function). RLS policy status unverified (no migration files in repo) — check Supabase dashboard.
- **trigger_link_clicks** — click tracking on trigger links. Scoped via `trigger_link_id` → business; same concern as above — if `trigger_links` lacks `business_id`, this table inherits the gap.
- **call_logs** — records of inbound phone calls. Written by `handle-inbound-call` edge function. Columns confirmed from insert code: `business_id`, `contact_id` (nullable), `phone_number_from`, `phone_number_to`, `call_status` (default "answered", overwritten by status callback), `twilio_call_sid`, `reply_sent` (bool, deduplication flag). `handle-call-status` reads this table to decide whether to send missed-call SMS. `business_id` confirmed present. RLS policy status unverified (no migration files in repo) — check Supabase dashboard.
---

## Possibly Missing / Unclear — Needs Confirmation

- **Team Members / Permissions** — referenced in Settings UI. Confirm whether this is its own table (e.g. `team_members`, `permissions`, `roles`) or modeled via `profiles` + role column.
- **Business Hours** — referenced in roadmap (`_visibleHourRange()` in calendar). Confirm whether this is a dedicated table, a JSON column on `businesses`, or part of `calendars`.
- **Stripe / Subscription state** — confirmed: billing columns live directly on businesses. Columns written by stripe-webhook: \is_paid` (bool), `client_id` (Stripe customer ID), `subscription_id`, `subscription_status` (starter|growth|pro|cancelled). No separate `subscriptions` or `stripe_customers` table exists. Also present on `businesses` but unconfirmed/unused: `minutes_used_this_month` and `included_minutes` — read by the Billing settings screen (settings_screen.dart, ~line 3048) to render a usage bar, but no edge function or code path in the repo writes to either column. Likely scaffolding for the not-yet-built usage-limit system referenced in business_rules.md Plan Gating.`
- **Webhook / inbound SMS logs** — useful for debugging Make scenarios and the SMS booking flow. Confirm whether this is covered by `automation_logs` / `ai_usage_logs` or needs its own table.

---

## Open Items Summary (for AI agents)

When proposing schema changes, treat the following as **known debt, not acceptable patterns to replicate**:
1. `conversation_views` — RLS disabled
2. `messages` — RLS pending re-enable
3. `snippets` and `trigger_links` — `business_id` column confirmed present in app code. RLS policy correctness still needs Supabase dashboard verification.
4. `deals` — `contact_id` FK debt resolved; now uses `lead_id` FK referencing `leads` table.
Do not copy these patterns into new tables. New tables must follow Business Rules in full from creation.
