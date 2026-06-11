# Kyverno Civo K3s Enforce PoC — INT-496 Test Requirements

| Field | Value |
|-------|-------|
| Task | INT-496 — Kyverno enforce-mode admission policies on Civo K3s |
| Cluster | kubenine-intern |
| Context | `kubenine-intern-pinniped` |
| Kubernetes | `v1.34.2+k3s1` (Civo managed K3s) |
| Kyverno app | `v1.18.1` |
| Helm chart | `kyverno/kyverno` `3.8.1` |
| Helm revision | 8 (2026-06-11) |
| Test namespace | `ayoob-kyverno` |
| Tester | ayoob |
| Date | 2026-06-11 |
| Result | **PASS** |

---

## 1. Objective

Prove Kyverno can enforce a baseline policy set on Civo managed K3s, including the audit → enforce rollout path and the webhook failure-mode caveat.

**Success criteria (one sentence):** A pod that violates policy (e.g. privileged, no resource limits) is rejected at admission with a clear message when Kyverno is healthy, while compliant workloads pass — and we documented what happens to the cluster when the Kyverno admission webhook has no backends.

**Observed:** PASS — privileged pod denied at admission with explicit Kyverno message; compliant `good-pod` allowed; failure-mode test completed with empty `kyverno-svc` endpoints documented.

---

## 2. Environment

| Item | Value |
|------|-------|
| Nodes | 2× Ready, Alpine Linux 3.22, containerd `2.1.5-k3s1` |
| kubectl client | `v1.35.5` |
| API server | `v1.34.2+k3s1` |
| Install method | Helmfile (`helmfile sync`) from `kyverno/` |
| Admission replicas | 3/3 |
| Background replicas | 2/2 |
| Reports replicas | 2/2 |
| Webhook service | `kyverno-svc:443` → admission controller pods |
| `failurePolicy` | `Fail` on `kyverno-resource-validating-webhook-cfg` (and 5/6 other validating webhooks; TTL webhook uses `Ignore`) |

---

## 3. Install steps

### Prerequisites

```bash
kubectl config use-context kubenine-intern-pinniped
kubectl get nodes
helm version
helmfile version
```

### Deploy / upgrade Kyverno

```bash
cd kyverno/
helmfile sync
```

### Verify install

```bash
kubectl -n kyverno get pods
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=kyverno -n kyverno --timeout=180s
kubectl get deployment -n kyverno
helm get values kyverno -n kyverno
kubectl get crd | grep kyverno
kubectl get validatingwebhookconfiguration | grep kyverno
kubectl get validatingwebhookconfiguration kyverno-resource-validating-webhook-cfg \
  -o jsonpath='{.webhooks[*].failurePolicy}' && echo
```

### Install results (this PoC)

| Check | Result |
|-------|--------|
| `helmfile sync` | PASS — revision 8, chart `3.8.1` |
| Admission HA | PASS — `3/3` READY |
| Helm values applied | PASS — non-null (`install/values.yaml`) |
| Webhook exclusions | PASS — `kube-system` + `excludeKyvernoNamespace: true` |
| CRDs + webhooks | PASS |

**Post-install controller inventory:**

```
kyverno-admission-controller     3/3 Running
kyverno-background-controller    2/2 Running
kyverno-reports-controller       2/2 Running
kyverno-cleanup-controller       1/1 Running
```

---

## 4. Civo-specific gotchas

| Gotcha | Detail | Mitigation / note |
|--------|--------|-------------------|
| PSA + CEL VAP on Civo K3s | Civo ships Pod Security Admission and CEL ValidatingAdmissionPolicy | Kyverno complements for mutate/generate/verifyImages; avoid duplicating the same rules in PSA labels |
| Webhook `namespaceSelector` override | Overriding `config.webhooks.namespaceSelector` **replaces** chart defaults | Must include `kube-system`; use `excludeKyvernoNamespace: true` for `kyverno` ns |
| Chart 3.8.1 cleanup | No `cleanupJobs` CronJobs (removed in 3.8.x) | TTL cleanup via `cleanup-controller` Deployment |
| Intern cluster shared | Broad Enforce affects all teams | Audit first; enforce **one policy at a time** |
| Non-root images | Stock `nginx` runs as root | Use `nginxinc/nginx-unprivileged` for compliant demo pods with `runAsNonRoot: true` |
| EndpointSlice API | `Endpoints` deprecated in k8s 1.33+ | Use `kubectl get endpointslice -n kyverno -l kubernetes.io/service-name=kyverno-svc` for outage verification |

