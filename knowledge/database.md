# Database (Supabase)

This is a high-level list of tables. For exact columns, check Supabase directly — this file tracks purpose, multi-tenancy status, and known issues, not full schema details.

Project ref: `rllriopqojaraceytdno` (us-east-1)

---

## Core / Multi-Tenant

- **businesses** — each customer account using the CRM. The root tenant record; every business-scoped table should ultimately resolve to a row here via business_id. Also confirmed present: default_tax_rate(used when creating a new quote/invoice),stripe_connect_id, stripe_connect_ready, stripe_connect_onboarded (Stripe Connect status, used by Invoicing to gate payment collection), dedicated_email (the business's unique inbound-email address; `receive-email` matches inbound mail to a business by this column, and `outbound-email-send` uses it as Reply-To so customer replies land back in Conversations instead of the owner's personal inbox), pdf_settings (jsonb — brand color, accent color, header layout/style, logo size, footer text, disclaimer text, and show/hide toggles for phone/email/website/page numbers/generated date; edited from Settings → Documents and read by `generate-job-form-pdf`).
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

- **conversations** — message threads. `business_id` required. Also confirmed present (all from `receive-email/index.ts`): `ai_enabled` (bool — set `false` to pause AI on this conversation), `flagged_for_abuse` (set when a conversation hits 45 AI replies in a rolling 24h — the "abuse circuit breaker", line 769), `flagged_for_beta_cap` (set when a beta business hits its usage cap with no card on file, line 799), `ai_reply_count_24h` / `ai_reply_window_reset_at` (drive the abuse breaker, lines 762/1057), `ai_reply_mode_override` (per-conversation override of the business's `email_ai_reply_mode` — `'autopilot'` or `'draft'`, line 1070), `relevance_score` / `relevance_checked_at` (EM-03 — a 0-1 heuristic score of how likely an inbound email is a genuine customer inquiry vs. an automated/notification sender, written by `receive-email` and `gmail-inbound-webhook`; when the score falls below the tunable threshold in `platform_settings`, the AI does not reply).
- **conversation_views** — saved filter views for the Conversations screen. RLS confirmed enabled with a real `business_id`-scoped policy as of 9/16 database check. The table does have a `user_id` column, but the current policy only scopes by `business_id` — any staff member at a business can see all of that business's saved views, not just their own.
- **messages** — individual messages. `business_id` required. RLS confirmed enabled with a real `business_id`-scoped policy as of 9/16 database check.
- **support_chats** — support conversations with staff. Scoped to business + superuser visibility.
- **support_tickets** — support ticket records. Scoped to business + superuser visibility.
- **snippets** — saved canned-reply text. `business_id` confirmed present in insert code (`snippets_screen.dart`). RLS confirmed enabled with a `business_id`-scoped policy as of 9/16 database check.

---

## AI / Knowledge Base

- **nexaflow_kb** — internal knowledge base powering the support chatbot for the CRM product itself. **No `business_id` — internal/global only.** AI agents must never mix this with `knowledge_base`.
- **knowledge_base** — per-business knowledge base used by AI to answer that business's customers and book appointments. **Must have `business_id`**, and AI agents must only query a business's own rows. Do not confuse with `nexaflow_kb`.
- **ai_usage_logs** — tracking AI usage/costs. `business_id` required for per-business cost attribution.

---

## Calendar / Appointments

- **appointments** — scheduled appointments. `business_id` required. Also confirmed present: `latitude`, `longitude` (set by the `geocode-location` edge function from a typed address; used for map/route display).
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

- **job_expenses** — cost entries logged against a job (an appointment or a deal). `business_id` required. Confirmed columns from insert code (`log-job-expense` edge function): `business_id`, `appointment_id` (nullable), `deal_id` (nullable), `category_id` (FK → `expense_categories`, required — validated server-side against the business), `expense_type` (labor|material|subcontractor|other, still present alongside `category_id`), `amount_cents`, `description`, `logged_by_profile_id`, `logged_at`, `receipt_photo_path` (nullable — set by the `upload-expense-receipt` edge function, which stores the file in the `job-expense-receipts` Storage bucket; called from both `appointments_screen.dart` and `pipelines_screen.dart`), `deleted_at` (referenced by `compute-job-cost-snapshot` as a soft-delete filter). Gated behind Growth plan via `check_plan_feature` RPC (`job_costing` feature).
- **expense_categories** — a business's custom expense categories (e.g. "Fuel", "Materials"), managed from Settings (\_ExpenseCategoriesSectionState`, `settings_screen.dart` line 10237). `business_id` required. Confirmed columns: `business_id`, `name`, `is_active`, `deleted_at`. Referenced by `job_expenses.category_id`.
- **job_revenue_snapshots** — computed profit/loss snapshot per job (appointment or deal), upserted by the `compute-job-cost-snapshot` edge function on conflict of `appointment_id` or `deal_id`. Confirmed columns: `business_id`, `appointment_id`, `deal_id`, `total_expenses_cents`, `total_revenue_cents`, `gross_profit_cents`, `profit_margin_pct`, `job_type`, `snapshotted_at`. Revenue side is pulled from the `invoices` table (see Open Questions — `invoices` itself is not yet documented here).

---

  ## Jobs / Quotes / Invoicing

  - **quotes** — customer-facing price quotes. `business_id` required. ⚠️ Primary key is NOT `bigint` — see IDs exception in Business Rules. Confirmed columns: `business_id`, `contact_id` (FK → `leads`, same "labeled Contact, actually a leads FK" pattern as `deals`), `quote_number` (format `Q-###`), `status` (draft|sent|approved|declined), `job_title`, `expires_at`, `notes`, `tax_rate`, `subtotal`, `tax_amount`, `total`, `sent_at`, `approved_at`, `approved_via`, `updated_at`, `created_at`, `deleted_at`.
  - **invoices** — customer-facing invoices, created fresh or converted from a `quotes` row. `business_id` required. ⚠️ Primary key is NOT `bigint` — see IDs exception in Business Rules. Confirmed columns: `business_id`, `contact_id` (FK → `leads`), `quote_id` (nullable FK → `quotes`, set when converted from a quote), `invoice_number` (format `INV-###`), `status` (approved|paid, at minimum), `amount_due`, `subtotal`, `tax_amount`, `tax_rate`, `job_title`, `notes`, `due_date`, `payment_link_id` (FK → `payment_links`), `is_progress_billed` (bool — flags an invoice as milestone/progress-billed, gates `invoice_milestones` processing), `paid_at`, `stripe_checkout_session_id` (set by `create-invoice-payment` when a whole-invoice Stripe Checkout session is created; matched by `stripe-connect-webhook` when the session completes), `updated_at`, `created_at`, `deleted_at`.
  - **invoice_milestones** — individual progress-billing milestones on an `is_progress_billed` invoice, auto-sent when due via `process-due-milestones` → `send-milestone-invoice` (AI-drafted send message via GPT-4o-mini, same pattern as `send-invoice`). `business_id` required. Confirmed columns: `business_id`, `invoice_id` (FK → `invoices`), `label`, `due_date`, `amount_due` (the dollar amount for this one milestone — separate from the invoice's own `amount_due`), `status` (pending|ready_to_bill|sent, at minimum), `stripe_checkout_session_id` (set by `create-invoice-payment` when this milestone is paid individually), `updated_at`, `deleted_at`.
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
- **referrals** — a tracked referral: an existing lead (`referrer_id`/`referrer_type`) who referred a new lead (`referred_lead_id`). `business_id` required. Confirmed columns: `business_id`, `referrer_type` (`'lead'`, with a code comment noting a future `'contact'` B2B type), `referrer_id`, `referred_lead_id`, `deleted_at`. Written by `handle-referral-signup`.
- **referral_reward_settings** — a business's referral-program reward configuration (percentage of paid invoice, minimum job size to qualify). \business_id` required, unique per business (upserts use `onConflict: 'business_id'`). Confirmed columns (from `settings_screen.dart` `_rewardDb` upserts, lines 1206–1354): `business_id`, `mode`, `default_amount`, `default_reward_type`, `default_reward_basis`, `minimum_payout_threshold`.
- **referral_reward_recipients** — tracks who has earned a referral reward and at what amount. \business_id` required. Confirmed columns (from `settings_screen.dart` insert/update code, lines 12315–12328): `id`, `business_id`, `referrer_type`, `referrer_id`, `amount`, `reward_type`, `reward_basis`, `deleted_at`. Note: NOT read/written in `reporting_screen.dart` — that screen only reads the `referrals` table for its leaderboard.

---

## Employee Hub / Field Ops

- **employee_hub_tokens** — token-based auth for the public, unauthenticated Employee Hub (`/hub/:token`, same pattern as the Client Portal's `client_access_token`). Confirmed columns: `token`, `profile_id`, `business_id`, `revoked_at` (set on revoke/reissue via `resend-employee-hub-link`).
- **job_forms** — form templates (checklists, inspection forms, etc.) built by staff via the \"Job forms\" tab in the Jobs Hub, or generated by the AI Form Recreation flow. business_idrequired. Confirmed columns:business_id, name, form_type, fields(jsonb — array of{id, type, label, required}, plus optionsfor select fields),requires_signature, deleted_at, recreation_mode('standard' | 'visual_recreation' — set by AI Form Recreation),background_pages(jsonb — page image references used to render the original form as a background whenrecreation_modeisvisual_recreation).
- **job_form_submissions** — an instance of a job form attached to a specific appointment, filled out by a crew member through the Employee Hub. business_idrequired. Confirmed columns:business_id, job_form_id, appointment_id, status(not_started|in_progress, at minimum),answers(jsonb),photo_urls, completed_by_profile_id, deleted_at, signature_url, signed_by_name, signed_at, pdf_url, submission_label, extra_pages, rendered_page_urls, view_token (lazily created the first time a submission is emailed to a customer via the new `email-job-forms` edge function — permanent once set, never regenerated; confirmed at `email-job-forms/index.ts` line 309).
- **job_types** — a business's job categories, managed from the "Service Menu" area of Calendar Settings. `business_id` required. Confirmed columns: `business_id`, `name`, `is_active`, `deleted_at`.
- **phone_numbers** — Twilio numbers provisioned per business via `provision-phone-number`. `business_id` required. Confirmed columns: `business_id`, `twilio_sid`, `phone_number`, `friendly_name`, `deleted_at`.
- **routes** — a day's optimized stop order for a crew member, computed by `optimize-route`. `business_id` required. Confirmed columns: `business_id`, `assigned_user_id`, `route_date`, `stops` (jsonb — ordered `{appointment_id, sequence, lat, lng}` array), `optimized_at`.
- **team_locations** — live GPS position per crew member, upserted by `update-team-location` (called from the Employee Hub) and read by `routes_screen.dart` to plot crew on the map. Confirmed columns: `user_id`, `business_id`, `latitude`, `longitude`.
- **time_entries** — clock-in/clock-out records. `business_id` required. Confirmed columns: `business_id`, `appointment_id` (nullable), `user_id`, `clocked_in_at`, `clock_in_lat`, `clock_in_lng`, `status` (active, at minimum), `deleted_at`. Written by `clock-in-out`; also read directly by `main_layout.dart` and `appointments_screen.dart` to show an active-clock indicator app-wide.
- **service_menu_items** — services offered for booking (distinct from `service_library`, which feeds quote/invoice line items — see Open Questions). `business_id` required. Confirmed columns: `business_id`, `name`, `description`, `duration_minutes`, `price`, `calendar_ids`, `is_active`, `updated_at`. Full CRUD lives in the "Service Menu" tab of the Calendar Settings dialog inside `appointments_screen.dart` — not in `settings_screen.dart`.
- **stripe_connect_accounts** — a second Stripe Connect record, separate from the `stripe_connect_id`/`stripe_connect_ready`/`stripe_connect_onboarded` columns already documented on `businesses` above (both systems are intentionally in use, per Mike — the split rationale isn't visible from code). Confirmed columns: `business_id`, `stripe_account_id`, `onboarding_complete`, `charges_enabled`, `payouts_enabled`, `deleted_at`. Written by `stripe-connect-onboard`/`stripe-connect-webhook`; read by `generate-payment-link`. ⚠️ This table (\stripe_connect_accounts`) still has no confirmed `lib/` UI entry point. Resolved: `create-connect-account` and `get-connect-status` are now called from `settings_screen.dart`'s Payment Options section (lines 5675, 5721), and `get-stripe-balance` is called from `jobs_overview_screen.dart` (line 421) — all three read/write the `businesses`-column system (`stripe_connect_id`/`stripe_connect_ready`/`stripe_connect_onboarded`), NOT this table. `get-express-dashboard-link` still has no `lib/` caller.
- **accounting_connections** — a business's connection to an accounting provider (currently QuickBooks Online only). `business_id` required. Confirmed columns: `business_id`, `provider` (`'quickbooks'`), `connection_status` (`connected`|`error`, at minimum), `access_token_secret_id`, `refresh_token_secret_id` (both reference Supabase Vault secrets via `qb_vault_read_secret` RPC — tokens are never stored in plaintext), `token_expires_at`, `qb_realm_id`, `deleted_at`. Written by `quickbooks-oauth-connect`; read by `quickbooks-sync-contact` and `quickbooks-sync-invoice`. UI: Settings → Payment Options, \_PaymentOptionsSectionState` (`settings_screen.dart` class starts line 5541; QuickBooks connect logic at lines 5560-5650).
- **accounting_customer_links** — maps a `leads` row to its corresponding QuickBooks customer record. `business_id` required. Confirmed columns: `business_id`, `lead_id` (FK → `leads`), `provider`, `qb_customer_id`, `updated_at`, `deleted_at`. Written/read by `quickbooks-sync-contact`.
- **accounting_invoice_links** — maps an \invoices` row to its corresponding QuickBooks invoice record. `business_id` required. Confirmed columns (from `quickbooks-sync-invoice/index.ts` lines 415-421 and 588-598): `business_id`, `invoice_id` (FK → `invoices`), `provider`, `qb_invoice_id`, `deleted_at`, `updated_at`. Unique on `(invoice_id, provider)`.
- **accounting_sync_log** — audit log of each sync attempt to the accounting provider. `business_id` required. Confirmed columns: `business_id`, `entity_type` (`'contact'`, at minimum — invoice sync likely adds `'invoice'`, unconfirmed), `local_id`, `qb_id`, `direction` (`'to_qb'`), `status` (`success`|`failed`), `error_message`, `synced_at`. Written by `quickbooks-sync-contact` (and presumably `quickbooks-sync-invoice`, unconfirmed).

---

## AI Form Recreation & Template Library

- **job_form_ai_drafts** — an in-progress AI form recreation session (photo/PDF upload → OCR → GPT → confirmed job form). `business_id` required. Confirmed columns: `business_id`, `status` ('processing'|'confirmed', at minimum), `created_by_profile_id`, `is_blank_template`, `source_file_url`, `source_page_urls`, `confirmed_job_form_id` (FK → `job_forms`, set on confirm), `updated_at`. Source files live in the `job-form-ai-sources` Storage bucket.
- **form_templates** — a shared library entry publishing one business's `job_forms` row for reuse (optionally across businesses). `business_id` required (owning business). Confirmed columns: `business_id`, `title`, `description`, `source_job_form_id` (FK → `job_forms`), `min_tier` (present for a possible future per-template plan gate — confirmed by code comment as not currently used by any gate), `deleted_at`. RLS gives authenticated users read-only access; all writes go through the `job-form-editor` edge function.
- **form_template_tags** — join table linking a `form_templates` row to one or more `form_tags`. Confirmed columns: `form_template_id`, `tag_id`.
- **form_tags** — the pool of tags available for labeling templates in the library. Confirmed columns: `id`, `name`, `deleted_at`.
- **job_form_photo_attachments** — GPS/location-tagged photos attached to a marker placed during AI Form Recreation (distinct from the `photo_urls` field-answer array on `job_form_submissions`, which is unrelated). Confirmed columns: `submission_id` (FK → `job_form_submissions`), `marker_id`, `storage_path`, `latitude`, `longitude`, `captured_at`, `created_at`, `deleted_at`.

---

## Email (Gmail + Outlook Sync)

- **oauth_connections** — a business's connected third-party OAuth account. Two providers now confirmed: `'gmail'` (from `gmail-oauth-connect/index.ts` insert, lines 168–180) and `'microsoft'` (from `microsoft-oauth-callback/index.ts` insert, line 184 — same shape, different provider value). Confirmed columns: `business_id`, `provider`, `connected_account_email`, `access_token_secret_id`, `refresh_token_secret_id` (Supabase Vault secret refs), `token_expires_at`, `connection_status` ('active', at minimum), `deleted_at`, `updated_at`.
- **email_sync_subscriptions** — Microsoft-only parallel to `gmail_sync_state` below (the two tables are separate and coexist; `microsoft-*` functions never touch `gmail_sync_state` and vice versa). Tracks the Microsoft Graph push subscription used to notify `microsoft-graph-webhook` of new mail. Confirmed columns (`microsoft-oauth-callback/index.ts` lines 260–263): `oauth_connection_id`, `business_id`, `graph_subscription_id`, `client_state` (echoed back on every Graph push and must be verified — same role as Mailgun's HMAC signature on `receive-email`), `expiration_datetime`, `last_renewed_at`, `deleted_at`, `updated_at`. Renewed by `renew-graph-subscriptions` (Graph mail subscriptions expire after ~3 days).
- **gmail_sync_state** — tracks Gmail History API sync position per connection. Confirmed columns (from `gmail-oauth-connect/index.ts` and `gmail-inbound-webhook/index.ts`): `oauth_connection_id` (FK → `oauth_connections`), `history_id`, `updated_at`.
- **pubsub_dedup** — dedup table for Google Pub/Sub push notifications (prevents double-processing the same Gmail push message). Confirmed column: `message_id` (from `gmail-inbound-webhook/index.ts` line 413).
- **platform_settings** — a global (non-per-business) key/value settings table. Confirmed columns: `key`, `value`, `updated_at`, `updated_by`. Currently holds one row (`key = 'email_relevance_threshold'`, default 0.9) read by `receive-email` and `gmail-inbound-webhook` (EM-03) to decide whether an inbound email is too automated/low-relevance to get an AI reply — see `conversations.relevance_score` above. Editable by superusers from the new `/platform-settings` screen (`platform_settings_screen.dart`).
- **unmatched_inbound_emails** — logs inbound emails that couldn't be matched to any business's `dedicated_email` (e.g. a misconfigured Mailgun domain), so nothing silently disappears. Confirmed columns: `id`, `raw_to_address`, `raw_from_address`, `subject`, `received_at`, `deleted_at`. Written by `receive-email` (line 544); viewed read-only by superusers at the new `/unmatched-emails` screen (`unmatched_emails_screen.dart`).

---

## PTO / Overtime / Payroll

- **pto_requests** — an employee's paid-time-off request. Confirmed columns (from `employee_pto_screen.dart` insert, lines 365–372): `business_id`, `profile_id`, `start_date`, `end_date`, `hours_requested`, `status` ('pending', at minimum — approve/deny logic lives in `pto_requests_screen.dart`), `note`. Read by `get-timesheets`, `export-timesheets-pdf`, `quickbooks-sync-hours`, and `notify-pto-request` (which emails the owner on new submission).
- **overtime_rules** — a business's daily/weekly overtime thresholds. Confirmed columns (from `settings_screen.dart` `_PayrollSettingsSectionState`, lines 11307–11780): `business_id`, `daily_threshold_hours`, `daily_ot_enabled`, `weekly_threshold_hours`, `weekly_ot_enabled`, `updated_at`, `deleted_at` (code comment at line 11747 confirms a partial unique index on `business_id WHERE deleted_at IS NULL`).
- **time_entry_breaks** — break periods logged against a `time_entries` row, used by `check-overtime-thresholds` to compute actual worked hours. Full column set not yet confirmed — flagging for next pass.
- **overtime_notifications** — logs sent overtime-threshold alerts (dedup/audit trail for `check-overtime-thresholds`). Full column set not yet confirmed — flagging for next pass.
- **pay_periods** — a business's payroll period definitions, read/written by `decide-timesheet` and `submit-timesheet`. Full column set not yet confirmed — flagging for next pass.
- **employee_pay_period_status** — per-employee submission/approval status for a pay period. Full column set not yet confirmed — flagging for next pass.
- **pay_period_status_history** — audit trail of status changes on `employee_pay_period_status`. Full column set not yet confirmed — flagging for next pass.
- **team_member_provider_mappings** — maps a `profiles` row to its corresponding QuickBooks employee record (same pattern as `accounting_customer_links`, but for staff instead of leads). Written/read by `quickbooks-sync-hours`. Full column set not yet confirmed — flagging for next pass.

---

## Owner Notifications

- No new table — this is implemented as two Postgres trigger functions defined directly in a migration (`supabase/migrations/20260811210530_...sql`, updated by `20260811212118_...sql`): `notify_owner_new_lead()` (fires `AFTER INSERT` on `leads`) and `notify_owner_new_appointment()` (fires `AFTER INSERT` on `appointments`). Both call the `notify-owner` edge function via `net.http_post` with an `x-cron-secret` header, passing lead/appointment details so the owner gets notified automatically.

---

## Seat / Beta Billing

- **business_seats_live** — appears to be a view/live variant feeding seat-based billing overage, read by `sync-seat-overage` alongside `businesses` and `cron_run_log`. Not yet confirmed whether it's a table or view, or its full column set — flagging for next pass, same treatment as `business_usage_live` above.
- **system_alert_log** — a general internal alert log, written to by `decide-timesheet` (line 26). Full column set not yet confirmed — flagging for next pass.

---

## Billing / Usage

- **business_usage** — per-business, per-month AI-message usage counter, replacing the dead `minutes_used_this_month`/`included_minutes` scaffolding previously noted in Possibly Missing/Unclear below (that item can now be removed — see Business Rules). Confirmed columns (from `report-ai-overage`): `id`, `business_id`, `period_start`, `ai_messages_used`, `ai_messages_included`, `overage_units_reported`. Read via a `get_business_usage_summary` Postgres RPC (Settings → Billing, `_BillingSectionState._loadUsage`, `settings_screen.dart` ~line 4266) which returns \ai_messages_used`, `ai_messages_included`, `is_overage` (`_loadUsage` now at `settings_screen.dart` line 4515, not 4266).
- **business_usage_live** — appears to be a view/live variant of `business_usage` (selected in `report-ai-overage` instead of the base table, joined out to `businesses.client_id`) but I have not confirmed whether it's a table or a database view, or its full column set. Flagging for next pass.
- **dashboard_insights** — stores each business's AI-generated weekly insight. `business_id` required (inferred). Written by `generate-weekly-insight`; full column list not yet confirmed. Flagging for next pass.

---

## Misc

- **trigger_links** — trackable links created per-business for campaigns. `business_id` confirmed present in insert code (`conversations_screen.dart`, `handle-trigger-link` edge function). RLS confirmed enabled with a `business_id`-scoped policy as of 9/16 database check.
- **call_logs** — records of inbound phone calls. Written by `handle-inbound-call` edge function. Columns confirmed from insert code: `business_id`, `contact_id` (nullable), `phone_number_from`, `phone_number_to`, `call_status` (default "answered", overwritten by status callback), `twilio_call_sid`, `reply_sent` (bool, deduplication flag — verified atomic via `.eq('reply_sent', false)` in `handle-call-status`, so the double-send race is closed). `handle-call-status` reads this table to decide whether to send missed-call SMS. RLS confirmed enabled with a `business_id`-scoped policy as of 9/16 database check.
- **trigger_link_clicks** — click tracking on trigger links. Scoped via `trigger_link_id` → business; same concern as above — if `trigger_links` lacks `business_id`, this table inherits the gap.
- **cron_run_log** — generic run-history table so any scheduled edge function can log a heartbeat row (success/failure + detail jsonb), keyed by function_name. Currently only daily-ticket-digestlogs to it, watched weekly bycron-heartbeat. Confirmed columns: function_name, success, detail, ran_at, id.

---

## Possibly Missing / Unclear — Needs Confirmation

- **Business Hours** — referenced in roadmap (`_visibleHourRange()` in calendar). Confirm whether this is a dedicated table, a JSON column on `businesses`, or part of `calendars`.
- **Stripe / Subscription state** — confirmed: billing columns live directly on businesses. Columns written by stripe-webhook: is_paid(bool),client_id(Stripe customer ID),subscription_id, plan(tier: starter|growth|pro, kept after cancellation),subscription_status(Stripe lifecycle: active|trialing|past_due|cancelled — NOT the tier, see Business Rules Plan Gating). No separatesubscriptionsorstripe_customerstable exists. The `minutes_used_this_month`/`included_minutes` scaffolding previously noted here is gone — no references remain anywhere in the codebase as of this sync. It's been replaced by the `business_usage`/`business_usage_live` tables and `get_business_usage_summary` RPC (see Billing / Usage section above).
- **Webhook / inbound SMS logs** — useful for debugging Make scenarios and the SMS booking flow. Confirm whether this is covered by `automation_logs` / `ai_usage_logs` or needs its own table.

---

## Open Items Summary (for AI agents)

When proposing schema changes, treat the following as **known debt, not acceptable patterns to replicate**:
1. `conversation_views` — RLS disabled
2. `messages` — RLS pending re-enable
3. `snippets` and `trigger_links` — `business_id` column confirmed present in app code. RLS policy correctness still needs Supabase dashboard verification.
4. `deals` — `contact_id` FK debt resolved; now uses `lead_id` FK referencing `leads` table.
5. `employee_hub_tokens`, `job_forms`, `job_form_submissions`, `job_types`, `phone_numbers`, `routes`, `team_locations`, `time_entries`, `service_menu_items`, `stripe_connect_accounts` — RLS status unverified, same as `snippets`/`trigger_links`/`call_logs` above. Migration files now exist in the repo but none touch these tables. Check Supabase dashboard before treating any of these as a template for new tables.
Do not copy these patterns into new tables. New tables must follow Business Rules in full from creation.
