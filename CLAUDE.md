## Rules

1. Context must be verified before any change: `kubectl config current-context`
2. Never commit kubeconfigs, Cosign private keys, or registry credentials
3. Pin and record Kyverno chart + app + cluster versions in `docs/poc-report.md`
4. **Install Kyverno via Helm + Helmfile only** — use `helmfile sync` / `helmfile apply` from this folder. Do not use raw `kubectl apply` for the Kyverno install itself.
5. **Namespaces are created by Helmfile** — the release sets `createNamespace: true` (idempotent, re-sync-safe). The test namespace `ayoob-kyverno` is created separately for PoC demos.
6. Apply policies with `kubectl apply -f policies/` **after** Kyverno is Running.
7. Start all baseline validate policies in **Audit** mode — never switch all policies to Enforce at once.
8. Webhook exclusions: `kube-system` and `kyverno` ns (chart `excludeKyvernoNamespace`) — do not remove without platform approval.
9. Stuck? [Kyverno docs](https://kyverno.io/docs/) · `docs/complete-guide.md` · `docs/runbook.md` · ask the user

---

## Overview

Deploy **Kyverno** — a Kubernetes-native policy engine — on **Civo K3s** to validate, mutate, generate, and verify container images at admission time.

| Item | Value |
|------|--------|
| PoC cluster / context | `kubenine-intern` |
| Production target | `kubenine` |
| Kubernetes | k3s `v1.34.2+k3s1` |
| Kyverno app | `v1.18.1` |
| Helm chart | `kyverno/kyverno` `3.8.1` |
| Kyverno namespace | `kyverno` |
| Test namespace | `ayoob-kyverno` (PoC demos only) |
| Baseline policies | `require-labels`, `require-requests-limits`, `disallow-privileged-containers` |
| Default policy mode | **Audit** (log violations, do not block) |

**Civo gap:** Civo K3s ships Pod Security Admission (PSA) and CEL ValidatingAdmissionPolicy but no policy engine for **mutate**, **generate**, or **Cosign verifyImages**. Kyverno fills that gap.

**Out of scope (this PoC):**

- Production Enforce on shared intern cluster without violation burn-down
- Cosign signing in customer CI/CD (`verify-image` policy is demo-only)
- Mutate/generate policies in production baseline (optional advanced tests only)

**Done when:**

- [ ] All Kyverno pods **Running** in `kyverno` namespace
- [ ] 3 baseline `ClusterPolicy` resources **Ready: True**
- [ ] `demos/good-pod.yaml` applies successfully
- [ ] `demos/bad-*` apply in Audit mode; `PolicyReport` shows `fail`
- [ ] `demos/good-pod.yaml` shows `pass` in PolicyReport
- [ ] Report filed (`docs/poc-report.md`)

> PoC tested on `kubenine-intern` with Kyverno `v1.18.1`. See `docs/poc-report.md` for pass/fail and go/no-go for `kubenine` prod.

---

## Phases

### 1 — Prerequisites

- `kubectl config use-context kubenine-intern` (PoC) or `kubenine` (prod rollout)
- RBAC sufficient to install cluster-scoped webhooks and `ClusterPolicy` CRDs
- `helm` 3.x and `helmfile` installed
- Nodes Ready: `kubectl get nodes`

Create PoC test namespace (idempotent):

```bash
kubectl create namespace ayoob-kyverno --dry-run=client -o yaml | kubectl apply -f -
```

### 2 — Install Kyverno

**Install method: Helm chart** (via Helmfile). Do not install Kyverno with raw manifests or unpinned `helm install` without values.

`kyverno/kyverno` chart `3.8.1`: admission controller (3 replicas) + background controller + reports controller + cleanup controller. Webhook exclusions: `kube-system` + `kyverno` (via `excludeKyvernoNamespace`). PolicyExceptions enabled in `install/values.yaml`.

From `k8-extended/kyverno/`:

```bash
kubectl config use-context kubenine-intern
helmfile sync
```

**Check:**

```bash
kubectl -n kyverno get pods
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=kyverno -n kyverno --timeout=120s
kubectl get crd | grep kyverno
kubectl get validatingwebhookconfiguration | grep kyverno
kubectl get mutatingwebhookconfiguration | grep kyverno
```

Expected pods: `kyverno-admission-controller`, `kyverno-background-controller`, `kyverno-reports-controller`, `kyverno-cleanup-controller`.

### 3 — Apply baseline policies (Audit)

Apply in this order:

```bash
kubectl apply -f policies/require-pod-requests-limits.yaml
kubectl apply -f policies/disallow-privileged-containers.yaml
kubectl apply -f policies/require-labels.yaml
kubectl get clusterpolicy
```

**Check:** all three policies `READY: true`. Verify Audit mode:

```bash
kubectl describe clusterpolicy require-labels | grep "Validation Failure Action"
```

Expected: `Audit`

### 4 — Test with demos (Audit mode)

Violating pods are **still created** in Audit — violations appear in PolicyReports only.

```bash
kubectl apply -f demos/bad-no-label.yaml
kubectl apply -f demos/bad-privileged.yaml
kubectl apply -f demos/bad-no-limits.yaml
kubectl apply -f demos/good-pod.yaml

kubectl get policyreport -n ayoob-kyverno
kubectl describe policyreport -n ayoob-kyverno
```

**Check:**

- `bad-no-label`, `bad-privileged`, `bad-no-limits` → PolicyReport `fail`
- `good-pod` → PolicyReport `pass`
- All four pods exist (Audit does not block)

Clean up demos when done:

```bash
kubectl delete -f demos/ --ignore-not-found
```

### 5 — Optional advanced policies (not baseline)

```bash
# Mutate — auto-adds team=devops label in ayoob-kyverno namespace
kubectl apply -f policies/add-team-label.yaml
kubectl apply -f demos/mutate-test.yaml
kubectl get pod mutate-test -n ayoob-kyverno --show-labels

# Generate — creates ConfigMap when a new Namespace is created
kubectl apply -f policies/generate-configmap.yaml
kubectl create namespace kyverno-generate-test
kubectl get configmap -n kyverno-generate-test
kubectl delete namespace kyverno-generate-test

# VerifyImages — demo only (Kyverno public test images)
kubectl apply -f policies/verify-image.yaml
kubectl apply -f demos/verify-signed.yaml     # allowed
kubectl apply -f demos/verify-unsigned.yaml  # denied (Enforce)
```

### 6 — Rollout to Enforce (production only — one policy at a time)

**Never enforce all policies at once.** Order: labels → privileged → resource limits.

1. Run Audit for 2+ weeks; burn down violations via PolicyReports
2. Change `validationFailureAction: Enforce` in **one** policy file
3. `kubectl apply -f policies/<policy>.yaml`
4. Monitor 48 hours; rollback to Audit if unexpected blocks occur

```bash
# Example: enforce labels first
kubectl apply -f policies/require-labels.yaml
kubectl apply -f demos/bad-no-label.yaml   # expect: admission denied
kubectl apply -f demos/good-pod.yaml       # expect: allowed
```

### 7 — Report

Fill `docs/poc-report.md`: chart version, Kyverno app version, cluster context, policy test results, blockers, go/no-go for `kubenine` prod.

---

## Repo layout

```
kyverno/
├── CLAUDE.md              ← you are here (agent entry point)
├── AGENTS.md              → CLAUDE.md
├── helmfile.yaml          # kyverno release; creates kyverno ns itself
├── install/
│   └── values.yaml        # HA replicas, namespace exclusions, metrics
├── policies/              # ClusterPolicy manifests
│   ├── require-labels.yaml
│   ├── require-pod-requests-limits.yaml
│   ├── disallow-privileged-containers.yaml
│   ├── add-team-label.yaml          # optional
│   ├── generate-configmap.yaml      # optional
│   └── verify-image.yaml            # demo only
├── demos/                 # good/bad test Pods (namespace: ayoob-kyverno)
└── docs/
    ├── complete-guide.md  # full production reference (28 sections)
    ├── runbook.md         # operational commands
    └── poc-report.md      # PoC results and go/no-go
```

Detailed reference → `docs/complete-guide.md`  
Step-by-step operations → `docs/runbook.md`

---

## Gotchas

- **failurePolicy: Fail** (default) — if Kyverno admission controller is down, **all** matching admissions fail (cluster looks broken). Run 2+ admission replicas in production; `install/values.yaml` sets 3.
- **Webhook in critical path** — Kyverno sits between API server and etcd. Monitor admission controller health and latency.
- **Audit vs Enforce** — Audit logs violations but allows resources; Enforce blocks at admission. Always start Audit.
- **One policy at a time to Enforce** — switching all three baseline policies to Enforce simultaneously causes widespread deployment failures.
- **verify-image is demo-only** — uses `ghcr.io/kyverno/test-verify-image` and a public key. Production requires Cosign signing in CI/CD before Enforce.
- **Policies match Pod kind** — Kyverno autogen may create rules for Deployments/ReplicaSets depending on chart settings; verify with `kubectl describe clusterpolicy`.
- **Background scanning** — policies with `background: true` produce `PolicyReport` CRDs for existing resources. verifyImages uses `background: false`.
- **Install is Helm** (`kyverno/kyverno` chart) — record chart + app version in `docs/poc-report.md`. Upgrade via `helmfile sync`, not manual pod edits.
- **Rollback** — set policy back to Audit and re-apply; or `helm rollback kyverno <revision> -n kyverno`. Emergency only: scale admission controller to 0 (cluster unprotected).