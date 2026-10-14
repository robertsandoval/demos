# ACM Host Inventory + GitOps Operators

Provision a Single Node OpenShift (SNO) spoke with **ACM Host Inventory** and Redfish `BareMetalHost`s (including **kubevirt-redfish**), then configure the cluster with an **OpenShift GitOps** Application that installs **NMState** and **OpenShift Virtualization**.

Customer-facing narrative: [docs/talk-track.md](docs/talk-track.md).

## Architecture

```text
Hub (ACM + Host Inventory + OpenShift GitOps)
  │
  ├─ hub/cluster/     → InfraEnv, ClusterDeployment, AgentClusterInstall,
  │                     BareMetalHost (Redfish), ManagedCluster
  │
  └─ hub/gitops/      → Argo CD AppProject + Application
                          │
                          ▼
                     Spoke (managed cluster)
                          │
                          └─ cluster-config/operators/
                               ├─ nmstate
                               └─ openshift-virtualization (+ HyperConverged)
```

## Prerequisites

| Requirement | Notes |
|-------------|--------|
| Hub OpenShift | **4.22** (or matching current release) |
| ACM / MCE | Host Inventory (Central Infrastructure Management / Assisted Installer) enabled |
| OpenShift GitOps | Installed on the hub (`openshift-gitops`) |
| Pull secret | Red Hat pull secret for the spoke install |
| SSH public key | Injected into InfraEnv / AgentClusterInstall |
| BMC reachability | Hub (or provisioning network) can reach kubevirt-redfish / Redfish endpoints |
| ClusterImageSet | An image set for the target OCP version exists on the hub (example name in `site.env.example`) |

Optional UI path: create the infrastructure environment and cluster in the ACM console; the CRs under `hub/cluster/` match what the UI creates so you can demo either console or Git/CLI.

## Repository layout

```text
acm/baremetal-host-inventory/
  examples/site.env.example     # Lab-specific values
  hub/cluster/                  # Host Inventory install CRs (hub)
  hub/gitops/                   # Argo CD AppProject + Application (hub)
  cluster-config/               # Synced to the spoke
    operators/                  # Extensible operator pack
  docs/talk-track.md
```

## 1. Configure lab values

```bash
cd acm/baremetal-host-inventory
cp examples/site.env.example examples/site.env
# Edit examples/site.env — cluster name, domain, CIDR, BMC URL, MAC, etc.
set -a && source examples/site.env && set +a
```

List ClusterImageSets on the hub and set `CLUSTER_IMAGE_SET` accordingly:

```bash
oc get clusterimageset
```

## 2. Materialize hub manifests

Replace tokens in a working copy (do not commit filled-in secrets):

```bash
WORKDIR=$(mktemp -d)
cp -R hub/cluster "$WORKDIR/cluster"

SSH_PUBLIC_KEY=$(cat "${SSH_PUBLIC_KEY_FILE}")

# macOS: use sed -i '' ; Linux: sed -i
replace() {
  local file=$1
  sed -i.bak \
    -e "s|REPLACE_CLUSTER_NAME|${CLUSTER_NAME}|g" \
    -e "s|REPLACE_BASE_DOMAIN|${BASE_DOMAIN}|g" \
    -e "s|REPLACE_CLUSTER_IMAGE_SET|${CLUSTER_IMAGE_SET}|g" \
    -e "s|REPLACE_MACHINE_NETWORK_CIDR|${MACHINE_NETWORK_CIDR}|g" \
    -e "s|REPLACE_HOST_NAME|${HOST_NAME}|g" \
    -e "s|REPLACE_BOOT_MAC_ADDRESS|${BOOT_MAC_ADDRESS}|g" \
    -e "s|REPLACE_BMC_ADDRESS|${BMC_ADDRESS}|g" \
    -e "s|REPLACE_SSH_PUBLIC_KEY|${SSH_PUBLIC_KEY}|g" \
    "$file"
  rm -f "${file}.bak"
}

for f in "$WORKDIR"/cluster/*.yaml; do replace "$f"; done
```

## 3. Create secrets and apply Host Inventory CRs

```bash
oc create namespace "${CLUSTER_NAME}" --dry-run=client -o yaml | oc apply -f -

oc create secret generic pull-secret \
  -n "${CLUSTER_NAME}" \
  --from-file=.dockerconfigjson="${PULL_SECRET_FILE}" \
  --type=kubernetes.io/dockerconfigjson \
  --dry-run=client -o yaml | oc apply -f -

oc create secret generic bmc-credentials \
  -n "${CLUSTER_NAME}" \
  --from-literal=username="${BMC_USERNAME}" \
  --from-literal=password="${BMC_PASSWORD}" \
  --dry-run=client -o yaml | oc apply -f -

oc apply -k "$WORKDIR/cluster"
```

### Redfish / kubevirt-redfish notes

- BMC address form: `redfish-virtualmedia+https://<host>:<port>/redfish/v1/Systems/<id>`
- `disableCertificateVerification: true` is set for typical lab TLS
- `inspect.metal3.io: disabled` avoids inspection issues common with emulated BMCs
- `bootMACAddress` must match the NIC that will PXE/virtual-media boot

