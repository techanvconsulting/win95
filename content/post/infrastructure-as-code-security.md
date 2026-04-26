+++
author = "Anubhav Gain"
title = "Infrastructure as Code Security: Stop Deploying Misconfigurations at Scale"
date = "2026-03-22"
description = "IaC lets you deploy infrastructure in seconds. It also lets you deploy misconfigurations at scale. Here is how to secure Terraform, Pulumi, and Ansible."
tags = ["iac", "terraform", "security", "devsecops", "cloud"]
categories = ["security", "devops"]
+++

Before IaC, a misconfigured S3 bucket was a one-off mistake. A sysadmin clicked the wrong checkbox, someone noticed, and it got fixed. With Terraform, that same misconfiguration lives in a module — and now it deploys to every environment, every region, every time the pipeline runs. IaC made infrastructure reproducible, auditable, and fast. It also made misconfigurations programmable and scalable.

<!--more-->

## The IaC Security Problem

The fundamental issue with IaC security is not that teams write insecure Terraform. It is that insecure Terraform gets committed, reviewed by people focused on functionality, merged, and then replicated across production. A vulnerable module in an internal registry can instantiate the same open security group across fifty microservices before anyone notices.

The attack surface of IaC spans the code itself, the pipeline that runs it, the state files that record what was built, and the secrets required to build it. Each layer needs attention.

## Terraform: The Most Common Misconfigurations

Most Terraform security findings cluster around a small set of recurring patterns:

**S3 buckets with public access.** The classic. An `aws_s3_bucket` resource with `acl = "public-read"` or without `aws_s3_bucket_public_access_block` set correctly exposes data to the internet. With versioning and lifecycle policies, many teams assume default-private behavior that is not actually enforced.

**Security groups with 0.0.0.0/0 ingress.** `cidr_blocks = ["0.0.0.0/0"]` on port 22 or 3389 is the most scanned-for finding on public cloud infrastructure. It appears routinely in "temporary" debug configurations that outlive their intended lifespan by years.

**Unencrypted EBS volumes.** `aws_ebs_volume` resources without `encrypted = true` store data in plaintext on disk. AWS does not encrypt at rest by default unless you enable account-level enforcement — a setting most teams do not configure.

**IAM policies with `*` actions on `*` resources.** Convenience policies that grant `iam:*` or `s3:*` on `*` routinely make it to production because they work and nobody challenged them during review.

## Static Analysis for IaC: Checkov, tfsec, and Terrascan

The first line of defense is static analysis — running these checks before any infrastructure is provisioned.

**Checkov** covers the broadest surface: Terraform, CloudFormation, Kubernetes manifests, Dockerfiles, Helm, and ARM templates. It maps findings to CIS benchmarks, NIST, SOC 2, and PCI-DSS. Drop it into CI as a blocking gate:

```bash
# Scan a Terraform directory, fail on HIGH or CRITICAL
checkov -d ./terraform --check CKV_AWS_18,CKV_AWS_20 --soft-fail-on LOW,MEDIUM

# Full scan with SARIF output for GitHub Advanced Security
checkov -d ./terraform --output sarif --output-file-path checkov-results.sarif
```

**tfsec** focuses exclusively on Terraform and runs fast — typically under five seconds on a medium-sized codebase. Its checks are Terraform-idiomatic and the output is clean enough to pipe into PR comments:

```bash
# Scan with color output, fail pipeline on any finding
tfsec ./terraform

# Output as JSON for custom tooling or dashboard ingestion
tfsec ./terraform --format json --out tfsec-results.json

# Ignore a specific check inline (use sparingly, document why)
# tfsec:ignore:aws-s3-enable-bucket-logging
```

**Terrascan** adds policy-as-code capabilities via Rego policies and supports Terraform, Kubernetes, Helm, and Kustomize. For teams that need custom organizational policies beyond the built-in ruleset, Terrascan's Rego layer provides extensibility that Checkov's Python checks require more effort to replicate.

In CI, run at minimum one of these tools as a blocking check on every pull request that touches infrastructure code. Checkov in strict mode or tfsec with `--minimum-severity HIGH` is a reasonable starting threshold.

## Terraform State Security

Terraform state files are a security liability that most teams underestimate. `terraform.tfstate` stores the actual resource values of your infrastructure — including sensitive outputs, database passwords passed through module outputs, and private key material from `tls_private_key` resources. A state file is not just metadata; it is a high-value target.

**Never store state locally.** Local state files get committed to version control. This is how database credentials and private keys end up in git history.

**Use remote state with encryption.** S3 remote backends support server-side encryption with KMS. Every team using Terraform on AWS should have this configured:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/..."
    dynamodb_table = "terraform-state-lock"
  }
}
```

**State locking via DynamoDB** prevents concurrent runs from corrupting state, but it also prevents a class of race-condition attacks where two simultaneous applies produce inconsistent infrastructure.

**Restrict access to the state bucket** with IAM policies that limit who and what can read state. CI runners need read/write; most humans only need read during incident response.

## Secret Management in IaC

Hardcoding secrets in `.tf` files is the IaC equivalent of committing passwords to source control — which is exactly what it becomes once the file is committed. Variables passed via `-var` flags appear in shell history. Secrets in Terraform outputs get written to state.

The correct pattern is to never let secrets touch Terraform HCL at all. Two production-grade approaches:

**HashiCorp Vault provider** fetches secrets at plan/apply time and injects them into resources without writing to state:

```hcl
data "vault_generic_secret" "db_creds" {
  path = "secret/prod/database"
}

