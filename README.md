# test-repo
# Billing Service Setup Guide

## Overview

A dedicated billing microservice that handles payments, credit accounting, coupons, and Stripe integration. It serves both `dave-v2-backend` (survey analysis) and `conversation-analysis-backend` (conversation analysis) — any platform that needs to check, hold, or deduct AI credits calls this service over HTTP.

Follows the same architectural pattern as the existing `auth-service`: a standalone FastAPI application with its own PostgreSQL database, called by other backends via internal HTTP.

## Prerequisites

- Python 3.10 or higher
- PostgreSQL database (same shared instance as auth-service)
- Stripe account with API keys
- Auth service running (for token validation)

## Installation

1. **Clone the repository** (if not already done)

2. **Create virtual environment**:
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

## Environment Configuration

1. **Copy the example environment file**:
   ```bash
   cp ENV_TEMPLATE.txt .env
   ```

2. **Edit `.env` file** with your actual credentials. All variables are described below:

   ### Server Configuration
   | Variable | Description | Default |
   |---|---|---|
   | `ENVIRONMENT` | Runtime environment (`development`, `production`) | `development` |
   | `PORT` | Port the service listens on | `8002` |

   ### Auth Service
   | Variable | Description | Default |
   |---|---|---|
   | `AUTH_SERVICE_URL` | URL of the auth service used to validate user tokens via `/verify` | `http://localhost:8000` |

   ### PostgreSQL Database
   | Variable | Description | Default |
   |---|---|---|
   | `DB_USERNAME` | PostgreSQL username | `gollm_admin` |
   | `DB_PASSWORD` | PostgreSQL password | `asdf1234` |
   | `DB_HOST` | Database host | `localhost` |
   | `DB_PORT` | Database port | `5432` |
   | `DB_NAME` | Database name | `billing_db` |

   ### Stripe
   | Variable | Description | Default |
   |---|---|---|
   | `STRIPE_SECRET_KEY` | Stripe secret API key (starts with `sk_test_` or `sk_live_`) | _(must be set)_ |
   | `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret (starts with `whsec_`). Found in Stripe Dashboard → Developers → Webhooks → Signing secret. | _(must be set)_ |
   | `STRIPE_PUBLISHABLE_KEY` | Stripe publishable key (starts with `pk_test_` or `pk_live_`). Used by frontends, exposed via config if needed. | _(must be set)_ |
   | `STRIPE_PRICE_ID_ONE_TIME` | Stripe Price ID for the one-time credit purchase ($2,500). Found in Stripe Dashboard → Products → Prices. | _(must be set)_ |
   | `STRIPE_PRICE_ID_RECURRING` | Stripe Price ID for the monthly recurring credit subscription ($2,500/month). Same Product, different Price. | _(must be set)_ |

   ### Frontend Redirect URLs
   | Variable | Description | Default |
   |---|---|---|
   | `DAVE_FRONTEND_URL` | Dave (survey analysis) frontend base URL. Used for Stripe Checkout success/cancel redirects. | `http://localhost:8080` |
   | `CONV_FRONTEND_URL` | Conversation analysis frontend base URL. Used for Stripe Checkout success/cancel redirects. | `http://localhost:8081` |

## Stripe Dashboard Setup

Before running the service, configure Stripe:

1. **Create a Product** in Stripe Dashboard for credit purchases (e.g., "AI Credits Pack").

2. **Create two Prices** on that Product:
   - **One-time**: $2,500 (mode: one-time) → copy ID to `STRIPE_PRICE_ID_ONE_TIME`
   - **Monthly recurring**: $2,500/month (mode: recurring) → copy ID to `STRIPE_PRICE_ID_RECURRING`

3. **Create a Webhook endpoint** in Stripe Dashboard → Developers → Webhooks:
   - URL: `https://<your-billing-domain>/webhooks/stripe` (or use Stripe CLI for local dev)
   - Events to listen for:
     - `checkout.session.completed`
     - `invoice.paid`
     - `invoice.payment_failed`
     - `customer.subscription.updated`
     - `customer.subscription.deleted`
   - Copy the signing secret → `STRIPE_WEBHOOK_SECRET`

4. **For discount coupons**, create Coupons in Stripe Dashboard and note their IDs. These are referenced when creating coupons via the admin API.

### Local Webhook Testing with Stripe CLI

