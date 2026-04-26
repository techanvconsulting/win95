+++
author = "Anubhav Gain"
title = "OWASP Top 10 2024: Every Vulnerability Explained and How to Fix It"
date = "2026-04-15"
description = "Broken access control is still #1. Here is every OWASP Top 10 category, why it exists, and the concrete fix."
tags = ["owasp", "security", "web", "appsec"]
categories = ["security", "appsec"]
+++

The OWASP Top 10 is not a trivia list — it is a ranked index of the vulnerability classes that appear in real breach reports, CVEs, and pen test findings year after year. The 2024 update reflects a web landscape shaped by APIs, cloud-native workloads, and AI-integrated applications. Each category below covers what it is, how it gets exploited, and the specific fix.

<!--more-->

## A01 — Broken Access Control

**What it is.** The application enforces business rules in the UI but not at the API layer, so an attacker who calls the endpoint directly bypasses them entirely. IDOR (insecure direct object reference) is the canonical form: changing `GET /orders/1042` to `GET /orders/1041` returns another user's order.

**Real exploit.** An attacker enumerates sequential numeric IDs in a `/api/users/{id}` endpoint. The frontend only renders the authenticated user's profile, but the backend performs no ownership check — every account is exposed.

**Fix.** Enforce authorization server-side on every route, not just the UI. Check resource ownership before returning data:

```python
# Bad
order = Order.get(order_id)

# Good
order = Order.get(order_id)
if order.user_id != current_user.id:
    raise Forbidden()
```

Use UUIDs instead of sequential integers to remove enumeration as a trivial attack vector.

---

## A02 — Cryptographic Failures

**What it is.** Sensitive data transmitted or stored without adequate encryption — or encrypted with weak, deprecated algorithms (MD5, SHA-1, DES, RC4).

**Real exploit.** A database backup is stored in S3 with public-read access. Passwords are hashed with unsalted MD5. An attacker downloads the backup, rainbow-tables the hashes, and recovers plaintext credentials within minutes.

**Fix.** Hash passwords with bcrypt, scrypt, or Argon2 (never MD5/SHA-1). Enforce TLS 1.2+ and reject weak cipher suites. Flag sensitive fields for encryption at rest:

```yaml
# Nginx: enforce TLS 1.2+, strong ciphers only
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers on;
```

---

## A03 — Injection

**What it is.** Untrusted input is interpreted as code: SQL, OS commands, LDAP queries, NoSQL operators. The application does not separate data from instructions.

**Real exploit.** `GET /search?q='; DROP TABLE users; --` — a raw SQL query assembled via string concatenation. One request drops a production table.

**Fix.** Use parameterized queries or prepared statements without exception:

```python
# Bad
cursor.execute(f"SELECT * FROM users WHERE name = '{name}'")

# Good
cursor.execute("SELECT * FROM users WHERE name = %s", (name,))
```

For OS commands, use allow-lists and avoid shell=True in subprocess calls.

---

## A04 — Insecure Design

**What it is.** Security was never part of the design, not just implemented incorrectly. Missing rate limits, absent anti-automation controls, business logic that can be abused by design.

**Real exploit.** A password reset flow sends a 6-digit code with no rate limit and a 24-hour expiry. An attacker brute-forces all 1,000,000 combinations before the code expires and takes over the account.

**Fix.** Apply threat modeling before building features. For the above: exponential backoff, code expiry under 15 minutes, lockout after 5 failed attempts. Bake these into the design spec, not the bug backlog.

---

## A05 — Security Misconfiguration

**What it is.** Default credentials, open admin interfaces, verbose error messages, unnecessary services enabled, cloud storage buckets with public access.

**Real exploit.** A Kubernetes dashboard deployed with default settings is exposed on a public load balancer. No authentication required. An attacker gains cluster-admin access within seconds.

**Fix.** Automate baseline hardening: disable default accounts, remove unused services, suppress stack traces in production responses. Infrastructure as Code makes this auditable:

```hcl
# Terraform: block public access to S3 buckets
resource "aws_s3_bucket_public_access_block" "default" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## A06 — Vulnerable and Outdated Components

**What it is.** Dependencies with known CVEs remain in production because no one is tracking them. This includes libraries, frameworks, container base images, and OS packages.

**Real exploit.** Log4Shell (CVE-2021-44228). A single outdated Log4j dependency, present in thousands of applications as a transitive dependency, enabled unauthenticated RCE at massive scale.

**Fix.** Run software composition analysis (SCA) in CI — Dependabot, Renovate, or Snyk. Fail builds on critical/high CVEs in direct dependencies. Pin container base images to specific digests, not floating tags.

---

## A07 — Identification and Authentication Failures

**What it is.** Weak credential policies, missing MFA, session tokens that do not expire, credential stuffing with no rate limiting.

**Real exploit.** An attacker runs a credential stuffing campaign using a breach dataset of 100 million email/password pairs. No MFA, no rate limiting. Account takeover rate: 0.5% = 500,000 accounts.

**Fix.** Enforce MFA for all accounts, especially privileged ones. Invalidate sessions on logout and password change. Implement lockout and rate limiting on auth endpoints:

```python
@limiter.limit("5 per minute")
@app.route("/login", methods=["POST"])
def login():
    ...
```

Use short-lived JWTs (15–60 minutes) with refresh token rotation.

---

## A08 — Software and Data Integrity Failures

**What it is.** The application trusts unsigned code, unverified updates, or deserializes objects from untrusted sources without validation. CI/CD pipelines that pull dependencies without integrity checks fall here.

**Real exploit.** SolarWinds. A build pipeline was compromised; a malicious DLL was signed with the vendor's key and shipped to 18,000 customers as a legitimate update.

**Fix.** Sign artifacts and verify signatures before execution. Use Sigstore/cosign for container images. Pin dependencies to specific commit SHAs or hashes. Enforce code review and branch protection on CI/CD configuration files — those files are now part of your attack surface.

---

## A09 — Security Logging and Monitoring Failures

**What it is.** The application generates no audit trail, or logs exist but are never ingested, correlated, or acted on. Attackers dwell undetected for weeks or months.

**Real exploit.** The average attacker dwell time before detection is still measured in weeks. In many breaches, the compromise was discovered by a third party (law enforcement, a customer complaint) rather than internal monitoring.

**Fix.** Log authentication events (success, failure, MFA bypass), privilege escalation, and access to sensitive resources. Ship logs to a centralized SIEM with retention. Set alerting thresholds — 50 failed logins in 60 seconds should trigger immediate investigation, not a weekly report.

---

## A10 — Server-Side Request Forgery (SSRF)

**What it is.** The server fetches a URL supplied by user input. An attacker points it at internal services — the EC2 metadata endpoint, internal APIs, or services behind the firewall.

**Real exploit.** Capital One breach (2019). SSRF against the AWS EC2 metadata service (`169.254.169.254`) exposed an IAM role credential. That credential was used to exfiltrate over 100 million records from S3.

**Fix.** Validate and allowlist URLs before the server fetches them. Block internal IP ranges and cloud metadata endpoints at the network level:

```python
import ipaddress

BLOCKED = [ipaddress.ip_network("169.254.0.0/16"),
           ipaddress.ip_network("10.0.0.0/8"),
           ipaddress.ip_network("172.16.0.0/12"),
           ipaddress.ip_network("192.168.0.0/16")]

def is_safe_url(url):
    host = urlparse(url).hostname
    addr = ipaddress.ip_address(socket.gethostbyname(host))
    return not any(addr in net for net in BLOCKED)
```

---

## How OpenArmor Detects These at Runtime

Static analysis and hardening guides catch known patterns at build time. Runtime is where the rest lives.

OpenArmor monitors your running environment for behavioral indicators that map directly to the OWASP Top 10: anomalous access patterns and IDOR sequences across API endpoints (A01), data exfiltration signals and unencrypted channel usage (A02), injection payloads in request bodies and headers (A03), component vulnerability correlations against live CVE feeds (A06), auth failure spikes and session anomalies (A07), and SSRF-pattern outbound requests from application hosts (A10).

Because OpenArmor is fully open-source, every detection rule is inspectable and tunable — no black-box scoring, no vendor lock-in. When a rule fires, you can read exactly why.

The OWASP Top 10 does not change because defenders are incompetent. It persists because the same architectural shortcuts get made under schedule pressure, in every stack, every cycle. The fix is systematic: automated scanning in CI, enforce-at-code-level controls, runtime detection. Each layer closes the gap the others miss.

**OpenArmor provides open-source runtime security tooling for every layer of this stack. Explore the project at [techanv.com](https://techanv.com) or on GitHub.**
