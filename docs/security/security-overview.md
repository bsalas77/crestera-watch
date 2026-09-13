# Security & Data-Handling Overview

*A summary of how Crestera Watch protects information, written for a buyer's
due-diligence and third-party-risk team. It has a plain-English section
anyone can read and a controls reference for security staff.*

> **Document status.** This is the vendor's own description of the product's
> security design ("self-assessment"). It is accurate to the software as
> built, but it is **not** a third-party audit. Where an independent
> attestation or penetration test is required, see *Independent assurance*
> at the end.

---

## The single most important point: you host it, so we never hold your data

Crestera Watch is **self-hosted software** — a single Go binary plus a set of
native agents. You install it on your own server (Windows or Linux), or run
the Postgres/OpenSearch/VictoriaMetrics standard profile in your own Docker
environment. There is **no vendor-operated service** anywhere in the data
path.

What that means for your risk assessment:

- **Your telemetry never leaves your infrastructure.** The vendor has no
  server in the data path, no copy of your logs or metrics, and no access to
  your running system.
- **There is no telemetry, analytics, or update-check mechanism in the
  product at all** — not an opt-in one, not a disabled-by-default one. This
  has been verified by reading the code and its dependency list: there are
  no analytics libraries, no hardcoded outbound endpoints, and nothing that
  contacts the vendor. The only outbound network calls Crestera Watch ever makes
  are to destinations **you** configure — your identity provider, your email
  server, your Slack/Teams webhook, Microsoft Graph or Google's Admin SDK if
  you enable those connectors, or the optional cloud AI provider — and every
  one of them is off unless you turn it on.
- **The optional AI assistant defaults to fully off** (`CRESTERAWATCH_AI_PROVIDER=disabled`).
  If enabled, you choose between a **local** model (Ollama, on your own
  hardware — nothing leaves the box) and a **cloud** model (Anthropic Claude,
  Pro edition only) — your log and detection data is sent to Anthropic only
  in that second case, and only for the query you actually ask.
- **Many standard SaaS-vendor questions don't apply** — "where is my data
  stored," "who at the vendor can see it" — because the vendor is never in
  possession of your data. Those controls live entirely in your environment.

This is a **shared-responsibility** model:

| The vendor is responsible for… | You (the operator) are responsible for… |
|---|---|
| Security of the application code | Security of the server/network you install it on |
| Fixing reported vulnerabilities | Applying updates, OS patching, network controls |
| Secure defaults and fail-closed startup checks | TLS certificates, strong admin secrets, backups |
| Documentation of secure deployment | Access control, monitoring, and incident response for your environment |

---

## In plain English

- **It refuses to run without encryption.** Crestera Watch terminates TLS itself —
  it will not serve plaintext HTTP on a non-loopback address unless you
  explicitly set `CRESTERAWATCH_ALLOW_PLAINTEXT_HTTP=true` and accept the printed
  warning. A half-configured certificate (one file set, not the other) is
  also refused outright rather than silently falling back to plaintext.
