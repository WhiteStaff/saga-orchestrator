# Shipment Saga Orchestrator

A pet project to build a saga orchestrator **from scratch** — without Temporal, without MassTransit sagas, without any workflow engine. The goal is to internalize the durability, compensation, and failure-handling machinery that platform solutions normally hide.

## Why

Most engineers have used a saga framework. Few have built one. Understanding what the engine does for you — and what it costs — is a Senior-level differentiator, especially relevant for logistics and fulfillment domains (Flexport, Amazon Logistics).

After this: rebuild the same workflow on Temporal .NET and write up the comparison.

## Domain

Shipment fulfillment workflow:

```
Reserve Inventory → Authorize Payment → Allocate Carrier → Dispatch → Confirm Delivery
```

Long-running (can span hours to days). Each step can fail or time out. External services are mocked as activities that fail on command.

## What This Must Exercise

These are the hard parts — the whole point of the project:

| Concern | What it means here |
|---|---|
| **Durable state machine** | Saga instance state persisted in Postgres; survives process crash |
| **Outbox pattern** | Atomically commit state change + outgoing command — solves dual-write |
| **Compensation / rollback** | Reverse completed steps in reverse order on failure; compensations are durable and retryable |
| **Idempotency** | At-least-once delivery; every step and compensation must be dedup-safe |
| **Retries + backoff + timeouts** | Flaky mocked services; parked/dead-letter state for exhausted retries |
| **Crash recovery** | Background recoverer finds in-flight sagas on startup and resumes them |
| **Concurrency** | Multiple workers; `SELECT FOR UPDATE SKIP LOCKED` to avoid two workers driving the same saga |

## Design Decision: Orchestration vs Choreography

This project builds **orchestration** — a central coordinator owns the saga and drives each step explicitly. Choreography (event-driven, no coordinator) is deliberately out of scope. Orchestration is where the design judgment lives and maps to prior experience.

## Stack

- **ASP.NET Core** — HTTP API to start/query sagas
- **PostgreSQL** — saga state persistence, outbox table
- **C#** — worker/recoverer background services

No message broker. The outbox is polled by a background worker. Deliberately minimal infrastructure.

## Scope

- One happy path (all steps succeed)
- Three deliberate failure scenarios:
  - Payment authorization fails → compensate inventory reservation
  - Carrier allocation fails → compensate payment + inventory
  - Dispatch times out → retry, then dead-letter
- Zero real integrations — all activities are injectable fakes that fail on command

## Roadmap

### Phase 1 — Core State Machine
- [ ] Saga instance model + Postgres persistence schema
- [ ] State machine transitions (explicit, no magic)
- [ ] Basic HTTP endpoint: start a saga, query its state

### Phase 2 — Outbox + Worker
- [ ] Outbox table + atomic commit (state change + outgoing command in one transaction)
- [ ] Background worker polls outbox, dispatches commands to activity handlers
- [ ] At-least-once delivery with idempotency keys

### Phase 3 — Compensation
- [ ] Compensation actions registered per step
- [ ] Reverse-order rollback on failure
- [ ] Compensation steps are durable + retryable (same outbox mechanism)

### Phase 4 — Resilience
- [ ] Retry with exponential backoff
- [ ] Timeout detection per step
- [ ] Parked/dead-letter state for exhausted retries
- [ ] `SELECT FOR UPDATE SKIP LOCKED` for concurrent workers

### Phase 5 — Crash Recovery
- [ ] Recoverer background service: on startup, find in-flight sagas and resume
- [ ] Deterministic replay from persisted state

### Phase 6 — Observability (optional, follow-up)
- [ ] OpenTelemetry tracing across saga steps
- [ ] Saga state dashboard endpoint

## Follow-up Project

Rebuild the same workflow on **Temporal .NET**. Then write a structured comparison:
- What the engine gave for free
- What control was surrendered
- Where the abstraction leaks

This comparison is a strong interview narrative for companies building durable workflow infrastructure.
