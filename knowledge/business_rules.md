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
- Known exceptions (tracked, to be resolved): `conversation_views` has RLS disabled; `messages` table RLS is pending re-enable.

## IDs

- All primary keys are UUIDs.
- Never reuse or guess IDs — always generate via `gen_random_uuid()` or equivalent.

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
- Never propose features that conflict with `crm_vision.md` ("Things We Avoid").
- Prefer Supabase-native solutions (Edge Functions, triggers, cron) over Make for new work.
- Flag any schema change that affects RLS policies — these require extra care and explicit review.
- Never write directly to production. All changes go through review/approval first.

## Data Privacy

- Customer data (a business's contacts, leads, conversations, etc.) must never be used to train models or shared across businesses.
- AI agents accessing `knowledge_base` for a business must only access that business's own records.
