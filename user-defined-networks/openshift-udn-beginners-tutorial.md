# OpenShift User Defined Networks (UDN) - Beginner's Tutorial

## Table of Contents
1. [Introduction](#introduction)
2. [What is UDN?](#what-is-udn)
3. [Prerequisites](#prerequisites)
4. [Understanding UDN Concepts](#understanding-udn-concepts)
5. [Example 1: Pod with UDN](#example-1-pod-with-udn)
6. [Example 2: KubeVirt Virtual Machine with UDN](#example-2-kubevirt-virtual-machine-with-udn)
7. [Verification and Testing](#verification-and-testing)
8. [Troubleshooting](#troubleshooting)
9. [Best Practices](#best-practices)
10. [Conclusion](#conclusion)

---

## Introduction

This tutorial introduces OpenShift User Defined Networks (UDN), a feature that allows you to create isolated network segments for your workloads without requiring additional physical NICs or network bonds. UDN works on top of OpenShift's default OVN-Kubernetes network, making it accessible on standard OpenShift clusters.

**What you'll learn:**
- How to create and configure User Defined Networks
- Deploy a Pod application with UDN
- Deploy a KubeVirt Virtual Machine with UDN
- Verify network connectivity and isolation
- Troubleshoot common issues

---

## What is UDN?

User Defined Networks (UDN) is a feature in OpenShift that enables you to:

- **Create isolated network segments** within your cluster
- **Assign custom IP ranges** to your workloads
- **Control network policies** at a granular level
- **Segment traffic** between different applications or tenants
- **Work with existing infrastructure** - no additional hardware required

UDN uses OVN (Open Virtual Network) to create overlay networks on top of your existing cluster network, providing network isolation without physical network changes.

### Key Benefits:
- **Multi-tenancy**: Isolate workloads from different teams or applications
- **Security**: Enhanced network segmentation and isolation
- **Flexibility**: Custom IP addressing schemes per network
- **Simplicity**: No physical network reconfiguration needed

---

## Prerequisites

Before starting this tutorial, ensure you have:

### Cluster Requirements:
- OpenShift 4.14 or later
- OVN-Kubernetes as the default CNI (Container Network Interface)
- Cluster admin privileges

### For KubeVirt Examples:
- OpenShift Virtualization Operator installed
- Sufficient resources for VM workloads

### Tools Required:
- `oc` CLI tool installed and configured
- `kubectl` (optional, but helpful)
- Basic understanding of Kubernetes/OpenShift concepts

### Verify Your Cluster:

```bash
# Check OpenShift version
oc version

# Verify OVN-Kubernetes is the network plugin
oc get network.config.openshift.io cluster -o jsonpath='{.spec.networkType}'
# Should output: OVNKubernetes

# Check if you have cluster-admin privileges
oc auth can-i create userDefinedNetwork
# Should output: yes
```

---

## Understanding UDN Concepts

### Network Architecture

```
┌────────────────────────────────────────────────────────┐
│                   OpenShift Cluster                    │
│                                                        │
│  ┌────────────────────────────────────────────────┐    │
│  │         Default Cluster Network                │    │
│  │         (OVN-Kubernetes)                       │    │
│  │                                                │    │
│  │  ┌──────────────┐      ┌──────────────┐        │    │
│  │  │   UDN-1      │      │   UDN-2      │        │    │
│  │  │ 10.100.0.0/16│      │ 10.200.0.0/16│        │    │
│  │  │              │      │              │        │    │
│  │  │  ┌────┐      │      │  ┌────┐      │        │    │
│  │  │  │Pod │      │      │  │ VM │      │        │    │
│  │  │  └────┘      │      │  └────┘      │        │    │
│  │  └──────────────┘      └──────────────┘        │    │
│  └────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────┘
```

### Key Components:

1. **UserDefinedNetwork (UDN)**: Custom resource that defines a network segment
2. **Network Attachment Definition (NAD)**: Connects workloads to the UDN
3. **Primary Network**: Default cluster network (always present)
4. **Secondary Network**: Additional UDN networks attached to workloads

### Network Modes:

- **Layer 2**: Workloads on the same network can communicate directly
- **Layer 3**: Routed network with gateway capabilities
- **Primary**: Replaces the default pod network (advanced use case)
- **Secondary**: Additional network interface (most common for beginners)

---

## Example 1: Pod with UDN

In this example, we'll create a User Defined Network and deploy a simple web application pod connected to it.

### Step 1: Create a Namespace

```bash
# Create a dedicated namespace for our UDN examples
oc create namespace udn-demo
oc project udn-demo
```

### Step 2: Create a User Defined Network

Create a file named `udn-network.yaml`:

```yaml
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: demo-network
  namespace: udn-demo
spec:
  topology: Layer2
  layer2:
    role: Secondary
    subnets:
    - "10.100.0.0/16"
```

Apply the configuration:

```bash
oc apply -f udn-network.yaml

# Verify the UDN was created
oc get userdefinednetwork -n udn-demo
```

**Expected output:**
```
NAME           AGE
demo-network   5s
```

### Step 3: Create a Network Attachment Definition

The Network Attachment Definition (NAD) is automatically created by the UDN controller, but let's verify it:

```bash
# Check the NAD
oc get network-attachment-definitions -n udn-demo

# View details
oc describe network-attachment-definitions demo-network -n udn-demo
```

### Step 4: Deploy a Pod with UDN

Create a file named `nginx-pod-udn.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-udn
  namespace: udn-demo
  annotations:
    k8s.v1.cni.cncf.io/networks: demo-network
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
      name: http
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
```

**Key annotation:**
- `k8s.v1.cni.cncf.io/networks: demo-network` - This attaches the pod to our UDN

Deploy the pod:

```bash
oc apply -f nginx-pod-udn.yaml

# Wait for the pod to be ready
oc wait --for=condition=Ready pod/nginx-udn -n udn-demo --timeout=60s

# Check pod status
oc get pod nginx-udn -n udn-demo
```

### Step 5: Verify Network Configuration

```bash
# Check pod network interfaces
oc exec -n udn-demo nginx-udn -- ip addr show

# You should see multiple interfaces:
# - eth0: Primary cluster network
# - net1: UDN network (10.100.x.x)

# Check routing table
oc exec -n udn-demo nginx-udn -- ip route show
```

**Expected output (partial):**
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    inet 127.0.0.1/8 scope host lo
2: eth0@if123: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP
    inet 10.128.2.45/23 brd 10.128.3.255 scope global eth0
3: net1@if124: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP
    inet 10.100.0.2/16 brd 10.100.255.255 scope global net1
```

### Step 6: Deploy a Second Pod for Testing

Create `test-pod-udn.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-client-udn
  namespace: udn-demo
  annotations:
    k8s.v1.cni.cncf.io/networks: demo-network
spec:
  containers:
  - name: test-client
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["/bin/bash", "-c", "sleep infinity"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"
```

Deploy and test connectivity:

```bash
oc apply -f test-pod-udn.yaml

# Wait for pod to be ready
oc wait --for=condition=Ready pod/test-client-udn -n udn-demo --timeout=60s

# Get the UDN IP of nginx pod
NGINX_UDN_IP=$(oc exec -n udn-demo nginx-udn -- ip -4 addr show net1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo "Nginx UDN IP: $NGINX_UDN_IP"

# Test connectivity from test-client to nginx via UDN
oc exec -n udn-demo test-client-udn -- curl -s http://$NGINX_UDN_IP
```

**Expected output:**
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

---

## Example 2: KubeVirt Virtual Machine with UDN

Now let's create a Virtual Machine using KubeVirt and connect it to our User Defined Network.

### Step 1: Verify OpenShift Virtualization

```bash
# Check if OpenShift Virtualization operator is installed
oc get csv -n openshift-cnv | grep kubevirt

# Check HyperConverged resource
oc get hyperconverged -n openshift-cnv
```

If not installed, you can install it via OperatorHub in the OpenShift Console or using the CLI.

### Step 2: Create a UDN for VMs

Create `vm-udn-network.yaml`:

```yaml
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: vm-network
  namespace: udn-demo
spec:
  topology: Layer2
  layer2:
    role: Secondary
    subnets:
    - "10.200.0.0/16"
```

Apply the configuration:

```bash
oc apply -f vm-udn-network.yaml

# Verify
oc get userdefinednetwork vm-network -n udn-demo
```

### Step 3: Create a Virtual Machine with UDN

Create `fedora-vm-udn.yaml`:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-vm-udn
  namespace: udn-demo
  labels:
    app: fedora-vm
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: fedora-vm-udn
      annotations:
        k8s.v1.cni.cncf.io/networks: vm-network
    spec:
      domain:
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
          - name: udn-interface
            bridge: {}
        resources:
          requests:
            memory: 1Gi
            cpu: "1"
          limits:
            memory: 2Gi
            cpu: "2"
      networks:
      - name: default
        pod: {}
      - name: udn-interface
        multus:
          networkName: vm-network
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/fedora:latest
      - name: cloudinitdisk
        cloudInitNoCloud:
          userDataBase64: I2Nsb3VkLWNvbmZpZwpwYXNzd29yZDogZmVkb3JhCmNocGFzc3dkOiB7IGV4cGlyZTogRmFsc2UgfQpzc2hfcHdhdXRoOiBUcnVl
```

**Key configurations:**
- `annotations: k8s.v1.cni.cncf.io/networks: vm-network` - Attaches VM to UDN
- `interfaces` section defines two interfaces: default (masquerade) and UDN (bridge)
- `networks` section maps interfaces to networks

**Note:** The `userDataBase64` is base64 encoded cloud-init config that sets password to "fedora"

Deploy the VM:

```bash
oc apply -f fedora-vm-udn.yaml

# Watch VM status
oc get vm -n udn-demo -w

# Check VMI (Virtual Machine Instance)
oc get vmi -n udn-demo
```

### Step 4: Access and Verify VM Network

```bash
# Wait for VM to be ready
oc wait --for=condition=Ready vmi/fedora-vm-udn -n udn-demo --timeout=300s

# Connect to VM console (use Ctrl+] to exit)
virtctl console fedora-vm-udn -n udn-demo

# Or use SSH if you prefer (from within the console):
# Login with user: fedora, password: fedora
```

Inside the VM console, verify network interfaces:

```bash
# Check network interfaces
ip addr show

# You should see:
# - eth0: Default pod network (masquerade)
# - eth1: UDN network (10.200.x.x)

# Check routing
ip route show
```

### Step 5: Test Connectivity Between VM and Pod

From your local terminal:

```bash
# Get VM's UDN IP
VM_UDN_IP=$(oc get vmi fedora-vm-udn -n udn-demo -o jsonpath='{.status.interfaces[?(@.name=="udn-interface")].ipAddress}')
echo "VM UDN IP: $VM_UDN_IP"

# Test connectivity from test-client pod to VM
oc exec -n udn-demo test-client-udn -- ping -c 4 $VM_UDN_IP
```

**Expected output:**
```
PING 10.200.0.x (10.200.0.x) 56(84) bytes of data.
64 bytes from 10.200.0.x: icmp_seq=1 ttl=64 time=0.234 ms
64 bytes from 10.200.0.x: icmp_seq=2 ttl=64 time=0.189 ms
...
```

### Step 6: Test VM to Pod Connectivity

Inside the VM console:

```bash
# Get nginx pod's UDN IP (you noted this earlier)
# Or from another terminal: oc exec -n udn-demo nginx-udn -- ip -4 addr show net1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}'

# From VM, test connectivity to nginx pod
curl http://10.100.0.2  # Replace with actual nginx UDN IP
```

---

## Verification and Testing

### Comprehensive Network Testing

Create a test script `test-udn-connectivity.sh`:

```bash
#!/bin/bash

NAMESPACE="udn-demo"

echo "=== UDN Connectivity Test ==="
echo ""

# Get IPs
NGINX_UDN_IP=$(oc exec -n $NAMESPACE nginx-udn -- ip -4 addr show net1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
VM_UDN_IP=$(oc get vmi fedora-vm-udn -n $NAMESPACE -o jsonpath='{.status.interfaces[?(@.name=="udn-interface")].ipAddress}')

echo "Nginx UDN IP: $NGINX_UDN_IP"
echo "VM UDN IP: $VM_UDN_IP"
echo ""

# Test 1: Pod to Pod via UDN
echo "Test 1: Pod to Pod connectivity via UDN"
if oc exec -n $NAMESPACE test-client-udn -- curl -s -m 5 http://$NGINX_UDN_IP > /dev/null; then
    echo "✓ SUCCESS: test-client can reach nginx via UDN"
else
    echo "✗ FAILED: test-client cannot reach nginx via UDN"
fi
echo ""

# Test 2: Pod to VM via UDN
echo "Test 2: Pod to VM connectivity via UDN"
if oc exec -n $NAMESPACE test-client-udn -- ping -c 2 -W 5 $VM_UDN_IP > /dev/null 2>&1; then
    echo "✓ SUCCESS: test-client can reach VM via UDN"
else
    echo "✗ FAILED: test-client cannot reach VM via UDN"
fi
echo ""

# Test 3: Check network isolation
echo "Test 3: Network isolation verification"
echo "Checking that UDN traffic is isolated from default network..."
DEFAULT_NGINX_IP=$(oc get pod nginx-udn -n $NAMESPACE -o jsonpath='{.status.podIP}')
echo "Nginx default network IP: $DEFAULT_NGINX_IP"
echo "Nginx UDN IP: $NGINX_UDN_IP"

if [ "$DEFAULT_NGINX_IP" != "$NGINX_UDN_IP" ]; then
    echo "✓ SUCCESS: UDN provides separate IP space"
else
    echo "✗ FAILED: IPs are the same (unexpected)"
fi
echo ""

echo "=== Test Complete ==="
```

Make it executable and run:

```bash
chmod +x test-udn-connectivity.sh
./test-udn-connectivity.sh
```

### Verify Network Policies

UDN respects NetworkPolicies. Let's test this:

Create `network-policy-test.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-udn-traffic
  namespace: udn-demo
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  ingress: []  # Deny all ingress
```

```bash
# Apply the policy
oc apply -f network-policy-test.yaml

# Test connectivity (should fail now)
oc exec -n udn-demo test-client-udn -- curl -s -m 5 http://$NGINX_UDN_IP

# Remove the policy to restore connectivity
oc delete networkpolicy deny-udn-traffic -n udn-demo
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: Pod Not Getting UDN IP

**Symptoms:**
- Pod starts but doesn't have net1 interface
- `oc exec pod -- ip addr` shows only eth0

**Solutions:**

```bash
# Check if UDN exists
oc get userdefinednetwork -n udn-demo

# Check NAD
oc get network-attachment-definitions -n udn-demo

# Verify pod annotation
oc get pod <pod-name> -n udn-demo -o yaml | grep -A 5 annotations

# Check pod events
oc describe pod <pod-name> -n udn-demo

# Check multus logs
oc logs -n openshift-multus -l app=multus --tail=50
```

**Fix:** Ensure the annotation is correct:
```yaml
annotations:
  k8s.v1.cni.cncf.io/networks: demo-network  # Must match UDN name
```

#### Issue 2: VM Not Starting

**Symptoms:**
- VM stuck in "Starting" state
- VMI not created

**Solutions:**

```bash
# Check VM status
oc describe vm fedora-vm-udn -n udn-demo

# Check VMI
oc get vmi -n udn-demo

# Check virt-launcher pod
oc get pods -n udn-demo | grep virt-launcher

# Check virt-launcher logs
oc logs -n udn-demo virt-launcher-fedora-vm-udn-xxxxx

# Check events
oc get events -n udn-demo --sort-by='.lastTimestamp'
```

**Common fixes:**
- Ensure sufficient resources (CPU, memory)
- Verify OpenShift Virtualization is properly installed
- Check if container disk image is accessible

#### Issue 3: No Connectivity Between Workloads

**Symptoms:**
- Pods/VMs have UDN IPs but can't communicate

**Solutions:**

```bash
# Verify both workloads are on the same UDN
oc get pod <pod-name> -n udn-demo -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/networks}'

# Check if IPs are in the same subnet
oc exec -n udn-demo <pod-name> -- ip addr show net1

# Test basic connectivity
oc exec -n udn-demo <pod-name> -- ping -c 2 <target-ip>

# Check OVN logs
oc logs -n openshift-ovn-kubernetes -l app=ovnkube-node --tail=100

# Verify no NetworkPolicies are blocking traffic
oc get networkpolicies -n udn-demo
```

#### Issue 4: UDN Creation Fails

**Symptoms:**
- `oc apply` succeeds but UDN doesn't work
- NAD not created automatically

**Solutions:**

```bash
# Check UDN status
oc get userdefinednetwork -n udn-demo -o yaml

# Check OVN-Kubernetes operator
oc get pods -n openshift-ovn-kubernetes

# Check cluster network operator
oc get clusteroperator network

# View operator logs
oc logs -n openshift-network-operator deployment/network-operator
```

### Debug Commands Reference

```bash
# Check all UDNs in namespace
oc get userdefinednetwork -n udn-demo

# Check all NADs in namespace
oc get network-attachment-definitions -n udn-demo

# List all pods with their IPs
oc get pods -n udn-demo -o wide

# Get detailed pod network info
oc exec -n udn-demo <pod-name> -- ip addr
oc exec -n udn-demo <pod-name> -- ip route

# Check OVN-Kubernetes components
oc get pods -n openshift-ovn-kubernetes

# View multus logs
oc logs -n openshift-multus -l app=multus --tail=100

# Check network operator status
oc get clusteroperator network
oc describe clusteroperator network
```

---

## Best Practices

### 1. Network Planning

- **Plan IP ranges carefully**: Avoid overlapping with existing networks
- **Use appropriate subnet sizes**: /16 for large deployments, /24 for smaller ones
- **Document your networks**: Keep track of which UDNs are used for what purpose

```yaml
# Good practice: Use descriptive names and comments
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: frontend-network  # Clear, descriptive name
  namespace: production
  labels:
    environment: production
    tier: frontend
spec:
  topology: Layer2
  layer2:
    role: Secondary
    subnets:
    - "10.100.0.0/16"  # Frontend apps: 10.100.0.0/16
```

### 2. Resource Management

- **Set resource limits**: Always define CPU and memory limits for pods and VMs
- **Use namespaces**: Organize UDNs by namespace for better isolation
- **Monitor usage**: Track network performance and resource consumption

```yaml
# Always include resource limits
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### 3. Security

- **Apply NetworkPolicies**: Control traffic between workloads
- **Use RBAC**: Limit who can create UDNs
- **Audit regularly**: Review UDN configurations periodically

```yaml
# Example: Restrict UDN creation to specific users
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: udn-creator
  namespace: udn-demo
rules:
- apiGroups: ["k8s.ovn.org"]
  resources: ["userdefinednetworks"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
```

### 4. Naming Conventions

Use consistent naming:
- UDN names: `<purpose>-network` (e.g., `frontend-network`, `database-network`)
- Namespaces: `<team>-<environment>` (e.g., `team-a-prod`, `team-b-dev`)
- Labels: Use labels for organization and selection

### 5. Testing

- **Test in dev first**: Always test UDN configurations in development
- **Verify connectivity**: Test all expected communication paths
- **Document test results**: Keep records of what works and what doesn't

### 6. Monitoring

```bash
# Monitor UDN resources
oc get userdefinednetwork --all-namespaces

# Check network attachment status
oc get network-attachment-definitions --all-namespaces

# Monitor pod network interfaces
for pod in $(oc get pods -n udn-demo -o name); do
  echo "=== $pod ==="
  oc exec -n udn-demo $pod -- ip addr show 2>/dev/null || echo "Cannot access pod"
done
```

---

## Conclusion

Congratulations! You've completed the OpenShift User Defined Networks beginner's tutorial. You've learned:

✅ **What UDN is** and why it's useful
✅ **How to create** User Defined Networks
✅ **How to deploy Pods** with UDN connectivity
✅ **How to deploy KubeVirt VMs** with UDN
✅ **How to verify** network connectivity and isolation
✅ **How to troubleshoot** common issues
✅ **Best practices** for UDN management

### Next Steps

1. **Experiment with Layer 3 networks**: Try creating routed networks
2. **Implement NetworkPolicies**: Add fine-grained traffic control
3. **Multi-network scenarios**: Connect workloads to multiple UDNs
4. **Performance testing**: Measure network throughput and latency
5. **Integration with service mesh**: Explore UDN with Istio or other service meshes

### Additional Resources

- [OpenShift Documentation - OVN-Kubernetes](https://docs.openshift.com/container-platform/latest/networking/ovn_kubernetes_network_provider/about-ovn-kubernetes.html)
- [OpenShift Virtualization Documentation](https://docs.openshift.com/container-platform/latest/virt/about_virt/about-virt.html)
- [Multus CNI Documentation](https://github.com/k8snetworkplumbingwg/multus-cni)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

### Cleanup

When you're done experimenting, clean up the resources:

```bash
# Delete all resources in the demo namespace
oc delete namespace udn-demo

# This will remove:
# - All UDNs
# - All pods
# - All VMs
# - All network attachment definitions
```

---

## Appendix: Quick Reference

### Common Commands

```bash
# Create UDN
oc apply -f udn-network.yaml

# List UDNs
oc get userdefinednetwork -n <namespace>

# Describe UDN
oc describe userdefinednetwork <name> -n <namespace>

# Delete UDN
oc delete userdefinednetwork <name> -n <namespace>

# Check pod network interfaces
oc exec -n <namespace> <pod-name> -- ip addr

# Get VM network info
oc get vmi <vm-name> -n <namespace> -o jsonpath='{.status.interfaces}'

# Test connectivity
oc exec -n <namespace> <pod-name> -- ping <target-ip>
oc exec -n <namespace> <pod-name> -- curl http://<target-ip>
```

### YAML Templates

**Minimal UDN:**
```yaml
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: my-network
  namespace: my-namespace
spec:
  topology: Layer2
  layer2:
    role: Secondary
    subnets:
    - "10.100.0.0/16"
```

**Pod with UDN:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: my-network
spec:
  containers:
  - name: app
    image: nginx:latest
```

**VM with UDN:**
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-vm
  annotations:
    k8s.v1.cni.cncf.io/networks: my-network
spec:
  running: true
  template:
    spec:
      domain:
        devices:
          interfaces:
          - name: default
            masquerade: {}
          - name: udn
            bridge: {}
      networks:
      - name: default
        pod: {}
      - name: udn
        multus:
          networkName: my-network
```
