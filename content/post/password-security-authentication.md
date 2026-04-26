+++
author = "Anubhav Gain"
title = "Password Security in 2026: Everything You Know Is Probably Wrong"
date = "2026-03-27"
description = "NIST dropped the 90-day rotation rule in 2017. Length beats complexity. Passphrases beat random strings. Here is the current science."
tags = ["password", "authentication", "security", "mfa", "identity"]
categories = ["security", "identity"]
+++

Your organization's password policy is probably making you less secure. Mandatory 90-day rotations, complexity rules that produce `P@ssw0rd1!`, bans on writing passwords down — these are not security hygiene. They are cargo-culted rules NIST formally abandoned in 2017. The science has moved. Most corporate IT has not.

<!--more-->

## What NIST SP 800-63B Actually Says

NIST SP 800-63B (2017, revised 2024) is the authoritative federal guidance on digital identity. Its conclusions contradict most enterprise policies still in production.

**Do not require periodic rotation** unless there is evidence of compromise — forced rotation produces `Password1` → `Password2` patterns that are trivially predictable. **Do not require complexity rules** — they produce substitution patterns attackers already enumerate. **Do require 15+ characters.** And critically: **check new passwords against a breached-password corpus** and reject matches.

Stop requiring rotation. Stop mandating special characters. Start checking breach lists. That is the current federal standard.

## Length vs. Complexity: The Entropy Math

An 8-character password drawn from 94 printable ASCII characters yields about 52 bits of entropy — brute-forceable at scale with modern GPUs. Complexity requirements do not meaningfully raise this; users exploit every substitution shortcut available.

Four random common words — `correct horse battery staple` — give over 70 bits of entropy from something a human can remember and type. Length wins. Passphrases beat `P@ssw0rd!` by every measurable metric.

Policy implication: set a 15-character minimum, a 64-character maximum (NIST's required ceiling), and drop complexity mandates entirely.

## Password Managers: Now Mandatory Infrastructure

Without a password manager, credential reuse is mathematically guaranteed across hundreds of services. Credential stuffing attacks exist precisely because reuse is universal. Password managers are now mandatory infrastructure, not a nice-to-have.

**Bitwarden** is open-source, audited, and self-hostable — the right choice for air-gapped or strict data-residency environments. **1Password** is a mature managed service with polished enterprise controls, SSO integration, and travel mode. Both support FIDO2 and TOTP for vault unlock. Avoid any manager that cannot export in a standard format.

## Breached Password Checking

Have I Been Pwned (HIBP) now contains over 13 billion compromised credentials. Its API uses k-anonymity: your app sends the first 5 characters of a SHA-1 hash, receives all matching hashes back, and checks locally. Your plaintext never leaves your system.

Integrate at registration and at password change. The Passwords API is free and documented at `haveibeenpwned.com/API/v3`. For enterprise deployments with strict data-residency requirements, the full hash list is downloadable via Cloudflare R2 for local lookups.

## Passkeys: The Actual End of Passwords

Passkeys implement FIDO2/WebAuthn. The device generates a public/private key pair, stores the private key in secure hardware (TPM, Secure Enclave, Android StrongBox), and sends the public key to the server. Authentication is a cryptographic challenge-response — nothing shared, nothing phishable, nothing the server can lose in a breach.

The credential is bound to the origin domain at registration. A spoofed login page gets a credential that only works at the real domain. Device possession plus biometric or PIN provides two factors by construction.

In 2026, Apple, Google, and Microsoft support passkeys natively. GitHub, Google, 1Password, and dozens of major services accept them. For any new application, passkeys should be the primary path — passwords a fallback only.

## MFA Hierarchy: What Actually Stops Attackers

Not all second factors are equal. Ranked by resistance to phishing and account takeover:

1. **FIDO2/WebAuthn** (hardware key, platform authenticator) — phishing-resistant, origin-bound. No known practical attack.
2. **TOTP** (Authenticator apps) — phishable in real time via adversary-in-the-middle proxies, but stops password-only attacks. Widely supported.
3. **Push notification with number matching** — vulnerable to MFA fatigue attacks without number matching; with it, significantly better.
4. **SMS OTP** — subject to SIM-swapping and SS7 interception. Treat as a last resort. Many financial regulators now consider it insufficient for high-value transactions.

For privileged access: FIDO2 is the requirement, not a recommendation. For general users: TOTP is the acceptable minimum. SMS should be removed from any policy that claims meaningful account security.

## Enterprise Policy: What to Require, What to Stop

Stop requiring: scheduled rotation, complexity rules, password history beyond 5 entries.

Start requiring: 15-character minimum, TOTP or FIDO2 MFA for all accounts (FIDO2 mandatory for privileged roles), a funded password manager subscription (Bitwarden Business or 1Password Teams), and breached-password screening at creation and change.

Rolling this out: announce what you are removing first. Users rotating every 90 days will feel relief before they feel new friction. Cite NIST — "the federal standard changed" lands better than "IT decided." Make password manager enrollment part of onboarding and expect adoption curves, not instant compliance.

## How OpenArmor Monitors Authentication Events

Policy changes mean nothing without visibility. OpenArmor ingests authentication logs from identity providers (Okta, Entra ID, Google Workspace), VPNs, and application login events, applying behavioral analytics to surface anomalies.

Credential stuffing has a characteristic signature: high login-endpoint volume, distributed source IPs, low error rates on valid usernames, and timing patterns consistent with automation. OpenArmor flags velocity anomalies, impossible travel, and first-seen device fingerprints on privileged accounts. Alerts route to automatic session termination and forced re-authentication.

During MFA rollouts, OpenArmor tracks enrollment rates across the directory, surfaces accounts still on downgraded factors, and enforces step-up authentication at the SIEM layer when the IdP lacks granularity.

Passwords will not disappear tomorrow. But the gap between what most organizations enforce and what the evidence supports is large enough to matter. Fix the policy. The technology has been available since 2017.
