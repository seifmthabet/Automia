# ADR-005: Immutable Workflow Versions

| Field | Value |
|---|---|
| Status | **Accepted** |
| Date | 2026-10-05 |
| Deciders | Engineering |
| Supersedes | — |
| Related | [system-overview.md](../architecture/system-overview.md) §4, [execution-engine.md](../architecture/execution-engine.md) §2, [004-modular-monolith.md](004-modular-monolith.md) |

## Context

A workflow is a versioned artefact that people run repeatedly. The moment that
matters is when an execution is *already running* and the definition is *edited*.
The product must answer a question that is easy to state and easy to get wrong:

> This execution is three steps into a ten-step workflow. The definition changed.
> What does the fourth step do?

Every workflow system has an answer here, and the answers differ on three axes:

1. **What is pinned** — the running execution, or the definition at lookup time?
2. **What is mutable** — the definition, or a new version of it?
3. **What does an in-flight execution do about the change** — nothing, restart,
   resume, or migrate?

Requirements:

1. **An in-flight execution must be deterministic and reproducible.** If the same
   execution is inspected, exported, or replayed in six months, it must produce
   the same graph. An execution whose meaning drifts with the definition is not
   reproducible, and neither is its audit trail.
2. **Definitions must be versioned even when the user does not think of them as
   versioned.** A customer edits a workflow, does not press "publish", and expects
   it to work.
3. **Editors must be able to compare and revert.** "What changed between Tuesday's
   run and today's run?" is a question the product must support.
4. **Drafts must not run.** A half-finished edit must not be executable.
5. **Executions must be attributable.** Every execution row points at exactly one
   definition, and that definition is readable forever.
6. **Concurrent edits must not corrupt.** Two people editing the same workflow, or
   one person editing while a run is in flight, must not produce a definition that
   no version ever described.
7. **No code in the definition.** A definition must not be able to execute
   arbitrary logic ([security.md](../architecture/security.md) §1.4). This is why
   the format is a constrained declarative JSON document rather than a program.
8. **Validation must be total.** Any version that can be referenced by an
   execution must have passed validation, including graph-level checks the
   editor cannot do (cycles, unreachable nodes, unknown step types).

## Decision

**Workflow definitions are immutable, versioned, declarative JSON documents.
`workflows` is the mutable identity; `workflow_versions` is append-only. Every
execution references exactly one `workflow_version_id`, and that version's
`definition` is the graph the execution evaluates. Editing a workflow always
creates a new version. A running execution is never affected by an edit.**

### Data model

```text
workflows
  id                 uuid primary key
  organization_id    uuid not null
  slug               text not null          -- stable, unique per organization
  name               text not null
  current_version_id uuid                   -- the version new runs use
  created_at / updated_at

workflow_versions
  id                 uuid primary key
  workflow_id        uuid not null -> workflows(id) on delete cascade
  organization_id    uuid not null
  version            integer not null       -- 1, 2, 3, ... per workflow
  status             enum('draft','published','archived') not null
  definition         jsonb not null
  definition_hash    text not null          -- sha256 of canonical form
  created_by         uuid not null          -- users.id
  created_at         timestamptz not null

  UNIQUE (workflow_id, version)
  UNIQUE (organization_id, workflow_id, version)
```

`executions.workflow_version_id` is a `NOT NULL` foreign key to
`workflow_versions.id`, with `ON DELETE RESTRICT`. An execution's definition
therefore cannot be deleted out from under it, ever, by any code path. This single
constraint is what makes requirement 1 and 5 structural rather than procedural.

### The three states of a version

| Status | Editable? | Runnable? | Meaning |
|---|---|---|---|
| `draft` | **Yes, in place** | No | A work in progress. Mutable. Never referenced by an execution. |
| `published` | No | Yes | Immutable. Referenced by executions. |
| `archived` | No | No | Immutable. Kept for historical reproducibility. Not selected for new runs. |

"Editing a published workflow" therefore means: create a new `draft` from it,
edit the draft, publish the draft, and repoint `workflows.current_version_id`.
The published version is untouched, so a running execution continues to resolve
against it.

The exception, and it is a deliberate convenience, is **auto-publish on a single
edit**: when the author holds the only lock and there is no draft, the API creates
the new version directly as `published`. Most edits do not need a draft/publish
cycle, and forcing one on every save would make the product annoying. When a
draft already exists, subsequent saves update that draft instead of creating more
versions — otherwise a user who reopens a workflow would find forty versions from
forty saves.

