# Vendor Security Questionnaire — Self-Assessment

*Structured answers to the questions a third-party-risk team typically asks,
aligned to common frameworks (Shared Assessments **SIG Lite**, Cloud Security
Alliance **CAIQ**). Use this to answer most of a standard questionnaire; the
vendor will also complete your organization's specific form on request.*

> **How to read this.** These are the **vendor's own representations** about
> the software as built — a self-assessment, not an independent audit.
> Because Crestera Watch is **self-hosted**, many controls are **shared**: the
> vendor secures the application code; the operator (you) secures the
> environment it runs in. Answers are marked **[V]** vendor-provided, **[O]**
> operator responsibility, or **[S]** shared. Verify against your own
> deployment before relying on any answer.

---

## A. Governance & program

**A1. Is there a documented information security program?** [V] Secure
defaults, a coordinated vulnerability-disclosure policy (`SECURITY.md`), and
a documented internal security review process (see
[hardening-review-2026-08.md](hardening-review-2026-08.md)).

**A2. Do you undergo independent audits (SOC 2, ISO 27001 certification)?**
[V] Not at this time. A third-party penetration test is scoped and not yet
performed; see `security-overview.md`. Crestera Watch **assesses** an
organization's coverage of NIST CSF 2.0, NIST SP 800-53, PCI DSS, FFIEC CAT,
NCUA 748, ISO/IEC 27001 Annex A, and GLBA Safeguards — that is a product
capability, not a vendor certification.

**A3. Where is customer data hosted?** [O] Entirely within the **customer's**
own environment — the server(s) you install Crestera Watch on, or your own
Postgres/OpenSearch/VictoriaMetrics containers. The vendor operates no
service in the data path and holds no copy of customer telemetry.

**A4. Does the software collect telemetry or analytics?** [V] No. There is no
analytics/tracking code, no update-check mechanism, and no hardcoded outbound
connection anywhere in the product — verified in the code and dependency
list. Every outbound call (an IdP, an email/Slack/Teams webhook, a connector
you enable, an optional cloud AI provider) is to a destination **you**
configure, and every one is off by default.

## B. Access control & authentication

**B1. How do users authenticate?** [V] Local username/password
(PBKDF2-SHA256, 600,000 iterations) on both editions. [S] Pro edition
federates to the operator's own OIDC provider, SAML 2.0 IdP, or LDAP/Active
Directory directory — the operator configures the connection.

**B2. Is MFA supported?** [V] Yes — built-in TOTP two-factor authentication
on the Free edition. [S] Pro-edition SSO defers MFA enforcement to the
operator's own identity provider (Conditional Access, etc.), the same as any
federated login.

**B3. Is access role-based and least-privilege?** [V] Yes — server-side RBAC
on every request, three fixed roles free (admin/analyst/viewer), and
Pro-edition custom roles over a granular permission set with optional
per-field redaction.

**B4. How is user provisioning/deprovisioning handled?** [O] Local accounts
are managed in-app by an admin. [S] SSO/LDAP/SAML users are provisioned
implicitly on first login with a role mapped from their IdP group;
deprovisioning in the operator's directory takes effect on their next
session, since Crestera Watch re-resolves group membership at login rather than
caching it.

**B5. How are application secrets/API keys handled?** [V] Agent bearer
tokens and session-signing secrets are random values, never derived from
guessable input. Per-tenant SSO/connector credentials (client secrets, bind
passwords, service-account keys, AWS access keys) are encrypted at rest with
AES-256-GCM under a server-held key and redacted on every read — a query for
a saved credential returns a masked value, never the plaintext secret.

**B6. Is there account lockout / brute-force protection?** [V] Yes — 8 failed
attempts within a 10-minute window locks the account for 15 minutes, scoped
per tenant so one organization's failed logins cannot be used to lock out a
different tenant's identically-named account. The client IP used for this is
trusted from `X-Forwarded-For` only when the immediate connection comes from
an operator-configured trusted proxy — otherwise it's ignored, so a caller
cannot forge its way around the lockout by setting its own header.

