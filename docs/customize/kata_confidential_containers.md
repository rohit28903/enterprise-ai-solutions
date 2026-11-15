# Kata Containers & Confidential Runtime — Quick Guide

[← Docs Index](../README.md)

Run pods in their own lightweight VM (Kata Containers) instead of sharing the host kernel, and optionally
in an Intel® TDX Trust Domain (TD), whose memory the host and cluster admin cannot read.

> [!NOTE]
> This guide covers the `kata_containers` infrastructure component. It is off by default
> (`kata_containers_enabled: false`). Workloads opt in per pod with `runtimeClassName`.

---

## 1. Runtimes

| RuntimeClass | VMM | Protects against | Needs |
|---|---|---|---|
| `kata-fc` | Firecracker | container escape to the host | KVM, devmapper (set up by the role) |
| `kata-qemu` | QEMU | container escape to the host | KVM |
| `kata-qemu-coco-dev` | QEMU | same as `kata-qemu`; runs the Confidential Containers flow (image pulled inside the guest) without TDX — for testing | KVM |
| `kata-qemu-tdx` | QEMU | escape **and** the host reading pod memory (one TD per pod) | KVM + TDX host (section 2) |

Each pod gets its own VM / TD, created when the pod starts and destroyed when it is deleted.
Firecracker has no TDX support; `kata-fc` stays isolation-only.

## 2. Host prerequisites for `kata-qemu-tdx`

| Layer | Requirement |
|---|---|
| CPU | Intel Xeon with TDX: 4th/5th Gen Xeon Scalable (TDX SKUs) or Xeon 6 |
| Memory | Every memory channel populated the same way |
| BIOS | VT-x; TME; TME-MT (memory integrity disabled); TDX; TDX SEAM Loader; TME-MT/TDX key split > 0; SGX (for attestation) |
| Host OS | Ubuntu 24.04 with the HWE kernel (installer-supported, tested). Ubuntu 25.10 / 26.04 have TDX host support built in, but the installer's kubespray supports Ubuntu 22.04 / 24.04 only, so use them with `existing_kubernetes`. Any kernel needs upstream KVM TDX support (Linux ≥ 6.16) |
| Kernel parameters | `nohibernate kvm_intel.tdx=1` |

BIOS menu location depends on the platform (Intel reference BIOS):

| Platform | Menu |
|---|---|
| Birch Stream, Oak Stream | EDKII Menu → Socket Configuration → **Security Configuration** |
| Eagle Stream | EDKII Menu → Socket Configuration → **Processor Configuration** |

