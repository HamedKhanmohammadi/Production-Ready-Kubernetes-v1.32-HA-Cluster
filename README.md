```markdown
# The Definitive Guide: Production-Ready Kubernetes v1.32 HA Cluster

This guide provides a complete, robust setup for a highly available (HA) Kubernetes v1.32 cluster using `kubeadm`. The architecture features an external `etcd` cluster, `containerd`, `Cilium` CNI, and a fault-tolerant load balancing layer with `HAProxy` and `keepalived`.

---

### 1. Cluster Architecture and IP Plan

-   **Control Plane:** 3 nodes
-   **Worker Nodes:** 3 nodes
-   **External etcd:** 3 nodes
-   **Load Balancer:** 2 nodes
-   **Virtual IP (VIP):** `192.168.55.20`

| Role              | Hostname  | IP Address      |
| ----------------- | :-------- | :-------------- |
| **Load Balancer 1** | `lb1`     | `192.168.55.44` |
| **Load Balancer 2** | `lb2`     | `192.168.55.45` |
| **etcd 1** | `etcd1`   | `192.168.55.41` |
| **etcd 2** | `etcd2`   | `192.168.55.42` |
| **etcd 3** | `etcd3`   | `192.168.55.43` |
| **Master 1** | `master1` | `192.168.55.24` |
| **Master 2** | `master2` | `192.168.55.25` |
| **Master 3** | `master3` | `192.168.55.26` |
| **Worker 1** | `worker1` | `192.168.55.31` |
| **Worker 2** | `worker2` | `192.168.55.32` |
| **Worker 3** | `worker3` | `192.168.55.33` |

---

### 2. Prerequisite Steps (Run on ALL Nodes)

#### 1. Set Hostname
Run on each respective node:
```bash
sudo hostnamectl set-hostname <hostname>
```

#### 2. Configure `/etc/hosts`
Add this configuration to all nodes to enable local name resolution.
```bash
sudo tee -a /etc/hosts > /dev/null <<EOT

# Kubernetes HA Cluster
192.168.55.20  lb-vip
192.168.55.24  master1
192.168.55.25  master2
192.168.55.26  master3
192.168.55.31  worker1
192.168.55.32  worker2
192.168.55.33  worker3
192.168.55.41  etcd1
192.168.55.42  etcd2
192.168.55.43  etcd3
192.168.55.44  lb1
192.168.55.45  lb2
EOT
```

#### 3. Disable Swap
Kubernetes requires swap to be disabled.
```bash
sudo swapoff -a
sudo sed -i.bak '/\sswap\s/s/^#?/#/' /etc/fstab
```

#### 4. Load Kernel Modules and Configure `sysctl`
These settings are required for container networking.
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

#### 5. Install `containerd` Runtime
```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install -y containerd.io
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