**Deduplication:** a `PUT` whose `definition_hash` equals the current draft's or
current published version's is a no-op that returns the existing version. Editing
a workflow and putting it back exactly as it was should not create version 41.

**Versioning is per workflow, not per organization.** `version` restarts at 1 for
each new workflow, which is what a human expects to read in a UI.

### The definition document

```jsonc
{
  "schemaVersion": 1,          // the *format* version, not the workflow version
  "errorPolicy": "fail-fast",  // "fail-fast" (default) | "continue"
  "timeoutMs": 3600000,        // whole-execution budget
  "nodes": [
    {
      "id": "charge-payment",     // stable, unique within the document
      "type": "http",             // must exist in the step-type registry
      "config": {
        "method": "POST",
        "url": "https://api.example.internal/charges",
        "headers": { "Content-Type": "application/json" },
        "body": { "orderId": "{{ trigger.input.orderId }}" },
        "timeoutMs": 10000,
        "retry": { "maxAttempts": 5, "baseDelayMs": 1000, "maxDelayMs": 60000,
                   "retryOn": [408, 429, 500, 502, 503, 504] }
      },
      "timeoutMs": 15000
    },
    { "id": "hold-for-review", "type": "delay", "config": { "durationMs": 86400000 } }
  ],
  "edges": [
    { "source": "validate-order",  "target": "charge-payment",  "condition": "success" },
    { "source": "validate-order",  "target": "hold-for-review", "condition": "failure" },
    { "source": "charge-payment",   "target": "notify-customer", "condition": "always" }
  ]
}
```

Note the shape: **edges are explicit and carry a condition**, not a `dependsOn`
array plus an `onError`. A node's successors are determined entirely by the edges
whose `condition` its terminal state satisfies, which is what makes the pruning and
join rules in
[execution-engine.md](../architecture/execution-engine.md) §10 expressible
without a `condition` node type. `condition` is one of four edge conditions in v1 —
`success` (default), `failure`, `always` — with `expression` marked `Proposed` and
rejected by the validator.

The `executions` module owns the step-type registry; the `workflows` module owns
the envelope, the node envelope, the edge envelope, and the graph rules. Retry
policy lives inside `config`, per step type, because only the step type knows what
a retryable failure is (a status code, an SMTP error code, a timeout). Validation
is layered:

| Layer | Checks | When |
|---|---|---|
| Envelope | `schemaVersion` known, `nodes` non-empty, unique node ids, `errorPolicy` and `timeoutMs` in bounds | On write |
| Node | `type` is one of the four in the registry, `config` matches that type's schema, `timeoutMs` in bounds, `config.retry` well-formed | On write |
| Edge | `source` and `target` both resolve to a node, `condition` is a known condition (`expression` rejected) | On write |
| Graph | Acyclic, no unreachable nodes, at least one entry node, no node whose inbound edges are all `failure` (unreachable by construction) | On write |
| `http` static | `https` only, or `http` to a tenant-allowlisted host; literal IPs must be public; no userinfo in the URL | On write |
| Runtime | Trigger input is JSON; `{{ ... }}` references resolve against stored step outputs, never against a runtime | On trigger |

All four write-time layers run on **every** draft save, not on publish. A draft
that cannot be made valid is not worth saving, and validating early means the
error is next to the edit that caused it rather than at trigger time.

The engine treats the document as **read-only**. It never mutates the definition
it is evaluating; all state lives in `executions` and `step_executions`
([ADR-002](002-postgresql.md)). This is what allows a graph to be cached in
process memory across many executions.

### Why the definition is not code

`schemaVersion`, a step-type registry, and per-type config schemas. There is no
scripting language, no `eval`, no JavaScript in a step body, and no expression
language evaluated at runtime beyond the deliberately tiny, total
`condition` step ([execution-engine.md](../architecture/execution-engine.md) §3.6).

The reason is a product decision, not a technical preference. Automia's security
model is built on the claim that a workflow is **data**; a tenant cannot escalate
to the platform by publishing a workflow. The moment a definition can execute
code, a tenant who can publish a workflow has code execution against the worker,
its network, and every credential the worker can decrypt — which is the
authenticated-RCE threat in [security.md](../architecture/security.md) §1.4 and the
most severe threat in the model. Making that threat structurally impossible is
worth more than the flexibility of a scripting step, and the flexibility is
recoverable later behind a sandbox that is a separate ADR.

