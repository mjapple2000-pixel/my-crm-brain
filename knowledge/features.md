# Features

Tracks status of features/modules so AI agents know what already exists vs what's planned. Status reflects actual implementation in the Flutter app + Supabase — not aspirational roadmap items. See `database.md` for table details.

Status values: **Built** / **In Progress** / **Planned** / **Not Started**

---

## CRM Core

### Contacts
- **Status:** Built
- **Description:** Standard contact records, linked to conversations, appointments, deals.
- **Tables:** `contacts`
- **Issues:** None known.

### Leads
- **Status:** Built
- **Description:** Incoming lead capture and tracking, status pipeline (`lead_status`), created via forms, AI SMS booking flow, and manual entry.
- **Tables:** `leads`
- **Issues:** Column naming is non-generic (`lead_name`, `lead_email`, `lead_phone`, `lead_status`) — agents must use exact names.

### Deals / Opportunities
- **Status:** In Progress
- **Description:** Sales pipeline deals with color-coded preset tags, forecast value stat pill (`win_probability`), pipeline delete. Won/Lost stats partially implemented.
- **Tables:** `deals`, `pipelines`, `pipeline_stages`, `custom_values`
- **Issues:**  `deals` now uses a proper `lead_id` FK referencing the `leads` table (not a notes-field hack). The UI labels this field "Contact" but the column is `lead_id`. Won/Lost stats incomplete.This screen is complex — always paste current `pipelines_screen.dart` before proposing changes.

### Pipelines / Pipeline Stages
- **Status:** Built
- **Description:** Pipeline definitions and stage configuration, used by Deals/Opportunities screen.
- **Tables:** `pipelines`, `pipeline_stages`
- **Issues:** Stage-to-business scoping via `pipeline_id` join — not yet verified for cross-tenant leakage.

### Tasks
- **Status:** Built
- **Description:** To-do/follow-up items.
- **Tables:** `tasks`
- **Issues:** None known.

### Custom Fields
- **Status:** Built
- **Description:** Custom fields/values attachable to contacts or deals.
- **Tables:** `custom_values`
- **Issues:** None known.

---

## Conversations / Messaging

### Conversations Inbox
- **Status:** Built
- **Description:** Full GHL-style conversations screen — message type filter chips, conversation tags (pills + editor), DND toggle (banner + disabled reply box), search with result count, header checkbox multi-select, right-side contact panel, assignment dropdown, internal notes (Reply/Note tabs), starred conversations, bulk actions (long-press).
- **Tables:** `conversations`, `messages`
- **Issues:** ⚠️ `messages` RLS pending re-enable (tracked exception).

### Saved Filter Views
- **Status:** Built
- **Description:** Save and reuse conversation filter configurations.
- **Tables:** `conversation_views`
- **Issues:** ⚠️ RLS currently disabled, no confirmed `business_id`/`user_id` scoping — tracked exception, needs fix before relying on this for multi-tenant safety.

### Snippets (Canned Replies)
- **Status:** Built
- **Description:** Saved quick-reply text, inserted via click into reply box.
- **Tables:** `snippets`
- **Issues:** `business_id` confirmed present — insert in `snippets_screen.dart` includes it. RLS policy correctness cannot be verified from code alone (no migrations folder); needs Supabase dashboard check.
  
### Active Automations Sidebar
- **Status:** Built
- **Description:** Shows automations a contact/conversation is enrolled in.
- **Tables:** `automation_enrollments`, `automations`
- **Issues:** See Automations section — enrollment table scoping concern.

### Inbound Email
- **Status:** Built
- **Description:** Inbound email via Mailgun → `receive-email` edge function → creates/updates conversation.
- **Tables:** `conversations`, `messages`
- **Issues:** None known beyond general `messages` RLS note above.

---

## Calendar / Appointments

### Calendar Views (Day/Week/Month)
- **Status:** Built
- **Description:** Full GHL-style calendar rebuild with Day/Week/Month views.
- **Tables:** `appointments`, `calendars`
- **Issues:** None known for view rendering itself.

### Appointment Creation
- **Status:** Built
- **Description:** New Appointment dialog with Appointment and Blocked Off Time tabs; Blocked Off Time supports One-Time and Recurring.
- **Tables:** `appointments`, `calendars`
- **Issues:** None known.

