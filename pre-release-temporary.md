# Pre-release close-out (temporary)

Order: U6, then A10, then B8, then C7.

U6 is git and a unit test. A10 installs an OS and reboots the x86 host. B8 and C7 run only after a DPF `DPU` is already Ready. They do not reflash and they do not delete a `DPU`.

Run the lab sections on the x86 host as root. After `sudo -i`, home is `/root`. Clone into `/root` if the trees are not already there. Run the commit in a normal terminal. A commit made by the Cursor agent adds `Co-authored-by: Cursor`.

Tick an item only after the command in its "Pass" section succeeds.

## 0. Tools on the x86 host

```bash
sudo -i
command -v git
command -v go
go version
command -v kubectl
command -v docker
command -v make
command -v gh
gh auth status
```

`go version` must be 1.24 or newer. `gh auth status` must show a logged-in account that can push to `opiproject/dpu-operator`.

## 1. U6

### 1.1 Clone and check out the U4 branch

```bash
sudo -i
if [ ! -d /root/dpu-operator/.git ]; then
  git clone https://github.com/opiproject/dpu-operator.git /root/dpu-operator
fi
cd /root/dpu-operator
git status
git fetch origin
git checkout fix/ready-status-merge
git pull origin fix/ready-status-merge
git status -sb
git rev-parse HEAD
git log -1 --format='%H%n%an <%ae>%n%B'
```

`git rev-parse HEAD` must print `f3882b5dfcca239ef884d9d9ef5b5cb9e4d78d2a`.

`git log -1` must show subject `daemon: test that status sync writes only Ready`, author `Sridhar K N Rao <srao@linuxfoundation.org>`, and a `Signed-off-by` line for that same name. It must not contain `Co-authored-by`.

### 1.2 Run the U4 unit test

```bash
cd /root/dpu-operator
go test -mod=vendor ./internal/daemon/ -count=1 -run 'TestSyncSingleDpuCR'
echo exit:$?
```

Pass: the command prints `ok` and `exit:0`. Both `TestSyncSingleDpuCRKeepsForeignCondition` and `TestSyncSingleDpuCRSkipsUnchangedReady` ran.

### 1.3 Confirm PR 9

```bash
gh pr view 9 --repo opiproject/dpu-operator --json url,headRefOid,state,statusCheckRollup
gh pr checks 9 --repo opiproject/dpu-operator
```

Pass: `headRefOid` is `f3882b5dfcca239ef884d9d9ef5b5cb9e4d78d2a`, state is `OPEN`, and the DCO check is `pass`.

### 1.4 Add the condition-ownership paragraph on PR 7

```bash
cd /root/dpu-operator
git fetch origin
git checkout feat/vendor-provider-spi
git pull origin feat/vendor-provider-spi
git status -sb
git rev-parse --abbrev-ref HEAD
```

The branch must be `feat/vendor-provider-spi`.

```bash
cd /root/dpu-operator
python3 - <<'PY'
from pathlib import Path
p = Path("doc/vendor-provider-spi.md")
text = p.read_text()
anchor = "Intel and Marvell keep the in-tree VSP pod.\n"
paragraph = (
    "\n"
    "The daemon owns only the `Ready` condition on a `DataProcessingUnit`. "
    "Each status sync sets or removes that condition and leaves every other "
    "condition as it is, so a vendor adapter keeps the conditions it writes. "
    "When `Ready` is already unchanged, the daemon does not write status.\n"
)
if "The daemon owns only the `Ready` condition" in text:
    print("paragraph already present")
elif anchor not in text:
    raise SystemExit("anchor not found in doc/vendor-provider-spi.md")
else:
    p.write_text(text.replace(anchor, anchor + paragraph, 1))
    print("paragraph inserted")
PY
git diff -- doc/vendor-provider-spi.md
```

The diff must be that one paragraph and nothing else.

