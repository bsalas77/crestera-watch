# ADR-001: High availability and multi-node clustering

**Status:** Proposed
**Date:** 2026-09-03
**Deciders:** Project owner (Barry Salas)

## Context

Crestera Watch runs today as a single Go process. If that process or its host dies,
the honest answer to "what happens to monitoring while it's down" is "you
lose it until it's back" — no leader election, no shared-state clustering,
no documented multi-node topology. This is roadmap Phase 5: the one
architectural question every enterprise security buyer eventually asks.

This ADR is a design decision, not an implementation — per the roadmap, HA
deserves its own pass before any code lands, since the right answer depends
on state Crestera Watch already has, not a generic HA pattern applied blind.

### What the codebase actually does today (read from source, not assumed)

**Two deployment profiles, and they don't have the same answer:**

- **Lite/single-organization profile** (`CRESTERAWATCH_STORE=json`): an
  embedded JSON application-data file plus an embedded SQLite log engine
  (`cresterawatch-logs.db`), both on local disk. There is no mechanism for a
  second process to share this state — SQLite's file locking is not
  reliable over a network filesystem, and nothing in this codebase
  attempts it.
- **Standard profile** (`CRESTERAWATCH_STORE=postgres`): Postgres for metadata
  (including, since Phase C/D/E, cases/UEBA/compliance/host state and
  agent-tenant identity), OpenSearch for logs, VictoriaMetrics for metric
  samples. All three are already services Crestera Watch talks to over the
  network, not files on its own disk.

**Conclusion of this alone**: HA can only ever apply to the standard
profile. The lite profile's storage model is single-node by construction,
the same boundary multi-tenancy already drew (`ForTenant` is a no-op on
`FileStore` — see `internal/store/store.go`). This ADR does not propose
changing that; it proposes what a *second Crestera Watch process* pointed at the
same standard-profile backends needs to be safe.

**Session authentication is already stateless — a real point in HA's
favor.** `signSession` (`internal/api/auth.go`) issues an HMAC-signed cookie
carrying username/role/tenant/expiry; verification just re-checks the HMAC
against `cfg.SessionSecret`. There is no server-side session store to
share. Any node holding the same `CRESTERAWATCH_SESSION_SECRET` can validate a
cookie issued by any other node. **The catch**: `sign()` falls back to
`cfg.AdminPassword` when `SessionSecret` is unset, and `AdminPassword` is
auto-generated per-process on first run when not configured
(`EnsureBootstrapAdmin`, `cmd/api/main.go`). Two nodes that both leave
`CRESTERAWATCH_SESSION_SECRET` unset would each mint a different secret and
silently reject each other's sessions — this is the single most important
prerequisite for anything below, and it's a five-minute fix independent of
the rest of this ADR (see Action Items).

**Nine background loops start unconditionally, once per process, with zero
coordination** (`NewServer`, `internal/api/server.go:278-310`):
`retentionLoop`, `tierTransitionLoop`, `loginAttemptSweepLoop`,
`runSyslogListener`, `detectionLoop` (notifications + automated response +
SOAR playbook triggers), `complianceLoop`, `m365Loop`, `gworkspaceLoop`,
`awstrailLoop`, plus `azlogLoop`. Every one of these has an externally
visible side effect if it fires twice: `detectionLoop` sends a real email/
Slack/webhook and can run a real containment script; `complianceLoop`
generates and delivers a report digest; the connector loops advance a
resume watermark and ingest possibly-overlapping data; `retentionLoop`/
`tierTransitionLoop` delete and archive rows. **Running two nodes of
today's code against the same Postgres database means every one of these
fires twice, independently, with no dedup.** This is the actual blocker to
active-active, not a generic "distributed systems are hard" concern — it's
nine concrete, named pieces of code.

**Two more node-local footguns, smaller but real:**

- `loginAttempts` (the brute-force lockout, `internal/api/server.go`) is an
  in-process map. Behind a load balancer with no session affinity, an
  attacker gets a fresh lockout budget on every node — the login rate
  limit is only as strong as `1 / (number of nodes)`.
