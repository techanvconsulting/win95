+++
author = "Anubhav Gain"
title = "Threat Hunting with YARA and Sigma: Finding Evil Before It Finds You"
date = "2026-04-05"
description = "Reactive security waits for alerts. Threat hunters go looking. YARA hunts files, Sigma hunts logs — here is how to use both."
tags = ["threat-hunting", "yara", "sigma", "soc", "security"]
categories = ["security", "threat-hunting"]
+++

Alert-driven security has a coverage gap that most SOC teams quietly acknowledge and almost no one addresses directly. Roughly 80 percent of attacks that result in confirmed breaches did trigger some alert — the problem was triage backlog, false-positive fatigue, or analyst bandwidth. But there is a harder 20 percent: sophisticated intrusions that specifically engineer themselves to stay below detection thresholds, evade signature-based rules, and blend into the noise of legitimate activity. That 20 percent does not show up in your SIEM. It shows up in your breach notification letter. Threat hunting closes the gap.

<!--more-->

## Reactive vs Proactive: Why Alert-Driven SOC Is Not Enough

A traditional SOC workflow is event-driven: an alert fires, an analyst investigates, a ticket is opened or closed. This model is efficient at processing known-bad signals. It is structurally blind to attacker behaviors that do not match existing rules — which includes every novel TTP, every living-off-the-land technique using legitimate admin tools, and every low-and-slow credential harvesting campaign that never crosses a frequency threshold.

Threat hunting inverts the model. Instead of waiting for the system to surface something, hunters start with a hypothesis — "I think an adversary is using scheduled tasks for persistence on our Windows endpoints" — and go looking for evidence that confirms or refutes it. The output is not just a finding; it is a new detection rule that catches the same behavior automatically next time. Hunting converts unknown unknowns into known detections.

The two tools that operationalize this at scale are YARA for file and memory scanning, and Sigma for log-based hunting across SIEM platforms.

## YARA: Pattern Matching for Files and Memory

YARA is a pattern-matching engine purpose-built for malware identification. A YARA rule describes characteristics of a file or memory region — byte sequences, strings, file structure markers — and the engine scans targets to find matches. Security teams use YARA to scan filesystem artifacts, process memory dumps, network captures, and email attachments.

A YARA rule has three sections: metadata (descriptive, not used for matching), strings (the patterns to find), and condition (the logic that combines strings into a match decision).

Here is a rule that detects the default Cobalt Strike beacon configuration pattern. The beacon stores its configuration as an XOR-encoded blob with a recognizable header signature:

```yara
rule CobaltStrike_Beacon_Config {
    meta:
        author      = "TechAnv / OpenArmor"
        description = "Detects Cobalt Strike beacon configuration block"
        date        = "2026-04-05"
        reference   = "https://www.cobaltstrike.com/help-malleable-c2"
        severity    = "critical"

    strings:
        // Default watermark and config header bytes
        $cs_config_magic = { 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
                              00 01 00 01 00 01 00 02 00 01 00 02 00 02 }
        // XOR key used in default profile
        $cs_xor_key     = { 69 69 69 69 }
        // Staging URI pattern common to default profiles
        $cs_uri_pattern = "/submit.php" nocase
        $cs_pipe_prefix = "\\\\.\\pipe\\msagent_" nocase

    condition:
        uint16(0) == 0x5A4D and                  // PE file
        filesize < 2MB and
        ($cs_config_magic or $cs_xor_key) and
        1 of ($cs_uri_pattern, $cs_pipe_prefix)
}
```

The `condition` block is where the detection logic lives. `uint16(0) == 0x5A4D` checks for the MZ PE header, keeping this rule from matching non-executables. Combining the magic bytes with URI or pipe patterns reduces false positives substantially.

## Writing YARA Rules: Strings, Conditions, Metadata

A more general-purpose example — detecting a generic PowerShell downloader dropper commonly used in commodity malware:

```yara
rule PS_Downloader_Dropper {
    meta:
        author      = "TechAnv / OpenArmor"
        description = "Detects PowerShell-based downloader dropper patterns"
        date        = "2026-04-05"
        severity    = "high"

    strings:
        $ps1 = "powershell" nocase wide ascii
        $dl1 = "DownloadString" nocase wide ascii
        $dl2 = "DownloadFile"   nocase wide ascii
        $dl3 = "WebClient"      nocase wide ascii
        $enc = "-EncodedCommand" nocase wide ascii
        $byp = "bypass"          nocase wide ascii
        $iex = "IEX"             wide ascii
        $iex2 = "Invoke-Expression" nocase wide ascii

    condition:
        $ps1 and
        (2 of ($dl1, $dl2, $dl3)) and
        ($enc or $byp) and
        (1 of ($iex, $iex2))
}
```

Key authoring principles: use `wide ascii` for strings that may appear in UTF-16 encoded PE resources. Use `nocase` sparingly — it increases scan time. Write conditions that require multiple independent indicators rather than single-string matches to control false positives. Always include `filesize` guards.

## Sigma: Universal SIEM Rule Format

