# OPI NVIDIA parity — work tracker

Goal: OPI can say NVIDIA BlueField-3 is supported the way Intel and Marvell are. The user installs `dpu-operator`, applies `DpuOperatorConfig` with `vendor=nvidia`, attaches pods with the OPI CNI, and applies a `ServiceFunctionChain`. DPF is used as-is. The user never writes a DPF object.

Companion documents: `opi-nvidia-parity-plan.html` (design, flows, call mapping) and `opi-dpf-architecture-review.html` (original review).

## How to use this file

- Work items have IDs (`S1`, `U3`, `A4`, …). The status table says which repo and branch each item is committed to. Adapter work is a branch off `main` named `feat/<ID>-<short>`, for example `feat/A4-init-gating`, and merges to `main`. Upstream work is committed directly to the branch named in the table.
- Tick the checkbox when the item's acceptance checks pass, and fill in the PR link in the status table.
- An item may start when everything in its "Depends on" line is done. Items without a dependency can run in parallel.
- A spike result that answers "no" changes the items listed under it. Update those items before coding them.
- The adapter repo is public. Nothing in this file, in PRs, or in committed docs may contain lab IPs, SSH key names, hostnames, or real serials. `hack/check-no-lab-details.sh` enforces this.

## Assumptions and decisions

| Topic | Decision |
|---|---|
| Upstream PRs | PR 7 (`VendorProvider`, `spec.vendor`) and PR 8 (Helm `chart` on a network function) are treated as merged. |
| DPF | Unmodified. Not imported into `dpu-operator`. |
| Hardware | BlueField-3 only: PCI `15b3:a2dc` and `15b3:a2dd`. Skip the SoC management function `c2d3` and skip VFs. |
| `DPUServiceInterface` writer | The VSP, on `CreateBridgePort`. The translator owns chains, `DPUService`, and provisioning objects. |
| DPU-side `DataProcessingUnit` | Deferred. Host-side CR only. |
| Network functions | Helm charts are acceptable. Remove the HBN-specific NAD, `bf_sf: "4"`, and `6Gi`. |
| VF count | DPF creates VFs from `DPUFlavor` `NUM_OF_VFS`. OPI never writes `sriov_numvfs`. VF0 is DPF's management channel and is never used. |
| `dpu-operator` footprint | Four small, vendor-neutral edits (phase 1). Everything that names a DPF object stays in the adapter. |
| Upstream e2e | Not a gate for this round. Tracked in phase 5. |

## Repos and branches

Two git repos. The local `dpu-operator` checkout is on `main`; the open PRs live on the branches below. Commit upstream items onto those branches, not onto a new branch off `main`, except U4.