## 4. Watch install and import

```bash
oc get bmh -n "${CLUSTER_NAME}" -w
oc get agents -n "${CLUSTER_NAME}" -w
oc get agentclusterinstall -n "${CLUSTER_NAME}" "${CLUSTER_NAME}" -o jsonpath='{.status.conditions}' | jq
oc get managedcluster "${CLUSTER_NAME}" -o wide
```

Wait until `ManagedCluster` is `Ready` / `hubAcceptsClient` and the cluster shows available in ACM.

### Approve the agent (if required)

Some labs require explicit approval before install proceeds:

```bash
oc get agents -n "${CLUSTER_NAME}"
# oc -n "${CLUSTER_NAME}" patch agents.agent-install.openshift.io <agent> \
#   --type merge -p '{"spec":{"approved":true}}'
```

## 5. Register the spoke in OpenShift GitOps

The Application destinations use the **Argo CD cluster name**, which should match `CLUSTER_NAME`.

If the managed cluster is not already registered in GitOps (ACM placement / cluster secret), register it using your standard method (ACM GitOps cluster registration, or `argocd cluster add` with the spoke kubeconfig). Confirm:

```bash
oc -n openshift-gitops get secrets -l argocd.argoproj.io/secret-type=cluster
# or
oc -n openshift-gitops exec -it deploy/openshift-gitops-server -- argocd cluster list
```

## 6. Apply the GitOps Application

```bash
GITWORKDIR=$(mktemp -d)
cp hub/gitops/*.yaml "$GITWORKDIR/"

sed -i.bak \
  -e "s|REPLACE_CLUSTER_NAME|${CLUSTER_NAME}|g" \
  "$GITWORKDIR"/application-cluster-config.yaml
rm -f "$GITWORKDIR"/*.bak

oc apply -f "$GITWORKDIR/appproject.yaml"
oc apply -f "$GITWORKDIR/application-cluster-config.yaml"

oc -n openshift-gitops get application "${CLUSTER_NAME}-cluster-config"
```

The Application syncs [cluster-config/](cluster-config/) from `https://github.com/robertsandoval/demos.git` (`main`) to the managed cluster.

> Until this repository path exists on `main`, either push the demo first or temporarily set `targetRevision` / a fork URL in the Application.

## 7. Verify operators on the spoke

Using the spoke kubeconfig (or `oc --context`):

```bash
oc get csv -n openshift-nmstate
oc get nmstate -n openshift-nmstate

oc get csv -n openshift-cnv
oc get hyperconverged -n openshift-cnv
oc get network-attachment-definitions -A
```

In Argo CD: Application **Synced** + **Healthy** (HyperConverged / NMState may briefly Degraded until CRDs exist; sync options allow retry).

## Adding more operators

1. Create `cluster-config/operators/<operator-name>/` with `Namespace`, `OperatorGroup`, `Subscription` (and any operand CR).
2. Add the files to [cluster-config/operators/kustomization.yaml](cluster-config/operators/kustomization.yaml).
3. Commit and push; the Application self-heals to the new desired state.

Example candidates: LVM Storage, MetalLB, OpenShift AI, Advanced Cluster Security (spoke components).

## Console alternative (Host Inventory UI)

1. **Infrastructure → Host inventory** — create infrastructure environment (maps to `InfraEnv`).
2. Add host with BMC credentials and boot MAC (maps to `BareMetalHost` + secret).
3. **Infrastructure → Clusters → Create cluster** — select hosts, SNO, networking (maps to `ClusterDeployment` + `AgentClusterInstall`).
4. After import, apply `hub/gitops/` as above for day-2 operators.

Use the YAML under `hub/cluster/` to show that UI and Git declare the same model.

## Cleanup

```bash
# Stop GitOps managing the spoke config (optional: leave operators installed)
oc -n openshift-gitops delete application "${CLUSTER_NAME}-cluster-config" --wait=false

# Deprovision Host Inventory cluster (destroys the spoke install)
oc delete managedcluster "${CLUSTER_NAME}" --wait=false
oc delete clusterdeployment -n "${CLUSTER_NAME}" "${CLUSTER_NAME}"
# After uninstall completes:
oc delete namespace "${CLUSTER_NAME}"
```

For kubevirt-redfish VMs, power off / recreate lab VMs as needed after BMH deletion.

## Placeholder reference

| Token | Meaning |
|-------|---------|
| `REPLACE_CLUSTER_NAME` | Spoke / namespace / ManagedCluster name |
| `REPLACE_BASE_DOMAIN` | Base DNS domain |
| `REPLACE_CLUSTER_IMAGE_SET` | Hub `ClusterImageSet` name |
| `REPLACE_MACHINE_NETWORK_CIDR` | Machine network CIDR |
| `REPLACE_HOST_NAME` | BMH / node hostname |
| `REPLACE_BOOT_MAC_ADDRESS` | Boot MAC |
| `REPLACE_BMC_ADDRESS` | Redfish virtual media URL |
| `REPLACE_SSH_PUBLIC_KEY` | SSH public key string |
