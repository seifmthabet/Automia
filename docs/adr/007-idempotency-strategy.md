# ADR-007: Layered Idempotency Strategy

| Field | Value |
|---|---|
| Status | **Accepted** |
| Date | 2026-10-05 |
| Deciders | Engineering |
| Supersedes | — |
| Related | [execution-engine.md](../architecture/execution-engine.md) §7, [reliability.md](../architecture/reliability.md) §4, [006-transactional-outbox.md](006-transactional-outbox.md), [security.md](../architecture/security.md) §7 |

## Context

Automia performs the same logical operation many times, by design, and does so
across four boundaries where duplication is expected rather than exceptional. This
is not a bug to be eliminated; it is a property of an at-least-once system. The
work is to decide, per boundary, what "the same operation" means and what makes a
repeat safe.

The four boundaries, and the failure each must handle:

| # | Boundary | What can repeat | Consequence if it is not handled |
|---|---|---|---|
| 1 | Client → API | A retried `POST /executions`, a double-clicked button, a client that re-sends on a timeout | Two executions run. The customer is billed twice and the duplicate is visible in their workflow history. |
| 2 | Trigger → execution | The same webhook event delivered twice, a schedule firing twice, a manual replay | A duplicate execution for one business event. |
| 3 | Engine → worker | An outbox row published twice, a BullMQ job redelivered after a stall, a reconciler re-dispatching | A step executes twice. A customer system gets charged, emailed, or written to twice. |
| 4 | Worker → customer system | A step re-executed after a crash whose side effect actually succeeded, a customer retrying our call, a customer system being genuinely called twice | An irreversible external effect happens twice. **This is the only one Automia cannot fix alone.** |

Requirements:

1. **Duplicates must be cheap and silent** for the operator, and *correct* for the
   customer. A duplicate that produces a second execution, a second charge, or a
   second email is a product defect, not a curiosity.
2. **Exactly-once is not available and must not be claimed.** Not across
   PostgreSQL, Redis, and HTTP. Any design that says otherwise is wrong, and
   saying so plainly is part of this ADR.
3. **Idempotency must be enforced by the storage layer wherever possible.** A
   check-then-act in application code is a race; a unique index is not.
4. **The retry window must be stated.** Idempotency keys without an expiry window
   are a memory leak with a correctness veneer.
5. **The fourth boundary requires the customer to participate.** We must hand them
   a key and make honouring it trivial. This is a product responsibility, not an
   implementation detail.

## Decision

**Four independent dedup layers, each with a different key and a different storage
mechanism, each covering exactly one boundary. A duplicate that survives all four
is detected, not duplicated.**

```text
client ──▶ ① API replay ──────────▶ ② trigger dedup ──────────▶ ③ dispatch dedup ──▶ worker ──▶ ④ external key
            PK(org_id, key)           UNIQUE(dedup_key)          UNIQUE(step_exec, attempt)      Idempotency-Key header
            (request hash)           on executions             + lease_token claim
```

### ① API request replay

```text
idempotency_keys
  organization_id  uuid not null
  key              text not null          -- client-supplied
  request_hash     text not null          -- sha256(method + path + canonical body)
  state            enum('in_progress','completed') not null
  response_status  integer
  response_body    jsonb
  locked_until     timestamptz not null   -- in_progress expiry
  created_at       timestamptz not null
  expires_at       timestamptz not null   -- retention window

  PRIMARY KEY (organization_id, key)
```

Rules:

- **The key is client-supplied** via `Idempotency-Key` on `POST /executions`,
  `POST /executions/{id}/retry`, and webhook-registration mutations. Required for
  execution creation, because that is the operation whose duplication is visible
  and billable.
- **The `request_hash` is over method, path, and a canonicalised body** — object
  keys sorted, so a client that re-serialises its JSON differently still matches.
  A matching key with a *different* hash is **`422 Unprocessable Content`** with
  `code: IDEMPOTENCY_KEY_MISMATCH`, as specified in
  [security.md](../architecture/security.md) §3.6 (T22). This is what stops a
  client from accidentally reusing one key for two different intents, which would
  otherwise be silently wrong. Note it is `422`, not `409`: the request is
  semantically invalid, not in a conflicting state.
