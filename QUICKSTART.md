# Crestera Watch Quickstart

## Windows

1. Unzip `cresterawatch-windows-deploy.zip`.
2. In an **elevated** PowerShell, install the server as a Windows Service:

```powershell
powershell -ExecutionPolicy Bypass -File .\install-server.ps1
```

3. Note the generated `admin` password printed by the installer, then open:

```text
http://localhost:8080
```

4. Create an enrollment token in the dashboard (Agents view), then install the
   agent on each host to monitor:

```powershell
powershell -ExecutionPolicy Bypass -File .\install-agent.ps1 -Server http://SERVER-IP:8080 -EnrollToken TOKEN
```

The agent enrolls with the one-time token and stores its own credentials under
`C:\ProgramData\Crestera Watch\Agent`. See `README-WINDOWS.md` for service management
and HTTPS.

## Standard Docker Profile

Ships in the **Linux** package (and the source repo) as `deploy/docker-compose.yml`;
it is not included in the Windows package. From the unzipped Linux package:

```text
docker compose -f deploy/docker-compose.yml up -d
```

Open:

```text
http://localhost:8080
```

This profile runs Postgres, OpenSearch and VictoriaMetrics alongside the server
instead of the single-file store, which is what you want once you are past a
handful of hosts. See `docs/deployment.md` for which profile to choose.

## Linux Server Or Raspberry Pi

Use this path for an Ubuntu laptop, Ubuntu server, or 64-bit Raspberry Pi OS.

1. Unzip `cresterawatch-linux-deploy.zip`.
2. Install the server:

```bash
sudo bash deploy/linux/preflight.sh
sudo bash deploy/linux/install-server.sh
sudo bash deploy/linux/verify-install.sh
```

3. Open:

```text
http://localhost:8080
```

4. Sign in with `admin` and the generated password from `/etc/cresterawatch/server.env`.

Optional lab HTTPS:

```bash
sudo bash deploy/linux/setup-tls-self-signed.sh --hostname cresterawatch.local --ip SERVER-IP
```
5. Create an enrollment token from the dashboard.
6. Install an agent on another Linux host:

```bash
sudo bash deploy/linux/install-agent.sh --server http://SERVER-IP:8080 --enroll-token TOKEN --profile server
```

Linux services:

```bash
systemctl status cresterawatch-api
systemctl status cresterawatch-agent
```

The Linux agent tails common system logs and tracks offsets in `/var/lib/cresterawatch-agent/log-offsets.json`.

The Settings view includes local JSON backup and restore controls.

## Before You Run This In Production

Everything above gets you running in a few minutes. Four of those defaults are
chosen for a first run, not for a production deployment, and this is a SIEM —
it holds the credentials and the log data for everything else you own.

**1. Turn on TLS.** The installer prints a warning and falls back to plaintext
HTTP when no certificate is configured, because refusing to start on a first run
is unhelpful. That fallback is not something to leave in place: dashboard
logins, agent tokens and all collected log data cross the network in clear text
without it. Set both paths and remove the plaintext opt-out, then restart:

```bash
# Linux: /etc/cresterawatch/server.env
CRESTERAWATCH_TLS_CERT_FILE=/etc/cresterawatch/tls/server.crt
CRESTERAWATCH_TLS_KEY_FILE=/etc/cresterawatch/tls/server.key
# delete or set to false:
CRESTERAWATCH_ALLOW_PLAINTEXT_HTTP=true

sudo systemctl restart cresterawatch-api
```

On Windows the same two keys go in `cresterawatch.env` in the install directory,
followed by `Restart-Service CresteraWatchServer`. `setup-tls-self-signed.sh`
produces a certificate good enough for an internal lab; use one from your own CA
or ACME for anything real, and point agents at it with `--tls-ca-file`.

**2. Replace the bootstrap admin.** The first run generates an admin password and
writes it to `admin-credentials.json` in the data directory so you can get in.
Create your own named accounts, then delete that file — it is a plaintext
password sitting on disk.

**3. Decide the storage profile before you load data.** The single-file store is
the default and is fine for a small estate. Past roughly a dozen hosts, or if you
need retention measured in months, move to the standard profile (Postgres +
OpenSearch + VictoriaMetrics) *before* you accumulate history, since that is the
cheaper moment to switch. See `docs/deployment.md`.

**4. Set retention to match your obligations.** The shipped defaults are 14 days
of logs, 30 of metrics, 365 of audit history and 180 of alert history. If you are
a credit union or bank, your examiner has an opinion about those numbers and it
is probably not 14 days.

Two more things worth knowing before you rely on this: it has **not** had an
independent penetration test (`docs/security/pentest-engagement.md` tracks that
openly), and `docs/security/security-overview.md` states plainly what it does and
does not defend against. Read that one before pointing it at anything you care
about.