Host OS, following [Ubuntu's Intel TDX guide](https://ubuntu.com/server/docs/how-to/virtualisation/intel-tdx/):

```bash
# Ubuntu 24.04: switch to the HWE kernel (Ubuntu 25.10 / 26.04 already have TDX host support)
sudo apt-get install -y linux-generic-hwe-24.04

# all releases: enable TDX in KVM
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 nohibernate kvm_intel.tdx=1"/' /etc/default/grub
sudo update-grub && sudo reboot
```

Kata ships its own QEMU and TDX firmware, so the host's `qemu-system-x86` / `ovmf-inteltdx` packages are not needed.

> [!WARNING]
> The PPA-based setup from the former canonical/tdx project installs a 6.8 `-intel` kernel on Ubuntu 24.04. It
> enables TDX but uses a pre-upstream KVM interface, and Kata's QEMU fails with `vm-type TDX not supported by KVM`.
> Its setup script also pins GRUB to that kernel; select the HWE kernel with `grub-set-default`.

Verify on each node:

```bash
sudo dmesg | grep -i tdx                     # "virt/tdx: module initialized"
cat /sys/module/kvm_intel/parameters/tdx     # Y
```

The role also checks that KVM offers the TDX VM type and leaves out nodes that do not.

## 3. Enable

In `global_config.yaml`:

```yaml
kata_containers_enabled: true
kata_runtime: "kata-qemu"                      # kata-fc | kata-qemu | both
kata_confidential_runtime: "kata-qemu-tdx"     # "" | kata-qemu-coco-dev | kata-qemu-tdx
```

Set `kata_confidential_runtime` before the cluster is created so the kubelet timeout below is applied.
Then install or update the component:

```bash
./es_auto_installer.sh install kata_containers --only
```

What the role does:

- Checks each node for KVM (and devmapper for `kata-fc`, TDX for `kata-qemu-tdx`); nodes without KVM are left out,
  and nodes failing the TDX checks don't get the TDX label.
- Labels every KVM node `ei-kata-eligible=true`. The TDX label `ei-tdx-eligible=true` is set by this role only on
  nodes that pass the TDX checks, and removed again from nodes that no longer do.
- Installs kata-deploy (Helm) with the selected shims and creates the RuntimeClasses; `kata-qemu-tdx`
  is scheduled only on `ei-tdx-eligible` nodes.
- Passes `https_proxy` / `no_proxy` to the guest agent, which pulls images inside the VM.
- Raises the kubelet `runtimeRequestTimeout` to 30 minutes (VM boot plus image pull) when the installer creates
  the cluster. It applies to every pod on the cluster and replaces any other `kubelet_config_extra_args` you set.
  On an existing cluster (the installer never changes it) set it in the kubelet configuration yourself; small images
  start well within the 2-minute default.
- Restricts the pod annotations containerd forwards to each Kata runtime (see below).

### Settings

| Setting | Default | Description |
|---|---|---|
| `kata_containers_enabled` | `false` | Install Kata Containers (kata-deploy Helm chart) |
| `kata_runtime` | `kata-fc` | Isolation runtime(s): `kata-fc` (Firecracker) \| `kata-qemu` \| `both` |
| `kata_confidential_runtime` | `""` | `""` (off) \| `kata-qemu-coco-dev` (the Confidential Containers flow in a normal VM: for testing, **no memory protection**) \| `kata-qemu-tdx` (Intel TDX Trust Domain). Installed alongside `kata_runtime`; an invalid value fails the install |
| `kata_version` | `4.2.0` | kata-deploy chart version |
| `kata_pod_annotations` | per runtime, see below | `io.katacontainers.*` pod annotations containerd forwards to each Kata runtime |
| `kata_tdx_node_label_key` | `ei-tdx-eligible` | Label set on nodes that pass the TDX checks; `kata-qemu-tdx` pods are scheduled only there |
| `kata_teardown_force` | `false` | Tear Kata down even while pods still run in Kata VMs (their VMs then keep running until stopped by hand) |

### Pod annotations

kata-deploy up to 4.2.0 tells containerd to forward every `io.katacontainers.*` pod annotation to the runtime,
so a pod author can override most of the Kata configuration and, with untrusted users, gain root on Kata nodes
([GHSA-37rf-pj8c-g5g3](https://github.com/kata-containers/kata-containers/security/advisories/GHSA-37rf-pj8c-g5g3)).
The role replaces that list per runtime with `kata_pod_annotations`:

| RuntimeClass | Forwarded annotations |
|---|---|
| `kata-fc`, `kata-qemu` | none |
| `kata-qemu-coco-dev`, `kata-qemu-tdx` | `io.katacontainers.config.hypervisor.cc_init_data`, `io.katacontainers.config.runtime.create_container_timeout` |

Extend a list only when a workload needs another annotation. `validate` fails if any node still forwards
all Kata annotations.

### Version

`kata_version` is `4.2.0`, the kata-deploy release Confidential Containers 0.23.0 is built on (its guest image
carries the image-pull, attestation and secret components). Move it together with the Confidential Containers
release it belongs to, so the host and guest components match.

### Upgrading an existing Kata install (kata-deploy 3.32)

- `install kata_containers` upgrades the kata-deploy Helm release in place. kata-deploy restarts containerd once on
  each Kata node; running pods keep running.
- Kata pods that are already running keep their VM on the old Kata version until they are recreated; new pods use
  4.2.0. (Tested: an Agent Sandbox Kata warm-pool pod kept running through the upgrade, and its replacement started
  on 4.2.0.)
- kata-deploy 4.2.0 also labels the nodes it installs on `kata-deploy.katacontainers.io/default=true`, and the
  `kata-qemu` RuntimeClass reserves 320Mi pod overhead.

## 4. Verify

```bash
./es_auto_installer.sh validate kata_containers --only
kubectl get runtimeclass
kubectl get nodes -l ei-tdx-eligible=true
```

A test pod in a TD:

```yaml
apiVersion: v1
kind: Pod
metadata: {name: td-test}
spec:
  runtimeClassName: kata-qemu-tdx
  containers:
  - name: c
    image: docker.io/library/busybox:1.37
    imagePullPolicy: Always
    command: ["sh", "-c", "dmesg | grep -i tdx; sleep 3600"]
    resources: {limits: {cpu: "1", memory: 512Mi}}
```

`kubectl logs td-test` shows `tdx: Guest detected`.

## 5. Limits

- **VM memory is fixed at start:** 2 GiB plus the pod's memory limit. Always set limits.
- **Per-pod cost:** each `kata-qemu-tdx` pod reserves 2 GiB and 1 CPU on top of its requests (RuntimeClass overhead).
- **TD count:** limited by the TDX key IDs set by the BIOS key split. kata-deploy accounts them as
  `tdx.intel.com/keys` only when Node Feature Discovery runs at install time. On a fresh install Kata (infrastructure
  layer) is installed **before** NFD (platform layer), so the count is missing and the scheduler can place **more TD
  pods on a node than it has key slots**; the extra ones can't start. Re-run `install kata_containers` after NFD is
  deployed: kata-deploy then creates its `kata-tee-keys` NodeFeatureRule.
- **The TDX label is a snapshot.** `ei-tdx-eligible` reflects the node when the install last ran. After changing a
  node (BIOS TDX setting, kernel), re-run `install kata_containers` so the label is set or removed.
- **Teardown** refuses while pods still run in Kata VMs (kata-deploy's cleanup doesn't stop running VMs). Remove those
  pods, and the components that recreate them (e.g. Agent Sandbox's Kata warm pool), first, or set
  `kata_teardown_force: true`.
- **Storage:** `emptyDir` volumes are encrypted block devices inside the guest (`emptydir_mode = "block-encrypted"`
  for the confidential runtimes): safe scratch space, lost when the pod stops. Volumes the host mounts — PVCs
  (NFS, NFS CSI, local-path), hostPath — are not shared into the guest: the pod sees an empty in-memory folder and
  nothing reaches the volume. Persistent confidential storage (CSI block volumes opened inside the guest) is
  still in progress upstream.
- **Images** must come from a registry, with `imagePullPolicy: Always`.
- **The image lives in VM memory (confidential runtimes).** With no shared filesystem, the guest unpacks the image
  into its own RAM: the Agent Sandbox image (163 MB compressed) takes about 530 MiB. Include it in the memory limit.
- **No `kubectl port-forward`** into Kata pods (kata-containers issue #1693).
- **Attestation** (proving a pod runs in a real TD before releasing secrets) is not set up by this component yet.

## 6. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Install fails "no cluster node exposes /dev/kvm" | VT-x off in BIOS, or a VM without nested virtualisation |
| Warns "kvm_intel tdx=Y but KVM does not offer the TDX VM type" | Host kernel too old (see the warning in section 2) |
| Warns "no node can run TDX guests" | TDX not enabled in BIOS, or `kvm_intel.tdx=1` missing from the kernel command line |
| `kata-qemu-tdx` pod stuck in `ContainerCreating`, containerd log `vm-type TDX not supported by KVM` | Host kernel too old (section 2) |
| Validate warns "no TDX-eligible node advertises tdx.intel.com/keys" | NFD was not running when kata-deploy was installed; re-run the install |
| Install warns "passed the TDX checks, but no Kubernetes node received ei-tdx-eligible" | Inventory host names don't match the Kubernetes node names |
| Teardown fails "pod(s) still run in Kata VMs" | Remove those pods (and their warm pool / controller) first, or set `kata_teardown_force: true` |
| After such a refused `teardown infrastructure`, the next install reports "a teardown of infrastructure did not finish" | The layer's deployment record was set to "deleting" before the Kata check refused, and nothing was removed. Today the check is advisory and the install goes ahead (and rewrites the record). If the layer declares upgrade support, it refuses the install: remove the Kata pods and finish the teardown, or install with `--force` to keep Kata |
| Validate fails "Check containerd does not forward all Kata pod annotations" | The annotation drop-in was not applied; re-run the install and check `containerd config dump` on the node |
| Data written to a PVC from a confidential pod is missing on the volume | Host-mounted volumes are not shared into the guest; use `emptyDir` for scratch data |
