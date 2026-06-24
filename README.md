# MetalLB-K8s-Setup
Here is a simple, step-by-step guide to installing and configuring MetalLB in Kubernetes:

### 1. Preparation (Only if using IPVS mode)
If your cluster uses `kube-proxy` in **IPVS mode**, you must enable strict ARP. Edit the config map:
```bash
kubectl edit configmap -n kube-system kube-proxy
```
Set `strictARP: true` under the `ipvs` section:
```yaml
ipvs:
  strictARP: true
```
*(Note: Not required if you use kube-router or standard iptables mode).*

### 2. Install MetalLB
Apply the official manifest to deploy MetalLB to the `metallb-system` namespace. The **native** manifest is the simplest choice for Layer 2 setups:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
```
Wait for the pods to be ready:
```bash
kubectl get pods -n metallb-system
```

### 3. Configure IP Address Pool & Layer 2
MetalLB needs a pool of IP addresses to assign to your `LoadBalancer` services. Create a file named `metallb-config.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  # Replace with an unused IP range from your local network
  - 192.168.1.240-192.168.1.250 
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2-advertisement
  namespace: metallb-system
```

Apply the configuration:
```bash
kubectl apply -f metallb-config.yaml
```

### 4. Verify
Create a simple service with `type: LoadBalancer`. MetalLB will automatically assign an IP from your configured pool to the `EXTERNAL-IP`:
```bash
kubectl get svc
```
---
Installing MetalLB on **Talos Linux** requires a slightly different approach than standard Kubernetes because Talos is an immutable OS. You cannot simply `kubectl edit` the `kube-proxy` configmap, as Talos will overwrite it during reconciliation.

Here is the Talos-native guide to installing and configuring MetalLB.

### 1. Enable Strict ARP (Only if using IPVS mode)
If your Talos cluster uses `kube-proxy` in **IPVS mode**, you must enable strict ARP via the Talos machine config. *(If you are using the default `iptables` mode, you can skip this step).*

Create a patch file named `talos-proxy-patch.yaml`:
```yaml
cluster:
  proxy:
    extraArgs:
      ipvs-strict-arp: "true"
      proxy-mode: "ipvs" # Add this if you are also switching to IPVS
```
Apply it to your nodes:
```bash
talosctl patch mc -n <node-ip> --patch @talos-proxy-patch.yaml
```

### 2. Fix Control Plane Label (Crucial for Talos)
By default, Talos applies the label `node.kubernetes.io/exclude-from-external-load-balancers` to control plane nodes. If you have a single-node cluster or run workloads on control plane nodes, MetalLB won't announce the IPs until you remove this label.

Create a patch file named `talos-lb-patch.yaml`:
```yaml
machine:
  nodeLabels:
    node.kubernetes.io/exclude-from-external-load-balancers:
      $patch: delete
```
Apply it to your control plane node(s):
```bash
talosctl patch mc -n <control-plane-ip> --patch @talos-lb-patch.yaml
```

### 3. Install MetalLB
Apply the official manifest to deploy MetalLB to the `metallb-system` namespace:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
```
Wait for the pods to be ready:
```bash
kubectl get pods -n metallb-system
```

### 4. Configure IP Address Pool & Layer 2
MetalLB needs a pool of IP addresses to assign to your `LoadBalancer` services. Create a file named `metallb-config.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  # Replace with an unused IP range from your local network
  - 192.168.1.240-192.168.1.250 
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2-advertisement
  namespace: metallb-system
```

Apply the configuration:
```bash
kubectl apply -f metallb-config.yaml
```

### 5. Verify
Create a simple service with `type: LoadBalancer`. MetalLB will automatically assign an IP from your configured pool to the `EXTERNAL-IP`:
```bash
kubectl get svc
```

---
**💡 Note on CNI:** If you are using **Cilium** as your CNI, it has built-in LoadBalancer capabilities (LB IPAM / BGP). You may not need MetalLB at all and can use Cilium's native LoadBalancer instead.