- **`in_progress` returns `409` with `Retry-After`** and
  `code: IDEMPOTENCY_KEY_IN_PROGRESS`, not the eventual response. `409` is
  correct here — a state conflict — and it is more honest than blocking the
  request: a client that retried while the first is still running is better told
  to wait than queued behind an unbounded request. The window is short (seconds).
- **`completed` replays the stored status and body verbatim**, including the
  original `Location` header, so a retry after a network failure returns the same
  `201`.
- **Retention: 24 hours**, enforced by the sweeper. Stated rather than implied. A
  longer window helps clients with very long retry gaps and costs a table; 24 hours
  covers every realistic mobile-network or proxy retry. Beyond the window, a repeat
  key creates a new execution, which is the documented behaviour.
- **Scoped to `organization_id`**, so two tenants using `"abc"` cannot collide or
  read each other's stored responses. The primary key is the composite, which
  enforces that at the storage layer.

**Why not just use `executions.dedup_key` for this?** Because the two answer
different questions. "Has this *client* retried this *request*?" is about HTTP
semantics and needs to replay a response. "Has this *business event* already
produced an execution?" is about triggers and needs no response replay. Merging them
would mean storing a response body for a webhook delivery, and would lose the
request-hash mismatch detection that catches a real client bug.

### ② Trigger dedup

```text
executions.dedup_key  text null      -- NULL means "never deduplicated"

  UNIQUE (dedup_key)                  -- global, not per-organization
```

The key is globally unique and every format is already tenant-prefixed
(`api:{organizationId}:…`, `hook:{endpointId}:…`), so uniqueness across tenants
costs nothing and is not a cross-tenant information leak: a key is not guessable
and a `UNIQUE` violation reveals only that the key was used, never by whom. A
nullable `UNIQUE` allows an unlimited number of `NULL` rows, which is what makes
"unkeyed triggers are never deduplicated" expressible in the schema rather than in
application code.

| Trigger | `dedup_key` |
|---|---|
| API, with an `Idempotency-Key` | `api:{organizationId}:{key}` |
| API, without one | `null` — no dedup; the client's choice, and documented |
| Webhook | `hook:{webhookEndpointId}:{externalEventId}` |
| Schedule | `sched:{scheduleId}:{scheduledForIso8601}` |
| Operator retry | `retry:{parentExecutionId}:{n}` |
| Manual "run now" | `null` — a human explicitly asked for another run |

```text
INSERT INTO executions (...)
VALUES (...)
ON CONFLICT (dedup_key) DO NOTHING
RETURNING id;
```

**`ON CONFLICT DO NOTHING` is the whole implementation.** The insert is the dedup
check, so there is no check-then-act race, and the webhook delivery path retries
safely for free: a second delivery of the same event does no work, records its own
`webhook_deliveries` row, and returns the existing execution id. No transaction,
no lock, no read.

**Webhook replay protection is the same mechanism**, and it is worth being precise
about: `webhook_deliveries` has `UNIQUE (webhook_endpoint_id, external_event_id)`.
A repeated `externalEventId` is rejected at the ingestion boundary *before*
signature verification is even relevant, because a replay of a previously valid
request is the threat. A delivery that fails signature verification is recorded with
`rejected` status and never reaches `executions`. The external event id is
extracted from a configured JSON path or an HTTP header
([security.md](../architecture/security.md) §7.2); if the endpoint is configured
without one, replay protection for that endpoint degrades to the 5-minute
in-progress window only, and the UI says so.

**The schedule key embeds the exact occurrence**, so a schedule that fires twice for
the same instant cannot create two executions, while a schedule that legitimately
fires at 09:00 on Monday and 09:00 on Tuesday creates two — because the ISO
timestamps differ. This is why the key is not `sched:{scheduleId}`.

### ③ Dispatch dedup

Two mechanisms, both in PostgreSQL, both mandatory:

**(a) The `jobs` insert.** `UNIQUE (step_execution_id, attempt)` on `jobs`. A
dispatch for attempt *n* can be recorded at most once, forever, no matter how many
publishers, reconcilers, or rollbacks try. This constraint — not a key on the
outbox row — is the dedup mechanism: the engine inserts the `jobs` row in the same
transaction that inserts the `outbox_events` `STEP_DISPATCH` row, so the intent
and its dedup guarantee commit together
([ADR-006](006-transactional-outbox.md)).

**(b) The worker's claim.** This is the one that actually matters, because it is
the last line of defence and the only one that runs when the step is *already* due:

```sql
UPDATE step_executions
SET state = 'RUNNING',
    lease_token   = $token,
    lease_expires_at = now() + interval '60 seconds',
    worker_id     = $workerId,
    started_at    = coalesce(started_at, now())
WHERE id           = $stepExecutionId
  AND state        = 'QUEUED'
  AND attempt     = $attempt
  AND lease_token  IS NULL;
```

`rowCount = 0` means someone else already claimed it, the step is no longer
`QUEUED`, or the attempt has moved on. **The worker stops, and the work is
abandoned as a duplicate** — no log at `warn`, no alert, because a lost race is
normal and expected under at-least-once delivery.

The `lease_token` is a fresh random value per claim. A worker whose lease expired
cannot write: every subsequent update it attempts carries the old token, and the
`WHERE lease_token = $token` predicate fails. **This is what makes a late-finishing
stalled worker harmless** — it cannot clobber the state of the attempt that
replaced it, which is the specific hazard that a lock-free "just update it" design
has.

### ④ The external idempotency key

For the `http` step type, Automia sends:

```text
Idempotency-Key: exec:{executionId}:step:{nodeId}:g{generation}
```

| Component | Why it is there |
|---|---|
| `executionId` | Distinguishes independent runs of the same workflow |
| `nodeId` | Distinguishes steps within one execution. Stable, because definitions are immutable ([ADR-005](005-workflow-versioning.md)) |
| `generation` | Increments **only** for an operator-initiated retry |

**The `generation` suffix is the subtle part and deserves a worked example.**

```text
Run 1: exec:9f3a…:step:charge:g0   → customer's ledger shows 1 charge
       worker crashes after the charge succeeds, before writing SUCCEEDED
       job redelivered, external key is *identical*  exec:…:charge:g0
       customer system dedups  → returns the original response, no second charge  ✓

Operator presses "retry step" (a deliberate second attempt to do the work again):
       generation → 1
       new key:  exec:…:charge:g1   → customer's ledger shows a 2nd charge
       which is what the operator asked for  ✓
```

Without `generation`, an operator retry would be silently swallowed by the
customer's dedup and nothing would happen — a bug that is very hard to diagnose
because the system looks like it succeeded. With it, automatic recovery is
deduplicated and deliberate retry is not. The `generation` column lives on
`executions`, and only `POST /executions/{id}/retry` increments it.

**We send the key; we cannot enforce that it is honoured.** This is stated plainly
in the API documentation and in the `http` step's docs. For a customer system with
no idempotency support, Automia guarantees at-least-once and the customer must
tolerate it. For systems that do support it (Stripe, and most payment and
messaging APIs), the key makes the retry safe. Where a step is marked
`idempotent: false` in its config, the key is still sent and the UI warns.

**A timeout is not a proof of failure.** A step that times out may have succeeded.
Automia cannot know, and does not guess: it retries with the same key and lets the
customer's dedup decide. This is the correct behaviour and the reason ④ exists.

### What we explicitly do not do

- **No global dedup across executions.** Two independent runs of the same workflow
  are two executions. Deduplicating them would be a product decision nobody asked
  for.
- **No dedup by request body.** Two identical `POST /executions` bodies from a
  client that did not send a key create two executions. Fingerprinting would
  reject legitimate repeated runs.
- **No dedup on reads.** `GET` is naturally idempotent.
- **No dedup on `DELETE` beyond natural idempotence.** Deleting an already-deleted
  execution returns `404`, which is correct.
- **No application-level `SELECT`-then-`INSERT` anywhere in these paths.** Every
  layer is `INSERT ... ON CONFLICT` or a conditional `UPDATE`. A code review that
  finds a check-then-act in a dedup path should fail.

## Alternatives Considered

### 1. Rely on BullMQ's `jobId` de-duplication alone

Rejected. It is scoped to jobs that still exist, so it expires when the job is
removed (24h after completion here), and it is a Redis structure that a flush
erases. It is a useful transport-level guard and we use it via `jobs.id`, but it
cannot be the business answer. Rejected as the *sole* mechanism.

### 2. Rely on customer-system dedup only

Rejected. It is necessary (layer ④) and it is not sufficient: it does not stop a
duplicate *execution* from existing, which is what a customer sees in their history
and what a schedule replay would produce. Layers ①–③ exist to stop the execution
from being created twice in the first place.

