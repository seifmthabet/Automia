# ADR-001: Authentication Strategy

| Field | Value |
|---|---|
| Status | **Accepted** |
| Date | 2026-10-05 |
| Deciders | Engineering |
| Supersedes | — |
| Related | [security.md](../architecture/security.md) §2, [system-overview.md](../architecture/system-overview.md) §19 |

## Context

Automia is a multi-tenant SaaS backend. It needs to answer two independent
questions, and conflating them is the single most common cause of serious
authentication bugs:

```text
Authentication  ->  Who is calling?          (an identity, a credential, a session)
Authorization   ->  May they do this, here?  (a tenant, a role, a permission)
```

The concrete requirements:

1. **Google and GitHub OAuth** for customer sign-in. Both are OAuth 2.0; Google
   also implements OpenID Connect. GitHub does not always assert a verified
   primary email, which materially affects account-linking safety.
2. **Email and password** as a first-class method, not an afterthought, because
   enterprise buyers ask for it and because password-based accounts are the
   vector for credential stuffing.
3. **Server-side sessions** that can be revoked. A platform holding customer
   third-party credentials needs "sign out everywhere now" to be a real option.
4. **Programmatic access** via API keys, scoped to an organization and a subset
   of its permissions.
5. **PostgreSQL** as the system of record, with **Drizzle ORM** as the data layer
   ([ADR-002](002-postgresql.md)). The auth solution must share the database and
   the ORM rather than introducing a second datastore.
6. **NestJS + Fastify** as the HTTP stack. NestJS has **no built-in
   authentication**; this is by design, and the choice of an auth solution is
   ours.
7. **MFA and passkeys** as a plausible future requirement that should not require
   a rewrite.
8. **We must not reimplement cryptographic authentication protocols.** This is a
   production-oriented product. Reimplementing session management, PKCE, or
   password hashing to "learn" them is a security liability, not an educational
   exercise.

What we are explicitly **not** deciding here: the roles, permissions and
tenant-isolation model. That is Automia's own authorization design, and it is
described in [security.md](../architecture/security.md) §4. This ADR covers
identity only, plus the API-key surface, which we own because it is part of our
authorization model.

## Decision

