# Crestera Watch Linux Install

Use the Linux deployment package for Ubuntu laptops, Ubuntu servers, and 64-bit Raspberry Pi OS.

## Server

Unpack the package on the target machine. The `.tar.gz` needs nothing extra; the `.zip`
needs `unzip` (`sudo apt install unzip` on a fresh Ubuntu).

```bash
tar -xzf cresterawatch-linux-deploy.tar.gz
cd cresterawatch-linux-deploy
```

Install and start the server:

```bash
sudo bash deploy/linux/preflight.sh
sudo bash deploy/linux/install-server.sh
sudo bash deploy/linux/verify-install.sh
```

The verifier checks the running API, deployment readiness, login/session endpoint, web files, data directory permissions, retention settings, and the agent enrollment endpoint.

Open:

```text
http://localhost:8080
```

## Enable HTTPS For A Lab

After the server is installed, generate a self-signed lab certificate:

```bash
sudo bash deploy/linux/setup-tls-self-signed.sh --hostname cresterawatch.local --ip SERVER-IP
```

Open:

```text
https://SERVER-IP:8080
```

For agents that connect to this HTTPS server, copy the generated certificate to the agent host and point the agent at it:

```bash
sudo bash deploy/linux/install-agent.sh --server https://SERVER-IP:8080 --enroll-token TOKEN --profile server --tls-ca-file /path/to/cresterawatch.crt
```

For quick isolated testing only, the agent also supports:

```bash
--tls-insecure-skip-verify
```

Sign in with:

```text
Username: admin
Password: value printed by the installer or stored in /etc/cresterawatch/server.env
```

The server runs as `cresterawatch` using systemd:

```bash
systemctl status cresterawatch-api
journalctl -u cresterawatch-api -f
```

Configuration lives at:

```text
/etc/cresterawatch/server.env
```

Admin credentials are stored in the same file:

```text
CRESTERAWATCH_ADMIN_USERNAME
CRESTERAWATCH_ADMIN_PASSWORD
CRESTERAWATCH_SESSION_SECRET
```

Local data lives at:

```text
/var/lib/cresterawatch
```

Retention defaults can be adjusted in `/etc/cresterawatch/server.env`:

```text
CRESTERAWATCH_LOG_RETENTION_DAYS=14
CRESTERAWATCH_METRIC_RETENTION_DAYS=30
CRESTERAWATCH_AUDIT_RETENTION_DAYS=365
CRESTERAWATCH_ALERT_HISTORY_RETENTION_DAYS=180
```

The server applies cleanup at startup and then hourly. The Settings view shows the active retention policy and last cleanup time.

## Agent

Create an enrollment token from the dashboard, then install the agent on a Linux host:

```bash
sudo bash deploy/linux/install-agent.sh --server http://SERVER-IP:8080 --enroll-token TOKEN --profile server
```

To add Nagios-style service checks during install:

```bash
sudo bash deploy/linux/install-agent.sh --server http://SERVER-IP:8080 --enroll-token TOKEN --profile server --service-checks "web=http://127.0.0.1:8080,ssh=tcp://127.0.0.1:22"
```

## Optional Syslog Intake

To receive firewall, router, switch, NAS, or Linux syslog messages, edit `/etc/cresterawatch/server.env`:

```text
CRESTERAWATCH_SYSLOG_ADDR=:5514
```

Restart the API service:

```bash
sudo systemctl restart cresterawatch-api
```

Point devices to the Crestera Watch server on UDP `5514`. The default service runs without root privileges, so UDP `514` should be handled by a firewall/NAT redirect or local syslog forwarder.

You can also edit `/etc/cresterawatch/agent.env` later:

```text
CRESTERAWATCH_SERVICE_CHECKS=web=http://127.0.0.1:8080,ssh=tcp://127.0.0.1:22
```

Then restart the agent:

```bash
sudo systemctl restart cresterawatch-agent
```

The agent runs as a systemd service:

```bash
systemctl status cresterawatch-agent
journalctl -u cresterawatch-agent -f
```

Agent credentials are stored at:

```text
/var/lib/cresterawatch-agent/credentials.json
```

From 2.2 the agent signs its log batches (see
[log integrity](security/security-overview.md#log-integrity)). Its signing key
and the position of its log chain sit beside the credentials, readable by root
only:

```text
/var/lib/cresterawatch-agent/credentials.json.signing-key
/var/lib/cresterawatch-agent/credentials.json.chain
```

Back them up with the credentials, or not at all. If a host is restored from a
backup older than its last log batch, or loses these files, the server sees
batches signed by an unknown key or out of sequence and raises the log-tampering
detection. If that was you, use **Reset** next to the agent on the Agents page
(it is recorded in the audit trail), or enrol the agent again.

Agent log offsets are stored at:

```text
/var/lib/cresterawatch-agent/log-offsets.json
```

Default Linux log paths are configured in:

```text
/etc/cresterawatch/agent.env
```

## Troubleshooting

Run:

```bash
sudo bash deploy/linux/diagnose.sh
```

Uninstall while keeping data:

```bash
sudo bash deploy/linux/uninstall-agent.sh
sudo bash deploy/linux/uninstall-server.sh
```

Uninstall and delete local data:

```bash
sudo KEEP_DATA=false bash deploy/linux/uninstall-agent.sh
sudo KEEP_DATA=false bash deploy/linux/uninstall-server.sh
```

## Supported Architectures

- `amd64` for most Ubuntu laptops and servers
- `arm64` for 64-bit Raspberry Pi OS and ARM servers

32-bit Raspberry Pi OS is not packaged yet.