### 3. Distributed locks (Redis or Postgres advisory) around operations

Rejected. A lock is a *mutual exclusion* mechanism being used as a
*deduplication* mechanism, and the two are not interchangeable: a lock does not
remember that the work was already done. Locks also have a failure mode that is
strictly worse — if the holder dies, whether the operation happened becomes
unknowable. The conditional `UPDATE` has the property we want: it is a claim, it is
atomic, and its `rowCount` tells us definitively whether we won.

### 4. Distributed transaction / XA

Rejected. Not available across PostgreSQL, Redis, and HTTP, and not needed once
the intent is recorded ([ADR-006](006-transactional-outbox.md)). Its actual role
here is nil.

### 5. Event sourcing with a dedup event log

Rejected for v1 per [ADR-004](004-modular-monolith.md) §3. Event sourcing is a
legitimate home for idempotency — the log *is* the dedup structure — but it makes
the engine's hot path a conditional update against a projection, which is a poor
fit for a claim-based state machine. A first-class `idempotency_keys` table plus
unique indexes achieves the same result with none of that cost.

### 6. A distributed cache for request dedup

Rejected. A cache is the wrong structure for a durability requirement: it evicts,
it has a TTL you must choose, and it is not part of any transaction. Idempotency
records must be as durable as the operation they protect, so they live in
PostgreSQL with the operation.

### 7. Timestamps, request hashes, or fuzzy matching for API dedup

Rejected. `request_hash` is used, but only to *validate* a client-supplied key, not
to identify duplicates. Time-window fingerprinting produces false positives —
legitimately identical requests inside the window are rejected — and false negatives
across a window boundary. A client-supplied key is exact, and it puts the burden
where the knowledge is: on the client that knows it retried.

### 8. Redis Streams or Kafka for the dedup layer

Rejected as a layer. Streams give consumer-side dedup by `XREADGROUP` and message
id, which is genuinely good — and it is a Redis structure, so it is not durable
([ADR-002](002-postgresql.md)), and it dedups *consumption*, not the creation of an
execution. Our duplicate executions appear at trigger time, in PostgreSQL, and must
be prevented there.

## Why

**1. The four boundaries are genuinely different problems.** Client retry is HTTP
semantics and needs response replay. Trigger duplication is a business-fact problem
and needs a unique constraint. Dispatch duplication is an at-least-once delivery
problem and needs an atomic claim. External effects need customer cooperation. One
mechanism cannot serve all four, and collapsing them is how systems end up with a
dedup layer that covers two of the four.

**2. `ON CONFLICT DO NOTHING` makes dedup racy-proof by construction.** The insert
*is* the check. There is no window between reading "no record exists" and writing
one, which is exactly the window that produces a duplicate under concurrency. With
`RETURNING id` empty, we know someone else created it and we read their id. This is
the single strongest technical reason for the whole ADR.

**3. Storage-layer enforcement survives code that has not been written yet.** The
reconciler, a future bulk-import tool, a retry script written during an incident at
2am — all of them get deduplication for free, because `UNIQUE` cannot be forgotten.
Application-level dedup is only correct in the paths someone remembered to write.

**4. The conditional claim makes a lost race normal instead of exceptional.** Two
workers racing for the same step is not an error condition; it is a Tuesday. The
`UPDATE ... WHERE state = 'QUEUED' AND lease_token IS NULL` returning zero rows is
the designed response. Designing for the common case is what keeps the logs
readable.

**5. `lease_token` closes the late-writer hole.** Without it, a worker that stalls
past its lease, wakes up, and writes `SUCCEEDED` would overwrite the state of the
attempt that replaced it — a corruption that is nearly impossible to diagnose and
frequently destroys work. The token makes the stale writer a no-op.

**6. `generation` separates accidental duplication from intentional retry.** This
is the distinction that makes automatic recovery safe *and* operator retry
meaningful. Without it, one of the two is broken; with it, both are correct. It is
also the reason `POST /executions/{id}/retry` exists as a distinct operation rather
than being the same code path as a redelivery.

**7. The fourth boundary is honestly scoped.** We send a key and document the
customer's obligation. Claiming otherwise would be a lie that surfaces as a
double-charge. Stating it in the ADR, the API docs, and the `http` step docs is the
deliverable.

