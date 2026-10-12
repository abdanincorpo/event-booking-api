---
title: Quzuu Api Dev
emoji: 🐠
colorFrom: indigo
colorTo: gray
sdk: docker
pinned: false
---

# Event Ticket Booking API


A deliberately small REST API built to be tested: it must never oversell a
limited stock of tickets, never take two payments for one booking, and release
abandoned holds on time. Built on the
[go-clean-layered-architecture](https://github.com/abdanhafidz/go-clean-layered-architecture)
starter (Gin, GORM, PostgreSQL, provider-based DI).

The PRD, test plan and RFC live in the project log:
`logs.abdanhafidz.com/projects/event-booking-api`.

## Endpoints

| Method and path | Success | Notes |
| --- | --- | --- |
| `GET /health` | `200` | Database ping and build version |
| `GET /v1/events/{id}` | `200` | Ticket types with live `available` |
| `POST /v1/bookings` | `201` | Needs `X-User-Id` and `Idempotency-Key` |
| `GET /v1/bookings/{id}` | `200` | Own bookings only |
| `POST /v1/bookings/{id}/pay` | `200` | Needs `Idempotency-Key`; token `tok_success`, `tok_decline` or `tok_timeout` |
| `POST /v1/bookings/{id}/cancel` | `200` | Pending or confirmed, before the event starts |
| `GET /swagger/index.html` | `200` | OpenAPI UI; the document is at `/swagger/doc.json` |

Responses use the starter's envelope:

```json
{ "status": "success", "data": { "id": "…", "status": "PENDING_PAYMENT" }, "message": "…", "meta_data": null }
{ "status": "error", "code": "INSUFFICIENT_TICKETS", "errors": { "available": 3 }, "message": "…", "meta_data": null }
```

## Run locally

```bash
cp .env.example .env      # fill in the database settings
go run ./cmd/seed         # load the fixed dataset; skips if already seeded
go run .
```

## Test data

The container runs `entrypoint.sh` on startup: `seed`, then the API. The
seeder loads one fixed dataset (10,000 users and three fixture events) and
records a version marker in `seed_markers`. If that version is already there,
it logs "already seeded" and skips, so restarts and redeploys never reseed.
Two containers starting together are serialised by an advisory lock. Set
`SEED_ON_STARTUP=false` to skip seeding entirely.

| Fixture | Stock | Why it never needs re-seeding |
| --- | --- | --- |
| `load` | 5 × 1,000,000 | PT-01 to PT-04 use a few thousand tickets per run |
| `flash` | 500 | PT-05 never pays; the sweeper returns the stock when its holds expire, and the pipeline waits for that |
| `smoke` | 5 | The smoke test cancels or fails everything it books |

All IDs are deterministic (UUIDv5), so `go run ./cmd/seed -export DIR` writes
`users.csv` and `event.properties` for JMeter and the smoke test without
touching the database. Change `DatasetVersion` in `cmd/seed/fixtures.go` when
the fixtures change.

With Docker: `docker compose up --build` starts PostgreSQL and the API on
`localhost:8080`.

## Tests

```bash
go test ./config/... ./services/...                    # unit
export TEST_DATABASE_URL="host=localhost port=5432 user=postgres password=postgres dbname=booking_test sslmode=disable"
go test ./tests/integration/... -count=1 -v            # integration (27 cases)
go test ./tests/integration/... -run Concurrent -count=20
```

Integration tests run the API in-process against a real PostgreSQL. Each test
creates its own event and users, and after every test the harness checks the
inventory invariant `capacity = available + tickets held by active bookings`.
Point `TEST_DATABASE_URL` at a dedicated test database, never at the QA or
production database.

## How correctness is enforced

| Risk | Mechanism |
| --- | --- |
| Overselling | `UPDATE ticket_types SET available = available - q WHERE id = ? AND available >= q`, plus `CHECK (available BETWEEN 0 AND capacity)` |
| Double payment | Booking row locked `FOR UPDATE` during payment; partial unique index on successful payments |
| Duplicate requests | `idempotency_keys` row claimed as the first statement of the transaction; duplicates wait on it and replay the stored outcome |
| Abandoned holds | Sweeper every 5 s with `FOR UPDATE SKIP LOCKED`; payment also checks the hold itself |

## Configuration

See `.env.example`. The values that tests change are `HOLD_TTL`,
`SWEEP_INTERVAL`, `SWEEPER_ENABLED`, `PAYMENT_TIMEOUT`, `MOCKPAY_LATENCY_MS` and
`IDEMPOTENCY_TTL`. For Supabase use the Session pooler (port 5432) with
`DB_SSLMODE=require`.

## Delivery pipeline

`.github/workflows/pipeline.yml` runs on every push to `master`:

```
Run Unit Test -> Run Dependabot -> Deploy to HF -> Run Smoke-test OpenAPI -> Run Blazemeter Test -> Report
```

| Job | What it does | Fails when |
| --- | --- | --- |
| Run Unit Test | Unit tests, 27 integration tests on a PostgreSQL service container, concurrency tests ×20 | Any test fails |
| Run Dependabot | `go mod verify` and `govulncheck`. Dependabot itself opens update PRs from `.github/dependabot.yml` | A reachable known vulnerability |
| Deploy to HF | Copies the `QA_DB_*` secrets into the Space, uploads the repo, waits until `/health` reports this commit | Build or health check times out (15 min) |
| Run Smoke-test OpenAPI | Every path in `/swagger/doc.json` is routed; book, replay, pay, decline, sold out, cancel | Any check fails |
| Run Blazemeter Test | Exports fixture IDs, waits for flash stock (PT-05), uploads JMX, users and Taurus config, runs the scenario, then runs read-only correctness queries on the QA database | Correctness check, strict gate, or (after calibration) a target |
| Report | Stage table in the run summary | Any stage did not succeed |

Pull requests run `ci.yml` instead (tests only, no deploy).

**One-time setup**, all on a GitHub environment named `qa`:

| Name | Kind | Value |
| --- | --- | --- |
| `HF_TOKEN` | secret | Hugging Face token with write access to the Space |
| `QA_DB_HOST`, `QA_DB_PORT`, `QA_DB_USER`, `QA_DB_PASSWORD`, `QA_DB_NAME` | secret | Supabase session pooler settings |
| `BLAZEMETER_API_KEY_ID`, `BLAZEMETER_API_KEY_SECRET` | secret | BlazeMeter API key |
| `HF_SPACE` | variable | `owner/space-name` (a Docker Space) |
| `QA_BASE_URL` | variable | `https://owner-space-name.hf.space` |
| `BLAZEMETER_TEST_ID` | variable | ID of the BlazeMeter test (below) |
| `BLAZEMETER_SCENARIOS` | variable, optional | Scenarios on push; default `PT-01,PT-05` |

In BlazeMeter, create one performance test of type **Taurus** with main file
`bzt-test.yml`. Generate the three files once with
`scripts/perf/build_taurus.sh` and upload them; every run then replaces them.
Scenarios PT-02 to PT-04 are long and are meant for **Run workflow** with the
`scenarios` input rather than every push.

Run a scenario locally with JMeter:

```bash
go run ./cmd/seed && go run ./cmd/seed -export perf/data
jmeter -n -t perf/jmeter/booking.jmx -q perf/data/event.properties -Jusers_csv=perf/data/users.csv \
       -Jhost=localhost -Jport=8080 -Jbuyers=1 -Jbuyer_loops=50 -Jthink_min_ms=0 -Jthink_range_ms=0 \
       -l out/results.jtl -e -o out/report
python scripts/perf/check_thresholds.py PT-01 out/report/statistics.json perf/thresholds.json
```

## Deployment

The image listens on `7860`, the Hugging Face Spaces default. Set the `DB_*`
variables as Space secrets. A `VERSION` file in the build context (the commit
SHA) is baked into `/health`, so a pipeline can wait for the new build to go
live before smoke testing.
