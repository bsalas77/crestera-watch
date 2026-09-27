# End-to-end verification — Ubuntu 26.04, 20 August 2026

First full-stack run of Crestera Watch on real Linux: the Go server, the SQLite log
engine, agent enrollment, search, Crestera Control forwarding, and Sigma import,
all against a running server on an Ubuntu VM. (The raw transcript is kept with
the source, not in the release package.)

Everything below was observed on the VM, not reasoned about.

## Proven

- **The static Go binary runs with no libc** — `CGO_ENABLED=0` with the pure-Go
  `modernc.org/sqlite`, confirmed `statically linked`. The single-binary,
  clean-cross-compile property holds with SQLite in the build.
- **The SQLite log engine works on Linux**, created with the intended
  permissions: data dir `drwx------` (0700), `cresterawatch.json` `-rw-------`
  (0600). The store-permission fix is verified on the real target, closing the
  gap that only skipped on Windows.
- **Agent enrollment and reporting**: a token minted via the API enrolled the
  agent, which reported host `ubuntu1` with live metrics (CPU cores, memory,
  load, processes).
- **Logs land and are searchable** — the whole reason for the engine rebuild.
  `log_volume_24h: 207`, and a search returned hits in **0 ms** through the FTS
  index.
- **Crestera Control forwarding** (`POST /api/ingest/crestera`): a real block
  payload was accepted (`202`) and is searchable as
  `Crestera Control: BLOCKED powershell.exe — ringfenced`, severity `critical`,
  every structured field (path, sha256, verdict, enforcement, reason, user)
  intact. A wrong ingest token is rejected `401`.
- **Sigma import** (`POST /api/detection-rules/import-sigma`): a rule compiled
  to `EventID=4625` and was stored.
- **Clean shutdown checkpoints the WAL**: the `-wal`/`-shm` files consolidated
  into the db on Close.

## Found and fixed

- **A configured admin password the server accepted but rejected at login.**
  `EnsureUser` skipped an existing admin, so `CRESTERAWATCH_ADMIN_PASSWORD` set after
  a first run (which had generated one) was ignored — the startup log said
  "configured admin password" while the login page rejected it. Fixed to make
  the configured password authoritative; the run then logged in cleanly with no
  bootstrap-credentials file. See the commit "A configured admin password the
  server accepted but rejected at login".

## Notes

- Two earlier attempts hit a port collision (a stale server on 8080) and then an
  OOM/RCU stall on the VM from accumulated orphaned processes. The script now
  kills all Crestera Watch processes before and after a run, and uses a dedicated
  port; the clean run above is the result.
- Readiness reports `degraded` only because agent-download files are absent from
  this bare-binary layout (a `warn`, not a failure) — expected outside the
  packaged install.
