# System Architecture Diagrams

| Field | Value |
|---|---|
| Status | **Proposed** |
| Date | 2026-10-05 |
| Companion docs | [system-overview.md](../architecture/system-overview.md), [execution-flow.md](./execution-flow.md), [failure-recovery.md](./failure-recovery.md), [database-erd.md](./database-erd.md) |
| Decisions | [ADR-002](../adr/002-postgresql.md), [ADR-003](../adr/003-bullmq.md), [ADR-004](../adr/004-modular-monolith.md) |

This document holds the diagrams that describe the system as a whole. Execution
mechanics live in [execution-flow.md](./execution-flow.md), failure and recovery
in [failure-recovery.md](./failure-recovery.md), and the physical schema in
[database-erd.md](./database-erd.md).

**The one boundary to read every diagram against:** PostgreSQL holds
authoritative state; Redis holds delivery state and is fully reconstructible.
Most of the apparent redundancy below exists to serve that line.

---

## 1. System context

```mermaid
flowchart TB
    subgraph callers["callers"]
        DASH["Dashboard<br/>(browser, cookie session)"]
        CLI["API clients<br/>(API key: ak_live_…_…)"]
        PROV["Webhook providers<br/>(signed inbound POST)"]
        SCHED_SRC["Scheduler clock<br/>(our own, not external)"]
    end

    subgraph automia["Automia"]
        API["apps/api<br/>NestJS + Fastify"]
        SCHED["apps/scheduler<br/>publisher, ticker,<br/>reconciler, sweeper"]
        WORKER["apps/worker<br/>BullMQ consumer + step executors"]
    end

    subgraph ext["external systems the platform depends on"]
        IDP["Google / GitHub<br/>(OAuth 2.0 / OIDC)"]
        EMAIL["Email provider"]
    end

    subgraph customer["customer systems — called BY steps, never BY the engine"]
        HTTP1["Customer API"]
        HTTP2["Customer API"]
    end

    DASH -->|"HTTPS"| API
    CLI -->|"HTTPS"| API
    PROV -->|"HTTPS + HMAC-SHA256"| API
    API -->|"browser redirect"| IDP
    WORKER -->|"SMTP / provider API"| EMAIL
    WORKER -->|"outbound HTTP, SSRF-checked"| HTTP1
    WORKER -->|"outbound HTTP, SSRF-checked"| HTTP2
    SCHED_SRC --> SCHED

    style automia fill:#e8f0fe,stroke:#4285f4
    style customer fill:#fef7e0,stroke:#f9ab00
    style ext fill:#f1f3f4,stroke:#5f6368
```

Two boundaries in this diagram are load-bearing:

1. **The engine never calls a customer system.** `apps/api` and `apps/scheduler`
   have no outbound path to `customer`. Only `apps/worker` does, which is why the
   SSRF controls ([security.md](../architecture/security.md) §6) are effective
   rather than advisory: a public request can never reach step execution.
2. **The scheduler is internal.** A schedule is a business fact in PostgreSQL, not
   an external cron service. That is what makes "when does this run next?"
   answerable and what makes catch-up-after-downtime possible.

---

## 2. Component diagram

```mermaid
flowchart TB
    subgraph api["apps/api — one codebase, HTTP only"]
        direction TB
        AUTHM["identity<br/>Better Auth mount, Principal"]
        TENM["organizations<br/>members, roles, API keys"]
        WFM["workflows<br/>authoring, validation"]
        EXCM["executions<br/>start, inspect, cancel, retry"]
        GUARD["guards<br/>Session → Organization → Permission"]
        CTRLS["controllers + DTOs<br/>RFC 9457 error mapping"]
    end

    subgraph sch["apps/scheduler — loops, no HTTP"]
        direction TB
        PUB["outbox publisher"]
        TICK["schedule ticker"]
        REC["reconciler"]
        SWEEP["retention sweeper"]
    end

    subgraph wk["apps/worker — steps, no HTTP"]
        direction TB
        CONS["BullMQ consumer"]
        REG["step-type registry<br/>http · condition · delay · email"]
        EGRESS["egress policy<br/>DNS pinning, blocklists"]
    end

    PG[("PostgreSQL<br/>system of record")]
    REDIS[("Redis / BullMQ<br/>steps queue — reconstructible")]
    S3[("Object storage<br/>payloads — a cache of jsonb")]

    CTRLS --> GUARD
    GUARD --> AUTHM
    GUARD --> TENM
    CTRLS --> WFM
    CTRLS --> EXCM
    EXCM -->|"transaction"| PG
    WFM --> PG
    TENM --> PG
    AUTHM --> PG

    PUB --> PG
    TICK --> PG
    REC --> PG
    SWEEP --> PG
    PUB -->|"queue.add — the only producer"| REDIS
    REC -->|"re-dispatch"| REDIS

    REDIS --> CONS
    CONS --> REG
    REG --> EGRESS
    EGRESS -->|"conditional UPDATE claim"| PG
    CONS --> PG

    style PG fill:#e6f4ea,stroke:#34a853
    style REDIS fill:#fce8e6,stroke:#ea4335
```

