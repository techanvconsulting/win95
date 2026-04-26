+++
author = "Anubhav Gain"
title = "Penetration Testing Methodology: What Real Pentesters Actually Do"
date = "2026-03-26"
description = "Penetration testing is not running automated scanners and generating a PDF. Here is what a real engagement looks like — and how to prepare."
tags = ["pentesting", "security", "red-team", "methodology"]
categories = ["security", "research"]
+++

Most organizations have had a penetration test. Far fewer have had a good one. The difference is not the tools — it is the methodology. Here is what a structured engagement actually looks like, phase by phase, and what your team should do before a tester connects to your network.

<!--more-->

## Types of Tests: Black Box, Grey Box, White Box

The first decision in scoping a pentest is how much information the tester starts with.

**Black box:** No prior knowledge — no architecture diagrams, no credentials, no documentation. This most closely mimics an external attacker. The trade-off is efficiency: a significant portion of the engagement is spent on reconnaissance that your internal teams already know. Useful for validating what an unaided attacker can discover and exploit.

**Grey box:** Partial information — in-scope IP ranges, a low-privilege account, or a high-level architecture overview. The most common and often most valuable configuration. It skips reconnaissance dead ends and focuses effort on whether internal systems, authenticated surfaces, and trust relationships are exploitable.

**White box:** Full disclosure — source code, architecture diagrams, credentials, network maps. Not "easy mode" — it asks a different question: not "can an attacker find this?" but "does this vulnerability exist at all?" White box surfaces the highest findings density per hour, making it the right choice when thoroughness matters more than attacker simulation.

Use black box to measure external exposure. Grey box for internal posture. White box before a major release or compliance audit.

## Scope and Rules of Engagement: Define Everything Before Day One

Vague scope destroys pentests. Before the engagement begins, commit to writing: exact IP ranges and domains in scope; systems explicitly out of scope; testing hours and blackout windows; whether social engineering and phishing are permitted; who receives real-time notification if a critical finding is discovered mid-engagement; and the process for pausing if an actual incident occurs.

An emergency contact chain is not a formality. If a tester triggers an outage or discovers active attacker infrastructure already on the network, both sides need to know who calls whom in minutes, not hours.

## Phase 1 — Reconnaissance

Reconnaissance separates methodical testers from tool-runners. Before any active scanning, good pentesters invest heavily in passive information gathering.

OSINT tools include Shodan and Censys for discovering internet-exposed services — often ones the organization did not know were exposed. theHarvester aggregates email addresses, subdomains, and employee names from search engines and public data sources. LinkedIn maps organizational structure and surfaces names useful for credential attacks. DNS recon via Amass or dnsrecon surfaces subdomains, MX records, zone transfer misconfigurations, and historical DNS data.

Everything discovered here shapes every phase that follows.

## Phase 2 — Scanning

Active scanning begins once recon maps the target surface. Nmap drives host discovery and port enumeration — service version detection and script scanning identify what is running and flag misconfigurations. Nessus and OpenVAS cross-reference discovered services against CVE databases, generating a prioritized list of potentially vulnerable systems.

For web applications: Nikto checks HTTP services for known vulnerabilities and exposed files. Burp Suite handles manual and semi-automated testing — intercepting traffic, mapping application logic, and probing input fields for injection vulnerabilities. Automated scanning catches easy wins; manual testing finds logic flaws scanners miss.

## Phase 3 — Exploitation

This is where hypotheses become confirmed findings. CVE-based exploitation uses Metasploit, custom scripts, or public PoC code to validate that discovered vulnerabilities are actually exploitable — not just scanner-reported. Web application attacks include SQL injection, XSS, authentication bypass, insecure direct object references, and SSRF, tested against the application's specific logic. Credential attacks cover password spraying against exposed auth services, default credential testing on network devices, and validating whether leaked credentials from public breach data still work.

A scanner finding that cannot be exploited is a lead, not a vulnerability. Exploitation confirms it.

## Phase 4 — Post-Exploitation

Post-exploitation answers the most important question: what can an attacker do after gaining a foothold? This phase is included not as a guide to attack techniques but because understanding attacker behavior here is what drives effective defensive architecture.

Lateral movement explores whether access to one system reaches others — through credential reuse, Active Directory misconfigurations, over-permissioned service accounts, or trust relationships between segments. Privilege escalation tests whether a low-privilege foothold can reach administrator or root. Persistence evaluation examines whether an attacker could maintain access across reboots and credential rotations without triggering detection.

Understanding these patterns is how you design defenses that contain a breach, not just prevent initial access.

## Phase 5 — Reporting

A pentest report serves two audiences. The executive summary — one to three pages — explains business impact, overall risk posture, and the highest-priority findings in plain language. No acronyms. Designed to be read by a CISO or board member in ten minutes.

The technical findings section documents each vulnerability: steps to reproduce, evidence (screenshots, request/response pairs), and a CVSS score. Every finding must include remediation guidance specific enough to act on — not "patch the system" but which patch, and what compensating control applies if patching immediately is not feasible. Prioritize by exploitability and impact, not just CVSS score: a critical finding on an isolated internal host may matter less than a medium finding on an internet-facing authentication endpoint.

## How to Prepare as the Target

The quality of a pentest is partly determined by how prepared the target is. Before the engagement: ensure network and application architecture diagrams are current and shareable under the engagement agreement. Asset inventories should be accurate — undocumented systems within scope create delays and noise. Patch status documentation helps the tester focus on unknowns rather than re-discovering already-fixed issues. Designate a technical point of contact who can answer architecture questions quickly. The more context the tester has, the more efficiently they target actual risk instead of reverse-engineering your infrastructure.

## Continuous Testing vs Point-in-Time: Why Annual Is Not Enough

A once-a-year test is a snapshot: specific scope, specific day, specific tools. Three months later your infrastructure has changed, new vulnerabilities have been disclosed, and the report is already partially stale.

The argument for continuous testing is not that it finds more per dollar — it is that it produces an updated picture of risk rather than a compliance event that gets filed and ignored. Shorter, more frequent assessments with rotating scope; automated scanning integrated into CI/CD pipelines; and detection validation exercises that confirm your defenses actually fire — these replace the annual model with a feedback loop.

Annual pentests satisfy compliance requirements. Continuous testing improves security posture. Both have a role, but mistaking the former for the latter is how organizations pass audits and still get breached.

**OpenArmor provides open-source security infrastructure for teams running continuous detection and response programs. Learn more at [techanv.com](https://techanv.com).**
