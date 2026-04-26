+++
author = "Anubhav Gain"
title = "Identity and Access Management: Building Bulletproof IAM from Scratch"
date = "2026-04-01"
description = "80% of breaches involve compromised credentials. IAM is not an IT problem — it is your most critical security control. Here is how to build it right."
tags = ["iam", "identity", "security", "authentication", "authorization"]
categories = ["security", "identity"]
+++

Eighty percent of breaches involve compromised or abused credentials. Firewalls and endpoint agents matter — but none of them stop an attacker who logs in with valid credentials. Identity is not an IT provisioning problem. It is your most critical security control, and in cloud-native environments it is the only perimeter that still means anything.

<!--more-->

## Why Identity Is the New Perimeter

The traditional network perimeter dissolved when organizations adopted SaaS, remote work, and public cloud. Requests now arrive from home laptops, CI/CD pipelines, and third-party integrations — network location tells you almost nothing about whether to trust them. Every major cloud breach post-mortem traces back to an overpermissioned role, a stolen API key, or a misconfigured trust policy. Build your security model around identity first.

## Authentication vs. Authorization

These are frequently conflated, and conflating them produces bugs that become vulnerabilities. **Authentication** answers: who are you? It verifies a principal's identity and produces an assertion. **Authorization** answers: what are you allowed to do? It evaluates that identity against policy to determine permitted operations. The critical failure mode is granting access based solely on successful authentication without enforcing authorization separately. Authenticate at the edge — authorize at the resource.

## MFA: Not All Second Factors Are Equal

**TOTP** (Google Authenticator, Authy) is widely supported but phishable — an attacker can relay the code in real time. **FIDO2/WebAuthn** (YubiKey, Touch ID, Windows Hello) is phishing-resistant: the credential is bound to the origin domain at registration, so it cannot be replayed against a spoofed site. For engineers and privileged users, FIDO2 should be the standard.

**Push notifications** are convenient but vulnerable to MFA fatigue — spamming approvals until a user accepts. Pair push with number matching to close that gap. **SMS codes are not real MFA**: SIM-swapping is common and SS7 interception is documented. Treat SMS as a last-resort fallback only.

## SSO: SAML vs. OIDC

SSO centralizes authentication — one place to require MFA, audit access, and terminate sessions. **SAML** is XML-based and well-supported by legacy enterprise applications; its signature validation rules are complex and have produced real CVEs, so use a mature library with strict assertion validation. **OIDC** (built on OAuth 2.0) uses JSON tokens and is the right default for modern applications. Validate `redirect_uri` against a strict allowlist to prevent open redirects, use PKCE for public clients, and keep access tokens short-lived. Default to OIDC; keep SAML for legacy integrations that require it.

## Least Privilege: RBAC, ABAC, and Just-in-Time Access

**RBAC** assigns permissions to roles and roles to users — simple to reason about, but prone to role proliferation where dozens of nearly identical roles accumulate permissions unchecked. **ABAC** makes authorization decisions based on attributes of the subject, resource, and environment, handling dynamic patterns RBAC cannot express, at the cost of a policy engine and disciplined attribute management.

**Just-in-time access** grants elevated permissions only for the duration of a specific task — engineers request elevation, receive a time-limited grant, and access expires automatically. Use RBAC as your baseline, ABAC where dynamic context is required, and JIT for anything privileged.

## Service Identities: Machine-to-Machine Auth

Avoid shared service accounts with keys embedded in environment variables and permissions broad enough to "just work" for everything. Use **workload identity** wherever your platform supports it — AWS IAM roles for Lambda/ECS, GCP Workload Identity Federation, Kubernetes OIDC tokens — eliminating long-lived static credentials entirely. Where workload identity is unavailable, use a secrets manager with automatic rotation and never share credentials across services.

## IAM for Cloud: AWS as the Model

A few principles that apply across providers. **Roles over users**: IAM users carry long-lived access keys; roles issue temporary STS credentials — reserve IAM users for break-glass scenarios only. **Permission boundaries** cap the maximum permissions a role can exercise regardless of attached policies, preventing privilege escalation when delegating IAM management. **SCPs** enforce guardrails at the AWS Organizations level; an SCP denying `iam:CreateUser` in production prevents mistakes no per-role configuration can prevent. Run AWS Access Analyzer regularly to surface unused grants and overly broad cross-account trust.

## Privileged Access Management (PAM)

Admin credentials — root accounts, domain admin, database superusers — require a dedicated control model. PAM platforms (HashiCorp Vault, CyberArk) vault these so no individual holds the standing password; access requires a workflow, sessions are recorded, and credentials rotate after each use. Session recording is non-negotiable: knowing what commands ran under a privileged session is the difference between an hour of investigation and a week. **Break-glass access** must be documented, time-limited, dual-authorized, fully audited, and tested before you need it.

## How OpenArmor Monitors Identity Events

Strong IAM controls are necessary but not sufficient. Credentials get phished, tokens leak, and overpermissioned roles get abused in ways that technically comply with policy. OpenArmor ingests authentication logs and cloud audit trails — AWS CloudTrail, Okta System Log, Kubernetes audit logs — and builds behavioral baselines to surface anomalies. A service account that has never called `s3:GetObject` suddenly exfiltrating data at 2 AM is technically authorized; static enforcement would not catch it, but behavioral detection does. Patterns monitored include impossible travel, MFA fatigue signatures, lateral movement via role assumption chains, and token reuse outside expected service boundaries.

IAM is not a one-time configuration project. It is an ongoing discipline of reducing standing privilege and monitoring for abuse. Build it right and your identity plane becomes your strongest defense layer, not your largest attack surface.