- **Signing in is protected.** Passwords are hashed with PBKDF2-SHA256 at
  600,000 iterations (OWASP's current recommendation), with automatic
  rehashing to a higher iteration count as it rises over time. Repeated
  failed logins lock an account out (8 attempts / 10-minute window / 15-minute
  lock), scoped per tenant so one org's lockout can't be used to deny another
  org's identically-named account. Free edition adds optional **TOTP-based
  two-factor authentication**; Pro edition adds enterprise **SSO** (OIDC,
  SAML 2.0, LDAP/Active Directory), each mapping identity-provider groups to
  Crestera Watch roles.
- **The entire API surface is rate-limited, and requests bodies are size-
  capped.** Login, platform login, and every SSO/SAML entry point are
  throttled per client IP, independent of the per-account lockout above (that
  one protects a password from being brute-forced; this one protects the
  endpoint itself from being flooded regardless of which account, or how
  many different ones, each request names). Agent enrollment, heartbeat, and
  every ingest endpoint (logs, metrics, inventory, posture, software,
  evidence, webhook ingest) are throttled per agent id — or per client IP
  before an agent id is known, at enrollment. Every remaining session-
  authenticated dashboard/admin route is throttled per signed-in identity
  (tenant + username, not client IP — several real users can legitimately
  share one corporate egress address), with a deliberately generous default
  since a single busy dashboard session can legitimately fire a double-digit
  burst of calls loading several panels at once (confirmed live: a normal
  click-through session comfortably stays under a third of the default
  burst). Every request body on the unauthenticated/agent surface is capped
  via `http.MaxBytesReader` so nothing can exhaust memory or disk regardless
  of authentication. Defaults are deliberately set well above real measured
  load (see the sustained-load test referenced in the roadmap), not guessed,
  so they trip on genuine abuse or a malfunctioning agent, not legitimate
  peak traffic — all are operator-tunable via
  `CRESTERAWATCH_*_RATE_LIMIT_*`/`CRESTERAWATCH_MAX_*_BODY_BYTES` environment
  variables. On a multi-node Postgres/HA deployment, every limit is
  distributed rather than enforced independently per node: each node
  periodically reports its own presence and reads back how many nodes are
  currently active, dividing its configured share of each deployment-wide
  budget accordingly — so an identity able to reach every node behind a
  load balancer sees roughly the configured aggregate limit, not that limit
  multiplied by the node count. The sync is periodic (every few seconds),
  not a write on every request, so the request path itself never touches
  the database for this — only the periodic reconciliation does.
- **Multi-tenant deployments are really isolated, not just filtered.** On the
  Postgres profile, every tenant-owned table is protected by PostgreSQL
  **row-level security**, enabled and forced at the database level — not a
  `WHERE tenant_id = ?` clause an application bug could omit. A connection
  that hasn't set its tenant context sees zero rows from a protected table,
  by construction.
- **Every action is logged.** A tamper-evident audit trail records who did
  what, and Crestera Watch is designed to be a source *for* your SIEM/monitoring
  stack as much as a monitor itself.
- **It's kept patched.** Every change runs through automated tests and a
  dependency vulnerability scan (`govulncheck`) in CI before it ships; the
  product currently reports **zero known vulnerabilities** in its production
  dependencies.
- **No lock-in.** Postgres and OpenSearch, where used, are the operator's own
  databases; both a JSON application-data backup and a raw SQLite log
  snapshot can be pulled and restored without the vendor's involvement.

---

## Controls reference (for security teams)

### Identity & access
- **Authentication:** local username/password (PBKDF2-SHA256, 600k
  iterations) on both editions, with a strong admin password auto-generated
  and printed once on first run — the server never ships an open or
  default-credentialed state. Free edition adds TOTP MFA. Pro edition adds
  OIDC, SAML 2.0 (SP-initiated, signature-verified via `crewjam/saml`, replay
  protection via `InResponseTo`), and LDAP/Active Directory bind
  authentication.
- **Authorization:** role-based access control enforced server-side on every
  request. Three fixed roles ship free (admin/analyst/viewer); Pro adds
  custom roles over a ~20-permission taxonomy, with optional per-role field
  redaction (e.g. masking `src_ip`/`user`) applied **before** query matching,
  not just before rendering — so a redacted field can't be searched for its
  real value either.
- **Multi-tenancy:** Postgres row-level security, enabled and forced per
  table, keyed on a `tenant_id` set from the authenticated session/API
  credential — never from a request parameter. Agent enrollment binds an
  agent to its real tenant (`agent_tenant_index`), so ingested data can't
  land in the wrong tenant's database.
- **Per-tenant SSO/connector credentials** (LDAP bind password, OIDC/SAML
  client secrets, Google Workspace service-account key, AWS access keys) are
  encrypted at rest with **AES-256-GCM** under a server-held key, never
  stored or transmitted in plaintext, and redacted on every read path — a
  GET on a saved credential returns a masked value, never the secret itself.
- **SSRF hardening on tenant-supplied URLs:** OIDC issuer discovery, SAML
  metadata fetch, and LDAP bind targets are dialed through a guard
  (`internal/netguard`) that blocks loopback, RFC1918/ULA-private,
  link-local (including the `169.254.169.254` cloud metadata address), and
  multicast ranges — checked against the **actual resolved IP at dial time**,
  so a public hostname that resolves to a private address (DNS rebinding)
  is blocked identically to a literal private IP in the URL.

### Data protection
- **In transit:** TLS terminated by Crestera Watch itself, refused to boot without
  it (see above) unless explicitly overridden for local testing.
- **At rest:** the operator's PostgreSQL/OpenSearch/disk encryption; Crestera Watch
  does not hold a separate copy.
- **Secrets:** agent bearer tokens and session secrets are random values;
  passwords are PBKDF2-hashed, never stored in plaintext; per-tenant
  connector credentials are AES-256-GCM encrypted (see above).
- **Log redaction:** server-side redaction strips common secret patterns
  (tokens, keys) from ingested log text before storage.

### Application security
- **Content-Security-Policy** with a per-request nonce on every dashboard
  response.
- **SQL injection posture:** database access goes through parameterized
  queries throughout; the only string-built SQL statements are internal DDL
  built from a fixed table-name allowlist, never from request input.
- **Continuous dependency scanning:** `govulncheck` runs in CI on every
  change against the actual call graph (not just "a CVE exists somewhere in
  a dependency," but "a vulnerable function is actually reachable from this
  code") — currently zero reachable vulnerabilities.

### Secure development lifecycle
- CI (GitHub Actions) runs build (native + cross-compile), `go vet`, the full
  test suite **under the race detector** with the Postgres tier included, and
  `govulncheck` on every change — a change cannot merge with a failing gate.
- **589 automated test functions** across the codebase, including adversarial
  tenant-isolation tests (two tenants writing identically-named entities,
  concurrent-write races, cross-tenant read/restore attempts) and SSRF
  regression tests that spin up a real local server and assert it is never
  reached through a guarded path.
- **Coordinated vulnerability disclosure** — see [`SECURITY.md`](../../SECURITY.md).
- **A structured internal security review** was performed in August 2026,
  every finding fixed and covered by a regression test that fails without
  the fix — see the [hardening review](hardening-review-2026-08.md). This is
  the vendor's own review, not a third-party assessment.

### Logging, monitoring & resilience
- **Audit logging** of sensitive actions, queryable in-app.
- Crestera Watch is itself designed to be **a destination** for other products'
  security events (e.g. Crestera Control's block/execution decisions), and
  its own detections/alerts can notify to email, Slack, Teams, or an
  arbitrary webhook — so it's a source for your broader monitoring stack as
  much as a monitor of your hosts.
- **Backups:** a documented backup/restore path for both the application-data
  store and the full log history (`VACUUM INTO` snapshots for the embedded
  log engine; scripted Postgres/OpenSearch/VictoriaMetrics backup for the
  standard profile), restorable without vendor involvement.
- **Agent resilience:** the native agent retries failed submissions with
  exponential backoff and spools to a bounded local disk queue on sustained
  failure, so a server restart or network blip does not silently lose
  telemetry.

### Regulatory alignment
Automated, indicative coverage assessments against **9 published frameworks**
out of the box — NIST CSF 2.0, NIST SP 800-53, PCI DSS, FFIEC CAT, NCUA 748,
ISO/IEC 27001:2022 (Annex A), GLBA Safeguards, plus CIS Controls and a general
Essential Security Baseline — available on **both** the Free and Pro edition.
A control genuinely outside a SIEM's observational reach (e.g. physical or
personnel security) reports **N/A with an explanation**, never a guessed
pass or fail.

---

## Known gaps, stated plainly

One thing a security reviewer will likely ask about and not find:

- **No SOC 2 or ISO 27001 vendor certification, and no completed independent
  penetration test** — see *Independent assurance* below.

---

## Independent assurance (roadmap)

The items below require third parties and are **not** claimed as complete.
They are listed so your team knows the current state and the plan:

- **Third-party penetration test** — scoped and ready to book; see the
  [engagement package](pentest-engagement.md). **The test has not been
  performed.**
- **SOC 2 / ISO 27001 certification** — not pursued at this time; available
  for discussion on request.
- **Completed security questionnaire** — a SIG-Lite/CAIQ-aligned
  self-assessment is provided in
  [`vendor-security-questionnaire.md`](vendor-security-questionnaire.md); the
  vendor will also complete your organization's specific questionnaire on
  request.

*Contact: barry.salas@gmail.com*
