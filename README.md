# OpenShift GitOps Backup Platform

GitOps-driven backup and disaster recovery for OpenShift clusters using **Flux CD**, **OADP**, **Velero**, and **MinIO**.

**Stack:** Flux CD v2 · OADP v1.5.5 · Velero · MinIO (S3-compatible) · Kustomize

---

## How It Works

Flux CD watches this GitHub repository every minute. Any change committed to Git is automatically applied to the cluster — no manual `oc apply` needed. The backup platform is entirely self-healing: if any component is deleted or modified, Flux restores it within 1 minute.

```
Developer → git push → GitHub → Flux CD → OpenShift Cluster
                                    ↓
                          OADP Operator (installs Velero)
                          Velero (runs backups)
                          MinIO (stores backup archives)
```

---

## Repository Structure

```
clusters/
├── crc/                              # CRC (OpenShift Local) — development environment
│   ├── flux-system/
│   │   ├── gotk-components.yaml      # Flux controllers (source, kustomize, helm, notification)
│   │   ├── gotk-sync.yaml            # GitRepository + root Kustomization
│   │   ├── kustomization.yaml        # Patches for CRC SCC compatibility
│   │   └── scc-bindings.yaml         # anyuid SCC grants for Flux controllers
│   ├── backup-platform-config/
│   │   ├── kustomization.yaml        # CRC overlay — references production base, applies patches
│   │   └── patch-dpa-no-csi.yaml     # Disables CSI plugin (CRC has no VolumeSnapshot support)
│   ├── backup-operator.yaml          # Flux Kustomization: installs OADP operator
│   ├── backup-platform.yaml          # Flux Kustomization: deploys MinIO + Velero (depends on operator)
│   └── kustomization.yaml
│
└── production/                       # Production base — shared by all environments
    ├── flux-system/                  # Created by 'flux bootstrap' — do not edit
    ├── backup-platform/
    │   ├── namespace.yaml            # Creates openshift-adp namespace
    │   ├── kustomization.yaml
    │   ├── oadp/
    │   │   ├── operatorgroup.yaml    # OLM OperatorGroup
    │   │   ├── subscription.yaml     # Installs redhat-oadp-operator from OperatorHub
    │   │   └── dpa.yaml              # DataProtectionApplication (Velero config)
    │   ├── minio/
    │   │   ├── deployment.yaml       # MinIO server (quay.io/minio/minio)
    │   │   ├── service.yaml          # ClusterIP service (ports 9000 API, 9001 console)
    │   │   ├── pvc.yaml              # 10Gi PersistentVolumeClaim for backup storage
    │   │   ├── route.yaml            # OpenShift Route for console access
    │   │   └── bucket-job.yaml       # Job: creates openshift-backups bucket on startup
    │   ├── velero/
    │   │   ├── bsl.yaml              # BackupStorageLocation → MinIO S3 endpoint
    │   │   └── kustomization.yaml
    │   └── secrets/
    │       ├── cloud-credentials.yaml # MinIO access credentials ⚠️ do not commit plaintext
    │       └── kustomization.yaml
    └── kustomization.yaml
```

---

## Flux Kustomization Ordering

Two Flux Kustomizations are used with a `dependsOn` relationship to solve the OADP operator chicken-and-egg problem:

```
backup-operator  →  installs OADP operator (waits until CRDs are ready)
      ↓
backup-platform  →  deploys MinIO, Velero, DPA, BSL (depends on backup-operator)
```

This ensures the `DataProtectionApplication` CRD exists before Velero resources are applied.

---

## Environment Differences

| Feature | CRC (Development) | Production (vSphere) |
|---------|------------------|----------------------|
| CSI snapshots | Disabled (not supported) | Enabled |
| SCC patches for Flux | Required (anyuid) | Not needed |
| Storage class | default | vSphere thin-csi |
| Flux bootstrap | Manual (gotk-components.yaml) | `flux bootstrap github` |

---

## Quick Start — CRC (OpenShift Local)

> Prerequisites: CRC running, `oc` logged in as kubeadmin

**1. Grant Flux SCC permissions (one-time manual step):**
```bash
oc adm policy add-scc-to-user anyuid \
  system:serviceaccount:flux-system:source-controller \
  system:serviceaccount:flux-system:notification-controller \
  system:serviceaccount:flux-system:kustomize-controller \
  system:serviceaccount:flux-system:helm-controller
```

**2. Apply Flux bootstrap:**
```bash
oc apply -k clusters/crc/flux-system
```

**3. Create the credentials secret (manual — never commit plaintext):**
```bash
oc create secret generic cloud-credentials \
  -n openshift-adp \
  --from-literal=cloud="[default]
aws_access_key_id=minioadmin
aws_secret_access_key=minioadmin"
```

**4. Push the two Flux Kustomization files — Flux handles the rest:**
```bash
git add clusters/crc/backup-operator.yaml clusters/crc/backup-platform.yaml
git commit -m "add backup platform kustomizations"
git push
```

Flux installs everything automatically within 10–15 minutes.

---

## Quick Start — Production (vSphere / OpenShift 4.x)

> Prerequisites: Flux CD already running on cluster, GitRepository pointing to this repo

**1. Create the credentials secret:**
```bash
oc create secret generic cloud-credentials \
  -n openshift-adp \
  --from-literal=cloud="[default]
aws_access_key_id=<minio-user>
aws_secret_access_key=<minio-password>"
```

**2. Update storage class** in `clusters/production/backup-platform/minio/pvc.yaml` to match your vSphere storage class.

**3. Commit and push Flux Kustomization files:**
```bash
# Create clusters/production/backup-operator.yaml and backup-platform.yaml
# (same pattern as clusters/crc/ but without CRC-specific patches)
git push
```

Flux detects the commit within 1 minute and deploys everything automatically.

---

## Verify the Platform

```bash
# Flux sync status
oc get kustomization -n flux-system

# All components running
oc get pods -n openshift-adp

# OADP operator installed
oc get csv -n openshift-adp

# Storage location available
oc get backupstoragelocation -n openshift-adp

# Run a test backup
oc apply -f - <<EOF
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: test-backup
  namespace: openshift-adp
spec:
  includedNamespaces:
    - openshift-adp
  storageLocation: default
  ttl: 24h0m0s
EOF

oc get backup -n openshift-adp -w
```

---

## Restore from Backup

See [docs/restore.md](docs/restore.md) for the full restore guide including restoring from local files.

---

## Useful Commands

```bash
# Check all Flux resources
oc get kustomization,gitrepository -n flux-system

# Force Flux to reconcile immediately
oc annotate kustomization backup-platform \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -n flux-system --overwrite

# Watch backup progress
oc get backup -n openshift-adp -w

# Watch restore progress
oc get restore -n openshift-adp -w

# Access MinIO console (port-forward)
oc port-forward svc/minio 9001:9001 -n openshift-adp
# then open http://localhost:9001 (minioadmin / minioadmin)
```

---

## Architecture Diagram

See [docs/architecture.md](docs/architecture.md) for the full component diagram.
