# my-crm-brain
Master knowledge base and AI operating system for my CRM platform.
# CRM — Knowledge Base

This repository is the central knowledge base and AI operating system for CRM, built by VantageCareTech.

## Purpose

This is not a code repository. It contains the documents that give AI agents the context they need to plan, design, and implement features correctly — without inventing conventions, duplicating work, or violating business rules.

## How It Works

All AI agents (Architect, Developer, QA, etc.) read these files before responding to any request. The files in this repository are the single source of truth for product decisions, database structure, business rules, and feature status.

## Files

| File | Purpose |
|------|---------|
| `crm_vision.md` | What CRM is, who it's for, and what it will never be |
| `tech_stack.md` | Every technology in use and why |
| `database.md` | Every table, column, and known issue |
| `business_rules.md` | Rules every agent must follow — multi-tenancy, RLS, naming, soft deletes |
| `features.md` | Every feature and its current status — Built, In Progress, Planned, Not Started |
| `competitors.md` | GHL and HubSpot analysis and Nexaflow positioning |

## Agent Rules

- Read all six files before responding to any request
- Never propose changes that conflict with `business_rules.md`
- Never duplicate something already marked Built in `features.md`
- Always flag conflicts or gaps rather than silently working around them