**B7. Is there rate limiting / request-size limiting beyond account
lockout?** [V] Yes, across the whole API surface. Login, platform login, and
every SSO/SAML entry point are throttled per client IP (independent of B6's
per-account lockout); agent enrollment, heartbeat, and every ingest endpoint
are throttled per agent id (per client IP before one is known) and body-
capped via `http.MaxBytesReader`; every remaining session-authenticated
dashboard/admin route is throttled per signed-in identity (tenant +
username). All are operator-tunable via environment variables, with
defaults set well above real measured sustained/interactive load. On a
multi-node Postgres/HA deployment, every limit is distributed rather than
enforced independently per node — nodes periodically (every few seconds,
not per request) reconcile their share of each deployment-wide budget
against how many nodes are currently active.

## C. Data protection & encryption

**C1. Is data encrypted in transit?** [V] Yes — Crestera Watch terminates TLS
itself and **refuses to serve plaintext HTTP** on any non-loopback address
unless the operator explicitly opts in for local testing. [O] The operator
supplies the certificate.

**C2. Is data encrypted at rest?** [O] Via the operator's PostgreSQL/
OpenSearch/disk encryption. Crestera Watch does not hold a separate copy of
customer telemetry to encrypt independently.

**C3. Is customer data segregated in a multi-tenant deployment?** [V] Yes —
PostgreSQL **row-level security**, enabled and forced on every tenant-owned
table, keyed on a tenant id resolved from the authenticated session or agent
credential, never from request input. A connection without a tenant context
set sees zero rows from a protected table by construction, not by
application-layer filtering alone. Agent enrollment is bound to a specific
tenant so ingested telemetry cannot land in the wrong tenant's data. In a
single-organization (non-multi-tenant) install, this question doesn't apply
— there is one tenant.

**C4. Can data be exported / is there portability?** [V] Yes — a JSON
application-data backup, a raw SQLite snapshot of the full log history
(`VACUUM INTO`, safe to run against a live server), and scripted
Postgres/OpenSearch/VictoriaMetrics backup for the standard profile. No
proprietary lock-in format.

**C5. How is data retention/disposal handled?** [S] Free edition: a single
adjustable retention cutoff (default 90 days) past which log data is
deleted. Pro edition adds independently-adjustable hot/warm/cold/glacier
tiers, with cold/glacier data exported to a compressed archive (local disk or
an operator-configured S3-compatible bucket) before removal from the live
store, and permanent deletion once past the glacier boundary.

## D. Application & secure development

**D1. Is there a secure SDLC?** [V] Yes — CI runs build, `go vet`, the full
test suite under the race detector (Postgres tier included), and a
dependency vulnerability scan on every change; a failing gate blocks the
change from merging. 589 automated test functions as of this writing,
version-controlled with full history.

**D2. How is input validated (injection/XSS)?** [V] Database access is
parameterized throughout — no user input is ever concatenated into SQL. A
per-request-nonce Content-Security-Policy is applied to every dashboard
response. The August 2026 internal review found and fixed 3 stored-XSS sinks
in the dashboard's role-rendering code; see the
[hardening review](hardening-review-2026-08.md) for the finding and its fix.

