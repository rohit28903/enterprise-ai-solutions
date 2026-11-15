# Confidential Containers Contract — Quick Guide

[← Docs Index](../README.md)

The `confidential_containers` platform component publishes everything a workload needs to run its pods in a
confidential VM (an Intel® TDX Trust Domain, or the `kata-qemu-coco-dev` test VM) as **cluster objects**:

| Object | Content |
|---|---|
| RuntimeClass (from `kata_containers`) | `kata-qemu-tdx` or `kata-qemu-coco-dev` |
| ConfigMap `kube-system/confidential-runtime` | `runtimeClass`, `initdata` (gzip + base64), `initdataDigest` (sha384), `kbsAddress`, `namespaceLabel`, `allowedKataAnnotations`, `createContainerTimeout` |
| Kyverno `ValidatingPolicy` `confidential-runtime` | Pod CREATE rules for namespaces labelled `confidential.intel.com/enforce=true` (also enforced natively as ValidatingAdmissionPolicy `vpol-confidential-runtime`) |
| Kyverno `ValidatingPolicy` `confidential-runtime-immutable` | Pod UPDATE rules there: Kata annotations can't change, no `kubectl debug` (admission only) |

Workloads (for example the Agent Sandbox TDX template) label their namespace and copy `runtimeClass` and
`initdata` from the ConfigMap into their pods; they do not depend on files in this repository.

## Enable

Requires [Kata & Confidential Runtime](kata_confidential_containers.md) with `kata_confidential_runtime` set, and
the [Kyverno](kyverno.md) component. In `global_config.yaml`:

```yaml
kata_containers_enabled: true
kata_confidential_runtime: "kata-qemu-tdx"      # or kata-qemu-coco-dev for testing
kyverno_enabled: true
confidential_containers_enabled: true
```

```bash
./es_auto_installer.sh install kyverno --only
./es_auto_installer.sh install confidential_containers --only
./es_auto_installer.sh validate confidential_containers --only
```

## Settings

| Setting | Default | Description |
|---|---|---|
| `confidential_containers_enabled` | `false` | Publish the contract. Needs `kata_confidential_runtime` set and `kyverno_enabled: true` |
| `cc_namespace_label_key` | `confidential.intel.com/enforce` | Namespace label (value `"true"`) that opts a namespace in |
| `cc_kbs_address` | `kbs_address`, else `""` | Trustee KBS URL. Empty: initdata carries only the agent policy (no attestation settings) |
| `cc_agent_policy_denied` | `ExecProcessRequest`, `SetPolicyRequest`, `ReadStreamRequest`, `GetDiagnosticDataRequest` | Kata agent requests the TD refuses |
| `cc_allowed_kata_annotations` | `…hypervisor.cc_init_data`, `…runtime.create_container_timeout` | The only `io.katacontainers.*` annotations a confidential pod may carry (matches the containerd allowlist) |
| `cc_allowed_volume_types` | `emptyDir`, `configMap`, `secret`, `projected`, `downwardAPI` | Volume types a confidential pod may use |
| `cc_create_container_timeout` | `"1800"` | Seconds; published in the ConfigMap for workloads |
| `cc_validate_smoke_test` | `true` | `validate` dry-runs approved and non-compliant pods |

The ConfigMap name and namespace (`cc_configmap_name`, `cc_configmap_namespace`) and the policy name
(`cc_policy_name`) are part of the contract other components read; don't change them.

## initdata

`initdata` is a small TOML document handed to the confidential VM at boot through the pod annotation
`io.katacontainers.config.hypervisor.cc_init_data`. In a TD its sha384 digest is part of the attestation evidence
(MRCONFIGID), so a verifier can tell whether it was changed. It carries:

- `policy.rego` — the Kata agent policy. Every agent request is allowed except those in
  `cc_agent_policy_denied`: `ExecProcessRequest` (`kubectl exec`), `SetPolicyRequest` (swapping the policy),
  `ReadStreamRequest` (reading container output: **`kubectl logs` is empty**) and `GetDiagnosticDataRequest`.
  Remove `ReadStreamRequest` from the list to allow logs.
- `aa.toml` / `cdh.toml` — the Trustee KBS address for attestation and sealed secrets, only when `cc_kbs_address`
  is set (not configured by this release).

## The policy

In namespaces labelled `confidential.intel.com/enforce=true`, a pod is admitted only when:

1. `runtimeClassName` is the configured confidential runtime;
2. its `cc_init_data` annotation equals the approved initdata;
3. it carries no other `io.katacontainers.*` annotation than `cc_allowed_kata_annotations`;
4. every volume is `emptyDir`, `configMap`, `secret`, `projected` or `downwardAPI` — host-mounted volumes (PVC,
   hostPath, NFS, CSI) are not shared into the TD; `emptyDir` is an encrypted disk inside it.
5. every container and init container uses `imagePullPolicy: Always` (the TD pulls the image itself).

Afterwards:

- `io.katacontainers.*` annotations of a running pod cannot be changed (they are read only when the TD starts, so a
  changed value would only mislead anyone auditing the pod);
- ephemeral containers (`kubectl debug`) are rejected.

**Both policies are also enforced as native Kubernetes ValidatingAdmissionPolicies** (Kyverno generates
`vpol-confidential-runtime` and `vpol-confidential-runtime-immutable`). The API server evaluates them itself, so they
keep working while Kyverno is down. Kyverno removes its own webhooks when its admission controller stops, so
webhook-only policies would let pods through during a Kyverno outage or restart.

