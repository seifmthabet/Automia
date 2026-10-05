# ADR-002: PostgreSQL as the System of Record

| Field | Value |
|---|---|
| Status | **Accepted** |
| Date | 2026-10-05 |
| Deciders | Engineering |
| Supersedes | — |
| Related | [system-overview.md](../architecture/system-overview.md) §2, [execution-engine.md](../architecture/execution-engine.md) §3, [reliability.md](../architecture/reliability.md) §3, [003-bullmq.md](003-bullmq.md), [006-transactional-outbox.md](006-transactional-outbox.md) |

## Context

Automia is a workflow orchestration engine. Its correctness requirements are
stronger than a typical CRUD application's, and every state decision it makes is
made against data that must survive a crash, a failover, and a concurrent writer.

The properties we actually need:

1. **Durability of workflow state.** An execution's current step, its attempt
   count, and its available-at time must never be lost. A lost state transition
   means an execution that is neither runnable nor terminal — a zombie that only
   a reconciler can clean up.
2. **Conditional writes.** The engine's core operation is compare-and-set: "if
   this step is in state `QUEUED` and this is attempt 2, mark it `RUNNING`". This
   is one `UPDATE ... WHERE state = $expected AND attempt = $attempt`. We need a
   conditional-update primitive with a reliable result.
3. **Advisory row locks with `SKIP LOCKED`.** The scheduler and the reconciler
   poll the same tables and must never block each other or double-handle a row.
4. **Strict serialisation per execution.** All transitions for one execution must
   be serialised, so a `SELECT ... FOR UPDATE` on the execution row is the unit of
   mutual exclusion ([execution-engine.md](../architecture/execution-engine.md) §5).
5. **Enforced uniqueness as an invariant.** `executions.dedup_key`,
   `UNIQUE (step_execution_id, attempt)` on `jobs`, and
   `UNIQUE (webhook_endpoint_id, external_event_id)` must be correctness
   guarantees, not application discipline.
6. **Composite foreign keys that carry the tenant.** Every child row references
   its parent with `organization_id` in the key, so a cross-tenant reference is
   *impossible at the storage layer*, not merely unlikely.
7. **Time-travel reads of a queue.** An API caller polling a slow execution must
   see a consistent snapshot, and the audit trail must be reconstructible.
8. **One transaction spanning a decision and its consequences.** When the engine
   advances a workflow it must, atomically: update step rows, update the
   execution, insert outbox rows, and record an audit event.
9. **A language for ad-hoc investigation.** When a customer reports a stuck
   execution, an operator needs to query it. This is a real operational
   requirement, not a convenience.
10. **Mature, boring, and widely operated.** This system is a product. Its
    database is not a place for novelty.

