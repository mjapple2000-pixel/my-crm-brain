# Database (Supabase)

This is a high-level list of tables. For exact columns, check Supabase directly — this file tracks purpose, multi-tenancy status, and known issues, not full schema details.

Project ref: `rllriopqojaraceytdno` (us-east-1)

---

## Core / Multi-Tenant

- **businesses** — each customer account using the CRM. The root tenant record; every business-scoped table should ultimately resolve to a row here via `business_id`.
- **profiles** — user profiles, linked to businesses. This is the source of truth for a user's `business_id` association (join here for tenant checks, not `users`).
- **users** — Supabase auth records. Do not assume `business_id` lives here — always resolve via `profiles`.
- **superusers** — staff admin accounts (elevated access per Business Rules RLS section).
- **beta_testers** — beta program tracking (status, invite tokens, activation).

---

## CRM Core

- **contacts** — a business's customers/contacts. `business_id` required.
- **leads** — incoming leads. Uses `lead_name` / `lead_email` / `lead_phone` / `lead_status` naming (not generic columns). `business_id` required.
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
- **Stripe / Subscription state** — confirmed: billing columns live directly on `businesses`. Columns written by `stripe-webhook`: `is_paid` (bool), `client_id` (Stripe customer ID), `subscription_id`, `subscription_status` (`starter` | `growth` | `pro` | `cancelled`). No separate `subscriptions` or `stripe_customers` table exists.
- **Webhook / inbound SMS logs** — useful for debugging Make scenarios and the SMS booking flow. Confirm whether this is covered by `automation_logs` / `ai_usage_logs` or needs its own table.

---

## Open Items Summary (for AI agents)

When proposing schema changes, treat the following as **known debt, not acceptable patterns to replicate**:
1. `conversation_views` — RLS disabled
2. `messages` — RLS pending re-enable
3. `snippets` and `trigger_links` — `business_id` column confirmed present in app code. RLS policy correctness still needs Supabase dashboard verification.
4. `deals` — `contact_id` FK debt resolved; now uses `lead_id` FK referencing `leads` table.
Do not copy these patterns into new tables. New tables must follow Business Rules in full from creation.
