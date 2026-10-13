# Psychology session booking — PostgreSQL edition

A static booking page plus Vercel serverless API for a small online therapy practice. The source of truth for bookings is PostgreSQL; Redis is **not** required. Use Neon Postgres Free as the recommended starting point, or any compatible managed PostgreSQL provider.

> Important: this is an implementation starter, not a guarantee of legal/GDPR compliance. Review consent text, retention, refunds/cancellations, access permissions and the handling of sensitive client details before using it with real clients.

## What happens after a client pays

1. The client chooses a slot and clicks **Pay**. The API validates the details and consent server-side, checks Google Calendar and atomically inserts the booking plus a temporary slot hold in PostgreSQL.
2. The API creates a Stripe Checkout Session with a server-controlled amount/currency and a stable Stripe idempotency key.
3. Stripe sends a signed webhook to `/api/stripe-webhook`. The API verifies the signature and the actual session payment status before fulfilling the booking.
4. The backend creates a Google Calendar event with a deterministic event ID and `sendUpdates: all`. The temporary hold is released only after Calendar owns the time block.
5. Resend sends a separate email to the therapist and a confirmation to the client. Sent-email markers are saved after each successful send.
6. A scheduled Vercel Cron endpoint checks unfinished bookings and retries recovery. The success page only reads verified server status; it is not the payment authority.

PostgreSQL stores bookings, temporary slot holds, request rate limits and Stripe webhook processing records. Advisory locks and database uniqueness constraints coordinate concurrent requests across serverless instances. The existing Vercel Global Config Store can remain configured for lightweight, mostly-read configuration, but this app does not use it as the transactional booking database. Completed booking records are not automatically deleted by this starter: define an explicit retention/deletion policy before production, balancing privacy requirements with any accounting, payment-dispute and business-record obligations.

## Files

```text
api/
  availability.js             # Public calendar availability (safe busy ranges only)
  health.js                    # Safe database/schema readiness check
  booking-status.js           # Server-verified status page endpoint
  cancel-booking.js           # Release an unpaid checkout hold
  create-checkout-session.js  # Reserve a slot and create Stripe Checkout
  reconcile-bookings.js       # Cron recovery for incomplete bookings/emails
  stripe-webhook.js           # Signed Stripe webhook
db/
  schema.sql                  # One-time PostgreSQL schema
lib/
  calendar.js                 # Google Calendar/Meet creation
  clients.js                  # Stripe, Calendar OAuth and Resend clients
  config.js                   # Environment configuration
  db.js                       # PostgreSQL pool, transactions, locks and rate limits
  email.js                    # Idempotent therapist/client email notifications
  fulfillment.js              # Paid booking → Calendar → email flow
  http.js                     # HTTP, CORS and request helpers
  store.js                    # PostgreSQL booking/slot persistence
  validation.js               # Input, consent, date and time validation
booking.html                  # Static front-end, served by GitHub Pages or another host
.env.example                  # Variable names only; no real secrets
vercel.json                   # API runtime and daily recovery cron
```

## Recommended free-tier starting point: Neon Postgres

Neon currently advertises a $0 Free plan and supports a Vercel Marketplace integration. Limits and free-tier policies can change; check the live plan before production use. The project is intentionally small and does not need a separate Redis service.