What we are explicitly **not** deciding here: the ORM ([Drizzle](#1-drizzle-orm)),
the tenant isolation strategy ([ADR-001](001-authentication-strategy.md) §4 and
[security.md](../architecture/security.md) §4), the work-delivery queue
([ADR-003](003-bullmq.md)), or the dual-write pattern
([ADR-006](006-transactional-outbox.md)). This ADR covers one question: what
holds authoritative state, and what the consequences of that choice are.

## Decision

**PostgreSQL is the sole system of record. Redis is not durable state. Object
storage is not durable state. Every piece of state that determines what Automia
does next lives in PostgreSQL.**

Concretely, this means:

| State | Authoritative in | Notes |
|---|---|---|
| Workflow definitions | `workflows`, `workflow_versions` (immutable) | [ADR-005](005-workflow-versioning.md) |
| Executions and step executions | `executions`, `step_executions` | The engine's state machine |
| Retry schedule | `step_executions.available_at` | Redis's delayed set is a *delivery* detail, not the schedule |
| Attempt counts and attempt log | `step_executions.attempt`, `step_executions.attempts_log` | Not BullMQ's `attempts` counter |
| Pending external side effects | `outbox` | [ADR-006](006-transactional-outbox.md) |
| Trigger dedup | `executions.dedup_key` unique index | [ADR-007](007-idempotency-strategy.md) |
| Dispatch dedup | `jobs` unique index, `(step_execution_id, attempt)` | [ADR-007](007-idempotency-strategy.md) |
| API replay dedup | `idempotency_keys` | [ADR-007](007-idempotency-strategy.md) |
| Identity, sessions, orgs, memberships | PostgreSQL, via Better Auth's Drizzle adapter | [ADR-001](001-authentication-strategy.md) |
| Audit log | `audit_logs` | Append-only, retention-enforced |
| Locks and leases | `lease_token`, `lease_expires_at` columns | Separate from BullMQ's internal lock |
| Customer credentials | `credentials` | Envelope-encrypted; resolved to a step at execution time ([security.md](../architecture/security.md) §5) |
| Object storage | Step payloads, large inputs/outputs | Content-addressed, referenced by URI; the *reference* is authoritative in PostgreSQL |
| Schedules and their cursors | `schedules`, `executions` | The `scheduled_for` value lives on the execution row |
| Workflow runs triggered by webhooks | `webhook_endpoints`, `webhook_deliveries`, `executions` | Replay protection by unique constraint |

**Target version: PostgreSQL 18.** We are on a currently supported major version
with logical replication, parallel query, and declarative partitioning available
if `step_executions` grows large enough to need it.

**The rules that follow from "sole system of record":**

1. **No business logic may read or write workflow state from Redis.** A code path
   that answers "what is the state of this execution?" by reading Redis is a
   defect, even if the cache is usually correct.
2. **A committed transaction is a fact.** Once a transition is committed, the
   execution is in that state, regardless of whether any queue message was
   produced, whether a worker was running, or whether the transaction's
   subsequent side effects ever happened.
3. **Redis may be rebuilt from PostgreSQL at any time.** This is the single
   strongest architectural test in this document, and it is the reason for the
   outbox. `FLUSHDB` is a recoverable incident, not a data-loss event
   ([reliability.md](../architecture/reliability.md) §3.4).
4. **PostgreSQL availability is Automia's availability.** There is no degraded
   read mode, because there is no second source for a read to fall back to.
5. **Transactions stay short and never span external I/O.** The engine opens a
   transaction, applies domain state changes, and commits. HTTP calls to
   customer systems happen outside it ([execution-engine.md](../architecture/execution-engine.md) §6).

**Isolation level: `READ COMMITTED`, the PostgreSQL default.** We do not use
`SERIALIZABLE`, and we do not use `REPEATABLE READ` for engine operations. The
engine's correctness comes from `SELECT ... FOR UPDATE` on the execution row,
which serialises advancement per execution *without* holding a predicate lock
across the whole transaction. This is a deliberate choice, not the default one:
`READ COMMITTED` with explicit row locks has the highest concurrency of the three
options and the simplest failure mode (no serialization failures to retry), and
the engine's invariant is "one transition per execution at a time", which row
locks express exactly. Where a read needs a consistent snapshot, the API uses
`REPEATABLE READ` on a read-only transaction, which is a narrow, deliberate
exception.

**Schema layout:** one database, shared schema, `organization_id` on every
tenant-owned table. Composite foreign keys carry the tenant
([security.md](../architecture/security.md) §4.3). Row-level security is
`Proposed` defence in depth, not a substitute for repository-level enforcement.

**Migrations:** Drizzle-generated SQL, applied as a single migration transaction
per release, with `pg_advisory_lock` serialising concurrent deploys. Additive
migrations only within a release; destructive changes follow expand-contract across
two releases ([Operational Impact](#operational-impact)).

## Alternatives Considered

### 1. Drizzle ORM (chosen for the *access* layer, not the store)

**Accepted as the data-access mechanism.** A thin, SQL-shaped TypeScript query
builder over PostgreSQL. No lazy-loading proxies, no identity map, no hidden
`flush()`. The SQL it emits is close to what you would write by hand, which is
exactly the property we want for a system whose correctness depends on precise
`UPDATE ... WHERE` predicates.

*Why not Prisma:* its query engine adds a Rust process to the request path, and
its higher-level API makes the exact conditional updates and locking that the
engine needs awkward to express. Prisma is a perfectly good ORM; it is a poor fit
for a hand-written state machine that needs explicit `FOR UPDATE SKIP LOCKED` and
multi-statement transactional updates. Drizzle's `sql` escape hatch means that
when we need something it cannot express, we write the SQL.

*Why not Kysely:* also a strong option, and a closer comparison. Decisive factors:
Drizzle's first-class `pgTable` schema definition is shared with the SQL
migration workflow and with Better Auth's Drizzle adapter
([ADR-001](001-authentication-strategy.md)), so there is one schema language, not
two. This is a smaller decision than it looks — the store was PostgreSQL
regardless, and the ORM choice does not change the data model.

### 2. MySQL

**Rejected.** The engine's concurrency requirements are stated in terms of
PostgreSQL primitives: `SKIP LOCKED`, `advisory_xact_lock`, `FOR UPDATE` with
`SKIP LOCKED`, and partial-expression indexes. MySQL can emulate `SKIP LOCKED`, but
loses `pg_advisory_xact_lock` — a single, robust primitive for serialising
migrations and scheduler sweeps. Also rejected on operator experience: the team
has materially more operational depth in PostgreSQL, and "boring to operate" was
a stated requirement.

### 3. MongoDB

**Rejected, and the mismatch is structural rather than preference-based.**

- **No multi-document transactions with the ergonomics we need.** MongoDB has
  multi-document transactions, but the conditional-update-and-verify semantics
  this engine is built on is a *document-shaped* operation, not a
  transaction-shaped one. Every `step_executions` row carries state, attempt,
  lease, and payload; a workflow transition touches several such rows plus the
  execution plus the outbox. That is a transaction, and modelling it in MongoDB
  means either embedding (which destroys the ability to claim one step
  concurrently) or a transaction per transition with worse operational cost.
- **No partial indexes.** `WHERE` on `UNIQUE` — used for dedup and for
  `jobs.step_execution_id` uniqueness — is a first-class, cheap concept in
  PostgreSQL and absent in MongoDB.
- **No `SKIP LOCKED`.** Direct replacement for the scheduler's poll loop.
- **Schema enforcement is application-side.** We would have to trust every writer,
  exactly the property that an enforced schema buys us.
- **Weaker fit for the relational audit model.** `audit_logs` with actor, target,
  and JSONB detail is naturally relational.

The honest summary: MongoDB's document model is a poor match for a state machine
whose transitions are the product.

### 4. Redis as the primary store (plus a persistence service)

**Rejected — decisively.** This is the decision the whole architecture exists to
avoid.

- **Durability is a configuration setting, not a guarantee.** Redis persistence is
  RDB snapshots and/or AOF with an `everysec` (one second) fsync policy. Under
  the default policy, up to one second of committed state can be lost on host
  failure. For an execution engine, a lost state transition is a lost fact.
- **Replication is asynchronous.** A promoted replica does not have the writes
  that were in flight at failover.
- **A single key holds a workflow's live state.** Concurrent transitions to
  different steps of the same execution require either one key per execution
  (contention) or locks and retry loops, replacing `SELECT ... FOR UPDATE` with
  distributed-lock logic we would then have to prove correct.
- **There is no query language for the audit and investigation requirement.** A
  reported "my execution is stuck" case needs a filter, a join and a time range.
- **It makes the product's correctness depend on a component whose primary
  reputation is speed, not durability.** The narrative inverts: Redis is at risk
  of being treated as a cache that must be correct, because it is the truth.

**Where Redis is right:** work delivery, short-lived coordination, caching, and
rate-limit counters ([ADR-003](003-bullmq.md)). All four are reconstructible.

### 5. Kafka or a dedicated event log as the source of truth

**Rejected on proportionality.** An event-sourced design — Kafka plus a projection
into PostgreSQL — is architecturally interesting, and for a genuinely
event-sourced product it would be the better answer. Here it is not:

- The authority for "what should this execution do next" would be a *projection*,
  and a projection can be behind. The engine's central read is a conditional
  update against current state, which is a poor fit for a lagging projection.
- Operationally it is a second stateful system to deploy, monitor, back up and
  reason about during an incident. Our stated requirement was fewer moving parts.
- It does not actually reduce the number of moving parts, because we would still
  need PostgreSQL for the read model and for the audit queries.

If Automia ever needs sub-second fan-out to a large number of independent
consumers, or long-term replay as a product feature, this should be revisited.
Neither is a v1 requirement.

### 6. DynamoDB / a managed NoSQL store

**Rejected** for the same structural reasons as MongoDB, plus single-table-design
complexity and cost modelling. Conditional writes are available
(`ConditionExpression`), which is the one property we would need — but the
multi-row transactional pattern, the query ergonomics, and the operational cost
model all favour PostgreSQL at this scale.

## Why

**1. The engine's central operation is a relational conditional write.** The
whole design rests on `UPDATE step_executions SET state = 'RUNNING', ... WHERE
id = $id AND state = 'QUEUED' AND attempt = $attempt`, whose `rowCount` is the
claim. This is not a pattern PostgreSQL is merely able to express; it is a pattern
it implements optimally, with an index, an atomicity guarantee, and a return
value that tells us whether we won. Every alternative either emulates it (locks
plus retries) or weakens it (application-side read-then-write, which is racy).

**2. Serialisation per execution maps to a row lock.** `SELECT ... FROM
executions WHERE id = $1 FOR UPDATE` is one line, it is exactly the mutual
exclusion we need, and it is scoped to the execution rather than the table. The
lock ordering rule (execution first, then step rows by ascending id,
[execution-engine.md](../architecture/execution-engine.md) §5.2) makes deadlock
structurally impossible, which is worth more than the raw throughput of any
alternative.

**3. The invariants belong in the schema.** `dedup_key` unique,
`(step_execution_id, attempt)` unique, `(webhook_endpoint_id,
external_event_id)` unique, composite tenant-carrying foreign keys. Each of these
is a correctness guarantee that becomes *unbypassable* by any writer, including
the reconciler, a future import tool, and an engine bug. In an application-layer
implementation, each is a `SELECT` followed by an `INSERT` and a hope. The
scheduler's need for "claim a pending step, skip the ones someone else has"
([ADR-003](003-bullmq.md)) is the same story: `FOR UPDATE SKIP LOCKED` is the
primitive, not a pattern to reimplement.

**4. The dual-write problem is solvable here and nowhere else.**
[ADR-006](006-transactional-outbox.md) works because the state change and the
intent-to-notify commit in the same transaction. That requires a store with
multi-row transactions, and it is the reason we can state that Redis loss is
recoverable. A store without adequate transactions forces a choice between losing
notifications and a bespoke saga — both worse.

**5. MVCC gives the snapshot isolation the API needs, for free.** A client polling
an execution while the engine advances it sees a consistent view, via the default
`READ COMMITTED` semantics on a single `SELECT`, without a replica, a cache, or a
snapshot table. Adding a read replica later is a scaling decision we can defer
precisely because it is not a correctness one.

**6. "Sole system of record" is a cheap invariant to hold.** It converts an
entire class of distributed-systems problems into ordinary single-database
transactions. The cost is a hard availability dependency on one service, and we
have stated it plainly in [system-overview.md](../architecture/system-overview.md)
§6.1 rather than disguising it. But the systems we are *not* building — a
saga coordinator, a projection pipeline, a consistency-repair service — are
individually cheap to add later and expensive to add while the data model is still
young.

**7. Operational depth is a feature.** This is a product. On-call at 3am for a
PostgreSQL incident is a well-trodden path; inventing a recovery procedure for a
custom state store at the same time is not. The requirement was a system whose
recovery is boring, and the most direct way to achieve that is to keep the number
of things that can lose authoritative state at one.

## Consequences

### Positive

- **The engine's invariants are enforced by the database.** Dedup, dispatch
  uniqueness, and tenant-consistent references cannot be bypassed by any code
  path.
- **The outbox pattern is sound**, so Redis is fully reconstructible
  ([ADR-006](006-transactional-outbox.md)).
- **The scheduler and reconciler never block each other**: `FOR UPDATE SKIP LOCKED`
  partitions work without coordination.
- **Operational queries are first-class.** Every customer question ("why is this
  execution waiting?", "which step failed most often last week?") is a query, not
  an engineering task.
- **One transaction covers a decision and its consequences** — state changes, the
  execution pointer, outbox rows, and the audit event — which is what makes an
  execution's history trustworthy.
- **Schema migration is a solved problem**: additive-first expand/contract, single
  transaction per release, advisory-lock serialisation.
- **One dependency for durability, transactions, and querying** instead of a
  database plus a log plus a projection pipeline.
- **Advisory locks are available** for the cases where we need cross-transaction
  mutual exclusion — migrations, and cross-execution sweeps that must not
  interleave with a per-execution transition.
- **Cheap, available scaling levers** that are not correctness levers: read
  replicas for API reads, connection pooling, partial indexes, then declarative
  partitioning of `step_executions` by `created_at` if it becomes the largest
  table.

### Negative

- **A hard dependency, with a correspondingly large blast radius.** An outage or
  unreachable primary means total unavailability, including read-only traffic. We
  accept this explicitly; the alternative is a second source of truth.
- **Write throughput is bounded by the primary.** The engine is not
  write-horizontal. Mitigated by the fact that per-execution work is serialised
  anyway, so the natural scaling axis is *across* executions, and that can be
  sharded by tenant later if needed.
- **Hot-row contention on `executions`** during high fan-out. A wide parallel
  workflow can generate many step transitions in one execution, all serialised on
  one row. Bounded by design; mitigated by batching transitions per transaction
  and by keeping transactions short.
- **`step_executions` will be the largest table**, growing with
  `executions × steps`. Requires retention and archival work early
  ([ADR-004](004-modular-monolith.md)), not as an afterthought.
- **`SKIP LOCKED` and long transactions interact badly.** A slow transaction
  holding an execution lock stalls that execution's progress. Mitigated by the
  rule that no transaction spans external I/O — a hard rule, because it is the
  single most important invariant in the engine.
- **`READ COMMITTED` requires explicit reasoning** for any read that spans multiple
  rows and must be consistent. The engine avoids it by serialising on the
  execution row; the API uses `REPEATABLE READ` where it genuinely needs a
  snapshot. Neither is automatic, so this is a review concern.
- **No in-database execution of customer code.** PL/pgSQL and stored procedures
  would reduce round-trips but put business logic in a second language with its
  own deployment and testing story. We keep logic in TypeScript and accept the
  round-trips.
- **Vertical scaling first.** Connection limits and storage are the levers before
  distribution. Acceptable at our scale; stated so it is not a surprise later.
- **Drizzle does not mean we avoid SQL.** When the engine needs
  `FOR UPDATE SKIP LOCKED` or a partial index, we write SQL. Drizzle is a
  convenience, not a constraint, and the team must be comfortable reading query
  plans.

## Operational Impact

**Provisioning:** one managed instance (`rds.aws` or equivalent) with
multi-AZ, `synchronous_commit = on`, and automated backups with
point-in-time recovery. `synchronous_commit` matters: the outbox guarantees a
state change and its intent commit together, and an asynchronous commit policy
would let the machine acknowledge a transaction that a crash later erases. We keep
`on`.

**Migrations, applied per release:**

1. Acquire `pg_advisory_lock(<fixed key>)` so two deploys cannot migrate
   concurrently.
2. Apply generated Drizzle SQL in a single transaction.
3. Release the lock.

**Expand/contract across two releases** for anything destructive, so that old and
new code both work against the schema during a rolling deploy:

| Release | Step |
|---|---|
| N | Add new nullable columns / new tables. No behaviour change. |
| N | Backfill. Batch it, outside any request path. |
| N | Dual-write and read the new shape, if the change is not atomic. |
| N+1 | Switch reads to the new shape; verify. |
| N+1 | Drop the old column or table, in a later release if the data is large. |

**Routine operations:**

| Operation | Frequency | Notes |
|---|---|---|
| Verify migrations applied | Per deploy | Fail the deploy, not the health check |
| `pg_stat_statements` review of engine queries | Weekly during early development | Look for `step_executions` scans that are not using an index |
| `VACUUM` / autovacuum health | Continuous, monitored | High-churn state columns; a table-level autovacuum scale factor tuned down for `step_executions` |
| Archive or partition `step_executions` | `Proposed` — trigger point to be defined from real data volume | Before it hurts |
| Restore drill | Quarterly | A backup that has never been restored is a hypothesis |

**Backup and RPO/RTO:** point-in-time recovery with continuous WAL archiving.
Recovery point is bounded by the archive interval. Recovery time is bounded by
restore speed and is **to be measured during performance testing**; no target is
claimed here. The recovery runbook is in
[reliability.md](../architecture/reliability.md) §6.2.

**Connection management:** a pooler (PgBouncer in transaction mode) between the
application and the primary, sized so that `max_connections` is not the thing
that falls over. Every connection is a scarce shared resource, and a leaked one
degrades the whole system.

**Monitoring — the signals that matter, and their thresholds.** Nothing here is a
fabricated SLO; these are *alert conditions* chosen because they indicate a
correctness risk, not because they breach a service level:

| Signal | Alert condition | Meaning |
|---|---|---|
| Transactions waiting on a lock | Any sustained wait > 2s | The no-external-I/O-in-transactions rule is being broken |
| `executions` stuck non-terminal past SLA-by-tenant | Any execution beyond its `expires_at` | Needs a terminator; see [ADR-004](004-modular-monolith.md) |
| `outbox` oldest unpublished row age | Growing | The publisher is wedged, or the engine cannot enqueue |
| `step_executions` with `lease_expires_at` in the past | Count > 0 beyond a grace period | Leases not being renewed; a stalled worker |
| Replication lag | Threshold set after measurement | Read replica staleness |
| Autovacuum lag on `step_executions` | Growing | State churn outpacing maintenance |
| Connection pool saturation | Sustained high waiters | Query or transaction-duration regression |

**Failure modes:**

| Failure | Behaviour | Blast radius |
|---|---|---|
| Primary unavailable | Total unavailability. Fail fast, do not queue writes. | All traffic |
| Primary fails over | Automatic with multi-AZ. In-flight transactions fail; callers retry with `Idempotency-Key` ([ADR-007](007-idempotency-strategy.md)) | Seconds of partial failures |
| Redis lost | **None.** Rebuilt from `outbox` and `jobs` | None, after reconciliation |
| Region loss | Regional outage. Out of scope for v1; documented in [system-overview.md](../architecture/system-overview.md) §12 | — |
| Slow query on one execution | That execution's transitions stall. Others unaffected | Single execution, because locking is per execution |
| Autovacuum starvation | Table bloat, degrading all queries on the table | Broad, gradual — mitigated by tuning and monitoring |
| Migration applied incorrectly | Rolling deploy may be broken | Mitigated by expand/contract and by migrations being additive |
| Data corruption in PostgreSQL | Restore from backup; `audit_logs` and the outbox do not survive restoration | Total state rollback to the recovery point. Bounded by RPO |
| `synchronous_commit = off` (misconfiguration) | Commits acknowledged but not durable; a crash erases recent facts | Silent, and severe. Therefore monitored explicitly |

**Runbook: "executions are stuck."** In order: (1) is the primary healthy and
accepting connections; (2) is there lock contention (`pg_locks` joined to
`pg_stat_activity`); (3) is the outbox draining; (4) are workers alive and are
leases being renewed; (5) is anything in `RETRYING` with an `available_at` in the
future. Steps 1–3 are PostgreSQL-side and can be answered while the API is
unhealthy, which is the main reason for operational fluency with the database.

## Security Impact

**Tenant isolation.** Shared schema with mandatory `organization_id`, and
composite tenant-carrying foreign keys so a cross-tenant reference is rejected by
the database. This is the primary defence, and it is why the schema is
compromised-by-design against a bug in one repository.

**Row-level security** is `Proposed` defence in depth
([security.md](../architecture/security.md) §4.6). It is explicitly *not* relied
upon: `FOR UPDATE SKIP LOCKED` does not compose safely with RLS-based
tenant filtering in the same query for the scheduler's claim path, and a
misconfigured `FORCE ROW LEVEL SECURITY` can turn a quiet bug into an outage.
`organization_id` indexes on every tenant-owned table keep the performance
argument for RLS neutral when it is introduced.

**The migration path is privileged.** A migration can read and write every
tenant's data, so it is treated as production code: reviewed, generated, and
applied from a controlled identity. Migration credentials are not held by the
application at runtime.

**The database is a high-value target.** It contains session tokens, API key
hashes, encrypted credential snapshots and the full audit trail. Consequences:
enforced TLS to the primary, encryption at rest from the provider, per-environment
credentials with least privilege, no direct public access, and the
`credentials` envelope encryption (`ciphertext` + `wrapped_dek`) described in
[security.md](../architecture/security.md) §5.3. A dump of the database must not
be a usable credential store, which is why API keys are hashed and step
credentials are envelope-encrypted.

**Auditability.** `audit_logs` is append-only and written in the same transaction
as the action it describes. Application-level rules prevent update and delete;
the retention sweeper is the only legitimate remover, and it writes its own audit
row. An attacker with write access to the application can attempt to erase
history, which is why the append-only constraint plus separate retention
authority matter.

**Backups inherit the data's sensitivity.** Restores land in the same
confidentiality boundary as the primary. Backup retention is bounded and
documented, because an unbounded, long-lived, under-protected backup is a
long-lived copy of every tenant's credentials.

**Connection strings are secrets.** Held in the secret manager, never in the
repository, rotated, and redacted from logs. The `pg` driver logs connection
parameters in error messages, so the log-redaction layer is what stands between a
misconfigured DSN and a log pipeline.

## Revisit Conditions

1. **Write throughput on a single primary becomes the measured bottleneck** for a
   customer, at a scale where per-execution serialisation is not the limiting
   factor. Answer would be: sharding by `organization_id`, or a purpose-built log.
   Not a prediction — a trigger.
2. **We need cross-region write availability.** A single PostgreSQL primary in one
   region is a single region of failure. This would force either a
   multi-writer store or an explicit statement that single-region is the product's
   availability boundary, decided commercially.
3. **An enterprise requirement demands a customer-managed database** (their own
   PostgreSQL cluster, or a deployment into their VPC). PostgreSQL is the most
   portable choice we could have made; this would be a deployment change, not a
   data-model change.
4. **Retention or archival of `step_executions` becomes a product requirement**
   (for example, legally required multi-year execution history). Currently a
   sweeper; would become partitioning plus a cold-tier store.
5. **PostgreSQL reaches a licensing or governance change** that conflicts with how
   we intend to use it. PostgreSQL is open source; a change would need assessment.
6. **A real incident shows a PostgreSQL primitive we relied on misbehaving** under
   contention. This would prompt a specific fix, not a store change.
7. **`step_executions` exceeds what a single node handles comfortably** even after
   indexing and partitioning. Then the question is whether to partition
   differently, or whether the model itself is wrong.

**What would *not* justify revisiting:** a preference for schemaless flexibility, a
belief that MongoDB or DynamoDB would be simpler, or a desire to avoid
schema-migration discipline. Each trades a real correctness property for an
ergonomic one. The specific, concrete triggers above are the ones that would
warrant reopening the decision; a general sense that "SQL feels old" is not among
them.

## References

- PostgreSQL 18 documentation — <https://www.postgresql.org/docs/18/>
- `SELECT ... FOR UPDATE` and lock modes —
  <https://www.postgresql.org/docs/18/sql-select.html#SQL-FOR-UPDATE-SHARE>
- Advisory locks — <https://www.postgresql.org/docs/18/explicit-locking.html#ADVISORY-LOCKS>
- Transaction isolation levels —
  <https://www.postgresql.org/docs/18/transaction-iso.html>
- `INSERT ... ON CONFLICT` —
  <https://www.postgresql.org/docs/18/sql-insert.html#SQL-ON-CONFLICT>
- Partial indexes — <https://www.postgresql.org/docs/18/indexes-partial.html>
- Explicit locking, including deadlock avoidance — <https://www.postgresql.org/docs/18/explicit-locking.html>
- Row-level security — <https://www.postgresql.org/docs/18/ddl-rowsecurity.html>
- Continuous archiving and point-in-time recovery —
  <https://www.postgresql.org/docs/18/continuous-archiving.html>
- Drizzle ORM — <https://orm.drizzle.team/docs/overview>
- Drizzle PostgreSQL transactions — <https://orm.drizzle.team/docs/transactions>
- PostgreSQL `synchronous_commit` —
  <https://www.postgresql.org/docs/18/wal-async-commit.html#WAL-SYNCHRONOUS-COMMIT>