**D3. Are third-party dependencies monitored for vulnerabilities?** [V] Yes —
`govulncheck` (which checks actual reachability, not just "a CVE exists
somewhere in the dependency tree") runs in CI on every change. The August
2026 review found 8 then-reachable CVEs, including a signature-bypass
vulnerability in a SAML dependency sitting directly in the assertion-
validation path; all were fixed the same stretch. **Currently zero known
reachable vulnerabilities.**

## E. Vulnerability & patch management

**E1. Is there a vulnerability-disclosure process?** [V] Yes — coordinated
disclosure via `SECURITY.md`; acknowledgement within 5 business days.

**E2. How are patches delivered?** [S] Security fixes land on `main`; [O]
the operator applies updates (a new package/binary; there is no
auto-update). Operators pin to a release and watch the repository for
security-relevant changes.

**E3. Do you perform penetration testing?** [V] A third-party penetration
test is scoped (see [pentest-engagement.md](pentest-engagement.md)) but has
**not yet been performed** as an independent engagement. An internal
security review was performed in August 2026 by the development team; see
the hardening review.

## F. Logging, monitoring & incident response

**F1. Is there audit logging?** [V] Yes — sensitive actions are recorded in
an audit trail viewable in-app.

**F1a. Are collected logs protected against tampering?** [V] In transit and
at the source, yes, from 2.2: each agent signs, sequences and hash-chains its
log batches, and the server detects missing, replayed, altered and forged
batches, keeps them tagged, and raises a detection. [V] Not yet for data at
rest: the stored copy and the audit trail are not hash-chained, so an
administrator with database access could alter them undetected; signed
checkpoints and a verification command are planned. [O] Restrict database
access, and forward detections off-host. Details: docs/security/security-overview.md
(*Log integrity*).

**F2. Can logs feed our SIEM?** [S] Crestera Watch is itself commonly deployed
*as* the SIEM; it also accepts events forwarded from other tools (e.g.
Crestera Control) via an authenticated ingest endpoint, and can alert out to
email, Slack, Teams, or an arbitrary webhook. [O] The operator wires up
whichever external destinations they need.

**F3. Is there an incident-response capability?** [S] Pro edition includes
multi-step SOAR playbooks (containment actions, evidence collection, case
management) that an operator configures and runs against their own
environment. For vulnerabilities in the product itself, the vendor follows
the disclosure policy above.

## G. Business continuity, backups & resilience

**G1. Are backups provided?** [S] A documented backup/restore mechanism
exists for both the application-data store and the full log history (see
C4). [O] The operator owns backup scheduling, storage, and testing —
Crestera Watch does not currently run backups on a schedule for you; the
mechanisms are there for the operator to invoke.

**G2. Is there BCP/DR guidance?** [V] Documented restore procedures for both
the lite (embedded) and standard (Postgres/OpenSearch/VictoriaMetrics)
deployment profiles; see the README's Backup and Restore section.

**G3. What is the availability model?** [S] Self-hosted; on the standard
(Postgres) profile, Crestera Watch supports running multiple stateless
application nodes behind a load balancer, with automatic leader election
(a Postgres advisory lock — no separate consensus service) for the
background work that must have a single owner. See
[deployment-ha.md](../deployment-ha.md) and
[ADR-001](../architecture/adr-001-ha-clustering.md) for the design and
topology. [O] The operator still owns HA for Postgres/OpenSearch/
VictoriaMetrics themselves — the application tier being safe to run more
than once doesn't make the data tier redundant on its own — and for the
lite/single-file profile, which is single-instance by design and has no
multi-node story.

## H. Third-party / supply chain

**H1. What sub-processors receive customer data?** [V] **None** by default.
Operator-enabled connectors (Microsoft Graph, Google Admin SDK, AWS
CloudTrail, an SSO/LDAP directory, a cloud AI provider) send data only where
the operator explicitly configures and only for the data type each connector
polls or the query the operator submits.

**H2. Is a Bill of Materials available?** [V] Yes — dependencies are
declared in `go.mod`/`go.sum`; a fixed, exact-version dependency tree with no
floating ranges.

## I. Deployment & shared responsibility

**I1. What are the deployment options?** [O] Self-hosted: a native Windows
Service or Linux systemd install with a bundled JSON-file/embedded-SQLite
store for single-organization use, or a Docker Compose "standard profile"
(Postgres + OpenSearch + VictoriaMetrics) for larger or multi-tenant
deployments.

**I2. Who is responsible for what?** [S] See the shared-responsibility table
in `security-overview.md`: vendor = application-code security and secure
defaults; operator = hosting, network, OS patching, TLS certificates,
secrets, backups, and access management.

---

*Prepared as a self-assessment for prospective customers' third-party-risk
review. Contact: barry.salas@gmail.com*
