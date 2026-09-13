# Security & trust package

The documents a prospective customer's security or third-party-risk team
will ask for during due diligence. Hand these over when evaluating Crestera Watch
for a production rollout.

| Document | For whom | What it is |
|---|---|---|
| [security-overview.md](security-overview.md) | Executives, buyers, examiners | Plain-English + technical summary of how Crestera Watch protects data, and the shared-responsibility model. **Start here.** |
| [vendor-security-questionnaire.md](vendor-security-questionnaire.md) | TPRM / security analysts | SIG-Lite / CAIQ-aligned self-assessment answering the standard vendor questionnaire. |
| [../../SECURITY.md](../../SECURITY.md) | Everyone | Vulnerability-disclosure policy and secure-deployment pointer. |
| [hardening-review-2026-08.md](hardening-review-2026-08.md) | Security analysts | Record of the August 2026 **internal** security review: what was checked, what was fixed, what's still open. An in-house review, **not** a third-party test. |
| [pentest-engagement.md](pentest-engagement.md) | Us, and the testing firm | Scope, system description, rules of engagement and pre-flight checklist for an independent penetration test. **The test has not been performed** — this is the packet that sets one up. |

## The short version

- **Self-hosted** — you run it in your own environment; the vendor never
  receives your data, and there is **no telemetry, analytics, or
  phone-home mechanism anywhere in the product**.
- **Secure by default** — refuses to serve plaintext HTTP, PBKDF2-SHA256 at
  OWASP's current iteration count, TOTP MFA (Free) and OIDC/SAML/LDAP SSO
  (Pro), server-side RBAC, a strict per-request-nonce CSP, and real Postgres
  row-level-security tenant isolation on the multi-tenant profile.
- **Continuously tested** — 589 automated test functions, CI-gated build +
  vet + race-detector test suite + dependency vulnerability scan on every
  change, **zero known reachable vulnerabilities** in production
  dependencies.
- **Shared responsibility** — the vendor secures the code; you secure the
  environment. See the table in the overview.

## Stated plainly, not glossed over

One gap a reviewer will ask about and should hear about directly, not
discover on their own:

- **No third-party penetration test or SOC 2/ISO certification yet.** The
  [August 2026 hardening review](hardening-review-2026-08.md) is an
  **internal** self-review by the development team, published for
  transparency about what's tested in-house. It is not a substitute for
  either of the above and isn't presented as one. The
  [pentest engagement package](pentest-engagement.md) is scoped and ready to
  book — what's outstanding is the engagement itself.
