# Standard Profile Architecture

## Components

```text
Native Agent
  -> Go API / Ingest Service
  -> Postgres metadata
  -> VictoriaMetrics metrics
  -> OpenSearch logs
  -> Alert worker
  -> Web dashboard
```

## Responsibilities

### Agent

- Enroll with a server URL and token.
- Exchange one-time enrollment tokens for agent credentials.
- Collect host identity and heartbeat data.
- Collect CPU, memory, disk, and network counters.
- Tail configured log files.
- Send batched telemetry to the ingest API.
- Receive a simple collection profile from the server.

Current starter collection paths:

- Linux: `/proc/meminfo`, `/proc/net/dev`, `/proc/loadavg`, and filesystem stats for `/`
- Windows: performance counters for CPU, memory pressure, logical disk free space, and network throughput

### API

- Accept agent enrollment.
- Require issued agent tokens for ingest.
- Accept heartbeats.
- Accept metrics and logs.
- Serve dashboard data.
- Store metadata in Postgres.
- Write logs to OpenSearch.
- Write metrics to VictoriaMetrics.
- Evaluate simple alert rules.

### Dashboard

- Show host status, system health, logs, and alerts.
- Generate agent install commands.
- Manage retention, profiles, and notification channels.
- Keep advanced search and rule writing optional.

## First Data Types

### Host

```json
{
  "id": "host_01",
  "name": "pi-lab",
  "os": "linux",
  "arch": "arm64",
  "profile": "server",
  "last_seen": "2026-07-15T22:00:00Z"
}
```

### Metric

```json
{
  "timestamp": "2026-07-15T22:00:00Z",
  "host_id": "host_01",
  "name": "cpu.usage_percent",
  "value": 41.2,
  "tags": {
    "core": "all"
  }
}
```

Metrics are sent as a batch:

```json
{
  "host_id": "host_01",
  "metrics": [
    {
      "timestamp": "2026-07-15T22:00:00Z",
      "host_id": "host_01",
      "name": "cpu.usage_percent",
      "value": 41.2
    }
  ]
}
```

### Log Event

```json
{
  "timestamp": "2026-07-15T22:00:00Z",
  "host_id": "host_01",
  "source": "syslog",
  "severity": "warning",
  "message": "disk usage exceeded warning threshold on /var/log"
}
```
