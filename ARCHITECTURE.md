# Architecture

## Folder layout

```
src/
  app/                    Next.js routes: pages, server actions (actions.ts), API route handlers
    page.tsx              marketing home (hero, features, happy clients, pricing, FAQ)
    pricing/ terms/ privacy/ refunds/ cookies/ contact/
    (auth)/               login, signup, forgot-password, reset-password
    onboarding/           first-run store builder
    dashboard/            creator app: overview, store, products, orders, customers, analytics,
                          growth (AI), marketing, settings (account, billing, payments, payouts)
    admin/                platform admin panel
    [handle]/             public storefront  →  /{username}
    [handle]/[product]/   product page + checkout
    order/[id]/           buyer order page, receipt, downloads
    pay/                  subscription payment pages
    api/                  webhooks, cron, uploads, downloads, tracking, exports, auth verification
  components/             UI kit (ui/), dashboard, landing, storefront renderer, motion helpers
  lib/                    shared helpers usable on client + server (money, theme, catalogues)
  middleware.ts           security headers / routing helpers
  server/                 server-only code (never imported by client components)
    env.ts                typed access to environment variables
    db/                   schema.ts (every table), index.ts (driver/pool)
    auth/                 sessions, bcrypt, guards (requireCreatorPage, requireAdmin…)
    billing/              plans.ts (seed catalogue), entitlements, subscriptions (platform billing)
    payments/             provider interface; razorpay.ts, paypal.ts, sandbox.ts;
                          connections.ts (creator-owned Razorpay accounts); webhooks.ts
    services/             business logic: commerce (checkout/orders/refunds), products, store,
                          storefront, analytics, growth, launch, campaigns, lists, payouts,
                          subscribers, showcase, admin, audit, auth
    ai/                   provider.ts (Anthropic), mock.ts (demo provider), features.ts (prompts + schemas)
    email/                Resend client + HTML template
    storage/              Vercel Blob / local file storage, private downloads
    lib/                  errors, rate limits, secrets (AES-256-GCM seal/open)
drizzle/                  SQL migrations 0000–0005 + meta snapshots (applied in order)
scripts/                  dev launcher, PGlite server, migrate (+ plan seeding), make-admin
tests/                    service tests (tests/unit/*.test.ts) on an in-memory Postgres
public/                   static assets (images, illustrations)
vercel.json               region, daily cron, redirect of the old *.vercel.app host
```

## Request flow

UI → server action or route handler → Zod validation → guard (session + ownership + plan) → service in `src/server/services` → Postgres / external provider → typed result → UI.

Server components read data directly through services; client components call server actions. Nothing under `src/server` is shipped to the browser (`server-only` import guard).

## Two separate money flows

### 1. Buyers paying creators (creator-owned Razorpay)

- A creator connects **their own** Razorpay account in *Settings → Payments* by pasting a Key ID and Key Secret. `connectRazorpay` validates the keys against Razorpay, then stores the secret **sealed** (AES-256-GCM, key derived from `PAYOUT_ENCRYPTION_KEY`, falling back to `AUTH_SECRET`) in `payment_connections`. At most one `ACTIVE` connection per creator; replaced ones keep their keys so old orders can still be refunded/reconciled.
- Checkout (`startCheckout` in `services/commerce.ts`): `checkoutProviders(creatorId)` returns the creator's connection (plus the sandbox in local dev). **There is no platform fallback**, so a creator without a connection can't sell paid products.
- A PENDING order is created, then a Razorpay order on the creator's account. The buyer pays in Razorpay Checkout.
- `confirmOrderPayment` verifies the signature with the creator's secret and re-reads the payment from Razorpay; `applyVerifiedPayment` locks the order row, checks amount and currency, and marks it PAID idempotently. Delivery, receipt emails and analytics follow.
- Platform fee is **0**, and direct sales never touch the payout ledger: the money is already in the creator's bank account.
- Safety nets for buyers who close the tab: `reconcileOrder` runs when the order page loads and daily via `/api/cron/reconcile` (Bearer `CRON_SECRET`). Optional per-creator webhook: `/api/webhooks/razorpay/<token>`. Webhook events are treated as hints: Razorpay is re-queried, and the order must belong to that connection.

