# Production-Ready Kubernetes v1.32 HA Cluster

This guide provides a complete, robust setup for a highly available (HA) Kubernetes v1.32 cluster using kubeadm. The architecture features an external etcd cluster, containerd, Cilium CNI, and a fault-tolerant load balancing layer with HAProxy and keepalived.

## 1. Cluster Architecture and IP Plan

- **Control Plane**: 3 nodes  
- **Worker Nodes**: 3 nodes  
- **External etcd**: 3 nodes  
- **Load Balancer**: 2 nodes  
- **Virtual IP (VIP)**: `192.168.55.20`

| Role             | Hostname | IP Address     |
|------------------|----------|----------------|
| Load Balancer 1  | lb1      | 192.168.55.44  |
| Load Balancer 2  | lb2      | 192.168.55.45  |
| etcd 1           | etcd1    | 192.168.55.41  |
| etcd 2           | etcd2    | 192.168.55.42  |
| etcd 3           | etcd3    | 192.168.55.43  |
| Master 1         | master1  | 192.168.55.24  |
| Master 2         | master2  | 192.168.55.25  |
| Master 3         | master3  | 192.168.55.26  |
| Worker 1         | worker1  | 192.168.55.31  |
| Worker 2         | worker2  | 192.168.55.32  |
| Worker 3         | worker3  | 192.168.55.33  |

...

✅ **Congratulations!** Your robust, production-ready Kubernetes v1.32 High Availability Cluster is now fully operational.
