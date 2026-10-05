# ADR-004: Modular Monolith with an Explicit Domain Model

| Field | Value |
|---|---|
| Status | **Accepted** |
| Date | 2026-10-05 |
| Deciders | Engineering |
| Supersedes | — |
| Related | [system-overview.md](../architecture/system-overview.md) §2, [execution-engine.md](../architecture/execution-engine.md) §4, [002-postgresql.md](002-postgresql.md) |

## Context

Automia is a workflow orchestration engine with real domain complexity: a DAG
evaluation loop, a state machine with three interacting lifecycles (execution, step,
attempt), retries, timeouts, cancellation, concurrency limits, and a durable
guarantee that a committed state transition is never lost. The *engine* is
genuinely intricate.

The opposite end of the product is ordinary: a REST API, an admin UI, webhook
delivery, an API-key management screen. The engineering question is not "is this
complex?" — it is — but "where do we put the complexity so that it is isolated,
and what stops it from spreading?"

The context that shapes the answer:

1. **The engine is a distributed-systems problem wearing a single-database
   costume.** At-least-once delivery, idempotency, leases, reconciliation, outbox
   draining. All of it is only correct because the data is in one transactional
   store ([ADR-002](002-postgresql.md)). Splitting the engine across services
   would mean inventing a two-phase commit between the state machine and its
   callers. That is a much harder problem than the one we have.
2. **Domain boundaries are the real boundary.** A "workflows service" and an
   "executions service" in separate databases would need an inter-service
   transaction to advance a step — precisely the thing that must be atomic.
3. **We do not yet know the load shape.** We cannot say which component will need
   independent scaling, so splitting by guesswork is unjustified.
4. **We do need independent *deployment* of the three process types.** The API must
   not run step handlers; a step handler must never be able to serve a request.
5. **We need operational simplicity** to run a supportable product.
6. **We need the option to extract later.** Choosing an irreversible architecture
   when the correct one is uncertain is the mistake to avoid.

## Decision

**One codebase, one database, one deployable, three process entrypoints, and hard
module boundaries enforced at the import level. Modules are the architectural unit;
deployables are the operational unit. The engine is a module, not a service.**

### Deployables

| App | Role | May run engines? | May serve HTTP? | Scaling axis |
|---|---|---|---|---|
| `apps/api` | NestJS/Fastify HTTP, auth, tenant guard | **Never** | Yes | Request rate |
| `apps/worker` | BullMQ handler, step side effects | **Yes** | No | Step concurrency |
| `apps/scheduler` | Outbox publisher, schedule ticker, reconciler, retention sweeper | Partially (publisher only) | Health only | Single, or few with claim-based coordination |

The boundary is enforced structurally: `apps/worker` has no HTTP server, `apps/api`
has no BullMQ consumer. This is the one place where "do not run arbitrary step code
in the request path" ([security.md](../architecture/security.md) §1.4) is enforced by
the process topology rather than by a code review.

### Modules

The **nine domain modules** and the tables each one owns, as specified in
[system-overview.md](../architecture/system-overview.md) §11:

| Module | Owns (tables) | Never does |
|---|---|---|
| `identity` | `users`, `sessions`, `accounts`, `verifications` | Decide permissions |
| `organizations` | `organizations`, `memberships`, `membership_invitations` | Evaluate workflows |
| `workflows` | `workflows`, `workflow_versions` | Execute anything |
| `executions` | `executions`, `step_executions` | Perform external I/O |
| `jobs` | `jobs`, `outbox_events` | Know about workflow semantics |
| `webhooks` | `webhook_endpoints`, `webhook_deliveries` | Own identity |
| `schedules` | `schedules` | Evaluate workflow logic |
| `credentials` | `credentials`, `api_keys` | Reveal stored secrets to a client |
| `audit` | `audit_logs`, `idempotency_keys` | Contain business logic |

Three supporting concerns are **not** modules, because they are not domain
concepts and they have no table ownership of their own. They are libraries with
explicit interfaces, and they are listed here so their status is unambiguous:

| Concern | Realised as | Why it is not a module |
|---|---|---|
| Step-type registry (`http`, `condition`, `delay`, `email`) | An injected map inside `executions`, resolved by node `type` | It is the strategy half of `executions`, not a bounded context. Adding a type must not require a new module. |
| Leases, clock, jitter, `Retry-After` parsing | `reliability` library | Pure functions with an injected clock. No state, no tables, no identity. |
| The DAG decision (which step is next) | A use case in `executions` | It is the core of the state machine. Splitting "decide" from "apply" into two modules was considered and rejected — see [Open Questions](#open-questions) Q6. |
| Drizzle client, repositories, migrations | `persistence` library | Infrastructure adapter, not a domain. |

**Direction of dependency** follows the layered structure in
[system-overview.md](../architecture/system-overview.md) §11:

```text
api layer        controllers, guards, pipes, DTOs
  -> application layer   use cases, transaction boundaries
       -> domain layer   entities, state machines, DAG rules
       -> ports          repository + gateway interfaces
            <- infrastructure   Drizzle, BullMQ, outbound HTTP, SMTP
```

Never the reverse. A domain module must not import the `api` layer. `executions`
must not import a concrete step handler; it asks the registry for a strategy by
node `type` and executes it through a narrow interface. That inversion is what
allows step types to be added without touching the state machine.

**Enforcement, because a boundary that is only documented is a boundary that erodes:**

| Rule | Mechanism | CI failure |
|---|---|---|
| No cross-module internal imports | ESLint `no-restricted-imports` with per-module path allowlists, driven by the table above | Yes |
| Domain modules import nothing from the `api` layer | Same rule | Yes |
| The nine-module table-ownership map matches the Drizzle schema | CI check: every table has exactly one owning module, and only that module's repository file references it | Yes |
| `api` has no BullMQ consumer; `worker` has no HTTP server | Import graph check | Yes |
| No module reaches around a repository into another module's tables | Same CI check as above | Yes |
| Schema migrations reviewed for tenant-scoping | Review checklist | Manual, blocking |

**Extraction criteria — the specific, measurable conditions under which a module
becomes its own service.** These are written down now so the decision is not
re-litigated by whoever first hits friction:

1. **Independent scaling is required and measurable.** One module demonstrates a
   sustained need to scale on a different axis than the rest, by measurement, not
   estimate. (Step execution is the current candidate; it is already a separate
   process, so a *service* boundary would be the second step.)
2. **A failure domain that must be isolated.** A module's outages must not degrade
   the others' availability, and the isolation cannot be achieved by process
   boundaries.
3. **A different data-consistency requirement.** The module needs its own
   transactional scope that cannot coexist in one database.
4. **A different release cadence or technology.** The module needs to ship
   independently, or genuinely requires a different runtime.
5. **A team boundary with independent ownership.** A separate team owning a module
   end-to-end, sustained — not a shared-ownership arrangement that lint rules
   cannot express.

Two of five, with measurement, is the threshold. "The code is getting big" is not
one of them, and neither is "microservices are more scalable in general" — the
scale of a single database is [ADR-002](002-postgresql.md)'s problem, not a module
boundary's.

## Alternatives Considered

### 1. Microservices from the start

Rejected. The decisive argument is the transaction: advancing a workflow updates
`step_executions`, `executions`, the outbox, and the audit log, and that must be
atomic. In one database it is one `BEGIN`/`COMMIT`. Across services it is a saga,
with compensating transactions, a distributed failure taxonomy, and idempotency
required for *every* intermediate step — for a system whose entire purpose is
demonstrating correct idempotency, that irony is a signal we picked the wrong
architecture. Add five independently deployable services, five deployment
pipelines, five sets of dashboards, and no demonstrated load shape to justify any
of it.

### 2. Hexagonal / clean architecture with ports and adapters

Rejected as a *whole* prescription, while adopting its best idea. Pure hexagonal
fans out to many small interfaces and can bury a three-line query behind three
files. What we take from it: **the domain does not import infrastructure**, and
the clock, the ID generator, and the HTTP client are injected so the engine is
testable without a real world. That is the part that pays for itself. The rest is
adopted where it costs nothing.

### 3. DDD aggregates with event sourcing

Adopted in part, rejected in part. The aggregates are real and useful
(`Workflow`, `WorkflowVersion`, `Execution`, `StepExecution`), and they are the
basis for the module boundaries. **Full event sourcing is rejected**: the engine's
hot path is a conditional `UPDATE` and its read path is "what is the current
state, and can I claim it". A projection that can be behind is a poor fit for
"can I claim it". We keep the audit log — which captures the *effect* of every
transition — without making it the source of truth.

### 4. Entity Framework (Microsoft)

Rejected on ecosystem mismatch. It is a strong fit for large .NET systems, and
it is a poor fit for a TypeScript monorepo: wrong language, wrong ORM
assumptions, and it brings a migration story we would not use.

### 5. A deliberately unstructured single application

Rejected. It is the failure mode of choosing "modular monolith" without the
"modular". The complexity in this system is real, and unstructured code would let
the engine's invariants leak into HTTP handlers, where they cannot be reviewed or
tested coherently.

### 6. Serverless functions per step

Rejected. A step is a stateful claim of a row, plus a lease, plus a comparison —
one filesystem per execution, and a cold start inside a step is a latency source
customers feel. Better Auth's AWS-Lambda adapter exists, but that is a deployment
adaptation of the library, not a reason to make functions the unit of execution.

## Why

**1. The engine's correctness depends on transactions, and transactions do not
cross a service boundary.** This is the whole argument. A workflow transition
that can half-apply is a corrupted execution, and a corrupted execution in a
workflow engine is a support incident about somebody's business process being
stuck. Keeping the engine's state changes in one database makes "cannot
half-apply" a property of the architecture rather than of a compensation routine.

**2. Three processes buy nearly all of the microservice benefit that applies here.**
Independence of *scaling* and independence of *failure blast radius* — the two
real reasons to separate — are obtained by `apps/api` / `apps/worker` /
`apps/scheduler`. What we do not get is independent deployment, independent
databases, or cross-service transactions, and we have concluded we do not need
them yet.

**3. Module boundaries preserve the extraction option.** Because modules have no
outward imports and own their tables, a module can later be lifted into a service
with its data, and the boundary becomes a network call. The discipline is
*designed* to be convertible. Note the asymmetry: choosing microservices first and
collapsing them later is much harder, because data ownership has already been
split and cross-service transactions have already been written.

**4. Lint-enforced imports make boundaries real.** A documented boundary decays
within two sprints. A CI failure does not. This is why "modular" here is
specified as four import rules and a table-ownership map, not as a principle.

**5. The cost is low and the cost of the alternative is not.** A monorepo has
shared tooling, shared types, and one migration story. A microservice estate has
N versioned contracts, N deployment pipelines, and distributed tracing as a
requirement rather than a convenience.

**6. The complexity has exactly one home.** The engine — the genuinely difficult
part — is one module with one state machine and one transaction boundary. The
simple parts stay boring. The alternative is a simple part (an admin UI) and a
hard part (the engine) that are equally hard to change because they are equally
distributed.

## Consequences

### Positive

- **Transitions are atomic by construction.** One transaction, one database.
- **Real scaling and blast-radius separation** where it is needed, without a
  service network.
- **One migration stream, one schema, one connection pool.**
- **Boundaries are mechanically enforced**, so they survive team changes.
- **Extraction remains possible** because modules own their tables and import
  inward only.
- **TypeScript types and Drizzle types are shared** across the three apps, so a
  schema change is a compile error everywhere it matters.
- **The engine can be tested in isolation**, with an injected clock and
  repository interface — no Redis, no real time, no HTTP.
- **Local development is one `pnpm dev`**, with a shared database and a local
  Redis.

### Negative

- **One database is a scaling ceiling and a single availability dependency**
  ([ADR-002](002-postgresql.md)).
- **One deploy unit couples releases.** A bug fix to a web-hook endpoint ships the
  engine change too. Mitigated by the three deployables being independently
  deployable, and by expand/contract schema evolution.
- **Module boundaries will be violated under time pressure.** A lint rule catches
  the import, but a *design* violation that compiles (reaching around a repository)
  is caught only by review. Honest cost.
- **Cross-module transactions are permitted**, and therefore will be used. A
  scheduler sweep that writes execution state inside a workflow transaction is
  legal and hard to reason about. Mitigation: the table-ownership map plus the
  review checklist; we accept the residual risk.
- **No independent scaling of a single module** beyond the three-process split. If
  the `steps` handlers become the bottleneck, `worker` scales — but `worker`
  carries every step type.
- **Enforcement is tooling-dependent.** ESLint rules can be disabled. Reviewed
  quarterly.

## Operational Impact

**Deployment:** three images from one repository, one pipeline. `apps/api` and
`apps/worker` scale on demand; `apps/scheduler` runs one replica by default, with
a `--replicas=2` path supported because all scheduler work is claim-based
(`FOR UPDATE SKIP LOCKED`) and therefore safe to run concurrently.

**Database migrations:** one stream, applied before the new code starts, with
expand/contract ([ADR-002](002-postgresql.md)). Because all three apps share the
schema, a migration must be compatible with the *currently running* version of all
three during a rolling deploy — the practical reason the discipline is expand/contract
and not "migrate and restart".

**Local development:** `pnpm dev` starts all three plus Postgres and Redis via
containers. The engine can be exercised without a worker by calling the engine
module directly, which is how the unit and integration suites run
([execution-engine.md](../architecture/execution-engine.md) §11).

**Observability:** one OpenTelemetry service per process
(`automia-api`, `automia-worker`, `automia-scheduler`), so module-level
attribution comes from spans rather than from a service boundary. That is the
practical upside of the monorepo: a cross-module trace is free and correct.

**Configuration:** one config schema shared by all three apps, with per-app
required keys validated at startup. Failing fast on a missing secret is a
correctness requirement, not a nicety
([security.md](../architecture/security.md) §5.1).

**On-call:** one codebase, one language, one set of runbooks. Module ownership is
visible in the trace, so a page about a slow execution identifies the module
without a service-ownership map.

## Security Impact

**The process boundary is a security boundary.** `apps/api` never runs step
handlers, so a step-type bug — a template-injection defect, a malformed
condition — cannot be reached through a public HTTP request. This is the single
most valuable structural consequence of the deployment split, and it is why
`apps/worker` runs with a restricted network policy and no public ingress.

**Tenant scoping has one place to be wrong.** Every module receives
`organizationId` from the tenancy guard, and repositories require it as a
mandatory argument ([security.md](../architecture/security.md) §4.2). A missing
scoping bug is caught in review and by the table-ownership map — CI cannot type-check
that a filter was present, which is an honest limitation.

**Credentials are only in the worker.** The step-type registry inside `executions` — the only code that
decrypts `credentials` — runs exclusively in `apps/worker`, so a
compromise of the API process does not yield step credentials. The `http` step type
applies the SSRF controls ([security.md](../architecture/security.md) §6); those
controls are only effective if that code is not reachable from a request handler,
which the process split guarantees.

**The engine is not reachable over HTTP.** There is no public "run this step"
endpoint, and no endpoint that accepts a caller-supplied workflow graph to execute
immediately without the validation and versioning path. A public workflow API is
a code-execution surface, which is why
[security.md](../architecture/security.md) §1.4 treats "authenticated RCE" as the
dominant threat and why the step-type surface is deliberately small.

**Blast radius of a vulnerability** is a module plus a process, and is bounded by
the process topology rather than by the size of the codebase.

## Open Questions

1. **Will `steps` handlers need to scale independently of each other?** A
   customer whose workflows are all long-poll `wait` steps would want different
   worker capacity from one doing HTTP bursts. Today one `worker` process handles
   all step types. Worker-side admission control handles it; if it turns out to be
   insufficient, the answer is a second queue — which
   [003-bullmq.md](003-bullmq.md) §1 rejected, and which would need revisiting
   against that ADR's revisit conditions. `Proposed`.
2. **Should `scheduler` be split into `dispatcher` and `reconciler` deployments?**
   Their latency characteristics differ: the outbox publisher is latency-sensitive
   (milliseconds after commit), the reconciler and sweeper are not (minutes is
   fine). Two processes with different SLOs is the honest structure. It may not
   matter at our scale, and the claim-based design means one deployment can do
   both. `Proposed`.
3. **Where does the retention/archive job live?** It is a batch workload, not a
   scheduler tick, and a heavy `DELETE` on `step_executions` in the same process
   as the publisher risks interfering with it. Likely a separate long-running
   process or an externally scheduled job. `Proposed`.
4. **Should the scheduler ever run in more than one region?** Today: no. The
   outbox claim and the schedule tick are both claim-based, so multiple replicas in
   one region are safe, but cross-region would need a defined authority for
   `schedules`, which is a bigger question than this ADR. `Proposed`.
5. **Should the three-guard authorisation chain (§4.2 of
   [security.md](../architecture/security.md)) live in the `api` layer or in
   `organizations`?** Today it is `api`-layer guards reading from
   `organizations`. If it stays a thin guard plus a repository, moving the
   permission check into `organizations` would reduce indirection without losing
   safety. Not decided; the table-ownership map is the forcing function.
   `Proposed`.
6. **Should "decide which step runs next" and "apply the decision" be separate
   modules?** They are today one module (`executions`) with two layers, because
   the split is a code-level separation with no independent deployment, scaling or
   failure requirement — so it does not meet any of the five extraction criteria.
   If it is ever needed, the two-process split (`api` never runs engines) is the
   part that must survive regardless. `Proposed`.

## Revisit Conditions

1. **Two or more extraction criteria in [Decision](#extraction-criteria--the-specific-measurable-conditions-under-which-a-module-becomes-its-own-service) are met for the same module, with measurement.** That is the
   trigger, and it is deliberately specific.
2. **A measured scaling limit in `worker` that a second queue would relieve**, after
   worker-side admission control has been tried. Revisit against
   [003-bullmq.md](003-bullmq.md).
3. **Independent release cadence becomes a real product requirement** for one
   module, e.g. step types delivered by a third-party integration team.
4. **Per-tenant data residency** requires data to live in more than one jurisdiction
   with different databases. Then tenant-affinity routing, not extraction, is the
   work, and `organization_id` sharding becomes the design constraint.
5. **An organizational change** creates genuinely independently-owning teams for
   two or more modules (criterion 5).
6. **A cross-service transaction is required for correctness** and cannot be
   avoided by moving the state into one module. If the only way to make something
   atomic is to co-locate it, we co-locate — and if that is impossible, a
   different consistency model is needed.

**What would *not* justify revisiting:** the codebase being large, a desire for
"proper" service separation, an individual contributor's preference, or the
general claim that microservices scale better. The first three are aesthetic; the
last is a claim about a different architecture at a scale we have not measured.

## References

- [002-postgresql.md](002-postgresql.md) — the transaction argument
- [003-bullmq.md](003-bullmq.md) — the worker/dispatcher split
- [system-overview.md](../architecture/system-overview.md) §2 — component boundaries
- [execution-engine.md](../architecture/execution-engine.md) §4 — the `executions` module
- Fowler, "MonolithFirst" — <https://martinfowler.com/bliki/MonolithFirst.html>
- Fowler, "PresentationDomainDataLayering" — <https://martinfowler.com/articles/presentation-domain-data-lifting.html>
- Fowler, "MicroservicePremium" — <https://martinfowler.com/bliki/MicroservicePremium.html>
- AWS, "Monolith to Microservices" (compatibility prerequisites) — <https://docs.aws.amazon.com/modernization/amazon-monolith-to-microservices/>
- Azure Architecture Center, "Use the monolith, not microservices" — <https://learn.microsoft.com/azure/architecture/guide/architecture-styles/modernization>
- Google Cloud, "Define service boundaries" — <https://cloud.google.com/architecture/define-service-boundaries>
- DDD, bounded contexts — <https://martinfowler.com/bliki/BoundedContext.html>