**Use [Better Auth](https://www.better-auth.com/docs) as the authentication
engine, integrated into NestJS/Fastify through a thin adapter module that we own.
Better Auth owns cryptographic authentication; Automia owns authorization.**

Concretely:

| Concern | Owner |
|---|---|
| Password hashing and verification | Better Auth |
| OAuth 2.0 / OIDC flows, `state`, PKCE | Better Auth |
| Session creation, validation, expiry, revocation | Better Auth |
| Account linking policy | Better Auth's `accountLinking` configuration, plus Automia policy in the UI and audit trail |
| 2FA (TOTP), passkeys, backup codes | Better Auth plugins, opt-in |
| `users`, `sessions`, `accounts`, `verifications` tables | Better Auth's schema, in our PostgreSQL, via its Drizzle adapter |
| Organizations, memberships, roles, permissions | **Automia** |
| API keys | **Automia** |
| Step credentials | **Automia** ([security.md](../architecture/security.md) §5) |

### Integration shape

Better Auth is framework-agnostic. It is mounted as a request handler; because
our HTTP adapter is Fastify, we register a route group that converts the Node
request to a Fetch `Request` (`fromNodeHeaders`) and the response back. This
mount lives in our own NestJS module rather than in a third-party wrapper, for
the reasons in §4 below.

```text
apps/api/src/identity/
  better-auth.module.ts     NestJS module: config, DI, lifecycle
  better-auth.controller.ts catch-all handler for /v1/auth/*
  session.strategy.ts       maps a Better Auth session -> our principal
  principals.ts             Principal types (UserPrincipal | ApiKeyPrincipal)
```

The engine and the domain have **no dependency on Better Auth**. They receive a
`Principal` value object. This is what keeps [ADR-004](004-modular-monolith.md)'s
module boundaries intact and what would make a future identity-provider swap a
change to one module.

### Sessions

Server-side sessions in PostgreSQL, presented as an opaque 256-bit token in a
`__Host-sid` cookie with `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`. The
`__Host-` prefix is chosen deliberately: browsers enforce `Secure`, forbid
`Domain`, and require `Path=/` for it, which is a free floor against subdomain
cookie injection. No token ever lives in `localStorage`, so an XSS bug is not an
immediate full account takeover. See [security.md](../architecture/security.md)
§2.3.

### API keys

Owned by Automia, not by Better Auth's API-key plugin. Rationale in §4.

### Password storage

Delegated to Better Auth, configured at current OWASP guidance — Argon2id with a
minimum of `m = 19456` (19 MiB), `t = 2`, `p = 1`, or scrypt with
`N = 2^17, r = 8, p = 1` where Argon2id is unavailable. bcrypt and PBKDF2 are
legacy or FIPS-only paths. A pepper is held in configuration, never in the
database. We do not choose these parameters ourselves and we do not implement the
hash.

## Alternatives Considered

### 1. `@nestjs/authentication` (official NestJS package)

A first-party NestJS package with official documentation, a global guard with a
`@Public()` opt-out, current-user decorators, scrypt at OWASP parameters
(`N = 2^17, r = 8, p = 1`), `__Host-sid` cookies with only a SHA-256 of the token
stored, Drizzle persistence, TOTP, magic links, and an API-key format
(`cat_<id>_<secret>`, constant-time SHA-256 verification). It is architecturally
the most natural fit for a NestJS application, and its session and cookie design
is better thought through than most.

**Rejected because:**

- **It is at version `0.0.1`.** A pre-1.0 package in the authentication path of a
  production SaaS product is a stability risk we cannot quantify, and the API may
  change without a deprecation cycle. This alone is not fatal — many good
  libraries start at `0.x` — but it is a real cost.
- **Narrower feature set for our needs.** No organization/multi-tenancy story, no
  first-class API-key scoping to organization permissions, no passkeys. We would
  build those ourselves anyway.
- **We would be building the OAuth surface ourselves** on top of it, which is
  exactly the protocol reimplementation this project has ruled out.

**Not rejected on merit.** It is a serious candidate, and it is the most likely
successor if Better Auth's NestJS path becomes uncomfortable. See §9.

### 2. Auth.js (formerly NextAuth)

Mature, widely deployed, and now maintained under the Better Auth organisation.
Its own maintainers state that they recommend new projects start with Better Auth
unless there is a specific feature gap — most notably stateless session
management without a database.

**Rejected because:**

- **Framework-shaped around Next.js.** Official integrations exist for Next.js,
  Qwik, SvelteKit and Express. There is no first-party NestJS or Fastify
  integration, and the project is explicitly not a Next.js application.
- **Its maintainers point new projects elsewhere**, so the ecosystem investment
  (plugins, examples, integrations) is migrating to Better Auth.
- **Stateless sessions are not a requirement for us.** We require revocation, which
  means a server-side session store. That is the one gap Auth.js names, and it is
  the gap we do not want.

### 3. A managed OIDC / identity provider

Delegate authentication entirely: Google, GitHub, Okta, Auth0, WorkOS, and
similar. The application becomes an OIDC relying party.

**Rejected as the primary mechanism because:**

- **Authorization stays ours regardless**, so the main architectural benefit —
  removing auth code — is only partly realised.
- **It couples the tenant model to the provider's.** Enterprise plans typically
  enforce one directory per organisation, which forces our `organizations` and
  `memberships` to mirror the provider's structure. For a product where
  membership and role are core domain concepts, that is an unacceptable inversion
  of control.
- **Vendor cost and availability become platform availability.**

**Not rejected as an *additional* mechanism.** Enterprise SSO (SAML or OIDC per
tenant) is a genuine future requirement and is the natural use of this option. It
is marked `Proposed` in [security.md](../architecture/security.md) §10.5 and is
deliberately not designed now, because per-tenant keys, claim mapping and
just-in-time provisioning are product decisions that should be driven by a real
requirement.

### 4. A hand-rolled implementation in NestJS

Guards, a `users` table, `scrypt` from `node:crypto`, session rows, and hand-built
OAuth redirects.

**Rejected, and this is the important one.** It would mean implementing OAuth
state handling, PKCE, redirect validation, session fixation protection, password
hashing and comparison, and account-linking safety — all of which are
security-critical protocol and cryptographic concerns. Doing that in a production
SaaS backend to gain familiarity is a bad trade. Familiarity with these protocols
is gained by reading the RFCs and by operating the chosen library, not by owning a
custom implementation that a future engineer has to keep safe.

### 5. Build vs. buy, summarised

| Criterion | Better Auth | `@nestjs/authentication` | Auth.js | Managed IdP | Hand-rolled |
|---|---|---|---|---|---|
| Security maturity | High, actively developed | Early (`0.0.1`) | High | High | **Our liability** |
| Google / GitHub OAuth | Built in | Would be ours to build | Built in | Delegated | Ours to build |
| Server-side revocable sessions | Yes | Yes | Yes (its known gap is *stateless*) | Yes | Ours to build |
| Password auth | Yes | Yes | Yes | Delegated | Ours to build |
| MFA / passkeys | 2FA + passkey plugins | TOTP only | Via provider | Delegated | No |
| Account linking | Configurable, verified-email rules | — | Provider-dependent | Delegated | Ours to get right |
| API keys scoped to an organization | Possible, but our shape | Basic | No | No | Ours |
| PostgreSQL + Drizzle | Drizzle adapter | Drizzle adapter | Drizzle adapter | External DB | Ours |
| NestJS integration | Community wrapper; Fastify path is beta | **First-party, documented** | None | N/A (framework-agnostic) | N/A |
| Maintenance outlook | Active, MIT | Very new, first-party | Stable but superseded by Better Auth | Vendor-dependent | Ours forever |
| Operational complexity | One more service to run and patch | Fewer moving parts | Fewer moving parts | **External dependency + cost** | Low now, high later |
| Learning value | Learn to integrate and to reason about the boundary | Learn to integrate | Learn to integrate | Learn OIDC | Learn the hard way, in production |

## Why

Three reasons, in order of weight.

**1. The boundary is the point, not the library.** The failure mode we are
avoiding is a system where "who is this?" and "may they do this?" are answered by
the same mechanism. Better Auth gives us a genuinely narrow, well-defined surface
— session validation and OAuth protocol handling — and leaves the tenant model,
roles and permissions to us. That separation is enforced in code: the engine
receives a `Principal` and has no idea whether it came from a cookie or an API
key. This is what makes [ADR-004](004-modular-monolith.md)'s module boundaries
real rather than aspirational.

**2. Feature completeness without protocol work.** Google, GitHub, password auth,
server-side sessions, TOTP and passkeys all arrive from a maintained project
instead of from our own commits. Every one of those is a place where a subtle
mistake is a vulnerability. The only thing we build is the integration layer,
which is the part where subtlety is a maintainability problem rather than a
security problem.

**3. We own the integration, on purpose.** Better Auth has no first-party NestJS
integration. The documented community wrapper
(`@thallesp/nestjs-better-auth`) exists, but putting an unreleased, community-
maintained wrapper in front of the authentication path of a production product is
a worse trade than writing ~150 lines of request/response plumbing that we own,
can read, and can test. The plumbing is: convert a Fastify request to a Fetch
`Request`, call Better Auth's handler, convert the response back. There is no
security logic in it. We also get better NestJS ergonomics (`@Public()`-style
decorators, DI-provided `AuthService`) than a generic wrapper would give us.

**On the "community wrapper is beta" point:** the wrapper's Fastify support being
labelled beta is one of the reasons we own the integration. It is not a reason to
avoid Better Auth itself, whose Fastify guide is first-party.

## Consequences

### Positive

- No cryptographic authentication protocol is implemented or maintained by us.
- Google, GitHub, password auth, revocable server-side sessions, TOTP and
  passkeys are all available without building them.
- Sessions are revocable, which is a requirement for a platform holding customer
  credentials.
- The authorization model — organizations, memberships, roles, permissions,
  API-key scoping — stays entirely ours, and is not coupled to an identity
  provider's data model.
- The domain and engine depend on a `Principal` interface, not on Better Auth.
  Swapping the identity engine, or adding a second one, touches one module.
- MIT licensed and self-hosted: no per-request cost, no vendor availability risk,
  no data leaving our infrastructure for authentication.
- Drizzle adapter means the auth tables live in the same database and the same
  migration stream as everything else, so tenant filtering and backup policies
  are uniform.

### Negative

- **We maintain an integration layer** that a first-party solution would provide.
  It is small and mostly mechanical, but it is code we own.
- **The mount is a catch-all route**, which interacts awkwardly with Fastify's
  router and requires the standard body-parser opt-out on that path. This is a
  known cost of the framework-agnostic design, not a surprise.
- **Better Auth's schema is not ours to design.** We accept its `users`,
  `sessions`, `accounts` and `verifications` tables, and its column names, even
  where we would have chosen differently. We can map names, but the shape is
  fixed by the library.
- **A dependency on a fast-moving project.** Better Auth is `1.7.x` and
  pre-2.0. Its APIs can change. We mitigate with a thin adapter (a change is
  localised) and by pinning exact versions in the lockfile.
- **Account linking is the sharpest edge.** GitHub does not reliably assert a
  verified email, so verified-email-only linking must be configured, and the
  configuration must be tested rather than assumed
  ([security.md](../architecture/security.md) §2.6).
- **We deliberately do not use two of its plugins.** The organization and API-key
  plugins would work, but our authorization model needs to be ours; see §4.
- **No built-in enterprise SSO.** Deferred, not solved.

### Deliberate non-use of Better Auth's organization and API-key plugins

Both plugins are credible, and both were rejected on the same grounds: our
`organizations` and `memberships` are core domain concepts with our own
invitation flow, our own role model, and our own permission table. Adopting a
plugin's tables would couple the tenant model to a library schema, and it would
mean the ERD in [database-erd.md](../diagrams/database-erd.md) is not the
product's actual data model. The API-key plugin can own organization-scoped
keys, but our keys must be uniformly verifiable by the same guard that handles
session principals, with our own permission subsetting and our own
`last_used_at` semantics.

The cost of owning them: two small CRUD surfaces we must build and maintain. We
accept that, because authorization is the part of this system we most want to own.

## Operational Impact

**Additional service to run:** none. Better Auth is a library inside the existing
`api` process. There is no new container, no new port, no new datastore.

**Deployment:** none beyond the normal application deploy. A Better Auth upgrade is
an ordinary dependency bump, and because the integration is thin, its blast
radius is one module.

**Migrations:** Better Auth's schema enters our migration stream. Two options, and
a decision is required:

| Option | Mechanism | Risk |
|---|---|---|
| Generate Better Auth's tables with its CLI | `npx @better-auth/cli generate` produces SQL we adopt | The library's schema and our Drizzle schema can drift; we own the reconciliation |
| Declare the tables in our Drizzle schema | Hand-written, reviewed, migrated like everything else | Must be kept in sync with library expectations on every upgrade |

**Recommended:** declare the tables in our Drizzle schema and add an integration
test that asserts our declared columns satisfy the library's requirements. The
test is the synchronisation mechanism; without it, drift is discovered in
production.

**Operational tasks added:**

| Task | Frequency | Notes |
|---|---|---|
| OAuth client secret rotation at the provider | On rotation schedule | Dual-secret window supported |
| Session table cleanup of expired rows | Periodic, in the retention sweeper | Expired sessions are useless but not free |
| Review of `auth.sign_in_failed` audit volume | Ongoing | Credential stuffing signal |
| Verify account-linking configuration after any Better Auth upgrade | Per upgrade | The sharpest edge (§4) |

**Observability:** `auth.sign_in`, `auth.sign_in_failed`,
`auth.session_revoked`, `auth.account_linked` and `auth.account_unlinked` are
audited transactionally ([security.md](../architecture/security.md) §8.3). Metrics
cover sign-in success/failure rate, active sessions, and API-key
`last_used_at` distribution. Trace spans cover the auth handler, not the internal
OAuth exchange, so provider latency is not visible in our traces — a known gap,
since the provider call is opaque to us.

**Failure modes:**

| Failure | Behaviour | Blast radius |
|---|---|---|
| Better Auth throws on a request | `500` with a `traceId`; no session created | One request |
| Session table unavailable | Sign-in and session validation both fail | **All authenticated traffic.** Same blast radius as PostgreSQL, because the sessions are in PostgreSQL. |
| Identity provider unavailable | Sign-in via that provider fails; existing sessions continue to work | New sign-ins only — sessions are local, which is a benefit of self-hosting |
| Better Auth upgrade breaks the adapter | Caught by integration tests in CI, not in production | — |

The second row deserves emphasis: because sessions live in our PostgreSQL, an
identity-provider outage does not log anyone out. That is a direct operational
benefit of the self-hosted choice, and it is a stronger argument for it than
feature completeness.

**Runbook: "everyone is logged out."** Check the `sessions` table and PostgreSQL
health before suspecting the auth code. If sessions are present, the problem is
cookie configuration (domain, `Secure`, proxy headers) or the session secret, not
the data.

## Security Impact

**What we gain by delegating:**

- Password hashing at current OWASP parameters, maintained by the library.
- OAuth `state` and PKCE handling maintained by the library, rather than by us.
- Session fixation, rotation and expiry handled by a maintained implementation.
- A CVE response path that is somebody else's full-time job.

**What we must own and can therefore get wrong:**

| Risk | Control | Where |
|---|---|---|
| Session cookie misconfiguration (missing `Secure`, wrong domain) | `__Host-sid` prefix makes `Secure` and `Path=/` browser-enforced; `HttpOnly` set explicitly | [security.md](../architecture/security.md) §2.3 |
| CSRF against cookie-authenticated mutations | `SameSite=Lax` **plus** mandatory `Origin`/`Referer` validation on unsafe methods | §3.5 |
| Account takeover via account linking | Verified-email-only linking; sign-in required to link; confirmation step; refuse to unlink the last method; audit every change | §2.6 |
| API key leakage | Prefix + Argon2id hash at rest, constant-time comparison, shown once, per-key rate limit, `last_used_at` | §2.5 |
| API key used to escalate beyond its scope | Keys carry an explicit permission subset; a key can never change the membership that authorises it | §4.5 |
| Session not revoked on privilege change | Revoke all sessions of a user whose role, password or linked-account set changes | §2.3 |
| XSS escalating to account takeover | No tokens in `localStorage`; CSP; redaction of cookie headers from logs | §5.5 |
| Better Auth misconfiguration in *our* config | Config is code-reviewed and asserted by integration tests; config drift is caught per upgrade | §Operational Impact |
| Multi-tenant privilege confusion | Better Auth holds no tenant data. `organizations` and `memberships` are ours and are filtered by our own guards | §4 |

**Explicit non-goals:** we do not implement MFA enforcement logic; if enabled, the
library does it. We do not implement OAuth provider configuration in a database;
client secrets come from configuration so a database compromise does not yield
the ability to impersonate our OAuth application to Google.

**Threats from the ADR table in [security.md](../architecture/security.md) §1.3
that this decision addresses:** T1 (credential theft), T2 (CSRF), T4 (privilege
escalation), T7/T8 (out of scope here), T17 (API key brute force), T18 (mass
assignment — the request DTOs are ours, `.strict()`), T20 (unauthenticated
enumeration), T22 (idempotency replay — API keys are bound to a request hash).

## Revisit Conditions

Revisit this decision if **any** of the following becomes true:

1. **`@nestjs/authentication` reaches `1.0`** *and* documents the features we
   currently lack (organization model, organization-scoped API keys, passkeys).
   At that point first-party, stable, NestJS-native is a strong argument, and
   Better Auth's lack of a first-party NestJS integration stops being a
   differentiator. This is the most likely reason to switch.
2. **Better Auth's API breaks in a way the adapter cannot absorb** — specifically a
   change to the session contract or the Drizzle adapter that requires
   application-wide edits. One breaking change is a cost; a pattern of them is a
   signal.
3. **Better Auth's licensing or governance changes** in a way that conflicts with
   how we intend to use it. It is MIT today; a change would need reassessment.
4. **Enterprise SSO becomes a committed requirement** (per-tenant OIDC or SAML
   with claim mapping). At that point a managed provider may become the right
   answer for *enterprise tenants only*, as an additional mechanism, while
   self-hosted auth remains the default. That would be a new ADR, not a reversal
   of this one.
5. **A passkey-first or passwordless-first product requirement** arrives that the
   library's passkey support does not meet.
6. **We need stateless sessions** at the edge for latency or geographic reasons.
   That is Auth.js's one documented advantage over Better Auth, and it would
   require a real answer to "how do we revoke?", not just a library swap.
7. **An independent security review** recommends a different approach. The threat
   model in [security.md](../architecture/security.md) §1 is an engineering
   artifact; a qualified external review outranks it.

**What would *not* justify revisiting:** a preference for fewer dependencies in the
abstract, discomfort with a pre-1.0 dependency, or a preference for
hand-rolled authentication. Each of those trades a real security property for an
aesthetic one.

## References

- Better Auth documentation — <https://www.better-auth.com/docs>
- Better Auth Drizzle adapter — <https://www.better-auth.com/docs/adapters/drizzle>
- Better Auth session management —
  <https://www.better-auth.com/docs/concepts/session-management>
- Better Auth users and accounts / account linking —
  <https://www.better-auth.com/docs/concepts/users-accounts>
- Better Auth 2FA plugin — <https://www.better-auth.com/docs/plugins/2fa>
- Better Auth API key plugin — <https://www.better-auth.com/docs/plugins/api-key>
- Better Auth Fastify integration — <https://www.better-auth.com/docs/integrations/fastify>
- Better Auth NestJS integration guide (community wrapper) —
  <https://www.better-auth.com/docs/integrations/nestjs>
- Auth.js, "getting started" (documented framework support) —
  <https://authjs.dev/getting-started>
- Auth.js joins Better Auth (maintenance and recommendation for new projects) —
  <https://better-auth.com/blog/authjs-joins-better-auth>
- NestJS authentication documentation —
  <https://docs.nestjs.com/security/authentication>
- NestJS guards — <https://docs.nestjs.com/security/guards>
- OpenID Connect Core 1.0 — <https://openid.net/specs/openid-connect-core-1_0.html>
- OAuth 2.0 (RFC 6749) — <https://www.rfc-editor.org/rfc/rfc6749>
- PKCE (RFC 7636) — <https://www.rfc-editor.org/rfc/rfc7636>
- OAuth 2.0 Security Best Current Practice (RFC 9700) —
  <https://www.rfc-editor.org/rfc/rfc9700>
- RFC 6750 Bearer Token Usage — <https://www.rfc-editor.org/rfc/rfc6750>
- OWASP Authentication Cheat Sheet —
  <https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html>
- OWASP Session Management Cheat Sheet —
  <https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html>
- OWASP Password Storage Cheat Sheet —
  <https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html>
