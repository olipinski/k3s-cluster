# K3s Cluster Deployment Guide

This guide provides instructions for deploying a secure, heterogeneous K3s Kubernetes cluster using Ansible.

## Prerequisites

- Servers provisioned with MAAS (Metal as a Service)
- SSH access to all nodes
- Ansible installed on your control node
- `kubectl` installed for cluster management

## Cluster Architecture

- **Server nodes**: Control plane nodes running etcd and Kubernetes control components
- **Agent nodes**: Worker nodes that run application workloads
- **Support for mixed architectures**: Both amd64 and arm64 architectures supported with proper tainting

## Deployment Steps

### 1. Configure Inventory

Edit the `inventory/cluster/hosts.yaml` file to define your cluster nodes:

```yaml
server:
  hosts:
    apollo:
    boreas:
    cerus:

agent:
  hosts:
    chaos:
    crios:
    helios:
    hermes:
    hypnos:

cluster:
  children:
    server:
    agent:
```

### 2. Configure Global Settings

Edit the `inventory/cluster/group_vars/all.yaml` file to set encryption keys and credentials:

```yaml
global_map:
  credentials:
    # Various encrypted credentials
  k3s:
    encryption:
      key: !vault |
        $ANSIBLE_VAULT;1.1;AES256
        [encrypted-key-here]
```

### 3. Deploy the Cluster Infrastructure

Run the provisioning playbook to set up the base cluster:

```bash
ansible-playbook -i inventory/cluster/hosts.yaml provisioning.yaml --tags kubernetes
```

This will:
- Deploy K3s on all nodes with proper architecture tainting
- Set up encryption for Kubernetes secrets
- Configure the control plane and worker nodes

### 4. Deploy Core Components

Run the chart provisioning to deploy essential services:

```bash
ansible-playbook -i inventory/cluster/hosts.yaml provisioning.yaml --tags charts
```

This will deploy:
- **Networking**: Cilium for CNI and service mesh
- **Storage**: Longhorn with encryption support
- **Certificate Management**: cert-manager
- **DNS**: CoreDNS
- **External DNS**: Integration with DNS providers
- **Monitoring**: VictoriaMetrics and VictoriaLogs
- **GitOps**: ArgoCD
- **Reboot Management**: Kured for coordinated node reboots

### 5. Verify Deployment

Check the cluster status:

```bash
kubectl get nodes
kubectl get pods -A
```

### 6. Accessing Component UIs

- **ArgoCD**: https://argocd.[your-domain]
- **Grafana**: https://grafana.[your-domain]
- **Longhorn**: https://longhorn.[your-domain]
- **Hubble**: https://hubble.[your-domain]

## Security Features

### Kubernetes Secrets Encryption

The cluster uses AES-CBC encryption for Kubernetes secrets with automatic key rotation.

### Volume Encryption

Longhorn volumes can be encrypted using the `longhorn-encrypted` storage class, which uses LUKS encryption.

## Architecture Support

The cluster automatically detects node architectures and applies appropriate taints to ensure workloads run on compatible nodes.

## Upgrading the Cluster

To upgrade the cluster components:

```bash
ansible-playbook -i inventory/cluster/hosts.yaml upgrade.yaml
```

## Troubleshooting

- Check node status: `kubectl get nodes -o wide`
- View pod status: `kubectl get pods -A`
- Check component logs: `kubectl logs -n [namespace] [pod-name]`
- Validate encryption: `kubectl get secret [secret-name] -o yaml`