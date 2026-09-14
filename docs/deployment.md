# Crestera Watch Deployment

Crestera Watch ships as two production download packages — one for Windows and one for
Linux — each installing the SIEM server and monitoring agent as managed services.

## Deployment Modes

### Windows

For a Windows Server or workstation. Installs as Windows Services.

Package:

```text
outputs\cresterawatch-windows-deploy.zip
```

Contents:

- `cresterawatch-api.exe` (SIEM server)
- `cresterawatch-agent.exe` (monitoring agent)
- `web\` (dashboard)
- `install-server.ps1`, `install-agent.ps1`
- `uninstall-server.ps1`, `uninstall-agent.ps1`
- `README-WINDOWS.md`

Install the server (elevated PowerShell):

```powershell
powershell -ExecutionPolicy Bypass -File .\install-server.ps1
```

This registers the `CresteraWatchServer` service, generates the admin password, and
serves the dashboard on `http://localhost:8080`.

### Linux (Ubuntu, 64-bit Raspberry Pi OS)

Use this for an Ubuntu laptop, Ubuntu server, or 64-bit Raspberry Pi OS.

Package:

```text
outputs\cresterawatch-linux-deploy.zip
```

Contents:

- Linux `amd64` server and agent binaries
- Linux `arm64` server and agent binaries
- systemd service files
- server install script
- agent install script
- dashboard files

Install server:

```bash
sudo bash deploy/linux/install-server.sh
```

Install agent:

```bash
sudo bash deploy/linux/install-agent.sh --server http://SERVER-IP:8080 --enroll-token TOKEN --profile server
```

Service checks:

```bash
systemctl status cresterawatch-api
systemctl status cresterawatch-agent
```

Open `http://localhost:8080` and sign in as `admin` with the password printed by
the server installer (also in `/etc/cresterawatch/server.env`).

For a lab HTTPS deployment:

```bash
sudo bash deploy/linux/setup-tls-self-signed.sh --hostname cresterawatch.local --ip SERVER-IP
```

Agents can trust that certificate with:

```bash
sudo bash deploy/linux/install-agent.sh --server https://SERVER-IP:8080 --enroll-token TOKEN --profile server --tls-ca-file /path/to/cresterawatch.crt
```

## Agent Identity

Agents use one-time enrollment tokens instead of a shared token. A token enrolls exactly one host. For 10 minutes after its first use, that same host (and no other) may redeem it again; this covers antivirus that runs a new agent once in a sandbox before letting it run for real. Each redemption issues a new agent credential and the previous one stops working. A revoked agent cannot use that window to come back; it needs a new token.

After enrollment, the agent sends:

- `Authorization: Bearer <agent-token>`
- `X-CresteraWatch-Agent-ID: <agent-id>`

The local store saves only the hash of the issued agent token.

### Standard Docker Profile

Use this for Linux servers, Raspberry Pi testing, and small work environments.

The standard profile includes:

- API/dashboard server
- Postgres
- OpenSearch
- VictoriaMetrics

Standard mode stores users, audit events, host metadata, enrollment tokens, agent token hashes, alert rules, alert mutes, alert history, and alert acknowledgements in Postgres.

Retention defaults are set for small-team use:

```text
CRESTERAWATCH_LOG_RETENTION_DAYS=14
CRESTERAWATCH_METRIC_RETENTION_DAYS=30
CRESTERAWATCH_AUDIT_RETENTION_DAYS=365
CRESTERAWATCH_ALERT_HISTORY_RETENTION_DAYS=180
```

The API server runs a cleanup job at startup and then hourly. In local mode it trims the JSON store directly. In standard mode it trims Postgres metadata and asks OpenSearch to delete old logs. VictoriaMetrics metric retention is controlled by the Docker `-retentionPeriod` value, which follows `CRESTERAWATCH_METRIC_RETENTION_DAYS`.

The dashboard backup controls work in standard mode too. They export and restore Postgres metadata, including users, roles, audit history, hosts, enrollment state, managed agent profiles, alert rules, alert mutes, alert acknowledgements, alert history, and alert notification delivery state. OpenSearch logs and VictoriaMetrics metric samples are not included in that backup yet, so those backend data directories still need their own infrastructure backup plan.

For a full standard-profile filesystem backup, use:

```text
sudo bash deploy/backup-standard.sh
```

This creates a timestamped folder under `backups/` with Postgres, API data, OpenSearch data, and VictoriaMetrics data. To restore one of those folders:

```text
sudo bash deploy/restore-standard.sh backups/cresterawatch-standard-YYYYMMDDTHHMMSSZ
```

