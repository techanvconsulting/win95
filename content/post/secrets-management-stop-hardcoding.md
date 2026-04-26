+++
author = "Anubhav Gain"
title = "Secrets Management: Stop Hardcoding Credentials Before They End Up on GitHub"
date = "2026-04-12"
description = "6 million secrets leaked on GitHub in 2023. Hardcoded credentials are a solved problem — here is the solution stack."
tags = ["secrets", "security", "devops", "vault", "credentials"]
categories = ["security", "devops"]
+++

In 2023, GitHub's secret scanning program detected over **6 million secrets** leaked in public repositories — API keys, database passwords, private keys, OAuth tokens. That number is not slowing down. Every day, developers push credentials to version control, and every day, automated scrapers harvest those credentials within seconds of the push. The breach does not happen weeks later — it happens before you have time to revert the commit.

<!--more-->

## The Scale of the Problem

The GitHub numbers are damning on their own, but real-world breaches make the cost concrete. In 2022, an Uber contractor's credentials were compromised through social engineering — but the blast radius was amplified by hardcoded secrets in internal code repositories that gave attackers lateral movement across AWS, Google Cloud, and internal tooling. Samsung's 2022 breach exposed nearly 200GB of source code to the Lapsus$ group, including hardcoded credentials embedded in build configurations. In both cases, hardcoded secrets were not the initial vector — they were the accelerant.

GitGuardian's 2024 State of Secrets Sprawl report found that **1 in 10 authors** on public GitHub repositories has exposed a secret at some point. On private repositories — which developers assume are safe — the rate is worse because the false sense of security reduces vigilance.

## Why It Keeps Happening

The root cause is not carelessness — it is friction. The ergonomically easiest path is `DB_PASSWORD="hunter2"` directly in the config file. Secrets managers require setup, IAM policies, SDK calls, and local emulation. `.env` files are the middle ground developers reach for, and they work — until someone runs `git add .` without a `.gitignore` entry, or until the `.env` file gets committed as part of a "quick fix" at 11pm.

CI pipelines create a second surface. Secrets injected as environment variables appear in build logs when a careless `printenv` or verbose test runner dumps the environment. Third-party actions on GitHub can exfiltrate `GITHUB_TOKEN` or cloud credentials if the action itself is compromised — a supply chain attack vector that is increasingly common.

## Secret Scanning Tools

Before fixing your pipeline, scan what is already there. Several tools target this:

- **GitGuardian**: Real-time monitoring of commits across public and private repos. Detects 350+ secret types with low false-positive rates. Integrates with Slack, Jira, and GitHub PR checks.
- **truffleHog**: Open-source, entropy-based and regex-based scanner. Run it against your full git history: `trufflehog git file://. --only-verified` to confirm secrets that are still active.
- **git-secrets**: AWS Labs tool that hooks into pre-commit to block commits containing patterns matching AWS credential formats. Lightweight and fast for local enforcement.
- **GitHub Secret Scanning**: Native to GitHub, automatically alerts repository admins when supported secret types are pushed. For public repos, GitHub notifies the issuing service provider directly.

Run these against your full commit history, not just HEAD. Secrets committed two years ago and "deleted" in a follow-up commit are still in the git object store and fully readable.

## The Right Way: Environment Variables vs Secrets Managers

Environment variables are a step up from hardcoded strings, but they are not a secrets management solution — they are a secrets transport mechanism. The secret still has to come from somewhere, and that somewhere should be a secrets manager.

For local development, tools like **direnv** combined with `.envrc` files (excluded from version control) keep secrets out of code. For production, the secret should never touch the filesystem at all — it should be fetched at runtime from a dedicated secrets store and injected into the process environment or passed directly to the SDK client.

## HashiCorp Vault

Vault is the reference implementation for secrets management in complex environments. Its key differentiator over static secrets stores is **dynamic secrets**: instead of storing a database password, Vault generates a unique, time-limited credential on demand for each requesting service. When the lease expires, the credential is revoked. An attacker who steals that credential has a ticking clock.

