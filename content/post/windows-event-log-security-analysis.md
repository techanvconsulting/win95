+++
author = "Anubhav Gain"
title = "Windows Event Logs: The Security Analyst's Complete Reference"
date = "2026-04-03"
description = "Windows event logs contain everything an attacker tries to hide. Know your Event IDs — 4624, 4625, 4688, 4698, 7045 — and what they mean for detection."
tags = ["windows", "event-logs", "soc", "detection", "security"]
categories = ["security", "detection-engineering"]
+++

Windows event logs are not just audit records — they are the primary forensic artefact on every Windows endpoint and server. Every threat actor who has touched a Windows environment has left footprints here: authentication attempts, process execution, service installs, scheduled tasks, account changes, and eventually the attempt to erase it all. The job of a SOC analyst is knowing exactly which logs to look at, which Event IDs matter, and what a malicious sequence looks like against a backdrop of legitimate noise.

<!--more-->

## Why Windows Event Logs Are the Primary Forensic Source

Windows Event Logging is native. It exists on every Windows machine in your environment before any agent is deployed. EDR telemetry and network flow data are valuable, but they depend on third-party software being present and running. Event logs are not. When an attacker authenticates, spawns a process, installs a service, or clears their tracks, it hits the Windows Event Log infrastructure first.

The Security channel carries most of the signal. The System channel covers service and driver activity. The PowerShell/Operational and Sysmon channels fill in the gaps native logging leaves open. Together they give you a near-complete picture of host activity — if you have configured Windows to record it.

## Critical Event IDs Reference

Default audit policy is minimal. Most of the Event IDs below are disabled out of the box and require explicit configuration via Group Policy or `auditpol.exe`.

| Event ID | Channel       | Description                              | Detection Relevance                                               |
|----------|---------------|------------------------------------------|-------------------------------------------------------------------|
| 4624     | Security      | Successful logon                         | Logon type: 2=interactive, 3=network, 10=RDP — type reveals method |
| 4625     | Security      | Failed logon                             | Brute force and spray; correlate source IP + username volume      |
| 4648     | Security      | Explicit credential logon                | Credential use under a different account — lateral movement, RunAs |
| 4688     | Security      | Process creation                         | Gold mine; enable command-line logging or arguments are invisible  |
| 4698     | Security      | Scheduled task created                   | Persistence; inspect action path and trigger                      |
| 4702     | Security      | Scheduled task modified                  | Hijacking of existing task for persistence                        |
| 4720     | Security      | User account created                     | Backdoor account creation                                         |
| 4732     | Security      | Member added to local security group     | Privilege escalation; watch additions to Administrators           |
| 4776     | Security      | NTLM credential validation               | NTLM where Kerberos expected — pass-the-hash indicator            |
| 7045     | System        | New service installed                    | Common malware persistence; check binary path carefully           |
| 1102     | Security      | Audit log cleared                        | Anti-forensics; near-certain attacker activity                    |
| 4104     | PS/Operational | Script block logged                     | Full PowerShell code captured including decoded obfuscation       |

**Logon types for 4624 matter.** Type 3 (network) covers SMB and is high volume. Type 10 (RemoteInteractive) is RDP. An unexpected Type 10 from an external IP after hours is a detection-worthy signal. Bulk Type 3 failures across multiple accounts from one source is a spray attempt.

## Enabling Audit Policies and PowerShell Logging

Enable these via Group Policy under `Security Settings > Advanced Audit Policy`:

- **Audit Process Creation** — enables Event 4688
- **Include command line in process creation events** — without this, 4688 shows the process name but not arguments. `powershell.exe -enc <base64>` is invisible without it.
- **Audit Logon Events** and **Audit Account Logon Events** — enables 4624, 4625, 4776
- **Audit Policy Change** and **Audit System Events** — captures 1102 log clearing

For PowerShell, enable **Script Block Logging** via `Computer Configuration > Administrative Templates > Windows Components > Windows PowerShell`. Event ID 4104 records the actual script content as executed — including decoded Base64, downloaded payloads, and dynamically constructed commands. Alert on anything containing `IEX`, `DownloadString`, `WebClient`, or `Invoke-Expression` that does not match a known-good baseline.

## Sysmon: Where Native Logging Falls Short

Native Windows logging does not record network connections per process, loaded DLLs, or cross-process thread injection. Sysmon fills those gaps with a kernel-mode driver writing to its own event channel.

Top Sysmon Event IDs:

- **Event 1** (Process Create): reliable command line + parent details + file hash
- **Event 3** (Network Connection): which process made which outbound connection — essential for C2 detection
- **Event 7** (Image Loaded): DLL loading; detects hijacking and injection
- **Event 8** (CreateRemoteThread): cross-process thread injection — almost always malicious
- **Event 11** (File Created): new file with full process context

The SwiftOnSecurity Sysmon configuration is the community standard starting point. It filters high-volume legitimate noise while preserving the signal that matters.

## Detection Patterns

**Pass-the-hash** produces Type 3 logons (4624) from unexpected source workstations combined with 4776 NTLM events where Kerberos would normally be used.

**Lateral movement via PsExec/SMB** generates a Type 3 logon on the target, then a 7045 service install (PSEXESVC), then process creation under that session.

**Persistence via scheduled tasks** surfaces as 4698/4702 with action paths pointing to `%TEMP%` or `AppData\Roaming`, SYSTEM as the principal, and on-logon or on-idle triggers.

**Privilege escalation** appears as 4732 — an account added to Administrators or Domain Admins outside business hours or from an unexpected principal.

**Anti-forensics**: Event ID 1102 (Security log cleared) or 104 in System should generate an immediate alert. It is almost never legitimate.

## Forwarding to SIEM

**Windows Event Forwarding (WEF)** is native — endpoints push to a Windows Event Collector via WinRM with no agent required, but adds a collection hop before reaching your SIEM.

**Agent-based collection** (Wazuh, Elastic Agent, Splunk UF) installs directly on endpoints, ships events with lower latency and richer field normalization. For SOC teams building on open-source tooling, this is the practical path.

Either way: collect Security, System, Application, and PowerShell/Operational at minimum. Add Sysmon/Operational when deployed. Filter high-volume low-value events (Type 3 logons from service accounts) at the agent level to manage ingestion costs.

## OpenArmor Windows Agent and Built-In Detection Rules

OpenArmor's Windows agent reads directly from the Windows Event Log API and the Sysmon channel with no dependency on WEF. Built-in rules map Event IDs to MITRE ATT&CK techniques out of the box — pass-the-hash (T1550.002), scheduled task persistence (T1053.005), new service installation (T1543.003), and log clearing (T1070.001) all fire without additional configuration.

The detection gap most environments discover on first deployment is not missing rules — it is missing audit policy. The agent can only alert on events Windows actually records. Run the OpenArmor audit policy benchmark against a freshly joined workstation and you will typically find a third to half of the critical audit categories are disabled. Fix the audit policy first. Then tune the rules.

Windows event logs will tell you almost everything that happened on a compromised host. The prerequisite is configuring Windows to write it down.
