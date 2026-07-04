# Tech Stack

Frontend: Flutter (native, not FlutterFlow)

Database: Supabase

Automation: Supabase (Edge Functions, Database Functions/Triggers, Cron Jobs) — preferred over Make whenever possible

Automation (fallback only): Make — use only when Supabase cannot handle the task

SMS: Twilio

Email: Mailgun\n\nAI / LLM: OpenAI (gpt-4o-mini) — used by ai-chat, receive-sms, receive-email, and nexaflow-support edge functions

Payments: Stripe

Hosting: Firebase Hosting (web build) + Supabase (backend/data)
