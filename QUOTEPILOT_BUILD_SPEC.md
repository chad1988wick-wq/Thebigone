# QuotePilot — Complete Product and Development Specification

## Product
QuotePilot is a mobile-first, multi-tenant SaaS for small contractors and service companies to create professional estimates, manage customers, share quotes, track approvals, and automate follow-ups. Tagline: **Create estimates. Win more jobs.**

## Customers
Independent contractors, handymen, remodelers, painters, landscapers, HVAC, plumbers, electricians, cleaners, flooring installers, and other local service businesses.

## User roles
- Owner: account, billing, settings, users, customers, estimates, analytics.
- Staff (Business plan): assigned customer and estimate operations, without billing access.
- Quote recipient: read-only access to a single estimate through a long random public token; may accept/decline.

## Screens and functions
### Marketing home
Hero with Start Free and See Demo, benefits, workflow, pricing, testimonials only if authentic, FAQ, terms, privacy, contact, login/signup links.
### Authentication
Email/password signup, verification, login/logout, password reset, persistent secure sessions, business onboarding.
### Dashboard
Accepted estimate value, outstanding estimate value, sent/accepted counts, win rate, recent quotes, reminders, New Estimate CTA. Date filters.
### Customers
Create/edit/archive, contact details, company, job address, notes, searchable customer history. Duplicate detection.
### Estimate editor
Choose/create customer; estimate number; job name/address; issue and expiry dates; labor/material/service line items with description, quantity, unit and unit price; discount (fixed or percentage), tax, deposit, notes, terms. Autosave draft; preview, duplicate, PDF, send.
### Estimates list
Search and filter Draft, Sent, Viewed, Accepted, Declined, Expired; sort date/amount; view details, duplicate, edit draft, resend.
### Public quote page
Branded quote with items and totals, contractor contact, expiry, Accept/Decline and optional message. On first open, log view event. Never expose private business data or customer lists. Prevent changing an accepted quote without issuing a revision.
### Follow-ups
Configurable delay (default 3 days), scheduled email reminders, cancel on accepted/declined/expired, audit of sends, idempotent jobs and unsubscribe/compliance handling.
### Settings
Business name, email, phone, logo, address, tax default, payment terms, numbering prefix, timezone, follow-up preferences, team management (Business), billing portal.
### Billing
Free: 3 estimates created/month. Pro: $19/month, unlimited estimates, branding, follow-ups, analytics. Business: $49/month, Pro plus team access and advanced reports. Monthly usage resets by billing/account cycle as specified in code. Show upgrade prompts without losing drafts.

## Recommended architecture
- Next.js App Router + TypeScript + Tailwind CSS, deployed on Vercel.
- Supabase Auth + PostgreSQL + Storage; Row Level Security on every tenant-owned table.
- Stripe Checkout + Billing Portal + verified webhooks.
- Resend for transactional emails; verified sending domain.
- Server-rendered/route-generated PDFs with an appropriate library.
- Vercel Cron or scheduled background worker for follow-ups; authenticated cron endpoint.
- Sentry or comparable error monitoring, privacy-respecting product analytics.

## Data model
All IDs UUID, timestamps timezone-aware, monetary amounts integer cents, currency ISO code, tax rates stored as basis points or decimal with documented precision.

**businesses**: id, owner_user_id, name, email, phone, address, logo_path, timezone, tax_rate_bps, payment_terms, followup_days, created_at, updated_at.
**memberships**: business_id, user_id, role(owner/staff), status, created_at; unique business_id+user_id.
**customers**: id, business_id, name, company, email, phone, billing_address, service_address, notes, archived_at, created_at, updated_at.
**estimates**: id, business_id, customer_id, estimate_number, job_name, job_address, issued_at, expires_at, status, currency, subtotal_cents, discount_cents, tax_cents, deposit_cents, total_cents, notes, terms, public_token_hash, sent_at, viewed_at, accepted_at, declined_at, created_at, updated_at, version.
**estimate_items**: id, estimate_id, description, category, quantity_decimal, unit, unit_price_cents, line_total_cents, sort_order.
**estimate_events**: id, estimate_id, event_type, actor_type, metadata_json, created_at.
**followups**: id, business_id, estimate_id, scheduled_at, status, sent_at, provider_message_id, idempotency_key, created_at.
**subscriptions**: id, business_id, stripe_customer_id, stripe_subscription_id, plan, status, current_period_end, updated_at.
**usage_counters**: business_id, period_start, period_end, estimates_created, unique(business_id,period_start).

Indexes: business_id on tenant tables; estimates(business_id,status,created_at); customers(business_id,name); followups(status,scheduled_at); unique(business_id,estimate_number). Add constraints for nonnegative money and permitted statuses.

