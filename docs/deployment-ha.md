# Multi-node deployment (high availability)

See [ADR-001](architecture/adr-001-ha-clustering.md) for the design and why
it's shaped this way. This document is the practical how-to.

**Applies to the standard/Postgres profile only.** The lite/single-file
profile (`CRESTERAWATCH_STORE=json`) stores everything in one local JSON file and
an embedded SQLite database — there is no shared backend for a second node
to coordinate through, and running two lite-mode nodes against the same data
directory is not supported. If you want redundancy, run the standard
profile.

## Topology

```text
                    ┌─────────────┐
   clients ───────► │ load balancer│
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌─────────┐  ┌─────────┐  ┌─────────┐
         │ node 1  │  │ node 2  │  │ node 3  │   any number of
         │(follower│  │(LEADER) │  │(follower│   stateless Crestera Watch
         │  )      │  │         │  │  )      │   API processes
         └────┬────┘  └────┬────┘  └────┬────┘
              └────────────┼────────────┘
                            ▼
              ┌─────────────────────────┐
              │  Postgres · OpenSearch  │   the operator's own
              │  · VictoriaMetrics      │   HA'd data tier
              └─────────────────────────┘
```

Every node runs the same `cresterawatch-api` binary, pointed at the same
Postgres/OpenSearch/VictoriaMetrics backends, behind any ordinary HTTP load
balancer. Nodes need no awareness of each other — leadership is negotiated
entirely through Postgres.

## The two settings every node must share

Set these identically on **every** node before starting more than one — the
server refuses to boot without the first on this profile (see below), and
the second breaks silently rather than loudly if it's inconsistent:

```text
CRESTERAWATCH_SESSION_SECRET=<the same random 32+ byte value on every node>
CRESTERAWATCH_CREDENTIAL_ENCRYPTION_KEY=<the same base64-encoded 32-byte key, if you use per-tenant SSO/connector credentials>
```

`CRESTERAWATCH_SESSION_SECRET` is **required** on the Postgres profile — the
server fails to start without it, rather than each node silently generating
its own and rejecting every other node's session cookies. Generate one value
and put it in every node's environment or `server.env`:

```bash
openssl rand -hex 32
```

`CRESTERAWATCH_CREDENTIAL_ENCRYPTION_KEY` is only required if you use per-tenant
SSO/LDAP/OIDC/SAML or connector credential storage; if you don't, it can stay
unset on every node consistently.

Everything else — `CRESTERAWATCH_ADMIN_PASSWORD`, TLS certificates, and so on —
can be set once and copied, or configured independently per node; they
don't need to match (a per-node TLS certificate for the node's own hostname
is normal, for instance).

## Health checks

Point the load balancer's health check at:

```text
GET /api/readiness
```

`200` means ready to serve; `503` means not ready. The response includes a
`cluster leadership` check reporting whether this specific node is currently
the leader or a follower — both are healthy states for a request-serving
health check; that field is informational (useful for confirming the
cluster is behaving as expected), not something the load balancer should
route on.

## What runs where

Every node serves the dashboard and the full API — reads, writes, agent
enrollment and ingest, everything a request needs is safe to answer from
any node.

Exactly one node at a time — the current leader — runs:

- Detection evaluation, notification, and automated response/SOAR playbook
  triggers
- Log/metric/audit retention cleanup and tiered-archive transitions
- Scheduled compliance report generation
- The Microsoft 365, Google Workspace, Azure Activity Log, and AWS
  CloudTrail connector polling loops
- The built-in syslog UDP/TCP listener (see below)

If the leader's process or host dies, another node picks up leadership
automatically, typically within a few seconds — no manual failover step, no
external tool to configure for this specific piece.

## Syslog ingestion

The built-in syslog listener only ever runs on the current leader — running
it on every node would mean every node accepts network syslog traffic with
no way to avoid ingesting the same message more than once. For multi-node
deployments that need syslog ingestion HA specifically (not just API/
dashboard HA), point network devices at one or more `cmd/collector`
instances instead: the standalone collector enrolls like an agent and
forwards over HTTP(S), which the same load balancer already in front of
your API nodes can distribute normally. This needs no new configuration on
the API tier itself.

## What you still own

Postgres, OpenSearch, and VictoriaMetrics HA is your infrastructure's job,
same shared-responsibility split as everywhere else in Crestera Watch's security
model — the application tier being safe to run more than once doesn't make
the data tier redundant on its own. Standard, well-documented options:
Postgres streaming replication or a managed HA offering; an OpenSearch
cluster with more than one data node; VictoriaMetrics's cluster edition if
metrics volume ever needs it (single-node VictoriaMetrics is fine for most
deployments and is what the default Docker Compose profile runs).

## Known limitations (tracked, not silent)

- **Login-lockout is per-node.** The brute-force lockout counter lives in
  each node's memory, not the shared store — an attacker who round-robins
  across nodes gets a fresh budget on each one. Lower severity than it
  sounds (PBKDF2 at 600k iterations is still doing the real work), but a
  real gap, tracked as a follow-up in ADR-001.
- **Archive-restore jobs are node-local.** A restore started on one node
  writes its scratch data to that node's local disk; searching it later
  needs to land back on the same node. Behind a load balancer without
  session affinity, this can produce a confusing "restore not found" on a
  genuinely completed restore. Also tracked as a follow-up in ADR-001.