```bash
cd /root/dpu-operator
git add -- doc/vendor-provider-spi.md
git commit -m "$(cat <<'EOF'
Document that the daemon writes only the Ready condition

Vendor adapters keep the conditions they own across a daemon status sync.

Signed-off-by: Sridhar K N Rao <srao@linuxfoundation.org>
EOF
)"
git log -1 --format='%H%n%an <%ae>%n%B'
git status -sb
```

`git log -1` must show `Signed-off-by: Sridhar K N Rao <srao@linuxfoundation.org>` and must not show `Co-authored-by`. If the trailer is present, do not push. Reset that commit and create it again from a normal shell:

```bash
cd /root/dpu-operator
git reset --soft HEAD~1
git status -sb
```

Then run the `git commit` command above again from that shell.

Push only after the log is clean:

```bash
cd /root/dpu-operator
git push origin feat/vendor-provider-spi
gh pr view 7 --repo opiproject/dpu-operator --json url,headRefOid,state
gh pr checks 7 --repo opiproject/dpu-operator
```

Pass: PR 7 is `OPEN` and DCO is `pass`.

### 1.5 Maintainer review

```bash
gh pr view 7 --repo opiproject/dpu-operator --json url,reviews,statusCheckRollup
```

U6 stays open until `reviews` contains an approval from someone other than the author. There is no local command that creates that review.

## 2. A10

This deletes the current DPU, installs an OS, and reboots the host.

### 2.1 Adapter tree

```bash
sudo -i
if [ ! -d /root/opi-nvidia-dpf-adapter/.git ]; then
  git clone https://github.com/sknrao/opi-nvidia-dpf-adapter.git /root/opi-nvidia-dpf-adapter
fi
cd /root/opi-nvidia-dpf-adapter
git status
git fetch origin
git checkout main
git pull origin main
git status -sb
git rev-parse HEAD
git merge-base --is-ancestor 67e3a54 HEAD && echo 'contains 67e3a54' || echo 'MISSING 67e3a54'
```

Pass: the second line prints `contains 67e3a54`.

### 2.2 Operator tree for the lab image

This is a local lab branch. It is not PR 7, PR 8, or PR 9. Do not push it.

```bash
sudo -i
if [ ! -d /root/dpu-operator/.git ]; then
  git clone https://github.com/opiproject/dpu-operator.git /root/dpu-operator
fi
cd /root/dpu-operator
git status
git fetch origin
git checkout -B lab/spi-sfc-e2e origin/feat/vendor-provider-spi
git merge origin/feat/sfc-helm-chart
git status
git diff --stat
```

If `git merge` stops with conflicts, stop. Do not continue A10.

```bash
cd /root/dpu-operator
git apply --check /root/opi-nvidia-dpf-adapter/hack/lab-u4-ready-only.patch
git apply /root/opi-nvidia-dpf-adapter/hack/lab-u4-ready-only.patch
git diff --stat
```

`git diff --stat` must list only `internal/daemon/daemon.go`.

```bash
cd /root/dpu-operator
git status -sb
```

The branch must be `lab/spi-sfc-e2e` and it must not be pushed. Do not run `git push` in this directory.

### 2.3 Build and push the operator images

```bash
sudo -i
cd /root/opi-nvidia-dpf-adapter
docker login -u sknrao
REGISTRY=docker.io/sknrao DPU_OPERATOR_DIR=/root/dpu-operator \
  make lab-build-push-dpu-operator
echo exit:$?
```

Pass: the output contains `OK dpu-operator`, `OK dpu-daemon`, and `OK network-resources-injector`, and `exit:0`.

### 2.4 See the lab VIP and bridge the overlay will use

```bash
cd /root/opi-nvidia-dpf-adapter
grep -n 'dpu-cluster-' config/overlays/lab/manager-pull-always-patch.yaml
```

Those flag values are the VIP and the OOB bridge for this host. The bridge must already exist before milestone 1. DPF does not create it.

```bash
ip -4 addr show br-dpu
```

Pass: the command prints an address on `br-dpu`.

### 2.5 Tear down and prepare

