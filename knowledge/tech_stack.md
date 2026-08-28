# Tech Stack

Frontend: Flutter (native, not FlutterFlow)

Database: Supabase

Automation: Supabase (Edge Functions, Database Functions/Triggers, Cron Jobs) — preferred over Make whenever possible

Automation (fallback only): Make — use only when Supabase cannot handle the task

SMS: Twilio

Email: Mailgun

AI / LLM: OpenAI (gpt-4o-mini) — used by ai-chat, receive-sms, receive-email, nexaflow-support, send-invoice, send-quote, send-milestone-invoice, generate-weekly-insight, daily-ticket-digest, and extract-job-form-ai edge functions

OCR: AWS Textract (extract-job-form-ai edge function only) — reads AWS_REGION, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY env vars
  
Payments: Stripe

Hosting: Firebase Hosting (web build) + Supabase (backend/data)
