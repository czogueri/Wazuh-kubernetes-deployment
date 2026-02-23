# Wazuh-kubernetes-deployment
This repository documents a complete, production-like deployment of **Wazuh** (open-source SIEM/XDR platform) on a lightweight **K3s** Kubernetes cluster running across three virtual machines. The setup includes the Wazuh manager (master + worker), indexer, and dashboard, exposed via LoadBalancer services using MetalLB on bare-metal VMs.
# Deploying Wazuh on a 3-Node K3s Cluster with MetalLB


The goal was to demonstrate hands-on Kubernetes skills: cluster bootstrapping, add-on installation, storage management with local-path provisioner, StatefulSet persistence, node affinity/taint troubleshooting, and security certificate handling.

## Architecture Overview

- **Cluster**: K3s (lightweight Kubernetes) with 1 control-plane node + 2 agents
- **Nodes** (3 VMs on local network 192.168.1.0/24):
  - `ubuntu-vm` (192.168.1.181) – control-plane
  - `minty` (192.168.1.210) – worker
  - `pve2` (192.168.1.206) – worker
- **Components**:
  - Wazuh Manager (StatefulSet: master + worker for scalability)
  - Wazuh Indexer (StatefulSet: OpenSearch backend)
  - Wazuh Dashboard (Deployment: Kibana-like UI)
- **Networking**: MetalLB (Layer 2 mode) for assigning real external IPs to LoadBalancer services
- **Storage**: K3s default `local-path` provisioner (hostPath-based PVs)
- **Certificates**: Self-signed generated via official Wazuh scripts

![High-level architecture](https://via.placeholder.com/800x400.png?text=Wazuh+on+K3s+Architecture)  
*(Replace with a real diagram created in draw.io / excalidraw if desired)*

## Prerequisites

- 3 VMs with Ubuntu/Mint/Kali (or similar Linux), ≥4 CPU, ≥8 GB RAM, ≥50 GB disk each recommended
- Static IPs on the same L2 subnet (192.168.1.0/24 in this case)
- SSH access and sudo privileges
- `kubectl` configured on the control-plane node

## Step-by-Step Deployment

### 1. Install K3s Cluster

On the control-plane VM (`ubuntu-vm`):`curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644`



### 2. Install MetalLB for LoadBalancer Support

MetalLB provides external IPs in bare-metal environments (no cloud LB): `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml`


### 3. Deploy Wazuh

Clone official manifests (v4.14.3 – latest stable at time of deployment): `git clone https://github.com/wazuh/wazuh-kubernetes.git -b v4.14.3 --depth=1`


### 4. Troubleshooting 

During deployment, several real-world Kubernetes issues were encountered and resolved:

Pods Pending due to unbound PVCs → Confirmed rancher.io/local-path was default StorageClass.

FailedScheduling: volume node affinity conflict + untolerated taint → local-path PVs are node-specific (affinity locked to ubuntu-vm). Added nodeSelector: kubernetes.io/hostname: ubuntu-vm to wazuh-indexer and wazuh-manager-worker StatefulSets.

node.kubernetes.io/disk-pressure:NoSchedule taint → Caused by low disk space on control-plane node. Expanded VM disk, pruned container images (k3s crictl rmi --prune), cleaned logs → taint auto-removed by kubelet.

Dashboard "not ready yet" → Normal during indexer initialization (OpenSearch health check). Waited for wazuh-indexer-0 → Running and green cluster health.

![image alt](https://github.com/czogueri/Wazuh-kubernetes-deployment/blob/b0f32cdcd96b8a4056cb80ae86f00de3531844e0/Screenshot%202026-02-22%20211628.png)
