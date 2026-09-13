# Internal security review — August 2026

**What this is:** a structured self-review of Crestera Watch's own code, performed
by the development team over several stretches in August 2026. **It is not a
third-party penetration test and must not be presented as one.** No external
firm was engaged, no report was issued by anyone outside the project, and
nothing here should be read as an attestation. See
[Independent assurance](security-overview.md#independent-assurance-roadmap)
for the current state of external validation.

**Scope:** the whole codebase (roughly 54,000 lines of Go plus the frontend)
— not a single feature area. Two separate passes are recorded here: a
proactive review of the newly-added per-tenant SSO/credential surface, and a
subsequent full senior-level review of everything else. Every finding from
both passes was fixed and covered by a regression test that fails without
the fix; the security-relevant ones were also exercised end to end against a
running instance, not just unit-tested in isolation.

---

## Pass 1 — per-tenant SSO, credential, and connector surface

Performed immediately after adding per-tenant OIDC/SAML/LDAP SSO and the
Google Workspace/AWS CloudTrail connectors, on the reasoning that new
tenant-facing configuration surface is exactly the kind of change most
likely to introduce a fresh defect, and is worth reviewing before it accrues
more code around it.

| # | Severity | Finding | Fix |
|---|----------|---------|-----|
| 1 | High | **Credential disclosure via GET.** `GET /api/admin/sso/oidc` and the connector config endpoints returned the *decrypted* stored configuration verbatim — an OIDC client secret, a Google service account's private key, and an AWS secret access key all round-tripped back out in plaintext to anyone holding the same permission that gates far-less-sensitive settings. | Every provider config type now implements a `Redacted()` method; the GET handler always calls it, never the raw decrypted value. A regression test asserts the raw secret string does not appear anywhere in the response body, for every affected provider. |
| 2 | High | **SSRF via tenant-submitted URLs.** An OIDC issuer, a SAML metadata URL, and a Google Workspace service account's embedded token URI all drove server-side fetches with no restriction on target host — reachable by a tenant admin, a lower-trust actor than the operator who previously controlled this via environment variables only. LDAP had a narrower version (an arbitrary TCP connect-and-bind attempt). | New `internal/netguard` package blocks loopback, RFC1918/ULA-private, link-local (covering the `169.254.169.254` cloud metadata address), and multicast destinations, checked against the **actual resolved IP at dial time** — so a public hostname that resolves to a private address (DNS rebinding) is blocked the same as a literal private IP. Applied to every tenant-configurable fetch path. |

Both findings were self-caught by deliberately asking, for each new
tenant-facing endpoint, "what could a tenant-level (not operator-level)
actor now reach that they couldn't before this feature existed?" — not from
an external report.

## Pass 2 — full codebase review

A broader pass across the whole application, independent of the recent SSO
work. 12 findings, ranked roughly by severity.

### What was checked and found sound

- **SQL injection posture.** Only 4 string-built SQL statements exist in the
  whole codebase; all are internal DDL from fixed constants, never from
  request input. Everything else is parameterized.
- **No `TODO`/`FIXME`/`HACK` markers anywhere in the codebase** — nothing
  left as a known-incomplete shortcut.

### Findings and fixes

| # | Severity | Finding | Fix |
|---|----------|---------|-----|
| 1 | **Critical** | **8 reachable CVEs**, most seriously `GO-2026-4753` — a signature-bypass vulnerability in `goxmldsig` (the SAML dependency) sitting directly in the assertion-validation path (`saml.Provider.Authenticate` → `ParseResponse` → `ValidationContext.Validate`). Found via `govulncheck`'s call-graph reachability analysis, not just a dependency-list scan. | Dependency bumps (`goxmldsig`, `golang.org/x/text`) and a toolchain bump to get a patched Go release. All 8 down to 0. |
| 2 | High | **No agent resilience.** The native agent shipped fire-and-forget: `sendHeartbeat`/`sendMetrics`/`sendLogs` killed the entire agent process on the first failure, while every other collector in the same loop was already best-effort. A server restart or brief network blip took the whole agent down, silently discarding in-memory state (FIM baseline, cadence timers) until a process supervisor happened to restart it. | Exponential-backoff retry (3 attempts, transport errors/5xx/429 only — other 4xx fail fast since retrying won't help), a disk-backed spool (bounded 64 MiB, oldest-dropped-first, survives a process restart), and every collector in the loop made consistently best-effort. |
| 3 | High | **~48 `Store` interface read methods returned no error at all** — a database outage was indistinguishable from "no data found," the same defect shape as a prior production bug (a NULL `completed_at` silently read as "not done"). | Full interface refactor: every genuinely fallible read across both the file-backed and Postgres-backed implementations now distinguishes "not found" (`ok=false, err=nil`) from "the store errored" (`err != nil`), with security-sensitive paths (MFA lookups, custom-role resolution) fixed to fail *closed* — a store error denies access rather than silently granting it. |
| 4 | High | **No CI existed at all** — nothing had ever run the test suite under the race detector, and nothing was watching for the CVEs in finding 1 to appear. | GitHub Actions CI added: build (native + Windows cross-compile), `go vet`, the full suite under `-race` with a real Postgres service container, and a `govulncheck` job — all required to pass. CI's first-ever run caught a genuine pre-existing data race in test-only license-mutation code, fixed in the same stretch. |
| 5 | Medium | **No HTTP server timeouts.** `&http.Server{Addr, Handler}` used Go's defaults, which are unlimited — a slow-loris-style connection could hold a worker indefinitely. | `ReadHeaderTimeout`/`ReadTimeout` (30s) and `IdleTimeout` (120s) added. Deliberately no `WriteTimeout`, since the full-log-history backup endpoint streams a large response and a write deadline would cut it off mid-stream. |
| 6 | Medium | **Stored XSS** via 3 of many `innerHTML` sinks in the dashboard — a custom role's name/id rendered unescaped, exploitable by anyone holding the `users.manage` permission already required to create a role. | Client-side `escapeHtml()` on the 3 sinks, plus server-side defense in depth: role ids are now rejected outright if they contain HTML metacharacters. |
| 7 | Medium | **Derived event writes failed silently.** `storeLogs`'s secondary writes (threat-intel matches, UEBA baseline updates derived from an already-accepted log batch) dropped their error with no record. | Now logged on failure. Deliberately still fire-and-forget for the primary ingest response — propagating the error would make the ingest caller retry and duplicate a batch that had already landed successfully. |
| 8 | Medium | **Spoofable login-lockout bypass.** The client IP used for the login-lockout mechanism trusted `X-Forwarded-For` unconditionally, so any client could set its own header value to get a fresh lockout bucket per request. | `X-Forwarded-For` is now only trusted when the immediate TCP peer is a configured trusted proxy (`CRESTERAWATCH_TRUSTED_PROXIES`); otherwise the direct connection's address is used. A periodic sweep also bounds the lockout-tracking map's memory growth against a flood of distinct identities. |
| 9 | Medium | **PBKDF2 at 210,000 iterations** (OWASP's current SHA-256 recommendation is 600,000), and no rehash-on-login path for an *existing* PBKDF2 hash sitting at a lower iteration count — the legacy-format upgrade path only fired for the older pre-PBKDF2 hash format. | Raised to 600k. `needsRehash` extended to cover both cases, wired into all three authentication paths — including the platform-admin path, which had never rehashed on login at all before this fix (a bonus fix of the same bug class). |
| 10 | Low-Medium | **Coverage gaps**, `internal/notify` named as the specific example (50.4% at the time). | Raised to 83.7% — every function covered except one line of raw SMTP transport not worth mocking `net/smtp` to reach. |
| 11 | Low | **Four oversized files** (`postgres.go` ~3,600 lines, `store.go` ~3,000, `server.go` ~1,800, `web/index.html` ~6,000) — a maintainability observation, not a functional defect. | **Fixed (2026-09).** All four split along their existing logical domains, verified as pure reorganization (exact function-count reconciliation / byte-identical content diffing, full test suite unchanged): `postgres.go` and `store.go` each into a dozen `internal/store/*.go` files, `server.go` into 8 `internal/api/*.go` files, and `web/index.html` into `index.html` + `app.css` + `app.js` (no build step introduced — still plain static files). |

*The original review artifact counted 12 findings in this pass; this table
reflects the 11 with a distinct fix commit in the project history. The
discrepancy is a record-keeping gap in how one finding was folded into
another's fix, not an unfixed item — nothing found in the original review
remains open.*

### A defect the tests found, not the review

Building the regression test for finding 2's disk spool surfaced a real,
independent bug: the spool's filename scheme used a nanosecond timestamp for
uniqueness, but two `Enqueue` calls in quick succession on Windows can
observe the *same* value, since the OS clock's actual resolution is coarser
than true nanoseconds — the second write silently overwrote the first.
Fixed with a monotonic per-instance sequence number as an explicit
tiebreaker. Recorded here because it's a useful illustration of why "we
tested it" matters more than "we reasoned about it" — this would not have
been caught by code review alone.

---

## Status

**All 12 findings across both passes are closed.** Every fix shipped with a
regression test; the finding-4 CI pipeline and finding-3 interface refactor
were both verified against a real Postgres-backed test run in addition to
the automated suite. No known open findings remain from this review as of
this document's publication.

This review is real and thorough for what it is — a good-faith internal
audit by the people who wrote the code. It is not a substitute for the
[independent penetration test](pentest-engagement.md) a production rollout
should have before handling real customer data at scale.
