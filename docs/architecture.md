# Architecture

## Component Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  Developer Workstation                                               │
│  git push → github.com/rwaki1/openshift-backup-gitops               │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      ▼ watches every 1 minute
┌─────────────────────────────────────────────────────────────────────┐
│  namespace: flux-system                                             │
│                                                                     │
│  source-controller       → fetches Git repo                        │
│  kustomize-controller    → applies manifests to cluster            │
│  helm-controller         → manages Helm releases                   │
│  notification-controller → sends alerts                            │
└─────────────────────┬───────────────────────────────────────────────┘
                      │ deploys
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│  OpenShift Cluster                                                  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  namespace: openshift-adp                                   │   │
│  │                                                             │   │
│  │  OADP Operator (v1.5.5)   manages lifecycle                 │   │
│  │       ↓                                                     │   │
│  │  Velero                   backup / restore engine           │   │
│  │       ↓ stores backups                                      │   │
│  │  MinIO                    S3-compatible object storage      │   │
│  │       ↑                                                     │   │
│  │  BackupStorageLocation    connects Velero → MinIO           │   │
│  │  BackupSchedule           triggers daily at 02:00           │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## Flux Kustomization Dependency Chain

```
GitRepository: flux-system
(github.com/rwaki1/openshift-backup-gitops)
        │
        ├── Kustomization: backup-operator
        │     path: clusters/production/backup-platform/oadp
        │     wait: true, timeout: 10m
        │     → installs OADP operator via OLM Subscription
        │     → waits for CRDs to be registered
        │
        └── Kustomization: backup-platform
              path: clusters/crc/backup-platform-config  (CRC)
                 or clusters/production/...              (production)
              dependsOn: backup-operator
              → deploys namespace
              → deploys MinIO (deployment, service, PVC, route)
              → deploys cloud-credentials secret
              → applies DataProtectionApplication
              → applies BackupStorageLocation
```

## Why Two Kustomizations?

The OADP operator must be fully installed before any `DataProtectionApplication` or `BackupStorageLocation` resources can be applied — those CRDs only exist after the operator installs.

Using `dependsOn` with `wait: true` on `backup-operator` solves this. Flux will not start `backup-platform` until the operator's CRDs are available.

## CRC vs Production Differences

### CRC Overlay (`clusters/crc/backup-platform-config/`)

The CRC overlay references the production base and applies two patches:

**patch-dpa-no-csi.yaml** — removes the `csi` plugin from DPA:
```yaml
# CRC does not support VolumeSnapshotContent / CSI snapshots
spec:
  configuration:
    velero:
      defaultPlugins:
        - openshift
        - aws
        # csi removed
```

**scc-bindings.yaml** — grants `anyuid` SCC to Flux controllers:
```yaml
# CRC restricts fsGroup and seccompProfile — anyuid SCC allows Flux pods to start
# Also removes fsGroup and seccompProfile via JSON patches in flux-system/kustomization.yaml
```

### Production (vSphere)

- CSI plugin enabled — vSphere supports `VolumeSnapshotContent`
- No SCC patches needed — Flux already running with correct permissions
- Use vSphere storage class for MinIO PVC

## Data Flow — Backup

```
1. BackupSchedule triggers at 02:00 (or manual: oc apply Backup CR)
2. Velero discovers all resources in target namespace
3. Velero serializes resources → compressed tar.gz archive
4. Archive uploaded to MinIO → bucket: openshift-backups / prefix: ocp-production
5. Backup CR status updated → Phase: Completed
```

## Data Flow — Restore

```
1. Create Restore CR (references a Backup by name)
2. Velero fetches archive from MinIO
3. Velero deserializes resources → applies to target namespace
4. Namespace mapping optionally restores to a different namespace
5. Restore CR status updated → Phase: Completed
```

## Storage Layout in MinIO

```
openshift-backups/              ← bucket
└── ocp-production/             ← prefix (set in BSL)
    └── backups/
        └── <backup-name>/
            ├── <name>.tar.gz                        ← all Kubernetes resources
            ├── velero-backup.json                   ← Backup CR metadata
            ├── <name>-logs.gz
            ├── <name>-resource-list.json.gz
            ├── <name>-podvolumebackups.json.gz
            └── <name>-volumesnapshots.json.gz
```
