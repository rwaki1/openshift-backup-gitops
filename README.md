# OpenShift OADP Backup — GitOps Repository

GitOps-driven disaster recovery for vSphere-hosted OpenShift clusters using OADP, Velero, and Flux CD.

**Stack:** OADP 1.4 · Velero · Flux CD v2 · vSphere CSI · SOPS + Age Encryption

## Repository Structure

```
clusters/
└── production/
    ├── flux-system/               # Created by 'flux bootstrap' — do not edit manually
    └── backup/
        ├── namespace.yaml         # Creates openshift-adp and openshift-etcd-backup
        ├── kustomization.yaml     # Root Flux Kustomization for the backup stack
        ├── oadp/
        │   ├── operatorgroup.yaml
        │   └── subscription.yaml
        ├── velero/
        │   ├── dpa.yaml
        │   ├── bsl.yaml
        │   ├── vsl.yaml
        │   └── schedules/
        │       ├── etcd-schedule.yaml
        │       ├── cluster-full-schedule.yaml
        │       ├── namespaced-apps-schedule.yaml
        │       └── pv-snapshots-schedule.yaml
        ├── etcd/
        │   ├── namespace.yaml
        │   ├── sa.yaml
        │   ├── rbac.yaml
        │   └── cronjob.yaml
        └── secrets/
            └── s3-credentials.yaml   # ⚠️ Encrypt with SOPS before committing!
```

## Quick Start

1. Bootstrap Flux onto your cluster:
   ```bash
   flux bootstrap github \
     --owner=<your-org> \
     --repository=<this-repo> \
     --branch=main \
     --path=clusters/production
   ```

2. Create the SOPS Age key and store it in the cluster:
   ```bash
   age-keygen -o age-private-key.txt
   kubectl create secret generic sops-age \
     --namespace=flux-system \
     --from-file=age.agekey=age-private-key.txt
   ```

3. Update `secrets/s3-credentials.yaml` with your credentials, encrypt with SOPS, then commit.

4. Update `velero/dpa.yaml` with your S3 endpoint and bucket details.

## Backup Schedule Summary

| Schedule          | Frequency  | Retention | Method         |
|-------------------|------------|-----------|----------------|
| etcd-backup       | Every 6h   | 14 days   | CronJob + PVC  |
| cluster-full      | Daily 2 AM | 30 days   | Velero + CSI   |
| namespaced-apps   | Every 4h   | 7 days    | Velero (S3)    |
| pv-snapshots      | Daily 1 AM | 14 days   | CSI Snapshot   |

## Useful Commands

```bash
# Check backup status
velero backup get
velero schedule get

# Restore a namespace
velero restore create ns-restore \
  --from-backup namespaced-apps-<timestamp> \
  --include-namespaces production \
  --wait

# Flux status
flux get all
flux get kustomizations
```