**8. At-least-once plus idempotency is a complete story.** We tolerate duplicates at
every layer because every layer makes them free. That is a stronger position than
either "exactly once" (unavailable, and usually a lie) or "at-most once" (which
loses work, and a lost dispatch is a customer's workflow stuck forever with no
error message anywhere).

## Consequences

### Positive

- **No duplicate executions from client retries**, with response replay so the
  client gets its original `201` back.
- **No duplicate executions from webhook redelivery or schedule double-fire**, at
  any volume, enforced by a unique index.
- **A duplicate dispatch costs one `UPDATE` that returns zero rows** and is
  invisible in the logs.
- **A late or stalled worker cannot corrupt state**, because its `lease_token` no
  longer matches.
- **Automatic recovery is deduplicated end-to-end**, including at the customer
  system, via the stable external key.
- **Deduplication is enforced by the database**, so it applies to every writer,
  present and future.
- **Reconcilers and repair tools are safe to run repeatedly** — the most valuable
  property for operational tooling, and a direct consequence of layer ③.
- **The retention windows are stated** (24h for API keys), so storage is bounded and
  behaviour at the boundary is documented.
- **Replay protection is structural**, not a check in the handler.
- **Tenant isolation holds for keys**, since the primary key is
  `(organization_id, key)`.

### Negative

- **Four keys, four stores, four code paths to understand.** The cognitive cost is
  real and is paid by whoever debugs a duplicate at 3am. Mitigated by this ADR,
  by the naming convention, and by all four being visible in one execution's
  `step_executions` and `jobs` rows.
- **A `422` on a key mismatch is a new failure mode for clients** to handle. It
  catches a real client bug, and it is a behaviour change from a naive API.
- **`ON CONFLICT DO NOTHING` requires knowing the constraint name or shape**, which
  is a schema coupling in the repository layer. Drizzle makes this explicit and
  reviewable.
- **Layer ④ depends on customer behaviour.** A customer system that ignores
  `Idempotency-Key` will see duplicates. This is stated in the docs and in the UI
  warning, and it is the one guarantee Automia cannot provide alone.
- **24-hour API-key retention means a retry after 24 hours creates a duplicate
  execution.** Chosen and documented; extending it costs a growing table.
- **`ON CONFLICT` on a partial or expression index requires a matching index
  predicate.** A schema change that narrows the index can silently turn dedup into
  a normal insert, with no error. This is why the dedup indexes are reviewed as
  correctness-critical and asserted by a migration test.
- **Webhook endpoints without a configured external-event-id path lose true replay
  protection** and fall back to a 5-minute in-progress window. Surfaced in the UI.
- **More database round-trips**: ① is a read plus a write, ② is one write, ③ is one
  conditional write. Acceptable given the guarantees; noted for honesty.

## Operational Impact

**Throughput:** layer ② is a single `INSERT` for the whole path, which is why the
outbox pattern can carry a high trigger rate. Layer ① adds one indexed lookup per
API request that supplies a key. Layer ③ adds one conditional `UPDATE` per job
delivery — i.e. one write *more* than a naive design, in exchange for making
duplicate delivery free. That trade is correct and worth remembering the next time
someone profiles the claim query.

**Indexes, treated as correctness-critical:**

| Table | Index | Why correctness depends on it |
|---|---|---|
| `idempotency_keys` | `PRIMARY KEY (organization_id, key)` | Without it, ① does not work |
| `executions` | `UNIQUE (dedup_key)` (nullable column) | Without it, ② does not work |
| `jobs` | `UNIQUE (step_execution_id, attempt)` | Without it, ③(a) does not work |
| `step_executions` | `(state, available_at)` | The reconciler's claim predicate |
| `webhook_deliveries` | `UNIQUE (webhook_endpoint_id, external_event_id)` | Replay protection |

A migration test asserts every one of these exists, because a dropped unique index
does not fail loudly — it silently permits duplicates. **This is the single most
important operational test in the system.**

**Migrations:** all four constraints ship in the initial schema, in one migration,
before any code that depends on them. `idempotency_keys` is swept by
`expires_at`; the sweeper's allowlist includes it, and the index on `expires_at`
keeps the sweep cheap.

**Monitoring:**

