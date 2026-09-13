# End-to-end verification — live Windows Server install, 3 September 2026

The other still-open piece of roadmap Phase 3: the Windows self-installer's
UAC → service registration → Programs & Features chain, verified so far
only by inspecting the built artifact, never by actually running it on a
real machine. This is that run — on a real VM (VirtualBox, Windows Server
2025 Standard Evaluation), not a container or a mock.

Everything below was observed on the VM, not reasoned about.

## A real bug, found only by actually running the installer

The first attempt failed. `cresterawatch.exe` ran, logged
`Crestera Watch — installing...`, registered the Windows service, and then:

```text
Error: install failed: register Windows Service: The service did not respond
to the start or control request in a timely fashion.
```

The service *was* registered (`Get-Service` showed `CresteraWatchServer`,
`Stopped`) — the failure was in starting it. The Service Control Manager's
own event log (`Get-WinEvent -LogName System`) narrowed it to a 30-second
start timeout (event 7009), and the service's own log file
(`C:\Program Files\Crestera Watch\Server\logs\cresterawatch-api.log`) had the real
answer:

```text
cresterawatch api: refusing to serve unencrypted: no TLS certificate configured.
Set CRESTERAWATCH_TLS_CERT_FILE and CRESTERAWATCH_TLS_KEY_FILE, or set
CRESTERAWATCH_ALLOW_PLAINTEXT_HTTP=true if this really is a local test...
```

**Root cause:** `internal/installer/installer.go`'s generated
`cresterawatch.env` never set `CRESTERAWATCH_ALLOW_PLAINTEXT_HTTP`, and left no TLS
certificate configured either — a combination the server's fail-closed TLS
check (`internal/config/listen.go`, added well after this installer was
first built) refuses outright. The process logged the refusal and exited
within about a second of starting; the Windows service wrapper never got
far enough to report "running," so the SCM just waited the full 30 seconds
and gave up with a generic timeout — no indication anywhere in the visible
error of the actual, one-line cause.

**Blast radius, checked once the pattern was clear:** all four documented
install paths generate this same combination out of the box —
`internal/installer/installer.go` (the Windows self-installer),
`deploy/windows/install-server.ps1` (the older Windows zip installer),
`deploy/linux/install-server.sh` (which copies `server.env.example`'s
deliberately conservative `CRESTERAWATCH_ALLOW_PLAINTEXT_HTTP=false` verbatim,
with no separate step telling an unattended install to override it), and
`deploy/docker-compose.yml`'s `api` service. Every one of them would leave
a fresh install refusing to start. Fixed in all four, same stretch: default
to plaintext-allowed on first install (a fresh install has no certificate
to offer yet), with the existing documented upgrade path (set
`CRESTERAWATCH_TLS_CERT_FILE`/`CRESTERAWATCH_TLS_KEY_FILE`, or run
`deploy/linux/setup-tls-self-signed.sh` for a lab HTTPS setup) unchanged.
Added a regression test (`TestRenderEnvFileContainsRequiredSettings`) that
locks the Windows installer's output to include the plaintext line, so this
exact defect can't silently return.

This is exactly the class of gap unit tests and CI don't catch: each piece
was correct in isolation (the TLS refusal is tested; the installer's file
generation is tested), and the *combination* only breaks when something
actually runs the installed service — which nothing had, until now.

## The verified run, after the fix

Rebuilt the Windows package, removed the broken install and service
registration entirely (`sc.exe delete`, then a fresh directory), and
re-ran the installer from scratch.

```text
Crestera Watch — installing...

Crestera Watch server installed and started.
Dashboard:  http://localhost:8080
            http://10.0.2.15:8080
Username:   admin
Password:   [redacted — a real generated password from this test VM; the
            installer prints a freshly generated one on every install]
Config:     C:\Program Files\Crestera Watch\Server\cresterawatch.env
Data:       C:\ProgramData\Crestera Watch\Server\data

Manage the service from services.msc (CresteraWatchServer).
Uninstall from Settings > Apps > Crestera Watch.
```

Confirmed independently, not just taken on the installer's own word:

- **Service registration and start:** `Get-Service CresteraWatchServer` →
  `Running`, display name `Crestera Watch Server`.
- **Dashboard actually reachable:** `(iwr http://localhost:8080/healthz).StatusCode`
  → `200`.
