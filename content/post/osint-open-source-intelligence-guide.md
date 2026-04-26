+++
author = "Anubhav Gain"
title = "OSINT: How Attackers Research Your Organization Before They Attack"
date = "2026-03-24"
description = "Before an attacker sends a phishing email or launches an exploit, they spend hours doing reconnaissance using only public sources. Here is exactly what they find."
tags = ["osint", "reconnaissance", "security", "social-engineering"]
categories = ["security", "research"]
+++

Before a single exploit is launched, before a single phishing email is drafted, skilled attackers spend days — sometimes weeks — doing reconnaissance. They never touch your systems. They never trigger an alert. They use the same search engines, job boards, and social platforms your employees use every day. The discipline is called Open Source Intelligence, or OSINT, and the information it surfaces is more dangerous than most security teams realize.

<!--more-->

## Reconnaissance Is the Real First Stage

Port scanning is loud. Exploitation is loud. OSINT is silent. Modern attackers — whether nation-state operators, ransomware affiliates, or competitive intelligence contractors — front-load their campaigns with passive reconnaissance that generates zero network traffic toward the target. They are reading your job postings, browsing your engineers' GitHub profiles, and mapping your infrastructure through public certificate logs before you have any indication you are being studied. Understanding what they find, and how they find it, is the first step toward limiting what they can use.

## What Your Organization Inadvertently Publishes

**Job postings.** A single job description for a Senior DevOps Engineer that mentions Kubernetes 1.28, Terraform, HashiCorp Vault, and Datadog has just told an attacker your infrastructure toolchain, your cloud provider hints, and your likely internal segmentation model. Stack-specific vulnerabilities become targeted vulnerabilities the moment an attacker knows your stack.

**LinkedIn.** The org chart your company never published is fully reconstructed from LinkedIn profiles. An attacker identifies the CFO, the IT director, the three-person SOC team, the contractor who joined six months ago, and the email format (`firstname.lastname@company.com`) from the dozen employees whose addresses appear in conference speaker bios. They know who to impersonate and who to target.

**GitHub.** Developers commit configuration files, environment variable examples with real-looking keys, internal hostnames, and occasionally production credentials. Repositories that have since been scrubbed still exist in commit history. Forks of internal tooling carry comments that describe internal architecture. GitHub is one of the most reliable sources of unintentional technical disclosure in existence.

## Email Harvesting: Your Employees Are Already Public

Tools like theHarvester automate the collection of employee email addresses from search engines, LinkedIn, and data breach aggregators. Hunter.io maintains a searchable index of email addresses tied to domains, including confidence scores based on how many addresses from a domain share the same format.

An attacker running theHarvester against a mid-size company routinely recovers fifty to two hundred valid employee email addresses in under five minutes. Those addresses feed directly into phishing campaigns, credential stuffing attacks against corporate SSO portals, and social engineering pretexts. The employees did nothing wrong — they signed up for webinars, submitted speaker proposals, and posted to professional forums using their work email. The aggregation is the risk.

## Infrastructure OSINT: What Shodan and Certificate Logs Expose

Shodan continuously scans the public internet and indexes every responsive service it finds. An attacker querying Shodan for your organization's IP ranges or ASN sees every exposed port, every TLS certificate, every banner, and every misconfigured service — often with version information that maps directly to CVEs.

```
# Find exposed services on a target organization's ASN
org:"Target Company Inc" port:8443
org:"Target Company Inc" product:"Apache Tomcat"

# Find exposed remote management interfaces
org:"Target Company Inc" port:3389 has_screenshot:true
org:"Target Company Inc" "MongoDB Server Information"
```

Certificate Transparency logs are equally illuminating. Every TLS certificate issued for your domain is logged publicly in CT logs and queryable through tools like crt.sh. Subdomains that were spun up for an internal staging environment, a vendor integration, or a deprecated product are all indexed. Attackers enumerate subdomains from CT logs before they run a single DNS brute-force query.