## Alternatives Considered

### 1. Mutable definitions with a version snapshot taken at run start

Rejected as equivalent-but-worse. The snapshot has to be taken *somewhere*, and
"somewhere" becomes the problem: if it is a column on `executions`, the semantics
are identical to a version table but the schema cannot answer "which version was
this?" for an execution created three days later, because the answer is only in
rows created in the same instant. The version table makes the reference explicit
and queryable, and it gives comparison, revert, and audit for free.

### 2. Pin the execution to the *latest* version, evaluating the newest graph mid-run

Rejected. This is the "live definition" model, and it is the source of a
characteristic and very confusing class of bug: an execution whose third step runs
from graph version 7 and whose fourth step runs from version 8, where the node was
renamed. It makes in-flight executions non-reproducible, makes the audit trail
untrustworthy, and makes "why did this step run with these inputs?" unanswerable.
Pinning is a product requirement, not an implementation preference.

### 3. Versioned but with a *graph diff* applied to a running execution

The most capable model: publish a new version and let running executions migrate
onto it, preserving completed steps by node id.

Rejected on the honest reason that it is a large amount of machinery — a diff
algorithm, a migration state machine, rules for what happens when a completed
step no longer exists, and a guarantee that in-flight semantics do not change
underneath a step that is already running. All of that is designed against
requirement 1. It is the right answer if "edit a workflow without restarting its
runs" becomes a committed product requirement, and it is marked in
[Revisit Conditions](#revisit-conditions).

### 4. Draft and published as separate tables

Rejected. Same information, two places to keep consistent, and `ON DELETE
behaviour` gets subtle. A `status` column with a partial unique index expresses
"at most one draft per workflow" in the schema:

```sql
CREATE UNIQUE INDEX one_draft_per_workflow
  ON workflow_versions (workflow_id) WHERE status = 'draft';
```

### 5. Storing versions as a JSON array on `workflows`

Rejected. It makes "fetch version 7" a full-row rewrite, makes concurrent publishes
a lost-update problem, and makes `ON DELETE RESTRICT` on the execution's
definition impossible — the reference would be an array index, which cannot be
protected by a foreign key. Requirement 5 would become procedural instead of
structural.

### 6. Git-backed definitions with a separate versioning service

Rejected for v1. It gives excellent diff and revert semantics, and it introduces a
second system of record, an authorisation model over repositories, and a
synchronisation problem between Git and the `workflow_versions` that the executions
reference. Revisit if a compliance requirement demands signed, externally
auditable definitions.

## Why

**1. Reproducibility is a hard requirement of a workflow engine.** When a customer
asks "why did execution X do that?" in six months, the answer must be derivable
from a row. `executions.workflow_version_id` → `workflow_versions.definition`
gives it, with no history table and no "what did the definition look like then?"
question. Every alternative without a pin fails this.

**2. `ON DELETE RESTRICT` turns a correctness requirement into a database
constraint.** An execution physically cannot reference a deleted definition. No
code review can forget to check, because there is nothing to forget.

**3. Immutability of published versions removes a whole category of bug.** If a
published version cannot change, then any code holding a version id is correct for
the life of the process. Graph compilation can be cached. A running execution
cannot be invalidated. There is no read path that must consider "did the definition
change under me?".

**4. The draft/publish split is what makes concurrent editing workable.** Two
authors cannot corrupt a published version, because neither can write to one. The
"one draft per workflow" partial index means a second editor gets a clear
conflict rather than silently overwriting. Optimistic concurrency on
`definition_hash` handles the same-author, two-tab case.

**5. Comparability is a product feature, and it is nearly free.** `version` is a
dense integer and `definition_hash` identifies identical content, so "what
changed between version 4 and 5" is a JSON diff between two rows, and "has
anything changed since I looked" is a hash comparison.

**6. Keeping definitions as data removes the worst threat in the model.** A
workflow is the largest untrusted input the system accepts, and making it pure data
means the ceiling on what a tenant can do is the four step types. This is the
decision that makes [security.md](../architecture/security.md) §1.4's dominant
threat tractable.

## Consequences

### Positive

- **In-flight executions are deterministic and reproducible.** Pinned at trigger.
- **The audit trail is trustworthy**, because the definition a step ran under is
  permanently resolvable.
- **Diff and revert are trivial** — two rows and a JSON diff.
- **Drafts cannot run**, enforced by the status check on the execution path.
- **Published versions are immutable**, so caches cannot be stale and running
  executions cannot be invalidated.
- **Attribution is exact.** `executions.created_by` plus
  `workflow_versions.created_by` answers "who changed what, and who ran it".
- **Deduplication keeps version numbers meaningful** rather than turning
  `versions` into a save log.
- **Definition-as-data keeps the threat ceiling low**: four step types, no eval.
- **Schema-level protection against deleting a referenced version.**

### Negative

- **Fixing a bug in a workflow requires a new version**, and the bad version stays
  in history. Correct, and it means a "hotfix" is a publish operation.
- **Draft/publish is a concept users must learn.** Mitigated by auto-publish for
  the single-editor, no-draft case, so most users never see it.
- **A validation failure blocks a save**, which can feel obstructive when someone
  is mid-edit. Mitigated by validating on every draft save so errors are immediate
  and local, and by the editor running the same validator client-side.
- **Old versions accumulate indefinitely.** They must be retained for the
  reproducibility guarantee, so `workflow_versions` is *not* subject to the
  execution retention sweeper. A deliberate storage cost.
- **No live migration of running executions** (Alternative 3). Changing a workflow
  does not affect runs already in flight; they finish on the old graph.
- **`definition` is JSONB, so the database cannot enforce node-level structure.**
  The unique indexes and the `ON DELETE RESTRICT` on `executions` are
  envelope-level; graph correctness is guaranteed by write-time validation, so a
  bypassed validator could insert an invalid graph. Mitigated by making the
  validator the only write path, by the CI check that `workflows` is the sole
  writer of `workflow_versions`, and by a post-publish re-validation assertion in
  the CI integration suite.
- **Comparison is a JSON diff, not a semantic diff.** A reorder of nodes produces a
  large textual diff even when behaviour is identical. The `definition_hash` is
  computed over a canonical form that orders node arrays by id, which reduces this
  to cosmetic.
- **No workflow inheritance or composition.** A shared fragment must be copied.
  Deliberate; it would compromise reproducibility.

## Operational Impact

**Writes:** cheap. A version insert plus a `current_version_id` update, in one
transaction. `definition_hash` is computed in the application over a canonicalised
serialisation (sorted object keys, nodes sorted by `id`) so that semantically
identical documents hash identically.

**Reads:** hot, and cacheable. Version definitions are read on every trigger and
every step decision. A per-process LRU cache keyed by `workflow_version_id` is
valid indefinitely, because published versions are immutable — an advantage that
falls straight out of this ADR. Cache invalidation is therefore a non-problem for
published versions and a `workflow_id`-keyed invalidation for drafts.

**Migrations:** `workflow_versions` and `executions.workflow_version_id` ship
together in the initial schema. The `ON DELETE RESTRICT` constraint is added in the
same migration that adds the column, so there is never a window where the
reference is unenforced.

**Backups:** `workflow_versions` is included in the standard PostgreSQL backup and
retention policy. It is **explicitly excluded from the execution retention
sweeper**, which is a decision to be enforced in code — the sweeper's table
allowlist contains `step_executions` and `executions`, not `workflow_versions`. An
accidental inclusion would break reproducibility for historical executions, and
that is a review item on the sweeper.

**Monitoring:**

| Signal | Alert condition | Meaning |
|---|---|---|
| `PENDING` executions with a null `workflow_version_id` | Any | Must be impossible; a not-null violation in practice |
| Trigger-time validation failures | Sudden rise | A client is publishing invalid graphs, or the validator changed |
| Definition cache hit rate | Falling | A workflow is being edited in a loop, or the cache size is too small |
| Version count per workflow | Growth far above expectation | Auto-publish is not deduplicating — a bug in the hash comparison |

**Runbook: "an execution ran with a definition I did not expect."** Read
`executions.workflow_version_id` and diff that `workflow_versions.definition`
against `workflows.current_version_id`. If they differ, the execution predates the
publish — which is the documented, intended behaviour, not a bug. If they are the
same, the suspicion is on the node id in an edge, and the
`step_executions` rows for that execution show which graph was actually
evaluated.

## Security Impact

**A workflow is untrusted, tenant-supplied input.** Every field is attacker-chosen,
including URLs, headers, expressions, and node ids. It is validated on write, and
the runtime re-checks the parts that time-dependent validation cannot settle.

**Immutability is a security property.** An attacker who compromises an API session
cannot modify a published definition, so they cannot alter the behaviour of
in-flight or historical executions. Nor can they inject an edge into
a version an execution has already resolved.

**Version pinning resists a specific attack: definition drift.** If a workflow
could be edited mid-run, an attacker with temporary write access could change what
a step does after it has been reviewed. Pinning makes the reviewed artefact the
executed artefact.

**The step-type registry is the security boundary.** A `type` outside the registry
is rejected at write time, so no step can be introduced that the `executions`
module does not implement and audit. The registry is **four types** — `http`,
`condition`, `delay`, `email` — and `webhook`, `database`, `transform`, `parallel`,
`join` and `subworkflow` are `Proposed` and rejected by the validator. A
registry of four is what makes the SSRF controls in the `http` type
([security.md](../architecture/security.md) §6) comprehensive rather than
partial: the entire set of things a tenant can make a worker do is enumerable, and
therefore reviewable.

**Tenant isolation of versions.** `workflow_versions` carries `organization_id`,
the foreign key to `workflows` is composite and tenant-carrying, and reads are
scoped by the tenancy guard. A version from another organization is not reachable
by slug, by id, or by a trigger reference.

**The `definition_hash` is integrity, not authentication.** It detects accidental
divergence and powers deduplication. It is not a signature, and a tenant
publishing their own workflow is expected to control its content. If definitions
must be provably unaltered for compliance, that is a different mechanism
(Alternative 6) and a separate ADR.

**Secrets do not belong in a definition.** A definition is readable by anyone with
read access to the workflow and is retained indefinitely. Step credentials are
referenced by id and resolved from `credentials` at execution time
([security.md](../architecture/security.md) §5.2), never inlined. This is partly a
usability rule and mostly a security rule: a definition is the longest-lived,
most widely-shared, most exportable object in the system.

## Revisit Conditions

1. **"Edit a workflow without restarting its in-flight executions" becomes a
   committed product requirement.** That is the graph-diff migration model
   (Alternative 3), and it is a substantial piece of work with a new state machine.
2. **A requirement for reusable sub-workflows or shared fragments.** Requires either
   composition (which must define its own reproducibility semantics) or a documented
   copy-only policy. Today: copy-only.
3. **Enterprise compliance requires signed, externally auditable definitions**, or
   definitions in Git (Alternative 6).
4. **`workflow_versions` storage becomes material** at a scale where old versions
   must be archived rather than retained. Only safe once an execution's definition
   is guaranteed to be retrievable from the archive, so this is a storage project,
   not a retention-policy change.
5. **The document format needs a breaking change** beyond `schemaVersion`. The
   field exists for this; the work is a versioned reader that can interpret old
   documents, since old definitions must remain executable.
6. **Users need expression power beyond the `condition` step.** Options are a
   strictly more capable but *total and sandboxed* evaluator, or a
   customer-supplied WASM step behind a separate ADR. Never a general `eval`.
7. **A measured need for draft branching** (multiple parallel drafts merged into
   one version). Today: one draft per workflow, enforced by a partial index.

**What would *not* justify revisiting:** a preference for fewer tables, a desire
for the definitions to be editable in place, or an assumption that "most workflow
tools let you change a running workflow". The last is a real customer expectation
and it is a legitimate *product* input — which is why it is item 1 — but the
answer to it, if it arrives, is a diff engine, not mutable definitions.

## References

- [execution-engine.md](../architecture/execution-engine.md) §2 — the execution/definition relationship
- [security.md](../architecture/security.md) §1.4, §5.2, §6 — definitions as untrusted input
- [database-erd.md](../diagrams/database-erd.md) — `workflows`, `workflow_versions`, `executions`
- [system-overview.md](../architecture/system-overview.md) §4 — workflow model
- JSON Schema — <https://json-schema.org/>
- GitOps branching models for reference (not adopted) — <https://gitops.community/patterns/>
- Argo Workflows, workflow templates and versioned artefacts (comparison) — <https://argo-workflows.readthedocs.io/en/latest/workflow-templates/>
- Temporal, workflow versioning (comparison; code-first) — <https://docs.temporal.io/workflows#workflow-versioning>