### Appointment Editing
- **Status:** Built
- **Description:** Full edit sheet opens when tapping any appointment on the calendar. Editable fields: name, type, status, start/end date+time (with date and time pickers), location, lead name/phone/email, notes, booking source, admin email, calendar assignment, and assigned team member. Save writes all fields back to the `appointments` table. Marking status as Completed fires the `appointment_completed` automation trigger (which drives the review request flow).
- **Tables:** `appointments`
- **Issues:** None known.
  
### Calendar Settings
- **Status:** In Progress
- **Description:** Settings with sub-tabs for Calendars, Groups, Service Menu, Rooms, Equipment. Equipment tab fully implemented.
- **Tables:** `calendars`, `calendar_groups`, `calendar_rooms`, `calendar_equipment`
- **Issues:** Service Menu and other sub-tabs not confirmed complete beyond Equipment.

### Calendar Grid Filtering (Users/Calendars/Groups)
- **Status:** In Progress (UI only, no backend logic)
- **Description:** Right-panel checkboxes for filtering the calendar grid by user/calendar/group.
- **Tables:** `calendars`, `calendar_groups`
- **Issues:** ⚠️ Checkboxes are visual-only — do not actually filter the displayed appointments yet.

### Business Hours
- **Status:** Built
- **Description:** Per-day availability editor (`_AvailabilityHoursEditor`) lives inside the Business Profile settings tab (Settings → Business Profile, under the "Business Hours" group). Saves as a JSON column `availability_hours` on the `businesses` table. The calendar screen (`appointments_screen.dart`) reads `availability_hours` and `slot_duration_minutes` from `businesses` at load time. The `get-available-slots` and `receive-sms` edge functions also read `availability_hours` from the `calendars` table for public booking slot generation.
- **Tables:** `businesses` (availability_hours JSON column, slot_duration_minutes). Also on `calendars` (per-calendar override).
- **Issues:** None known.

### Public Booking Page
- **Status:** Built
- **Description:** Public-facing `/book/:calendarId` page — 3-step flow: (1) customer picks a date and available time slot, (2) enters name, email, phone, (3) sees confirmation screen. Routed unauthenticated in `app_router.dart`. Calls the `submit-booking` edge function (317 lines), which resolves `business_id` server-side from the calendar, writes the appointment and lead rows, then sends a Twilio confirmation SMS.
- **Tables:** `appointments`, `calendars`, `leads`
- **Issues:** None known.

---

## AI / Knowledge Base & Chatbot

### AI Inbound SMS Booking Flow
- **Status:** Built (end-to-end test pending)
- **Description:** `receive-sms` edge function runs a conversational state machine — greets by name, collects email and address, offers 3 appointment slots, books on reply (1/2/3), sends confirmation SMS. Uses GPT-4o-mini.
- **Tables:** `leads`, `contacts`, `appointments`, `messages`, `knowledge_base`
- **Issues:** ⚠️ Deployed but not yet end-to-end tested with a phone number not already present in `leads`. `sent_via_twiml` flag prevents double-send.

### Per-Business Knowledge Base
- **Status:** Built
- **Description:** Per-business knowledge base content used by AI to answer that business's customers and assist with booking.
- **Tables:** `knowledge_base`
- **Issues:** Must confirm `business_id` scoping is enforced on every AI read — do not confuse with `nexaflow_kb`.

### Internal Support Chatbot KB
- **Status:** Built
- **Description:** Internal knowledge base powering the support chatbot for the CRM product itself (not customer-facing).
- **Tables:** `nexaflow_kb`
- **Issues:** No `business_id` by design — internal/global only. Agents must never query this for customer-facing answers.

### AI Chat Widget
- **Status:** Built
- **Description:** Embeddable AI chat widget for a business's website/customers.
- **Tables:** `knowledge_base`, `conversations`, `messages`, `ai_usage_logs`
- **Issues:** None known beyond general KB scoping note.

### AI Usage / Cost Tracking
- **Status:** Built
- **Description:** Logs AI usage for cost tracking per business.
- **Tables:** `ai_usage_logs`
- **Issues:** None known.

### AI Fix Suggestions (Support Tickets)
- **Status:** Built
- **Description:** AI-generated fix suggestions for support tickets, written back via PATCH; surfaced in superuser ticket viewer with daily digest via Make.
- **Tables:** `support_tickets`
- **Issues:** None known.

