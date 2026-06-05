# Kyverno Rollout Plan

Step-by-step plan to roll out Kyverno on our clusters safely.

**Goal:** Start with **Audit** (log only), fix problems, then move to **Enforce** (block bad Pods) one policy at a time.

**Tested on:** k3s cluster, Kyverno `v1.18.1`, namespace `ayoob-kyverno`

---

## Big picture

```
Week 1-2: Install Kyverno + Audit mode
    ↓
Week 3:   Teams fix violations
    ↓
Week 4:   Enforce require-labels
    ↓
Week 5:   Enforce disallow-privileged
    ↓
Week 6:   Enforce require-requests-limits
```

**Golden rule:** Never switch all policies to Enforce at once.

---

## Phase 1 — Install and Audit (Week 1–2)

### What we do

- [x] Install Kyverno with Helm in `kyverno` namespace
- [x] Apply 3 baseline policies in **Audit** mode
- [ ] Set admission controller to **2 replicas** (HA) — currently 1 replica on test cluster
- [ ] Exclude system namespaces (`kube-system`, `kyverno`, `cert-manager`, `ingress-nginx`)
- [ ] Review PolicyReports every week

### Commands

```bash
# Install Kyverno
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

# Apply policies (Audit mode)
kubectl apply -f policies/

# Check policies are ready
kubectl get clusterpolicy

# Check violations weekly
kubectl get policyreport -A
```

### Success criteria

- All Kyverno pods are `Running`
- All 3 policies show `Ready: True`
- PolicyReports are being created (no errors in Kyverno logs)

---

## Phase 2 — Fix violations (Week 3)

### What we do

- Share PolicyReport results with app teams
- Each team fixes their workloads:
  - Add `app.kubernetes.io/name` label
  - Add CPU/memory requests and memory limits
  - Remove `privileged: true` from containers
- Track progress: count `Fail` results week over week

### How teams check their namespace

```bash
kubectl get policyreport -n <their-namespace>
kubectl describe policyreport -n <their-namespace>
```

### What teams need to fix

| Violation | Fix |
|-----------|-----|
| Missing label | Add `app.kubernetes.io/name: <app-name>` to Pod/Deployment |
| Missing limits | Add `resources.requests` and `resources.limits` |
| Privileged container | Remove `privileged: true` or set to `false` |

### Success criteria

- Fail count drops by at least 80% across the cluster
- No critical apps still violating security policies

---

## Phase 3 — Enforce labels first (Week 4)

**Why first?** Easiest fix for teams — just add one label.

### Steps

1. Change `require-labels.yaml`:

```yaml
validationFailureAction: Enforce
```

2. Apply:

```bash
kubectl apply -f policies/require-labels.yaml
```

3. Monitor for **48 hours**
4. Watch for support tickets / blocked deployments

### Rollback (if needed)

Change back to `Audit` and re-apply:

```bash
# Edit require-labels.yaml → validationFailureAction: Audit
kubectl apply -f policies/require-labels.yaml
```

### Success criteria

- No unexpected production outages
- New Pods without labels are blocked
- Existing compliant Pods still work

---

## Phase 4 — Enforce security (Week 5)

**Policy:** `disallow-privileged-containers`

### Steps

1. Confirm zero (or near-zero) privileged violations in PolicyReports
2. Change to `Enforce` in `disallow-privileged-containers.yaml`
3. Apply and monitor 48 hours

```bash
kubectl apply -f policies/disallow-privileged-containers.yaml
```

### Success criteria

- Privileged Pods are blocked at admission
- No platform components broken (system namespaces excluded)

---

## Phase 5 — Enforce resource limits (Week 6)

**Policy:** `require-requests-limits`

**Why last?** Most teams have the most violations here — needs the most prep work.

### Steps

1. Confirm teams added limits to their manifests
2. Change to `Enforce` in `require-pod-requests-limits.yaml`
3. Apply and monitor 48 hours

```bash
kubectl apply -f policies/require-pod-requests-limits.yaml
```

### Success criteria

- Pods without requests/limits are blocked
- Cluster resource usage is more predictable

---

## Phase 6 — Future additions (after baseline is stable)

Add these only when the team is ready:

| Policy | Type | When to add |
|--------|------|-------------|
| `verify-image` | verifyImages | After Cosign image signing is set up in CI/CD |
| `add-team-label` | mutate | When we want auto-labeling by namespace |
| `generate-configmap-on-ns` | generate | When we want auto-provisioning per namespace |

---

## Risk: failurePolicy and availability

Kyverno webhooks use `failurePolicy: Fail` by default.

**What this means:** If Kyverno is down → **nobody can create Pods** (cluster looks broken).

### How we reduce risk

| Action | Why |
|--------|-----|
| Run 2+ admission controller replicas | Survives one pod crash |
| Monitor Kyverno pod health | Alert before outage |
| Start with Audit, not Enforce | Lower blast radius during rollout |
| Exclude system namespaces | Platform components keep working |

Check current setting:

```bash
kubectl get validatingwebhookconfiguration kyverno-resource-validating-webhook-cfg -o yaml | grep failurePolicy
```

---

## Rollout checklist (summary)

| Week | Action | Mode |
|------|--------|------|
| 1–2 | Install Kyverno, apply all 3 policies | Audit |
| 3 | Teams fix violations | Audit |
| 4 | Enforce `require-labels` | Enforce (1 policy) |
| 5 | Enforce `disallow-privileged-containers` | Enforce (2 policies) |
| 6 | Enforce `require-requests-limits` | Enforce (all 3) |

---

## Who does what

| Role | Responsibility |
|------|----------------|
| Platform team | Install Kyverno, manage policies, monitor health |
| App teams | Fix violations in their namespaces |
| Security team | Review policy choices, approve Enforce timeline |
| On-call | Know how to rollback a policy to Audit in emergency |