| Repo | Branch | Items |
|---|---|---|
| [opiproject/dpu-operator](https://github.com/opiproject/dpu-operator) | `feat/vendor-provider-spi` (PR 7, open, base `main`) | U1, U2, U3, and the U1–U3 part of U6 |
| [opiproject/dpu-operator](https://github.com/opiproject/dpu-operator) | `feat/sfc-helm-chart` (PR 8, open, base `main`) | U5, and the U5 part of U6 |
| [opiproject/dpu-operator](https://github.com/opiproject/dpu-operator) | `fix/ready-status-merge` (local, cut from `origin/main` `3092bcbe`; not pushed yet) | U4, and the U4 part of U6 |
| [sknrao/opi-nvidia-dpf-adapter](https://github.com/sknrao/opi-nvidia-dpf-adapter) | `main` | A1–A9, B1–B7, C1–C6, H1–H3, V1, V2, R1, R3, R4, R5 |
| — | no commit | S1–S6, A10, B8, C7, V3 |
| this workspace, not a git repo | — | S7, V4, R2 |

NVIDIA DPF itself is not a repo we change.

## Status

| ID | Item | Repo / branch | Depends on | Status | PR / notes |
|---|---|---|---|---|---|
| S1 | Who creates host VFs, which are reserved | — | — | ☐ | lab only |
| S2 | Host node to DPU node mapping | — | — | ☐ | lab only |
| S3 | Chain port selecting many VFs, L2 between them | — | — | ☐ | lab only |
| S4 | `vf` interface Ready latency | — | — | ☐ | lab only |
| S5 | Chain swap disruption, interface in two chains | — | — | ☐ | lab only |
| S6 | Upstream CNI VF operations on mlx5 | — | — | ☐ | lab only |
| S7 | Record spike results and adjust items | this file | S1–S6 | ☐ | |
| U1 | `VendorProvider` detection fields | dpu-operator `feat/vendor-provider-spi` | — | ☑ | `61ad0802` pushed. Dropped `hostSideOnly` and `dedupeBy`: detection is host-side only and always dedupes by serial. `3947c115` (not pushed) vendors the fake client so the controller tests compile with `-mod=vendor`. |
| U2 | Provider-driven detector in the daemon | dpu-operator `feat/vendor-provider-spi` | U1 | ☑ | `61ad0802` pushed |
| U3 | Skip VSP rendering for provider-owned products | dpu-operator `feat/vendor-provider-spi` | U1 | ☑ | `61ad0802` pushed |
| U4 | Ready-only status merge | dpu-operator `fix/ready-status-merge` | — | ☐ | Code change is on the branch, cut from `origin/main`. Builds for linux. Unit test and push still open. Not blocked by PR 7 or PR 8. |
| U5 | SFC reconciler skips chart network functions | dpu-operator `feat/sfc-helm-chart` | — | ☑ | `5778fcc1` pushed |
| U6 | Upstream docs and tests for U1–U5 | same branch as the item it covers | U1–U5 | ☐ | U1–U3 docs are in `61ad0802`. U5 docs are in `5778fcc1`. U4 docs remain. |
| A1 | Publish detection data on `VendorProvider/nvidia` | adapter `main` | U1 | ☑ | Lab 2026-11-17: daemon created `nvidia-bluefield-<serial>-host` from `spec.detection` (`15b3`, `a2dc`/`a2dd`, exclude `c2d3`, product `BlueField-3`). |
| A2 | NFD rule labels BlueField-3 hosts `dpu=true` | adapter `main` | S6 | ☐ | Not met. NFD stores a slash-less key as `feature.node.kubernetes.io/dpu`. `dpu-daemon` selects bare `dpu=true`. Lab copies that label in `lab-milestone-01-cluster.sh`. |
| A3 | Manager ensures the VSP DaemonSet | adapter `main` | U3 | ☐ | Not exercised. Lab installs the adapter with `make deploy`. |
| A4 | VSP `Init` gated on DPF `DPU` Ready | adapter `main` | — | ☑ | Lab 2026-11-17: after DPF Ready, VSP served `127.0.0.1:50051` and the host CR went Ready. Requires VSP `hostNetwork` and health port `18082`. |
| A5 | OPI lifecycle heartbeat in the VSP | adapter `main` | A4 | ☑ | Same run: `Ping` reached Ready. A VSP restart leaves Ready false until `dpu-daemon` is bounced; the daemon does not redial. |
| A6 | Provisioning mapping gated on product and host side | adapter `main` | U2 | ☑ | Same run: one DPF `DPU` reached Ready from the daemon-created host CR. Flavor came from the lab `--flavor` flag. |
| A7 | Annotation ownership and delayed delete | adapter `main` | — | ☑ | Parity: daemon restart kept the DPF `DPU` uid and `DPFReady`. The lab daemon image includes the U4 patch. Upstream U4 is still open. |
| A8 | Remove `opi-daemon` and `ReportReady` | adapter `main` | A4, A5 | ☐ | Lab overlay drops the kind sidecar. Not signed off as removed from every profile. |
| A9 | VSP and manager RBAC | adapter `main` | A3, B4 | ☐ | |
| A10 | Phase A lab check | — | A1–A9, U1–U4 | ☐ | Blocked on A2, A3, A8, A9, and upstream U4. |
| B1 | VF pool definition | adapter `main` | S1 | ☑ | Lab attach used PF0 VFs above VF0. `openshift.io/dpu` was allocatable (45 on the first passing bring-up). |
| B2 | `GetDevices` returns VF PCI addresses | adapter `main` | B1 | ☑ | Pods scheduled on `openshift.io/dpu` and got a `net1`. |
| B3 | `SetNumVfs` verifies, never writes | adapter `main` | B1 | ☑ | No lab step writes `sriov_numvfs`. Pool size matched the existing firmware VFs. |
| B4 | BridgePort service writes `DPUServiceInterface` | adapter `main` | S2, S4, B2 | ☑ | Parity: host VF interfaces appeared when `parity-a` and `parity-b` attached, and one went away when `parity-b` was deleted. |
| B5 | Interface ownership and restart reconciliation | adapter `main` | B4 | ☑ | Parity: VSP restart kept the interface still held by `parity-a`. Delete must happen before that restart; a dead dial leaks the CR. |
| B6 | Translator default chain and `p0` interface | adapter `main` | S2, S3 | ☑ | Parity: `opi-default-<dpu>` and `opi-<dpu>-p0` existed, and the external ping from `parity-a` passed. |
| B7 | Network function RPCs return `Unimplemented` | adapter `main` | — | ☐ | Code is in the VSP. This lab did not call those RPCs. |
| B8 | Phase B lab check | — | B1–B7, A10 | ☐ | Attach and ping passed inside `lab-verify-parity`. Formal B8 stays open until A10 and B7 are closed. |
| C1 | Two service interfaces per network function | adapter `main` | S2 | ☑ | Parity SFC created the chart function's in and out interfaces. |
| C2 | Derived `bf_sf`, remove HBN hard-coding | adapter `main` | C1 | ☑ | Same SFC. No HBN NAD in the parity path. |
| C3 | SFC `nodeSelector` mapped to DPU nodes | adapter `main` | S2 | ☑ | The function pod ran on the tenant node for that DPU. |
| C4 | Per-node stitched chain and swap strategy | adapter `main` | S5, B6, C1 | ☑ | Parity: applying the SFC overlaid the default chain, and deleting it restored `opi-default-<dpu>`. |
| C5 | Network function chart contract and sample chart | adapter `main` | C1 | ☑ | Parity chart pull succeeded. Milestone 2 separately ran `ping-sf` `0.1.0`. |
| C6 | SFC status reporting | adapter `main` | C4 | ☑ | Parity SFC checks passed, including status. |
| C7 | Phase C lab check | — | C1–C6, U5, B8 | ☐ | SFC section of `lab-verify-parity` passed on 2026-11-17. Left open because B8 is open. |
| H1 | `DPUCluster` and Kamaji defaults in the lifecycle manager | adapter `main` | — | ☐ | Lab brought Kamaji up from `lab-bringup-pre-create.sh`, not from the lifecycle manager. |
| H2 | BFB URL default | adapter `main` | — | ☐ | Lab still sets `DPU_BFB_URL` in `config/overlays/lab/pci-patch.yaml`. |
| H3 | Kamaji kubelet-config skew handled without a manual step | adapter `main` | — | ☐ | Milestone 1 still runs `lab-strip-bf-kubelet-config` while the phase is `DPU Cluster Config`. The tenant node did reach Ready after that. |
| V1 | Unit and conformance tests | adapter `main` | per item | ☐ | same branch as the item |
| V2 | Lab verification scripts per phase | adapter `main` | A10, B8, C7 | ☐ | `make lab-verify-parity` passed on dh5 on 2026-11-17 (re-run). It is one script, not the three phase scripts named below. |
| V3 | Upstream e2e subset on BlueField-3 | — | C7 | ☐ | run upstream e2e; no branch unless a test has to change |
| V4 | Feature parity matrix signed off | this file | V3 | ☐ | |
| R1 | User documentation | adapter `main` | C7 | ☐ | Lab runbook is `docs/user-guide-lab-milestones.md`. `docs/configuration.md` separates lab values from the product. A generic install guide is still open. |
| R2 | Correct the architecture review | this workspace, not a git repo | — | ☐ | `opi-dpf-architecture-review.html` |
| R3 | Public-repo hygiene | adapter `main` | — | ☐ | |
| R4 | Images, versions, release notes | adapter `main`; image key on dpu-operator `feat/vendor-provider-spi` | C7 | ☐ | |
| R5 | TSC proposal update and parity claim | adapter `main` | V4 | ☐ | `docs/tsc-vendor-provider-spi-proposal.md` |

---

## Phase 0 — Spikes on the lab

A spike answers one question the design depends on before the code assumes the answer. Each has a fallback, so none blocks the plan.

Script: `opi-nvidia-dpf-adapter/hack/lab-spike-nvidia-dataplane.sh` (also `make lab-spike-nvidia-dataplane`). Run on the lab host as root with the DPU up.

```bash
cd opi-nvidia-dpf-adapter
bash ./hack/lab-spike-nvidia-dataplane.sh
ACTIVE=1 DPU_IP=<dpu-ssh-ip> DPU_USER=ubuntu DPU_PASS=<pw> \
  bash ./hack/lab-spike-nvidia-dataplane.sh
```

Output: `/tmp/spike-nvidia-dataplane-<stamp>.tgz` and `SUMMARY.txt`. It contains serials and addresses; keep it out of git.

### [ ] S1 — Who creates the host VFs, and which ones are reserved

- **Question:** Does DPF create the host VFs (count equal to `NUM_OF_VFS`)? Besides VF0, which VFs does DPF already use?
- **Evidence:** `s1-vf-ownership.txt`: PF `sriov_totalvfs` / `sriov_numvfs`, VF netdevs and bridge membership, kernel SR-IOV events, boot-time files that write `sriov_numvfs`, DPF host pod logs, `DPUFlavor` `nvconfig`.
- **If no:** `B1` changes. If something other than DPF creates the VFs, document that prerequisite; `SetNumVfs` still must not write sysfs.
- **Affects:** B1, B3.

### [ ] S2 — How a host node maps to its DPU node

- **Question:** What is the Kamaji tenant node name for this DPU, and which labels can a `nodeSelector` use on `DPUServiceInterface`, `DPUServiceChain`, and `DPUService`?
- **Evidence:** `s2-node-mapping.txt`: DPF `DPU` (`spec.dpuNodeName`), `DPUNode`, tenant node labels, the tenant `ServiceInterface` / `ServiceChain` objects DPF generated for the existing lab interface.
- **If no stable label exists:** the translator derives the selector from `DPU` status. Objects still have to land on that one node.
- **Affects:** B4, B6, C1, C3.

### [ ] S3 — Chain semantics

- **Question:** Can one chain port's `matchLabels` select several `vf` interfaces? Do VFs in one switch get L2 connectivity to each other?
- **Evidence:** `s3-s4-chain-and-latency.txt`, `ovs-snapshots.txt`, SUMMARY lines `S3 chain mode=single` and `mode=pair`.
- **If no:** the translator lists one port per VF. DPF allows 50 ports per switch, so the pool caps at 49 VFs per switch (or the chain uses more switches).
- **Affects:** B6, C4.

### [ ] S4 — `CreateBridgePort` latency

- **Question:** How long does a `vf` `DPUServiceInterface` take to become Ready, and to appear on `br-sfc`?
- **Evidence:** SUMMARY line `S4 vf DPUServiceInterface Ready after`.
- **If too slow for the CNI ADD timeout:** `CreateBridgePort` returns once DPF accepts the object, and readiness is reported later.
- **Affects:** B4.

### [ ] S5 — Chain swap

- **Question:** What happens to live traffic when one chain replaces another? Can one interface belong to two chains at once?
- **Evidence:** `s5-chain-swap.txt`, `s5-ping.txt`, SUMMARY `S5` lines (replies and losses per window).
- **If two chains cannot share an interface:** the translator deletes the old chain before applying the new one and accepts a measured gap.
- **Affects:** C4.

### [ ] S6 — Upstream CNI on mlx5 VFs

- **Question:** Do the PF-side VF operations the upstream SR-IOV CNI performs (MAC, VLAN, spoof check, trust, link state) succeed on these VFs? Are CNI binary and config directories, device-plugin sockets, and the `default-sriov-net` NAD in place on k3s?
- **Evidence:** `s6-opi-cni-static.txt`, `s6-vf-netlink.txt`, SUMMARY `S6` lines.
- **If an operation fails:** list it as a host prerequisite or an upstream CNI fix. If k3s paths differ from OpenShift, document how `dpu-cni` is installed.
- **Affects:** A2, B8.

### [ ] S7 — Record results and adjust

- Write a short results section at the end of this file: answer, evidence file, decision.
- Update each affected item before work on it starts.
- **Done when:** every spike has an answer and a decision recorded.

---

## Phase 1 — `dpu-operator` (upstream, vendor-neutral)

Repo: `opiproject/dpu-operator`. U1–U3 go on `feat/vendor-provider-spi` (PR 7). U5 goes on `feat/sfc-helm-chart` (PR 8). U4 is a new branch, `fix/ready-status-merge`, cut from `origin/main` (`3092bcbe`). It does not depend on PR 7 or PR 8. None of these changes names NVIDIA, except the fallback in U2.

### [ ] U1 — Detection fields on `VendorProvider`

- **What:** Add an optional `spec.detection` to the `VendorProvider` type from PR 7:
  - `pciVendorID` (string, for example `15b3`)
  - `pciDeviceIDs` (list)
  - `excludePCIDeviceIDs` (list, optional)
  - `productName` (string, the value used for `DataProcessingUnit` `dpuProductName`)
  - `platformName` (string, the value used for `DpuPlatformName`)
  - `dedupeBy` (enum: `serial`, `pciAddress`)
  - `hostSideOnly` (bool)
- **Why:** The daemon's detector list is compiled in. Publishing detection data on the provider keeps NVIDIA PCI IDs in the adapter.
- **Files:** `api/v1/vendorprovider_types.go`, generated deepcopy and CRD under `config/crd/bases/`, `bindata` copy of the CRD if the operator ships one.
- **Acceptance:**
  - [ ] Existing `VendorProvider` objects without `detection` still validate.
  - [ ] CEL or webhook rejects an empty `pciDeviceIDs` when `detection` is set.
  - [ ] `make generate manifests` is clean.

### [ ] U2 — Provider-driven detector in the daemon

- **What:** In `internal/platform`, add a detector built from every `VendorProvider` that is Available and has `spec.detection`. `NewDpuDetectorManager` appends these to the built-in Intel, Marvell, and NetSec detectors.
  - `IsDPU`: vendor and device ID match; excluded IDs and VFs skipped.
  - De-duplicate by serial using the existing `dpuDevices` argument, so a two-port card is one DPU. The daemon currently exits when it finds more than one DPU.
  - `GetDpuIdentifier`: `<platformName>-<lower(serial)>` (serial read from sysfs `serial`, VPD `SN`, or PCIe DSN).
  - `IsDpuPlatform`: `false` when `hostSideOnly`.
  - `Name()` returns `productName`; `DpuPlatformName()` returns `platformName`.
  - The daemon watches `VendorProvider` and rebuilds the list when it changes.
- **Fallback:** if maintainers prefer no dynamic detection, a fixed `internal/platform/nvidia-bluefield.go` (about 80 lines, same behaviour as above with constants) registered in `NewDpuDetectorManager`.
- **Files:** `internal/platform/vendordetector.go`, new `internal/platform/provider-detector.go`, daemon RBAC (`get/list/watch` on `vendorproviders`) in `internal/controller/bindata/daemon/`.
- **Acceptance:**
  - [ ] Unit test: fake PCI tree with two ports of one card yields one DPU with the serial-based identifier.
  - [ ] Unit test: no `VendorProvider` means detection identical to today.
  - [ ] Intel and Marvell detection unchanged (existing tests pass).

### [ ] U3 — Skip VSP rendering for provider-owned products

- **What:** In the `DataProcessingUnit` reconciler, do not render `bindata/vsp/<platform>` when the CR's `dpuProductName` belongs to an Available `VendorProvider` that declares it deploys its own plugin (for example `spec.detection` set, or an explicit `spec.managesVendorPlugin: true`). Set a condition or event saying the plugin is provider-managed.
- **Why:** The adapter deploys the VSP on the same socket path. Without the skip, the operator would either error on an unknown platform directory or start a second plugin.
- **Files:** `internal/controller/dataprocessingunit_controller.go`, its tests.
- **Acceptance:**
  - [ ] Intel and Marvell CRs still get their VSP pod.
  - [ ] A provider-owned CR gets no VSP pod and no error loop.

### [ ] U4 — Ready-only status merge

- **What:** In `syncSingleDpuCR` (`internal/daemon/daemon.go`), replace `currentDpuCR.Status = dpuCR.Status` with `meta.SetStatusCondition` for the `Ready` condition only. Leave other conditions untouched.
- **Why:** The current write erases conditions owned by other writers (`DPFReady`, `DPFHardwareCompatible`). This is a bug for every vendor.
- **Acceptance:**
  - [ ] Unit test: a CR with an extra condition keeps it after a Ready change.
  - [ ] No extra status updates when nothing changed.

### [ ] U5 — SFC reconciler skips chart network functions

- **What:** In `internal/daemon/sfc-reconciler/sfc.go`, skip any network function that has `chart` set. Image network functions behave as today.
- **Why:** With PR 8 a chart network function has no image. The reconciler would try to create a pod from an empty image and request two `openshift.io/dpu` devices.
- **Acceptance:**
  - [ ] Unit test: chart-only SFC creates no pod.
  - [ ] Image SFC still creates its pod.

### [ ] U6 — Upstream docs and tests

- **What:** Update `doc/vendor-provider-spi.md` with the detection contract, the plugin-ownership rule from U3, and the condition-ownership rule from U4. Add the unit tests listed above.
- **Acceptance:**
  - [ ] Upstream CI green.
  - [ ] Maintainer review of the SPI change done in PR 7.

---

## Phase 2 — Adapter, part A: detection and the host CR

Repo: `sknrao/opi-nvidia-dpf-adapter`, branch `main`. Cut a `feat/<ID>-<short>` branch per item and merge it back to `main`.

### [ ] A1 — Publish detection data on `VendorProvider/nvidia`

- **What:** `EnsureVendorProviderNVIDIA` writes `spec.detection`: vendor `15b3`, device IDs `a2dc`, `a2dd`, exclude `c2d3`, product `NVIDIA BlueField-3`, platform `nvidia-bluefield`, dedupe by serial, host side only.
- **Files:** `pkg/lifecycle/vendor_provider.go`, `pkg/lifecycle/vendor_provider_test.go`.
- **Acceptance:**
  - [ ] Unit test asserts the published spec.
  - [ ] Applying twice is a no-op (server-side apply, stable field manager).

### [ ] A2 — NFD rule labels BlueField-3 hosts `dpu=true`

- **What:** Promote `config/overlays/lab/nodefeaturerule-bluefield.yaml` into the default install. Add the `dpu=true` label (the `dpu-daemon` DaemonSet selects on it) next to the existing `hardware.opiproject.org/profile=nvidia-bluefield`.
- **Note:** If NFD is not installed, document labelling the node by hand, which is what the Intel and Marvell README already asks for.
- **Acceptance:**
  - [ ] On a BlueField-3 host the node gets `dpu=true` without manual steps.
  - [ ] A host without a BlueField-3 does not.

### [ ] A3 — Manager ensures the VSP DaemonSet

- **What:** The manager (lifecycle manager) applies the VSP DaemonSet when `vendor=nvidia`, using the same image as the manager. This covers the case where PR 7 deployed only the manager. Keep `config/vsp/` as the source of the manifest.
- **Files:** `pkg/lifecycle/`, `config/vsp/daemonset.yaml`, `config/vsp/rbac.yaml`.
- **Acceptance:**
  - [ ] Deleting the DaemonSet makes the manager recreate it.
  - [ ] Removing `DpuOperatorConfig` removes it.
  - [ ] The VSP pod mounts `/var/run/dpu-daemon` at the socket path the daemon dials.

### [ ] A4 — VSP `Init` gated on DPF `DPU` Ready

- **What:** `Init` returns gRPC `Unavailable` until the DPF `DPU` for this card's serial has `phase=Ready`. Then it starts a localhost server and returns `127.0.0.1:<port>`. The daemon retries `Init` forever, so the CR shows `Ready=False` during provisioning instead of the daemon crash-looping.
- **Files:** `pkg/vsp/grpc_server.go`, `pkg/vsp/handoff.go`, `cmd/vsp/main.go`.
- **Acceptance:**
  - [ ] Unit test with a fake client: Unavailable before Ready, address after.
  - [ ] Port configurable by flag; default avoids the daemon's own ports.

### [ ] A5 — OPI lifecycle heartbeat

- **What:** On the localhost server from A4, register the OPI lifecycle `HeartbeatService` (the one the daemon calls), not the `dpu-api` vendor package. `Ping` is healthy only when the DPF `DPU` is Ready and its tenant node is Ready.
- **Files:** `pkg/vsp/grpc_server.go`, `pkg/vsp/opi_handshake.go`.
- **Acceptance:**
  - [ ] Handshake test against the daemon's real client code path (`grpc_handshake_test.go`).
  - [ ] Ping turns unhealthy when the tenant node goes NotReady.

### [ ] A6 — Provisioning mapping gated on product and host side

- **What:** In `config/mappings/dataprocessingunit.yaml`, gate the emit on `dpuProductName == "NVIDIA BlueField-3"` and `isDpuSide == false`. Remove the check against the adapter-invented annotation `discovery.opiproject.org/product-name`, since the daemon now sets the product name.
- **Acceptance:**
  - [ ] Conformance test: an Intel or Marvell CR produces no DPF objects.
  - [ ] A host-side BlueField-3 CR produces `DPUDevice`, `DPUFlavor`, `BFB`, `DPU`.

### [ ] A7 — Annotation ownership and delayed delete

- **What:** Stop stamping a controller owner reference from the `DataProcessingUnit` onto DPF provisioning objects (`stampOwner` in `pkg/translation/translation_controller.go`). Track them with an annotation naming the source. Delete them only when the source has been absent longer than a grace period (default a few minutes, configurable), or when `DpuOperatorConfig` is removed. A recreated CR with the same name re-adopts them.
- **Why:** The daemon deletes its DPU CRs on shutdown. With an owner reference, garbage collection would delete the DPF `DPU` and reflash the card on every daemon restart.
- **Keep owner references** where a short-lived parent is intended, such as SFC-generated objects. Decide per mapping and record it in the mapping spec.
- **Files:** `pkg/translation/translation_controller.go`, `pkg/mapping` (per-emit ownership mode), `docs/mapping-spec.md`.
- **Acceptance:**
  - [ ] Test: delete and recreate the CR within the grace period; the DPF `DPU` object keeps its UID.
  - [ ] Test: delete and wait past the grace period; the DPF objects go.

### [ ] A8 — Remove the stand-ins

- **What:** Delete `cmd/opi-daemon`, `config/opi-daemon/`, and `ReportReady` in `pkg/vsp/ready.go` (it writes `Ready` on `DpuOperatorConfig`, which the operator owns). Keep the identity annotator inside the VSP process.
- **Acceptance:**
  - [ ] No adapter code writes the `Ready` condition on any OPI object.
  - [ ] Build, tests, and docs no longer reference `opi-daemon`.

### [ ] A9 — RBAC

- **What:** VSP ServiceAccount: read DPF `DPU`, `DPUNode`; create, patch, delete `DPUServiceInterface` (B4); read nodes; patch `DataProcessingUnit` annotations. Manager: manage the VSP DaemonSet (A3), the default chain (B6), and SFC outputs (phase C).
- **Files:** `config/vsp/rbac.yaml`, `config/rbac/`.
- **Acceptance:**
  - [ ] Least-privilege review; no wildcard verbs or resources.

### [ ] A10 — Phase A lab check

- **Steps:** Fresh install, apply `DpuOperatorConfig` with `vendor=nvidia`, do nothing else.
- **Acceptance:**
  - [ ] Adapter manager and VSP come up; `VendorProvider/nvidia` is Available.
  - [ ] Host gets `dpu=true`; `dpu-daemon` pod runs there.
  - [ ] One host `DataProcessingUnit` appears automatically.
  - [ ] DPF provisions the card; CR shows `Ready=True` and `DPFReady=True` together.
  - [ ] Restart the `dpu-daemon` pod: the card is not reprovisioned and `DPFReady` stays True.
  - [ ] No VSP pod rendered by the operator.

---

## Phase 2 — Adapter, part B: VF attach and the default network

### [ ] B1 — VF pool definition

- **What:** Define which host VFs OPI may use: default VF1 to VF(N−1) on PF0, minus any VFs S1 shows DPF uses. Make it configurable (flag or `DPUFlavor`-derived), never including VF0.
- **Acceptance:**
  - [ ] Unit test on a fake sysfs tree: VF0 and reserved VFs are excluded.

### [ ] B2 — `GetDevices` returns VF PCI addresses

- **What:** Return the pool's VF PCI addresses as device IDs. The daemon's device plugin registers them as `openshift.io/dpu`. Today the VSP returns the card serial, which breaks the device plugin.
- **Acceptance:**
  - [ ] Node allocatable shows `openshift.io/dpu` equal to the pool size.

### [ ] B3 — `SetNumVfs` verifies, never writes

- **What:** The daemon calls `SetNumVfs(8)`. Check that DPF already created at least that many usable VFs and return the pool size. Never write `sriov_numvfs`.
- **Acceptance:**
  - [ ] Test: requested count larger than the pool returns an error with a clear message.
  - [ ] `sriov_numvfs` unchanged across daemon restarts on the lab.

### [ ] B4 — BridgePort service writes `DPUServiceInterface`

- **What:** Register the OPI evpn-gw `BridgePortService` on the localhost server. `CreateBridgePort("host<pf>-<vf>")` server-side-applies a `DPUServiceInterface`: `interfaceType: vf`, `pfID: 0`, `vfID: <vf>`, `parentInterfaceRef: p0`, template labels for host VF and this DPU, and the node selector from S2. `DeleteBridgePort` deletes it. Deterministic name, for example `opi-<dpu>-pf0vf<vf>`.
- **Latency:** follow S4. Either wait for Ready within a bound, or return once DPF accepts.
- **Acceptance:**
  - [ ] Unit test with a fake client: create, idempotent re-create, delete.
  - [ ] Refuses VF0 and VFs outside the pool.

### [ ] B5 — Interface ownership and restart reconciliation

- **What:** Interfaces are owned by the host `Node`, not by the VSP pod. On VSP start, delete any interface whose VF netdev is back in the host network namespace (no pod holds it). The CNI does not replay `CreateBridgePort` after a VSP restart.
- **Acceptance:**
  - [ ] Restart the VSP while pods hold VFs: their interfaces stay.
  - [ ] Delete a pod while the VSP is down: its interface is removed after the VSP returns.

### [ ] B6 — Translator default chain and `p0`

- **What:** For each DPU node with no SFC, the translator keeps `opi-default-<dpu>`: one switch, ports [host-VF label selector, uplink `p0`]. It also emits a `physical` `DPUServiceInterface` for `p0`. If S3 says one port cannot select many interfaces, generate one port per interface instead.
- **Files:** new mapping in `config/mappings/`, or a small controller in `pkg/translation` if the mapping language cannot express it.
- **Acceptance:**
  - [ ] Chain exists as soon as the DPU is Ready, before any pod.
  - [ ] Adding and removing pods does not change the chain object (selector case).

### [ ] B7 — Network function RPCs

- **What:** `CreateNetworkFunction` and `DeleteNetworkFunction` return `Unimplemented`. Only the daemon on the DPU calls them, and none runs on BlueField.
- **Acceptance:**
  - [ ] Test asserts the status code.

### [ ] B8 — Phase B lab check

- **Acceptance:**
  - [ ] Pod on `default-sriov-net` gets a VF; a `DPUServiceInterface` appears for it and goes Ready.
  - [ ] Two pods on the same host reach each other.
  - [ ] Pod reaches the external network through `p0`.
  - [ ] Deleting the pod removes the interface.
  - [ ] More pods than VFs: extra pods stay Pending, nothing else breaks.

---

## Phase 2 — Adapter, part C: network functions and service chains

### [ ] C1 — Two service interfaces per network function

- **What:** For each network function, emit `<nf>-in` and `<nf>-out` service interfaces attached to DPF's `br-sfc` NAD, instead of the HBN `mybrhbn` NAD.
- **Files:** `config/mappings/servicefunctionchain.yaml`.
- **Acceptance:**
  - [ ] Conformance test: two interfaces per function, correct NAD.

### [ ] C2 — Derived `bf_sf`, remove HBN hard-coding

- **What:** Set `nvidia.com/bf_sf` from the number of interfaces. Remove `"4"` and `6Gi`. Expose resources through the chart's values if needed.
- **Acceptance:**
  - [ ] No HBN-specific literal left in the mapping.

### [ ] C3 — SFC `nodeSelector` to DPU nodes

- **What:** Translate the SFC's host node selector into the DPU node selector found in S2.
- **Acceptance:**
  - [ ] SFC targeting one host lands only on that host's DPU node.

### [ ] C4 — Per-node stitched chain and swap strategy

- **What:** When an SFC with functions n1…nk targets a node, replace the default chain with switches [host VFs, `n1-in`], [`n1-out`, `n2-in`], …, [`nk-out`, `p0`]. On SFC delete, restore the default chain. Order of operations follows S5 (overlap if allowed, otherwise delete then apply).
- **Acceptance:**
  - [ ] Measured traffic gap documented.
  - [ ] Never two conflicting chains left on a node after reconcile.

### [ ] C5 — Network function chart contract

- **What:** Document what a chart must do: consume the two DPF-injected interfaces and forward between them. Provide a minimal sample chart (generalise the existing `ping-sf` lab chart).
- **Acceptance:**
  - [ ] Sample chart passes C7.

### [ ] C6 — SFC status

- **What:** Mirror DPF readiness of the `DPUService`, interfaces, and chain into `DPFReady` on the SFC with a useful reason. Do not write `Ready`.
- **Acceptance:**
  - [ ] Broken chart shows `DPFReady=False` with the DPF reason.

### [ ] C7 — Phase C lab check

- **Acceptance:**
  - [ ] SFC with a chart creates the `DPUService`, interfaces, and stitched chain.
  - [ ] Pod to network function, pod to pod through the function, function to external, pod to external all pass.
  - [ ] Deleting the SFC removes the function and restores the default chain.
  - [ ] No empty-image pod is created (U5).
  - [ ] An image network function on Intel or Marvell still works (regression check, if hardware is available).

---

## Phase 3 — Hands-off gaps

### [ ] H1 — `DPUCluster` and Kamaji defaults

- **What:** The lifecycle manager creates a default `DPUCluster` (Kamaji tenant) when none exists, from documented defaults, and reports progress as a condition on `DpuOperatorConfig`.
- **Acceptance:**
  - [ ] Fresh cluster reaches A10 with no manual `DPUCluster`.

### [ ] H2 — BFB URL default

- **What:** Default the BFB URL from `config/versions.yaml` for the pinned DPF release; allow override.
- **Acceptance:**
  - [ ] No manual BFB URL needed for the pinned release.

### [ ] H3 — Kamaji kubelet-config skew

- **What:** The BlueField kubelet rejects fields from the newer Kamaji kubelet config. Today `make lab-strip-bf-kubelet-config` fixes it by hand. Automate it (for example through DPF's own customisation hooks or a `DPUFlavor` setting), or document it as a known prerequisite until NVIDIA fixes the BFB.
- **Acceptance:**
  - [ ] Either automated, or listed as a prerequisite in R1.

---

## Phase 4 — Verification

### [ ] V1 — Unit and conformance tests

- Each item above lists its tests. The conformance suite (`test/conformance`) covers every mapping change. `make test` and the conformance suite run in CI.

### [ ] V2 — Lab verification scripts

- One script per phase, matching the style of `hack/lab-verify-sfc.sh` (`KUBECTL`, `ok`/`bad`, PASS/FAILED): `lab-verify-detection.sh` (A10), `lab-verify-vf-attach.sh` (B8), `lab-verify-sfc-chain.sh` (C7).

### [ ] V3 — Upstream e2e subset on BlueField-3

- Run the upstream `e2e_test` scenarios against the lab:
  - [ ] webhook tests
  - [ ] host workload pods start
  - [ ] traffic between pods
  - [ ] SFC creates the network function
  - [ ] pod to pod through the function
  - [ ] pod to function
  - [ ] function to external
  - [ ] pod to external
  - [ ] network function deleted
  - [ ] more SFCs than resources
- Record which tests need an NVIDIA-specific variant (for example chart instead of image).

### [ ] V4 — Feature parity matrix

| Feature (as Intel and Marvell provide it) | Delivered by | Status | Lab note |
|---|---|---|---|
| Automatic DPU detection and `DataProcessingUnit` creation | U1, U2, A1, A2 | ☐ | Detection and the daemon-created CR passed. Bare `dpu=true` is still copied by the lab script (A2). |
| Vendor plugin started per DPU | U3, A3 | ☐ | Lab VSP is `make deploy`, not started by the operator. |
| DPU readiness on `Ready` | A4, A5, U4 | ☐ | Ready passed on the lab image, which carries the U4 patch. Upstream U4 is not open as a PR yet. |
| `openshift.io/dpu` VF resource | B1, B2, B3 | ☑ | dh5, 2026-11-17. |
| Pod attach through `default-sriov-net` and the OPI CNI | B4, B5, S6 | ☑ | `parity-a` and `parity-b` Ready. Needs whereabouts and the daemon CNI hostPaths on k3s. |
| Pod to pod and pod to external traffic | B6 | ☑ | External ping from `parity-a` passed. Milestone 2 host↔DPU ping also passed. |
| Network function insertion through `ServiceFunctionChain` | C1–C6, U5 | ☑ | Parity SFC section passed. |
| Network function removal | C4 | ☑ | Deleting the SFC restored the default chain. |
| Resource exhaustion handled | B8, V3 | ☐ | Not run. |
| Daemon restart safe | A7, U4 | ☑ | Parity restart check passed on the lab image only. |
| Install from `DpuOperatorConfig` only | A1–A3, H1, H2 | ☐ | Not the lab install path. |

---

## Phase 5 — Release and claim

### [ ] R1 — User documentation

- Install guide from zero: prerequisites (DPF window, k8s version, NFD, Multus, cert-manager, Argo), apply `DpuOperatorConfig`, attach a pod, apply an SFC, teardown. Replace the M1–M3 lab guides with this flow, and move lab-only steps to private docs.

### [ ] R2 — Correct the architecture review

- `opi-dpf-architecture-review.html` cites `spec.mode` and `DpuNetwork`, which are not in upstream `main`. Fix those references and link the parity plan.

### [ ] R3 — Public-repo hygiene

- `working-directory/lab-state-20261114T165022Z/` is tracked and fails `hack/check-no-lab-details.sh` (it contains a lab address). Remove or redact it, and add `working-directory/` to `.gitignore`.
- Run the check in CI on every PR.

### [ ] R4 — Images, versions, release notes

- Publish adapter manager and VSP images. Set `NvidiaDpfAdapterImage` in `dpu-operator` (PR 7). Pin versions in `config/versions.yaml`. Release notes list the supported DPF window and BlueField-3 only.

### [ ] R5 — TSC proposal and parity claim

- Update `docs/tsc-vendor-provider-spi-proposal.md` with U1–U3. Present V4 to the TSC. Claim parity only when every V4 row is ticked.

---

## Spike results

Fill in after S1–S6 run.

| Spike | Answer | Evidence | Decision | Items updated |
|---|---|---|---|---|
| S1 | Both PFs show `sriov_numvfs` equal to firmware `NUM_OF_VFS` (46) and the `DPUFlavor` `nvconfig`. No boot file writes `sriov_numvfs`. PF0 VF0 is the only VF on `br-dpu`. VF10 was out of the host netns because a lab pod held it. PF1 VFs are unused by the data path. | `spike-nvidia-dataplane-20261116T113606Z/s1-vf-ownership.txt` | Pool is PF0 VF1 onward. Never VF0. Never write `sriov_numvfs`. | B1, B3 |
| S2 | DPF `DPU` name is the tenant node name. `spec.dpuNodeName` is the x86 node. Tenant `ServiceInterface.spec.node` is the DPU name. `parentInterfaceRef` is `p0-<dpu-name>`, not `p0`. | `s2-node-mapping.txt` | Interfaces and chains target `spec.node` = DPU name. Uplink ref is `p0-<dpu-name>`. | B4, B6, C1, C3 |
| S3 | One port whose labels match two VF interfaces stays `Ready=False`: DPF requires exactly one interface per port (`found 2`). One port per VF was Ready in 1.4s and pinged 5/5. | `s3-s4-chain-and-latency.txt` | One chain port per interface. 45 host VFs plus `p0` is 46 ports, under the 50-port cap. | B6, C4 |
| S4 | A `vf` `DPUServiceInterface` was Ready in about 0.5s. OVS timing was not captured: SSH to the BlueField was denied. | `SUMMARY.txt`, `ovs-snapshots.txt` | `CreateBridgePort` may wait for Ready. Re-run OVS timing only if a later lab check needs it. | B4 |
| S5 | A second chain on the same ports became Ready with no ping loss. Deleting the first chain lost nothing. Deleting the last chain stopped traffic (92 lost). Putting a chain back lost 3 replies, then recovered. | `s5-chain-swap.txt`, `SUMMARY.txt` | Overlap the new chain with the old one, then delete the old one. | C4 |
| S6 | `dpu-cni` is not on the x86 node. The node has no `dpu=true` label, so `dpu-daemon` is not scheduled. `default-sriov-net` exists. Setting a VF MAC and VLAN 0 succeeded. spoof check, trust, link state, and max tx rate returned `Operation not permitted`. Allocatable `nvidia.com/mlnx_sriov` is 1. | `s6-opi-cni-static.txt`, `s6-vf-netlink.txt` | A2 still labels the node. B8 cannot pass until `dpu-cni` is installed and the EPERM VF ops are handled. That does not block A4, A5, or A7. | A2, B8 |

## Lab run 2026-11-17 (dh5)

`make lab-milestone-01-cluster`, then `make lab-verify-parity` (re-run after the whereabouts, hostNetwork, and daemon-mount fixes), then `make lab-milestone-02-ping`. All three passed. The daemon image was PR 7 + PR 8 plus `hack/lab-u4-ready-only.patch`. The adapter was installed with `make deploy`, not by `dpu-operator`.

| Check | Result |
|---|---|
| Daemon-created host `DataProcessingUnit`, one DPF `DPU` Ready, tenant node Ready | Pass, after the kubelet-config strip |
| `lab-verify-parity` (Ready, default chain, `p0`, OPI CNI attach, SFC create and delete) | Pass |
| Host VF ↔ `ping-sf` ping (`lab-milestone-02-ping`) | Pass, 3/3, `fail_mode=secure` |

Still required on this k3s host, and not part of the generic product: bare `dpu=true` copied from the NFD feature label, whereabouts `v0.9.4` in both CNI directories, `dpu-daemon` `runAsUser: 0` plus hostPath mounts of `/opt/cni` and the k3s CNI dir, VSP `hostNetwork` with health port `18082`, and the U4 patch in the lab daemon image. A VSP restart still leaves Ready false until the daemon is bounced.

---

## Additional important items to consider

Not yet assigned IDs. Promote into A/B/H/R (or upstream) when you schedule them. The main A–R list stays the plan; these are gaps that showed up from the 2026-11-17 lab and a pass over the tracker.

### Daemon ↔ VSP reconnect after VSP restart

Lab: after a VSP restart, `Ready` stays false until `dpu-daemon` is bounced — the daemon does not redial. A5 notes the symptom; nothing owns the fix.

- **Do:** Upstream daemon redial on dial failure, and/or VSP handshake that recovers without a daemon bounce.
- **Why:** Intel/Marvell-like ops expect Ready to return when the vendor plugin comes back.
- **Touches:** A5 acceptance, A10 Ready checks, possibly `dpu-operator` HostSideManager.

### S6 EPERM VF operations — explicit owner

Spike S6: setting VF MAC and VLAN 0 succeeded; spoof check, trust, link state, and max tx rate returned `Operation not permitted`. The S6 decision says B8 needs this handled; no item describes how.

- **Do:** Pick one — CNI/NAD config that skips those ops, upstream CNI/driver fix, or document as unsupported on this mlx5 path — and attach it to B8 (or a small B/S follow-up).
- **Why:** Otherwise B8 stays open with no clear exit.

### Non-OpenShift host prerequisites (k3s)

Parity on dh5 needed whereabouts in `/opt/cni/bin` and `/var/lib/rancher/k3s/data/cni`, `dpu-daemon` `runAsUser: 0` plus hostPath mounts of `/opt/cni` and the k3s CNI dir, and careful Multus (`rke2-multus`). That is not a tracker ID today.

- **Do:** Add something like **H4** (automate what you can) and/or hard prerequisites in R1 / the support matrix.
- **Why:** Without it, “parity” silently assumes OpenShift-shaped CNI layout.

### VSP `hostNetwork` and health port (`:8082` → `:18082`)

`dpu-daemon` is hostNetwork, dials `127.0.0.1:50051`, and already binds `:8082`. Lab fixed PingFailed with VSP `hostNetwork: true` and health on `:18082`. Still lab-overlay folklore.

- **Do:** Make this acceptance on **A3/A4** (or a small A item): any co-located hostNetwork VSP must use a non-clashing probe port and share the host network for Init.
- **Why:** Product-critical on every cluster where the daemon is hostNetwork, not only dh5.

### A2 mechanism — bare `dpu=true` (NFD cannot emit it)

A2 text still reads like “put `dpu=true` on the NFD rule.” NFD stores a slash-less key as `feature.node.kubernetes.io/dpu`. The daemon selects bare `dpu=true`. Lab copies the label in `lab-milestone-01-cluster.sh`.

- **Do:** Specify the product mechanism in A2 (for example manager patches bare `dpu=true` from the feature label; hand-label only when NFD is absent). Do not tick A2 for a rule that only sets the prefixed feature label.
- **Why:** Acceptance “without manual steps” is otherwise impossible as written.

### Operator starts the companion (not only `make deploy`)

A3 is “manager ensures the VSP DaemonSet.” Lab installs the adapter with `make deploy`. R4 mentions `NvidiaDpfAdapterImage` on PR 7; there is no explicit gate that `dpu-operator` alone starts the companion.

- **Do:** Add an A10 or R4 check: apply only `DpuOperatorConfig` with `vendor=nvidia` (image set on the operator); no `make deploy`; manager + VSP still come up.
- **Why:** That is the install path the parity claim describes.

### Smaller hygiene (optional)

| Topic | Note |
|---|---|
| Product name string | A1 text says `NVIDIA BlueField-3`; lab/mapping often use `BlueField-3`. Pick one canonical value or A6 gates break. |
| S7 | Results table is filled; tick S7 or leave a one-line “recorded” note. |
| B7 | `Unimplemented` RPCs — cheap to close so B8 is not held on ceremony. |
| Multi-DPU / multi-node | Fine as a non-goal for this round; say so explicitly. |
| Uninstall / no-reflash | Optional V/R check that teardown does not reflash and does not delete shared CRDs (hybrid design already says this). |