| Signal | Alert condition | Meaning |
|---|---|---|
| ① `IDEMPOTENCY_KEY_IN_PROGRESS` (409) rate | Sustained non-zero | Clients retrying while the first is still running, or a stuck row — check `locked_until` |
| ① `IDEMPOTENCY_KEY_MISMATCH` (422) | Any | A **client bug**. The most actionable dedup signal we have |
| ② `ON CONFLICT DO NOTHING` hit rate on `executions` | Rising | Legitimate dedup working (retries, redeliveries). Alert if it exceeds the retry rate by an order of magnitude |
| ② hit rate dominated by `api:` keys | Rising | Client-side retry loops, or a proxy double-sending |
| ③ `jobs` `ON CONFLICT` hit rate | Sustained non-zero | Redelivery or reconciler re-dispatch. Expected during incidents; alarming at steady state |
| ③ claim returning zero rows | Any | Normal and unlogged. Exposed as a counter, never alerted |
| ④ 4xx from a customer's endpoint with an `Idempotency-Key` | Sustained | The customer rejects the header. Loud, and actionable |
| ① `idempotency_keys` growth | Beyond expectation | Retention sweeper not running |
| ④ steps marked `idempotent: false` in use | Any | **Customer-visible risk of duplicate effects.** Worth a product review list |

The last row is worth stating as an operational policy: a list of workflows using
non-idempotent steps is a list of workflows where a duplicate is a real
double-charge. It is a legitimate configuration; it should not be invisible.

**Failure modes:**

| Failure | Behaviour | Blast radius |
|---|---|---|
| A dedup index is dropped by a migration | Duplicates become possible, **silently** | Unbounded. Prevented by the migration test |
| Redis lost | Layers ①, ② unaffected. ③ rebuilds from `jobs` + `step_executions` | Delivery delay |
| Duplicate publish | `jobs` `ON CONFLICT` absorbs it; claim returns zero rows | None |
| A worker stalls past its lease | Job redelivered; new claim sets a new token; the old worker's writes fail | Duplicate *execution* of the step body, deduped by ④ at the customer |
| A step times out after succeeding | Retried with the same external key; the customer dedups | Depends on the customer honouring ④ |
| API key table unavailable | Cannot create executions | All execution creation. Same blast radius as PostgreSQL |
| A client sends the same key for two different bodies | `422` `IDEMPOTENCY_KEY_MISMATCH` | The second request. The client bug is surfaced, not masked |
| Clock skew between API and worker | Lease expiry computed from different clocks → premature or late reclaim. Bounded by the `lease_expires_at` margin and safe because the token still protects writes | Extra duplicate work, no corruption |

**Runbook: "a customer reports a duplicate execution."** (1) Compare
`executions.dedup_key` — if the two rows have the same key, layer ② worked and what
the customer is seeing is two *intentional* runs (a manual "run now" has a null key
by design, and an operator retry has a `retry:` key). (2) If the keys differ, the
client did not send an `Idempotency-Key` and layer ① never applied — check whether
the client retries. (3) If the two runs are from one execution, it is not a
duplicate execution but a duplicate *step* — check `step_executions.attempt` and
`lease_token` history, and the customer's idempotency support. (4) Check whether the
duplicate arrived from two publishers, which would point at the publisher rather
than the client.

## Security Impact

**Layer ② is a replay-protection mechanism, and it is in the threat model.** A
webhook endpoint is an unauthenticated internet-facing route into execution
creation. Without `UNIQUE (webhook_endpoint_id, external_event_id)`, anyone who
captures one valid signed request can replay it forever, creating executions at
will — a denial-of-service and a tenant-cost problem. With it, the second delivery
is rejected at ingestion, before any execution is created. This is why the
constraint is `NOT NULL`-backed by a required endpoint configuration, and why the
UI warns when an endpoint has no external-event-id mapping.

**A key-mismatch `422` leaks a small amount of information**: that a key has been
used, and for a different body. Scoped to `organization_id`, so a tenant can only
probe its own keys, and a client already knows its own keys. Acceptable.

**Idempotency keys are not secrets.** They are not credential material, they are
client-chosen identifiers, and they are stored in the clear because they must be
looked up. A leaked key does not grant access; a leaked key without the
organization's credentials grants nothing. They are still redacted from logs where
they appear alongside `Authorization`
([security.md](../architecture/security.md) §5.5), because a log line containing a
key and a response body is a replay-capable artefact for anyone who also obtains
the credentials.

