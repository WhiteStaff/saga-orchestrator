# CLAUDE.md — Shipment Saga Orchestrator

Project context for Claude Code. Read this before touching any file.

## What This Project Is

A saga orchestrator built from scratch. No Temporal, no MassTransit, no workflow engine.
Goal: internalize durability, compensation, and failure-handling machinery by owning it end-to-end.

## Current State

Skeleton only — `Class1.cs` placeholder in repo root, no real code yet. Phase 1 (core state machine) not started. The `.sln` has empty `src/` and `tests/` solution folders reserved for future layout.

## Commands

- Build: `dotnet build saga-orchestrator.sln`
- Run API: `dotnet run --project saga-orchestrator.csproj`
- Run tests: `dotnet test` (once a test project exists)
- Single test: `dotnet test --filter "FullyQualifiedName~SagaStateMachineTests.PaymentFails_TriggersCompensation"`

## Domain

Shipment fulfillment: `Reserve Inventory → Authorize Payment → Allocate Carrier → Dispatch → Confirm Delivery`

All external services (inventory, payment, carrier) are **injectable fakes** that fail on command.
No real integrations. No message broker.

## Stack

- **.NET 10** / C# 14 — `net10.0` target, `Nullable` and `ImplicitUsings` enabled. Prefer current .NET 10 idioms (primary constructors, collection expressions, `TimeProvider`); don't fall back to pre-LTS patterns.
- ASP.NET Core (HTTP API + background services via `IHostedService`)
- PostgreSQL (saga state + outbox)
- No additional frameworks for the orchestration logic itself

## Core Concepts — Understand Before Coding

**Saga instance** — one row per running workflow. Owns the current step and state.

**Outbox pattern** — state transition and outgoing command must commit in the same transaction.
Never update saga state and then publish a command separately. That's the dual-write bug this whole pattern prevents.

**Compensation** — every step that can succeed must register a compensation action.
On failure: run compensations in reverse order. Compensations go through the same outbox mechanism — they are durable, not fire-and-forget.

**Idempotency** — the worker delivers at-least-once. Every activity handler must be idempotent.
Use `idempotency_key` (step + saga_id) to dedup.

**Concurrency** — multiple worker instances are expected.
Use `SELECT FOR UPDATE SKIP LOCKED` when claiming outbox rows. Never optimistic locking for the outbox.

## Architecture

```
HTTP API
  └── SagaService          — starts sagas, handles queries
        └── SagaRepository — reads/writes saga_instances table

Background: OutboxWorker
  └── polls outbox_messages table
  └── dispatches to IActivityHandler (fake implementations)
  └── on success: marks message delivered, transitions saga state
  └── on failure: increments retry count, schedules next attempt

Background: RecoveryWorker
  └── on startup: finds sagas in non-terminal states with stale heartbeat
  └── re-enqueues their current pending command into outbox

IActivityHandler implementations (all fakes):
  └── ReserveInventoryHandler   — fails on command
  └── AuthorizePaymentHandler   — fails on command
  └── AllocateCarrierHandler    — fails on command
  └── DispatchHandler           — can simulate timeout
  └── ConfirmDeliveryHandler
```

## Database Schema (target)

```sql
-- Saga instances
saga_instances (
  id uuid PK,
  current_step text,
  status text,          -- running | compensating | completed | failed | parked
  payload jsonb,        -- domain data (order, amounts, ids from each step)
  created_at timestamptz,
  updated_at timestamptz,
  heartbeat_at timestamptz
)

-- Outbox
outbox_messages (
  id uuid PK,
  saga_id uuid FK,
  command_type text,
  payload jsonb,
  idempotency_key text UNIQUE,
  status text,          -- pending | delivered | failed
  retry_count int,
  next_attempt_at timestamptz,
  created_at timestamptz
)

-- Step audit log
saga_step_log (
  id uuid PK,
  saga_id uuid FK,
  step text,
  direction text,       -- forward | compensation
  result text,          -- success | failure
  error text,
  executed_at timestamptz
)
```

## Coding Rules

- **No dual-write** — state transition + outbox insert = one transaction, always
- **No fire-and-forget compensations** — compensations go through outbox
- **No ambient state** — saga instance carries everything needed to resume from any step
- **Explicit state machine** — transitions are a switch/match on (current_step, event), not implicit
- **Inject fakes via interface** — `IActivityHandler` registered in DI, swap implementation in tests

## What NOT to Do

- Don't use MediatR, MassTransit, or any saga framework — that defeats the purpose
- Don't use Kafka or RabbitMQ — outbox + polling is intentional, keep infra minimal
- Don't add real external API calls
- Don't use EF Core for the outbox worker hot path — use Dapper or raw Npgsql for explicit control

## Test Scenarios (the definition of done)

1. **Happy path** — all five steps succeed, saga reaches `completed`
2. **Payment fails** — carrier not reached; compensations run: void payment auth, release inventory
3. **Carrier fails** — compensations run in reverse: void payment, release inventory
4. **Dispatch timeout** — retries with backoff, eventually parks in `parked` state
5. **Crash recovery** — kill the process mid-saga, restart, saga resumes from correct step

## Mentor Review Protocol

Design decisions and MR reviews go to the Mentor agent (separate claude.ai chat).
Bring: the diff or the schema, not just a description.
The Mentor reads diffs directly — paste or describe the MR, don't summarize the code.