```
# Query crt.sh for all certificates issued for a domain
https://crt.sh/?q=%25.targetcompany.com

# Certificate entries routinely expose:
# internal.targetcompany.com
# vpn-staging.targetcompany.com
# legacy-crm.targetcompany.com
```

WHOIS and passive DNS data complete the picture: registration history, nameserver changes, IP-to-domain mappings over time, and infrastructure relationships that reveal hosting providers and CDN configurations.

## Social Media OSINT: Signals You Are Not Meant to Broadcast

LinkedIn provides the org chart. Twitter and X provide something more tactically valuable: employee frustrations. An engineer who tweets about a legacy VPN client they are finally getting rid of has disclosed a technology transition that implies a gap period. A sysadmin complaining about an ongoing patch deployment has disclosed a vulnerability window. Support employees who vent about a specific vendor's product have confirmed that product is in your environment.

Instagram and other photo-sharing platforms are a physical security resource. Badge photos, office layout photos, and conference check-ins provide building access card designs, floor plan details, and perimeter information that support tailgating and social engineering pretexts. Security teams rarely consider image metadata — photos taken inside facilities carry GPS coordinates in EXIF data unless devices are configured to strip it.

## Google Dorking: Finding What You Did Not Know Was Public

Google's advanced search operators surface documents, configuration files, and exposed interfaces that are indexed but not intended to be found. Attackers maintain libraries of dorks tuned to specific technologies and disclosure patterns.

```
# Find exposed documents belonging to a target domain
site:targetcompany.com filetype:pdf "confidential"
site:targetcompany.com filetype:xlsx "employee"

# Find login portals and admin interfaces
site:targetcompany.com inurl:admin
site:targetcompany.com inurl:login intitle:"Grafana"

# Find accidentally exposed configuration files
site:targetcompany.com filetype:env
site:targetcompany.com filetype:config inurl:database

# Find cached or indexed internal pages
site:targetcompany.com inurl:internal
cache:targetcompany.com/internal-wiki
```

Misconfigured S3 buckets, exposed Jenkins instances, publicly indexed Confluence spaces, and forgotten phpMyAdmin installations are discovered this way daily. The files were never meant to be public — but they are indexed, and dorks find them efficiently.

## Defending Against OSINT

Defense against OSINT is a combination of reduction, monitoring, and training.

**Reduce your exposure.** Audit job postings for unnecessary technology specificity — describe the role, not the exact version. Configure GitHub organization policies to require secret scanning and prevent accidental credential commits. Review what subdomains are publicly resolvable and decommission what is no longer in use. Establish a media and photography policy that prevents internal infrastructure details from appearing in social media posts.

**Monitor your footprint.** Set up Google Alerts and automated queries against Shodan, Censys, and crt.sh for your domains and IP ranges. Subscribe to breach notification services so you know when employee credentials appear in leaked databases. Monitor paste sites for content matching your domain patterns.

**Train employees on OSINT awareness.** The most effective training is experiential — show employees what an attacker finds about them specifically. People who see their own email address in a breach database, their own LinkedIn profile reconstructed into a spear-phishing pretext, and their own job description used to identify your tech stack change their behavior in ways that abstract policy cannot achieve.

## Run OSINT Against Yourself First

The most direct way to understand your OSINT exposure is to conduct a red team OSINT exercise against your own organization using the same tools and sources an attacker would use. Treat it as a structured assessment: enumerate employee addresses with theHarvester, map exposed infrastructure with Shodan and crt.sh, run targeted dorks, reconstruct the org chart from LinkedIn, and document everything found within forty-eight hours using only public sources.

What you find is what every motivated attacker finds before they contact your environment. The gaps that exercise exposes — credentials in commit history, overly detailed job postings, forgotten staging subdomains, exposed management interfaces — are remediable before they become intrusion vectors. At OpenArmor, this kind of proactive OSINT exercise is a foundational component of our red team methodology, because passive reconnaissance is not a precursor to attack. It is the attack, conducted in a space where you have no visibility and no alerts. Closing those gaps starts with seeing what is already out there.
