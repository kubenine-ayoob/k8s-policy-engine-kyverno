# Kyverno PoC Report — block `:latest` image tags on kubenine

| Field | Value |
|-------|-------|
| Task | PoC: Kyverno — block `:latest` image tags on kubenine cluster |
| Cluster | kubenine |
| Context | kubenine-pinniped |
| Kubernetes | v1.32.5+k3s1 |
| Kyverno app | v1.18.1 |
| Helm chart | kyverno/kyverno 3.8.1 |
| Helm revision | 1 |
| Policy | `disallow-latest-tag` |
| Policy mode | Enforce |
| Tester | ayoob |
| Date | 2026-06-11 |
| Result | **PASS** |

---

## Executive summary

Kyverno v1.18.1 (chart 3.8.1) was installed on the kubenine production cluster via `helmfile sync`. A single ClusterPolicy, `disallow-latest-tag`, was applied in Audit mode first, validated, then switched to Enforce. New pods using `:latest` or untagged images are rejected at admission with a clear Kyverno message. Civo-managed platform images in `kube-system` are exempt. Existing production workloads using `:latest` continue to run; app teams must pin image tags before the next rollout.

---

## Goal

Install Kyverno on kubenine and enforce a policy that blocks `:latest` and untagged image references on all workload namespaces.

---

## Success criteria

| # | Criterion | Result |
|---|-----------|--------|
| 1 | Kyverno installed and running on kubenine | **PASS** |
| 2 | Policy in Enforce — `:latest` / untagged rejected with clear message | **PASS** |
| 3 | Exemption list documented for Civo-managed images | **PASS** |
| 4 | Existing prod deployments verified (no breakage to running pods) | **PASS** |
| 5 | Write-up: which `vikasy/*` images need retagging vs exempt | **PASS** |

---

## Install

```bash
kubectl config use-context kubenine-pinniped
cd kyverno/
helmfile sync
```

| Check | Result |
|-------|--------|
| `helmfile sync` | PASS — revision 1, chart `3.8.1`, app `v1.18.1` |
| Admission controller | PASS — 3/3 Running |
| Background controller | PASS — 2/2 Running |
| Reports controller | PASS — 2/2 Running |
| Cleanup controller | PASS — 1/1 Running |
| CRDs | PASS — `clusterpolicies.kyverno.io` present |
| Webhook exclusions | PASS — `kube-system` + `kyverno` excluded |

Post-install pod inventory:

```
kyverno-admission-controller     3/3 Running
kyverno-background-controller    2/2 Running
kyverno-reports-controller       2/2 Running
kyverno-cleanup-controller       1/1 Running
```

Webhook `namespaceSelector` (verified after policy apply):

```
NotIn: kube-system
NotIn: kyverno
```

---

## Policy

**File:** `policies/disallow-latest-tag.yaml`

| Setting | Value |
|---------|-------|
| Rules | `require-image-tag` (no untagged images), `validate-image-tag` (no `:latest`) |
| Audit phase | `validationFailureAction: Audit` |
| Enforce phase | `validationFailureAction: Enforce` |
| Background scan | `true` |

```bash
kubectl apply -f policies/disallow-latest-tag.yaml
kubectl get clusterpolicy disallow-latest-tag
```

---

## Tests

### Audit phase

Test pod `nginx:latest` in namespace `kyverno-poc-test`:

| Check | Expected | Observed | Result |
|-------|----------|----------|--------|
| Pod admitted | Allowed | `1/1 Running` | PASS |
| PolicyReport | `fail` on `validate-image-tag` | PASS 1, FAIL 1 | PASS |

### Enforce phase

| Test | Expected | Observed | Result |
|------|----------|----------|--------|
| `nginx:latest` pod | Admission denied | Denied by Kyverno | PASS |
| `nginxinc/nginx-unprivileged:1.25` pod | Allowed | `pod/test-good-tag created` | PASS |

Enforce rejection message (captured):

```
admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/kyverno-poc-test/test-enforce-block was blocked due to the following policies

disallow-latest-tag:
  validate-image-tag: validation failure: validation error: Using a mutable image
    tag e.g. 'latest' is not allowed. rule validate-image-tag failed at path /image/
```

### Existing prod workloads (no restart performed)

| Deployment | Namespace | Ready | Result |
|------------|-----------|-------|--------|
| `moduai-github-slack-webhook` | `default` | 2/2 | PASS |
| `kubenine-app-chart` | `divyansh` | 1/1 | PASS |
| `civo-ccm` | `kube-system` | 1/1 | PASS |

---

## Exemption list — Civo-managed (`kube-system`)

These images use `:latest` but cannot be changed. They are excluded via webhook `namespaceSelector` and `resourceFilters`.

| Image | Workload | Kind | Namespace |
|-------|----------|------|-----------|
| `civo/civo-cloud-controller-manager:latest` | `civo-ccm` | Deployment | `kube-system` |
| `gcr.io/consummate-yew-302509/csi:latest` | `civo-csi-node` | DaemonSet | `kube-system` |
| `gcr.io/consummate-yew-302509/csi:latest` | `civo-csi-controller` | StatefulSet | `kube-system` |

---

## Retagging required — `vikasy/*` and in-scope workloads

These workloads currently use `:latest`. Existing pods keep running. **The next rollout will fail** until images are pinned to a fixed tag (e.g. `v1.2.3` or git SHA) in Helm values / CI/CD.

| Image | Deployment | Namespace | Action |
|-------|------------|-----------|--------|
| `vikasy/moduai-github-slack-webhook:latest` | `moduai-github-slack-webhook` | `default` | Retag before next rollout |
| `vikasy/kubenine-app:latest` | `kubenine-app-chart` | `divyansh` | Retag before next rollout |
| `vikasy/kubenine-app-temporalworker:latest` | `kubenine-temporalworker` | `divyansh` | Retag before next rollout |
| `vikasy/civo-cost-calculator:latest` | `civo-cost-calculator` | `default` | Retag before next rollout |
| `dbeaver/cloudbeaver:latest` | `cloudbeaver` | `cloudbeaver` | Retag before next rollout |

---

## Rollout path

1. Install Kyverno (`helmfile sync`)
2. Apply `disallow-latest-tag` in **Audit** — verify no admission blocks
3. Inventory `:latest` workloads cluster-wide
4. Switch to **Enforce** — verify block + compliant image allowed
5. Verify existing prod deployments still healthy

---

## Go / no-go

| Area | Verdict |
|------|---------|
| Kyverno install on kubenine prod | **GO** |
| `disallow-latest-tag` Enforce on workload namespaces | **GO** |
| Civo platform exemptions (`kube-system`) | **GO** |
| `vikasy/*` image retagging | **ACTION** — app teams must pin tags before next deploy |

**Overall: PASS**

---

## References

- Install: `helmfile.yaml`, `install/values.yaml`
- Policy: `policies/disallow-latest-tag.yaml`
- Intern PoC (separate): `docs/poc-report.md`
