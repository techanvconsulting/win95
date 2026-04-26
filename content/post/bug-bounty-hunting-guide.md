+++
author = "Anubhav Gain"
title = "Bug Bounty Hunting: From Beginner to First Report"
date = "2026-03-29"
description = "HackerOne paid out $300M+ to researchers. Bug bounties are real income. Here is how to go from zero to first valid report."
tags = ["bug-bounty", "security", "pentesting", "hacking", "research"]
categories = ["security", "research"]
+++

HackerOne has paid out over $300 million to security researchers. Bug bounties are not a lottery — they are a skill-based income stream that rewards people who learn to look at software differently than the developers who built it.

<!--more-->

## What Bug Bounties Are

A bug bounty program is a formal agreement: find vulnerabilities within a defined scope, report them responsibly, get paid. Three platforms dominate the public market — **HackerOne** (largest network), **Bugcrowd** (active community, good for beginners), and **Intigriti** (European-focused, high payout signal).

The critical concept is **scope**. Every program defines what is in scope (specific domains, APIs, mobile apps) and what is out of scope (third-party infrastructure, employee endpoints). Testing out-of-scope assets is unauthorized access, not bug bounty hunting. Read the policy in full. Responsible disclosure means reporting what you find, waiting for a patch before going public, and never exfiltrating user data.

---

## Setting Up Your Lab

You do not need expensive tools. The professional stack for web bounty work is mostly free:

- **Burp Suite Community** — HTTP proxy, repeater, intruder. Intercept and replay every request your browser makes.
- **subfinder** — passive subdomain enumeration via certificate transparency logs and DNS datasets
- **httpx** — probe a host list for live HTTP/HTTPS services and status codes
- **nuclei** — template-based scanner with thousands of community-maintained checks
- **amass** — OSINT-driven attack surface mapping

```bash
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
```

Run everything from a dedicated VM. Never route bounty traffic through a shared network.

---

## Recon Methodology

Good recon separates researchers who find one bug a month from those who find ten. Run subfinder against the root domain, merge and deduplicate, then probe with httpx to see what is alive. Programs with wide scope (`*.company.com`) have forgotten subdomains running legacy software, staging environments, or admin panels with default credentials.

Port scan live hosts with `nmap -sV --open` to surface non-standard services. Fingerprint frameworks with Wappalyzer — a site running WordPress 5.2 is a different target than a modern Node.js API. Check for `.git` directories, old JavaScript bundles referencing internal API URLs, and robots.txt entries pointing at interesting paths. Forgotten assets are where the real money is.

---

## High-Value Target Classes

Focus on vulnerability classes that consistently generate medium-to-critical payouts:

- **IDOR** — change a numeric ID in a request to access another user's data. Test every endpoint that references an ID.
- **Auth bypasses** — password reset flows with predictable tokens, JWT `alg: none` confusion, OAuth state mishandling.
- **SSRF** — any endpoint that fetches a URL you supply. Point it at `http://169.254.169.254/latest/meta-data/` on AWS to check for credential exposure.
- **XSS in unusual contexts** — stored XSS inside PDF generators, email templates, SVG upload handlers, and Markdown renderers. Scanners miss these.
- **Business logic flaws** — negative quantities in carts, discount codes applied multiple times, race conditions in transfer endpoints. These require understanding what the app is supposed to do.

---

## Writing a Good Report

A valid bug with a bad report gets downgraded or ignored. Triagers read dozens of reports a day — make their job easy:

1. **Title** — specific. "IDOR on /api/v1/orders allows reading any user's order history" beats "Broken access control."
2. **CVSS score** — calculate it. It shows you understand impact.
3. **Reproduction steps** — numbered, precise, include the exact curl command from Burp's "Copy as curl."
4. **Impact statement** — what does an attacker gain? "Read the full order history and shipping addresses of any user" is impact. "This is a security issue" is not.
5. **POC** — for XSS: screenshot of `alert(document.domain)` in the target origin. For IDOR: two accounts, one request, proof that account A's data returns to account B's session.

---

## Common Beginner Mistakes

**Duplicates.** The most common outcome. Check disclosed reports before submitting — if the endpoint looks interesting, it has probably been tested.

**Out-of-scope reports.** Re-read the scope before every submission. Third-party login widgets and CDN infrastructure are almost always out of scope.

**Low-quality POCs.** Self-XSS that fires only in your own session or CSRF on logout are informational at best. Demonstrate actual impact.

**Rate limit reports without impact.** "No rate limit on login" is valid only if you demonstrate user enumeration or a working brute-force attack.

---

## Moving Up

After your first valid report, go deep rather than wide. Researchers earning serious money specialize in one vulnerability class — SSRF, auth logic, mobile API security — well enough to find variants that scanners miss. Build a personal methodology document: every time you find something, write the exact steps that led you there.

Network actively. Read disclosed reports on HackerOne's Hacktivity feed and follow active researchers on X. The community shares techniques and program recommendations openly.

---

## TechAnv's Perspective: Bug Bounty as Security Hygiene

At TechAnv and OpenArmor, the most realistic threat model includes skilled external researchers — because that is who real attackers are. A well-run bug bounty program is the highest-ROI security investment most organizations can make: no false positives, no retainer, paid only on real findings.

For teams running OpenArmor, a bug bounty program complements runtime detection. The program finds what static analysis missed; OpenArmor catches what slips into production anyway.

The first report is the hardest. Pick a program with wide scope and a history of paying out medium-severity findings, read ten disclosed reports to understand what they value, and spend two hours on recon before touching a single endpoint.

**OpenArmor provides open-source security tooling for teams building serious programs. Explore the project at [techanv.com](https://techanv.com) or on GitHub.**