---

## Automations

### Automation Builder
- **Status:** Built
- **Description:** Visual automation builder.
- **Tables:** `automations`
- **Issues:** None known.

### Automation Execution
- **Status:** Built
- **Description:** `run-automation` edge function executes automations; triggers wired into contacts, appointments, and forms screens.
- **Tables:** `automations`, `automation_logs`
- **Issues:** None known.

### Automation Enrollments
- **Status:** Built (data model concern)
- **Description:** Tracks which contacts are enrolled in which automations; surfaced in conversation sidebar.
- **Tables:** `automation_enrollments`
- **Issues:** `business_id` IS written on insert in both `run-automation` (line ~320) and `handle-trigger-link` edge functions — the column exists. However `process-scheduled-automations` queries `automation_enrollments` without a `business_id` filter (it fetches all active rows due next, then processes them by automation). RLS policy correctness cannot be verified from code alone (no migration files in repo) — check Supabase dashboard to confirm RLS enforces tenant isolation on this table.
---

## Marketing

### Forms / Surveys
- **Status:** Built
- **Description:** Full forms screen with `create_lead_on_submit` toggle and submission deduplication.
- **Tables:** `forms`, `form_submissions`, `leads`
- **Issues:** None known.

### Campaigns
- **Status:** Built
- **Description:** Full SMS and email campaign system. Create/edit/delete campaigns with SMS or Email type, audience filtering by tags/lead_status/source, optional scheduling (`scheduled_at`), and Send Now flow. Send Now calls the `send-campaign` edge function which filters leads by `filter_config`, excludes DND contacts, queues rows into `campaign_contacts`, then `dispatch-campaign-sms` delivers in batches of 10 via Twilio. Status lifecycle: draft → scheduled → sending → sent/active. Smart Lists manager is accessible from the Campaigns top bar.
- **Tables:** `campaigns`, `campaign_contacts`, `leads`, `conversations` (DND check)
- **Issues:** Email campaign type exists in the UI and insert payload (`type: 'email'`, `subject` field) but the `send-campaign` edge function only queues SMS delivery via `dispatch-campaign-sms`. No email dispatch path was found in the codebase — email campaigns can be created and saved but not sent.

### Smart Lists
- **Status:** Built
- **Description:** Saved contact filter/segment presets. Full create/update/delete UI in `smart_lists_manager.dart` (455 lines), wired into the Contacts screen with filters JSON and `business_id` scoping.
- **Tables:** `smart_lists`
- **Issues:** None known.

### Trigger Links (Link Tracking)
- **Status:** Built
- **Description:** Trackable links with click tracking.
- **Tables:** `trigger_links`, `trigger_link_clicks`
- **Issues:** `business_id` confirmed present on insert (`conversations_screen.dart` and `handle-trigger-link` edge function both write it). Full create/list/delete UI exists in the Conversations screen sidebar. RLS policy correctness still needs a Supabase dashboard check — no migration files in repo.

### Email Marketing / Social Planner / Websites/Funnels / Memberships
- **Status:** Not Started
- **Description:** Long-term roadmap items, not yet started.
- **Tables:** None yet.
- **Issues:** N/A.
  
### Reputation Management (Review Requests)
- **Status:** Built
- **Description:** Full flow is live. (1) `reviews_screen.dart` (314 lines) is accessible via the main sidebar navigation under a Reviews entry (gated by `_can('settings')`). It reads and saves `google_review_link`, `facebook_review_link`, and `review_request_delay_minutes` on the `businesses` table. (2) Automation Builder wiring is live — `appointment_completed` trigger and `send_review_request` action both exist in `automations_screen.dart`. Note: a duplicate `_ReviewsSection` widget also exists inside `settings_screen.dart` (line 6357) but is dead code — never referenced in the settings sidebar or switch statement. The functional screen is the standalone `/reviews` route.
- **Tables:** `businesses` (google_review_link, facebook_review_link, review_request_delay_minutes), `automations`, `automation_enrollments`, `automation_logs`
- **Issues:** `_ReviewsSection` inside `settings_screen.dart` (line 6357) is dead code — never wired to a settings tab. Does not block the feature but is a cleanup item.
  
