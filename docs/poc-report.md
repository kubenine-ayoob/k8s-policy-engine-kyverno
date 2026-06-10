# Kyverno PoC Report — kubenine-intern

| Field | Value |
|-------|-------|
| Spec / task | Kyverno policy engine on Civo K3s |
| Cluster | kubenine-intern |
| Context | kubenine-intern-pinniped |
| Kubernetes | v1.34.2+k3s1 |
| Kyverno app | v1.18.1 |
| Helm chart | kyverno/kyverno 3.2.6 |
| Test namespace | ayoob-kyverno |
| Tester | Ayoob K Ibrahim |
| Date | 2026-06-08 |
| Result | PASS / FAIL |

---

## Install (`helmfile sync`)

| Check | Result | Notes |
|-------|--------|-------|
| `helmfile sync` succeeds | | |
| All pods Running in `kyverno` ns | | |
| Webhooks registered | | mutating + validating |
| CRDs present | | clusterpolicies.kyverno.io |
| Namespace exclusion | | `kyverno` only (`install/values.yaml`) |

---

## Baseline policies (Audit)

| Policy | Ready | Mode | Notes |
|--------|-------|------|-------|
| require-labels | | Audit | |
| require-requests-limits | | Audit | |
| disallow-privileged-containers | | Audit | |

---

## Demo tests

| Manifest | Expected (Audit) | PolicyReport | Result |
|----------|------------------|--------------|--------|
| demos/good-pod.yaml | Allowed | pass | |
| demos/bad-no-label.yaml | Allowed, logged | fail | |
| demos/bad-privileged.yaml | Allowed, logged | fail | |
| demos/bad-no-limits.yaml | Allowed, logged | fail | |

---

## Optional advanced tests

| Test | Result | Notes |
|------|--------|-------|
| add-team-label mutate | | team=devops on mutate-test |
| generate-configmap-on-ns | | ConfigMap on new Namespace |
| verify-image signed | | ghcr.io/kyverno/test-verify-image:signed |
| verify-image unsigned | | blocked in Enforce |

---

## Blockers

- 
- 

---

## Go / no-go for kubenine prod

| Area | Recommendation |
|------|----------------|
| Audit mode baseline | GO / NO-GO |
| Enforce mode | WAIT — burn-down required |
| verifyImages Enforce | NO-GO — needs Cosign in CI/CD |
| HA (3 admission replicas) | GO — configured in `install/values.yaml` |

**Overall:**
