[README.md](https://github.com/user-attachments/files/30525832/README.md)
# Knowledge Hub — Core Flow (v1)

This is the **first working slice** of the full Knowledge Hub spec: register → login →
subscribe (₦1,000 via Paystack) → payment verification → subscription activation →
gated dashboard access. Everything else in the original spec (courses, e-books, admin
panel, other gateways, AI features, etc.) is designed to build on top of this
foundation — see "What's next" below.

## Stack actually implemented here

- **Backend:** Node.js, Express, TypeScript, Prisma, PostgreSQL, JWT in httpOnly
  cookies, bcrypt, Helmet, rate limiting, Paystack (initialize + verify + webhook)
- **Frontend:** Next.js 15 (App Router), React 19, TypeScript, Tailwind

## Getting it running locally

### 1. Database
```bash
docker compose up -d
```
This starts Postgres on `localhost:5432` with the credentials already baked into
`backend/.env.example`.

### 2. Backend
```bash
cd backend
cp .env.example .env
# edit .env: set PAYSTACK_SECRET_KEY / PAYSTACK_PUBLIC_KEY from your Paystack
# dashboard (test mode keys are fine to start)
npm install
npm run prisma:migrate      # creates tables
npm run dev                 # http://localhost:5000
```

### 3. Frontend
```bash
cd frontend
cp .env.local.example .env.local
npm install
npm run dev                 # http://localhost:3000
```

### 4. Test the flow
1. Visit `http://localhost:3000` → Register → you land on `/subscribe`
2. Click "Subscribe Now" → you're sent to Paystack's hosted checkout
3. Use a Paystack test card (e.g. `4084 0840 8408 4081`, any future expiry/CVV/OTP —
   see Paystack's test card docs for current values)
4. On success, Paystack redirects to `/payment/callback`, which calls
   `GET /api/payment/verify` — this independently re-checks the transaction with
   Paystack rather than trusting the redirect query string
5. Subscription is activated for 30 days, and `/dashboard` becomes accessible

### 5. Webhook (recommended, not just the callback page)
The callback page is a good UX confirmation, but the **webhook is the real source of
truth** for payment status (redirects can be interrupted; webhooks are reliable).
For local testing, use the Paystack CLI or a tunnel (e.g. `ngrok http 5000`) and set
the webhook URL in your Paystack dashboard to:
```
https://<your-tunnel>/api/payment/webhook/paystack
```

## How the subscription gate works

Every piece of premium content (courses, e-books, downloads — not yet built, but this
is the pattern) should be protected the same way `backend/src/routes/dashboard.routes.ts`
demonstrates:

```ts
router.get("/some-premium-route", authenticate, requireActiveSubscription, handler);
```

- `authenticate` verifies the JWT from the httpOnly cookie
- `requireActiveSubscription` checks the user's subscription status, auto-expires it
  if the 30-day window has passed, and returns `402` with a machine-readable `code`
  (`SUBSCRIPTION_REQUIRED` / `SUBSCRIPTION_EXPIRED`) if access should be denied

On the frontend, `/dashboard` calls `GET /api/subscription/me` on load and redirects
to `/subscribe` if the status isn't `ACTIVE` — the same pattern should be repeated on
any other protected page (courses list, e-book reader, etc.).

## What's deliberately NOT in this first slice

To keep this a real, working codebase rather than hundreds of empty stub files, these
pieces from the original spec are intentionally deferred:

- Forgot/reset password + email verification (needs a transactional email provider —
  tell me which one you want, e.g. Resend, SendGrid, and I'll wire it in)
- Flutterwave / Monnify integrations (same shape as `utils/paystack.ts` — easy to add
  once Paystack is confirmed working end-to-end for you)
- Courses & e-books modules (upload, playback, reading progress, reviews)
- Admin panel, analytics, coupons, notifications, AI features, certificates,
  gamification, SEO, CI/CD, Docker for the app itself (only Postgres is dockerized
  here)

## Suggested build order from here

1. Confirm this core flow works end-to-end with your real Paystack test keys
2. Courses + e-books modules (Cloudinary for images, S3 for PDFs/videos)
3. Admin panel (upload/manage content, view payments)
4. Flutterwave/Monnify as alternate gateways
5. Everything else (analytics, AI features, gamification, SEO, CI/CD)

Tell me which of these you want next and I'll build it the same way — real,
runnable code, not a shallow scaffold.
