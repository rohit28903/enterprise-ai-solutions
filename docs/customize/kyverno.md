# Kyverno Policy Engine — Quick Guide

[← Docs Index](../README.md)

[Kyverno](https://kyverno.io/) checks objects before the Kubernetes API server accepts them, and can reject or
change them. The platform installs it as an optional component; other components ship their rules as Kyverno
policies (for example `confidential_containers`, which only lets confidential pods into opted-in namespaces).

> [!NOTE]
> Off by default (`kyverno_enabled: false`). Enable it when a component that needs it is enabled.

## Enable

In `global_config.yaml`:

```yaml
kyverno_enabled: true
```

```bash
./es_auto_installer.sh install kyverno --only
./es_auto_installer.sh validate kyverno --only
```

## Settings

| Setting | Default | Description |
|---|---|---|
| `kyverno_enabled` | `false` | Install Kyverno |
| `kyverno_version` | `3.9.2` | Helm chart version (Kyverno v1.19.2) |
| `kyverno_admission_replicas` | `1` | Admission controller replicas; use `3` in production |
| `kyverno_webhook_excluded_namespaces` | `[kube-system]` | Namespaces Kyverno's webhooks never see (the `kyverno` namespace is always excluded) |
| `kyverno_validate_smoke_test` | `true` | `validate` applies a throw-away policy and dry-runs two pods against it |
| `kyverno_teardown_delete_crds` | `true` | Teardown deletes the CRDs this release installed, and with them every Kyverno policy |
| `kyverno_teardown_force` | `false` | Tear Kyverno down even while platform policies exist |

## What is installed

| Item | Value |
|---|---|
| Chart | `kyverno/kyverno` `3.9.2` (Kyverno v1.19.2), release `kyverno` in namespace `kyverno` |
| Controllers | admission, background, cleanup, reports (`kyverno_admission_replicas`: 1; use 3 for HA) |
| Pod Security | namespace labelled `restricted` when `enforce_pss` is on |
| Webhooks | never see `kube-system` or the `kyverno` namespace (`kyverno_webhook_excluded_namespaces`) |
| Policy types | CEL policies in `policies.kyverno.io/v1`: `ValidatingPolicy`, `MutatingPolicy`. The `kyverno.io` `ClusterPolicy` type is deprecated |
| Native policies | `features.generateValidatingAdmissionPolicy` is on: a `ValidatingPolicy` that asks for it is also created as a Kubernetes `ValidatingAdmissionPolicy` |

Kyverno only intercepts the resources its policies select; with no policies installed it does not affect
other workloads.

**When Kyverno is down, its webhook policies are not enforced.** Kyverno removes its own webhook configurations
when its admission controller stops (seen on Kyverno 1.19.2), so during an outage or a restart, requests that
webhook-only policies would reject are admitted. Policies that must always hold should also be enforced as native
Kubernetes ValidatingAdmissionPolicies (`spec.autogen.validatingAdmissionPolicy.enabled: true`), which the API
server evaluates itself; `confidential_containers` does this. Run 3 admission replicas
(`kyverno_admission_replicas: 3`) in production to keep restarts short.

`validate` checks the release and controllers, then applies a throw-away `ValidatingPolicy` and sends a pod as a
server-side dry run: the policy must reject it (the pod is never created). Set
`kyverno_validate_smoke_test: false` to skip this.

## Writing platform policies

For a component that ships its own Kyverno policies (as `confidential_containers` does):

- Use CEL `ValidatingPolicy` / `MutatingPolicy` (`policies.kyverno.io/v1`).
- Label them `app.kubernetes.io/managed-by: ai-solutions`, so a Kyverno teardown refuses while they exist.
- For rules that must hold even while Kyverno is down, set `spec.autogen.validatingAdmissionPolicy.enabled: true`.
  Kyverno then also creates them as native `ValidatingAdmissionPolicy` objects, which the API server evaluates itself
  (Kubernetes 1.30 or later). Kyverno only does this when it doesn't also generate pod-controller rules for the
  policy, so set `spec.autogen.podControllers.controllers: []`.
- Match only the namespaces and operations the rule needs (a `namespaceSelector`, CREATE where UPDATE isn't needed,
  `matchConditions` to skip irrelevant requests): with `failurePolicy: Fail`, whatever the policy matches depends on
  the policy being evaluable.

## Teardown

```bash
./es_auto_installer.sh teardown kyverno --only
```

Teardown first **refuses** while platform policies (label `app.kubernetes.io/managed-by=ai-solutions`, e.g. from
`confidential_containers`) still exist, because removing Kyverno deletes them and silently switches that protection
off. Tear those components down first (a full platform teardown does this in the right order), or set
`kyverno_teardown_force: true`.

It then removes the release, the namespace and (with `kyverno_teardown_delete_crds: true`, the default) the CRDs
**this Helm release installed**, deleting every Kyverno policy. Report CRDs another tool installed
(`wgpolicyk8s.io`, `openreports.io`) are left alone. Finally it deletes any Kyverno webhook configuration left behind:
a webhook without Kyverno running behind it could block object creation in the cluster.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `validate` fails at "Dry-run a pod the policy must reject" | Kyverno's admission controller isn't serving requests: check `kubectl -n kyverno get pods` and its logs |
| Teardown fails "Kyverno still enforces platform policies (…)" | Tear down the components that own those policies first (e.g. `confidential_containers`), or set `kyverno_teardown_force: true` |
| A policy is not enforced while Kyverno restarts | Expected for webhook-only policies (Kyverno removes its webhooks when it stops). Generate a native ValidatingAdmissionPolicy for rules that must always hold (see above) |
| `kubectl get validatingadmissionpolicies` doesn't show a policy that asks for one | Its `ValidatingPolicy` status says why (`kubectl get validatingpolicy <name> -o yaml`): e.g. pod-controller autogen still enabled |

## Limits

- Kyverno is a guardrail, not a security boundary against a cluster admin: an admin can delete policies or
  Kyverno itself. Controls that must hold against the admin (for example releasing secrets only to an attested
  Trust Domain) live elsewhere (Trustee / KBS).
