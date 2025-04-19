# K3s Cluster Deployment with Ansible

This repository contains Ansible playbooks and roles for deploying a production-ready Kubernetes cluster based on K3s, capable of scaling from 2 to 200+ nodes.

## Features

- **High Availability**: K3s cluster with multiple server nodes and HAProxy/Keepalived for API server load balancing
- **Networking**: Cilium CNI with Gateway API support, L2 announcements, and Hubble UI for observability
- **Storage**: Longhorn distributed storage with volume encryption and backup capabilities
- **Certificate Management**: cert-manager for TLS certificate automation
- **External DNS**: Automated DNS management with Cloudflare integration
- **GitOps**: ArgoCD for continuous delivery and application management
- **Monitoring**: VictoriaMetrics and Grafana for metrics collection and visualization
- **Logging**: VictoriaLogs for centralized logging
- **Security**: Secret encryption at rest, volume encryption, and RBAC

## Prerequisites

- Ubuntu 24.04 LTS (Minimum 2 nodes, 4+ recommended for HA)
- Minimum system requirements per node:
  - 2 CPU cores
  - 2GB RAM (4GB+ recommended)
  - 20GB disk space (50GB+ recommended)
- Ansible 2.15+
- SSH access to all nodes with sudo privileges

## Deployment Guide

### 1. Prepare Inventory

Edit `inventory/cluster/hosts.yaml` to define your server and agent nodes:

```yaml
server:
  hosts:
    apollo: # Control plane node 1
    boreas: # Control plane node 2 (for HA)
    cerus: # Control plane node 3 (for HA)

agent:
  hosts:
    chaos: # Worker node 1
    crios: # Worker node 2
    # Add more worker nodes as needed
```

For non-HA clusters, you need at least 1 server node. For HA clusters, you need at least 3 server nodes.

### 2. Configure Variables

Review and update the following files as needed:

- `inventory/cluster/group_vars/all.yaml` - Global credentials and settings
- `roles/*/defaults/main.yaml` - Component-specific settings

Important configurations to check:

- `externaldns_vars.cloudflare.host.domain` - Set your domain for external access
- `global_map.credentials` - Update all credentials

### 3. Run Validation

```bash
ansible-playbook validation.yaml --tags validation
```

### 4. Deploy the Cluster

```bash
ansible-playbook provisioning.yaml
```

To deploy only specific components:

```bash
ansible-playbook provisioning.yaml --tags kubernetes,charts
```

Available tags:

- `cluster` - Basic OS configuration
- `kubernetes` - K3s and Helm
- `charts` - All applications (ArgoCD, Cilium, etc.)

### 5. Verify Deployment

```bash
# From any server node
kubectl get nodes
kubectl get pods -A
```

## Component Access

### ArgoCD

- URL: https://argocd.your-domain.com
- Default credentials:
  - Username: admin (or configured user)
  - Password: From `global_map.credentials.argocd.server.admin.password`

Access via CLI:

```bash
argocd login argocd.your-domain.com --username admin
```

### Grafana

- URL: https://grafana.your-domain.com
- Default credentials:
  - Username: admin
  - Password: From `global_map.credentials.victoriametrics.grafana.user.password`

### Longhorn

- URL: https://longhorn.your-domain.com

### Hubble UI

- URL: https://hubble.your-domain.com

## Deployment Examples

### Example 1: Deploy a basic web application

1. Create an application manifest (save as `web-app.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: kube-system
spec:
  project: default
  source:
    repoURL: https://github.com/username/web-app.git
    targetRevision: main
    path: kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: web-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

2. Apply the manifest:

```bash
kubectl apply -f web-app.yaml
```

### Example 2: Deploy a stateful application with persistent storage

1. Create a PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-data
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

2. Create a deployment using this PVC:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: my-app
spec:
  serviceName: database
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
        - name: database
          image: postgres:14
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: database-credentials
                  key: password
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: database-data
```

## Maintenance Operations

### Scaling the Cluster

To add new nodes:

1. Add the nodes to your inventory
2. Run the deployment:

```bash
ansible-playbook provisioning.yaml
```

### Upgrading Components

To upgrade the entire cluster:

```bash
ansible-playbook upgrade.yaml
```

To upgrade specific components:

```bash
ansible-playbook upgrade.yaml --tags k3s,cilium
```

### Encryption Key Rotation

To rotate K3s encryption keys:

```bash
ansible-playbook key-rotation.yaml
```

Follow the prompts to confirm the rotation.

### Resetting the Cluster

To completely reset the cluster:

```bash
ansible-playbook reset.yaml
```

## Troubleshooting

### Common Issues

1. **Node Not Joining Cluster**

   - Check firewall settings: `sudo ufw status`
   - Verify network connectivity between nodes

2. **Unable to Access Services**

   - Check DNS configuration
   - Verify Ingress/Gateway resources: `kubectl get gateway,httproute -A`

3. **PVC Stuck in Pending State**
   - Check Longhorn status: `kubectl -n kube-system get pods | grep longhorn`
   - Verify node has enough resources

### Logs to Check

- K3s logs: `sudo journalctl -u k3s`
- Cilium logs: `kubectl -n kube-system logs -l app.kubernetes.io/name=cilium`
- ArgoCD logs: `kubectl -n kube-system logs -l app.kubernetes.io/name=argocd-server`

## Security Considerations

- All Kubernetes secrets are encrypted at rest
- Longhorn volumes can be encrypted for sensitive data
- TLS certificates are automatically managed
- Network policies should be implemented for additional security

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the BSD 3-Clause License - see the LICENSE file for details.
