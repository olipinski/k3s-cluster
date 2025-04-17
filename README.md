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

## ArgoCD Usage

ArgoCD is deployed at https://argocd.[your-domain] with the following credentials:

- Username: admin
- Password: Set in your encrypted global variables

### Adding Applications

1. **Using the CLI**:

   ```bash
   # Login to ArgoCD
   argocd login argocd.[your-domain] --username admin --password <your-password> --insecure

   # Add a repository
   argocd repo add https://github.com/your-org/your-repo --name my-repo

   # Create an application
   argocd app create my-app --repo my-repo --path kubernetes/ --dest-server https://kubernetes.default.svc --dest-namespace default

   ```

2. Using the Web UI:

- Navigate to Settings → Repositories → Connect Repo
- Go to Applications → New Application
- Fill in the application details, repository URL, and path
- Set the destination cluster and namespace
- Click "Create"

### Syncing Applications

```bash

# Sync an application
argocd app sync my-app

# Set up auto-sync
argocd app set my-app --sync-policy automated

```

## Security Features

### Encryption Setup

### Kubernetes Secrets Encryption

The cluster uses AES-CBC encryption for Kubernetes secrets with automatic key rotation:

1. **Encryption Setup**:

   - Located in `k3s_vars.encryption.enabled: true` in the global vars
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

### Longhorn Volume Encryption:

- Use the longhorn-encrypted StorageClass for encrypted volumes
- Rotation managed by the scheduled job in the longhorn namespace

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
