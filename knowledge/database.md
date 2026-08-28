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

- **job_expenses** — cost entries logged against a job (an appointment or a deal). `business_id` required. Confirmed columns from insert code (`log-job-expense` edge function): `business_id`, `appointment_id` (nullable), `deal_id` (nullable), `category_id` (FK → `expense_categories`, required — validated server-side against the business), `expense_type` (labor|material|subcontractor|other, still present alongside `category_id`), `amount_cents`, `description`, `logged_by_profile_id`, `logged_at`, `deleted_at` (referenced by `compute-job-cost-snapshot` as a soft-delete filter). Gated behind Growth plan via `check_plan_feature` RPC (`job_costing` feature).
- **expense_categories** — a business's custom expense categories (e.g. "Fuel", "Materials"), managed from Settings (`_ExpenseCategoriesSectionState`, `settings_screen.dart` ~line 9900). `business_id` required. Confirmed columns: `business_id`, `name`, `is_active`, `deleted_at`. Referenced by `job_expenses.category_id`.
- **job_revenue_snapshots** — computed profit/loss snapshot per job (appointment or deal), upserted by the `compute-job-cost-snapshot` edge function on conflict of `appointment_id` or `deal_id`. Confirmed columns: `business_id`, `appointment_id`, `deal_id`, `total_expenses_cents`, `total_revenue_cents`, `gross_profit_cents`, `profit_margin_pct`, `job_type`, `snapshotted_at`. Revenue side is pulled from the `invoices` table (see Open Questions — `invoices` itself is not yet documented here).

---

  ## Jobs / Quotes / Invoicing

  - **quotes** — customer-facing price quotes. `business_id` required. ⚠️ Primary key is NOT `bigint` — see IDs exception in Business Rules. Confirmed columns: `business_id`, `contact_id` (FK → `leads`, same "labeled Contact, actually a leads FK" pattern as `deals`), `quote_number` (format `Q-###`), `status` (draft|sent|approved|declined), `job_title`, `expires_at`, `notes`, `tax_rate`, `subtotal`, `tax_amount`, `total`, `sent_at`, `approved_at`, `approved_via`, `updated_at`, `created_at`, `deleted_at`.
  - **invoices** — customer-facing invoices, created fresh or converted from a `quotes` row. `business_id` required. ⚠️ Primary key is NOT `bigint` — see IDs exception in Business Rules. Confirmed columns: `business_id`, `contact_id` (FK → `leads`), `quote_id` (nullable FK → `quotes`, set when converted from a quote), `invoice_number` (format `INV-###`), `status` (approved|paid, at minimum), `amount_due`, `subtotal`, `tax_amount`, `tax_rate`, `job_title`, `notes`, `due_date`, `payment_link_id` (FK → `payment_links`), `is_progress_billed` (bool — flags an invoice as milestone/progress-billed, gates `invoice_milestones` processing), `paid_at`, `updated_at`, `created_at`, `deleted_at`.
  - **invoice_milestones** — individual progress-billing milestones on an `is_progress_billed` invoice, auto-sent when due via `process-due-milestones` → `send-milestone-invoice` (AI-drafted send message via GPT-4o-mini, same pattern as `send-invoice`). `business_id` required. Confirmed columns: `business_id`, `invoice_id` (FK → `invoices`), `label`, `due_date`, `status` (pending|ready_to_bill, at minimum), `updated_at`, `deleted_at`.
  - **line_items** — shared line-item table for both quotes and invoices, distinguished polymorphically. `business_id` required. Confirmed columns: `business_id`, `parent_type` (`'quote'` | `'invoice'`), `parent_id`, `service_item_id` (nullable FK → `service_library`), `description`, `quantity`, `unit_price`, `discount_type`, `discount_value`, `total`, `sort_order`, `updated_at`, `deleted_at`.
  - **service_library** — a business's preset services/prices, managed from Settings → Service Library, used to prefill quote/invoice line items. `business_id` required. Confirmed columns: `business_id`, `name`, `description`, `default_price`, `unit`, `is_active`, `updated_at`, `deleted_at`.
  - **payment_links** — Stripe payment tracking, one per invoice (also used independently of invoices by the pre-existing `generate-payment-link` function). `business_id` required. Confirmed columns: `business_id`, `invoice_id`, `stripe_payment_intent_id`, `stripe_payment_link_url`, `amount_cents`, `currency`, `status` (pending|paid), `paid_at`, `deleted_at`.
  - **client_service_requests** — service requests submitted by a customer through the client portal. `business_id` required, uses a normal `bigint` id (follows the IDs rule, unlike quotes/invoices above). Confirmed columns: `business_id`, `lead_id`, `description`, `preferred_date`, `status` (new|reviewed|scheduled|declined), `internal_notes`, `created_at`, `deleted_at`.
  - **appointment_contact_info** — NOT a table, this is a database VIEW. Resolves an appointment's current contact info (live lead data if the lead still exists, otherwise falls back to the frozen snapshot on the appointment row) into appointment_id, resolved_name, resolved_phone, resolved_email. Read by appointments_screen.dart, employee-hub-action, and send-on-my-way-sms so all three don't duplicate the same live-vs-frozen fallback logic.

