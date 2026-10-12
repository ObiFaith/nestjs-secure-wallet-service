# Secure Wallet Backend

A secure wallet backend with Google sign-in, API keys, and Paystack funding.

## Links

- [Swagger URL](https://nestjs-google-paystack-backend.vercel.app/docs)
- [TODO: demo video URL]

## Features

- Email/password signup and login with JWT issuance.
- Google OAuth flow that creates or updates a user and returns a JWT.
- Wallet creation with generated wallet numbers and balance tracking.
- Paystack deposit initiation and verification.
- Webhook processing for `charge.success` and `charge.failed` events.
- API key creation and rollover with permission checks.
- Wallet transfers with transaction records and balance updates.
- Pending deposit recovery via a scheduled job.
- Swagger UI at `/docs`.

## Architecture

```mermaid
flowchart TD
    A[Client] --> B[POST /wallet/deposit]
    B --> C[WalletService.deposit]
    C --> D[PaymentService.initiatePayment]
    D --> E[Paystack /initialize]
    E --> F[Paystack authorization_url]
    A --> G[Browser redirects to Paystack]
    G --> H[Paystack webhook]
    H --> I[POST /wallet/paystack/webhook]
    I --> J[WalletService.handlePaystackWebhook]
    J --> K{event}
    K -->|charge.success| L[processSuccessfulCharge]
    K -->|charge.failed| M[processFailedCharge]
    L --> N[Credit wallet balance]
    M --> P[Mark payment failed]

    Q[Scheduled: EVERY_10_MINUTES] --> R[syncPendingTransactions]
    R --> S[Verify old pending deposits]
    S --> T{status}
    T -->|success| L
    T -->|failed| M
```

## Tech stack

- NestJS
- PostgreSQL + TypeORM
- Passport + JWT
- Paystack API
- Google OAuth
- Swagger/OpenAPI
- bcrypt + class-validator
- Jest

## API reference

| Method | Path | Auth required | Purpose |
| --- | --- | --- | --- |
| GET | `/` | none | Serves the home page and links to `/docs`. |
| POST | `/auth/signup` | none | Creates a user and returns JWT + wallet data. |
| POST | `/auth/login` | none | Logs in a user with email/password and returns JWT + wallet data. |
| GET | `/auth/google` | none | Generates a Google OAuth URL or redirects to it. |
| GET | `/auth/google/callback` | none | Exchanges the Google auth code and returns user + JWT data. |
| POST | `/keys/create` | JWT | Creates an API key for the authenticated user. |
| POST | `/keys/rollover` | JWT | Creates a replacement API key for an expired key. |
| POST | `/wallet/deposit` | both (JWT or API key) | Initializes a Paystack deposit for the current user. |
| GET | `/wallet/me` | both (JWT or API key) | Returns the wallet summary for the current user. |
| POST | `/wallet/paystack/webhook` | none | Receives and validates a Paystack webhook. |
| GET | `/wallet/deposit/:reference/status` | both (JWT or API key) | Verifies a deposit by Paystack reference. |
| GET | `/wallet/transfer/:reference/status` | both (JWT or API key) | Checks a transfer status by reference. |
| GET | `/wallet/balance` | both (JWT or API key) | Returns the current wallet balance. |
| POST | `/wallet/transfer` | both (JWT or API key) | Transfers funds to a target wallet number. |
| GET | `/wallet/transactions` | both (JWT or API key) | Returns the current user wallet transaction history. |

## How it works

### Authentication (Google OAuth + JWT)

The Google OAuth flow starts at `GET /auth/google`, which builds the Google authorize URL with the configured client ID, redirect URI, OAuth scope, and response type. The callback route is `GET /auth/google/callback` and it expects a `code` query parameter. It exchanges the code for tokens, verifies the returned `id_token`, checks that the Google email is verified, and then creates or updates the user record.

JWTs are created with the user `id` and `email` and are signed with the configured `JWT_SECRET`. The token expiry is set to `1h` in the JWT module. The app validates bearer tokens with `passport-jwt` and the `JwtAuthGuard`.

### API keys

API keys are created through `POST /keys/create` and can be rolled over through `POST /keys/rollover`. The key format is `sk_live_<keyId><secret>`, and the stored value is a bcrypt hash. Each key has a name, an expiry (`1H`, `1D`, `1M`, `1Y`), and a permission list: `deposit`, `transfer`, and `read`.

The app limits each user to 5 active API keys and rejects keys that are expired, revoked, or missing the required permission.

### Wallet funding flow

1. The client calls `POST /wallet/deposit` with an amount.
2. The app loads the user wallet and calls the Paystack initialize API.
3. Paystack returns an `authorization_url` and a `reference`.
4. The server stores a pending Payment record and a pending wallet transaction.
5. The user completes funding on Paystack.
6. Paystack sends a webhook to `POST /wallet/paystack/webhook`.
7. The app verifies the signature, reads the event, and updates the wallet balance when the payment is successful.

Example deposit request:

```bash
curl -X POST http://localhost:3000/wallet/deposit \
  -H "Authorization: Bearer <jwt-token>" \
  -H "Content-Type: application/json" \
  -d '{"amount": 5000}'
```

### Webhook verification

The webhook route reads the raw request body and compares the `x-paystack-signature` header to an HMAC-SHA512 hash created with `PAYSTACK_SECRET_KEY`. If the signature does not match, the request is rejected. The app currently handles `charge.success` and `charge.failed` events. Other events are ignored.

Example webhook request:

```bash
curl -X POST http://localhost:3000/wallet/paystack/webhook \
  -H "x-paystack-signature: <paystack-signature>" \
  --data-binary @payload.json
```

### Idempotency

Duplicate processing is prevented with status checks and row locks:

- `Payment.reference` is unique.
- `WalletTransaction` status is checked before crediting or failing a payment.
- The wallet and payment rows are locked with `pessimistic_write` before writing updates.
- A duplicate webhook does not double-credit because the success/failure handlers return early when the payment or wallet transaction is already marked as processed.

### Recovery jobs

A cron job runs every 10 minutes and checks for pending deposit transactions older than 5 minutes. It verifies them against Paystack and reconciles them to the final success or failed state. This is meant to recover from missed or delayed webhook delivery without double-crediting the user.

> Note: Scheduled recovery needs a long-running server. It is not active on the Vercel demo.

## Design decisions

- NestJS + TypeORM + PostgreSQL were chosen because the app config sets a PostgreSQL connection URL and uses TypeORM entities for wallets, users, API keys, and payments.
- JWTs use a short expiry of 1 hour, as configured in the JWT module.
- API keys are hashed with bcrypt before storage so the raw key is not kept in the database.
- Webhook requests are handled using `rawBody: true` so the app can verify the Paystack signature against the exact payload bytes.
- Pending deposit recovery is scheduled every 10 minutes to catch missed Paystack events.
- [TODO: add my reasoning] for the choice to allow either JWT or API key for wallet routes.

## Known limitations and what I would improve next

- There is no rate limiting or security middleware configured in the app bootstrap.
- `WalletTransaction.reference` is not unique at the database level, so the idempotency guard relies on status checks rather than a unique constraint.
- The current tests are a minimal smoke suite and do not cover the real payment or OAuth flows.

## Getting started

### Clone the repository

```bash
git clone https://github.com/ObiFaith/nestjs-secure-wallet-service.git
cd nestjs-secure-wallet-service
```

### Prerequisites

- Node.js and npm
- PostgreSQL database
- A Google OAuth app and Paystack account
- A local `.env.example` file with the required values

### Install dependencies

```bash
npm install
```

### Environment variables

Create a `.env` file in the project root:

```dotenv
DATABASE_URL=postgres://user:password@host:5432/database
JWT_SECRET=replace-with-a-local-secret
PAYSTACK_API_URL=https://api.paystack.co
PAYSTACK_SECRET_KEY=replace-with-your-paystack-secret
GOOGLE_API_URL=https://oauth2.googleapis.com
GOOGLE_OAUTH_URL=https://accounts.google.com/o/oauth2/v2/auth
GOOGLE_CLIENT_ID=replace-with-google-client-id
GOOGLE_REDIRECT_URI=http://localhost:3000/auth/google/callback
GOOGLE_CLIENT_SECRET=replace-with-google-client-secret
HOST=0.0.0.0
PORT=3000
```

### Run the app

```bash
npm run start:dev
```

### Run tests

```bash
npm test -- --runInBand --watch=false
```

## Project structure

```bash
src/
├── api-key/
│   ├── api-key.controller.ts
│   ├── api-key.module.ts
│   ├── api-key.service.ts
│   ├── doc/
│   ├── dto/
│   ├── entities/
│   ├── guard/
│   └── test/
├── app.controller.ts
├── app.module.ts
├── auth/
│   ├── auth.controller.ts
│   ├── auth.module.ts
│   ├── auth.service.ts
│   ├── dto/
│   ├── jwt/
│   └── test/
├── config/
│   └── index.ts
├── database/
│   ├── database.module.ts
│   └── database.service.ts
├── interface.ts
├── main.ts
├── payment/
│   ├── entities/
│   ├── payment.module.ts
│   ├── payment.service.ts
│   └── test/
├── types/
│   └── express.d.ts
├── user/
│   ├── dto/
│   ├── entities/
│   ├── test/
│   ├── user.module.ts
│   └── user.service.ts
└── wallet/
    ├── doc/
    ├── dto/
    ├── entities/
    ├── helpers/
    ├── test/
    ├── wallet.controller.ts
    ├── wallet.module.ts
    └── wallet.service.ts
```

## Testing

The project uses Jest from the scripts in `package.json`.

- `npm test`
- `npm run test:watch`
- `npm run test:cov`
- `npm run test:debug`
- `npm run test:e2e`

The current tests are mostly smoke tests and check that the services/controllers are defined. They do not meaningfully cover the OAuth or Paystack flows.