### Why there are three processes and not three services

| | Separate processes | Separate services |
|---|---|---|
| Independent scaling of step execution | **Yes** — scale `worker` | Yes |
| Failure blast radius | Yes — `api` cannot run steps | Yes |
| Atomic workflow transition | **Yes** — one transaction, one database | No — needs a saga |
| Deployables to operate | 3 | 3 + N contracts, N pipelines |
| Distributed tracing | Free | Required |

The deciding factor is the third row. Advancement updates `step_executions`,
`executions`, `outbox_events` and `audit_logs` atomically, and a cross-service
transaction would replace that with a distributed failure taxonomy in the one place
where correctness matters most. [ADR-004](../adr/004-modular-monolith.md) has the
full reasoning and the five extraction criteria.

**The colours are the operational rule.** Green is the system of record: if it is
lost, it is catastrophic. Red is reconstructible: `FLUSHDB` is a latency event,
repaired by the reconciler. An operator who internalises those two colours
internalises the whole reliability model.

---

## 3. Deployment view

```mermaid
flowchart TB
    subgraph edge["edge"]
        LB["TLS termination<br/>+ WAF"]
    end

    subgraph region["one region — single-region is the stated availability boundary"]
        subgraph stateless["stateless tier — horizontal"]
            API1["api-1"]
            API2["api-2"]
        end

        subgraph compute["compute tier — horizontal"]
            W1["worker-1"]
            W2["worker-2"]
            W3["worker-3 …"]
        end

        SCHED1["scheduler-1<br/>all four loops,<br/>claim-based"]
    end

    subgraph data["stateful tier — NOT horizontally scalable"]
        PGPRIMARY[("PostgreSQL primary<br/>multi-AZ, synchronous_commit=on")]
        PGREPL[("read replica")]
        REDISN[("Redis<br/>noeviction")]
        BUCKET[("Object storage<br/>private bucket")]
    end

    LB --> API1
    LB --> API2
    API1 --> PGPRIMARY
    API2 --> PGPRIMARY
    API1 --> PGREPL
    API2 --> PGREPL
    W1 --> PGPRIMARY
    W2 --> PGPRIMARY
    W3 --> PGPRIMARY
    W1 --> REDISN
    W2 --> REDISN
    W3 --> REDISN
    SCHED1 --> PGPRIMARY
    SCHED1 --> REDISN
    PGPRIMARY --> PGREPL
    API1 --> BUCKET
    W1 --> BUCKET
    SCHED1 --> BUCKET

    style data fill:#fce8e6,stroke:#ea4335
    style stateless fill:#e8f0fe,stroke:#4285f4
    style compute fill:#e8f0fe,stroke:#4285f4
```

| Tier | Scaling | State | Notes |
|---|---|---|---|
| `api` | Horizontal on request rate | None | Stateless. Sessions are in PostgreSQL. |
| `worker` | Horizontal on step concurrency | None | In-memory lease renewal only; a lost node loses nothing. |
| `scheduler` | **1 by default**, safe at N | None | Every loop claims with `FOR UPDATE SKIP LOCKED`, so N replicas partition work with no leader election. |
| PostgreSQL | Vertical, then replicas | All of it | The single hard availability dependency. |
| Redis | Vertical | Delivery only | `noeviction` so a job is never silently dropped. |

**Why the scheduler is 1 by default and still safe at N.** Each of its four loops
is a claim query. Two publishers never claim the same outbox row; two tickers never
claim the same schedule occurrence. That property comes from
`FOR UPDATE SKIP LOCKED` plus the unique `executions.dedup_key`, not from a
coordination service — which is why there is no lease table to run.