- Archive-restore scratch databases (`internal/api/archive.go`) are written
  to `cfg.DataDir/archive-scratch` on whichever node started the restore. A
  later "search this restore" request landing on a *different* node 404s
  even though the restore genuinely completed.

**One asset already exists that solves a problem this ADR would otherwise
have to solve from scratch**: `cmd/collector` (the standalone syslog/NetFlow
forwarder) already enrolls like an agent and forwards to the central API
over HTTP(S) — it's a distinct binary specifically for fronting network
telemetry at the edge. It is the natural answer to "how do multiple syslog
listeners share a port" (see Decision, below) without inventing anything
new.

## Decision

**Active-active for request serving, leader-elected active-passive for
background work — coordinated with a Postgres advisory lock, not a new
dependency.**

Concretely:

1. Every Crestera Watch node runs the HTTP API/dashboard and can serve any
   request, once `CRESTERAWATCH_SESSION_SECRET` and
   `CRESTERAWATCH_CREDENTIAL_ENCRYPTION_KEY` are required to be set explicitly
   (not defaulted) on the standard profile. Put all nodes behind any
   ordinary HTTP load balancer with a health check against `/api/readiness`.
2. Exactly one node at a time holds a Postgres **session-scoped advisory
   lock** (`pg_try_advisory_lock`) and is the *leader*. Only the leader
   starts the nine background loops and the built-in syslog UDP/TCP
   listener. Every other node polls for the lock on a short interval (e.g.
   5s) and stays passive until it acquires it.
3. Failover is automatic and needs no new bookkeeping: if the leader's
   process dies, its Postgres connection drops, and Postgres itself
   releases the session-scoped advisory lock — the next node's poll picks
   it up within one interval. No heartbeat table, no TTL logic, no
   election protocol to get wrong.
4. The built-in syslog listener stays leader-only for v1. For a deployment
   that needs syslog ingestion HA specifically, the documented answer is
   **use `cmd/collector` in front of the API tier** — it already exists,
   already enrolls and forwards over HTTP(S) (which the load balancer can
   already fan out across every node), and needs no new code. This turns
   "the built-in listener is a single point of listening" from a gap into
   a documented, already-solved deployment choice.

## Options Considered

### Option A: Active-passive only (single primary + cold/warm standby)

| Dimension | Assessment |
|---|---|
| Complexity | Low — a health check + external failover tool (VRRP/keepalived, or a cloud LB) |
| Cost | Standby capacity sits idle |
| Scalability | None — read/API capacity never grows past one node |
| Team familiarity | High — conventional, well-understood pattern |

**Pros:** Simplest to reason about; zero risk of the duplicate-side-effect
class of bugs above, since only one node is ever doing anything.
**Cons:** Wastes every standby node's capacity; failover depends on
external tooling Crestera Watch doesn't control; doesn't help the actual scaling
question (more dashboard/API/search load) at all.

### Option B: Full active-active (every node runs everything)

| Dimension | Assessment |
|---|---|
| Complexity | High — all nine loops plus the syslog listener need reworking |
| Cost | Full capacity used everywhere |
| Scalability | Best, on paper |
| Team familiarity | Requires new idempotency/dedup discipline throughout |

**Pros:** No leader-election component at all; every node pulls equal
weight.
**Cons:** Requires making nine independently-evolving pieces of code
safe under concurrent multi-node execution — each with a different shape
of side effect (webhook, script exec, resume-watermark race, delete-vs-
archive race). This is a large, correctness-critical surface for a single
ADR to sign off on, and every future change to any of those nine loops
would need to re-justify its concurrency safety forever after. Realistic
failure mode: a subtle bug ships that double-sends an alert or double-runs
a containment script — the worst possible category of bug for a security
product to ship quietly.

### Option C: Hybrid — active-active serving, leader-elected background work (recommended)

