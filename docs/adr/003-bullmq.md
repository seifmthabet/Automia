# ADR-003: BullMQ on Redis for Work Delivery

| Field | Value |
|---|---|
| Status | **Accepted** |
| Date | 2026-10-05 |
| Deciders | Engineering |
| Supersedes | — |
| Related | [system-overview.md](../architecture/system-overview.md) §2, [reliability.md](../architecture/reliability.md) §3, [002-postgresql.md](002-postgresql.md), [006-transactional-outbox.md](006-transactional-outbox.md), [007-idempotency-strategy.md](007-idempotency-strategy.md) |

## Context

Automia executes workflow steps. Steps are long-running relative to a request:
they may be HTTP calls with a 30-second timeout, a loop over a paginated API, or
a retry that must not happen for six hours. The requirements:

1. **Decouple acceptance from execution.** `POST /executions` must return in tens
   of milliseconds whether the workflow is 3 steps or 300. The API commits the
   execution and returns. Nothing runs in the request.
2. **Retry with delay.** A step that fails must not be retried immediately. It
   retries on a backoff schedule that can be measured in hours.
3. **Survive worker crashes mid-step.** A worker OOMs or is killed while a step is
   in flight. That step must be redelivered.
4. **Distribute workers.** Steps are CPU-light and I/O-bound, so many workers, and
   horizontal scale-out without a coordinator.
5. **Fair sharing.** One customer's runaway loop must not starve everyone else.
6. **Fairly common technology.** The team can operate it, and its failure modes
   are documented rather than folklore.
7. **Explicitly not authoritative.** Per [ADR-002](002-postgresql.md), Redis
   holds no state that determines what Automia does next. Everything here must be
   reconstructible from PostgreSQL.

Requirement 7 is the defining one. It means we are choosing a *delivery* mechanism,
not a *state* store, and every design decision below follows from keeping that
line clean.

## Decision

**Use BullMQ, backed by Redis, for work delivery only. One queue named `steps` in
v1. Only the outbox publisher calls `queue.add`. No component other than the
publisher ever enqueues work.**

The split of responsibilities is absolute:

| Question | Answered by | Not by |
|---|---|---|
| Which step should run next? | The engine, in PostgreSQL | BullMQ |
| How many attempts has it had? | `step_executions.attempt` | BullMQ's `attempts` counter |
| When should the next attempt happen? | `step_executions.available_at` | BullMQ's delayed set |
| Is this dispatch already sent? | `UNIQUE (step_execution_id, attempt)` on `jobs` | BullMQ's `jobId` de-duplication |
| Has the work been delivered? | BullMQ | — |
| Who is executing it right now? | BullMQ's lock, plus our `lease_token` | — |

### Configuration decisions and why each is what it is

```ts
// packages/queue/src/queues.ts
export const STEP_QUEUE = 'steps';

export const stepQueueOptions = {
  connection: redis,
  defaultJobOptions: {
    // Transport-level redelivery only. NOT the domain retry policy.
    attempts: 3,
    backoff: { type: 'exponential', delay: 1_000, jitter: 0.5 },
    removeOnComplete: { age: 86_400, count: 10_000 },
    removeOnFail: { age: 604_800 },
  },
};
```

| Setting | Value | Reason |
|---|---|---|
| `attempts` | `3` | Tolerates worker death and stalled jobs. Deliberately *not* the domain retry limit — a workflow configured for 8 attempts must still make 8 domain attempts, and this must not be the thing that caps it. |
| `backoff` | exponential, 1s base, 50% jitter | Spreads a thundering herd after a shared dependency fails. Not the domain backoff either. |
| `removeOnComplete` | keep 24h / 10k | Enough for debugging, bounded memory. Postgres still holds the truth. |
| `removeOnFail` | keep 7d | Failed jobs are the ones people ask about. |
| Queue name | `steps` only | See below. |

