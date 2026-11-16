Q1: We have this operator from OPI called dpu-operator (https://github.com/opiproject/dpu-operator) which support Intel and Marvell. We wanted to enhance it to support NVIDIA. Please refer to the design in @opi-dpf-architecture-review.html . An adapter was developed to support NVIDIA - you can find the adapter code in @opi-nvidia-dpf-adapter . I want you to review the adapter and say if we (OPI project) can claim the NVIDIA support - similar to Intel and Marvell. By similar I mean same features as Intel and Marvell.

**No. The adapter doesn't give NVIDIA the same features as Intel and Marvell.** It has a working lifecycle and provisioning path for BlueField-3 through DPF, proven on one lab host. The Intel/Marvell data path (VF plumbing, network-function steering) is missing, and the parts that look like parity depend on unmerged OPI changes.

I compared the adapter against upstream `opiproject/dpu-operator` `main` (commit `3092bcb`). I read the code, docs and the lab snapshot. I did not run the Go tests, because the Go toolchain wasn't installed.

## What Intel and Marvell do

- The daemon detects the DPU by PCI ID and creates a `DataProcessingUnit` for both the host and DPU side.
- A per-vendor VSP implements `Init`, `GetDevices`, `SetNumVfs`, `CreateBridgePort`/`DeleteBridgePort` and `CreateNetworkFunction`/`DeleteNetworkFunction`. The Marvell VSP, for example, creates SR-IOV VFs, adds VFs to an OVS bridge and installs VF→NF flow rules.
- The real daemon runs the device plugin (`openshift.io/dpu`), the CNI server and the SFC reconciler. An SFC with an `image` creates an NF pod on the DPU.
- Upstream e2e covers pod→pod through an NF, pod→NF, NF→external, pod→external and SFC delete cleanup.

## Where NVIDIA falls short

| Area | Intel / Marvell | NVIDIA adapter |
|---|---|---|
| VSP data path | Real VF and bridge-port work | `BridgePortService` is not registered anywhere in the adapter. `SetNumVfs` is a no-op echo. `CreateNetworkFunction` only checks that a `DPUService` exists. The adapter's own `docs/cleanup-backlog.md` says these "return success without performing work". |
| Daemon | Real `dpu-daemon` (device plugin, CNI, SFC reconciler) | In the lab snapshot the `dpu-daemon` DaemonSet has 0 pods scheduled. The adapter ships a stand-in `opi-daemon` that only does `Init`/`GetDevices` and writes `Ready`. |
| DPU detection | Auto-detected, both sides | Upstream has no NVIDIA detector. The `DataProcessingUnit` is applied by hand with a hand-edited `nodeName`, host side only. The `discovery.opiproject.org/product-name` annotation is invented by the adapter, not upstream. |
| SFC semantics | `image` creates an NF pod | The mapping skips image-only network functions. It needs a Helm `chart` field that exists only in unmerged PR #8, so on upstream `main` an SFC produces nothing. |
| SFC mapping content | Vendor-neutral | `config/mappings/servicefunctionchain.yaml` hard-codes the HBN NAD, `nvidia.com/bf_sf: "4"` and 6Gi memory into every network function. It is effectively HBN-only. |
| Traffic proof | 5 traffic scenarios in e2e | The lab ping test used hand-applied DPF objects (`07`/`08` samples) and NVIDIA's own SR-IOV device plugin (`nvidia.com/mlnx_sriov`), not OPI's device plugin and CNI. The OPI SFC lab run only shows DPF children reaching `Ready`. No traffic test goes through OPI APIs. |
| API dependencies | `main` as shipped | The adapter needs `DpuOperatorConfig.spec.vendor` (with an `nvidia\|amd` enum), a `VendorProvider` CRD, and SFC `chart`. None are on `main`. They live on the `feat/vendor-provider-spi` and `feat/sfc-helm-chart` branches. |
| Scope and maturity | Shipping hardware, CI | BlueField-3 only (BF-2 is observe-only), DPF 26.4 only, Kubernetes 1.36 only. It needs cert-manager, Argo CD, Multus and Kamaji. It was tested on a single-node k3s lab, not OpenShift. It needs a manual kubelet-config strip after Create. Its own checklist leaves image tags, docs and the support matrix as Pending. |

The architecture review recommended the hybrid design mainly because the NVIDIA VSP would satisfy "the same per-node gRPC contract Intel and Marvell use". The adapter implements only the handshake half of that contract, so the main parity argument isn't met yet. The review's API section also cites `spec.mode` and `DpuNetwork`, which I couldn't find in `opiproject/dpu-operator` `main`, so it may describe a different fork.

## What OPI can claim today

Something like "experimental: BlueField-3 DPU provisioning and DPF service delivery driven from OPI CRs, via an out-of-tree companion, validated on a single lab host." I would not claim "NVIDIA supported like Intel and Marvell".

## What parity would need

1. Land the core API changes on `main`: vendor selection, `VendorProvider`, and a decision on SFC `chart` versus `image`.
2. Add an NVIDIA detector in `internal/platform` so `DataProcessingUnit` objects are auto-created for host and DPU side.
3. Run the real `dpu-daemon` with the NVIDIA VSP, and implement `BridgePortService`, `SetNumVfs` and NF create/delete. These could map to DPF `DPUServiceInterface` (type `vf`) and `DPUServiceChain`, with VFs exposed through `openshift.io/dpu`.
4. Make `image`-based SFCs work, or explicitly document the chart-only difference. Remove the HBN-specific values from the generic mapping.
5. Port the upstream e2e traffic scenarios to run on BlueField hardware through OPI CRs only, on OpenShift, with CI.
6. Finish the support matrix, versioned images and docs.

If you'd rather define parity as "DPF does the work, OPI owns the intent", the TSC would need to approve that explicitly, with a different support statement and tests. I can write this up as a TSC-ready gap analysis, or start on the VSP data-path work (item 3).

Q2 Lets focus on your "What parity would need"
Assume the PRs 7 and 8 are merged into dpu-operator (it will be). So, this is not an issue.
We will come back to 2 and 3.
The 4 - I don't agree the first part. Its ok to add helm. The second part on the number of VF - I agree.
5 it not really critical for now.

Now for 2 and 3 give me a detailed plan considering the important requirement - We should not reinvent the wheel (use DPF operator AS IS) transparently.

Q2. Give me a plan for 2 and 3.
Refer to the HTML

Q3. Few questions:
For M-A, M-B and M-C, the changes will be on dpu-operator or the adapter or both? Can you give a breakup? and why?
What do you mean by spike checklist?

M-A, M-B, and M-C each touch both repositories, but for different reasons. `dpu-operator` only changes where NVIDIA has to plug into the existing Intel and Marvell seam, or where the current daemon would damage a DPU that DPF already owns. Everything that knows about DPF stays in the adapter, so DPF itself is never modified and never imported into `dpu-operator`.

## M-A — detection and the host-side DataProcessingUnit

| Repo | Change | Why it lives there |
|---|---|---|
| `dpu-operator` | NVIDIA detector (`15b3` / `a2dc` / `a2dd`, serial de-dupe, host side only) | Detection and `DataProcessingUnit` creation are the daemon's job. Intel and Marvell already do this in `internal/platform`. |
| `dpu-operator` | VSP pod template, RBAC, and the `nvidia_bluefield` image key | The `DataProcessingUnit` controller renders one VSP pod per detected DPU from `bindata/vsp/<vendor>`. The adapter should not deploy its own DaemonSet alongside that. |
| `dpu-operator` | Status update writes only the `Ready` condition | The daemon currently replaces the whole status and would erase `DPFReady` and `DPFHardwareCompatible`. This is a daemon bug that affects every vendor. |
| `dpu-operator` | Do not delete `DataProcessingUnit` objects on a graceful daemon shutdown | Today a daemon restart deletes those objects. The adapter's DPF children follow them, which would reflash the BlueField. |
| Adapter | `Init` waits until the DPF `DPU` is `Ready`, then returns the VSP's own localhost address | This is DPF-specific readiness. The daemon already retries `Init`, so no daemon change is needed. |
| Adapter | Heartbeat on the OPI lifecycle service, healthy only while the DPF `DPU` and its Kamaji node are ready | The daemon's `Ready` condition is driven by that ping. |
| Adapter | Mapping gated on `NVIDIA BlueField-3` and `isDpuSide: false`; provisioning objects tracked by annotation with delayed delete | The translator owns the DPF objects. Owner references from a `DataProcessingUnit` that the daemon may recreate would delete a live DPU. |
| Adapter | Drop `cmd/opi-daemon`, `ReportReady`, and the adapter's VSP DaemonSet | Those were stand-ins for the real daemon and the operator-rendered VSP pod. |

## M-B — VF attach and the default network

| Repo | Change | Why it lives there |
|---|---|---|
| Adapter VSP | `GetDevices` returns VF PCI addresses in the OPI pool (VF1 onward, never VF0) | The host device plugin already requires PCI addresses and advertises `openshift.io/dpu`. The VSP is the only new code. |
| Adapter VSP | `SetNumVfs` checks that DPF already created enough VFs and returns that count; it never writes `sriov_numvfs` | DPF creates the VFs from `DPUFlavor`. Writing sysfs would reset VF0 and cut the DPU off from Kamaji. |
| Adapter VSP | `CreateBridgePort` / `DeleteBridgePort` apply and delete a `DPUServiceInterface` of type `vf` | You chose the VSP as the writer, matching how Intel and Marvell program their data plane inside the VSP. |
| Adapter translator | Default `DPUServiceChain` (host VFs to uplink `p0`) and the `p0` interface | Chain composition is cluster-wide and must stay stable while pods come and go. The VSP only adds and removes labelled interfaces. |
| Adapter VSP | Interfaces owned by the host `Node`; on restart, drop interfaces whose VF is back in the host namespace | The CNI does not replay `CreateBridgePort` after a VSP restart. |

M-B needs no `dpu-operator` change if M-A has landed. The daemon, CNI, and device plugin already make these calls. The NVIDIA behaviour is entirely inside the VSP and translator.

## M-C — network functions and service chains

| Repo | Change | Why it lives there |
|---|---|---|
| Adapter translator | Two `service` interfaces per network function, on the `br-sfc` NAD; `nvidia.com/bf_sf` taken from the interface count; SFC `nodeSelector` mapped onto DPU nodes | This is the OPI-to-DPF mapping. The HBN-specific NAD, `4`, and `6Gi` come out here. |
| Adapter translator | Per-node chain: host VFs to the first network function, between functions, and the last function to `p0` | Same reason as the default chain: one chain per node, label-selected ports, DPF's sfc-controller installs the flows. |
| Adapter docs | Chart contract: a chart must consume the two DPF-injected interfaces | Users supply the chart; the adapter cannot guess the chart's values. |
| `dpu-operator` | SFC reconciler skips a network function that has `chart` set | PR 8 adds the chart field, but the reconciler still creates a pod from `nf.Image`. A chart network function would get an empty-image pod that also consumes two `openshift.io/dpu` devices. That pod path belongs to `image` network functions and stays unchanged. |

`CreateNetworkFunction` and `DeleteNetworkFunction` are not implemented. The DPU-side daemon is what calls them, and that daemon does not run on BlueField. The VSP returns `Unimplemented`. Network functions arrive only through the `ServiceFunctionChain` translation above.

## What "spike checklist" means

A spike is a short experiment on the dh5 lab that answers one question the design depends on, before the code assumes the answer. The checklist is that list of questions, each with the command or object to look at and the fallback if the answer is not what we expect. It is not a code change and not a test suite.

The six questions, in the order they unblock the milestones:

1. **Who creates the host VFs, and which indexes DPF reserves besides VF0?** Confirms `SetNumVfs` must not touch sysfs, and where the OPI pool starts. Unblocks M-B.
2. **How does a host node map to its DPU node in the Kamaji cluster?** Needed so interfaces, chains, and `DPUService` objects land on the right node. Unblocks M-A and M-B.
3. **Can one chain port match several VF interfaces, and do those VFs then share L2?** If not, the translator lists ports explicitly, and the 50-port limit caps the pool at 49. Unblocks M-B.
4. **How long does a VF `DPUServiceInterface` take to become Ready?** Compared with the CNI ADD timeout. If it is too slow, `CreateBridgePort` returns once DPF accepts the object. Unblocks M-B.
5. **What happens to live traffic when the default chain is swapped for an SFC chain?** Unblocks M-C.
6. **Does the upstream `dpu-cni` and device plugin work with mlx5 VFs on this host?** Unblocks M-B on the host side, and is independent of DPF.


Q4. I'm confused.. Can you clarify, how currently the dpu-operator and this adapter are used -- at which point, and how the operator is loading the adapter. And, What is your proposal - with respect to this (lifecycle)? In addition, why can't we load adater early and move things to adapter instead of dpu-operator .. I want to keep changes to dpu-operator minimal. Finally, create that spike, I'll run it on dh5 and get the output.

I created the spike script, and I've reworked the plan so the adapter does almost all the work, starting earlier in the lifecycle. The dpu-operator still needs four small, vendor-neutral changes. Two of them can be folded into PR 7 and one into PR 8, so only a two-line bug fix needs a new PR.

## How the two are used today

**On dh5 right now, the operator doesn't load the adapter at all.** They were installed separately:

1. The dpu-operator was installed with `hack/lab-deploy-dpu-operator.sh` into `openshift-dpu-operator`.
2. The adapter was installed separately with `make deploy` into `opi-nvidia-dpf-adapter-system`. It registered `VendorProvider/nvidia` as Available.
3. The adapter's own VSP DaemonSet runs on the host because of its NFD label (`hardware.opiproject.org/profile=nvidia-bluefield`).
4. The operator's `dpu-daemon` DaemonSet has **0 pods**, because no node carries `dpu=true`. So the operator's device plugin, CNI and detection never run on the BlueField host.
5. The `DataProcessingUnit` was applied by hand. The adapter's translator turned it into DPF objects, and DPF flashed the card.

**With PR 7, "loading" means deploying a Deployment, not a plugin.** When `DpuOperatorConfig.spec.vendor=nvidia` and `VendorProvider/nvidia` isn't Available yet, the operator applies `bindata/nvidia-adapter`. That's one Deployment running the adapter's manager (translator + LCM), and not the VSP. After that, the two only talk through Kubernetes objects. On dh5 this step was skipped because the separately installed adapter had already registered.

## Why some dpu-operator change is unavoidable

You're right that the adapter should come first and own almost everything. It can't own quite everything, for three reasons:

- **The workload contract lives in the daemon.** "Same features as Intel/Marvell" means the same `default-sriov-net` NAD, the same `openshift.io/dpu` resource and the same CNI server. All of that is served by the `dpu-daemon`. Rebuilding it inside the adapter would mean forking the daemon.
- **The adapter can't reuse the daemon as a library.** That code sits under `internal/`, which Go forbids other repos from importing.
- **The daemon only recognises built-in hardware.** Its detector list (Intel, Marvell, NetSec) is compiled in, so it will never create a `DataProcessingUnit` for a BlueField unless it learns about it somehow.

## Revised split: what stays upstream

| Upstream change | Size | Where it goes |
|---|---|---|
| A generic detector driven by `VendorProvider`: the adapter publishes its PCI IDs and product name in the `VendorProvider` spec, and the daemon builds a detector from that. No NVIDIA strings in core. | Small. The fallback is an ~80-line NVIDIA detector like Marvell's. | Fold into PR 7 (same SPI) |
| The `DataProcessingUnit` controller skips rendering a VSP pod when a provider owns that product, because the adapter already deploys its own VSP. | A few lines | Fold into PR 7 |
| The daemon merges only the Ready condition instead of overwriting the whole status. | 2 lines | New bug-fix PR |
| The SFC reconciler skips Helm-chart NFs instead of trying to create a pod with an empty image. | 1 line | Fold into PR 8 |

**Moved out of upstream:** the "don't delete CRs on daemon shutdown" change. The adapter handles that itself: no ownerReferences on DPF provisioning objects, a delay before deleting, and re-adopting objects by deterministic name. That way a daemon restart can never trigger a reflash.

**Everything else is adapter code:**
- deploying the VSP DaemonSet;
- holding `Init` back until the DPF DPU is Ready;
- turning BridgePort calls into `DPUServiceInterface` objects;
- building the default and SFC chains;
- translator ownership and cleanup;
- labelling hosts `dpu=true` through the adapter's NFD rule.

## Proposed lifecycle

1. Install the dpu-operator.
2. Apply `DpuOperatorConfig` with `vendor=nvidia`. The operator deploys the adapter manager (PR 7). **The adapter is loaded here, before any DPU is detected.**
3. The adapter's LCM installs or validates DPF. The adapter registers `VendorProvider/nvidia` (Available, with its detection info) and deploys its VSP DaemonSet plus the NFD rule that labels BlueField-3 hosts `dpu=true`.
4. Because of that label, the `dpu-daemon` lands on the BlueField host. Its generic detector matches the BlueField-3 and creates the host `DataProcessingUnit`. The operator skips VSP rendering for it.
5. The translator creates the DPF provisioning objects. DPF flashes the card, brings it up and creates the VFs.
6. The VSP answers `Init` only once the DPF DPU is Ready. The daemon then advertises the spare VFs as `openshift.io/dpu`, and the CR goes Ready.
7. When a pod is attached, the CNI calls `CreateBridgePort`. The VSP writes a `DPUServiceInterface`, and the translator keeps the chain current.
8. When a `ServiceFunctionChain` is created, the translator builds DPF chains; chart NFs become a `DPUService`.
9. On teardown, the translator removes the DPF objects after the delay.

## The spike

I added `hack/lab-spike-nvidia-dataplane.sh` and a Makefile target, `lab-spike-nvidia-dataplane`. I've only checked that it parses; it hasn't run against a cluster yet. It answers the six questions the plan depends on:

- **S1, who creates the VFs:** which VFs DPF already uses, plus firmware and flavor VF counts.
- **S2, host-to-DPU node mapping:** how the host node maps to the DPF DPU and to the tenant node.
- **S3, chain behaviour:** whether one chain port can select several VFs, and whether VFs on one switch can ping each other.
- **S4, timing:** how long a VF interface takes to become Ready and to show up in OVS.
- **S5, chain swap:** packet loss while swapping chains, and whether one interface can sit in two chains.
- **S6, upstream CNI compatibility:** whether the upstream CNI's VF operations (MAC, VLAN, spoof check, trust) are allowed on these VFs, plus CNI and device-plugin prerequisites.

By default it is read-only. With `ACTIVE=1` it temporarily uses the two highest-numbered spare VFs (never VF0, and never writes `sriov_numvfs`). It moves them into throwaway network namespaces and creates spike-labelled `DPUServiceInterface` and `DPUServiceChain` objects. A cleanup handler removes everything on exit, failure or Ctrl-C.

Run it on dh5 as root, with the DPU in OPERATIONAL state:

```bash
cd opi-nvidia-dpf-adapter
bash ./hack/lab-spike-nvidia-dataplane.sh                     # read-only first
ACTIVE=1 DPU_HOST=<bf-oob-ip> DPU_USER=ubuntu DPU_PASS=<pw> \
  bash ./hack/lab-spike-nvidia-dataplane.sh                   # full run
```

`DPU_HOST` is optional; without it the OVS ports, flows and OVS timings are skipped. You can pin the VFs with `VF_A=` and `VF_B=`, and the namespace with `SPIKE_NS=`.

Please send back the `/tmp/spike-nvidia-dataplane-<stamp>.tgz` it produces, or at minimum `SUMMARY.txt`. Output goes to `/tmp` rather than `working-directory/` because it contains serials and addresses.

On that point: `hack/check-no-lab-details.sh` currently fails on a file that was already in the repo, `working-directory/lab-state-20261114T165022Z/host/90-host-net.txt`, which contains a lab IP. The new files pass the check. You'll want to remove or redact that snapshot before the repo goes public.

Q5. Is spike blocking any items?

No. The spike only gates the items whose design changes based on its answers. Everything else can start now, and the spike can run in parallel.

**Start now, no spike needed:**

| ID | Work | Why it is safe to start |
|---|---|---|
| U1 | Detection fields on `VendorProvider` | API shape is already decided. |
| U2 | Provider-driven detector | Follows U1. PCI IDs and serial de-dupe are already decided. |
| U3 | Skip VSP rendering for provider-owned products | Follows U1. |
| U4 | Ready-only status merge | Two-line daemon bug fix, all vendors. |
| U5 | SFC reconciler skips chart network functions | One-line guard for PR 8. |
| A4 | VSP `Init` stays Unavailable until the DPF `DPU` is Ready | The daemon already retries `Init`. |
| A5 | OPI lifecycle heartbeat | Follows A4. |
| A7 | Annotation ownership and delayed delete | This is what stops a daemon restart from reflashing the card. |
| B7 | Network-function RPCs return `Unimplemented` | No DPU-side daemon exists to call them. |
| R3 | Remove the tracked lab snapshot that fails the public-repo check | Independent cleanup. |

**Wait for the spike:**

| Spike | Items it gates |
|---|---|
| S1, who creates the VFs and which are reserved | B1, B3 (the VF pool and `SetNumVfs`) |
| S2, host node to DPU node | B4, B6, C1, C3 (where DPF objects land) |
| S3, one chain port selecting many VFs | B6, C4 (selector chain versus one port per VF) |
| S4, how long a VF interface takes to become Ready | B4 (whether `CreateBridgePort` waits for Ready) |
| S5, chain swap | C4 (overlap the chains or delete then apply) |
| S6, CNI VF operations on these mlx5 VFs | A2, B8 (the `dpu=true` label and the pod-attach lab check) |

The practical order is to open the upstream items (U1 through U5) and A4, A5, and A7 now, and run the spike on dh5 alongside them. Bring back `/tmp/spike-nvidia-dataplane-<stamp>.tgz` and the VF-attach and chain work can start without a redesign.
