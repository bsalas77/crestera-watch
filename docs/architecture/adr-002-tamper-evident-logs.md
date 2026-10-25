# ADR-002: Tamper-evident log shipping

**Status:** Accepted — phase 1 in 2.2.0; phase 2 planned
**Date:** 2026-10-25
**Deciders:** Project owner (Barry Salas)

## Context

External feedback on the Free release asked for "tamper-evident log shipping".
Before 2.2, an agent posted each log batch as a bare JSON array over TLS with
its bearer token. That proves the sender held the token, and nothing else:

- a batch lost in transit, evicted from the agent's offline spool, or
  deleted by someone on the path never shows up anywhere — the server cannot
  tell "nothing happened" from "the evidence is gone";
- a batch can be replayed, and anyone holding the agent's token can post
  arbitrary events that look exactly like real ones;
- once stored, nothing records what was stored, so an edit or deletion in
  the log store leaves no trace.

Regulated customers (FFIEC, NCUA Part 748, GLBA, PCI DSS 10.3, NIST SP
800-92) are asked to show that collected logs are complete and unaltered.

## Decision

Two phases.

### Phase 1 (2.2.0): signed, sequenced, hash-chained agent batches

Each agent holds an **Ed25519 key** (generated locally, stored in its
credential file, never sent anywhere except the public half). Every log batch
is **sealed** before it is sent or spooled:

```
seq      monotonically increasing per agent, starting at 1
prev     hash of the previous batch (64 zeros for seq 1)
events   the batch's events, exactly as sent (SHA-256 over the raw bytes)
dropped  sequence ranges the agent itself discarded (spool overflow, or a
         failed send with no spool), so the server can tell an admitted
         loss from an unexplained one
sig      Ed25519 over a fixed signing input built from all of the above
hash     SHA-256 of that signing input; becomes the next batch's prev
```

The agent persists its chain head (seq, hash) **before** a batch leaves the
sealer, so a crash loses at most that batch — which the server then reports
as a gap, honestly — and never reuses a sequence number.

The server keeps one **chain state** per agent (public key, last seq, last
hash, the hashes of recent batches, anomaly counters) in the shared store:
the embedded SQLite log database for single-node installs, Postgres (tenant
row-level security, compare-and-swap updates) for clustered ones, so every
node in a cluster sees the same chain.

For each batch the server classifies:

| Situation | Status | What happens |
|---|---|---|
| Signature valid, `seq = last+1`, `prev = last hash` | `verified` | stored |
| Signature valid, `seq > last+1` | `verified` + **gap** | stored; a `log_integrity.gap` event says which batches never arrived, and whether the agent admitted dropping them |
| Signature valid, `seq = last+1`, `prev` differs | `chain_break` | stored, flagged, alert |
| Same `seq` and hash as a batch already received | duplicate | not stored again (a retried send), `200` |
| `seq ≤ last`, different content | `replay` | stored, flagged, alert |
| Signature invalid, or signed with a different key | `bad_signature` | stored, flagged, alert; chain not advanced |
| Unsigned batch from an agent that has signed before | `unsigned` (downgrade) | stored, flagged, alert |
| Unsigned batch from an agent that never signed (pre-2.2) | `unsigned` | stored, counted, no alert |

**Nothing is discarded.** A SIEM that throws away events because they look
suspicious loses exactly the evidence an investigation needs. Suspicious
batches are stored with a `log_integrity` field on every event and raise a
detection, so an analyst can search `log_integrity=bad_signature` and decide.

**Key registration** is trust-on-first-use: the first validly signed batch
from an agent registers its public key (recorded in the activity log).
Enrolment and an admin "reset log chain" action clear the key and sequence,
so a re-enrolled or rebuilt agent can register a new one. A later batch
signed with a different key is `bad_signature`, never a silent key change.

Built-in detection rules (Defense Evasion, T1070/T1565) fire on the
`log_integrity.*` events, and the Agents page shows each agent's chain status.

### Phase 2 (planned): what is stored stays as stored

- A server-side **ledger**: every stored batch (agent, syslog, webhook,
  connector) gets an entry with the SHA-256 of what was written and the hash of
  the previous entry.
- **Signed checkpoints** of the ledger head, hourly, with the server's own
  Ed25519 key; optionally written to WORM storage (S3 Object Lock) so not even
  a server administrator can rewrite them.
- `cresterawatch verify-logs --from --to`: recomputes the ledger from the log
  store and reports any event changed or removed since it was written.
- The **audit trail** hash-chained the same way.
- Retention deletes are recorded as signed "pruned up to" ledger entries, so a
  retention purge is distinguishable from tampering.

## What this does and does not prove

- It **does** prove a stored agent batch is byte-for-byte what that agent's
  key signed, detect missing, replayed and injected batches, and make a stolen
  bearer token insufficient to forge an agent's logs undetected.
- It **does not** prove the events are *true*: an attacker with root on the
  monitored host can read the agent's key and sign whatever they like. Signing
  moves the trust boundary to the host; it cannot remove it.
- Phase 1 does **not** prove the server's stored copy is unaltered after
  ingestion; that is phase 2.
- Syslog, NetFlow and webhook sources are not signed at the source (the
  protocols don't allow it); phase 2's ledger covers them from the moment
  they are stored.

## Compatibility

- Pre-2.2 agents keep working: the server still accepts the bare-array format
  and shows those agents as "unsigned".
- A 2.2 agent seals any unsigned batches waiting in its offline buffer
  (from before the upgrade) before it seals a new one, so sequence numbers
  always follow sending order.
- A 2.2 agent talking to a pre-2.2 server gets "invalid log batch" back and
  resends that batch as a plain array, so upgrading agents before the server
  loses nothing; the batches are simply unsigned until the server is upgraded.
- On re-enrolment the agent discards its key and chain, and turns any batches
  still in its buffer back into plain events so they are signed again under
  the new key.
- A batch the server permanently refuses (malformed, too large) is dropped
  from the buffer rather than blocking it, and reported as dropped.