| Dimension | Assessment |
|---|---|
| Complexity | Medium — one new small package (`internal/cluster`, advisory-lock leadership), gate 10 existing `go func()` call sites behind it |
| Cost | Every node's request-serving capacity is used; background-loop capacity is bounded to one node's worth, which is fine — these are lightweight periodic jobs, not the bottleneck |
| Scalability | Read/API/search capacity scales with nodes added; background-loop throughput does not, but nothing here suggests it needs to for Crestera Watch's actual target scale (credit unions and similarly-sized security teams, not a hyperscale MSSP with thousands of tenants on one cluster) |
| Team familiarity | Postgres advisory locks are a well-documented, single-function primitive — no new service to operate, no client library beyond the Postgres driver already in `go.mod` |

**Pros:** Reuses infrastructure the deployment already has (Postgres) rather
than introducing etcd/Consul/ZooKeeper purely to elect a leader for nine
lightweight loops. Correctness is easy to reason about: "exactly one
process holds the lock" is a much smaller claim than "nine functions are
each independently safe under concurrent execution." Failover is genuinely
automatic, no separate health-check/failover tool to configure. Every
future new background loop just needs to be added inside the same
"if leader" gate — the safety property doesn't need re-deriving per loop.
**Cons:** Background-loop work itself doesn't scale past one node (a real
but currently theoretical limit — nothing here is close to being CPU-bound
on a 60-second detection cycle); the syslog listener and archive-restore
scratch-path node-affinity issues still need their own answers (addressed
in Decision and Consequences).

## Trade-off Analysis

