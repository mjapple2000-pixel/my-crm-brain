# CRM Vision

## Mission

Give home service businesses (roofing, HVAC, plumbing, and similar trades) a CRM that books and responds to customers automatically — even when no one is available to pick up the phone — without the setup overhead of GoHighLevel or the cost and complexity of HubSpot.

"Easier than HubSpot, less complicated than GoHighLevel" still holds as a positioning statement, but the sharper version is: **GHL's UX with an AI front desk built in from day one.** The AI inbound SMS booking flow and per-business knowledge base aren't add-ons bolted onto a generic CRM — they're the reason the product exists.

## Target Customers

Not "small businesses" broadly — specifically businesses that:

- **Take appointments and currently miss calls.** Home service businesses (roofing, HVAC, plumbing, etc.) where the owner or a small team is often on a job site, in a truck, or unavailable, and every missed call is a missed lead.
- **Get repetitive customer questions** that an AI with the right knowledge base can answer correctly (pricing ranges, service areas, availability, what's included in a visit) — freeing the owner from answering the same texts all day.
- **Need appointments booked without a human in the loop.** The AI SMS flow (greet → collect name/email/address → offer slots → book → confirm) is built for businesses where "someone will call you back" is a lost sale.
- **Are not large enough to justify GHL's setup time or HubSpot's price.** Solo operators and small teams (1–15 people) who need this working in days, not weeks, and who will not hire a "CRM admin."

If a business doesn't take appointments, doesn't get inbound calls/texts from customers, or already has a full-time dispatcher/CSR fielding everything, they're not the core customer — they may still get value from the CRM core (pipelines, contacts, tasks), but they won't benefit from what makes this product different.

## Core Principles

- **Simple** — Every screen should be usable without training. This is why we rebuilt Calendar, Conversations, and Pipelines to match GHL's proven UX patterns rather than inventing new ones: familiar beats novel for a business owner with five minutes between jobs.
- **Fast** — Setup-to-value should be measured in minutes, not days. The Launchpad (onboarding checklist) and beta paywall bypass exist so a new business can get the AI SMS flow and knowledge base live before they've finished exploring the rest of the app.
- **Affordable** — Pricing (Starter $97/mo, Growth $297/mo, Pro $497/mo, all with a 15-day free trial on whichever plan the customer chooses) is set to be a no-brainer replacement for "a part-time answering service" or "a missed-call problem." Each tier unlocks progressively more features — Starter covers the core CRM and AI booking workflow; Growth and Pro unlock the full AI suite, advanced automations, and priority support. Customers have a clear reason to upgrade: the features that matter most are on higher tiers, not buried in fine print.

- **AI Assisted** — AI isn't a chatbot widget tacked onto a contact form. It's wired into the actual workflow that drives revenue: the SMS booking state machine reads from the business's own knowledge base, books real appointments, and writes back into the same `conversations`/`messages`/`appointments` tables a human would use. AI usage is also tracked per business (`ai_usage_logs`) so cost stays predictable as this scales.

## Things We Avoid

- **Feature bloat from the GHL/HubSpot feature-parity checklist.** We track competitor features in `competitors.md`, but "they have it" is not a reason to build it. Specifically:
  - **Memberships, Websites/Funnels, Social Planner** — intentionally **Not Started and staying that way** for now. These are full product categories on their own (community platforms, page builders, social schedulers) that would pull focus from the AI-booking differentiator and are poorly suited to a small operator's actual workflow. Revisit only if customers consistently ask for them as a dealbreaker.
  - **Email Marketing as a full platform** (drag-and-drop builders, A/B testing, deliverability tooling) — **Not Started, low priority.** A simple campaign/broadcast tool may eventually be worth building, but a Mailchimp-style platform is explicitly out of scope, for now.
  - **White Label** — **Not Started, intentionally deferred.** This is an agency/reseller feature for a different customer segment (marketing agencies managing client accounts) than our target (the business owner themselves). Revisit only if we pivot toward an agency channel.
- **Items that are simply "not yet built" and should NOT be read as intentional avoidance** (these remain on the roadmap): Reputation Management, Missed Call Text Back, Meta Ads integration, Public Booking Page, Email Marketing (basic version). These are natural extensions of the AI-booking core and are expected to ship eventually — they're just sequenced behind higher-priority items like appointment editing and SMS flow testing.
- **Complicated setup.** No multi-day onboarding calls, no required integrations before the product is useful. If a feature requires a business to configure something technical (webhooks, API keys, custom domains) before getting value, it needs a guided/automated setup path or it doesn't ship.
- **Enterprise complexity.** No org-chart-style permission systems, no multi-business/franchise hierarchies beyond what superuser impersonation already covers, no approval workflows. Team/permissions stays simple (owner + team members with basic role-based access) — not a configurable RBAC system.
- **Generic AI chatbot as the differentiator.** A chat widget that answers FAQs is table stakes and easy to copy. We don't market or prioritize the AI Chat Widget in isolation — its value comes from sharing the same knowledge base and data model as the SMS booking flow, which is much harder to replicate.

## What Makes Nexaflow Different

The differentiator isn't "AI chat" — it's that the AI is wired directly into the booking pipeline using the business's own knowledge base. A customer can text the business's number, get recognized and greeted by name, ask a real question and get an answer specific to that business, and walk away with a booked, confirmed appointment — all without a human touching it, and all recorded in the same `conversations`, `messages`, and `appointments` tables the team already works from. Competitors bolt AI chat onto a contact form; we built the contact form *into* the AI flow.