**Why `api` readiness does not check Redis.** A Redis outage must not take the API
out of rotation. The API accepts executions and commits them; the outbox drains
when Redis returns. Failing readiness on Redis would turn a delay-only incident
into a total outage.

---

## 4. Module dependency direction

```mermaid
flowchart TB
    subgraph layers["allowed direction — enforced by ESLint no-restricted-imports"]
        direction TB
        L1["api layer<br/>controllers, guards, pipes, DTOs"]
        L2["application layer<br/>use cases, transaction boundaries"]
        L3["domain layer<br/>entities, state machines, DAG rules"]
        L4["ports<br/>repository + gateway interfaces"]
        L5["infrastructure<br/>Drizzle, BullMQ, outbound HTTP, SMTP"]
        L1 --> L2 --> L3
        L2 --> L4
        L5 -.->|implements| L4
    end

    style L3 fill:#e6f4ea,stroke:#34a853
```

The `domain` layer has no database client, no HTTP client, and no logger. That is
what makes the execution state machines testable as pure functions with an
injected clock, and what would make extracting a module into a service a packaging
change rather than a rewrite.

**Nine domain modules, and the tables each owns.** Table ownership is a CI-checked
map: exactly one module may reference each table.

| Module | Owns |
|---|---|
| `identity` | `users`, `sessions`, `accounts`, `verifications` |
| `organizations` | `organizations`, `memberships`, `membership_invitations` |
| `workflows` | `workflows`, `workflow_versions` |
| `executions` | `executions`, `step_executions` |
| `jobs` | `jobs`, `outbox_events` |
| `webhooks` | `webhook_endpoints`, `webhook_deliveries` |
| `schedules` | `schedules` |
| `credentials` | `credentials`, `api_keys` |
| `audit` | `audit_logs`, `idempotency_keys` |

Three concerns are **libraries, not modules**, because they have no table ownership
and no independent lifecycle: the step-type registry (inside `executions`), the
reliability helpers (clock, jitter, `Retry-After`, leases), and the persistence
adapter.

---

## 5. Trust boundaries

```mermaid
flowchart LR
    subgraph untrusted["untrusted"]
        U1["Tenant-authored workflow JSON"]
        U2["Inbound webhook body"]
        U3["API request"]
    end

    subgraph verified["verified at the edge"]
        V1["Signature + timestamp<br/>before any DB write"]
        V2["Schema validation<br/>per node type"]
        V3["Session / API key → Principal"]
        V4["SSRF policy at request time<br/>DNS resolve + pin, revalidate hops"]
    end

    subgraph trusted["trusted — server-controlled only"]
        T1["Domain state machine"]
        T2["PostgreSQL constraints"]
        T3["Egress from apps/worker only"]
    end

    U2 --> V1
    U1 --> V2
    U3 --> V3
    T1 --> V4
    V1 --> T1
    V2 --> T1
    V3 --> T1
    V1 -.->|"no payload stored on failure"| T2

    style untrusted fill:#fce8e6,stroke:#ea4335
    style trusted fill:#e6f4ea,stroke:#34a853
    style verified fill:#fef7e0,stroke:#f9ab00
```

Ordering is the point, not just membership:

- **A workflow definition is validated at write time *and* at run time.** Static
  checks (scheme, literal IP, no userinfo) happen on publish; the DNS-resolution
  and hop-revalidation checks can only happen per attempt, because DNS answers
  change. A URL that passed validation an hour ago is re-checked now.
- **A webhook signature is verified before any database write**, so an unsigned
  flood costs one HMAC computation. A rejected body is not stored.
- **`V3` produces a `Principal` and nothing downstream re-derives identity.** The
  domain cannot tell a cookie from an API key, which is what keeps the two
  authorisation paths from diverging.

---

## 6. Data flow — durable writes

Every durable state change follows one path. There is no second code path, and
this is the invariant most worth protecting in review.

```mermaid
flowchart TB
    UC["use case"] --> BEGIN["BEGIN"]
    BEGIN --> DOMAIN["domain state UPDATE / INSERT<br/>always conditional, always org-scoped"]
    DOMAIN --> AUDIT["audit_logs INSERT<br/>security-relevant actions only"]
    AUDIT --> OUTBOX["outbox_events INSERT<br/>if async work is implied"]
    OUTBOX --> COMMIT["COMMIT"]
    COMMIT --> VISIBLE["now visible to every reader"]
    VISIBLE -.->|"later, another process"| PUB["outbox publisher"]

    style COMMIT fill:#e6f4ea,stroke:#34a853
```

