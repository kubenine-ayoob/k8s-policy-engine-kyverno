# Kyverno Baseline Policies

Simple starting set of policies for our Kubernetes clusters.

**Tested on:** k3s cluster (`v1.34.2`), Kyverno `v1.18.1`, namespace `ayoob-kyverno`

---

## What is this?

These are **rules** that run every time someone creates or updates a Pod.
Kyverno checks the Pod YAML and either:

- **Audit** — allow the Pod, but log a warning (safe to start with)
- **Enforce** — block the Pod if it breaks a rule

**We recommend starting in Audit mode** on all production clusters.

---

## Our 3 baseline policies

| # | Policy name | What it checks | Why we need it |
|---|-------------|----------------|----------------|
| 1 | `require-requests-limits` | Every container must have CPU + memory **requests** and memory **limits** | Stops one app from eating all cluster resources |
| 2 | `disallow-privileged-containers` | Containers cannot run with `privileged: true` | Privileged containers can bypass most security — high risk |
| 3 | `require-labels` | Every Pod must have label `app.kubernetes.io/name` | Helps monitoring, logging, and cost tracking find the right app |

Policy files live in: `policies/`

---

## Policy 1 — require-requests-limits

**File:** `policies/require-pod-requests-limits.yaml`

**Good Pod example** (passes):

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi
```

**Bad Pod example** (fails): no `resources` section at all.

**Demo file:** `demos/bad-no-limits.yaml`

---

## Policy 2 — disallow-privileged-containers

**File:** `policies/disallow-privileged-containers.yaml`

**Good Pod example** (passes): no `privileged` field, or `privileged: false`

**Bad Pod example** (fails):

```yaml
securityContext:
  privileged: true
```

**Demo file:** `demos/bad-privileged.yaml`

---

## Policy 3 — require-labels

**File:** `policies/require-labels.yaml`

**Good Pod example** (passes):

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-app
```

**Bad Pod example** (fails): Pod has no labels, or missing `app.kubernetes.io/name`

**Demo file:** `demos/bad-no-label.yaml`

---

## How to apply (Audit mode)

Make sure each policy file has:

```yaml
validationFailureAction: Audit
```

Then apply:

```bash
kubectl apply -f policies/require-pod-requests-limits.yaml
kubectl apply -f policies/disallow-privileged-containers.yaml
kubectl apply -f policies/require-labels.yaml
```

Check they are ready:

```bash
kubectl get clusterpolicy
```

---

## How to check violations (Audit mode)

After someone creates a Pod, check reports:

```bash
kubectl get policyreport -A
kubectl describe policyreport -n <namespace>
```

- **Pass** = Pod follows the rule
- **Fail** = Pod broke the rule (but was still created in Audit mode)

---

## Namespaces to exclude (production)

Do **not** apply these policies blindly to system namespaces.
Exclude at minimum:

| Namespace | Why |
|-----------|-----|
| `kube-system` | Core Kubernetes components |
| `kyverno` | Kyverno itself |
| `cert-manager` | Certificate management |
| `ingress-nginx` | Ingress controller |

Use Helm `namespaceSelector` or Kyverno **PolicyExceptions** for break-glass cases.

---

## Policies we tested but are NOT in baseline

These were for learning only — not part of the default production set:

| Policy | Type | Purpose |
|--------|------|---------|
| `add-team-label` | mutate | Auto-add `team: devops` label |
| `generate-configmap-on-ns` | generate | Auto-create ConfigMap when Namespace is created |
| `verify-image` | verifyImages | Block unsigned container images |

Add these later when the team is ready (especially `verify-image` — needs Cosign signing in CI/CD).

---

## When to switch from Audit → Enforce

Do this **one policy at a time**, never all at once:

1. Run in Audit for 2+ weeks
2. Fix violations found in PolicyReports
3. Switch the easiest policy first (`require-labels`)
4. Wait 48 hours, watch for problems
5. Then enforce `disallow-privileged-containers`
6. Last: enforce `require-requests-limits` (most teams have the most violations here)

Change in the policy file:

```yaml
validationFailureAction: Enforce
```

Then re-apply: `kubectl apply -f policies/<policy-file>.yaml`

---

## Quick reference

```bash
# List policies
kubectl get clusterpolicy

# Check mode (Audit or Enforce)
kubectl describe clusterpolicy require-labels | grep "Validation Failure Action"

# View violations
kubectl get policyreport -A
```