---

## 5. Enforced policy set (baseline — 5 policies)

All policies applied as `ClusterPolicy` from `policies/`. Sources: repo + [Kyverno policy library](https://github.com/kyverno/policies).

| # | Policy | File | Rule summary | PoC mode |
|---|--------|------|--------------|----------|
| 1 | `require-requests-limits` | `policies/require-pod-requests-limits.yaml` | CPU/memory requests + memory limits required | Audit |
| 2 | `disallow-privileged-containers` | `policies/disallow-privileged-containers.yaml` | No `privileged: true` | **Enforce** (demo) |
| 3 | `require-labels` | `policies/require-labels.yaml` | `app.kubernetes.io/name` label required | Audit |
| 4 | `disallow-latest-tag` | `policies/disallow-latest-tag.yaml` | No `:latest` image tag | Audit |
| 5 | `require-run-as-nonroot` | `policies/require-run-as-nonroot.yaml` | `runAsNonRoot: true` required | Audit |

### Apply commands

```bash
kubectl apply -f policies/require-pod-requests-limits.yaml
kubectl apply -f policies/disallow-privileged-containers.yaml
kubectl apply -f policies/require-labels.yaml
kubectl apply -f policies/disallow-latest-tag.yaml
kubectl apply -f policies/require-run-as-nonroot.yaml
kubectl get clusterpolicy
```

### Out of scope (follow-ups)

| Policy | Type | Notes |
|--------|------|-------|
| `add-team-label` | mutate | Demo only; removed during PoC cleanup |
| `generate-configmap-on-ns` | generate | Demo only; removed during PoC cleanup |
| `verify-image` | verifyImages | Requires Cosign signing in CI/CD before prod Enforce |
| PolicyExceptions | — | Enabled in `kyverno` namespace via Helm values; multi-tenant exceptions not tested |

---

## 6. Audit phase results

### Test namespace setup

```bash
kubectl create namespace ayoob-kyverno --dry-run=client -o yaml | kubectl apply -f -
kubectl delete -f demos/ --ignore-not-found
kubectl apply -f demos/bad-no-label.yaml
kubectl apply -f demos/bad-privileged.yaml
kubectl apply -f demos/bad-no-limits.yaml
kubectl apply -f demos/bad-latest.yaml
kubectl apply -f demos/bad-run-as-root.yaml
kubectl apply -f demos/good-pod.yaml
```

### Demo pod results (Audit — all pods **created**, not blocked)

| Pod | Primary violation | PolicyReport (approx.) |
|-----|-------------------|------------------------|
| `bad-no-label` | Missing label | FAIL `require-labels` |
| `bad-privileged` | `privileged: true` | FAIL `disallow-privileged-containers` |
| `bad-no-limits` | No resource limits | FAIL `require-requests-limits` |
| `bad-latest` | `nginx:latest` | FAIL `disallow-latest-tag` |
| `bad-run-as-root` | `runAsNonRoot: false` | FAIL `require-run-as-nonroot` |
| `good-pod` | None | **6 pass / 0 fail** |

Compliant pod image: `nginxinc/nginx-unprivileged:1.25` with `securityContext.runAsNonRoot: true`.

### Cluster-wide violation inventory

```bash
kubectl get policyreport -A --no-headers | wc -l
# Result: 188 PolicyReports cluster-wide
```

**Interpretation:** Many existing workloads violate baseline policies. Broad cluster-wide Enforce requires audit burn-down before rollout.

---

## 7. Enforce rejection demo

### Policy enforced

`disallow-privileged-containers` set to `validationFailureAction: Enforce`.

### Commands

```bash
kubectl apply -f policies/disallow-privileged-containers.yaml
kubectl delete pod bad-privileged -n ayoob-kyverno --ignore-not-found
kubectl apply -f demos/bad-privileged.yaml 2>&1 | tee /tmp/reject-privileged.txt
kubectl apply -f demos/good-pod.yaml
```

### Rejection message (captured)

```
Error from server: error when creating "demos/bad-privileged.yaml":
admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/ayoob-kyverno/bad-privileged was blocked due to the following policies

disallow-privileged-containers:
  privileged-containers: 'validation error: Privileged mode is disallowed. The fields
    spec.containers[*].securityContext.privileged, spec.initContainers[*].securityContext.privileged,
    and spec.ephemeralContainers[*].securityContext.privileged must be unset or set
    to `false`. rule privileged-containers failed at path /spec/containers/0/securityContext/privileged/'
```

### Compliant pod

```
kubectl apply -f demos/good-pod.yaml
pod/good-pod unchanged    # allowed
```

**Result:** PASS — clear admission denial for violating pod; compliant pod passes.

---

## 8. failurePolicy blast-radius findings

### Configuration verified

```bash
kubectl get validatingwebhookconfiguration kyverno-resource-validating-webhook-cfg \
  -o jsonpath='{.webhooks[*].failurePolicy}'
# Fail
```

### Outage simulation procedure

```bash
# A — Normal state
kubectl get deploy kyverno-admission-controller -n kyverno          # 3/3
kubectl get endpointslice -n kyverno -l kubernetes.io/service-name=kyverno-svc
# 3 endpoints on kyverno-svc

# B — Simulate outage
kubectl scale deployment kyverno-admission-controller -n kyverno --replicas=0
kubectl wait --for=delete pod -l app.kubernetes.io/component=admission-controller -n kyverno --timeout=90s
kubectl get endpointslice -n kyverno -l kubernetes.io/service-name=kyverno-svc
# ENDPOINTS: <unset> (empty — no webhook backends)

# C — Tests during outage
kubectl run failure-test-$(date +%s) -n ayoob-kyverno \
  --image=nginxinc/nginx-unprivileged:1.25 --restart=Never
# Result: pod CREATED

kubectl apply -f demos/bad-privileged.yaml
# Result: pod CREATED (Enforce policy NOT applied — admission down)

# D — Restore
kubectl scale deployment kyverno-admission-controller -n kyverno --replicas=3
kubectl wait --for=condition=available deployment/kyverno-admission-controller -n kyverno --timeout=120s
```

### Observed blast radius (kubenine-intern, Civo K3s)

| Scenario | Namespace | Expected (textbook `failurePolicy: Fail`) | **Observed** |
|----------|-----------|-------------------------------------------|--------------|
| New Pod CREATE | `ayoob-kyverno` | Blocked | **ALLOWED** |
| Privileged Pod (Enforce policy) | `ayoob-kyverno` | Blocked | **ALLOWED** |
| Pod CREATE | `kyverno` | Allowed (excluded) | **ALLOWED** |
| ConfigMap CREATE | `ayoob-kyverno` | Allowed (not Pod policy) | **ALLOWED** |
| `kyverno-svc` endpoints | `kyverno` | Empty during outage | **Confirmed empty** |

### Interpretation

1. **Availability:** On this Civo K3s cluster, scaling admission controller to zero did **not** block new Pod creation — differs from textbook `failurePolicy: Fail` behavior when webhook is unreachable.

2. **Security:** During outage, Enforce policies were **not applied** — a privileged pod was created. Cluster was temporarily **unprotected**.

3. **Mitigation (production):**
   - Run **3 admission replicas** + PodDisruptionBudget (`minAvailable: 1`)
   - Monitor admission pod health and `kyverno-svc` endpoint count
   - **Never** scale admission controller to 0 in production
   - Restore admission immediately if pods crash
   - Do not rely on outage blocking as a safety mechanism on this distribution without further validation on `kubenine` prod

---

## 9. Recommended audit → enforce rollout sequence

**Golden rule:** Never switch all policies to Enforce at once. Run Audit for 2+ weeks; burn down violations via PolicyReports; enforce one policy at a time; monitor 48 hours after each change.

### Recommended order (easiest → hardest)

| Week | Policy | Rationale |
|------|--------|-----------|
| 1–2 | All 5 policies | **Audit** — inventory violations (`kubectl get policyreport -A`) |
| 3 | Teams fix violations | Labels, limits, tags, runAsNonRoot, privileged |
| 4 | `require-labels` | Easiest — add one label |
| 5 | `disallow-latest-tag` | Pin image tags in CI/CD |
| 6 | `disallow-privileged-containers` | Security baseline |
| 7 | `require-run-as-nonroot` | May require image / securityContext changes |
| 8 | `require-requests-limits` | Most violations typically — enforce last |

### Per-policy enforce procedure

```bash
# 1. Set validationFailureAction: Enforce in ONE policy file
# 2. Apply
kubectl apply -f policies/<policy>.yaml
# 3. Monitor 48 hours
kubectl get policyreport -A
# 4. Rollback if needed: set back to Audit and re-apply
```

### Rollback

```bash
# Policy rollback
# Edit policy: validationFailureAction: Audit
kubectl apply -f policies/<policy>.yaml

# Helm rollback (emergency)
helm history kyverno -n kyverno
helm rollback kyverno <REVISION> -n kyverno
```

---

## 10. Go / no-go

| Area | Verdict | Notes |
|------|---------|-------|
| Kyverno install on Civo K3s | **GO** | Chart 3.8.1, HA 3/3, webhooks + CRDs healthy |
| Helm values (exclusions, HA, metrics) | **GO** | `kube-system` + `excludeKyvernoNamespace` |
| 5-policy Audit baseline | **GO** | All Ready; PolicyReports generated |
| Enforce admission with clear messages | **GO** | Privileged pod denied with explicit message |
| Compliant workload admission | **GO** | `good-pod` allowed |
| Broad Enforce on intern cluster | **WAIT** | 188 cluster-wide PolicyReports — burn down first |
| Broad Enforce on `kubenine` prod | **WAIT** | Phased rollout per Section 9 |
| Failure-mode: availability freeze | **N/A on k3s test** | Pods not blocked when AC scaled to 0 |
| Failure-mode: security during outage | **RISK** | Policies not enforced when AC down — mitigate with HA |
| verifyImages Enforce | **NO-GO** | Needs Cosign signing pipeline in CI/CD |
| mutate / generate in prod baseline | **NO-GO** | Follow-up; not in this PoC scope |

### Overall

| Target | Result |
|--------|--------|
| **INT-496 PoC (kubenine-intern)** | **PASS** |
| **kubenine prod — Audit rollout** | **GO** |
| **kubenine prod — full Enforce** | **WAIT** (violation burn-down + phased enforce) |

---

## 11. References

| Resource | Path |
|----------|------|
| Helmfile | `helmfile.yaml` |
| Values | `install/values.yaml` |
| Policies | `policies/` |
| Demos | `demos/` |
| Agent playbook | `CLAUDE.md` |
| Operations | `docs/runbook.md` |
| Deep dive | `docs/complete-guide.md` |
| Prior PoC report | `docs/poc-report.md` |

---

## Appendix — Quick validation commands

```bash
# Policy status
kubectl get clusterpolicy
kubectl describe clusterpolicy disallow-privileged-containers | grep "Validation Failure Action"

# Violations
kubectl get policyreport -n ayoob-kyverno
kubectl describe policyreport -n ayoob-kyverno

# Kyverno health
kubectl get pods -n kyverno
kubectl get endpointslice -n kyverno -l kubernetes.io/service-name=kyverno-svc

# Webhook failurePolicy
kubectl get validatingwebhookconfiguration -o custom-columns=\
NAME:.metadata.name,FAILURE:.webhooks[0].failurePolicy | grep kyverno
```