1. Create/sign in to a [Neon account](https://neon.tech/).
2. Create a project and a database (for example `psychology_booking`). Choose a region close to your Vercel functions.
3. In Neon, open **Connect** and copy the **pooled** connection string when available. It should be a PostgreSQL URL with SSL enabled, commonly including `sslmode=require`.
4. If you prefer to provision from Vercel, open **Vercel Dashboard → Marketplace → Neon → Install**, connect/create the database and connect it to this Vercel project. The integration can inject connection-string environment variables for you.
5. Copy the connection string into Vercel as `DATABASE_URL`. Never add it to source control or `booking.html`.
6. Open the Neon SQL Editor, paste the entire contents of `db/schema.sql`, and run it **once**. It creates the required tables and indexes.
7. Verify the tables exist (`bookings`, `slot_holds`, `rate_limit_buckets`, `stripe_webhook_events`) before testing checkout. After deployment, `https://YOUR-VERCEL-PROJECT.vercel.app/api/health` checks the database/schema readiness without exposing credentials.

If Neon changes its free quotas or your use exceeds them, pause and review billing before upgrading. Other managed PostgreSQL providers work as long as they provide a standard PostgreSQL connection string and allow TLS connections from Vercel.

## Deploying to Vercel

### 1. Upload to GitHub

Upload the contents of this project into the root of a repository (so `api/`, `lib/`, `db/` and `vercel.json` are at the repository root). Do not upload `.env` files containing actual secrets.

### 2. Import into Vercel

Import the GitHub repository into Vercel. Keep the project as a Node.js/Vercel Functions project. The booking page itself can be hosted with GitHub Pages or another static hosting service; the frontend and API can live on different origins.

### 3. Install/configure PostgreSQL

Follow the Neon steps above and run `db/schema.sql` once. Set `DATABASE_URL` in Vercel for the Production environment and any test/preview environment you actually use. After changing environment variables, redeploy so the functions receive them.

### 4. Add environment variables

Copy the *names* from `.env.example` to Vercel → Project → Settings → Environment Variables. Use real values only in Vercel. At minimum configure:

- `DATABASE_URL`
- `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET`
- `FRONTEND_ORIGIN` and `FRONTEND_BOOKING_URL`
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REFRESH_TOKEN`, `GOOGLE_CALENDAR_ID`, `GOOGLE_TIME_ZONE`
- `RESEND_API_KEY`, `EMAIL_FROM`, `THERAPIST_EMAIL`, `THERAPIST_NAME`
- Session settings and `CRON_SECRET`

`DATABASE_SSL` and `PG_POOL_MAX` are optional. Leave SSL enabled for managed Postgres; the default small connection pool is intended for serverless functions. Never use the PostgreSQL connection string, Stripe secrets, Google OAuth refresh token or Resend key in frontend JavaScript.

### 5. Set the frontend API URL

In `booking.html`, find `CONFIG.apiBaseUrl` near the top of the configuration object and replace `https://YOUR-VERCEL-PROJECT.vercel.app` with your actual Vercel project origin. Do not add `/api` to this value.

Set these variables consistently:

- `FRONTEND_ORIGIN`: the *origin only* of the site serving `booking.html`, for example `https://example.github.io` (no trailing slash). If GitHub Pages serves the page under a repository path, the origin still has no path.
- `FRONTEND_BOOKING_URL`: the full URL to `booking.html`, including any repository path.

### 6. Configure Google Calendar OAuth

Use OAuth credentials for an account that has permission to create events in the target calendar. Obtain a refresh token with calendar event write permission and add all four Google variables to Vercel. API-key access is not sufficient for creating private events on the therapist's calendar. Test that the calendar ID is the calendar that you actually want to block.

### 7. Configure Resend

Create a Resend API key and verify the sending domain/address. Set `EMAIL_FROM` to an address that Resend allows you to send from, and set `THERAPIST_EMAIL` to the email address that should receive every new paid booking notification. Google Calendar attendee updates are also requested, but the therapist email uses a separate Resend message.

### 8. Configure Stripe test mode

1. Add `STRIPE_SECRET_KEY` using a **test** secret key.
2. In Stripe Developers → Webhooks, create an endpoint with URL `https://YOUR-VERCEL-PROJECT.vercel.app/api/stripe-webhook`.
3. Subscribe to:
   - `checkout.session.completed`
   - `checkout.session.expired`
   - `checkout.session.async_payment_succeeded`
   - `checkout.session.async_payment_failed`
4. Copy the endpoint's signing secret into `STRIPE_WEBHOOK_SECRET` in Vercel. The signing secret is not your Stripe API key.
5. Ensure that the fixed session price configured by `SESSION_PRICE_MINOR` and `SESSION_CURRENCY` is correct. `16000` means PLN 160.00.
6. Redeploy after changing environment variables.

### 9. Test the entire flow before going live

- Open the booking page, choose a weekday/time and enter test contact details with consent.
- Complete a successful Stripe test payment.
- Verify one booking row in PostgreSQL, one Google Calendar event at the correct Warsaw local time, and two Resend messages (therapist + client).
- Confirm the event is visible on the correct therapist calendar, blocks the interval, and has a Google Meet link if Calendar provides one.
- Refresh the success page and replay a webhook from Stripe. Confirm that this does not create duplicate events or send duplicate emails.
- Use Stripe's simulated failed/expired checkout event and confirm unpaid holds are released.
- Try a second checkout for the same time before the first hold expires; it should be rejected.
- Temporarily misconfigure an email provider in test deployment, make sure the booking/event remains persisted, restore the key and run `/api/reconcile-bookings` from the Vercel Cron log or a secured manual invocation to confirm retries.
- Only switch to live Stripe keys after all checks pass. Test a real low-value payment/refund only if suitable for your business and accounting setup.

## Data model

`db/schema.sql` creates:

- `bookings`: appointment, Checkout Session, payment, Google event, status and email-delivery markers (with the fuller record in JSONB for flexible status fields).
- `slot_holds`: unique `(appointment_date, appointment_time)` lock for each pending checkout, with an expiry.
- `rate_limit_buckets`: persistent fixed-window rate limits.
- `stripe_webhook_events`: duplicate-delivery tracking and recovery after failed processing.

Keep the database region, Vercel function region and external services reasonably close where possible. Managed free databases can sleep/scale down and have quotas; the first request after inactivity may be slower. The database is the authoritative booking record, while Stripe and Google Calendar remain authoritative for payment and the calendar event respectively.

## Operations and troubleshooting

- **Checkout says server not configured:** inspect Vercel environment variables and redeploy. Confirm `DATABASE_URL`, Stripe and Google Calendar variables exist.
- **`relation "bookings" does not exist`:** run `db/schema.sql` against the same database referenced by `DATABASE_URL`.
- **Payment succeeded but the event is missing:** inspect Stripe webhook deliveries and Vercel logs. Reconciliation checks outstanding bookings and retries the same idempotent fulfilment path.
- **Therapist email missing:** check the Resend email log, sending-domain verification, `EMAIL_FROM`, `THERAPIST_EMAIL`, and Vercel logs. Retry recovery after correcting configuration.
- **Calendar event missing/permission denied:** refresh the Google OAuth token and confirm the Google account has write access to `GOOGLE_CALENDAR_ID`.
- **Duplicate webhook:** normal; the DB claim plus booking lock and deterministic Google event ID are intended to make processing repeatable.
- **Cron returns unauthorized:** set `CRON_SECRET` in Vercel. Vercel Cron sends `Authorization: Bearer <CRON_SECRET>`.
- **Cron recovery cadence:** the included job runs daily at 04:00 UTC. Stripe retries webhook failures more frequently; cron is a backstop, not the primary trigger. Check plan limits if you need a more frequent schedule.

## Development checks

Requires Node.js 20 or later.

```bash
npm install
npm run check
npm test
```

For local API testing, set the environment variables in a local `.env` file (ignored by git) or run `vercel dev` after pulling your Vercel environment variables. A real payment test requires Stripe test-mode credentials and working Google Calendar/Resend setup.

## Security and privacy notes

- The original booking page contained a browser-visible booking secret and Google API key. If either was previously deployed, rotate/restrict it in its provider console. The PostgreSQL version does not use that client-side secret.
- Calendar descriptions intentionally avoid copying any free-text reason for therapy into the appointment. Do not put sensitive mental-health information in calendar summaries, email subjects, logs or analytics.
- Protect access to the database and provider dashboards. Limit who can read booking records and avoid logging names/emails or database connection strings.
- This project records a consent timestamp/version; it does not independently establish legal compliance. Review Polish/EU privacy obligations, data retention/deletion, controller/processor agreements, refunds and cancellation wording with an appropriate professional.