#### 6. Install Kubernetes Tools (`kubeadm`, `kubelet`, `kubectl`)
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.32/deb/](https://pkgs.k8s.io/core:/stable:/v1.32/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.32.4-1.1 kubeadm=1.32.4-1.1 kubectl=1.32.4-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

---

### 3. Firewall Configuration (UFW)

**On ALL Master Nodes:**
```bash
sudo ufw allow 6443/tcp     # Kubernetes API Server
sudo ufw allow 2379:2380/tcp # etcd for clients & peers
sudo ufw allow 10250/tcp   # Kubelet API
sudo ufw allow 10257/tcp   # kube-controller-manager
sudo ufw allow 10259/tcp   # kube-scheduler
sudo ufw allow 8472/udp    # Cilium VXLAN
```
**On ALL Worker Nodes:**
```bash
sudo ufw allow 10250/tcp         # Kubelet API
sudo ufw allow 30000:32767/tcp   # NodePort Services
sudo ufw allow 8472/udp          # Cilium VXLAN
```
**On ALL etcd Nodes:**
```bash
sudo ufw allow 2379:2380/tcp # etcd client/peer communication
```
**On ALL Load Balancer Nodes:**
```bash
sudo ufw allow 6443/tcp # API Server Load Balancer
sudo ufw allow vrrp     # Keepalived protocol
```
**Finally, enable UFW on all nodes:**
```bash
sudo ufw enable
```

---

### 4. Set Up the External etcd Cluster (on `etcd1`, `etcd2`, `etcd3`)

**Prerequisite:** This step requires TLS certificates. These can be generated using tools like `cfssl` or `openssl`. You must have a CA and server/client certificates generated beforehand.

-   **Required files on each etcd node:** `/etc/etcd/certs/ca.crt`, `/etc/etcd/certs/etcd-server.crt`, `/etc/etcd/certs/etcd-server.key`.
-   **Required files for the control plane:** `/etc/kubernetes/pki/etcd/ca.crt`, `/etc/kubernetes/pki/etcd/apiserver-etcd-client.crt`, `/etc/kubernetes/pki/etcd/apiserver-etcd-client.key`.

#### 1. Install `etcd` binaries
```bash
ETCD_VER=v3.5.14
curl -L [https://github.com/etcd-io/etcd/releases/download/$](https://github.com/etcd-io/etcd/releases/download/$){ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz
tar xzvf /tmp/etcd.tar.gz -C /tmp --strip-components=1
sudo mv /tmp/etcd /usr/local/bin/ && sudo mv /tmp/etcdctl /usr/local/bin/
```

#### 2. Create `systemd` Service
Run on each etcd node, updating `INTERNAL_IP` and `NODE_NAME`.
```bash
INTERNAL_IP="<YOUR_NODE_IP>" # e.g., 192.168.55.41
NODE_NAME="<YOUR_NODE_HOSTNAME>"   # e.g., etcd1
sudo mkdir -p /etc/etcd/certs /var/lib/etcd

cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=[https://github.com/etcd-io/etcd](https://github.com/etcd-io/etcd)
[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${NODE_NAME} \\
  --data-dir /var/lib/etcd \\
  --listen-client-urls https://${INTERNAL_IP}:2379,[https://127.0.0.1:2379](https://127.0.0.1:2379) \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --initial-cluster etcd1=[https://192.168.55.41:2380](https://192.168.55.41:2380),etcd2=[https://192.168.55.42:2380](https://192.168.55.42:2380),etcd3=[https://192.168.55.43:2380](https://192.168.55.43:2380) \\
  --initial-cluster-token my-etcd-token-123 \\
  --initial-cluster-state new \\
  --client-cert-auth \\
  --trusted-ca-file=/etc/etcd/certs/ca.crt \\
  --cert-file=/etc/etcd/certs/etcd-server.crt \\
  --key-file=/etc/etcd/certs/etcd-server.key \\
  --peer-client-cert-auth \\
  --peer-trusted-ca-file=/etc/etcd/certs/ca.crt \\
  --peer-cert-file=/etc/etcd/certs/etcd-server.crt \\
  --peer-key-file=/etc/etcd/certs/etcd-server.key
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

#### 3. Start and Verify `etcd`
```bash
sudo systemctl daemon-reload && sudo systemctl enable --now etcd

# Verify health from a master node where client certs are present
ETCDCTL_API=3 etcdctl --endpoints="[https://192.168.55.41:2379](https://192.168.55.41:2379),[https://192.168.55.42:2379](https://192.168.55.42:2379),[https://192.168.55.43:2379](https://192.168.55.43:2379)" \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/apiserver-etcd-client.crt \
  --key=/etc/kubernetes/pki/etcd/apiserver-etcd-client.key \
  endpoint health --cluster
```

---

### 5. Set Up HA Load Balancer (on `lb1` and `lb2`)

#### 1. Install `HAProxy` and `keepalived`
```bash
sudo apt-get update && sudo apt-get install -y haproxy keepalived
```

#### 2. Configure `HAProxy`
Run on both `lb1` and `lb2`.
```bash
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<EOT
frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 192.168.55.24:6443 check fall 3 rise 2
    server master2 192.168.55.25:6443 check fall 3 rise 2
    server master3 192.168.55.26:6443 check fall 3 rise 2
EOT
```

#### 3. Configure `keepalived` (with HAProxy health check)
> **Note:** Find your network interface name with `ip a | grep mtu`. Replace `ens160` below with your actual interface name.

First, create the health check script on both `lb1` and `lb2`:
```bash
sudo tee /etc/keepalived/check_haproxy.sh > /dev/null <<'EOT'
#!/bin/sh
if pgrep haproxy > /dev/null; then
    exit 0
else
    exit 1
fi
EOT
sudo chmod +x /etc/keepalived/check_haproxy.sh
```

Next, configure `keepalived.conf` on `lb1` (MASTER):
```bash
sudo tee /etc/keepalived/keepalived.conf > /dev/null <<'EOT'
vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass "SUPER_SECRET_PASSWORD_123"
    }
    virtual_ipaddress {
        192.168.55.20/24
    }
    track_script {
        chk_haproxy
    }
}
EOT
```
And configure `keepalived.conf` on `lb2` (BACKUP):
```bash
sudo tee /etc/keepalived/keepalived.conf > /dev/null <<'EOT'
vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass "SUPER_SECRET_PASSWORD_123"
    }
    virtual_ipaddress {
        192.168.55.20/24
    }
    track_script {
        chk_haproxy
    }
}
EOT
```

#### 4. Start and Enable Services
```bash
sudo systemctl restart haproxy keepalived
sudo systemctl enable haproxy keepalived
```

---

### 6. Bootstrap the Kubernetes Control Plane

#### 1. Create `kubeadm-config.yaml` on `master1`
```bash
# Ensure your etcd client certs are in /etc/kubernetes/pki/etcd on master1
cat <<EOF | sudo tee kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "1.32.4"
controlPlaneEndpoint: "192.168.55.20:6443"
etcd:
    external:
        endpoints:
            - [https://192.168.55.41:2379](https://192.168.55.41:2379)
            - [https://192.168.55.42:2379](https://192.168.55.42:2379)
            - [https://192.168.55.43:2379](https://192.168.55.43:2379)
        caFile: /etc/kubernetes/pki/etcd/ca.crt
        certFile: /etc/kubernetes/pki/etcd/apiserver-etcd-client.crt
        keyFile: /etc/kubernetes/pki/etcd/apiserver-etcd-client.key
networking:
    podSubnet: "10.10.0.0/16"
EOF
```

#### 2. Initialize the first master node (`master1`)
```bash
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```

#### 3. Configure `kubectl` access (on `master1`)
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### 7. Join Remaining Nodes to the Cluster

#### 1. Join other Control Plane nodes (`master2`, `master3`)

First, get the join command and certificate key from `master1`:
```bash
# This gives you the main join command
kubeadm token create --print-join-command

# This gives you the certificate key for joining other masters
sudo kubeadm init phase upload-certs --upload-certs
```
Combine these pieces and run the following command on `master2` and `master3`:
```bash
# Example command structure. Use the values you just generated.
sudo kubeadm join 192.168.55.20:6443 \
    --token <your-token> \
    --discovery-token-ca-cert-hash sha256:<your-hash> \
    --control-plane \
    --certificate-key <your-cert-key>
```

#### 2. Join Worker nodes (`worker1`, `worker2`, `worker3`)

Use the standard join command generated earlier (or create a new token if needed). Run this on each worker node.
```bash
sudo kubeadm join 192.168.55.20:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-hash>
```

---

### 8. Install CNI Plugin: Cilium

Run on `master1`.
```bash
CILIUM_CLI_VERSION=$(curl -s [https://api.github.com/repos/cilium/cilium-cli/releases/latest](https://api.github.com/repos/cilium/cilium-cli/releases/latest) | grep tag_name | cut -d '"' -f 4)
curl -L --remote-name-all [https://github.com/cilium/cilium-cli/releases/download/$](https://github.com/cilium/cilium-cli/releases/download/$){CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}

cilium install
cilium status --wait
```

---

### 9. Install Essential Add-ons

#### 1. Metrics Server, Ingress, Cert-Manager
```bash
# Metrics Server
kubectl apply -f [https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml](https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml)
# Ingress NGINX
kubectl apply -f [https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/cloud/deploy.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/cloud/deploy.yaml)
# Cert-Manager
kubectl apply -f [https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml](https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml)
```
#### 2. MetalLB (for on-prem `LoadBalancer` services)
```bash
# Install
kubectl apply -f [https://raw.githubusercontent.com/metallb/metallb/v0.14.7/config/manifests/metallb-native.yaml](https://raw.githubusercontent.com/metallb/metallb/v0.14.7/config/manifests/metallb-native.yaml)
# Configure
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.55.200-192.168.55.220
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF
```

---

### 10. Final Steps and Recommendations

#### 1. Configure Bash Completion (Highly Recommended)
Run this on any machine where you will use `kubectl`.
```bash
sudo apt-get update && sudo apt-get install -y bash-completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'source <(kubeadm completion bash)' >>~/.bashrc
source ~/.bashrc
```

#### 2. Label Worker Nodes (Good Practice)
```bash
kubectl label node worker1 worker2 worker3 node-role.kubernetes.io/worker=worker
```

#### 3. Verify Final Cluster Status
```bash
# All nodes should be in 'Ready' state
kubectl get nodes -o wide

# Check all system pods are running
kubectl get pods -A
```
