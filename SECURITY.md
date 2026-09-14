# Security Policy

> **Evaluating Crestera Watch?** For due-diligence documents — a security & data-
> handling overview and a completed vendor security questionnaire — see
> [`docs/security/`](docs/security/README.md).

## Reporting a vulnerability

If you believe you've found a security vulnerability in Crestera Watch, please
report it privately so it can be fixed before public disclosure.

- **Do not** open a public issue for security problems.
- Report it through GitHub: open this repository's **Security** tab and choose
  **Report a vulnerability**. The report stays private between you and the
  maintainer.
- Or email **barry.salas@gmail.com**. Either way, include:
  - a description of the issue and its impact,
  - steps to reproduce (proof-of-concept if possible),
  - affected version / commit.
- You can expect an acknowledgement within **5 business days** and a status
  update as a fix is developed. Please allow reasonable time to remediate
  before any public disclosure (coordinated disclosure).

## Scope

Crestera Watch is self-hosted commercial software distributed as a Free and a Pro
edition from a private source repository. The maintainer addresses
vulnerabilities in the application code itself. Issues in a specific
deployment's configuration, hosting, or the third-party services it's wired
to (an identity provider, a cloud AI provider, an S3-compatible archive
backend) are the responsibility of that deployment's operator.

## Supported versions

Security fixes are applied to the latest release on the `main` branch. Pin to
a release and watch for updates.

## Deploying securely

Before exposing a deployment to untrusted networks: set `CRESTERAWATCH_TLS_CERT_FILE`
/ `CRESTERAWATCH_TLS_KEY_FILE` (the server refuses to serve plaintext HTTP at all
unless you explicitly opt in, and warns when that plaintext listener isn't on
loopback), configure a strong admin
password or wire up SSO/LDAP/SAML, use the Postgres standard profile with a
network-restricted database, and set `CRESTERAWATCH_TRUSTED_PROXIES` if you run
behind a reverse proxy or load balancer (otherwise `X-Forwarded-For` is
ignored for login-lockout purposes, by design). Dependencies are checked with
`govulncheck` on every change and again every week, so a vulnerability published
against a dependency is caught even when nothing has been committed, and the
full source history is scanned for committed secrets.
