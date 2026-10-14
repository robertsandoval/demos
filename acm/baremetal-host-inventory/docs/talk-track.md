# Talk track: ACM Host Inventory to bare metal + GitOps operators

Use this as a short customer narrative (5–10 minutes). Details and commands live in the [demo README](../README.md).

## Opening

Many customers want the same outcome: stand up OpenShift on bare metal (or Redfish-emulated hosts such as kubevirt-redfish), manage it from a central hub, and drive consistent day-2 configuration from Git—not click-ops per cluster.

## Act 1 — Inventory and provision from ACM

- On the hub, ACM Host Inventory (Assisted Installer / infrastructure operator) owns discovery and install.
- Hosts appear as `BareMetalHost` objects with Redfish BMC endpoints. From ACM’s point of view, kubevirt-redfish looks like any other bare-metal host.
- We declare the cluster once: namespace, `InfraEnv`, `ClusterDeployment`, `AgentClusterInstall`, BMH, and `ManagedCluster`.
- ACM boots discovery, binds the agent, installs SNO, and imports the cluster—no separate “join” ritual.

**Show:** Host Inventory / cluster install progress, then ManagedCluster `Ready`.

## Act 2 — Desired state for the spoke from Git

- Installation is only half the story. Operators and platform services should arrive the same way every time.
- An OpenShift GitOps `Application` on the hub points at this repo’s `cluster-config/` and syncs to the managed cluster.
- First wave: NMState and OpenShift Virtualization (including the `HyperConverged` CR so Virt is actually deployed).
- Adding another operator is a new folder under `cluster-config/operators/` plus one line in kustomize—not a new runbook.

**Show:** Argo CD Application Healthy/Synced; NMState and CNV / HyperConverged on the spoke.

## Close

Hub-centric lifecycle: **inventory → install → import → GitOps configure**. Same pattern scales from one SNO lab cluster to many sites; tomorrow’s operators (LVMS, MetalLB, etc.) slot into the same GitOps pack.
