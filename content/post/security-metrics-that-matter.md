+++
author = "Anubhav Gain"
title = "Security Metrics That Actually Matter (And the Ones That Don't)"
date = "2026-04-09"
description = "CVE count tells you nothing. Patch rate tells you little. Here are the metrics that actually measure your security posture."
tags = ["security", "metrics", "soc", "risk"]
categories = ["security", "engineering"]
+++

Most security dashboards are full of numbers that feel impressive and tell you almost nothing. CVEs discovered: 847. Patches applied: 612. Vulnerabilities open: 1,203. The board nods, the CISO moves to the next slide, and nobody is any clearer on whether a breach is more or less likely than six months ago. That is the vanity metrics trap — and most security programs are caught in it.

<!--more-->

## The Vanity Metrics Trap

CVE count is the worst offender. High can mean mature scanning. Low can mean barely scanning. The raw number carries no signal about exploitability, criticality, or remediation velocity — yet it anchors board conversations constantly.

Patch rate has the same problem. 90% patched sounds strong until the remaining 10% includes your internet-facing VPN concentrator. Aggregate rates obscure the distribution that actually matters.

Vulnerability count over time is meaningless without context: rising because of new infrastructure, better scanner coverage, or more public exploits? The trend tells you nothing without a denominator.

Easy to collect, easy to misinterpret. Stop centering your program around them.

## Signal Metrics: MTTD and MTTR

If you only instrument two things, instrument these.

**Mean Time to Detect (MTTD)** is how long an attacker lives in your environment before you know they are there. Industry average is around 194 days. A well-instrumented SOC should target under one hour for covered attack patterns and under 24 hours overall. Measure it precisely: estimated incident start time from log forensics against first credible detection alert, averaged across confirmed incidents per quarter.

**Mean Time to Respond (MTTR)** is the gap from first alert to confirmed containment. Under four hours for critical incidents is achievable with playbook automation. Without it, most teams land at 12–48 hours.

Both require tracking confirmed incidents, not alert volume — which means a reliable incident classification process comes first.

## Coverage Metrics: What Can You Actually See?

Visibility gaps are where attackers hide.

**EDR agent coverage** is the percentage of managed endpoints with an active detection agent. If 15% of your Windows fleet has no EDR, that 15% is invisible. Target 95%+ for managed assets.

**Log centralization rate** is the fraction of infrastructure whose logs are actually flowing into your SIEM. A firewall logging to nowhere is a gap an attacker will find before you do.

**Detection rule coverage vs. MITRE ATT&CK** is the most strategic metric here. Map active rules to ATT&CK tactics and techniques. Most organizations are dense in initial access, sparse in privilege escalation and lateral movement — exactly where breaches compound. A quarterly ATT&CK gap analysis forces an honest conversation about where detection investment needs to go.

## Risk Reduction Metrics

These answer the question the board should actually be asking: are we reducing breach probability and impact?

**Critical vulns remediated within SLA** measures whether you are closing highest-risk windows on time. Define SLAs by severity: CVSS 9.0+ with a public exploit gets 48 hours; CVSS 7.0–8.9 on internet-facing assets, 7 days. Below 60% SLA adherence is a systemic process failure.

**Exploitable attack surface area** counts internet-exposed services directly exploitable with known techniques — unpatched CVEs with public PoC, exposed admin interfaces, credentials in breach databases. This is the number attackers are working from. A 20% quarter-over-quarter reduction has no equivalent in raw CVE counts.

## Business Metrics

Security exists in an economic context. These three translate posture into language finance and the board can act on.

**Cost per incident** divides total IR cost (analyst hours, tooling, legal, remediation) by confirmed incident count per year. It shows whether investment in detection and automation is actually reducing breach cost over time.

**Security debt ratio** compares known accepted risks against total assessed risk. A rising ratio means the program accepts more risk than it closes — and justifies remediation headcount with data, not fear.

**Compliance drift rate** tracks controls that passed at last audit but have since degraded. Weekly automated checks against stored baselines surface regressions before the auditors do.

## Building a Metrics Dashboard That Works

Instrument at the source: SIEM for detection timing, EDR for agent coverage, scanner APIs for SLA data, ticketing for remediation velocity. Avoid manual exports — anything requiring human intervention will be stale before it is read.

Cadence matters: detection metrics weekly (SOC lead), coverage and risk metrics monthly (engineering team), business metrics quarterly (executives). Structure the dashboard in three tiers — operational (real-time), tactical (weekly trending), strategic (business impact) — and keep them separate. Mixing all three into one report ensures none gets the attention it deserves.

## OpenArmor's Built-In Metrics and Reporting

OpenArmor ships MTTD and MTTR tracking tied to alert lifecycle events — every timestamp from rule trigger to containment captured automatically. Coverage dashboards surface EDR gaps and ATT&CK coverage in real time. SLA-based vulnerability tracking maps scanner findings to asset criticality without manual tagging; compliance drift reports surface regressions on a configurable schedule.

Because it is open-source, metric definitions are not black boxes. Inspect how MTTD is calculated, tune the incident start-time heuristic, extend with custom metrics. No vendor lock-in on the numbers that define your program's health.

---

The difference between a mature program and a performative one is whether the metrics you report predict exposure and drive prioritization. CVE counts fill slides. MTTD, coverage gaps, exploitable surface, and SLA compliance drive decisions. Build your dashboard around the second list.

**OpenArmor ships the instrumentation layer out of the box. Explore at [techanv.com](https://techanv.com) or on GitHub.**
