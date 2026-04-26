+++
author = "Anubhav Gain"
title = "Ransomware Defense Playbook: Prevention, Detection, and Recovery"
date = "2026-04-14"
description = "Ransomware payments hit $1.1B in 2023. Most attacks are preventable. Here is the full prevention-detection-recovery playbook."
tags = ["ransomware", "security", "incident-response", "backup"]
categories = ["security", "incident-response"]
+++

Ransomware operators collected over $1.1 billion in payments in 2023. The number is staggering — but the more important fact is that the vast majority of those attacks succeeded because of entirely preventable failures: unpatched systems, no MFA, exposed RDP, absent network segmentation. This is not a fatalism problem. It is an execution problem. Here is the full playbook.

<!--more-->

## How Modern Ransomware Actually Works

Understanding the kill chain matters because each stage is a control point. Modern ransomware attacks do not look like movies. They are patient, methodical, and increasingly operated by criminal groups with internal HR departments and customer support teams.

**Initial access** typically arrives via one of three vectors: phishing emails delivering malicious macros or credential harvesters; exposed Remote Desktop Protocol (RDP) on port 3389, brute-forced or accessed with purchased credentials from initial access brokers; or exploitation of unpatched public-facing vulnerabilities (Exchange, Citrix, Fortinet, and similar perimeter appliances dominate the CVE list).

**Lateral movement** follows. The attacker escalates privileges — often via Mimikatz credential dumping, pass-the-hash, or exploiting misconfigured Active Directory — then pivots across the network to identify high-value targets: domain controllers, backup servers, file shares.

**Exfiltration** is the step that gets overlooked. Before encrypting anything, attackers exfiltrate sensitive data to their infrastructure. This enables double extortion: pay the ransom to decrypt, or we publish your data. This is now standard practice.

**Encryption** is the final step — and the one that triggers the ransom note. By the time files start renaming to `.encrypted`, the attacker has likely had access for days or weeks.

## Prevention Layer

**Patch cadence.** Most ransomware entry points exploit CVEs with patches available weeks or months before the breach. Treat critical and high-severity patches on internet-facing systems as a 24-hour SLA, not a monthly patch Tuesday ritual.

**MFA everywhere.** No exceptions for email, VPN, RDP, admin consoles, or cloud management planes. Phishing-resistant MFA (FIDO2/hardware keys) for privileged accounts. TOTP is better than nothing; hardware keys are significantly harder to bypass.

**Disable or gate RDP.** If RDP is not required, disable it. If it is required, it must sit behind a VPN with MFA enforcement — never exposed to the public internet. Shodan scans routinely surface tens of thousands of exposed RDP endpoints. Yours should not be among them.

**Email filtering and sandboxing.** Block macro-enabled documents from external senders. Enable attachment sandboxing. Implement DMARC, DKIM, and SPF to reduce spoofed sender attacks. Train users — but do not rely on training alone. Filters catch what training misses.

## Detection Layer

Prevention is not perfect. Detection catches what slips through.

**Canary files.** Drop decoy files with distinctive names in high-value directories — a file called `DO_NOT_OPEN_CONFIDENTIAL_HR_DATA.xlsx` that is never legitimately accessed. Any access or modification triggers an alert. This is low-cost and high-signal. Canaries catch ransomware in the act before it completes.

**Honeypots and honeytokens.** Deploy fake credentials in the registry or a credential store. If those credentials are used, an attacker is in your network. Tools like Canarytokens make this straightforward to deploy.

**Behavioral detection.** Ransomware has a distinctive fingerprint: mass file read/write operations, rapid file renaming, and deletion of shadow copies (`vssadmin delete shadows`). Any EDR worth running should flag this automatically. OpenArmor behavioral detection rules specifically watch for VSS deletion commands and mass rename events as high-confidence ransomware indicators — these fire early enough in the encryption cycle to contain blast radius before full encryption completes.

**SIEM correlation.** Correlate authentication anomalies, off-hours access to sensitive shares, and lateral movement indicators (pass-the-hash, unusual SMB activity) into a coherent attack timeline. A detection that fires 48 hours into an intrusion is still a detection — it may save your backups.

## Containment: Segmentation and Isolation

A flat network is a gift to ransomware operators. Once inside, they can reach everything.

**Network segmentation** limits blast radius. Finance, HR, engineering, and production systems should live on separate VLANs with explicit firewall rules governing inter-segment traffic. A compromised workstation in HR cannot reach production databases in a segmented network.

**Automated host isolation.** When a ransomware signal fires — EDR detection, canary trigger, or SIEM alert — the response time to isolate the host is critical. Automate it. An OpenArmor playbook can push a host isolation rule to the firewall or trigger an EDR isolation API within seconds of alert firing, before a human analyst has opened the ticket. Manual response at 2 AM takes 30 minutes. Automated response takes 30 seconds.

## Backup Strategy: 3-2-1-1-0

The backup rule has evolved. The modern standard is **3-2-1-1-0**:

- **3** copies of data
- **2** different media types
- **1** offsite copy
- **1** air-gapped or immutable copy (ransomware cannot encrypt what it cannot reach or modify)
- **0** errors on verified restore tests

The immutable copy is the critical addition. Cloud object storage with object lock enabled (AWS S3 Object Lock, Azure Immutable Blob Storage) prevents any process — including a ransomware operator with stolen admin credentials — from deleting or overwriting backup data within the retention window.

**Restore drills are not optional.** A backup that has never been restored is a theory. Run quarterly restore drills on a representative subset of data. Verify the restore works. Verify the restore time meets your RTO. Untested backups fail at the worst possible moment.

## Recovery: Decision Framework

When ransomware fires, you face a binary decision: **clean rebuild or restore from backup**.

Before either option, **preserve forensic evidence**. Snapshot affected systems before wiping. Capture memory where possible. The forensic record tells you what was exfiltrated, how the attacker got in, and what other systems may be compromised — information you need to prevent recurrence and answer legal disclosure obligations.

**Clean rebuild** is preferable when backups are old or untested, when the scope of compromise is unclear, or when you need certainty that no backdoor persists. It takes longer but provides a clean starting state.

**Restore from backup** is faster when you have recent, verified, immutable backups and high confidence in your ability to identify and close the initial access vector before restoration. Restoring into a still-compromised network defeats the purpose.

**Communications plan.** Define it before you need it. Who is the internal incident commander? Who speaks to the board? Who speaks to customers? Who coordinates with counsel on disclosure obligations? These decisions made under fire, without a plan, go badly.

## Should You Pay?

No.

Payment funds the next attack — yours or someone else's. Payment does not guarantee decryption; roughly 20% of organizations that pay do not fully recover their data. Payment signals that your organization pays, making you a higher-priority target for future attacks. Law enforcement and regulatory guidance increasingly discourages payment, and in some jurisdictions involving sanctioned threat actors, payment may carry legal exposure.

The organizations that pay are, almost universally, the organizations that did not invest in immutable backups. The correct answer to the ransom note is written in your backup architecture long before the attack happens.

Build the architecture. Run the drills. Automate the response. The playbook only works if it exists before you need it.
