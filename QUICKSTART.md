# Quick Start - HCP Infrastructure on SNO

## 3-Step Deployment

### Step 1: Configure (2 minutes)

```bash
# Find your disk device
oc debug node/$(oc get nodes -o name | head -1 | cut -d/ -f2) -- chroot /host lsblk

# Edit infrastructure config
vi overlays/infrastructure/kustomization.yaml

# Update these values:
# - Storage device: /dev/sdb → your disk
# - MetalLB IPs: 192.168.1.200-210 → your range
```

### Step 2: Deploy Infrastructure (15-25 minutes)

```bash
# Login to SNO cluster
oc login --token=<token> --server=<server>

# Deploy all infrastructure
oc apply -k overlays/infrastructure

# Watch deployment
watch oc get csv -A
```

### Step 3: Create HCP via Web Console (5 minutes + 30-40 min cluster creation)

1. **Create Credentials**:
   - Console → Infrastructure → Clusters → Credentials → Create
   - Type: "Red Hat OpenShift pull secret"
   - Paste your pull secret from cloud.redhat.com
   - (Optional) Add SSH public key

2. **Create Hosted Cluster**:
   - Console → Infrastructure → Clusters → Create cluster
   - Type: "Hosted control plane" → "OpenShift Virtualization"
   - Fill in:
     - **Name**: spoke1
     - **Base domain**: apps.baremetal-sno.gsslab.pnq2.redhat.com
     - **Credential**: Select your credential
     - **Workers**: 2 nodes, 8Gi RAM, 2 cores
   - Click "Create"

## Verify

```bash
# Check infrastructure
oc get lvmcluster,hyperconverged,metallb,mce

# Check hosted cluster (after creation)
oc get hostedcluster -n clusters
oc get pods -n clusters-spoke1

# Access hosted cluster
oc extract secret/spoke1-admin-kubeconfig -n clusters-spoke1 --to=- > spoke1-kubeconfig
export KUBECONFIG=spoke1-kubeconfig
oc get nodes
```

## Common Issues

**"OpenShift Virtualization operator is required"**:
```bash
# Ensure wildcard routes enabled
oc get ingresscontroller default -n openshift-ingress-operator \
  -o jsonpath='{.spec.routeAdmission.wildcardPolicy}'
# Should show: WildcardsAllowed

# If not, the infrastructure deployment includes it
```

**Cluster stuck "Pending Import"**:
- Normal! Takes 30-40 minutes for first cluster
- Check progress: `oc get pods -n clusters-spoke1`

**Storage issues**:
```bash
# Verify disk and LVM
oc get lvmcluster -n openshift-storage
oc get sc | grep lvms
```

## What Gets Deployed

| Component | Version | Namespace |
|-----------|---------|-----------|
| LVM Storage | Latest | openshift-storage |
| OpenShift Virtualization | 4.19+ | openshift-cnv |
| MetalLB | Latest | metallb-system |
| MultiCluster Engine | 2.9+ | multicluster-engine |

**Then you manually create**: Credentials + HCP clusters via web console

## Next Steps

- Create additional HCP clusters (spoke2, spoke3, etc.)
- Each cluster takes ~30-40 minutes
- Monitor: Console → Infrastructure → Clusters

## Resources

- [Full README](README.md) - Detailed documentation
- [HCP Docs](https://docs.openshift.com/container-platform/latest/hosted_control_planes/)