```bash
sudo -i
cd /root/opi-nvidia-dpf-adapter
make lab-teardown-full
echo exit:$?
kubectl get nodes
kubectl get nodes -o wide
```

Pass: `exit:0`, and the node is `Ready`. This deleted the DPU. The next milestone with `ALLOW_OS_INSTALL=1` installs an OS and reboots the host.

```bash
cd /root/opi-nvidia-dpf-adapter
make lab-prepare-cluster
echo exit:$?
kubectl get nodes
```

Pass: the output contains `OK lab-prepare-cluster`, `exit:0`, and the node is still `Ready`.

If Multus is broken, the rollback command is:

```bash
cd /root/opi-nvidia-dpf-adapter
bash ./hack/lab-fix-multus-k3s.sh rollback
```

Do not run that unless Multus is broken.

### 2.6 Watch the label gate

In a second root shell, start this before milestone 1 and leave it running:

```bash
sudo -i
watch -n 2 'echo == nodes ==; kubectl get nodes -l dpu=true; echo == flavor ==; kubectl -n dpf-operator-system get dpuflavor dpf-lab-ovs-flavor; echo == cluster ==; kubectl get dpucluster -A; echo == config ==; kubectl get dpuoperatorconfig -A'
```

Before `dpf-lab-ovs-flavor` exists, `kubectl get nodes -l dpu=true` must print no nodes. After that flavor exists and the DPU cluster is Ready, the same command must show the host. Do not run `kubectl label` on the node.

### 2.7 Milestone 1, first run

```bash
sudo -i
cd /root/opi-nvidia-dpf-adapter
ALLOW_OS_INSTALL=1 REGISTRY=docker.io/sknrao \
  DPU_OPERATOR_DIR=/root/dpu-operator \
  make lab-milestone-01-cluster
echo exit:$?
```

Do not apply a host `DataProcessingUnit` by hand:

```bash
# do not run this
# kubectl apply -f /root/opi-nvidia-dpf-adapter/config/samples/dataprocessingunit-bf3-lab.yaml
```

The host reboots during the BFB install. After it is back:

```bash
sudo -i
kubectl get nodes
ip -4 neigh show dev br-dpu
```

On the rshim console the first login is `ubuntu` / `ubuntu`. Set the new password there. `ip -4 neigh show dev br-dpu` prints the DPU OOB address. Use that address as `DPU_HOST` below.

### 2.8 Milestone 1, second run

Do not set `ALLOW_OS_INSTALL` on this run.

```bash
sudo -i
cd /root/opi-nvidia-dpf-adapter
DPU_HOST=<dpu-oob-ip> DPU_USER=ubuntu DPU_PASS='<new-password>' \
  REGISTRY=docker.io/sknrao DPU_OPERATOR_DIR=/root/dpu-operator \
  make lab-milestone-01-cluster
echo exit:$?
```

Pass: the output contains `OK milestone 1` and `exit:0`.

### 2.9 Objects after milestone 1

```bash
sudo -i
kubectl get dataprocessingunit -A
kubectl get dpu -n dpf-operator-system
kubectl get dpu -n dpf-operator-system -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.phase}{"\n"}{end}'
kubectl --kubeconfig=/tmp/dpu-cluster.kubeconfig get nodes
kubectl -n opi-nvidia-dpf-adapter-system get pods
kubectl get pods -A -l app.kubernetes.io/component=vsp
kubectl get nodes -l dpu=true
kubectl get dataprocessingunit -A -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.conditions[?(@.type=="Ready")].status}{" "}{.status.conditions[?(@.type=="DPFReady")].status}{"\n"}{end}'
```

Pass:

- Exactly one host `DataProcessingUnit`, named `nvidia-bluefield-<serial>-host`.
- Its `Ready` value and its `DPFReady` value are both `True`.
- `kubectl get dpu -n dpf-operator-system` shows phase `Ready`.
- The tenant node from `/tmp/dpu-cluster.kubeconfig` is `Ready`.
- `opi-nvidia-dpf-adapter-system` has the controller manager Running and one VSP Running.
- `kubectl get pods -A -l app.kubernetes.io/component=vsp` prints one pod, in `opi-nvidia-dpf-adapter-system`.
- `kubectl get nodes -l dpu=true` prints the host.