Key capabilities:

- **Secret leasing**: Every secret has a TTL. Short-lived credentials limit the window of exposure.
- **Audit log**: Every read, write, and renewal is logged with the requesting entity's identity. When an incident happens, you know exactly which service touched which secret and when.
- **Dynamic secrets engines**: Vault can generate credentials for PostgreSQL, MySQL, AWS IAM, PKI certificates, SSH keys, and more — eliminating long-lived static credentials entirely.

The operational overhead of running Vault is real, but HCP Vault (the managed offering) removes the Raft cluster management burden while retaining the full API surface.

## Cloud-Native Alternatives

If you are already inside a cloud provider's ecosystem, their native secrets managers reduce operational overhead at the cost of portability:

- **AWS Secrets Manager**: Supports automatic rotation for RDS, Redshift, and DocumentDB. Integrates with IAM for fine-grained access control. Costs $0.40/secret/month plus API call fees.
- **GCP Secret Manager**: IAM-native access control, version management, and audit logging via Cloud Audit Logs. Regional or global replication options.
- **Azure Key Vault**: Manages secrets, keys, and certificates. Integrates with Managed Identities to eliminate credential handling entirely for Azure-hosted workloads.

All three support SDK-based retrieval at runtime and can be combined with their respective IAM systems so that application code never handles a static secret at all — the pod or function's identity is the credential.

## Kubernetes Secrets: Not Actually Secret

Kubernetes `Secret` objects are base64-encoded, not encrypted. Anyone with `kubectl get secret` access — which is broader than most teams realize — can decode and read them. By default, secrets are stored unencrypted in etcd.

Two tools address this:

- **Sealed Secrets** (Bitnami): Encrypts secrets using a cluster-side controller's public key. The encrypted `SealedSecret` object is safe to commit to version control. Only the controller can decrypt it.
- **External Secrets Operator**: Syncs secrets from Vault, AWS Secrets Manager, GCP Secret Manager, or Azure Key Vault into native Kubernetes `Secret` objects at runtime. The source of truth stays in the external store; Kubernetes holds only ephemeral copies.

Additionally, enable [etcd encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) in your cluster configuration. It is not enabled by default.

## Rotation: Static Secrets Are Ticking Bombs

A secret that never rotates will eventually be compromised — through a breach, an insider threat, a phishing attack, or a forgotten Slack message. Rotation limits the damage window.

Automate rotation at the infrastructure level:

- AWS Secrets Manager can rotate RDS credentials on a configurable schedule using Lambda functions.
- Vault's dynamic secrets engines rotate by design — every credential is temporary.
- For secrets that cannot be dynamically generated, use Vault's static secret rotation to enforce periodic rotation and log every rotation event.

Treat secrets with long TTLs the same way you treat unpatched CVEs — as known-bad configurations that require remediation on a timeline.

## How OpenArmor Handles Post-Compromise Scenarios

Preventing credential leaks is the goal. Detecting credential abuse when prevention fails is the reality.

OpenArmor's behavioral detection layer monitors authentication patterns across cloud APIs, databases, and internal services. When a leaked credential is used — even a valid one — OpenArmor flags anomalous behavior: access from unexpected IP ranges, unusual API call patterns, off-hours activity, or access to resources that the credential has never touched before. This is not signature-based detection; it is behavioral baselining that catches the attacker using a stolen key even when the key itself looks legitimate.

Combined with OpenArmor's audit integration for Vault and cloud secrets managers, teams get a unified view: which secrets were accessed, by whom, from where, and whether that access pattern is consistent with normal behavior. When the answer is no, the alert fires before the attacker has time to pivot.

Hardcoded credentials are a solved problem at the tooling level. The gap is adoption. The stack — git-secrets for prevention, truffleHog for detection, Vault or a cloud-native manager for storage, External Secrets Operator for Kubernetes, and OpenArmor for runtime behavioral monitoring — covers the full lifecycle. The cost of setting it up is measured in hours. The cost of skipping it is measured in breach notifications.