**`Idempotency-Key` and authorization are separate.** A key is looked up *after* the
tenancy guard resolves `organization_id`, and the composite primary key means a
cross-tenant lookup finds nothing — `404`/`403` per
[security.md](../architecture/security.md) §4.4, never another tenant's cached
response. Storing a response body is therefore storing a body the caller was already
authorized to receive.

**Layer ③ limits the value of a compromised queue.** An attacker who can enqueue a
job for a known `stepExecutionId` must still win a conditional `UPDATE` on
`state = 'QUEUED' AND lease_token IS NULL`. If the step is not queued, or is already
claimed, the attacker's job is a no-op. Job data is never trusted to decide what to
execute; the worker re-reads the step and re-checks the tenant and state from
PostgreSQL.

**Layer ④'s key is derived, not secret.** `exec:{executionId}:step:{nodeId}:g{gen}`
is computable by anyone who knows the execution and node ids, which is not a
confidentiality problem: it is a correlation id, sent so the customer can recognise
a retry of *their own* request. It carries no secret material, and it must not be
used as an authorization token by a customer system — the API documentation says
so explicitly, because a customer who treats it as a capability would build an
insecure integration.

**Retention limits the exposure of stored responses.** `idempotency_keys.response_body`
holds the original `201` body, which includes the execution id — not a secret, but
not something that should persist forever. The 24-hour window bounds it.

## Revisit Conditions

1. **Cross-region write authority is required.** Every layer except ④ is a
   PostgreSQL constraint, and cross-region single-writer PostgreSQL is not a thing.
   Multi-region requires either a regional home per tenant (which keeps the
   constraints local) or a different dedup mechanism entirely. This is
   [ADR-002](002-postgresql.md) revisit condition 2 seen from this angle.
2. **A customer requires a longer or unlimited idempotency window** for financial
   operations. Extend `expires_at` and accept the table growth; if the volume makes
   that untenable, move the layer to a partitioned table by month.
3. **Step effects become more numerous or more varied** such that ④ is applied to
   more than the `http` type — for example, an event-subscription step needing its
   own external key. Same mechanism, more call sites; no design change.
4. **A compliance or audit requirement demands proof that an operation was not
   performed twice.** The current design guarantees the *effect* is deduplicated at
   the customer, not that the request was sent once. A signed, retained request log
   would be a different, larger commitment.
5. **Trigger volume makes `ON CONFLICT` a measured bottleneck.** Then partitioning
   `executions` by `organization_id` moves the constraint with the data; the
   mechanism stays identical.
6. **The API gains an at-most-once guarantee requirement** for a specific endpoint
   (a payment, say). That inverts the design for that endpoint — record intent,
   reject a repeat rather than replay it — and deserves its own ADR rather than a
   special case here.
7. **A customer system is discovered to mishandle the `Idempotency-Key` header**
   (for example, treating it as a unique business key and returning a conflict).
   That is a documentation and discovery problem, not an architecture one, and the
   step type should surface `4xx` responses loudly enough to find it.

**What would *not* justify revisiting:** a preference for fewer tables; a belief that
the customer's system will handle everything; or a desire to replace unique
constraints with application checks because they are "simpler". The first trades
clarity, the second trades correctness we could have had for free, and the third
reintroduces the race the whole ADR exists to eliminate.

## References

- [006-transactional-outbox.md](006-transactional-outbox.md) — the dispatch layer
- [execution-engine.md](../architecture/execution-engine.md) §5, §7 — claim, lease, external key
- [reliability.md](../architecture/reliability.md) §4 — duplicate and lease-failure matrix
- [security.md](../architecture/security.md) §7 — webhook replay protection
- [database-erd.md](../diagrams/database-erd.md) — `idempotency_keys`, `executions`, `jobs`, `webhook_deliveries`
- IETF Idempotency-Key HTTP header draft — <https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/>
- Stripe idempotent requests (the canonical customer-side behaviour we rely on) — <https://docs.stripe.com/api/idempotent_requests>
- PostgreSQL `INSERT ... ON CONFLICT` — <https://www.postgresql.org/docs/18/sql-insert.html#SQL-ON-CONFLICT>
- PostgreSQL partial indexes — <https://www.postgresql.org/docs/18/indexes-partial.html>
- Microservices.io, idempotent consumer — <https://microservices.io/patterns/communication-style/idempotent-consumer.html>
- Exactly-once semantics, and their limits — <https://kafka.apache.org/documentation/#semantics>
