# Network Plumbing Chart

This Helm chart configures network interfaces and network attachment definitions for the multitenant cluster, including support for VLAN-based secondary networks and Cluster User Defined Networks (CUDNs) with localnet topology.

## Overview

The chart manages:
- **NMState configuration** for cluster network state management
- **VLAN interfaces** for secondary networks
- **Network Attachment Definitions (NADs)** for pod-level network attachments
- **Cluster User Defined Networks (CUDNs)** with localnet topology for tenant isolation

## Components

### 1. NMState Operator Configuration
Creates the base NMState custom resource for network state management.

### 2. VLAN 150 Network Configuration

Based on the cluster investigation, the pod network uses **VLAN 149 on interface eno12399**. This chart extends the network configuration to add **VLAN 150** support.

#### VLAN Interface and Dedicated Bridge (NNCP)
Creates a `NodeNetworkConfigurationPolicy` that:
- Creates VLAN interface `eno12399.150` on physical interface `eno12399`
- Creates a **dedicated OVS bridge `br-vlan150`** (separate from main cluster bridge `br-ex`)
- Attaches the VLAN interface to the dedicated bridge
- Provides isolation from the main cluster networking

#### Network Attachment Definition (NAD)
Creates a secondary network NAD with:
- **Topology**: `localnet` - maps to physical network via VLAN
- **VLAN ID**: 150
- **CNI**: `ovn-k8s-cni-overlay` for OVN-Kubernetes integration
- **Namespace**: Configurable (default: `secondary-network`)

#### Cluster User Defined Network (CUDN)
Creates a primary User Defined Network with:
- **Topology**: `localnet` - enables external network connectivity
- **Role**: `primary` - can be used as the primary network for pods
- **VLAN ID**: 150 - maps to the physical VLAN
- **Subnet**: Configurable isolated subnet (default: `10.150.0.0/16`)
- **Namespace**: Configurable (default: `udn-and-secondary-network`)

## Configuration

### Values Structure

```yaml
vlan150:
  enabled: true                          # Enable VLAN 150 configuration
  name: "vlan150-localnet"              # NAD name
  namespace: "secondary-network"         # Namespace for secondary network NAD
  vlanID: 150                           # VLAN ID
  subnet: "192.168.150.0/24"            # Subnet for secondary network
  mtu: 1500                             # MTU size
  
  nncp:
    enabled: true                        # Enable NodeNetworkConfigurationPolicy
    name: "vlan150-config"              # NNCP resource name
    baseInterface: "eno12399"           # Physical interface (discovered from cluster)
    vlanInterface: "eno12399.150"       # VLAN interface name
    bridgeName: "br-vlan150"            # Dedicated OVS bridge (separate from br-ex)
  
  cudn:
    enabled: true                        # Enable Cluster User Defined Network
    name: "vlan150-cudn"                # CUDN NAD name
    namespace: "udn-and-secondary-network"  # Namespace for CUDN
    subnet: "10.150.0.0/16"             # Isolated subnet for CUDN
```

### Usage in values-sno.yaml

Configure overrides in the application definition:

```yaml
applications:
  network-plumbing:
    name: network-plumbing
    namespace: openshift-nmstate
    project: infra
    path: charts/infra/network-plumbing
    overrides:
      - name: vlan150.enabled
        value: "true"
      - name: vlan150.vlanID
        value: "150"
      - name: vlan150.nncp.baseInterface
        value: "eno12399"
      - name: vlan150.nncp.vlanInterface
        value: "eno12399.150"
```

## Using the CUDN with Namespaces

To use the CUDN in a namespace, the namespace must be labeled appropriately:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-tenant
  labels:
    k8s.ovn.org/user-defined-network: "enabled"
```

Then reference the CUDN in pod annotations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: my-tenant
  annotations:
    k8s.v1.cni.cncf.io/networks: udn-and-secondary-network/vlan150-cudn
spec:
  # pod spec
```

## Architecture

### Network Flow

```
Physical Network
         │
    ┌────┴────┐
    │ VLAN 149│ VLAN 150 │
    └────┬────┴──────┬────┘
         │           │
         ↓           ↓
   eno12399.149  eno12399.150 (on eno12399 physical NIC)
         │           │
         ↓           ↓
      br-ex      br-vlan150  (Separate OVS Bridges)
    (cluster)   (dedicated)
         │           │
         ↓           ↓
    Cluster      VLAN 150
    Networks     Networks
                    │
           ┌────────┴────────┐
           ↓                 ↓
    Secondary NAD          CUDN
    (vlan150-           (vlan150-cudn)
     localnet)
```

**Key Architecture Points:**
- **br-ex**: Main cluster bridge (VLAN 149) - untouched, remains stable
- **br-vlan150**: Dedicated bridge for VLAN 150 - isolated from cluster networking
- **Benefits**: No risk to cluster networking, clean separation of concerns

### VLAN Configuration Discovery

The base interface was discovered by inspecting the cluster:

```bash
# Check network operator config
oc get network.operator.openshift.io cluster -o yaml

# Inspect node network state
oc get nns master-0 -o yaml | grep -A 30 "name: eno12399"
```

Output showed:
- Physical interface: `eno12399` (MAC: 30:3E:A7:24:DC:0C)
- Pod network VLAN: `eno12399.149` (VLAN ID: 149)
- Main OVS Bridge: `br-ex` (used by cluster)

**Design Decision**: We create a **dedicated bridge `br-vlan150`** for VLAN 150 to avoid modifying the main cluster bridge and ensure isolation from cluster networking.

## OpenShift 4.20 CUDN Localnet Support

This configuration leverages OpenShift 4.20's support for Cluster User Defined Networks with localnet topology. Key features:

- **Localnet topology**: Enables direct connection to external networks via VLAN
- **Primary role**: Can replace the default pod network for isolated tenants
- **Multi-tenancy**: Each namespace can have its own isolated network segment
- **External connectivity**: Direct access to physical network infrastructure

## Troubleshooting

### Check NNCP Status
```bash
oc get nncp vlan150-config
oc get nnce -A  # Check enactments on nodes
```

### Verify VLAN Interface
```bash
oc debug node/master-0 -- chroot /host ip link show eno12399.150
```

### Check NAD
```bash
oc get network-attachment-definitions -n secondary-network
oc get network-attachment-definitions -n udn-and-secondary-network
```

### Test Pod Connectivity
```bash
oc run test-pod \
  --image=registry.access.redhat.com/ubi9/ubi:latest \
  --annotations="k8s.v1.cni.cncf.io/networks=udn-and-secondary-network/vlan150-cudn" \
  -n udn-and-secondary-network \
  -- sleep infinity
```

## References

- [OpenShift 4.20 Multiple Networks Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/multiple_networks/)
- [OVN-Kubernetes Localnet Topology](https://docs.openshift.com/container-platform/4.20/networking/multiple_networks/configuring-additional-network.html)
- [Kubernetes NMState](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/kubernetes_nmstate/)