- **Programs & Features / Settings → Apps entry:** present in the registry
  Windows actually reads for that UI
  (`HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\`) —
  `Crestera Watch`, version `1.0.0`.

## Beyond "it's running": real agent enrollment and Hosts-module data

The verification above only proved the server process comes up healthy
(`Get-Service` → `Running`, `/healthz` → `200`). That's necessary but not
sufficient — it says nothing about whether an actual agent can enroll and
whether the dashboard's Hosts module shows real resource data or nothing at
all. Re-run to close that gap directly, on the same VM.

**Enrollment, end to end, against the live install:** logged into the API
(`POST /api/auth/login`), created a real enrollment token
(`POST /api/enrollment-tokens`), and started the packaged Windows agent
binary against it (`-server http://localhost:8080 -enroll-token <token>
-profile server`). After 15 seconds, `GET /api/hosts` returned a real
enrolled host (`WIN-72BORSA8MKS`, `"health": "healthy"`) with a fully
populated `latest_metric` object — real CPU core count and usage percent,
memory usage, disk usage, network throughput, process count, uptime — not a
placeholder or an empty shell.

**The dashboard UI, not just the API:** to answer "is this present *under
the Host module* as designed," not just "does the API return it," opened
the actual dashboard in a browser against the running instance and
navigated to Hosts. Confirmed:

- The Host inventory table lists the enrolled host with a live/online
  indicator, OS, Profile, a Health pill, and CPU/Memory/Disk percentages
  that visibly changed across successive screenshots (22% → 15% → 19% CPU)
  — proof of ongoing live collection, not a static first-heartbeat snapshot.
- The host-detail view shows Operating system, Profile, Last seen, and
  separate CPU-usage and Memory-usage tiles each with a live sparkline
  chart.

This is the same class of gap as the plaintext-HTTP bug above: unit tests
already cover `/api/hosts`' JSON shape and the dashboard's rendering logic
in isolation, but nothing before this had driven a real agent binary
against a real freshly-installed server and then looked at the rendered
page.

## A second, smaller install gap found in the same pass: firewall

While reaching the dashboard from outside the VM (to drive the browser
check above), a NAT-forwarded connection to port 8080 timed out even
though the service was healthy and the forwarding rule was confirmed
correct. `Get-NetFirewallRule -DisplayName '*8080*'` returned nothing — a
fresh Windows Server install has no inbound allow rule for arbitrary
application ports. Loopback traffic bypasses that filtering, which is why
every earlier `iwr http://localhost:8080` check from inside the guest had
worked; a connection from anywhere else, including the LAN address the
installer prints (`http://10.0.2.15:8080`), was silently dropped. Adding
`New-NetFirewallRule -DisplayName CresteraWatchDash -Direction Inbound
-LocalPort 8080 -Protocol TCP -Action Allow` fixed it immediately.

Lower severity than the plaintext-HTTP bug — a real deployment typically
touches firewall policy as a matter of course — but a real gap between what
the installer promises (a LAN-reachable URL) and what it delivers out of
the box. Fixed the same way as the TLS gap, across every path that prints
that promise:

- `internal/installer/installer.go` — `openFirewallPort` runs `netsh
  advfirewall firewall add rule` for the configured port after the service
  registers; `Uninstall` removes it. Best-effort (doesn't fail the install
  if it errors), since local access still works either way and an admin
  may manage firewall policy deliberately.
- `deploy/windows/install-server.ps1` / `uninstall-server.ps1` — the same
  rule via `New-NetFirewallRule`/`Remove-NetFirewallRule`.
- `deploy/linux/install-server.sh` — opens the port via `ufw` or
  `firewall-cmd`, but only if one of them is actually installed and active;
  silent no-op otherwise, matching how conservatively the plaintext-HTTP
  override was scoped.
- `deploy/docker-compose.yml` left alone: Docker's own iptables rules for a
  published port bypass ufw/firewalld by default on Linux, so this specific
  gap doesn't reproduce there. Not independently re-verified live this
  pass — noted rather than assumed silently.

## Notes

- Run from an already-elevated PowerShell session, so the installer's own
  UAC self-elevation (it carries a manifest requesting
  `requireAdministrator` — confirmed in `cmd/installer/main.go`) wasn't
  separately exercised as a fresh prompt this run. The manifest itself
  isn't in question; what this run adds beyond static inspection is
  confirmation that the *installed, running service* actually works — the
  part that was never proven before regardless of how elevation was
  reached.
- Guest keyboard input into the VM (VBoxManage `keyboardputstring`/
  `keyboardputscancode`) unreliably dropped the letter `p` and the `"`
  character specifically, this session — worked around with single quotes
  and PowerShell's `[char]112` for `p` where needed. A test-environment
  quirk, unrelated to the product; not investigated further since it isn't
  one.
- The Publisher field in the Programs & Features entry is empty (cosmetic,
  not investigated further this pass).