---

## Marketing

- **campaigns** — marketing campaigns. `business_id` required.
- **campaign_contacts** — contacts targeted in campaigns. Scoped via `campaign_id` → business; verify join doesn't allow cross-business contact targeting.
- **forms** — lead capture forms. `business_id` required.
- **form_submissions** — form submission records. Scoped via `form_id` → business; verify join.
- **smart_lists** — saved contact filters/segments. `business_id` required.

---

## Employee Hub / Field Ops

- **employee_hub_tokens** — token-based auth for the public, unauthenticated Employee Hub (`/hub/:token`, same pattern as the Client Portal's `client_access_token`). Confirmed columns: `token`, `profile_id`, `business_id`, `revoked_at` (set on revoke/reissue via `resend-employee-hub-link`).
- **job_forms** — form templates (checklists, inspection forms, etc.) built by staff via the \"Job forms\" tab in the Jobs Hub, or generated by the AI Form Recreation flow. business_idrequired. Confirmed columns:business_id, name, form_type, fields(jsonb — array of{id, type, label, required}, plus optionsfor select fields),requires_signature, deleted_at, recreation_mode('standard' | 'visual_recreation' — set by AI Form Recreation),background_pages(jsonb — page image references used to render the original form as a background whenrecreation_modeisvisual_recreation).
- **job_form_submissions** — an instance of a job form attached to a specific appointment, filled out by a crew member through the Employee Hub. business_idrequired. Confirmed columns:business_id, job_form_id, appointment_id, status(not_started|in_progress, at minimum),answers(jsonb),photo_urls, completed_by_profile_id, deleted_at, signature_url, signed_by_name, signed_at, pdf_url, submission_label, extra_pages, rendered_page_urls.
- **job_types** — a business's job categories, managed from the "Service Menu" area of Calendar Settings. `business_id` required. Confirmed columns: `business_id`, `name`, `is_active`, `deleted_at`.
- **phone_numbers** — Twilio numbers provisioned per business via `provision-phone-number`. `business_id` required. Confirmed columns: `business_id`, `twilio_sid`, `phone_number`, `friendly_name`, `deleted_at`.
- **routes** — a day's optimized stop order for a crew member, computed by `optimize-route`. `business_id` required. Confirmed columns: `business_id`, `assigned_user_id`, `route_date`, `stops` (jsonb — ordered `{appointment_id, sequence, lat, lng}` array), `optimized_at`.
- **team_locations** — live GPS position per crew member, upserted by `update-team-location` (called from the Employee Hub) and read by `routes_screen.dart` to plot crew on the map. Confirmed columns: `user_id`, `business_id`, `latitude`, `longitude`.
- **time_entries** — clock-in/clock-out records. `business_id` required. Confirmed columns: `business_id`, `appointment_id` (nullable), `user_id`, `clocked_in_at`, `clock_in_lat`, `clock_in_lng`, `status` (active, at minimum), `deleted_at`. Written by `clock-in-out`; also read directly by `main_layout.dart` and `appointments_screen.dart` to show an active-clock indicator app-wide.
- **service_menu_items** — services offered for booking (distinct from `service_library`, which feeds quote/invoice line items — see Open Questions). `business_id` required. Confirmed columns: `business_id`, `name`, `description`, `duration_minutes`, `price`, `calendar_ids`, `is_active`, `updated_at`. Full CRUD lives in the "Service Menu" tab of the Calendar Settings dialog inside `appointments_screen.dart` — not in `settings_screen.dart`.
- **stripe_connect_accounts** — a second Stripe Connect record, separate from the `stripe_connect_id`/`stripe_connect_ready`/`stripe_connect_onboarded` columns already documented on `businesses` above (both systems are intentionally in use, per Mike — the split rationale isn't visible from code). Confirmed columns: `business_id`, `stripe_account_id`, `onboarding_complete`, `charges_enabled`, `payouts_enabled`, `deleted_at`. Written by `stripe-connect-onboard`/`stripe-connect-webhook`; read by `generate-payment-link`. ⚠️ Still no confirmed `lib/` UI entry point as of this sync — new edge functions `create-connect-account`, `get-connect-status`, `get-express-dashboard-link`, and `get-stripe-balance` also touch Stripe Connect but I have not yet traced whether they read this table or the `businesses`-column system. Flagging for next pass rather than guessing.
- **accounting_connections** — a business's connection to an accounting provider (currently QuickBooks Online only). `business_id` required. Confirmed columns: `business_id`, `provider` (`'quickbooks'`), `connection_status` (`connected`|`error`, at minimum), `access_token_secret_id`, `refresh_token_secret_id` (both reference Supabase Vault secrets via `qb_vault_read_secret` RPC — tokens are never stored in plaintext), `token_expires_at`, `qb_realm_id`, `deleted_at`. Written by `quickbooks-oauth-connect`; read by `quickbooks-sync-contact` and `quickbooks-sync-invoice`. UI: Settings → Payment Options, `_PaymentOptionsSectionState` (`settings_screen.dart` ~line 5215-5300).
- **accounting_customer_links** — maps a `leads` row to its corresponding QuickBooks customer record. `business_id` required. Confirmed columns: `business_id`, `lead_id` (FK → `leads`), `provider`, `qb_customer_id`, `updated_at`, `deleted_at`. Written/read by `quickbooks-sync-contact`.
- **accounting_invoice_links** — presumed equivalent mapping for invoices, referenced by filename in `quickbooks-sync-invoice` but I have not yet confirmed its columns from that function's code — flagging for next pass rather than guessing.
- **accounting_sync_log** — audit log of each sync attempt to the accounting provider. `business_id` required. Confirmed columns: `business_id`, `entity_type` (`'contact'`, at minimum — invoice sync likely adds `'invoice'`, unconfirmed), `local_id`, `qb_id`, `direction` (`'to_qb'`), `status` (`success`|`failed`), `error_message`, `synced_at`. Written by `quickbooks-sync-contact` (and presumably `quickbooks-sync-invoice`, unconfirmed).

---

## AI Form Recreation & Template Library

- **job_form_ai_drafts** — an in-progress AI form recreation session (photo/PDF upload → OCR → GPT → confirmed job form). `business_id` required. Confirmed columns: `business_id`, `status` ('processing'|'confirmed', at minimum), `created_by_profile_id`, `is_blank_template`, `source_file_url`, `source_page_urls`, `confirmed_job_form_id` (FK → `job_forms`, set on confirm), `updated_at`. Source files live in the `job-form-ai-sources` Storage bucket.
- **form_templates** — a shared library entry publishing one business's `job_forms` row for reuse (optionally across businesses). `business_id` required (owning business). Confirmed columns: `business_id`, `title`, `description`, `source_job_form_id` (FK → `job_forms`), `min_tier` (present for a possible future per-template plan gate — confirmed by code comment as not currently used by any gate), `deleted_at`. RLS gives authenticated users read-only access; all writes go through the `job-form-editor` edge function.
- **form_template_tags** — join table linking a `form_templates` row to one or more `form_tags`. Confirmed columns: `form_template_id`, `tag_id`.
- **form_tags** — the pool of tags available for labeling templates in the library. Confirmed columns: `id`, `name`, `deleted_at`.
- **job_form_photo_attachments** — GPS/location-tagged photos attached to a marker placed during AI Form Recreation (distinct from the `photo_urls` field-answer array on `job_form_submissions`, which is unrelated). Confirmed columns: `submission_id` (FK → `job_form_submissions`), `marker_id`, `storage_path`, `latitude`, `longitude`, `captured_at`, `created_at`, `deleted_at`.

---

## Misc

- **trigger_links** — trackable links created per-business for campaigns. `business_id` confirmed present in insert code (`conversations_screen.dart`, `handle-trigger-link` edge function). RLS policy status unverified (no migration files in repo) — check Supabase dashboard.
- **trigger_link_clicks** — click tracking on trigger links. Scoped via `trigger_link_id` → business; same concern as above — if `trigger_links` lacks `business_id`, this table inherits the gap.
- **call_logs** — records of inbound phone calls. Written by `handle-inbound-call` edge function. Columns confirmed from insert code: `business_id`, `contact_id` (nullable), `phone_number_from`, `phone_number_to`, `call_status` (default "answered", overwritten by status callback), `twilio_call_sid`, `reply_sent` (bool, deduplication flag). `handle-call-status` reads this table to decide whether to send missed-call SMS. `business_id` confirmed present. RLS policy status unverified (no migration files in repo) — check Supabase dashboard.
- **cron_run_log** — generic run-history table so any scheduled edge function can log a heartbeat row (success/failure + detail jsonb), keyed by function_name. Currently only daily-ticket-digestlogs to it, watched weekly bycron-heartbeat. Confirmed columns: function_name, success, detail, ran_at, id.

---

## Possibly Missing / Unclear — Needs Confirmation

- **Team Members / Permissions** — referenced in Settings UI. Confirm whether this is its own table (e.g. `team_members`, `permissions`, `roles`) or modeled via `profiles` + role column.
- **Business Hours** — referenced in roadmap (`_visibleHourRange()` in calendar). Confirm whether this is a dedicated table, a JSON column on `businesses`, or part of `calendars`.
- **Stripe / Subscription state** — confirmed: billing columns live directly on businesses. Columns written by stripe-webhook: is_paid(bool),client_id(Stripe customer ID),subscription_id, plan(tier: starter|growth|pro, kept after cancellation),subscription_status(Stripe lifecycle: active|trialing|past_due|cancelled — NOT the tier, see Business Rules Plan Gating). No separatesubscriptionsorstripe_customerstable exists. Also present onbusinessesbut unconfirmed/unused:minutes_used_this_monthandincluded_minutes — read by the Billing settings screen (settings_screen.dart, ~line 3048) to render a usage bar, but no edge function or code path in the repo writes to either column. Likely scaffolding for the not-yet-built usage-limit system referenced in business_rules.md Plan Gating.
- **Webhook / inbound SMS logs** — useful for debugging Make scenarios and the SMS booking flow. Confirm whether this is covered by `automation_logs` / `ai_usage_logs` or needs its own table.

---

## Open Items Summary (for AI agents)

When proposing schema changes, treat the following as **known debt, not acceptable patterns to replicate**:
1. `conversation_views` — RLS disabled
2. `messages` — RLS pending re-enable
3. `snippets` and `trigger_links` — `business_id` column confirmed present in app code. RLS policy correctness still needs Supabase dashboard verification.
4. `deals` — `contact_id` FK debt resolved; now uses `lead_id` FK referencing `leads` table.
5. `employee_hub_tokens`, `job_forms`, `job_form_submissions`, `job_types`, `phone_numbers`, `routes`, `team_locations`, `time_entries`, `service_menu_items`, `stripe_connect_accounts` — RLS status unverified, same as `snippets`/`trigger_links`/`call_logs` above (no migration files in repo). Check Supabase dashboard before treating any of these as a template for new tables.
Do not copy these patterns into new tables. New tables must follow Business Rules in full from creation.