Restore replaces the current standard-profile data.

Start from the project root:

```text
docker compose -f deploy/docker-compose.yml up -d --build
```

Use `--build` on upgrades too, so the server image is rebuilt from the new
binaries rather than reused. Data stays in the named volumes.

Open:

```text
http://localhost:8080
```

## Deployment Verification

Crestera Watch includes a readiness endpoint:

```text
GET /api/readiness
```

It returns the overall deployment state plus individual checks for local data write access, dashboard files, login/session configuration, retention configuration, and standard-mode services.

The Settings view includes a Deployment Status panel that shows the same checks. On Linux installs, use:

```text
sudo bash deploy/linux/verify-install.sh
```

The script checks API health, readiness, session availability, data permissions, retention config, web files, and agent enrollment endpoint reachability.

## Current Product Boundary

The application is production-ready for a single organization. It accepts agent heartbeats, metrics, and logs, serves the dashboard from the API server, supports team roles and audit history, and supports two storage modes.

Security baseline:

- User passwords are stored as salted PBKDF2-SHA256 hashes.
- Older simple password hashes are upgraded automatically after a successful login.
- Session cookies are signed, HTTP-only, and marked Secure when served through HTTPS or an HTTPS forwarding proxy.
- Repeated failed sign-in attempts are temporarily locked out by username and client address.
- Dashboard/API responses include baseline browser hardening headers.
- Native HTTPS is supported with `CRESTERAWATCH_TLS_CERT_FILE` and `CRESTERAWATCH_TLS_KEY_FILE`.
- Linux packages include a self-signed lab TLS setup helper.
- Log ingestion redacts common passwords, tokens, API keys, authorization headers, private keys, and AWS-style access keys before storage.

Storage status:

- Lite/local mode stores everything in `data\cresterawatch.json`.
- Standard Docker mode stores users, audit history, metadata, and agent credentials in Postgres.
- Standard Docker mode stores configurable alert rules, mutes, acknowledgements, and alert history in Postgres.
- Standard Docker mode writes metrics to VictoriaMetrics.
- Standard Docker mode writes and searches logs through OpenSearch.
- Retention status is visible in Settings, and cleanup runs automatically at startup and hourly.
- Deployment readiness is visible in Settings and available through `/api/readiness`.
- Dashboard backup and restore covers Postgres metadata.
- `deploy/backup-standard.sh` and `deploy/restore-standard.sh` cover Postgres, API data, OpenSearch data, and VictoriaMetrics data for full standard-profile recovery.

Notification setup:

- `CRESTERAWATCH_EMAIL_SMTP_HOST`
- `CRESTERAWATCH_EMAIL_SMTP_PORT`
- `CRESTERAWATCH_EMAIL_USERNAME`
- `CRESTERAWATCH_EMAIL_PASSWORD`
- `CRESTERAWATCH_EMAIL_FROM`
- `CRESTERAWATCH_EMAIL_TO`
- `CRESTERAWATCH_SLACK_WEBHOOK_URL`
- `CRESTERAWATCH_TEAMS_WEBHOOK_URL`

Agent service checks:

- `CRESTERAWATCH_SERVICE_CHECKS=web=http://127.0.0.1:8080,ssh=tcp://127.0.0.1:22`
- HTTP and HTTPS checks treat reachable responses below HTTP 500 as up.
- TCP checks verify that a connection can be opened.
- Failed checks create service-down alerts and appear in the host service-check panel.

Syslog intake:

- `CRESTERAWATCH_SYSLOG_ADDR=:5514` enables a UDP syslog listener on the API server.
- Use UDP `5514` for the default non-root Linux service.
- Devices that only support UDP `514` can send to a firewall/NAT redirect or local syslog forwarder that relays to `5514`.
- Syslog messages are redacted before storage and appear in the Logs view with source `syslog` or the parsed app name.

Managed agent profiles:

- `GET /api/agent-configs` lists collection profiles.
- `POST /api/agent-configs` updates profile interval, log paths, and service checks.
- Agents receive their assigned profile at heartbeat and apply changes without reinstalling.
- The Agents dashboard view includes a Collection profiles editor.

Alert delivery behavior:

- Successful alert notifications use a 30-minute cooldown to avoid repeated noise for the same active alert.
- Failed alert notifications record the error and use a 5-minute retry delay.
- Alert history records `notification_sent` and `notification_failed` events.
- Standard mode stores notification attempts, last attempt, last success, and last error in Postgres.

The Settings view shows configured channels and can send a test notification.
