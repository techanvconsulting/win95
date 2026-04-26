+++
author = "Anubhav Gain"
title = "Building a Security Operations Center on a Budget with Open-Source Tools"
date = "2026-04-11"
description = "Enterprise SOCs cost millions. A functional open-source SOC costs less than $500/month in infrastructure. Here is the full stack."
tags = ["soc", "security", "open-source", "wazuh", "siem"]
categories = ["security", "open-source"]
+++

Enterprise Security Operations Centers carry eye-watering price tags — Splunk licensing alone can run six figures annually before you hire a single analyst. But the core functions of a SOC are not proprietary: visibility, detection, response, and compliance are engineering problems, and the open-source ecosystem has solved most of them. A hardened, production-grade SOC running on commodity cloud infrastructure costs under $500 per month. Here is exactly how to build it.

<!--more-->

## What a SOC Actually Needs

Strip away the vendor marketing and a SOC has four jobs. **Visibility** means you know what is happening across every host, network segment, and cloud account — no blind spots. **Detection** means you have rules and models that flag anomalous or malicious activity with enough fidelity that analysts are not drowning in noise. **Response** means there is a structured workflow to investigate alerts, contain threats, and document actions taken. **Compliance** means you can produce audit evidence for PCI-DSS, SOC 2, ISO 27001, or whatever framework your customers or auditors demand.

Every commercial SOC platform sells you these four capabilities bundled together. Open source gives you best-of-breed components for each — at the cost of integration work you do once.

## The Open-Source SOC Stack

Five tools cover the full surface area:

- **Wazuh** serves as your SIEM and host-based intrusion detection system. It ingests agent telemetry from every endpoint, correlates events, and ships built-in rules for common attack patterns. It replaces OSSEC and natively integrates with the ELK stack for search and dashboarding.
- **Suricata** handles network intrusion detection. Deploy it on a span port or tap, point it at the Emerging Threats ruleset, and it generates structured alerts for known malware C2, exploitation attempts, and protocol anomalies.
- **TheHive** is your case management layer. When Wazuh or Suricata fires a high-confidence alert, TheHive creates a structured case, tracks analyst actions, records timelines, and gives you the documentation trail compliance requires.
- **Cortex** plugs into TheHive and automates enrichment — VirusTotal lookups, IP geolocation, passive DNS, WHOIS, and dozens of other analyzers run automatically the moment an observable lands in a case.
- **MISP** (Malware Information Sharing Platform) is your threat intelligence backbone. It ingests commercial and community feeds, de-duplicates indicators, and pushes fresh IOCs back into Wazuh and Suricata detection rules automatically.

Total infrastructure cost for this stack on a three-node cloud deployment: roughly $350–450 per month depending on your log volume and cloud provider.

## Log Collection Architecture

Wazuh agents run on every Linux, Windows, and macOS endpoint and forward normalized events over an encrypted channel to the Wazuh manager. For infrastructure and network devices that cannot run an agent, configure syslog forwarding to a dedicated log collector — rsyslog or Logstash both work. Cloud environments require specific connectors: AWS CloudTrail and GuardDuty pipe into Wazuh via its AWS module; GCP and Azure both have native integrations. Set retention at 90 days hot, 12 months cold in S3-compatible object storage. This satisfies most regulatory retention requirements and keeps query costs predictable.

## Detection Engineering

Raw rules are a starting point, not a destination. Tuning is the actual work. Start with Wazuh's default ruleset enabled, run for two weeks, and build a baseline of what normal looks like in your environment. Then disable or threshold the rules that produce the most noise without catching real threats — brute-force rules that fire on automated configuration management tools are a common culprit.

Write new detection logic in **Sigma format** — the vendor-neutral rule language that compiles to Wazuh, Splunk, Elastic, and every other backend. Sigma rules live in your git repository, get reviewed in pull requests, and can be tested against historical log data before they hit production. This is detection-as-code, and it is the practice that separates mature SOC programs from reactive ones. SigmaHQ's community repository contains thousands of rules mapped to MITRE ATT&CK that you can import and adapt immediately.

## Alerting and Escalation

Wazuh fires webhooks on rule matches. Route those webhooks into PagerDuty or OpsGenie to drive on-call rotation. Configure severity thresholds carefully: critical alerts (confirmed credential theft, active lateral movement) page the on-call analyst immediately; high alerts queue for same-day review; medium and below feed a daily digest. A simple three-tier model prevents alert fatigue from destroying analyst effectiveness. Document your escalation runbooks in TheHive as response templates so that any analyst — including one who just joined the team — can follow the correct containment steps at 2 AM without improvising.

## Compliance Reporting

Wazuh ships built-in compliance dashboards for PCI-DSS, HIPAA, GDPR, NIST 800-53, and TSC controls that map directly to SOC 2. The dashboards pull from the same events your analysts already review — no separate compliance data collection pipeline. For ISO 27001 Annex A controls, Wazuh's policy monitoring module performs continuous configuration checks against CIS benchmarks and reports deviations. Export these reports on a scheduled basis, store them in an evidence repository, and your next audit becomes a documentation exercise rather than a scramble.

## Scaling the Open-Source SOC

Open source takes you far, but recognize the thresholds where commercial tooling earns its cost. At roughly 50GB of daily log ingest, Elasticsearch cluster management becomes a significant operational burden — managed OpenSearch or a commercial SIEM backend starts making sense. When your analyst team grows past five people, a dedicated SOAR platform like Shuffle (open source) or commercial alternatives streamlines case automation at a level TheHive alone cannot match. And if your threat model includes nation-state adversaries, commercial threat intelligence feeds provide attribution and TTPs that community feeds lag by days or weeks.

The decision is not open source versus commercial. It is knowing which gaps commercial tooling fills after you have maximized what open source can do.

## OpenArmor as the AI Layer

The stack above gives you data, rules, and workflows. What it does not give you is a system that learns from your environment, surfaces the alerts most likely to be real, and explains its reasoning. That is the problem [OpenArmor](https://techanv.com) solves. OpenArmor sits as an AI layer on top of Wazuh, TheHive, and MISP — correlating signals across your entire telemetry pipeline, ranking alerts by confidence, and generating plain-language investigation summaries that cut analyst triage time from minutes to seconds. It does not replace the open-source stack. It makes every component in the stack dramatically more effective, without requiring you to swap tools or re-architect what is already working.

A functional SOC is not a budget problem. It is an integration and engineering problem. The tools exist. The infrastructure is cheap. The only remaining question is who builds it.