## Authorization and safety
- Validate auth and business membership on every private server action/API request.
- Enable PostgreSQL RLS and test cross-tenant denial, including child rows via parent estimate ownership.
- Service-role key server-only; never bundle into browser.
- Use long cryptographically random public tokens, store hashes where practical, rotate on revocation, rate-limit access.
- Recalculate every price and total on server using fixed-precision arithmetic; never trust browser totals or plan entitlements.
- Validate and sanitize user inputs; limit upload MIME/size; private storage and signed URLs where appropriate.
- Verify Stripe webhook signatures against raw request body; deduplicate webhook events and update subscription state server-side.
- Protect cron endpoint with secret; use database locks/idempotency to prevent duplicate emails.
- Log consent/acceptance timestamps and applicable disclosures; obtain legal review for e-signature language and terms.
- Backups, error reporting, security headers, privacy policy, retention/deletion workflow, account deletion.

## API/routes
Public: `/`, `/pricing`, `/login`, `/signup`, `/forgot-password`, `/quote/[token]`, `/privacy`, `/terms`.
Authenticated: `/dashboard`, `/customers`, `/customers/[id]`, `/estimates`, `/estimates/new`, `/estimates/[id]`, `/followups`, `/settings`, `/billing`.
Server endpoints/actions: customer CRUD; estimate CRUD and status transitions; generate PDF; send quote; public view/accept/decline; Stripe checkout, portal and webhook; cron follow-ups. Each action must enforce authorization, validation and plan limits.

## Suggested repository layout
```
quotepilot/
  app/
    (marketing)/page.tsx
    (auth)/login/page.tsx
    (auth)/signup/page.tsx
    (app)/dashboard/page.tsx
    (app)/customers/page.tsx
    (app)/estimates/page.tsx
    (app)/estimates/new/page.tsx
    (app)/followups/page.tsx
    (app)/settings/page.tsx
    (app)/billing/page.tsx
    quote/[token]/page.tsx
    api/stripe/webhook/route.ts
    api/cron/followups/route.ts
  components/
  lib/auth/ lib/db/ lib/stripe/ lib/email/ lib/pdf/ lib/money/
  supabase/migrations/
  public/
  tests/
  .env.example
  .gitignore
  package.json
  README.md
```

## Environment variable template
```
NEXT_PUBLIC_APP_URL=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PRO_PRICE_ID=
STRIPE_BUSINESS_PRICE_ID=
RESEND_API_KEY=
EMAIL_FROM=
CRON_SECRET=
```
Never commit actual values. Configure all production variables in Vercel Project Settings.

## Emails
Estimate delivery: subject `Estimate {number} from {business}`; includes job name, total, expiry and secure View Estimate CTA.
Follow-up: `Following up on estimate {number}` with secure quote CTA.
Accepted notification to contractor: customer, estimate, total, timestamp.
Declined notification to contractor: customer, estimate and optional reason.
Keep templates branded, accessible, with clear sender identity and appropriate opt-out handling.

## Landing-page copy
**Headline:** Create estimates. Win more jobs.
**Subheadline:** Build professional quotes in minutes, send them instantly, and know when customers are ready to move forward.
**Primary CTA:** Start Free
**Secondary CTA:** See How It Works
**Benefits:** Quote faster; impress customers; track every opportunity; automate follow-ups; understand your pipeline.
**Steps:** Add a customer → Build an estimate → Send a secure link → Get approval → Win the job.
**Pricing:** Free $0, Pro $19/mo, Business $49/mo; clearly state plan limits.

## Delivery phases
1. Scaffold Next.js, Supabase Auth, business onboarding and RLS migrations.
2. Build customers, estimates, line-item calculations and dashboard.
3. Add public quote page, acceptance workflow, branded PDF and transactional email.
4. Add Stripe subscriptions, webhook processing, feature gating and billing portal.
5. Add scheduled follow-ups, analytics, team access, monitoring and legal pages.
6. Test, deploy and launch.

## Acceptance tests
- Signup, email verification, login, reset, logout.
- New owner creates business; second business cannot read or modify first business data.
- Customer CRUD; archived customers do not disappear from historical estimates.
- Estimate calculations correct for quantities, tax, discount, zero/negative validation, rounding and deposits.
- PDF totals match stored totals; estimate numbering unique within business.
- Public quote link shows only one estimate and logs view once; accept/decline transitions are idempotent.
- Follow-ups only for eligible estimates; retries do not send duplicate messages.
- Free plan blocks fourth new estimate within same period; paid upgrade restores access.
- Stripe webhook retries and out-of-order events do not corrupt subscription state.
- Mobile layouts, keyboard access, screen-reader labels, loading/error/empty states.
- Build passes typecheck, lint, tests, production deployment, and smoke tests.

## How to deploy the actual SaaS
1. Implement this specification as a Next.js repository (the existing HTML prototype is NOT the production codebase).
2. Push repository to GitHub.
3. In Vercel choose **Add New → Project → Import Git Repository**; use Next.js framework preset.
4. Create Supabase project; apply SQL migrations and configure auth redirect URLs.
5. Create Stripe products and prices; configure webhook URL on deployed domain.
6. Verify domain in Resend and configure sender.
7. Set Vercel environment variables; deploy.
8. Connect domain and test all acceptance cases in production.

## Current state
A self-contained browser prototype exists as `quotepilot_app.html`, with localStorage demo data and functional UI interactions. It does not include real accounts, a database, real PDF generation, email, Stripe, or customer-facing hosted links. The prototype can be deployed as a static demo, but must not be marketed as a completed SaaS until the backend is built and tested.