resource "aws_db_instance" "main" {
  username = data.vault_generic_secret.db_creds.data["username"]
  password = data.vault_generic_secret.db_creds.data["password"]
  # ...
}
```

**Environment variable injection** for simpler cases — `TF_VAR_db_password` is read by Terraform at runtime without appearing in any file. CI systems (GitHub Actions, GitLab CI, Jenkins) all support injecting secrets as environment variables from a secrets store.

Mark any sensitive outputs with `sensitive = true` in Terraform 0.14+ to prevent them from appearing in plan output and to scrub them from most log sinks.

## Drift Detection: Infrastructure Drift is Security Drift

Drift is what happens when the actual state of your infrastructure diverges from what Terraform expects — a manual security group edit, an IAM policy attachment added by a third-party tool, a compliance scan that auto-remediates a finding Terraform will overwrite on the next apply.

Drift is a security problem because your static analysis, audit trail, and policy-as-code enforcement all operate on a false model of the environment the moment it diverges.

Run `terraform plan` in CI on a schedule against production state, not just on pull requests:

```bash
# Run plan in CI and fail if there are unexpected changes
terraform plan -detailed-exitcode -out=tfplan
# Exit code 2 means changes detected — alert on this
```

A non-zero exit code from `terraform plan` in a scheduled job means something changed outside of Terraform. Treat it as an incident until proven otherwise — pipe plan output to a security channel and investigate before the next apply.

## Ansible Security

Ansible introduces its own attack surface. Playbooks often run with elevated privileges, connect over SSH to production hosts, and handle sensitive configuration data.

**Ansible Vault** encrypts secrets at rest within playbooks and variable files:

```bash
# Encrypt a variables file containing secrets
ansible-vault encrypt group_vars/production/secrets.yml

# Run playbook with vault password from CI secret store
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

**`no_log: true`** on tasks that handle sensitive data prevents values from appearing in Ansible output or logs:

```yaml
- name: Set database password
  ansible.builtin.lineinfile:
    path: /etc/app/config.ini
    line: "db_password={{ db_password }}"
  no_log: true
```

**Minimal privilege for connection users.** The SSH user Ansible uses to connect should not be root and should not have unrestricted sudo. Use `become` with a specific `become_user` and limit sudo rules to only the commands the playbook actually needs. A compromised Ansible controller with a root SSH key is a full host takeover — limit the blast radius.

## Policy as Code: Sentinel and OPA

Static analysis catches known-bad patterns. Policy as code enforces organizational rules that may not have a built-in check in any scanner.

**Sentinel** (Terraform Enterprise/Cloud) evaluates policies against Terraform plans before apply. Sentinel policies can enforce cost constraints, require specific tags on all resources, block deployment to unapproved regions, or mandate encryption on every storage resource. Policies are first-class citizens of the Terraform Enterprise workflow.

**OPA (Open Policy Agent)** provides the same capability for open-source Terraform pipelines. Write policies in Rego, evaluate them against `terraform plan` JSON output in CI:

```bash
# Convert plan to JSON, evaluate against OPA policy bundle
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
opa eval --data policies/ --input tfplan.json "data.terraform.deny" --fail
```

A Rego policy that blocks any security group allowing 0.0.0.0/0 on port 22 is ten lines of code that runs in milliseconds and blocks an entire class of misconfiguration before a single resource is provisioned.

## How OpenArmor Monitors IaC-Provisioned Infrastructure for Runtime Drift

Static analysis and policy-as-code handle pre-deployment. OpenArmor handles what comes after.

Infrastructure provisioned by Terraform starts in a known-good state defined by code. Over time, the runtime state diverges — through manual changes, application behavior, attack activity, or third-party tool intervention. OpenArmor ingests the Terraform-defined expected state and continuously compares it against the observed runtime posture: active security group rules, IAM policy attachments, open ports, running processes, and network connections.

When OpenArmor detects a configuration change that has no corresponding Terraform commit — a new inbound rule, an added IAM permission, a newly opened port — it surfaces this as a drift alert with full context: which resource changed, what the previous state was, and whether the change pattern matches known attack techniques (lateral movement, persistence, privilege escalation). The correlation between IaC-defined intent and runtime observation is where infrastructure security becomes operational, rather than staying on paper.

For teams running Terraform in production, OpenArmor's IaC integration closes the gap between "what we deployed" and "what is running" — making infrastructure drift visible before it becomes a breach.

**OpenArmor is open-source. Documentation, integrations, and Terraform state ingestion setup are at [techanv.com](https://techanv.com).**
