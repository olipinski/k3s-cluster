# Kubernetes Cluster Encryption Guide

This document describes the encryption implementations in the cluster.

## 1. K3s Secret Encryption

The cluster uses K3s built-in encryption capabilities to encrypt all Kubernetes secrets at rest in etcd.

### Key rotation procedure:

1. Run the key rotation playbook:

   ```
   ansible-playbook key-rotation.yaml
   ```

2. Verify the rotation was successful:
   ```
   kubectl get secrets --all-namespaces -o json | grep -i "k8s:enc:aescbc:v1:" | wc -l
   ```
   This should return a number matching your total secrets count.

## 2. Longhorn Volume Encryption

Longhorn volumes can be encrypted using LUKS encryption.

### Using encrypted volumes:

1. Create PVCs using the `longhorn-encrypted` storage class:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: my-encrypted-volume
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: longhorn-encrypted
     resources:
       requests:
         storage: 10Gi
   ```

2. Verify encryption:
   ```
   kubectl get pv <pv-name> -o yaml | grep encrypted
   ```
   Should show `encrypted: "true"`

### Key rotation:

1. Run the Longhorn key rotation playbook:

   ```
   ansible-playbook longhorn-key-rotation.yaml
   ```

2. After rotation, new volumes will use the new key. Existing volumes continue using their original key.

## Security Best Practices

1. Rotate encryption keys regularly (at least every 90 days)
2. Keep offline backups of encryption keys
3. Test the restoration process regularly