### 2. Creators paying the platform (SaaS subscriptions)

- Uses the **platform's** Razorpay account (`RAZORPAY_KEY_ID/SECRET` env vars) and Razorpay Subscriptions. Plans and prices live in the `plans` / `plan_prices` tables; `billing/plans.ts` only seeds an empty DB.
- Webhook: `POST /api/webhooks/razorpay` (HMAC with `RAZORPAY_WEBHOOK_SECRET`), stored once in `webhook_events` (duplicates are no-ops).
- Entitlements (product limits, AI credits, premium themes, analytics depth) are enforced server-side from the active subscription.

### Legacy: platform-collected sales

Earlier versions collected buyer money into the platform account and paid creators out via a ledger (`transactions`, `payouts`). That code remains for history (Settings → Payouts appears only when a creator has legacy earnings). New sales don't use it.

PayPal support exists in code (`payments/paypal.ts`) but is **not configured** in production; the UI is INR-only until it is.

## Data model highlights (`src/server/db/schema.ts`)

users · sessions · auth_tokens · profiles · plans · plan_prices · subscriptions · stores · store_sections · products · product_files · customers · orders · order_items · payments · refunds · webhook_events · payment_connections · transactions · payout_accounts · payouts · analytics_events · analytics_daily · ai_generations · ai_insights · campaigns · subscribers · notifications · audit_logs · rate_limits · error_logs

Money is stored as integer minor units (paise/cents) plus a currency code.

## AI

`server/ai/provider.ts` wraps the Anthropic SDK with structured (schema-validated) outputs; default model `claude-opus-5-5` (`AI_MODEL` overrides). Without `ANTHROPIC_API_KEY`, `mock.ts` returns clearly labelled demo output that consumes no credits. Every real call checks plan + monthly credits first and is logged in `ai_generations`.

## Theming

Colours are CSS variables (`src/app/globals.css`) mapped in `tailwind.config.ts`, so dark mode is a variable swap. `.dark` on `<html>` is set before paint by `THEME_SCRIPT` (`src/lib/theme.ts`) from the `cs-theme` cookie. Buyer-facing pages (`isThemedPath`) always stay light; `.theme-light` pins any subtree to light.

## Security

- **AuthN:** bcrypt (cost 12); sessions are random 256-bit tokens in an HTTP-only, Secure, SameSite=Lax cookie (`__Host-` in production); only SHA-256 hashes are stored, so sessions are revocable. Verification/reset tokens are single-use and hashed. Login/signup are rate-limited.
- **AuthZ:** every action re-reads session, ownership and plan from the DB.
- **Input:** Zod everywhere; Drizzle parameterised queries.
- **CSRF:** server actions are origin-checked by Next.js; API routes enforce same-origin; webhooks use HMAC signatures.
- **Headers:** CSP, HSTS, `X-Frame-Options: DENY`, `nosniff`, Referrer-Policy, Permissions-Policy (`next.config.ts`).
- **Payments:** fulfilment only after server-side verification; duplicate provider payments blocked by a unique index; webhooks idempotent.
- **Secrets:** server-only; creator Razorpay secrets and payout details encrypted at rest; analytics store a salted daily hash instead of IPs.

## Tests

`npm test` runs four suites on an in-memory Postgres with Razorpay mocked at the `fetch` level: `commerce` (checkout, verification, tampering, refunds, webhooks), `billing` (subscriptions lifecycle, entitlements), `growth` (analytics + insights), `connections` (creator-owned Razorpay: key validation, encryption, no-fee direct sales, reconciliation, webhook isolation between creators).