**Why one queue and not one per step type or per priority tier.** A tempting design
is separate queues for `http`, `long_running`, and `retries`, or a priority ladder
of several named queues. Rejected: each additional queue needs its own workers, its
own concurrency configuration, and its own stalled-job monitoring, and a worker
attached to a low-priority queue cannot help with a high-priority backlog. With
one queue, **priority is a worker-side admission decision**, not a queue
topology: workers reserve part of their concurrency for urgent jobs
(`priority: 0–2`) and keep the remainder for the backlog. That achieves the
separation we want with none of the operational cost, and it stays adjustable
without a migration.

**Why `attempts: 3` and not `attempts: 1`.** A stalled job (worker died mid-step)
is retried by BullMQ automatically. If we set `attempts: 1`, a worker crash
permanently fails the delivery and the step would sit in `QUEUED` until the
reconciler noticed. `attempts: 3` means infrastructure-level flakiness is absorbed
inside the queue, and only genuine, repeated execution failures surface as failed
jobs for the reconciler. **This is the important sentence in this ADR:** BullMQ
retries and domain retries are different mechanisms with different counters, and
conflating them is how engines silently lose attempts.

**Why the domain backoff lives in PostgreSQL.** Because a delayed set in Redis is
not a schedule anyone can query, and a customer's question "when will this retry?"
must have an answer that survives Redis. `step_executions.available_at` answers
it. The consequence is that a retry-after-6-hours is *not* a BullMQ delayed job
sitting in Redis for 6 hours: the job is removed on failure, and the reconciler
picks up `step_executions` rows whose `state = 'RETRYING'` and whose `available_at
<= now`. The queue is therefore always short-lived, and memory usage is a function
of in-flight work rather than of pending retry windows.

**Why `jobId` is the UUID and not the external idempotency key.** BullMQ's
`jobId` de-duplication is scoped to jobs that still exist in the queue. Once a job
is removed (24h after completion, or on failure), a job with the same `jobId` can
be added again. That makes `jobId` sound for *transport* de-duplication and
unsound as *business* idempotency. Business idempotency is a unique constraint in
PostgreSQL ([ADR-007](007-idempotency-strategy.md)), which survives any amount of
Redis flushes. The `steps` queue therefore additionally avoids colon-containing
`jobId` values: `Redis` keys are constructed as `<queue>:<jobId>`, so a colon in
`jobId` changes the key layout, and the external key
`exec:{executionId}:step:{nodeId}:g{generation}` is exactly the kind of string
that would do that. The `jobs.id` UUID is used as `jobId`; the external key is
carried in job data and used as the header value to customer systems.

**Why the reconciler, not the queue, is the safety net.** BullMQ cannot tell us
"there is a step that should have run and no job exists". PostgreSQL can
([reliability.md](../architecture/reliability.md) §4.1). A periodic reconciler
compares `step_executions` in `QUEUED`/`RETRYING` against `jobs` and re-dispatches
through the outbox, deduped by the unique constraint. This is what makes
`FLUSHDB` a recoverable incident.

**What we explicitly do not use BullMQ for:** flows, rate limiting, or delayed
retries. BullMQ's `FlowProducer` and rate limiters are attractive and rejected:
they encode orchestration and scheduling in Redis, which violates requirement 7 —
the orchestration state would live in a non-durable store and the engine would
have to query Redis to know what to do next.

## Alternatives Considered

### 1. PostgreSQL as the queue — `FOR UPDATE SKIP LOCKED`

Rejected, and this is the closest call in the set.

**The advantages are real:** zero additional infrastructure, perfect consistency
with the domain state, and the claim pattern is one statement:

```sql
UPDATE step_executions SET state = 'RUNNING', worker_id = $1
WHERE id = (
  SELECT id FROM step_executions
  WHERE state IN ('READY', 'RETRYING') AND available_at <= now()
  ORDER BY available_at
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
RETURNING *;
```

**Why it is still wrong here:**

- **No delayed-set semantics.** The same pattern is used, but a 6-hour retry means
  polling a table every N seconds and finding nothing, 6 hours' worth of times.
  The poll interval is then a compromise between latency and cost, and the
  compromise is tuned for *someone else's* workload.
- **No stalled-job detection.** A PostgreSQL-only queue must hand-roll lease
  expiry and requeue on worker death. We would be building BullMQ's hardest and
  most operationally subtle feature.
