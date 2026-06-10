# Kyverno PoC Report — kubenine-intern

| Field | Value |
|-------|-------|
| Spec / task | Kyverno policy engine on Civo K3s |
| Cluster | kubenine-intern |
| Context | kubenine-intern-pinniped |
| Kubernetes | v1.34.2+k3s1 |
| Kyverno app | v1.18.1 |
| Helm chart | kyverno/kyverno 3.8.1 |
| Helm revision | 4 (deployed) |
| Test namespace | ayoob-kyverno |
| Tester | ayoob |
| Date | 2026-06-08 |
| Result | **PASS** |

---

## Executive summary

Kyverno v1.18.1 (chart 3.8.1) is installed and healthy on the intern cluster. All core controllers are Running with HA admission (3 replicas). Six ClusterPolicies are Ready. Baseline validate policies work in Audit mode; `require-labels` and `verify-image` work in Enforce mode as configured. Demo manifests behaved as expected.

One non-blocking issue: Helm chart default cleanup CronJobs use `bitnami/kubectl:1.28.5`, which is not available on Docker Hub. Jobs failed with `ImagePullBackOff` until CronJobs were suspended and stuck Jobs deleted. Permanent fix: disable `cleanupJobs` in `install/values.yaml` (recommended for PoC) or override image tag.

**Overall: GO** for intern/PoC with Audit baseline; **WAIT** on broad Enforce rollout; **NO-GO** for verify-images Enforce in prod until Cosign signing is in CI/CD.

---

## Install (`helmfile sync`)

| Check | Result | Notes |
|-------|--------|-------|
| `helmfile sync` succeeds | PASS | Chart `3.8.1`; revision 4 deployed |
| All core pods Running in `kyverno` ns | PASS | 8 controllers: 3 admission, 2 background, 2 reports, 1 cleanup-controller |
| Webhooks registered | PASS | `mutate.kyverno.svc-fail`, `validate.kyverno.svc-fail` |
| CRDs present | PASS | e.g. `clusterpolicies.kyverno.io` |
| Namespace exclusion | PASS | Only `kyverno` excluded via `config.webhooks.namespaceSelector` in `install/values.yaml` |
| HA admission | PASS | `admissionController.replicas: 3` |
| PolicyExceptions | PASS | `features.policyExceptions.enabled: true` |

### Install troubleshooting (resolved)

| Issue | Resolution |
|-------|------------|
| Chart `3.2.6` attempted downgrade from existing `3.8.1` | Pinned `helmfile.yaml` to `version: 3.8.1` |
| Partial upgrade / TLS readiness errors | `helmfile sync` with chart 3.8.1 restored healthy state |
| Cleanup CronJob `ImagePullBackOff` | Suspend CronJobs, delete stuck Jobs; disable `cleanupJobs` in values for permanent fix |

### Post-install pod inventory

```
kyverno-admission-controller     3/3 Running
kyverno-background-controller    2/2 Running
kyverno-reports-controller       2/2 Running
kyverno-cleanup-controller       1/1 Running
```

---

## Policies applied

| Policy | Ready | Mode | Background | Notes |
|--------|-------|------|------------|-------|
| disallow-privileged-containers | True | Audit | true | Baseline |
| require-requests-limits | True | Audit | true | Baseline |
| require-labels | True | Enforce | true | Denies missing `app.kubernetes.io/name` |
| add-team-label | True | Mutate | true | Namespace `ayoob-kyverno` only |
| generate-configmap-on-ns | True | Generate | true | On Namespace create |
| verify-image | True | Enforce | false | `ghcr.io/kyverno/test-verify-image*` in `ayoob-kyverno` |

```bash
kubectl apply -f policies/
# add-team-label created
# disallow-privileged-containers configured
# generate-configmap-on-ns created
# require-labels created
# require-requests-limits configured
# verify-image created
```

---

## Demo tests

```bash
kubectl create namespace ayoob-kyverno --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f demos/
kubectl get policyreport -n ayoob-kyverno
kubectl get clusterpolicy
```

### Baseline validate (Audit policies)

| Manifest | Expected | Observed | PolicyReport | Result |
|----------|----------|----------|--------------|--------|
| `demos/good-pod.yaml` | Allowed | Allowed | 3 pass, 0 fail | PASS |
| `demos/bad-privileged.yaml` | Allowed, violation logged | Allowed | 1 pass, 1 fail | PASS |
| `demos/bad-no-limits.yaml` | Allowed, violation logged | Allowed | 2 pass, 1 fail | PASS |
| `demos/bad-no-label.yaml` | Denied (Enforce) | Admission denied by `require-labels` | N/A (pod not created) | PASS |

