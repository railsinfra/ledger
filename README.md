# Ledger Service (Rails)

Double-entry ledger microservice for posting and querying financial transactions. This service is part of the Rails Financial API monorepo and is designed to be auditable, idempotent, and safe for multi-tenant workloads.

## Open Source Project

This service is maintained as part of the open-source repository:

- Project overview: [`README.md`](../../../README.md)
- Contribution guide: [`CONTRIBUTING.md`](../../../CONTRIBUTING.md)
- Code of conduct: [`CODE_OF_CONDUCT.md`](../../../CODE_OF_CONDUCT.md)
- Security policy: [`SECURITY.md`](../../../SECURITY.md)
- License: [`LICENSE`](../../../LICENSE)

## What This Service Does

- Posts transactions with strict double-entry behavior
- Stores ledger transactions and entries with idempotency support
- Maintains account balances derived from entry movements
- Exposes read APIs for transactions and entries
- Exposes gRPC methods for transaction posting and balance lookup

## Architecture Overview

The service follows conventional Rails boundaries:

- `app/controllers/api/v1/ledger/` - read APIs for transactions and entries
- `app/services/ledger_poster.rb` - core posting engine and entry creation
- `app/models/` - ledger transactions, ledger entries, accounts, balances
- `app/grpc/ledger_service.rb` - gRPC interface used by other services
- `db/` - migrations and schema

## API Surface

### HTTP endpoints

Base path: `/api/v1/ledger`

- `GET /entries` - paginated ledger entries (`page`, `per_page`, optional `account_id`)
- `GET /transactions` - recent transactions (optional `status`)
- `GET /transactions/:id` - single transaction with entries
- `GET /health` - service health check

Auth behavior:

- Requires `Authorization: Bearer <jwt>`
- Extracts `business_id` from JWT as organization scope
- Supports environment via `X-Environment` header (`sandbox` or `production`)

### gRPC methods

Defined in `app/grpc/ledger_service.rb`:

- `post_transaction` - posts a ledger transaction (idempotent via `idempotency_key`)
- `get_account_balance` - returns balance for account/currency/environment

## Local Setup

### Prerequisites

- Ruby 3.1+
- Bundler
- PostgreSQL 14+

### Environment configuration

Use the provided example file:

```bash
cp .env.dev.example .env
```

Expected variables:

- `DATABASE_URL` - Postgres connection string
- `PORT` - HTTP port (default examples use `3000`)
- `GRPC_PORT` - gRPC server port (default examples use `50053`)
- `RAILS_ENV` - Rails environment
- `SECRET_KEY_BASE` - Rails secret key
- `JWT_SECRET` - shared secret for validating `Authorization: Bearer` JWTs on read APIs (must match the issuer, for example the users service or gateway)
- `SENTRY_DSN` - optional error reporting DSN

For local development, use non-production credentials and local databases only.

### Install and run

```bash
bundle install
bin/rails db:prepare
bin/rails server -p ${PORT:-3000}
```

## Testing

Run the Rails test suite from this directory:

```bash
bin/rails test
```

When changing posting logic (`LedgerPoster`) or API behavior, add/update tests in `test/` as part of the same change.

## Operational Notes

- Posting is idempotent by `organization_id + environment + idempotency_key`.
- A ledger transaction should produce balanced debit/credit entries with matching amounts.
- System accounts such as `SYSTEM_CASH_CONTROL` are used for deposit/withdraw flow mapping.
- Unknown accounts in `get_account_balance` currently return `0` (no activity yet).

## Observability and Error Tracking

- Emit structured logs for request and gRPC flows.
- Capture unexpected system failures with Sentry; do not report expected validation/business-rule outcomes as errors.
- Include actionable context (service, route/method, environment, organization, correlation IDs), never secrets or sensitive personal/financial data.
- Keep monitoring behavior centralized and consistent with repo-wide observability guidance.

## Deployment

- Service deployment config is in `railway.toml`.
- For production usage, ensure secure env var management and strict secret handling.
- Keep migrations and deployment changes in sync with this service release.