Sigma is to logs what YARA is to files. It is a vendor-neutral, YAML-based rule format for describing detection logic over log data. Write a rule once; convert it to Splunk SPL, Elastic DSL, Wazuh rules, or Microsoft Sentinel KQL using the `sigma-cli` converter. This eliminates the pain of maintaining four separate rule sets for the same detection across different SIEM backends.

A Sigma rule defines the log source, the detection fields, the condition logic, and enriching metadata including MITRE ATT&CK technique mappings.

## Writing Sigma Rules: Real Windows Event Log Hunting Example

This rule hunts for `schtasks` abuse — a common persistence technique (T1053.005) where attackers create scheduled tasks that execute malicious payloads:

```yaml
title: Suspicious Scheduled Task Created via Schtasks
id: 9d3c6e7b-2f41-4a8e-b3c1-5d7f9e0a1234
status: experimental
description: >
  Detects scheduled task creation using schtasks.exe with suspicious
  arguments commonly used for persistence or lateral movement.
author: TechAnv / OpenArmor
date: 2026/04/05
references:
  - https://attack.mitre.org/techniques/T1053/005/
tags:
  - attack.persistence
  - attack.t1053.005
logsource:
  category: process_creation
  product: windows
detection:
  selection:
    Image|endswith: '\schtasks.exe'
    CommandLine|contains|all:
      - '/create'
      - '/tr'
  suspicious_args:
    CommandLine|contains:
      - 'powershell'
      - 'cmd.exe'
      - 'wscript'
      - 'cscript'
      - 'mshta'
      - 'rundll32'
      - 'regsvr32'
      - 'AppData'
      - 'Temp\\'
      - '%TEMP%'
  filter_legitimate:
    CommandLine|contains:
      - 'MicrosoftEdgeUpdateTask'
      - 'GoogleUpdate'
  condition: selection and suspicious_args and not filter_legitimate
falsepositives:
  - Legitimate software installation using schtasks with temp paths
level: high
```

Convert this to Splunk syntax with: `sigma convert -t splunk -p splunk_windows sigma_rule.yml`

The `filter_legitimate` block is critical. Hunting rules without suppressions for known-good behavior generate alert fatigue that defeats the purpose.

## Threat Hunting Workflow

A structured hunting workflow prevents hunts from becoming undirected fishing expeditions:

1. **Hypothesis formation.** Ground the hunt in a specific, falsifiable claim. "Attackers using T1055 process injection are running injected code in svchost.exe on workstations." Vague hypotheses produce vague results.
2. **Data collection.** Identify what data sources are needed. Process creation logs? Network flow data? EDR telemetry? File system events? Confirm the data is available, complete, and covers the relevant time window.
3. **Rule development.** Write the YARA or Sigma rule that operationalizes the hypothesis. Start broad, then tighten conditions based on what you find.
4. **Validate.** Run the rule against historical data and known-clean baselines. Tune false positives. Confirm true positives against lab samples or known-bad IOCs.
5. **Operationalize.** Graduate the validated rule from a hunt artifact into a production detection. Deploy it to the SIEM or EDR with appropriate alerting thresholds. Document the MITRE technique, expected false positives, and response guidance.

The final step is the one teams skip most often. A hunt that finds nothing still produces a validated rule. A hunt that finds something produces an incident and a rule. Either outcome has value only if the rule gets deployed.

## MITRE ATT&CK-Driven Hunting: Prioritizing What to Hunt

MITRE ATT&CK gives hunters a structured vocabulary for selecting hypotheses based on threat intel rather than intuition. The practical approach: obtain threat intelligence about which threat actors are most likely to target your industry and geography, map their known TTPs to ATT&CK technique IDs, then audit your existing detection coverage against those technique IDs.

The gaps in your detection coverage — ATT&CK techniques used by relevant threat actors that have no corresponding production rule in your SIEM — are your hunting backlog. Prioritize by technique prevalence in recent incidents in your sector, then by the availability of data sources needed to detect the technique. A technique you have no telemetry for requires a data collection investment before a hunt is meaningful.

For a financial services SOC, relevant clusters include T1566 (phishing), T1021.002 (SMB lateral movement), T1003 (credential dumping), and T1486 (data encryption for impact). Each maps to specific log sources and either a YARA rule, a Sigma rule, or both.

## How OpenArmor Integrates YARA and Sigma

OpenArmor ships with a curated, community-maintained library of YARA and Sigma rules mapped to ATT&CK techniques. The platform handles the operational friction that makes rule-based hunting hard in practice: automated conversion of Sigma rules to the target SIEM backend, scheduled YARA scanning jobs against file system paths and process memory, and a rule lifecycle workflow that tracks rules from draft through review to production deployment.

When a YARA scan or Sigma alert fires, OpenArmor links the finding to the relevant ATT&CK technique, surfaces related rules that may indicate broader campaign activity, and routes the alert through a configurable response playbook. For teams running Wazuh as their open-source SIEM, OpenArmor's Sigma integration pushes converted rules directly to the Wazuh rule set with no manual XML editing required.

The goal is to close the loop between hunting and detection: every confirmed finding becomes a production rule, and every production rule is backed by the telemetry and context that made the hunt possible. Threat hunting stops being a one-off project and becomes a continuous process that systematically shrinks the coverage gap — finding evil before it finds you.