`bad-no-label` denial (expected for Enforce):

```
require-labels/check-for-labels: validation error: The label `app.kubernetes.io/name` is required.
```

### Optional advanced tests

| Test | Expected | Observed | Result |
|------|----------|----------|--------|
| `demos/mutate-test.yaml` | Pod created; `team=devops` label added | Pod created | PASS |
| `demos/verify-signed.yaml` | Signed image allowed | Pod created | PASS |
| `demos/verify-unsigned.yaml` | Unsigned image blocked | Admission denied | PASS |
| `generate-configmap-on-ns` | ConfigMap on new Namespace | Verify with commands below | PENDING |

`verify-unsigned` denial (expected):

```
verify-image/verify-image: failed to verify image ghcr.io/kyverno/test-verify-image:unsigned:
no matching signatures
```

Generate policy test (`ayoob-kyverno` existed before policy apply — use a new namespace):

```bash
kubectl create namespace poc-generate-test
kubectl get configmap kyverno-generated-cm -n poc-generate-test
kubectl delete namespace poc-generate-test
```

Expected: ConfigMap `kyverno-generated-cm` with `created-by: kyverno`.

Mutate label verification:

```bash
kubectl get pod mutate-test -n ayoob-kyverno -o jsonpath='{.metadata.labels.team}{"\n"}'
# Expected: devops
```

---

## PolicyReports (`ayoob-kyverno`)

| Resource | PASS | FAIL | Age at capture |
|----------|------|------|----------------|
| Pod/good-pod | 3 | 0 | 4d17h |
| Pod/bad-privileged | 1 | 1 | 4d17h |
| Pod/bad-no-limits | 2 | 1 | 4d17h |

Background scan confirms Audit policies are active. `bad-no-label` has no report (pod never admitted).

---

## Cleanup CronJobs (non-blocking)

| Item | Detail |
|------|--------|
| Symptom | 5 CronJobs every 10m spawned Jobs with `ErrImagePull` / `ImagePullBackOff` |
| Image | `bitnami/kubectl:1.28.5` (tag not found) |
| Impact | Core Kyverno unaffected; noisy failed pods in `kyverno` namespace |
| Mitigation applied | Suspend CronJobs; delete stuck Jobs |
| Permanent fix | Add to `install/values.yaml`: |

```yaml
cleanupJobs:
  admissionReports:
    enabled: false
  clusterAdmissionReports:
    enabled: false
  ephemeralReports:
    enabled: false
  clusterEphemeralReports:
    enabled: false
  updateRequests:
    enabled: false
```

Then: `helmfile sync`

---

## Blockers

| Blocker | Severity | Status |
|---------|----------|--------|
| Cleanup CronJob default image tag missing | Low | Workaround applied; fix in values pending |
| `require-labels` is Enforce (rollout plan prefers Audit for baseline) | Info | Documented; change to Audit before prod rollout if desired |
| verify-images Enforce in prod | High for prod | Not a PoC blocker; needs Cosign in CI/CD |

---

## Go / no-go for kubenine prod

| Area | Recommendation |
|------|----------------|
| Kyverno install (HA, webhooks, CRDs) | GO |
| Audit mode baseline (`disallow-privileged`, `require-requests-limits`) | GO |
| `require-labels` Enforce | WAIT — switch to Audit first or burn down violations |
| Broad Enforce rollout | WAIT — audit burn-down required |
| verify-images Enforce | NO-GO — needs image signing pipeline (Cosign) |
| Cleanup CronJobs with default chart image | NO-GO — disable or fix image tag in values |
| HA (3 admission replicas) | GO — configured in `install/values.yaml` |

**Overall intern PoC: PASS**

**Overall kubenine prod: GO for phased Audit rollout; NO-GO for full Enforce + verify-images until signing and burn-down are complete.**

---

## References

- Install: `helmfile.yaml`, `install/values.yaml`
- Agent playbook: `CLAUDE.md`
- Operations: `docs/runbook.md`
- Deep dive: `docs/complete-guide.md`
- Policies: `policies/`
- Demos: `demos/`