Pod controllers (Deployment, StatefulSet, DaemonSet, Job, CronJob, ReplicaSet) are **not** checked themselves: a
non-compliant controller is accepted, and its pods are rejected when they are created. Look for `FailedCreate` events
on the ReplicaSet / Job.

The UPDATE policy is only evaluated when a pod's `io.katacontainers.*` annotations change or an ephemeral container is
added (`matchConditions`), so ordinary pod updates and pod deletion aren't affected.

Only pods in opted-in namespaces are checked; other namespaces are not affected. Opt a namespace
in with:

```bash
kubectl label namespace <ns> confidential.intel.com/enforce=true
```

`validate` checks the ConfigMap, both policies and both native ValidatingAdmissionPolicies with their `Deny`
bindings. It then sends dry-run pods (checked, never created) to a throw-away opted-in namespace: the approved pod must
be admitted, and a runc pod, wrong initdata, an extra Kata annotation, a hostPath volume and `imagePullPolicy:
IfNotPresent` must be rejected **by the native policy**.

## Using the contract in a workload

A workload that runs its pods in confidential VMs:

1. Labels its namespace `confidential.intel.com/enforce=true`.
2. Reads `runtimeClass`, `initdata` and `createContainerTimeout` from ConfigMap `kube-system/confidential-runtime`,
   and puts them into its pod template:

```yaml
metadata:
  annotations:
    io.katacontainers.config.hypervisor.cc_init_data: "<initdata from the ConfigMap>"
    io.katacontainers.config.runtime.create_container_timeout: "<createContainerTimeout>"
spec:
  runtimeClassName: <runtimeClass from the ConfigMap>   # kata-qemu-tdx (or kata-qemu-coco-dev)
  containers:
  - name: app
    image: registry.example.com/app:1.0
    imagePullPolicy: Always                           # required
    resources: {limits: {cpu: "1", memory: 1Gi}}      # TD memory = 2 GiB + this limit; include the unpacked image
    readinessProbe: {httpGet: {path: /healthz, port: 8888}}   # no exec probes
  volumes:
  - name: scratch
    emptyDir: {sizeLimit: 1Gi}                        # encrypted inside the TD; no PVC / hostPath
```

3. Reads the values at install time and re-reads them after `install confidential_containers` runs again (the
   initdata only changes when its settings change).

## Verify

```bash
./es_auto_installer.sh validate confidential_containers --only
kubectl -n kube-system get configmap confidential-runtime -o jsonpath='{.data.runtimeClass} {.data.initdataDigest}'
kubectl get validatingpolicies confidential-runtime confidential-runtime-immutable
kubectl get validatingadmissionpolicies vpol-confidential-runtime vpol-confidential-runtime-immutable
# in an approved pod: kubectl exec fails with "ExecProcessRequest is blocked by policy"
```

## Limits

- **Kyverno is a guardrail, not the control against the admin** — an admin can delete the policy. The control that
  holds is attestation: Trustee/KBS releases secrets only to a TD whose initdata digest matches (a later release).
- **The agent policy limits what reaches running containers, not which containers run.** It blocks exec, logs,
  policy swap and diagnostics, but allows every other agent request, including creating containers. Those
  requests come from the Kata shim on the (untrusted) host, so a hostile host or admin could start an
  *additional* container of its choice inside the TD. Blocking exec stops casual inspection, not a determined
  admin. Attestation proves which policy a TD has, not that it is strict. A strict per-workload policy (Kata
  `genpolicy`, pinning images, commands and mounts) is the fix; it is under discussion and not part of this release.
- **Images:** for the containers in the pod spec, the TD pulls the image itself and runs only its own copy, so the
  host cannot swap that image. Kubernetes still pulls the image on the host as well, so the host can *read* it; use
  encrypted images for confidential image content (needs KBS).
- **Probes:** `exec` liveness/readiness probes always fail (they use the blocked exec request). Use `httpGet` or
  `tcpSocket` probes.
- **The namespace label is the switch.** Anyone allowed to edit namespace labels can turn enforcement off, and pods
  created before the label was set are not re-checked.
- **Memory:** the TD keeps the unpacked image in its own RAM. Size pod memory limits for the image plus the
  workload (the TD has 2 GiB plus the pod's memory limit).
- **No `kubectl exec` or `kubectl logs`** with the default agent policy; workloads must expose what they need
  (for example the Agent Sandbox HTTP API).

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Install fails "kata_confidential_runtime is empty" | Set `kata_confidential_runtime` and install `kata_containers` first |
| Install fails "The confidential-runtime policy needs Kyverno" | Set `kyverno_enabled: true` and install `kyverno` |
| Install waits at "native ValidatingAdmissionPolicy bindings" | Kyverno isn't generating native policies: check `features.generateValidatingAdmissionPolicy` and that the API server serves `admissionregistration.k8s.io/v1` ValidatingAdmissionPolicy (Kubernetes 1.30 or later) |
| Pod rejected "must carry the approved initdata" | The pod's `cc_init_data` doesn't match the ConfigMap. Copy it again after the contract was reinstalled with changed settings |
| Pod rejected "must use imagePullPolicy: Always" | Set it on every container and init container (images without a tag default to `Always`) |
| Deployment shows `FailedCreate`, 0 pods | Its pod template breaks the contract; the event names the rule |
| `kubectl exec` / `kubectl logs` don't work | Expected with the default agent policy; use the workload's own API |
