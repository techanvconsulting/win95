+++
author = "Anubhav Gain"
title = "Security Automation with Python: Scripts Every Security Engineer Should Have"
date = "2026-03-30"
description = "Manual security tasks are toil. Python automates asset discovery, vulnerability scanning, log analysis, and threat intel enrichment. Here are the building blocks."
tags = ["python", "automation", "security", "scripting", "devops"]
categories = ["security", "engineering"]
+++

Security engineering has a toil problem. The average security team spends a significant portion of every week on tasks that are repetitive, deterministic, and well within the capability of a fifty-line Python script: pulling CVE details, grepping through auth logs, checking IP reputation, rotating through asset inventories. These are not judgment calls — they are mechanical tasks dressed up as engineering work. Python does not replace judgment, but it eliminates the toil that consumes the time you need for it.

<!--more-->

## Why Python for Security Automation

The security tooling ecosystem gravitates toward Python for good reason. The standard library covers most networking and file I/O primitives. The third-party ecosystem fills in everything else. `requests` handles REST APIs for every major threat intelligence platform. `scapy` gives you packet construction and inspection at the level of raw bytes. `paramiko` wraps SSH for fleet-level automation. `shodan` and `pymisp` provide typed clients for their respective platforms. `subprocess` bridges to system tools like `iptables`, `nmap`, and `openssl` when you need them. No other language offers the same combination of readable syntax, broad library support, and tight integration with the Linux system layer that security work requires.

The scripts below are building blocks, not production code. They demonstrate the pattern — API call, parse, act — that scales into a full pipeline.

## Asset Discovery: Fast Port Scanner

Before you can protect your perimeter, you need to know what is on it. This scanner uses `concurrent.futures` to parallelize socket probes across a host range without the overhead of a subprocess call to nmap for every quick check.

```python
import socket
from concurrent.futures import ThreadPoolExecutor, as_completed

COMMON_PORTS = [22, 80, 443, 3306, 5432, 6379, 8080, 8443, 27017]

def probe(host, port, timeout=0.5):
    try:
        with socket.create_connection((host, port), timeout=timeout):
            return (host, port, True)
    except (socket.timeout, ConnectionRefusedError, OSError):
        return (host, port, False)

def scan(hosts, ports=COMMON_PORTS, workers=100):
    results = []
    tasks = [(h, p) for h in hosts for p in ports]
    with ThreadPoolExecutor(max_workers=workers) as ex:
        futures = {ex.submit(probe, h, p): (h, p) for h, p in tasks}
        for f in as_completed(futures):
            host, port, open_ = f.result()
            if open_:
                results.append((host, port))
                print(f"[+] {host}:{port} open")
    return results

if __name__ == "__main__":
    targets = [f"10.0.0.{i}" for i in range(1, 255)]
    scan(targets)
```

Run this against your internal ranges during asset inventory cycles or after a cloud provisioning event. Open ports that should not exist are a finding before nmap ever runs.

## Vulnerability Enrichment: NVD API CVSS Lookup

Your scanner dumps a list of CVEs. Your downstream consumers need CVSS scores and severity ratings before they can prioritize. The NVD REST API returns structured data for any CVE identifier.

```python
import requests

NVD_API = "https://services.nvd.nist.gov/rest/json/cves/2.0"

def get_cvss(cve_id: str) -> dict:
    r = requests.get(NVD_API, params={"cveId": cve_id}, timeout=10)
    r.raise_for_status()
    vulns = r.json().get("vulnerabilities", [])
    if not vulns:
        return {}
    metrics = vulns[0]["cve"].get("metrics", {})
    cvss_v3 = metrics.get("cvssMetricV31", metrics.get("cvssMetricV30", []))
    if cvss_v3:
        data = cvss_v3[0]["cvssData"]
        return {"score": data["baseScore"], "severity": data["baseSeverity"]}
    return {}

if __name__ == "__main__":
    cve_list = ["CVE-2021-44228", "CVE-2023-44487", "CVE-2024-3094"]
    for cve in cve_list:
        result = get_cvss(cve)
        print(f"{cve}: {result.get('score', 'N/A')} ({result.get('severity', 'unknown')})")
```

Feed this a list from your scanner output and write the enriched results to a structured report. A CVSS 9.8 CRITICAL finding with a public exploit gets a different response timeline than a 4.3 MEDIUM with no known weaponization.

## Log Parsing: SSH Brute-Force Detection and Auto-Block

Auth logs record every failed authentication attempt. Left unread, they are noise. Parsed with Python, they become a detection and response pipeline. This script reads `/var/log/auth.log`, counts failed SSH attempts per IP, and calls `iptables` to block repeat offenders.

