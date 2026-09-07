# Kyverno

Kyverno evaluates Kubernetes resources against policies written as Kubernetes
YAML. It is installed before the application so the same cluster-wide
guardrails apply whether workloads are deployed with plain manifests, Helm or
ArgoCD.

The starter policies in [`policies/`](./policies/) are prepared for a staged
rollout and check that:

- application containers define CPU and memory requests and limits;
- privileged containers are not used;
- application images have an explicit tag other than `latest`.

All three policies use `Audit` mode and `background: false`. When they are
eventually applied, they evaluate only new admission requests in `staging` and
`production`; they do not scan the existing cluster. The policies are not
applied during the initial installation below. This lets us observe Kyverno
itself before adding policy evaluation.

The initial chart configuration uses one replica per controller and disables
background scans, admission reports, and the reports controller. The optional
external reports server also remains disabled. This avoids creating a large
PolicyReport watch/write footprint while the API server behavior is being
measured. Reporting or additional replicas can be enabled later with separate,
deliberate changes.

## Install Kyverno

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update kyverno

helm upgrade --install kyverno kyverno/kyverno \
  --version 3.9.0 \
  --namespace kyverno \
  --create-namespace \
  --values 4-Deploy-App/kyverno/values.yaml \
  --wait
```

The first install intentionally does not apply the policy directory. Verify
that the controllers are healthy and that the API server remains stable:

```bash
kubectl get deployments -n kyverno
kubectl get pods -n kyverno -o wide
kubectl get events -n kyverno --sort-by=.lastTimestamp
```

After this observation period, apply one narrow policy at a time:

```bash
kubectl apply -f 4-Deploy-App/kyverno/policies/disallow-latest-tag.yaml
```

Do not apply every policy as one batch during the first rollout. Check the
admission latency and API server metrics between each policy. The other starter
policies can then be applied individually.

## Verify the reports

```bash
kubectl get pods -n kyverno
kubectl get validatingpolicy
```

Policy reports are intentionally unavailable in this first phase. Enable the
reports controller and the report features only when report data is required.

Inspect the violations reported for an individual namespace:

```bash
kubectl describe policyreport -n production
```

When a policy is clean and should become a hard admission guardrail, change
only that policy's `validationActions` entry from `Audit` to `Deny` and apply
it again. Because `background: false`, existing resources are not retroactively
scanned; only new or updated non-compliant Pods are evaluated at admission.

## References

- [Kyverno installation](https://kyverno.io/docs/installation/installation/)
- [Policy reports](https://kyverno.io/docs/policy-reports/)
- [CEL-based ValidatingPolicy](https://kyverno.io/docs/policy-types/validating-policy/)
- [Kyverno sample policy library](https://kyverno.io/policies/)