- **The poll loop becomes load on the system of record.** Workers hammering
  `step_executions` with claim queries compete with the engine for the same
  locks, and a slow claim loop degrades *transitions*, not just throughput.
  Keeping the scheduling database out of the hot polling path is the main
  architectural reason to have Redis at all.
- **No per-job visibility tooling.** "Why is this job not running?" is a queue
  problem, and Redis answers it.

**Kept, deliberately, as a partial answer.** Long-horizon *scheduling* (a
six-hour wait) lives in PostgreSQL and is picked up by a sweep
([ADR-006](006-transactional-outbox.md)), not by a Redis delayed job. So we take
PostgreSQL for *scheduling* and BullMQ for *immediate delivery and crash
recovery* — the two problems have different requirements, and they get different
tools.

### 2. Temporal

**Rejected on operational proportionality, not capability.** Temporal is an
excellent durable-execution engine, and if Automia's product *were* "durable
execution as a service" with arbitrary workflow code, it would be the right
answer: history, replay, and human-in-the-loop are first-class.

Rejected because:

- **It introduces its own persistence database and execution model.** Workflow
  logic would become Temporal workflow code, and the `workflow_versions` /
  `executions` / `step_executions` model that this product's API exposes would
  sit *alongside* Temporal's history rather than being it. Two sources of truth
  for "what happened", which is the exact failure class
  [ADR-002](002-postgresql.md) exists to prevent.
- **Code-first workflows** conflict with the declarative JSON workflow model
  ([ADR-005](005-workflow-versioning.md)), which is what makes workflows
  versionable, diffable, and editable in a UI.
- **A separate service to deploy, version, monitor and upgrade** — plus
  Cassandra or PostgreSQL behind it. The stated requirement was to minimise moving
  parts in the hot path.

