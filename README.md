# HCP Infrastructure on SNO - Kustomize

This repository deploys the **infrastructure components** required for OpenShift Hosted Control Planes (HCP) on a Single Node OpenShift (SNO) cluster. HCP cluster creation and credentials are managed manually via the OpenShift web console.

## Quick Start

```bash
# Login to your SNO cluster
oc login --token=<your-token> --server=<your-server>


# Deploy infrastructure components
oc apply -k overlays/infrastructure

OR Use GitOps method.
<>Install GitOps Operator, then apply Application manifests.
oc apply -f argocd-apps/kustomization.yaml

To change the GoMaxProcs webhook, simply go to `overlays/gomaxprocs-webhook/kustomization.yaml` and change the patch section. 

# Then create HCP clusters manually via the web console
```

## What This Deploys

### Infrastructure Components (Automated via Kustomize)

| Component | Purpose | Namespace |
|-----------|---------|-----------|
| **Wildcard Routes** | Enable wildcard DNS for HCP ingress | openshift-ingress-operator |
| **LVM Storage** | Persistent storage for VMs and etcd | openshift-storage |
| **OpenShift Virtualization** | KubeVirt platform for worker VMs | openshift-cnv |
| **MetalLB** | LoadBalancer for API server | metallb-system |
| **MultiCluster Engine** | HCP management platform | multicluster-engine |

The HCP creation is not automated, and will need to be done manually via WebUI.

### HCP Clusters Creation (Manual via Web Console)

- **Credentials**: Create via Console → Infrastructure → Clusters → Credentials
- **Hosted Clusters**: Create via Console → Infrastructure → Clusters → Create cluster → Hosted control plane

## Repository Structure

```
.
├── base/                         # GitOps bootstrap (optional)
│   └── 00-gitops/               # OpenShift GitOps operator
├── argocd-apps/                 # ArgoCD Applications (optional)
│   └── hcp-infrastructure.yaml  # Infrastructure app
├── components/
│   └── infrastructure/          # Base infrastructure manifests
│       ├── 00-wildcard-routes.yaml
│       ├── 0-hcp_namespaces.yaml
│       ├── 01-kubelet-config.yaml
│       ├── 02-lvm-storage-operator.yaml
│       ├── 03-lvmcluster.yaml
│       ├── 04-openshift-virtualization-operator.yaml
│       ├── 05-hyperconverged.yaml
│       ├── 06-storage-profile.yaml
│       ├── 07-metallb-operator.yaml
│       ├── 08-metallb-config.yaml
│       ├── 09-multicluster-engine-operator.yaml
│       └── 10-multicluster-engine.yaml
└── overlays/
    └── infrastructure/          # Environment-specific config
        └── kustomization.yaml   # Patches for storage, network
```

## Installation

### Configure Infrastructure

Edit `overlays/infrastructure/kustomization.yaml`:

```yaml
patches:
  # Update storage device
  - target:
      kind: LVMCluster
      name: lvmcluster
    patch: |-
      - op: replace
        path: /spec/storage/deviceClasses/0/deviceSelector/paths/0
        value: /dev/nvme0n1  # CHANGE to your disk

  # Update MetalLB IP range
  - target:
      kind: IPAddressPool
      name: main-pool
    patch: |-
      - op: replace
        path: /spec/addresses/0
        value: 10.0.1.200-10.0.1.210  # CHANGE to your IPs
```

**Find your disk device**:
```bash
oc debug node/$(oc get nodes -o name | head -1 | cut -d/ -f2) -- chroot /host lsblk
```



### Create HCP Clusters via Web Console

1. **Create Credentials**:
   - Console → Infrastructure → Clusters → Credentials
   - Click "Create credential"
   - Credential type: "Red Hat OpenShift pull secret"
   - Add your pull secret from cloud.redhat.com
   - Optionally add SSH public key

2. **Create Hosted Cluster**:
   - Console → Infrastructure → Clusters → Create cluster
   - Select "Hosted control plane"
   - Choose "OpenShift Virtualization"
   - Fill in cluster details:
     - Name: spoke1, spoke2, etc.
     - Base domain: apps.baremetal-sno.gsslab.pnq2.redhat.com
     - Credentials: Select your created credential
     - Worker nodes: 2 replicas, 8Gi RAM, 2 cores
   - Click Create


