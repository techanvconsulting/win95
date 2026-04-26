+++
author = "Anubhav Gain"
title = "Kubernetes RBAC Done Right: Least Privilege in Practice"
date = "2026-04-10"
description = "Most Kubernetes clusters run with wildcard ClusterRoles. That is a breach waiting to happen. Here is how to do RBAC properly."
tags = ["kubernetes", "rbac", "security", "cloud-native"]
categories = ["security", "cloud"]
+++

In every Kubernetes security audit we have run, the same finding lands in the top slot: overpermissioned RBAC. Not misconfigured network policies, not exposed dashboards — RBAC. Teams set `verbs: ["*"]` in week one to unblock a deployment, ship to production, and never revisit it. That wildcard becomes permanent. That permanent wildcard becomes your blast radius.

<!--more-->

## Why RBAC Misconfiguration Is the Number One Kubernetes Security Finding

Kubernetes RBAC is additive and invisible. When you grant a permission, nothing breaks. When you grant too much, nothing breaks — until it does. There is no runtime error for over-permissioning. Security scanners may flag it, but they are easy to silence and easier to defer.

The consequence is clusters where CI/CD service accounts can delete namespaces, developers can read secrets across the entire cluster, and workloads run with cluster-admin effectively baked in. An attacker who compromises any of those identities — via a vulnerable container image, a leaked token, or a supply chain injection — gains unrestricted cluster access. That is not a theoretical risk. It is the lateral movement path in the majority of Kubernetes-related incidents.

## RBAC Primitives: The Building Blocks

Kubernetes RBAC has four core objects. Understanding the boundary between them is the first step to getting them right.

**Role** — grants permissions within a single namespace. Use this for almost everything.

**ClusterRole** — grants permissions cluster-wide, or defines a reusable role that can be bound within namespaces. Use this for cross-namespace resources (nodes, PersistentVolumes, namespaces themselves) or for shared role templates.

**RoleBinding** — binds a Role or ClusterRole to a subject (User, Group, ServiceAccount) within a specific namespace.

**ClusterRoleBinding** — binds a ClusterRole to a subject cluster-wide. This is the escalation lever. A ClusterRoleBinding to `cluster-admin` is effectively root access to the cluster.

**ServiceAccount** — the identity Kubernetes assigns to pods. Every pod runs with a ServiceAccount. If you do not specify one, it inherits the `default` ServiceAccount of the namespace.

## The Wildcard Anti-Pattern

The most common mistake looks like this:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployer
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

This is cluster-admin with a different name. It exists because it is the path of least resistance — it makes whatever was failing work immediately. It stays because RBAC debt is invisible.

The correct approach is to enumerate exactly what a role needs:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

If you cannot enumerate the required permissions, that is a signal the service doing the work has too broad a responsibility — not that it needs `*`.

## Least Privilege in Practice: Namespace-Scoped Roles

Namespace-scoped Roles are the default choice for application workloads. They contain blast radius by construction. Even if a service account is compromised, the attacker is limited to what the namespace-scoped binding allows.

A read-only service account for a metrics collector:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-reader
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: metrics-reader
  namespace: monitoring
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: metrics-reader
  namespace: monitoring
subjects:
  - kind: ServiceAccount
    name: metrics-reader
    namespace: monitoring
roleRef:
  kind: Role
  name: metrics-reader
  apiGroup: rbac.authorization.k8s.io
```

No `ClusterRole`. No `ClusterRoleBinding`. No cross-namespace access. The service account can read what it needs and nothing else.

## Auditing Existing RBAC: Three Tools Worth Running Today

Before you can fix RBAC, you need to see what it currently grants. Three tools cover this well:

**kubectl-who-can** — answers "who can do X?" queries directly:
```bash
kubectl-who-can delete pods -n production
kubectl-who-can create clusterrolebindings
```

**rakkess (access matrix)** — produces a full verb-by-resource matrix for any subject:
```bash
rakkess --sa kube-system:default
```

**rbac-tool** — generates a visual graph of bindings and highlights escalation paths:
```bash
kubectl-rbac-tool visualize
kubectl-rbac-tool lookup -k ServiceAccount -n production
```

Run all three in your cluster before doing anything else. The output will surprise you.

## ServiceAccount Token Automounting: Disabled by Default, or It Should Be

Every pod automatically mounts a ServiceAccount token at `/var/run/secrets/kubernetes.io/serviceaccount/token` unless you explicitly disable it. That token is valid for as long as the pod runs. Any code executing in the container — including injected code via a compromised dependency — can use it to call the Kubernetes API with whatever permissions the ServiceAccount holds.

Disable automounting at the ServiceAccount level for workloads that do not need API access:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-frontend
  namespace: production
automountServiceAccountToken: false
```

Or per-pod if the ServiceAccount cannot be changed:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-frontend
spec:
  automountServiceAccountToken: false
  containers:
    - name: app
      image: myapp:latest
```

The majority of application workloads have no legitimate reason to call the Kubernetes API. For those, the token is pure attack surface.

## RBAC for CI/CD Pipelines: Deployment-Only Roles

CI/CD systems are high-value targets. They typically need to update deployments and nothing more. Give them exactly that:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "patch", "update"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]
    verbs: ["get", "update"]
```

Note the `resourceNames` constraint — this locks the ConfigMap permission to a specific named resource, not all ConfigMaps in the namespace.

For tokens, use the `TokenRequest` API to issue short-lived credentials rather than storing static ServiceAccount tokens in CI secrets:

```bash
kubectl create token ci-deployer -n production --duration=1h
```

Rotate per-pipeline-run. A compromised token that expires in an hour is a fundamentally different risk than one that is valid indefinitely.

## How OpenArmor Monitors for RBAC Abuse

Correct RBAC configuration reduces the attack surface. It does not eliminate the need for runtime detection. Privilege escalation patterns are subtle and can bypass static configuration entirely: a user with `update` on RoleBindings can grant themselves cluster-admin without touching a ClusterRoleBinding.

OpenArmor's RBAC monitoring layer watches the Kubernetes audit log for:

- Binding mutations that extend permissions beyond the subject's current scope
- ServiceAccount tokens used outside their source namespace
- Unexpected API calls from workload service accounts (a pod calling `secrets/list` when it should only be calling `configmaps/get`)
- Short-lived token creation from unusual principals

Because OpenArmor is open-source, every detection rule is readable, auditable, and tunable to your environment. There is no black box between what the rule says and what fires.

RBAC done right is not a single configuration pass. It is ongoing: audit what you have, tighten what is loose, disable what is unused, and instrument the rest for runtime detection. Start with `kubectl-who-can create clusterrolebindings` and work backwards from there.

**OpenArmor provides open-source tooling to audit, enforce, and monitor Kubernetes RBAC across your clusters. Explore the project at [techanv.com](https://techanv.com) or on GitHub.**