**The trigger to revisit** is in [Revisit Conditions](#revisit-conditions), and it
is specific: if requirements arrive for arbitrary user code, a human-in-the-loop
form, or month-long durable waits with full history replay, that is Temporal's
home ground and the answer changes.

### 3. `pg-boss` (PostgreSQL-backed job queue)

Genuinely competitive, and the alternative that most directly threatens the
consistency story. Rejected on capacity ceiling: it uses `SKIP LOCKED` polling
against your own tables, so it inherits limitation 1 (no native delay, hand-rolled
stall detection, load on the system of record). It would move the claim loop from
Redis to PostgreSQL, which is the part we specifically wanted to keep off the
system of record.

### 4. SQS (and SNS/SQS fan-out)

A credible choice if running Redis were off the table. Rejected because of
requirement 7 and one specific property: SQS is a strong delivery channel, but
because we need payload versioning, the outbox pattern, and a Postgres-side claim
for the retry sweep, we would build the same two-layer structure anyway — while
paying for a network hop per job, accepting a 20-message-per-poll receive limit,
and gaining a queue we cannot inspect locally during an incident.

### 5. Kafka or Redpanda

Rejected: excellent for event *streaming* and fan-out to many consumers, and we
have neither. We have exactly one consumer per job and need per-job retry with
delay, which is a queue's job, not a log's. Adds a durable broker whose retention
we would pay for and not use.

### 6. A hand-rolled Redis queue

Rejected, and the failure mode is worth naming: hand-rolled queue code is where
duplicate deliveries, lost jobs, and requeue bugs live. It looks simple and is
not. The specific gap is stalled-job detection — noticing that a worker holding a
job has died — which requires heartbeats, a reaper, and careful reasoning about
lock expiry. BullMQ has this, tested, and documented.

## Why

**1. It keeps the hot path off the system of record.** The engine's job is
transitions, and transitions want uncontended short transactions. Worker polling
wants high-frequency `SKIP LOCKED` scans. Running both on the same database means
one degrades the other. Separating them means worker throughput can be scaled and
can fail without touching transition latency.

**2. Delayed jobs and stalled-job detection are solved and are genuinely hard.**
Six-hour backoffs and worker-death detection are the two features we cannot skip,
and both are exactly where hand-rolled implementations go wrong.

**3. The "not authoritative" line stays drawable.** Because the domain owns
attempt count and `available_at`, the only thing Redis can lose is *delivery*.
That is what makes `FLUSHDB` a non-event and what makes the reconciler's job
writable in an afternoon rather than a research project.

**4. It is well-understood, well-documented, and widely operated.** Team
familiarity is a real engineering asset, and Redis is a commodity dependency in
every hosting environment. A dedicated service with a licensing model we would
have to reason about is not.

**5. One queue keeps worker scaling trivial.** A worker process runs
`new Worker(STEP_QUEUE, handler, { concurrency })` and it drains whatever is
urgent. Multiple queues would multiply the processes and dashboards for no gain.

## Consequences

### Positive

- **Acceptance and execution are decoupled.** The API commits and returns.
- **Delayed work is a queue feature, not a polling loop.** Retries do not
  generate database load while waiting.
- **Worker crashes are absorbed.** Stalled jobs are redelivered up to
  `attempts: 3` without engine involvement.
- **Scaling is `n` worker processes.** No coordinator, no leader election.
- **Redis is fully disposable.** `FLUSHDB` loses deliveries; the reconciler
  rebuilds them from `step_executions` and `jobs` ([ADR-006](006-transactional-outbox.md)).
- **Priority without queue topology.** Urgent work is a worker-side admission
  decision.
- **One language, one toolchain.** Same TypeScript, same observability stack.
- **Failure isolation is explicit.** A Redis outage stops *dispatch*; committed
  executions and their state are intact in PostgreSQL, and the outbox drains when
  Redis returns.

### Negative

- **A second stateful dependency**, and Redis persistence is not durability
  ([ADR-002](002-postgresql.md) §2). We now have one system whose loss is
  catastrophic and one whose loss is an inconvenience, and operators must know
  which is which.
- **Two retry mechanisms must be explained, forever.** The single most likely
  source of confusion in this codebase is someone reading `attempts: 3` and
  concluding the workflow retries three times. Mitigated by the comment in
  `queues.ts`, by this ADR, and by the reconciler being the only retry authority.
- **Duplicate delivery still happens.** BullMQ is at-least-once, and `jobId`
  de-duplication is not business idempotency. Every handler must be
  idempotent ([ADR-007](007-idempotency-strategy.md)).
- **`attempts: 3` is an under-specified ceiling for crashes.** A step whose
  execution takes longer than `lockDuration` and cannot renew will be redelivered
  three times and then failed, even though the domain may allow ten attempts.
  Consequence: the reconciler must treat "job exhausted" as *resume the domain
  attempt*, not "fail the execution".
- **No cross-queue ordering or global fairness guarantees.** In-flight limit per
  organization is enforced in the engine, not the queue.
- **The delayed-set behaviour is invisible to the API.** "Show me the delayed
  jobs" is not answerable from Redis; the API answers from `available_at`.
- **A minor syntax gotcha:** using a colon-containing `jobId` changes the Redis
  key layout. The `jobs.id` UUID convention removes the class of problem
  ([Decision](#why-jobid-is-the-uuid-and-not-the-external-idempotency-key)).
- **Redis memory is unbounded in principle.** Bounded by `removeOnComplete` /
  `removeOnFail` and by the reconciler, but worth alerting on.

## Operational Impact

**Components:** a managed Redis deployment with a persistence policy set, TLS, and
`maxmemory-policy noeviction`. **`noeviction` is deliberate**: silently evicting a
job would lose work, and losing a job is recoverable but alarming. We would rather
fail the enqueue and have the outbox retry.

**Deployment:** unchanged. No new container, port, or process; the queue client is
a library in `apps/worker` and the publisher in `apps/scheduler`.

**Configuration and what it actually means:**

| Setting | Value | Effect |
|---|---|---|
| `maxmemory` | Set to a budget | Under pressure, enqueues fail rather than evict |
| `maxmemory-policy` | `noeviction` | No silent job loss |
| Persistence | Managed, on | Insurance, not durability |
| `stop-writes-on-bgsave-error` | `yes` | We stop accepting jobs rather than run with a broken snapshot |
| `notify-keyspace-events` | `Ex` | Needed by BullMQ completion/blocked-event delivery |
| TLS | Required | If unencrypted Redis is reachable from the app network, jobs and payloads are exposed to it |

**Job payload discipline.** Jobs carry identifiers, not payloads: `stepExecutionId`,
`executionId`, `organizationId`, `attempt`, `leaseToken`, `externalKey`. Not the
workflow body, not the step's input data, not credentials. The worker re-reads
what it needs from PostgreSQL. This keeps Redis memory small, keeps
[security.md](../architecture/security.md) §5's redaction story simpler, and means
a Redis dump is not a copy of customer data.

**Routine operations:**

| Operation | Frequency | Notes |
|---|---|---|
| Verify backups on the Redis deployment | Monthly | Do not rely on them |
| `INFO memory` and eviction counters | Continuous alert | Any eviction of a job key is a bug |
| Queue depth by priority | Continuous | A growing wait time here is a symptom, not a cause |
| Oldest waiting job age | Alert | The user-facing latency signal |
| Failed-job count | Alert | A rising count means the reconciler is being asked to clean up |
| `stalled` event rate | Alert | Rising stalls mean workers are being overloaded or killed |
| Verify reclaim of a dropped message in a staging drill | Quarterly | The only way to trust the reconciler is to have watched it work |

**Monitoring:**

| Signal | Alert condition | Meaning |
|---|---|---|
| Redis connection errors | Any sustained increase | Dispatch is stopping; the outbox grows |
| Outbox oldest unpublished row | Growing ([ADR-006](006-transactional-outbox.md)) | Redis unavailable or the publisher is wedged |
| Waiting job count / oldest age | Growing | Worker capacity is insufficient |
| Active worker count | Below desired | Deploy or crash loop |
| Failed jobs | Any sustained non-zero | Reconcile and inspect |
| Stalled events | Sustained non-zero | Workers blocked past `lockDuration` |
| Scheduler sweeps where nothing was claimed | Sustained | Claim query is not matching; a filter bug, not a capacity issue |

**Failure modes:**

| Failure | Behaviour | Blast radius |
|---|---|---|
| Redis unavailable | Outbox rows accumulate unpublished. The API still accepts executions. Committed state is safe. | **New step execution stops; no data loss.** Recoverable |
| Redis data loss (`FLUSHDB`, failover past AOF) | Same as unavailable, then the reconciler re-dispatches from `step_executions` | Delivery only. Minutes of latency |
| A worker is OOM-killed mid-step | BullMQ detects the stale lock and redelivers (up to `attempts: 3`) | That step re-executes; `externalKey` prevents duplicate side effects |
| Every worker dies | Jobs wait. On recovery they are delivered | Delivery delay only |
| Job payload too large | Enqueue fails; the row stays in `outbox` and retries | Dispatch of that job. Avoided by the payload discipline above |
| A handler throws a non-retryable error | The job fails; the engine marks the step `FAILED` and applies `errorPolicy` | Correct behaviour |
| Lock renewal fails (event loop blocked) | Job goes stalled, is redelivered, and the domain's `lease_token` claim fails — so the *previous* attempt's late write cannot clobber the new one | Safe, by design ([ADR-007](007-idempotency-strategy.md) §6) |
| Redis eviction of a job key (misconfiguration) | Silent delivery loss; the reconciler notices and re-dispatches | Recoverable, but why `noeviction` is mandatory |

**Runbook: "executions are stuck in QUEUED."** (1) Is the outbox draining — if
not, the problem is upstream of the queue (Redis or the publisher). (2) Are there
jobs for those `step_executions` rows? If not, the dispatch insert was lost; the
reconciler will rebuild it, and the `jobs` table tells us whether dedup is
blocking it. (3) Are workers alive and is their concurrency saturated? (4) Are
leases being renewed? (5) Are jobs failing repeatedly — a failing job with
`attempts: 3` exhausted needs the reconciler to resume the *domain* attempt.

## Security Impact

**Redis is an untrusted-network dependency.** It holds job identifiers, so the
blast radius of a Redis compromise is an attacker who can *enqueue* work, not one
who can read customer data — a meaningful reduction that comes from the payload
discipline above. Even so: TLS required, credentials from the secret manager, no
public endpoint, and network policy limiting access to the app subnets.

**Queue access is a control-plane credential.** Anyone who can enqueue a job with
a known `stepExecutionId` and a `leaseToken` can attempt a claim. This is why the
claim is a conditional `UPDATE` on `lease_token`
([ADR-007](007-idempotency-strategy.md) §5) and not trust in job data. Job data
is treated as **untrusted input** by the worker and is never used to decide
authorization, only to locate the row whose stored state is then checked.

**No credentials in payloads.** The `credentials` decryption path lives
only in the worker, and secrets are read from the secret manager at execution
time. A `redis-cli` dump yields identifiers and nothing sensitive.

**Least-privilege Redis credentials.** The app needs no `CONFIG`, no `FLUSHALL`,
no `SCRIPT DEBUG`. `EVAL`/`EVALSHA` is required by BullMQ; beyond that, the
credential is scoped to the key prefix this application uses. Redis `ACL` should
allow the app's commands and forbid administrative ones.

**Isolation between environments is a key-prefix matter.** Using distinct
`maxmemory`-isolated deployments per environment is preferred over prefixing,
because a prefix mistake in production then cannot read development data.

**Rate limiting is not delegated to the queue.** A per-organization in-flight limit
is an engine-side check, because the queue cannot count in-flight work per tenant
in a way that survives a Redis flush.

## Revisit Conditions

1. **Requirements for arbitrary user code, human-in-the-loop forms, or long-lived
   durable waits with full history replay.** That is Temporal's home ground, and it
   would be a genuine reversal — see §2.
2. **A customer requires the queue to be region-local or on-premises**, or
   otherwise requires no Redis. Then the PostgreSQL-only claim loop
   ([Alternatives Considered](#1-postgresql-as-the-queue--for-update-skip-locked))
   becomes the answer, accepting hand-rolled stall detection.
3. **Required volume exceeds one Redis primary's practical capacity.** Highly
   unlikely for I/O-bound workflow steps; would prompt sharding by queue.
4. **Two or more rejections of BullMQ's `lockDuration` model become operationally
   significant**, indicating step handlers routinely need longer leases than the
   renew loop supports.
5. **The API gains user-facing queue controls** (pause, inspect, bulk retry). BullMQ
   has partial support; a richer control plane might warrant a different queue or
   a bespoke `step_executions` scheduler. v1 explicitly does not expose these.
6. **A second significant queue workload appears** (for example, bulk replay or
   ML batch inference) whose latency or isolation needs differ enough from step
   execution to justify a separate system. Not a second queue name — a separate
   reason.

**What would *not* justify revisiting:** Redis key growth, a desire to reduce
dependency count in the abstract, or a preference for Kafka-style streaming. Each
trades a solved, disposable dependency for one that is load-bearing.

## References

- BullMQ documentation — <https://docs.bullmq.io/>
- Retrying failing jobs — <https://docs.bullmq.io/guide/retrying-failing-jobs>
- Stalled jobs — <https://docs.bullmq.io/guide/jobs/stalled>
- Job Ids — <https://docs.bullmq.io/guide/jobs/job-ids>
- Job priority — <https://docs.bullmq.io/guide/jobs/prioritized>
- Delayed jobs — <https://docs.bullmq.io/guide/jobs/delayed>
- Workers and concurrency — <https://docs.bullmq.io/guide/workers/concurrency>
- Idempotent jobs — <https://docs.bullmq.io/guide/jobs/idempotent>
- Redis eviction policy — <https://redis.io/docs/latest/develop/reference/eviction/>
- Redis persistence — <https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/>
- Temporal documentation — <https://docs.temporal.io/>
- `pg-boss` — <https://github.com/timgit/pg-boss>
- [reliability.md](../architecture/reliability.md) §3 — delivery failure and recovery
- [007-idempotency-strategy.md](007-idempotency-strategy.md) — claim and dedup layers