### Missed Call Text Back
- **Status:** Built
- **Description:** Two-function Twilio flow. `handle-inbound-call` receives the call via webhook, looks up the business by `ai_phone_number`, logs to `call_logs`, and dials the owner's real phone (`owner_phone`) for 20 seconds. `handle-call-status` fires on the Dial action callback — if the owner didn't answer (no-answer / busy / failed / canceled), it sends a missed-call SMS to the caller using the business's `missed_call_text_message` template, then logs the outbound SMS to `messages` and creates/updates the `conversations` record so it appears in the inbox.
- **Tables:** `call_logs`, `conversations`, `messages`, `businesses` (reads `ai_phone_number`, `owner_phone`, `missed_call_text_message`, `business_name`)
- **Issues:** `call_logs` table is not documented in `database.md` — needs to be added. Confirm RLS status and whether `reply_sent` deduplication is working correctly end-to-end.

---

## Support / Tickets

### Support Ticket System
- **Status:** Built
- **Description:** Superuser ticket viewer, Make-powered daily AI digest, AI fix suggestions written back via PATCH.
- **Tables:** `support_tickets`
- **Issues:** None known.

### Support Chats
- **Status:** Built
- **Description:** Support conversation threads with staff, separate from customer-facing conversations.
- **Tables:** `support_chats`
- **Issues:** None known.

---

## Billing / Subscriptions

### Stripe Checkout & Webhook
- **Status:** Built
- **Description:** Stripe webhook verified end-to-end — `checkout.session.completed` flips `is_paid` and populates `client_id`/`subscription_id` on the business row. Signature verification active.
- **Tables:** `businesses`
- **Issues:** Billing fields appear to live as columns on `businesses` — confirm in `database.md` open items rather than assuming a separate `subscriptions` table exists.

### Plan Tiers
- **Status:** Built
- **Description:** Starter/Growth/Pro plans with 15-day free trials, via Stripe Price IDs.
- **Tables:** `businesses`
- **Issues:** None known.

### Beta Bypass
- **Status:** Built
- **Description:** Stripe paywall bypassed for businesses flagged `is_beta`.
- **Tables:** `businesses`, `beta_testers`
- **Issues:** None known.

### Plan Upgrades (Proration)
- **Status:** Planned
- **Description:** Stripe plan upgrades using `proration_behavior: 'always_invoice'`.
- **Tables:** `businesses`
- **Issues:** Not yet implemented.

### Welcome / Weekly Check-in Emails
- **Status:** Built
- **Description:** Make scenarios for welcome email (on checkout) and weekly check-in email, active in production.
- **Tables:** `businesses`
- **Issues:** None known.

---

## Team / Permissions

### Team Members
- **Status:** Built (extent unconfirmed)
- **Description:** Team member management with permissions, accessible from Settings.
- **Tables:** Likely `profiles` + role/permission columns — exact model unconfirmed (see `database.md` open items).
- **Issues:** ⚠️ Data model for permissions not fully documented — confirm before building features that depend on role checks.

### Superuser Impersonation
- **Status:** Built
- **Description:** Business switcher in sidebar for superusers, with Mine/All toggle hidden for owners/superusers via `_isOwnerOrSuperuser` flag.
- **Tables:** `superusers`, `businesses`, `profiles`
- **Issues:** Per Business Rules, superuser access bypassing normal scoping should be logged — not yet confirmed if audit logging exists for impersonation sessions.

### Beta Tester Management
- **Status:** Built
- **Description:** Full CRUD by status, resend-invite regenerates token and re-fires webhook, invite email via Make/Gmail. Full funnel confirmed working.
- **Tables:** `beta_testers`, `businesses`
- **Issues:** None known.

---

## Cross-Cutting Open Items

These affect multiple areas above and should be considered before AI agents propose new features that depend on them:

1. **`messages` RLS pending re-enable** — affects Conversations, AI SMS flow, AI Chat Widget.
2. **`conversation_views` RLS disabled** — affects Saved Filter Views.
3. `snippets` and `trigger_links` — `business_id` column confirmed present in app insert code. RLS policy correctness unverified — no migration files in repo, needs Supabase dashboard check.
4. `deals` — `contact_id` FK debt resolved; now uses `lead_id` FK referencing `leads` table.
5. **Calendar grid filtering checkboxes are non-functional** — affects any feature relying on filtered calendar views.
6. **Team/permissions data model unconfirmed** — affects any feature gating by role.
