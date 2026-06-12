# Database (Supabase)

This is a high-level list of tables. For exact columns, check Supabase directly — this file tracks purpose, not schema details.

## Core / Multi-Tenant
- businesses — each customer account using Nexaflow
- profiles — user profiles (linked to businesses)
- users — auth/user records
- superusers — Nexaflow admin/staff accounts
- beta_testers — beta program tracking

## CRM Core
- contacts — customer's contacts/customers
- leads — incoming leads
- deals — sales pipeline deals
- pipelines — pipeline definitions
- pipeline_stages — stages within pipelines
- custom_values — custom fields/values for contacts or deals
- tasks — to-do items/follow-ups

## Conversations / Messaging
- conversations — message threads
- conversation_views — read/view tracking (unrestricted)
- messages — individual messages
- support_chats — support conversations with Nexaflow
- support_tickets — support ticket records
- snippets — saved canned text replies (unrestricted)

## AI / Knowledge Base
- nexaflow_kb — internal knowledge base for the Nexaflow chatbot (helps run/support the CRM itself)
- knowledge_base — per-business knowledge base, used by AI to answer that business's customers and book appointments
- ai_usage_logs — tracking AI usage/costs

## Calendar / Appointments
- appointments — scheduled appointments
- calendars — calendar definitions
- calendar_rooms — bookable rooms
- calendar_equipment — bookable equipment
- calendar_groups — grouping of calendars

## Automations
- automations — automation definitions
- automation_enrollments — contacts enrolled in automations (unrestricted)
- automation_logs — automation run history

## Marketing
- campaigns — marketing campaigns
- campaign_contacts — contacts targeted in campaigns
- forms — lead capture forms
- form_submissions — form submission records
- smart_lists — saved contact filters/segments

## Misc
- trigger_links — trackable links (unrestricted)
- trigger_link_clicks — click tracking on trigger links (unrestricted)
