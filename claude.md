# CLAUDE.md — lightning-payment-gateway

Instructions for Claude Code in this repo. Technical context only.

## 1. What this is

A Lightning Network payment gateway in Go, with merchant webhook notifications.

> *Lightning Network payment gateway in Go — idempotent payments, merchant webhooks (HMAC, retries, DLQ), double-entry ledger with reconciliation. Designed as payment infrastructure, built and benchmarked at lab scale.*

It is a lab project. The goal is to try real payment-infrastructure patterns, push them until they break, and write down what happened. When there are two reasonable options, prefer the one that teaches more, even if it is harder. Do not pick the simplest one by default.

## 2. Stack and conventions

- **Go** (latest stable). Router: stdlib `net/http` with ServeMux (Go 1.22+). No web frameworks.
- **PostgreSQL 16** with `pgx/v5`. Migrations with `golang-migrate` (SQL files in `migrations/`). The migrations are the source of truth for the schema.
- **Logging:** `log/slog` (JSON in production). Never log HMAC secrets, full invoices or IP addresses.
- **Metrics:** `prometheus/client_golang`.
- **Tests:** `go test`, table-driven. Acceptance criteria of each issue must be covered by real tests before opening a PR.
- **Commits:** Conventional Commits, in English (`feat(payments): add state machine transitions`).
- **PRs:** one per issue, with `Closes #N`, and a `## Design notes` section: what was chosen, what was discarded, and why.
- Code, comments, README and commits are in English.
- New dependencies need a short justification in the PR (why the stdlib is not enough).

## 3. Folder structure (target)

```text
cmd/
  api/            # HTTP server: payments, webhook-endpoints, health
  worker/         # outbox drainer + webhook delivery
  testreceiver/   # test endpoint: verifies HMAC signatures, simulates failures
internal/
  payment/        # Payment entity + state machine
  webhook/        # endpoints, HMAC signing, delivery, retries
  outbox/         # outbox table, SKIP LOCKED polling
  ledger/         # double-entry + reconciliation
  lightning/      # Lightning client interface, LND (gRPC) implementation, fake for tests
  platform/       # config, db, logger, metrics
migrations/
docker-compose.yml
Makefile
benchmarks.md     # created in the stress spike
```

## 4. Roadmap

One issue = one branch = one PR.

| # | Title | Branch |
| --- | --- | --- |
| 1 | chore: project scaffolding | `chore/scaffolding` |
| 2 | feature: payments API + state machine | `feature/payments-api` |
| 3 | feature: idempotency keys | `feature/idempotency` |
| 4 | feature: webhook subscriptions CRUD | `feature/webhook-subscriptions` |
| 5 | feature: transactional outbox + worker | `feature/outbox-worker` |
| 6 | feature: webhook delivery (HMAC + retries + DLQ) | `feature/webhook-delivery` |
| 7 | feature: observability + safe logging | `feature/observability` |
| 8 | feature: double-entry ledger + reconciliation | `feature/ledger-reconciliation` |
| 9 | feature: Lightning regtest (Polar + LND) | `feature/lightning-regtest` |
| 10 | spike: stress & measurement | `spike/stress-and-measure` |
| 11 | docs: final README | `docs/readme` |

Milestones: M1 Core Payments (#1-#3) · M2 Reliable Webhooks (#4-#6) · M3 Money Correctness (#7-#8) · M4 Lightning Integration (#9) · M5 Hardening and Launch (#10-#11).

## 5. Working rules

1. One issue = one branch = one PR. Do not mix scopes. Do not create a branch until you start that issue.
2. Before opening a PR: `go build ./... && go vet ./... && go test ./...` must pass.
3. Acceptance criteria are met with real tests, not with comments.
4. Implement what the issue asks. Extra ideas go in a PR comment, not in the code.
5. Test data is always regtest/local. This project never touches mainnet or real money.
6. Issues #2–#8 must not depend on a running LND node: use the Lightning client interface with a fake implementation in tests.

## 6. Domain rules

- Webhooks notify merchants about payment state changes.
- Delivery is at-least-once. The receiver deduplicates by event ID. Never claim exactly-once.
- Exact where money is involved: the ledger and reconciliation must be exact. Rate limiting can be approximate.
- Balances are a projection over ledger entries, never a mutable column.
- `payments.status` changes only through the state machine.
- When a dependency (Redis, etc.) is unavailable, decide fail-open or fail-closed based on the impact on money, and document the decision.