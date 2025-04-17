# K3s Cluster Deployment Guide

A comprehensive guide for deploying a secure, heterogeneous Kubernetes cluster using K3s and Ansible.

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Cluster Architecture](#cluster-architecture)
- [Deployment Steps](#deployment-steps)
  - [1. Configure Inventory](#1-configure-inventory)
  - [2. Configure Global Settings](#2-configure-global-settings)
  - [3. Deploy the Cluster Infrastructure](#3-deploy-the-cluster-infrastructure)
  - [4. Deploy Core Components](#4-deploy-core-components)
  - [5. Verify Deployment](#5-verify-deployment)
- [Accessing Component UIs](#accessing-component-uis)
- [Component Guides](#component-guides)
  - [Kubernetes Dashboard](#kubernetes-dashboard)
  - [ArgoCD](#argocd)
- [Security Features](#security-features)
  - [Kubernetes Secrets Encryption](#kubernetes-secrets-encryption)
  - [Longhorn Volume Encryption](#longhorn-volume-encryption)
- [Architecture Support](#architecture-support)
- [Cluster Maintenance](#cluster-maintenance)
  - [Upgrading the Cluster](#upgrading-the-cluster)
  - [Troubleshooting](#troubleshooting)
- [Resources](#resources)

## Introduction

K3s is a lightweight, certified Kubernetes distribution designed for resource-constrained environments and edge computing. This guide walks you through setting up a production-ready K3s cluster with essential components for networking, storage, monitoring, and GitOps.

## Prerequisites

- Servers provisioned with MAAS (Metal as a Service)
- SSH access to all nodes
- Ansible installed on your control node
- `kubectl` installed for cluster management

## Cluster Architecture

- **Server nodes**: Control plane nodes running etcd and Kubernetes control components
- **Agent nodes**: Worker nodes that run application workloads
- **Mixed architecture support**: Both amd64 and arm64 architectures with appropriate node tainting

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
- **Monitoring**: VictoriaMetrics, VictoriaLogs and Kubernetes Dashboard
- **GitOps**: ArgoCD
- **Reboot Management**: Kured for coordinated node reboots

### 5. Verify Deployment

Check the cluster status:

```bash
kubectl get nodes
kubectl get pods -A
```

## Accessing Component UIs

All web interfaces are accessible via HTTPS with the following URLs:

- **ArgoCD**: https://argocd.[your-domain]
- **Grafana**: https://grafana.[your-domain]
- **Longhorn**: https://longhorn.[your-domain]
- **Hubble**: https://hubble.[your-domain]
- **Kubernetes Dashboard**: https://dashboard.[your-domain]

## Component Guides

### Kubernetes Dashboard

The Kubernetes Dashboard provides a web-based UI for managing your cluster.

#### Accessing the Dashboard

1. Deploy the cluster with the `charts` tag:
   ```bash
   ansible-playbook -i inventory/cluster/hosts.yaml provisioning.yaml --tags charts
   ```

2. Get the access token:
   ```bash
   kubectl -n kube-system get secret admin-user-token -o jsonpath='{.data.token}' | base64 -d
   ```

3. Navigate to https://dashboard.[your-domain] and enter the token when prompted.

#### Dashboard Features

- View cluster resources and their status
- Monitor workload health
- View logs for pods
- Manage resources through the UI

### ArgoCD

ArgoCD provides GitOps-based continuous delivery for your applications.

#### Authentication

ArgoCD is deployed at https://argocd.[your-domain] with the following credentials:
- Username: admin
- Password: Set in your encrypted global variables

#### Adding Applications

**Using the CLI**:

```bash
# Login to ArgoCD
argocd login argocd.[your-domain] --username admin --password <your-password> --insecure

# Add a repository
argocd repo add https://github.com/your-org/your-repo --name my-repo

# Create an application
argocd app create my-app \
  --repo my-repo \
  --path kubernetes/ \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

**Using the Web UI**:
1. Navigate to Settings → Repositories → Connect Repo
2. Go to Applications → New Application
3. Fill in the application details, repository URL, and path
4. Set the destination cluster and namespace
5. Click "Create"

#### Syncing Applications

```bash
# Sync an application
argocd app sync my-app

# Set up auto-sync
argocd app set my-app --sync-policy automated
```

## Security Features

### Kubernetes Secrets Encryption

The cluster uses AES-CBC encryption for Kubernetes secrets with automatic key rotation:

1. **Encryption Setup**:
   - Enabled with `k3s_vars.encryption.enabled: true` in the global vars
   - Managed by the ansible cron job that calls `/usr/local/bin/k3s-key-rotation.sh`
   - Rotates keys every 30 days automatically

2. **Key Management**:
   - Primary encryption key stored in Ansible Vault: `global_map.k3s.encryption.key`
   - To update the key: `ansible-vault edit inventory/cluster/group_vars/all.yaml`
   - After updating, run `ansible-playbook upgrade.yaml --tags k3s` to apply

3. **Verifying Encryption**:
   ```bash
   # Check if a secret is encrypted
   kubectl get secret mysecret -o yaml | grep -i aescbc
   ```

### Longhorn Volume Encryption

- Use the `longhorn-encrypted` StorageClass for encrypted volumes
- Key rotation is managed by the scheduled job in the longhorn namespace

## Architecture Support

The cluster automatically detects node architectures (amd64, arm64) and applies appropriate taints to ensure workloads run on compatible nodes.

## Cluster Maintenance

### Upgrading the Cluster

To upgrade all cluster components:

```bash
ansible-playbook -i inventory/cluster/hosts.yaml upgrade.yaml
```

For specific component upgrades, use tags:

```bash
# Upgrade only K3s
ansible-playbook -i inventory/cluster/hosts.yaml upgrade.yaml --tags k3s

# Upgrade only charts
ansible-playbook -i inventory/cluster/hosts.yaml upgrade.yaml --tags charts
```

### Troubleshooting

Common commands for diagnosing cluster issues:

```bash
# Check node status
kubectl get nodes -o wide

# View pod status across all namespaces
kubectl get pods -A

# Check component logs
kubectl logs -n [namespace] [pod-name]

# Validate encryption configuration
kubectl get secret [secret-name] -o yaml
```

## Resources

- [K3s Official Documentation](https://rancher.com/docs/k3s/latest/en/)
- [Ansible Documentation](https://docs.ansible.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Longhorn Documentation](https://longhorn.io/docs/)
- [ArgoCD User Guide](https://argo-cd.readthedocs.io/en/stable/)
- [Cilium Documentation](https://docs.cilium.io/)