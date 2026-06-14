# Production Deployment — vSphere / OpenShift 4.x

## Prerequisites

- OpenShift 4.14+ cluster on vSphere
- Flux CD already running (`oc get pods -n flux-system`)
- GitRepository pointing to this repo (`oc get gitrepository -n flux-system`)
- `oc` logged in as cluster-admin

---

## Manual Steps (one-time)

### 1. Verify Flux is watching this repo

```bash
oc get gitrepository -n flux-system
```

Expected output:
```
NAME          URL                                                   READY
flux-system   https://github.com/rwaki1/openshift-backup-gitops   True
```

### 2. Get the vSphere storage class name

```bash
oc get storageclass
```

Note the storage class marked `(default)` or use `thin-csi` if available.

### 3. Update MinIO storage class

Edit `clusters/production/backup-platform/minio/pvc.yaml`:
```yaml
spec:
  storageClassName: <your-vsphere-storage-class>  # e.g. thin-csi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### 4. Create the credentials secret

This secret cannot be stored in Git. Create it manually on the cluster:

```bash
oc create namespace openshift-adp 2>/dev/null || true

oc create secret generic cloud-credentials \
  -n openshift-adp \
  --from-literal=cloud="[default]
aws_access_key_id=minioadmin
aws_secret_access_key=minioadmin"
```

> For production, replace `minioadmin` with strong credentials and store them in a password manager.

### 5. Create the Flux Kustomization files

Create `clusters/production/backup-operator.yaml`:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: backup-operator
  namespace: flux-system
spec:
  interval: 5m
  path: ./clusters/production/backup-platform/oadp
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  wait: true
  timeout: 10m
```

Create `clusters/production/backup-platform.yaml`:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: backup-platform
  namespace: flux-system
spec:
  interval: 10m
  path: ./clusters/production/backup-platform
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  dependsOn:
    - name: backup-operator
```

### 6. Commit and push

```bash
git add clusters/production/backup-operator.yaml \
        clusters/production/backup-platform.yaml \
        clusters/production/backup-platform/minio/pvc.yaml
git commit -m "add backup platform for production vSphere"
git push
```

---

## Automated by Flux (after git push)

| Step | Component | Time |
|------|-----------|------|
| Detects commit | source-controller | ~1 min |
| Creates openshift-adp namespace | kustomize-controller | instant |
| Installs OADP operator via OLM | backup-operator Kustomization | 3–5 min |
| Waits for CRDs to be ready | Flux `wait: true` | automatic |
| Deploys MinIO + PVC + Service + Route | backup-platform Kustomization | 1–2 min |
| Applies DataProtectionApplication | OADP controller | 1 min |
| Velero pods start | OADP operator | 1–2 min |
| BackupStorageLocation becomes Available | Velero | 30 sec |

**Total time from git push to fully working: ~10–15 minutes**

---

## Verify Production Deployment

```bash
# All 3 Flux Kustomizations ready
oc get kustomization -n flux-system

# All pods running in openshift-adp
oc get pods -n openshift-adp

# OADP operator installed successfully
oc get csv -n openshift-adp | grep oadp

# Storage location available
oc get backupstoragelocation -n openshift-adp
```

Expected pods:
```
oadp-operator-controller-manager-xxx   Running
velero-xxx                             Running
minio-xxx                              Running
```

---

## Production vs CRC Differences

On production vSphere you do **not** need:
- `scc-bindings.yaml` — Flux already has correct permissions
- `patch-dpa-no-csi.yaml` — vSphere supports CSI volume snapshots
- SCC patches in `flux-system/kustomization.yaml`

The DPA on production uses all three plugins:
```yaml
configuration:
  velero:
    defaultPlugins:
      - openshift
      - aws
      - csi    # enabled on vSphere
```

---

## Enable CSI Volume Snapshots (vSphere)

To back up PersistentVolumes using vSphere snapshots, create a VolumeSnapshotLocation:

```yaml
apiVersion: velero.io/v1
kind: VolumeSnapshotLocation
metadata:
  name: default
  namespace: openshift-adp
spec:
  provider: aws
  config:
    region: us-east-1
```

And add a VolumeSnapshotClass:
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: velero-csi
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: csi.vsphere.volume.vmware.com
deletionPolicy: Retain
```