For local development, use the [Stripe CLI](https://stripe.com/docs/stripe-cli) to forward webhooks:

```bash
# Install Stripe CLI (macOS)
brew install stripe/stripe-cli/stripe

# Login to your Stripe account
stripe login

# Forward webhooks to your local service
stripe listen --forward-to localhost:8002/webhooks/stripe
```

The CLI will print a webhook signing secret (starts with `whsec_`). Use that as your `STRIPE_WEBHOOK_SECRET` in `.env`.

## Database Setup

The billing service uses its own database (`billing_db`) on the shared PostgreSQL instance (same RDS/server that hosts `auth_service`, `survey_analytics`, etc.).

### Create the Database

```bash
# Connect as PostgreSQL superuser
psql -U postgres

# Create the database
CREATE DATABASE billing_db WITH OWNER = gollm_admin;

# Disconnect
\q
```

### Run the Migration

```bash
psql -U gollm_admin -d billing_db -f storage/database/migrations/001_initial_schema.sql
```

This creates:
- The `billing` schema
- All 9 tables (`stripe_customers`, `subscriptions`, `payment_history`, `credit_balances`, `credit_ledger`, `credit_config`, `processed_webhook_events`, `coupons`, `coupon_redemptions`)
- All indexes
- Seed data for `credit_config` (default rates and costs)

### Verify Setup

```sql
-- Connect to billing_db
psql -U gollm_admin -d billing_db

-- Verify schema exists
\dn billing

-- Verify tables exist
\dt billing.*

-- Verify config was seeded
SELECT key, value, description FROM billing.credit_config;
```

Expected seed data:

| key | value | description |
|---|---|---|
| `dollars_to_credits_rate` | `1` | Credits granted per £1 spent |
| `cost_base` | `-9.01` | Base cost in GBP for cost estimation formula |
| `cost_per_respondent` | `0.0055` | Cost in GBP per respondent |
| `cost_per_question` | `0.5186` | Cost in GBP per question (dominant cost driver) |
| `cost_margin_multiplier` | `1.3` | Margin multiplier (1.3 = 30% markup) |
| `cost_minimum` | `5.00` | Minimum cost floor in GBP |

## Running the Service

### Option 1: Using start_server.py (recommended)

```bash
# Basic startup
python start_server.py

# With auto-reload for development
python start_server.py --reload

# Custom port
python start_server.py --port 8002

# With file logging
python start_server.py --enable-file-logging --log-level DEBUG
```

### Option 2: Using uvicorn directly

```bash
uvicorn app_server:app --reload --host 0.0.0.0 --port 8002
```

### Verify the Service is Running

```bash
# Health check
curl http://localhost:8002/health

# Expected response:
# {"status": "healthy", "service": "billing-service"}
```

### Access API Documentation

- Swagger UI: http://localhost:8002/docs
- ReDoc: http://localhost:8002/redoc

## Ensuring Auth Service is Running

The billing service validates user tokens by calling the auth service's `/verify` endpoint. Make sure the auth service is running before testing user-facing endpoints:

```bash
# Check auth service health
curl http://localhost:8000/health
```

If the auth service is not running, all authenticated endpoints (`/billing/*`, `/internal/*`, and `/admin/*`) will return `502 Bad Gateway`.

## Testing the Endpoints

### Health Check (no auth)

```bash
curl http://localhost:8002/health
```

### Internal Endpoints (called by backends)

These are called by `dave-v2-backend` and `conversation-analysis-backend` during analysis execution. They use the same auth0 token authentication as user-facing endpoints — backends forward the user's token with each call.

```bash
# Check credit balance for a user
curl http://localhost:8002/internal/credits/balance/USER_PROFILE_UUID \
  -b "access_token=YOUR_TOKEN"

# Get credit config
curl http://localhost:8002/internal/credits/config \
  -b "access_token=YOUR_TOKEN"

# Place a credit hold
curl -X POST http://localhost:8002/internal/credits/hold \
  -b "access_token=YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "userProfileId": "USER_PROFILE_UUID",
    "analysisId": "ANALYSIS_UUID",
    "holdAmount": 500,
    "description": "Survey analysis: Test",
    "platform": "dave"
  }'

# Deduct credits (settle a hold)
curl -X POST http://localhost:8002/internal/credits/deduct \
  -b "access_token=YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "userProfileId": "USER_PROFILE_UUID",
    "analysisId": "ANALYSIS_UUID",
    "holdAmount": 500,
    "actualCost": 300,
    "llmCallCount": 30,
    "description": "Survey analysis: Test (30 AI calls)"
  }'

# Release a hold (analysis failure)
curl -X POST http://localhost:8002/internal/credits/release \
  -b "access_token=YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "userProfileId": "USER_PROFILE_UUID",
    "analysisId": "ANALYSIS_UUID",
    "holdAmount": 500,
    "reason": "Analysis failed: timeout"
  }'
```

### User-Facing Endpoints (token auth)

These require a valid auth token. Log in via the auth service first, then pass the token as a cookie or Bearer header.

```bash
# Get credit balance
curl http://localhost:8002/billing/credits/balance \
  -b "access_token=YOUR_TOKEN"

# Get credit history
curl "http://localhost:8002/billing/credits/history?limit=10&offset=0" \
  -b "access_token=YOUR_TOKEN"

# Get payment history
curl "http://localhost:8002/billing/payment-history?limit=10&offset=0" \
  -b "access_token=YOUR_TOKEN"

# Get subscription status
curl http://localhost:8002/billing/subscription \
  -b "access_token=YOUR_TOKEN"

# Create a checkout session (one-time purchase)
curl -X POST http://localhost:8002/billing/checkout-sessions \
  -b "access_token=YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "mode": "one_time",
    "redemptionId": null,
    "platform": "dave"
  }'

# Validate a coupon
curl -X POST http://localhost:8002/billing/coupons/validate \
  -b "access_token=YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"code": "WELCOME2025"}'

# Redeem a free credits coupon
curl -X POST http://localhost:8002/billing/coupons/redeem \
  -b "access_token=YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"code": "WELCOME2025"}'
```

### Admin Endpoints (requires is_admin: true)

The logged-in user must have `is_admin = TRUE` on the auth service's `user_profiles` table.

```sql
-- Set a user as admin (run on auth_service database)
UPDATE auth_service.user_profiles SET is_admin = TRUE WHERE email_address = 'admin@go-llm.ai';
```

```bash
# List all coupons
curl http://localhost:8002/admin/coupons \
  -b "access_token=ADMIN_TOKEN"

# Create a discount coupon
curl -X POST http://localhost:8002/admin/coupons \
  -b "access_token=ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "SAVE20",
    "type": "discount",
    "discountType": "percentage",
    "discountValue": 20,
    "stripeCouponId": "coupon_abc123",
    "maxRedemptions": 100,
    "expiresAt": "2026-12-31T23:59:59Z"
  }'

# Create a free credits coupon
curl -X POST http://localhost:8002/admin/coupons \
  -b "access_token=ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "WELCOME2025",
    "type": "free_credits",
    "freeCreditsAmount": 10000,
    "maxRedemptions": 1000
  }'

# Deactivate a coupon
curl -X PATCH http://localhost:8002/admin/coupons/COUPON_UUID \
  -b "access_token=ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"isActive": false}'

# View coupon redemption history
curl http://localhost:8002/admin/coupons/COUPON_UUID/redemptions \
  -b "access_token=ADMIN_TOKEN"

# View credit config
curl http://localhost:8002/admin/credit-config \
  -b "access_token=ADMIN_TOKEN"

# Update a credit config value
curl -X PUT http://localhost:8002/admin/credit-config/dollars_to_credits_rate \
  -b "access_token=ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "value": "60",
    "changeReason": "Increasing credit value to match new pricing model"
  }'
```

## Troubleshooting

### Database Connection Issues
- Verify PostgreSQL is running: `pg_isready`
- Check connection string variables in `.env`
- Ensure `billing_db` database exists: `psql -l | grep billing_db`
- Ensure the `billing` schema exists: `psql -U gollm_admin -d billing_db -c "\dn billing"`

### Auth Service Issues
- Verify auth service is running: `curl http://localhost:8000/health`
- Check `AUTH_SERVICE_URL` in `.env` points to the correct address
- All authenticated endpoints (`/billing/*`, `/internal/*`, `/admin/*`) return `502` if auth service is unreachable
- Internal endpoints use the same auth0 token as user-facing endpoints — backends must forward the user's token

### Stripe Issues
- Verify `STRIPE_SECRET_KEY` is correct (test vs live)
- For webhooks, ensure `STRIPE_WEBHOOK_SECRET` matches the signing secret in Stripe Dashboard
- For local testing, use Stripe CLI (`stripe listen --forward-to localhost:8002/webhooks/stripe`)
- Check Stripe Dashboard → Developers → Logs for webhook delivery attempts

### Credit Operations
- `402` from `/internal/credits/hold` means the user has insufficient credits
- Credits are granted via Stripe webhooks — if credits aren't appearing after payment, check webhook delivery in Stripe Dashboard
