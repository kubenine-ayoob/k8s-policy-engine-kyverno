# Kyverno Runbook — kubenine-intern / kubenine

Install Kyverno on Civo K3s, apply baseline policies in Audit mode, test violations,
and roll out to Enforce one policy at a time.

Operational reference: `complete-guide.md` · Agent entry: `../CLAUDE.md`

## 0. Prerequisites

- Context set: `kubectl config use-context kubenine-intern` (PoC) or `kubenine` (prod)
- RBAC sufficient to install cluster-scoped webhooks and `ClusterPolicy` CRDs
- `helm` 3.x and `helmfile` installed
- Nodes Ready: `kubectl get nodes`

Create PoC test namespace:

```bash
kubectl create namespace ayoob-kyverno --dry-run=client -o yaml | kubectl apply -f -
```

## 1. Deploy Kyverno via Helmfile

From `k8-extended/kyverno/`:

```bash
kubectl config use-context kubenine-intern
helmfile sync
```

`helmfile sync` creates the `kyverno` namespace (`createNamespace: true`) and installs/upgrades the release.

## 2. Verify Kyverno install

```bash
kubectl -n kyverno get pods
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=kyverno -n kyverno --timeout=120s
kubectl get crd | grep kyverno
kubectl get validatingwebhookconfiguration | grep kyverno
kubectl get mutatingwebhookconfiguration | grep kyverno
```

Expected: all four controller pods `Running`:
- `kyverno-admission-controller`
- `kyverno-background-controller`
- `kyverno-reports-controller`
- `kyverno-cleanup-controller`

## 3. Apply baseline policies (Audit)

```bash
kubectl apply -f policies/require-pod-requests-limits.yaml
kubectl apply -f policies/disallow-privileged-containers.yaml
kubectl apply -f policies/require-labels.yaml
kubectl get clusterpolicy
```

Verify Audit mode:

```bash
kubectl describe clusterpolicy require-labels | grep "Validation Failure Action"
# Expected: Audit
```

## 4. Test violations (Audit — pods still created)

```bash
kubectl apply -f demos/bad-no-label.yaml
kubectl apply -f demos/bad-privileged.yaml
kubectl apply -f demos/bad-no-limits.yaml
kubectl apply -f demos/good-pod.yaml
```

## 5. Check PolicyReports

```bash
kubectl get policyreport -n ayoob-kyverno
kubectl describe policyreport -n ayoob-kyverno
```

Expected:
- `bad-no-label`, `bad-privileged`, `bad-no-limits` → `fail`
- `good-pod` → `pass`

## 6. Optional advanced policies

```bash
kubectl apply -f policies/add-team-label.yaml
kubectl apply -f demos/mutate-test.yaml
kubectl get pod mutate-test -n ayoob-kyverno --show-labels

kubectl apply -f policies/generate-configmap.yaml
kubectl create namespace kyverno-generate-test
kubectl get configmap -n kyverno-generate-test
kubectl delete namespace kyverno-generate-test

kubectl apply -f policies/verify-image.yaml
kubectl apply -f demos/verify-signed.yaml
kubectl apply -f demos/verify-unsigned.yaml
```

## 7. Rollout to Enforce (one policy at a time)

Order: `require-labels` → `disallow-privileged-containers` → `require-requests-limits`.

1. Burn down violations in PolicyReports (2+ weeks in Audit)
2. Set `validationFailureAction: Enforce` in **one** policy file
3. `kubectl apply -f policies/<policy>.yaml`
4. Monitor 48 hours

```bash
kubectl apply -f policies/require-labels.yaml
kubectl apply -f demos/bad-no-label.yaml   # expect: denied
kubectl apply -f demos/good-pod.yaml       # expect: allowed
```

## 8. Rollback policy to Audit

```bash
# Edit policy: validationFailureAction: Audit
kubectl apply -f policies/<policy-file>.yaml
```

## 9. Rollback Helm release

```bash
helm history kyverno -n kyverno
helm rollback kyverno <REVISION> -n kyverno
```

## 10. Teardown

```bash
kubectl delete -f demos/ --ignore-not-found
kubectl delete clusterpolicy --all
helmfile destroy
kubectl delete namespace kyverno
kubectl delete namespace ayoob-kyverno --ignore-not-found
```

## 11. Troubleshooting

| Symptom | Check |
|---------|--------|
| `helmfile sync` fails | `helm repo add kyverno https://kyverno.github.io/kyverno/` |
| Pods not Ready | `kubectl -n kyverno describe pod` · OOM? increase limits in `install/values.yaml` |
| Admission denied unexpectedly | `kubectl describe clusterpolicy` · check Enforce vs Audit |
| No PolicyReports | Confirm `background: true` on policy · check background controller logs |
| All applies fail | Kyverno admission down · `kubectl -n kyverno get pods` |

See `complete-guide.md` Sections 16–17 for full troubleshooting.