The real question isn't "active-active vs active-passive" in the abstract
— it's *which parts* of Crestera Watch are safe to run more than once, and the
codebase already answers that unevenly: request serving is stateless and
safe today (module the shared-secret prerequisite); background work is
uniformly unsafe today. Option C is the only one of the three that treats
those two halves differently instead of forcing one clustering model onto
both. Option A wastes the half that's already safe (request serving).
Option B tries to fix the half that's genuinely hard (nine loops with real
external side effects) as a prerequisite to shipping anything, which is a
much bigger and riskier lift than the actual near-term need (a customer
wants "the dashboard doesn't go down if a server reboots," not "we can
horizontally scale detection evaluation").

Advisory locks specifically (over, say, a hand-rolled heartbeat-row-with-
TTL scheme) win because Postgres already solves the hard part — detecting
that a leader is gone — for free, as a consequence of connection loss. A
heartbeat table requires deciding a TTL, tuning it against false failover
under GC pauses or brief network blips, and writing the sweep logic; a
session-scoped advisory lock has no TTL to tune because it isn't time-based
at all.

## Consequences

- **Easier**: rolling zero-downtime upgrades (drain traffic from one node
  at a time behind the LB; the leader lock migrates automatically if the
  current leader is the one being drained); redundant dashboard/API
  availability across a host or process failure; a documented, buyer-facing
  answer to "what's your HA story" for the trust package
  (`docs/security/security-overview.md`'s *availability model* line, which
  currently says "single-instance... on the roadmap").
- **Harder**: deployment now has a hard prerequisite (shared session/
  credential secrets) that a misconfigured rollout can get wrong silently
  in a way that's confusing to debug (two nodes both accepting logins but
  rejecting each other's sessions) — worth a startup-time check, not just
  documentation (see Action Items).
- **Needs revisiting later, not now**: `loginAttempts`' per-node lockout gap
  (a real but lower-severity issue — worth a follow-up once the leader
  mechanism exists, since the fix is now "read this from the shared store
  instead of a local map," a small, well-scoped change); VictoriaMetrics'
  own cluster edition, if metric ingest volume ever becomes the bottleneck
  (out of scope — that's the operator's infrastructure choice, same
  shared-responsibility framing as Postgres/OpenSearch HA); whether the
  built-in syslog listener should eventually get real multi-listener
  support instead of staying leader-only, if `cmd/collector`-fronting turns
  out to be an unwelcome extra deployment step for customers who'd rather
  point network gear straight at the API tier.

## Action Items

1. [x] **Ship independently of the rest of this ADR**: on the standard/
   Postgres profile, fail closed at startup if `CRESTERAWATCH_SESSION_SECRET` or
   `CRESTERAWATCH_CREDENTIAL_ENCRYPTION_KEY` is unset, instead of silently
   falling back to `AdminPassword` / refusing writes later. Cheap, and closes
   the single most confusing multi-node misconfiguration on its own.
   (`3ef7beb`)
2. [x] Add `internal/cluster` (or similar): `AcquireLeadership(ctx,
   pool) (leader <-chan struct{}, lost <-chan struct{})`-shaped API around
   `pg_try_advisory_lock`/`pg_advisory_unlock`, with its own adversarial
   test (two connections racing for the same lock; a held connection dying
   and a second acquirer picking it up within the poll interval). (`3ef7beb`)
3. [x] Gate the ten `go func()` call sites in `NewServer` behind "is
   leader," each starting/stopping cleanly as leadership is gained/lost.
   (`3ef7beb`)
4. [x] Extend `/api/readiness` (and the Settings → Deployment Status panel)
   with this node's leader/follower state, so an operator can see it
   without cross-referencing logs. (`3ef7beb`)
5. [x] Write `docs/deployment-ha.md`: the load-balancer + shared-Postgres/
   OpenSearch/VictoriaMetrics topology, the two secrets that must match
   across nodes, and the `cmd/collector`-fronting recommendation for syslog.
   (`3ef7beb`)
6. [x] Update `docs/security/security-overview.md`'s availability-model line
   once this ships, so the trust package stops saying HA is unbuilt.
   (`db27169`)
7. [x] Follow-up, not blocking: move `loginAttempts` to the shared store so
   the lockout budget is per-account, not per-account-per-node. Shipped:
   `Server.loginAttemptStore` (`internal/store/login_attempts.go`) backs
   `loginLocked`/`recordFailedLogin`/`clearFailedLogin` with a real Postgres
   table on the standard profile, nil (in-memory map, unchanged) on
   lite/FileStore. Adversarial regression test constructs two independent
   `*Server`s sharing one database and proves a lockout split across both is
   enforced on both (`internal/api/login_lockout_ha_test.go`).
8. [x] Follow-up, not blocking: give archive-restore jobs either session
   affinity (route by job id) or a shared/replicated scratch location, so a
   restore started on one node is searchable from any node. Shipped: the
   shared-storage option — `runRestoreJob` uploads the merged scratch
   database through the deployment's own archive backend (keyed off the job
   id) on the Postgres/HA-capable profile; `searchArchiveRestore` rehydrates
   a local copy on demand if this node doesn't already have one
   (`ensureLocalRestoreScratch`). Every node computes its own local scratch
   path independently from the job id (`localRestoreScratchPath`) rather
   than trusting the DB row's stored path, which belongs to whichever node
   originally built it. Adversarial regression test builds two `*Server`s
   with separate local `DataDir`s sharing one archive backend directory and
   proves a restore built entirely on one is searchable entirely from the
   other (`internal/api/archive_restore_ha_test.go`).
9. [x] Retroactively added, same architectural family as 7 and 8: API rate
   limiting (`internal/api/ratelimit.go`, shipped after this ADR's original
   scope was written) inherited the identical per-node problem the moment
   it existed — an identity able to reach every node behind a load
   balancer got each node's full budget independently. Shipped: each node
   periodically (every few seconds — `distributedRateLimitSyncLoop`, not a
   write on every request) heartbeats its presence to a shared
   `rate_limit_nodes` table and divides its configured share of each
   deployment-wide budget by however many nodes are currently active
   (`internal/ratelimit.Limiter.SetRate`). Deliberately approximate, not
   exact: a few seconds of staleness in the node count, in exchange for
   the request path itself never touching the database — the same
   philosophy as the advisory-lock leadership mechanism above (Postgres
   already has the primitive; don't add a per-request cost to get exact
   consensus nothing here actually needs). Adversarial regression test
   constructs two independent `*Server`s sharing one database and proves
   each converges to enforcing half the configured burst, and that a lone
   node keeps the full budget
   (`internal/api/ratelimit_distributed_test.go`).
