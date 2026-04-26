+++
author = "Anubhav Gain"
title = "Container Security: From Image Scanning to Runtime Protection"
date = "2026-04-07"
description = "Scanning your Docker images at build time is table stakes. Real container security covers the full lifecycle — from Dockerfile to production runtime."
tags = ["containers", "docker", "kubernetes", "security", "devsecops"]
categories = ["security", "cloud"]
+++

Most teams add a vulnerability scanner to their CI pipeline, check the box, and call it "container security." But image scanning alone is roughly as effective as locking your front door while leaving the windows open. Attackers don't care where they get in — and once inside a running container, a clean scan result provides zero protection.

<!--more-->

## The Container Security Lifecycle: Build → Ship → Run

Container threats don't live in one place. They span three distinct phases, each with its own attack surface.

**Build time** is where vulnerabilities enter through base images, outdated dependencies, and secrets accidentally baked into layers. A `npm install` in your Dockerfile can pull in dozens of transitive packages with known CVEs. Hardcoded API keys in `ENV` statements or early `COPY` commands get buried in image history but remain extractable with `docker history --no-trunc`.

**Ship time** — the registry and CI/CD pipeline — is where unsigned, unverified images get pulled and deployed. A compromised registry or a man-in-the-middle attack can swap a legitimate image for a malicious one. Without image signing, there's no cryptographic proof that what you built is what your cluster runs.

**Runtime** is where most attacks actually happen. A container passes every static check, lands in production, and then gets exploited via a zero-day, a misconfigured capability, or a vulnerable application endpoint. Static analysis has already done all it can do at this point — and that wasn't enough.

## Dockerfile Hardening

Defense starts at the source. A few non-negotiable Dockerfile practices:

**Run as a non-root user.** The default `USER` in most base images is root (UID 0). Add `USER 1000` or create a dedicated user with `RUN useradd -r appuser`. If the container process is compromised, a non-root UID dramatically limits what an attacker can do.

**Use minimal base images.** `ubuntu:latest` ships with a shell, package managers, curl, and hundreds of utilities that attackers love. Distroless images (e.g., `gcr.io/distroless/java`) contain only the runtime and your application — no shell, no package manager, no attack surface. Alpine Linux is a reasonable middle ground at around 5 MB with a shell available when needed.

**Never embed secrets in image layers.** Use build-time `--secret` mounts (`RUN --mount=type=secret`) introduced in BuildKit, or inject secrets at runtime via environment variables, Kubernetes Secrets, or a secrets manager like Vault. Even if you `RUN rm .env` in a later layer, the secret exists permanently in the intermediate layer and is recoverable.

## Image Scanning: What Tools Find and What They Miss

Three tools dominate the open-source image scanning space:

- **Trivy** (Aqua Security): Fast, comprehensive. Scans OS packages, language dependencies, IaC configs, and secrets. Easy to integrate into CI with a single binary.
- **Grype** (Anchore): Focuses on OS and language-level CVEs with clean SBOM integration via Syft.
- **Snyk**: Commercial-friendly, integrates deeply with developer workflows, and includes license compliance checking.

All three are useful — but they share the same fundamental limitation: they analyze what's in the image at rest. They cannot tell you what an attacker will do with a vulnerability at runtime, whether your application has a logic flaw, or if a zero-day lands after the scan completes. Image scanning is necessary; it is not sufficient.

## Image Signing and Verification

Cosign and the Sigstore project solve the image integrity problem. `cosign sign` attaches a cryptographic signature to an OCI image stored in a registry. `cosign verify` checks that signature against a known public key or the Sigstore transparency log (Rekor) before trusting the image.

Signing is half the solution. The other half is enforcing verification at admission time — blocking unsigned or unverified images before they reach your cluster. Two leading policy engines:

- **Kyverno**: Kubernetes-native, YAML-based policies. A `ClusterPolicy` with `verifyImages` rules blocks any image that fails cosign verification.
- **OPA Gatekeeper**: Uses Rego policies evaluated by the OPA admission webhook. More expressive, steeper learning curve.

Neither matters if you don't actually enforce `deny` mode. Many teams run admission controllers in audit mode indefinitely, logging violations they never act on.

## Runtime Security: Why Static Analysis Isn't Enough

Consider this scenario: a containerized Node.js application passes Trivy with zero critical CVEs. Two weeks after deployment, a new RCE vulnerability is disclosed in an upstream npm package. Before you can patch and redeploy, an attacker exploits it, drops a reverse shell, and begins lateral movement.

Your image scan was accurate when it ran. It's now irrelevant.

Runtime attackers follow predictable patterns: they execute shells (`sh`, `bash`, `python`), download additional tooling (`curl`, `wget`), write files to unexpected paths, and attempt to access cloud metadata endpoints (`169.254.169.254`). These are behavioral signals — invisible to any static tool.

## eBPF-Based Runtime Detection

Falco (CNCF) uses eBPF to hook into kernel system calls without modifying container code or requiring kernel modules. It evaluates every syscall against a rule set in real time.

A simple Falco rule to detect shell execution inside a container:

```yaml
- rule: Shell Spawned in Container
  desc: Detects shell process execution inside a running container
  condition: >
    spawned_process and container
    and proc.name in (shell_binaries)
  output: >
    Shell spawned in container (user=%user.name container=%container.name
    image=%container.image.repository proc=%proc.name parent=%proc.pname)
  priority: WARNING
```

Similarly, rules fire on unexpected writes to `/etc`, reads of `/proc/*/environ` (credential harvesting), or outbound connections to unexpected IPs. eBPF overhead is minimal — typically under 2% CPU — making it production-viable.

## Container-Escape Attacks

A container escape turns a container compromise into a host compromise. The three most common vectors:

**Privileged containers** (`--privileged` or `securityContext.privileged: true`) give the container near-complete access to the host kernel. A single `mount /dev/sda1 /mnt` gives an attacker the entire host filesystem.

**Host path mounts** (`hostPath` volumes) expose host directories inside containers. Mounting `/` or `/etc` is essentially the same as running directly on the host.

**Excessive Linux capabilities.** Docker drops most capabilities by default, but `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, and `CAP_SYS_PTRACE` are frequently granted for convenience. `CAP_SYS_ADMIN` alone has enabled multiple documented container escapes.

Runtime detection catches these: Falco fires on privileged container creation; OPA policies can block them at admission. Defense-in-depth means both.

## OpenArmor: Unified Visibility Across the Container Lifecycle

Correlating signals across build, ship, and run phases is where most point tools fall short. A CVE from a Trivy scan means more when you can see whether that vulnerable package is actually loaded at runtime, whether the affected container has suspicious syscall activity, and whether that image was properly signed before admission.

[OpenArmor](https://openarmor.io) is built around this cross-lifecycle visibility. It ingests image scan results, admission controller events, and eBPF-based runtime telemetry into a unified graph — surfacing the containers where risk is actually concentrated, not just the ones with the most CVEs. For DevSecOps teams running Kubernetes at scale, that correlation is the difference between alert fatigue and actionable detection.

Container security in 2026 isn't a scanner problem. It's an observability problem. Build hardening and scanning give you a baseline. Signing and admission control close the supply chain gap. Runtime detection with eBPF covers everything that happens after the image lands. The goal isn't to pick one — it's to connect them all.
