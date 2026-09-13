# Crestera Watch

A self-hosted SIEM for Windows and Linux. Collection, detections, threat
hunting, automated response and compliance reporting, running on your own
hardware.

The Free edition is free for any number of hosts inside your own organisation,
commercial use included. No seat limits, no expiry, no quote process.

**[Download the latest release](../../releases/latest)**

---

## Why this exists

A credit union with 200 endpoints carries the same regulatory obligations as one
with 20,000, and nothing like the same budget. The usual answer to that is a
spreadsheet of log sources you cannot afford to collect. This is the other
answer.

## What the Free edition includes

| | |
|---|---|
| **Collection** | Native agent per host for logs, metrics, inventory, file integrity and posture. Syslog and NetFlow for devices that cannot run an agent. Windows Event Log and Linux syslog/auth.log out of the box. |
| **Detection** | Correlation and threshold rules over live telemetry, plus direct import from the Sigma community rule corpus. |
| **Hunting** | Structured query language across endpoint, network and audit data, with field faceting to narrow a result set. |
| **Response** | Isolate a host, block an address or kill a process from the console. Every action lands in the audit trail. |
| **Compliance** | Reporting mapped to nine frameworks, generated from what the agents actually observed. |
| **Assistant** | A local AI assistant that turns a plain question into a query, against your own model rather than a vendor's. |

Compliance frameworks: **FFIEC CAT**, **NCUA 748**, **GLBA Safeguards**, NIST CSF
2.0, NIST SP 800-53, CIS Controls, PCI DSS, ISO/IEC 27001:2022, and an Essential
Baseline. The first three are mapped directly rather than folded into a generic
NIST crosswalk, which is the distinction that costs a credit union a finding.

## Platforms

- **Windows** — Server 2019 and later, Windows 10/11
- **Linux** — amd64 and arm64: Ubuntu, Debian, RHEL family
- **Raspberry Pi** — 64-bit Raspberry Pi OS

Each release contains both the server and the agent for its platform.

## Install

Unzip the package for your platform, then:

**Windows**, from an elevated PowerShell:

```powershell
.\install-server.ps1
```

**Linux:**

```bash
sudo bash deploy/linux/preflight.sh
sudo bash deploy/linux/install-server.sh
sudo bash deploy/linux/verify-install.sh
```

Either way the installer prints a generated admin password and the dashboard
comes up on port 8080. Create an enrolment token there, then run the agent
installer on each host you want to watch.

Full walkthrough: **[QUICKSTART.md](QUICKSTART.md)**.

## Before you run it in production

Read the production checklist at the end of
[QUICKSTART.md](QUICKSTART.md#before-you-run-this-in-production). The short
version: **the installer falls back to plaintext HTTP when no certificate is
configured**, so that a first run does not simply refuse to start. That fallback
is not something to leave in place. This is a SIEM; it holds the credentials and
the log data for everything else you own.

## What this has and hasn't been through

Load tested at 2,000 events per second sustained across 300 hosts with zero
dropped events, and failover tested by killing the primary node mid-run
(recovery took about six seconds). Internally security reviewed, with
`govulncheck` clean.

It has **not** had an independent penetration test. That is the next piece of
work and it is tracked in the open:
[docs/security/pentest-engagement.md](docs/security/pentest-engagement.md).

[docs/security/security-overview.md](docs/security/security-overview.md) states
plainly what the product does and does not defend against. If you are evaluating
this for anything that matters, read that one first.

## Documentation

- [QUICKSTART.md](QUICKSTART.md) — install, first agent, production checklist
- [docs/deployment.md](docs/deployment.md) — storage profiles and which to choose
- [docs/linux-install.md](docs/linux-install.md) — Linux service and troubleshooting detail
- [docs/deployment-ha.md](docs/deployment-ha.md) — high availability
- [docs/security/](docs/security/) — security overview, hardening review, vendor questionnaire

## Licence

Free for unlimited internal use, including commercial use within your own
organisation. Redistribution, resale and hosting it as a service for others are
not permitted. See [LICENSE](LICENSE).

Third-party dependencies and their licences are listed in
[THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md). All are permissive, and none
require you to open-source anything.

## Pro edition

Multi-tenancy, SOAR playbooks, case management, UEBA and cloud AI are the Pro
edition, licensed separately. The Free edition is not a crippled trial of it;
it is the product with those modules held back.

## Feedback

Issues, rough edges and "this document did not work for me" reports are wanted.
Open an issue on this repository.