```python
import re
import subprocess
from collections import Counter

LOG_FILE = "/var/log/auth.log"
THRESHOLD = 10

def parse_failed_ssh(path=LOG_FILE):
    pattern = re.compile(r"Failed password for .+ from (\d+\.\d+\.\d+\.\d+)")
    ips = []
    with open(path) as f:
        for line in f:
            m = pattern.search(line)
            if m:
                ips.append(m.group(1))
    return Counter(ips)

def block_ip(ip: str):
    cmd = ["iptables", "-I", "INPUT", "-s", ip, "-j", "DROP"]
    subprocess.run(cmd, check=True)
    print(f"[!] Blocked {ip}")

if __name__ == "__main__":
    counts = parse_failed_ssh()
    for ip, count in counts.items():
        if count >= THRESHOLD:
            block_ip(ip)
```

Run this as a cron job or as a systemd service watching the log file via `inotify`. In practice, pair it with a whitelist check before the `iptables` call so you do not auto-block your own monitoring infrastructure.

## Threat Intel Enrichment: VirusTotal IP and Hash Reputation

When an IP or file hash appears in an alert, the first question is always reputation. VirusTotal's API answers that in one request.

```python
import requests

VT_API_KEY = "YOUR_VT_API_KEY"
VT_URL = "https://www.virustotal.com/api/v3"

def vt_check(ioc: str, ioc_type: str = "ip_addresses") -> dict:
    headers = {"x-apikey": VT_API_KEY}
    url = f"{VT_URL}/{ioc_type}/{ioc}"
    r = requests.get(url, headers=headers, timeout=10)
    r.raise_for_status()
    stats = r.json()["data"]["attributes"]["last_analysis_stats"]
    return stats

if __name__ == "__main__":
    iocs = [("198.51.100.42", "ip_addresses"),
            ("44d88612fea8a8f36de82e1278abb02f", "files")]
    for ioc, kind in iocs:
        stats = vt_check(ioc, kind)
        malicious = stats.get("malicious", 0)
        print(f"{ioc}: {malicious} engines flagged malicious")
```

Replace `ip_addresses` with `domains` or `urls` for other IOC types. The `malicious` count gives you immediate triage signal — anything above zero on an IP your systems contacted is worth escalating.

## Alerting: Slack Webhook Notification

Detection without notification is a log entry no one reads. Every automated script should route its findings to where your team operates. Slack incoming webhooks are the lowest-friction path.

```python
import requests
import json

SLACK_WEBHOOK = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

def alert(message: str, severity: str = "medium"):
    color_map = {"critical": "#FF0000", "high": "#FF6600",
                 "medium": "#FFCC00", "low": "#36A64F"}
    payload = {
        "attachments": [{
            "color": color_map.get(severity, "#AAAAAA"),
            "text": message,
            "footer": "OpenArmor Security Automation"
        }]
    }
    requests.post(SLACK_WEBHOOK, data=json.dumps(payload), timeout=5)

if __name__ == "__main__":
    alert("SSH brute-force: 198.51.100.42 blocked after 15 failed attempts", "high")
```

Wrap this in a decorator or call it at the end of any detection function. The rule is simple: if a script takes automated action, a human gets notified.

## Orchestration: Chaining Into a Lightweight SOAR Pipeline

Individual scripts are useful. Chained together, they approximate a Security Orchestration, Automation, and Response pipeline without the six-figure platform cost.

A minimal pipeline for a new alert: enrich the source IP via VirusTotal, pull CVSS data for any CVEs referenced in the alert, check whether the IP appears in active asset inventory, block if reputation is confirmed malicious, and fire a Slack notification with all context attached. Each step is a function. The pipeline is a `main()` that calls them in sequence and passes a shared context dictionary between them. Add `argparse` for CLI invocation and a cron entry for scheduled runs, and you have a repeatable playbook.

The overhead is not the code. It is the discipline of treating each script as a module with a defined input contract and a defined output, so the next engineer can extend the pipeline without reading your mind.

## OpenArmor's Python SDK and Integration Hooks

OpenArmor exposes its platform capabilities through a Python SDK designed for exactly this kind of automation. The SDK provides typed clients for the asset inventory, finding ingestion, and playbook trigger APIs — so your custom scripts write their results directly into the platform rather than into flat files that get lost.

Custom integration hooks let you register a Python function against any OpenArmor event type: new finding, asset change, threshold breach. The hook receives a structured event object, runs your logic, and can create findings, update asset records, or trigger downstream playbooks through the same API. You bring the logic; the platform handles state management, deduplication, and audit trail.

The combination of these building-block scripts and OpenArmor's integration layer is what moves an organization from reactive security — humans reading logs after the fact — to automated detection and response where the toil is handled before an analyst ever opens a ticket.

The scripts in this post are starting points. The patterns they demonstrate — query an API, parse the response, take a conditional action, notify a human — compose into the detection and response workflows your team actually needs. Start with one script that solves a real problem you have this week. Chain the next one in next week. The pipeline builds itself.
