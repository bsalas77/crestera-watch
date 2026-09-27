# Release review, September 2026 (2.0.3)

An internal review of the Free edition before it went back up for download.
It was done by installing the released packages the way a stranger would, on a
fresh Ubuntu 26.04 server (systemd and Docker Engine) and on Windows 11 with
consumer antivirus running, then attacking what came up. Like the
[August hardening review](hardening-review-2026-08.md), this is an in-house
review, **not** a third-party penetration test.

Everything below was observed on a running install, not inferred from code,
unless it says otherwise.

## What was found and fixed

### Serious

**A viewer could enroll a rogue agent.** Listing enrollment tokens only
required being signed in, while creating one required the enrollment
permission. The lowest role, viewer, read an unused token from
`GET /api/enrollment-tokens` and used it to enroll a fake host, which could
then send forged logs and receive response commands. Listing now needs the
same permission as creating.

**The Docker profile shipped published secrets and open backends.** With no
settings, `docker-compose.yml` started with a default admin password and a
default session-signing secret, both printed in the package itself. The
session secret signs login cookies, so anyone who had read the file could
produce a valid admin session for a deployment that kept the default (this
was confirmed from the code; we did not build a forged session). Postgres,
OpenSearch and VictoriaMetrics were also published on every network
interface, with a fixed database password and OpenSearch security off. Now
the compose file has no default secrets and refuses to start until
`deploy/docker-setup.sh` has generated them, and only the dashboard port is
published.

**A password changed in the dashboard was put back on restart.** Startup
re-applied the password from the env file every time, so an admin who
rotated the installer's password found it reverted after the next restart,
and the installer's password, stored in plaintext on disk, kept working. The
configured password is now applied when the account is created and again
only when an operator puts a new value in the file, which is still how a lost
password is recovered.

**The agents dropped events under load.** On Linux the log reader stopped
after 100 lines and saved a file position that the reader had already read
past, so any log that grew by more than one batch between reads lost lines: a
test of 250 new lines delivered 223. On Windows each poll took the 25 newest
events past the bookmark and then moved the bookmark to the newest, so a burst
of failed logons on the Security log lost everything in between (reproduced on
a real channel). Both readers now advance only past what they delivered, and
batch sizes went up (2000 lines, 500 events per cycle). A newly installed Linux
agent starts near the end of existing logs instead of replaying them, so new
events aren't queued behind months of history.

**Two ways to run commands on every Windows agent.** The Event Log channel
names in the agent configuration were pasted into a PowerShell command
unquoted, and evidence collection listed a folder with `cmd /c dir <target>`,
which runs anything after an `&` or `|` in the target. Anyone who could edit
agent configurations or request evidence could have used either to run
commands as SYSTEM on managed hosts. The channel name is now quoted and the
folder listing is done without a shell.

### Medium

- **Collected logs were readable by every local user.** On Linux the log
  database was `0644` in a `0755` folder; on Windows the data and agent
  folders inherited read access for all users from `ProgramData`, which also
  exposed the agent's credential file. The Linux service now runs with
  `UMask=0077` and a `0700` data folder (upgrades are corrected in place); the
  Windows installers restrict both folders to SYSTEM and Administrators.
- **Anonymous reconnaissance.** `GET /api/status` told anyone the number of
  hosts, active alerts and daily log volume; it now needs a login.
  `GET /api/readiness` still answers anonymously for load-balancer health
  checks but only with the overall status; the per-check detail needs a login.
  `/downloads/` no longer lists its contents.
- **Weak service sandboxing on Linux.** systemd rated the server unit 8.4
  ("EXPOSED"). With `ProtectSystem=strict`, a system-call filter, no
  capabilities beyond binding a low port, and the rest of the standard set, it
  now rates 1.7 ("OK").
- **Windows ACLs used English group names.** `icacls` with "Administrators"
  fails on non-English Windows, which would have left the config file
  readable. The installers now use well-known SIDs.

### Usability defects that would have stopped a first install

- `verify-install.sh` checked the server before it had finished starting and
  reported four failures on a working install; it also expected the wrong
  status from the enrollment endpoint. Both installer and verifier now wait
  for the health check.
- `backup-standard.sh` and `restore-standard.sh` never worked: they stopped
  the containers, then looked them up with a command that only lists running
  ones. A backup, change, restore round trip now returns exactly the backed-up
  data.
- The Linux zip carried no Unix permissions, so a normal user could not open
  its folders. It is now built with proper modes, ships also as a `.tar.gz`
  (a fresh Ubuntu has no `unzip`), and unpacks into a single folder.
- On Windows, a slow first service start (antivirus scanning a new, unsigned
  program) made the installers stop before they printed the admin password or
  opened the firewall. They now retry for two minutes and always finish.
- Uninstalling from **Settings > Apps** always failed, because the installed
  `cresterawatch.exe` could not delete itself. It now removes itself after it
  exits.
- OpenSearch 2.12+ would not start in the Docker profile without a variable
  the fix above had removed; it now skips the demo security setup instead.
- Docs: README's Windows command was blocked by the default execution policy;
  several docs gave the wrong location for the admin password; the Docker
  section now includes the setup step.

## What held up

- Every attempt by a viewer to perform an admin action was refused (403).
- Path traversal against `/downloads/` and the dashboard was refused.
- Oversized (3 MB) and malformed request bodies were rejected.
- A string of HTML and script injected into syslog was stored verbatim and
  shown as text in the dashboard; no element was created and no script ran,
  checked in a real browser.
- Security headers and a nonce-based CSP are present on every page; the
  session cookie is HttpOnly and SameSite=Lax.
- Login lockout is keyed on username plus source address, so an attacker
  cannot lock the admin out globally from one address.
- Passwords use PBKDF2-SHA256 with constant-time comparison; backups are
  admin-only and contain no plaintext passwords.
- `govulncheck`: no reachable vulnerabilities.

## Still open

- **No independent penetration test yet.** See
  [pentest-engagement.md](pentest-engagement.md).
- **The Linux agent reads log files, not the systemd journal.** Debian 12+ and
  current Raspberry Pi OS do not write `/var/log/syslog` by default; install
  `rsyslog` there. The agent installer warns when it finds no log files.
- **The Windows binaries are not code-signed.** SmartScreen warns on first run,
  and some antivirus (Avast and AVG's CyberCapture) holds or blocks new
  unsigned programs for a while.
- The OpenSearch backend in the Docker profile runs with its security plugin
  disabled. It is reachable only from the server container, but a deployment
  that exposes it would need to turn security on.