If `kubectl get dataprocessingunit -A` prints a second object, delete only the one whose name does not start with `nvidia-bluefield-`:

```bash
kubectl get dataprocessingunit -A
kubectl delete dataprocessingunit <hand-applied-name>
kubectl get dataprocessingunit -A
```

### 2.10 Detection script

This deletes the `dpu-daemon` pod once. It does not reflash and it does not delete a `DPU`.

```bash
sudo -i
cd /root/opi-nvidia-dpf-adapter
make lab-verify-detection
echo exit:$?
```

Pass: the last lines include `FAILED=0` and `exit:0`.

## 3. B8

Run this only after section 2.10 passed.

```bash
sudo -i
cd /root/opi-nvidia-dpf-adapter
kubectl get dpu -n dpf-operator-system -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.phase}{"\n"}{end}'
```

The phase must be `Ready`. If it is not, stop. Do not run milestone 1 again.

```bash
cd /root/opi-nvidia-dpf-adapter
make lab-verify-vf-attach
echo exit:$?
```

The script creates `parity-a` and `parity-b`, pings `parity-b` to `parity-a` on `net1`, pings out from `parity-a`, creates one pod past the node `openshift.io/dpu` count, deletes those extra pods, deletes `parity-b`, restarts the VSP, and deletes `parity-a`.

Pass: `FAILED=0` and `exit:0`.

If the next section reports that no DPU is Ready, bounce the daemon and wait:

```bash
kubectl -n openshift-dpu-operator delete pod -l app=dpu-daemon --wait=true
kubectl get dpu -n dpf-operator-system -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.phase}{"\n"}{end}'
```

Continue only when the phase is `Ready`.

## 4. C7

Run this only after section 3 passed.

`hack/lib/lab-verify-common.sh` sets `CHART_REPO` to `https://example.invalid/charts`. That placeholder cannot forward. `ping-sf` is the milestone 2 chart, not this chart. Set `CHART_REPO`, `CHART_NAME`, and `CHART_VERSION` to a chart that forwards between its two interfaces.

```bash
sudo -i
cd /root/opi-nvidia-dpf-adapter
grep -n '^CHART_' hack/lib/lab-verify-common.sh
kubectl get dpu -n dpf-operator-system -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.phase}{"\n"}{end}'
test -r /tmp/dpu-cluster.kubeconfig && echo 'tenant kubeconfig present' || echo 'MISSING tenant kubeconfig'
```

The DPU phase must be `Ready`, and the kubeconfig line must be `tenant kubeconfig present`.

```bash
cd /root/opi-nvidia-dpf-adapter
SFC_NAD=dpf-operator-system/mybrsfc-lab \
CHART_REPO=<forwarding-chart-repo> \
CHART_NAME=<chart-name> \
CHART_VERSION=<chart-version> \
TENANT_KUBECONFIG=/tmp/dpu-cluster.kubeconfig \
  make lab-verify-sfc-chain
echo exit:$?
```

The script creates `parity-sfc-a` and `parity-sfc-b`, applies `ServiceFunctionChain` `parity-sfc`, checks that the default chain is gone, pings pod to pod, pod to external, pod to the function `net0`, and from the function pod to external, then deletes the SFC and the two pods.

Pass: `FAILED=0` and `exit:0`.

There is no Intel or Marvell host in this run, so there is no command for the image-network-function regression.

## 5. What to tick

| Item | Tick after |
|---|---|
| U6 | Section 1.2 prints `exit:0`, section 1.4 DCO is `pass`, and section 1.5 shows a maintainer approval |
| A10 | Section 2.8 prints `OK milestone 1` and section 2.10 prints `FAILED=0` |
| B8 | Section 3 prints `FAILED=0` |
| C7 | Section 4 prints `FAILED=0` |
