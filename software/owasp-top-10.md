# OWASP Top 10 (2025)

Interview-prep cheat sheet. Source: https://owasp.org/Top10/2025/

| # | Vulnerability |
|---|---|
| [A01](#a01-broken-access-control) | Broken Access Control |
| [A02](#a02-security-misconfiguration) | Security Misconfiguration |
| [A03](#a03-software-supply-chain-failures) | Software Supply Chain Failures |
| [A04](#a04-cryptographic-failures) | Cryptographic Failures |
| [A05](#a05-injection) | Injection |
| [A06](#a06-insecure-design) | Insecure Design |
| [A07](#a07-authentication-failures) | Authentication Failures |
| [A08](#a08-software-or-data-integrity-failures) | Software or Data Integrity Failures |
| [A09](#a09-security-logging-and-alerting-failures) | Security Logging and Alerting Failures |
| [A10](#a10-mishandling-of-exceptional-conditions) | Mishandling of Exceptional Conditions |

---

## A01: Broken Access Control

Users can perform actions or view data outside their intended permissions — via parameter tampering, privilege escalation, forced browsing, or missing server-side checks. Ranked #1, present in ~100% of tested apps.

**Examples**
- Parameter tampering: `.../accountInfo?acct=notmyacct` — swapping an ID to view someone else's account.
- Forced browsing / URL guessing: hitting `.../admin_getappInfo` directly, hoping for no access check.
- Bypassing client-side-only restrictions with `curl` since the check never happens server-side.
- Privilege escalation, JWT tampering, CORS misconfiguration, missing checks on API methods (not just UI buttons).

**Prevention**
- Deny by default for anything non-public; enforce checks server-side, never trust the client.
- Enforce record ownership instead of unrestricted CRUD.
- Minimize/centralize CORS config instead of ad-hoc per-endpoint.
- Disable directory listing; remove backup/metadata files from web roots.
- Rate-limit APIs to blunt automated probing.
- Invalidate sessions server-side on logout; use short-lived JWTs + refresh tokens.
- Log and alert on access-control failures.
- Cover access control with automated tests, not just manual review.

## A02: Security Misconfiguration

Insecure defaults, unnecessary enabled features, unpatched systems, verbose errors, or missing security headers — anywhere from the app to the cloud platform underneath it.

**Examples**
- Leftover sample apps / default admin accounts with default credentials still active in production.
- Directory listing enabled, letting attackers enumerate files and decompile classes to find flaws.
- Verbose error pages exposing stack traces and library versions.
- Cloud storage buckets left at default "internet-accessible" permissions.

**Prevention**
- Automate a repeatable hardening process for every environment.
- Ship minimal platforms — no unused features, samples, or docs in production.
- Review/update configs (including cloud storage permissions) as part of patch management.
- Segment architecture so a misconfiguration in one component/tenant doesn't cascade.
- Send security headers (CSP, HSTS, etc.) by default.
- Automate configuration verification; manually audit at least annually if not.
- Prefer identity federation and short-lived credentials over static embedded secrets.

## A03: Software Supply Chain Failures

Vulnerabilities or malicious changes introduced through third-party dependencies, tools, or vendors the application relies on, rather than first-party code.

**Examples**
- SolarWinds (2019): malicious updates from a trusted vendor infected ~18,000 organizations.
- Bybit crypto theft (2025, $1.5B): wallet software compromised via a supply-chain attack that only activated under specific conditions.

**Prevention**
- Maintain a Software Bill of Materials (SBOM) covering direct and transitive dependencies.
- Continuously monitor CVE feeds for components in use.
- Pull components only from official, trusted sources over secure connections.
- Enforce rigorous change management across CI/CD, repos, and dev tooling.
- Require separation of duties — no single person can push straight to production.
- Roll out patches via staged deployment rather than all-at-once.
- Harden supply-chain systems themselves with MFA, access controls, and signed artifacts.

## A04: Cryptographic Failures

Sensitive data exposed due to missing encryption, weak/outdated algorithms, poor key management, or flawed crypto implementation (bad IVs, deprecated hashes, padding oracles).

**Examples**
- Attacker on insecure WiFi downgrades HTTPS to HTTP, steals session cookies, hijacks the session.
- A leaked password database using unsalted or weak hashes gets cracked via rainbow tables / GPU brute force — even salted hashes fall if the hash function itself is weak.

**Prevention**
- Classify data by sensitivity; know what actually needs protecting.
- Store sensitive keys in an HSM (hardware or cloud-based).
- Use well-vetted crypto library implementations, not homegrown crypto.
- Minimize what sensitive data you store; discard promptly or tokenize (PCI DSS style).
- Encrypt sensitive data at rest with strong, current algorithms and proper key management.
- Encrypt data in transit: TLS 1.2+, forward secrecy, HSTS to force HTTPS.
- Disable caching of sensitive responses at CDN/server/app layers.
- Avoid unencrypted protocols for sensitive data (plain FTP, SMTP, etc.).
- Hash passwords with strong adaptive salted algorithms (Argon2, scrypt, PBKDF2-HMAC-SHA-512).
- Use cryptographically secure, non-reused IVs; prefer authenticated encryption.
- Generate keys with proper CSPRNGs; avoid predictable seeding.
- Avoid deprecated primitives: MD5, SHA1, CBC mode, PKCS#1 v1.5.
- Have security specialists review crypto configuration.
- Start planning for post-quantum crypto migration (target: by end of 2030).

## A05: Injection

Untrusted input is passed to an interpreter (SQL, OS shell, LDAP, ORM/HQL, expression languages) and executed as part of a command instead of being treated as plain data.

**Examples**
- SQL injection: `"SELECT * FROM accounts WHERE custID='" + id + "'"` with input `' OR '1'='1` returns every row.
- ORM/HQL injection: same idea against Hibernate query strings — `' OR custID IS NOT NULL OR custID='` dumps all records.
- OS command injection: `"nslookup " + domain` with input `example.com; cat /etc/passwd` runs an arbitrary shell command.

**Prevention**
- Use safe/parameterized APIs, or an ORM, instead of building queries via string concatenation (note: stored procs that concatenate internally are still vulnerable).
- Apply positive (allow-list) server-side input validation — not sufficient alone, but a solid layer.
- Escape special characters for the target interpreter when dynamic queries can't be avoided (can't fully protect identifiers like table/column names).
- Bake SAST/DAST/IAST scanning into CI/CD to catch injection flaws before production.

## A06: Insecure Design

Missing or ineffective security controls baked into the architecture itself — a flaw in the design, not the implementation. Perfect coding can't fix a control that was never designed in.

**Examples**
- Security questions used for account recovery — multiple people often know the same answers, violating NIST 800-63b.
- A cinema booking system with no business-logic limits lets an attacker reserve hundreds of seats across every venue.
- An e-commerce site with no bot protection gets its high-value stock bought out entirely by scalper bots.

**Prevention**
- Build a secure development lifecycle with AppSec involvement on security/privacy-sensitive features.
- Maintain a library of vetted, reusable secure design patterns and components.
- Threat-model authentication, access control, and other critical/business-logic flows.
- Bake security requirements into user stories from the start.
- Validate input/business rules at every application tier, not just the edge.
- Write tests (unit/integration) that explicitly validate against the threat model.
- Define both use-cases and misuse-cases per tier.
- Segregate system/network layers by exposure level; enforce tenant isolation.

## A07: Authentication Failures

Weaknesses in login, credential storage, session, or MFA handling let an attacker be recognized as a legitimate user. Covers 36 CWEs; unchanged at #7 since 2021.

**Examples**
- Credential stuffing / password spraying with breached username-password lists, sometimes pattern-incrementing guesses (e.g. `Winter2025` → `Winter2026`); an app without defenses becomes a "password oracle."
- Legacy policies (forced rotation, complexity rules) push users toward weak, reused passwords instead of following NIST 800-63 + MFA.
- No session timeout / no Single Logout on SSO — a shared/public computer or closed tab leaves the session (and every federated app) open to the next person.

**Prevention**
- Enforce MFA everywhere feasible.
- Encourage password managers; don't block long/random passwords.
- Never ship default credentials.
- Check new passwords against top-10k weak-password and known-breach lists.
- Follow NIST 800-63b guidance; don't force periodic rotation without cause.
- Harden registration/recovery flows against account enumeration.
- Rate-limit/lock out failed logins, with logging and alerting.
- Use secure server-side session management with high-entropy random session IDs.
- Prefer established auth frameworks/libraries over rolling your own.
- Validate JWT claims fully — audience, issuer, scopes — not just the signature.

## A08: Software or Data Integrity Failures

Code, CI/CD pipelines, or serialized data are trusted without verifying they weren't tampered with — missing signature checks, unverified auto-updates, insecure deserialization.

**Examples**
- DNS mapped to an external support vendor inadvertently shares the main domain's auth cookies, enabling session hijacking.
- Router/IoT firmware updates ship unsigned, letting attackers push malicious firmware to every device.
- Developers pull packages from random websites instead of trusted package managers, risking injected malicious code.
- Deserializing untrusted user-controlled state (e.g. a Java object blob starting with `rO0` in base64) enables remote code execution.

**Prevention**
- Verify authenticity with digital signatures before trusting software/data.
- Pull dependencies only from trusted repositories (internal vetted mirrors for high-risk cases).
- Review code and config changes to catch malicious code before merge.
- Keep CI/CD pipelines segregated, properly configured, and access-controlled.
- Reject unsigned/unencrypted serialized data from untrusted clients; verify integrity before deserializing.

## A09: Security Logging and Alerting Failures

Without adequate logging, monitoring, and alerting, breaches go undetected and incident response is slow or nonexistent.

**Examples**
- A health plan provider's breach of 3.5M children's records went undetected for over 7 years due to no monitoring.
- An Indian airline's decade of passenger data was breached at a third-party cloud provider — discovered only when the provider eventually reported it.

**Prevention**
- Log all security-relevant events consistently (logins, failed attempts, high-value transactions).
- Emit logs in formats compatible with standard log-management tooling.
- Encode log data properly to prevent log-injection attacks.
- Use append-only/integrity-protected audit trails.
- Define real monitoring/alerting use cases with response playbooks, not just raw log collection.
- Back up logs and protect them from tampering or deletion.
- Have an incident response plan aligned with a standard (e.g. NIST 800-61).

## A10: Mishandling of Exceptional Conditions

Applications fail to prevent, detect, or properly respond to unusual/error conditions — missing validation, poor exception handling, uncontrolled failure states. "Any time an app is unsure of its next instruction, an exceptional condition has been mishandled."

**Examples**
- A file-upload handler catches the exception but never releases the lock it held, eventually exhausting resources (DoS).
- Raw database errors shown to users leak schema/internal details, helping attackers craft better injection payloads.

**Prevention**
- Catch exceptions at their source and handle them meaningfully, not just swallow them.
- Fail closed: roll back incomplete transactions entirely rather than partial recovery.
- Centralize error handling for consistent behavior app-wide.
- Add rate limiting, resource quotas, and throttling to blunt resource-exhaustion attacks.
- Log and alert on repeated errors — they often indicate an attack in progress.
- Apply strict input validation plus a global exception handler as a last-resort fallback.
- Include failure-mode threat modeling in design/security reviews.
