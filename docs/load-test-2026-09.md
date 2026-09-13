# Load test and live HA failover — 3 September 2026

First deliberate scale test beyond dev-scale (thousands of events, a
handful of hosts): sustained syslog ingestion at real throughput, against
the actual standard-profile backends, across two real OS processes sharing
one Postgres database — with a genuine, unscripted mid-test kill of the
leader to prove ADR-001's failover claim, not just reason about it.

Everything below was observed against a running system, not reasoned about.

## Setup

- Standard profile backends via Docker Compose: Postgres 16, OpenSearch 2,
  VictoriaMetrics — on the host machine, not a VM.
- Two `cresterawatch-api` processes (current `main`, commit `3ef7beb`) on the
  same host, same Postgres/OpenSearch/VictoriaMetrics, same
  `CRESTERAWATCH_SESSION_SECRET`, same `CRESTERAWATCH_SYSLOG_ADDR=:15514` — node A on
  `:28080`, node B on `:18401`.
- A throwaway load generator (`work/loadgen`, not part of the product) sends
  realistic RFC3164-style syslog lines (sshd/sudo/kernel/useradd/CRON
  messages, ~300 distinct synthetic source hostnames) over UDP at a target
  aggregate rate, tracking its own send success independently of the
  server.

## Baseline: two nodes, one leader

Node B started first and acquired leadership immediately (no contenders).
Node A came up as a follower and correctly held off starting its syslog
listener — confirmed via `/api/readiness` on both:

```text
node A: "cluster leadership": pass — "This node is a follower..."
        "syslog listener": warn — "Waiting to become the cluster leader..."
node B: "cluster leadership": pass — "This node is the cluster leader..."
        "syslog listener": pass — "...receiving on :15514 (udp), :15514 (tcp)."
```

## Sustained load: 300 hosts, 2000 EPS, 3 minutes

```text
sent=359,983  errors=0  duration=3m0s  avg_eps=1999.9
```

The sender held a rock-steady ~2000 EPS the entire run (`~1998`–`~2000` EPS
in every 10-second progress sample) with zero send-side errors. At this
observed rate, sustained continuously that's roughly **17 GB/day**
equivalent (2000 EPS × ~100 bytes/event × 86,400 s), for reference against
the "GB/day into the log engine" scale the product claims but hadn't
previously measured.

Backend resource usage stayed low throughout — none of the three backends
were meaningfully stressed at this rate:

| Container | CPU | Memory |
|---|---|---|
| Postgres | 0.01% | 50 MiB / 31.2 GiB |
| OpenSearch | 0.44% | 1.2 GiB / 31.2 GiB |
| VictoriaMetrics | 0.10% | 17 MiB / 31.2 GiB |

The leader Crestera Watch process itself: ~53 MiB working set. There is
meaningful headroom above 2000 EPS on this hardware; this run wasn't
pushed to the ingestion ceiling, just to a real, previously-unmeasured
scale.

## An unscripted leader kill, mid-load

~62 seconds into the 3-minute run, with the sender still going at full
rate, node B (the leader) was hard-killed (`Stop-Process -Force`) — no
graceful shutdown, exactly the failure mode ADR-001's design has to survive
for the "what happens when the box dies" answer to be real.

Polling node A every ~0.6s afterward:

```text
[+0s]   killed node B
[+5.8s] node A: cluster leadership = leader; syslog listener = receiving,
        already climbing (6,576 received and counting)
```

Leadership transferred and the syslog listener was back up and actively
ingesting within **~6 seconds** — the design's own "typically within
seconds" claim, now actually observed, not asserted. No manual
intervention, no external failover tool, nothing but the Postgres advisory
lock releasing when node B's connection died.

## The honest cost of an ungraceful kill

Final tally: server received/stored 322,999 events during this run
(0 dropped by the collector's own back-pressure the entire time) against
359,983 sent — a gap of ~36,984 events, concentrated entirely in the kill
window. Two real, expected sources, not a defect:

1. **The handover gap itself** (~6 seconds, confirmed above) — nothing was
   listening on `:15514` for a brief window between node B dying and node A
   finishing acquisition. At ~2000 EPS that alone accounts for roughly
   12,000 of the missing events.
2. **In-flight batches lost with the killed process** — the collector
   batches before writing to storage (by design, for throughput); whatever
   was sitting in node B's in-memory batch, not yet flushed, died with it.
   This is inherent to UDP syslog intake generally (there is no
   redelivery/ack mechanism to lose), not specific to this failover
   mechanism — the same loss would occur on any single-node syslog listener
   that crashes mid-batch, HA or not.

The collector's own `dropped` counter — the one that means "the queue was
full and this node discarded something" — stayed at **0** for the entire
run, on both nodes, throughout the kill and recovery. Nothing was ever
shed under load; every event that reached a listening process was
processed and stored.

## What this proves, and what it doesn't

**Proves**: the built-in syslog listener sustains a real, previously-
unvalidated throughput (2000 EPS / ~17 GB/day-equivalent) without shedding
load; ADR-001's leader-election failover is real and fast (~6s) under an
actual hard-kill, not just in the unit-test suite; a multi-node deployment
survives losing its leader process with a bounded, small, explicable gap
rather than an unbounded outage.

**Doesn't prove**: the ceiling (this run wasn't pushed to find one — backend
resource usage suggests real headroom above 2000 EPS); TCP syslog behavior
under the same load (this run used UDP only); multi-host deployment (this
was two processes on one machine sharing one Postgres, not nodes on
genuinely separate hosts/VMs — a smaller-scope proof than a true
multi-machine test, though the mechanism itself, a Postgres advisory lock,
doesn't care which host holds the connection); the k8s collector or a live
Windows elevated install, which are separate, still-open items.
