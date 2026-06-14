# Restore Guide

## Prerequisites

- `oc` logged into the target cluster
- Velero and OADP running (`oc get pods -n openshift-adp`)
- BackupStorageLocation Available (`oc get bsl -n openshift-adp`)

---

## Option 1 — Restore from Existing Backup in Cluster

The backup already exists as a Backup CR in the cluster.

**List available backups:**
```bash
oc get backup -n openshift-adp
```

**Restore to the same namespace:**
```bash
oc apply -f - <<EOF
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: my-restore
  namespace: openshift-adp
spec:
  backupName: <backup-name>
  restorePVs: true
EOF
```

**Restore to a different namespace (namespace mapping):**
```bash
oc apply -f - <<EOF
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: my-restore
  namespace: openshift-adp
spec:
  backupName: <backup-name>
  namespaceMapping:
    original-namespace: restored-namespace
  restorePVs: true
EOF
```

**Monitor progress:**
```bash
oc get restore -n openshift-adp -w
oc describe restore my-restore -n openshift-adp
```

---

## Option 2 — Restore from Backup Files on Local Disk

Use this when the cluster was rebuilt and the Backup CR no longer exists, but you have the backup files saved locally.

### Step 1 — Upload files to MinIO

**Port-forward MinIO S3 API:**
```bash
# Keep this running in a separate terminal
oc port-forward svc/minio 9000:9000 -n openshift-adp
```

**Download MinIO client:**
```powershell
# Windows
Invoke-WebRequest -Uri "https://dl.min.io/client/mc/release/windows-amd64/mc.exe" -OutFile "mc.exe"
```
```bash
# Linux / Mac
curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
```

**Configure mc and upload:**
```bash
./mc alias set local http://localhost:9000 minioadmin minioadmin

./mc cp --recursive /path/to/backup-folder/ \
  local/openshift-backups/ocp-production/backups/<backup-name>/
```

> **Important:** The path must match the BSL prefix. Check with:
> `oc get bsl -n openshift-adp -o jsonpath='{.items[0].spec.objectStorage}'`

### Step 2 — Fix backup expiration (if backup is older than TTL)

If the backup was created more than 30 days ago, Velero will auto-delete it immediately on sync.
Edit `velero-backup.json` before uploading:

```powershell
# PowerShell — update expiration to 10 years from now
$file = "/path/to/velero-backup.json"
$json = Get-Content $file -Raw | ConvertFrom-Json
$json.status.expiration    = "2036-01-01T00:00:00Z"
$json.spec.ttl             = "87600h0m0s"
$json.spec.storageLocation = "dpa-1"          # must match existing BSL name
$json.metadata.labels."velero.io/storage-location" = "dpa-1"
$json.status.phase         = "Completed"
$json | ConvertTo-Json -Depth 20 | Set-Content $file -Encoding UTF8
```

### Step 3 — Wait for Velero to sync

Velero syncs from BSL every ~1 minute. Check:
```bash
oc get backup -n openshift-adp
```
The backup name should appear. If it doesn't appear after 2 minutes, restart Velero:
```bash
oc rollout restart deployment/velero -n openshift-adp
```

### Step 4 — Apply the restore immediately

Once the backup appears, apply the restore before the TTL causes deletion:
```bash
oc apply -f restore.yaml
```

---

## Verify Restore

```bash
# Check pods in restored namespace
oc get pods -n <restored-namespace>

# Check all resources
oc get all -n <restored-namespace>

# Get credentials from restored secrets
oc get secret <secret-name> -n <restored-namespace> -o jsonpath='{.data}' 
```

---

## Common Restore Issues

| Error | Cause | Fix |
|-------|-------|-----|
| `Backup not found` | Backup CR missing from cluster | Upload files to MinIO, wait for sync |
| `FailedValidation: can't find backup` | Backup files in wrong MinIO path | Check BSL prefix with `oc get bsl -o yaml` |
| Backup appears then disappears | TTL expired | Edit `velero-backup.json` expiration before uploading |
| `ImagePullBackOff` after restore | Container image not in backup | Rebuild and push image to cluster registry |
| `PartiallyFailed` with warnings | Normal for cluster-scoped resources | Check if workloads are actually running |
| `Access denied` to database | Secret key name mismatch | Check `oc get secret <name> -o jsonpath='{.data}'` |

---

## Connect to Restored MySQL Database

```bash
# Get actual password from pod environment (most reliable)
oc exec -n <namespace> deployment/mysql -- env | grep MYSQL_ROOT_PASSWORD

# Connect
oc exec -it -n <namespace> deployment/mysql -- mysql -u root -p<password>
```