The rule that makes the whole system work: **a transaction that changes domain
state and implies asynchronous work must also insert the `outbox_events` row, in
the same transaction.** Forget the outbox row and the intent is lost with no error
anywhere; the reconciler eventually notices a `READY` step that was never
dispatched, but the work was already late.

`audit_logs` is in the same transaction for the same reason: an action that
succeeds without its audit record is a gap nobody can detect afterwards.

---

## 7. Observability topology

```mermaid
flowchart LR
    subgraph procs["instrumented processes"]
        A["apps/api"]
        W["apps/worker"]
        S["apps/scheduler"]
    end

    A -->|"OTLP"| COL["OTel collector"]
    W -->|"OTLP"| COL
    S -->|"OTLP"| COL
    A -->|"structured JSON"| LOGS["Log pipeline<br/>redacted at source"]
    W -->|"structured JSON"| LOGS
    S -->|"structured JSON"| LOGS

    COL --> TRACES["Trace backend"]
    COL --> METRICS["Metrics"]
    LOGS --> ALERTS["Alerting"]
    METRICS --> ALERTS
    TRACES --> ALERTS

    ERR["RFC 9457 response<br/>+ traceId"] -.->|"user quotes it<br/>in a support request"| LOGS

    style ERR fill:#fef7e0,stroke:#f9ab00
```

Every error response carries a `traceId`, and that trace id is present in the logs
and traces. This is the supported support path, and it exists specifically so that
internals never have to be returned to a caller.

Three metrics are correctness signals rather than performance signals, and they
are the ones worth alerting on:

| Metric | Alert condition | Why it is not a performance concern |
|---|---|---|
| `outbox_events` oldest `PENDING` age | Growing | Every second of this is work that is due and not running. |
| `step_executions` with an expired lease | Any, beyond a grace period | A stalled worker, and a step that may be executing twice. |
| `outbox_events` in `DEAD` | **Must be zero** | Permanently unperformed work. Not an error-rate question. |

---

## 8. What is deliberately absent

Naming absent components is as informative as naming present ones, because each
one is a mechanism someone will reasonably expect and that we have decided against.

| Not present | Why not | Reference |
|---|---|---|
| Message broker (Kafka, SNS) | One consumer per job with per-job retry is a queue's job, not a log's. The `outbox_events` table *is* the event log. | [ADR-006](../adr/006-transactional-outbox.md) §5 |
| BullMQ Flows | Encodes orchestration in Redis, making a non-durable store authoritative. | [ADR-003](../adr/003-bullmq.md) §1 |
| BullMQ Job Schedulers | A Redis-resident cron registry. `schedules` lives in PostgreSQL and firing is deduped by a unique `dedup_key`. | [system-overview.md](../architecture/system-overview.md) §15 |
| BullMQ rate limiters | Per-organization in-flight limits are engine-side, because the queue cannot count per tenant in a way that survives a flush. | [security.md](../architecture/security.md) §3.4 |
| Separate scheduling service | `scheduled_for` lives on an `executions` row, so "when does this run?" has one answer. | [ADR-003](../adr/003-bullmq.md) §2 |
| Leader election for the scheduler | Every loop claims with `SKIP LOCKED`. | [ADR-004](../adr/004-modular-monolith.md) §3 |
| Distributed cache of domain state | A read-through cache in front of `executions` would create a second source of truth. Definitions are cached; mutable state is not. | [ADR-002](../adr/002-postgresql.md) §1 |
| Scripting in workflows | Would make a tenant who can publish a workflow a code-execution principal. The registry is four types. | [ADR-005](../adr/005-workflow-versioning.md) §Decision |
| Event sourcing | The hot path is a conditional `UPDATE` and the read path is "can I claim this". A projection that can be behind is the wrong shape. | [ADR-004](../adr/004-modular-monolith.md) §3 |

---

## Related

- [execution-flow.md](./execution-flow.md) — the same components in motion
- [failure-recovery.md](./failure-recovery.md) — what each arrow does when it fails
- [database-erd.md](./database-erd.md) — the twenty tables these components own
