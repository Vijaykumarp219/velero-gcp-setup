# Velero Backup - Opt-out via Annotation

## Overview
- All file system backups (FSB) will be taken in the specified namespace.
- FSB is managed by Kopia.
- Use `backup.velero.io/backup-volumes-excludes` annotation on the Pod to exclude specific volumes from backup.
- All resource metadata will be backed up, but volumes with exclusion annotations will skip file system backup.

---

## PreChecks
1. Login to the Velero Client
2. Check Velero pods:  
   ```bash
   kubectl get po -n velero
   ```
3. Validate backup locations:  
   ```bash
   velero backup-location get
   ```
4. Ensure Busybox, Nginx, Webserver deployment YAMLs are ready.
5. Check available namespaces:  
   ```bash
   kubectl get ns | grep gcp-in
   ```

---

## Modify StorageClass & Namespace

Verify storage class:  
```bash
kubectl get sc
```

Sample output:
```
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gcp-fast     pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   578d
```

Update `namespace` and `storageClassName` in the YAML files accordingly.

---

## Deploy Applications

```bash
kubectl create namespace gcp-in

kubectl apply -f nginx.yaml -n gcp-in
kubectl apply -f busybox.yaml -n gcp-in
kubectl apply -f webserver.yaml -n gcp-in
```

Verify deployments:
```bash
kubectl get po -n gcp-in
kubectl get pvc -n gcp-in
```

---

## Insert Data into Mount Points

```bash
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "This is Velero Backup & restore Test in Nginx" > /usr/share/nginx/html/index.html'

kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "This is Velero Backup & restore Test in busybox" > /data/data.txt'

kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "This is Velero Backup & restore Test in  webserver" > /usr/local/apache2/htdocs/index.html'
```

### Verify Inserted Data
```bash
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- cat /data/data.txt

kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html

kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html
```

---

## Create Backup

```bash
velero backup create fs-backup-optout-annotate --include-namespaces gcp-in
```

### Verify Backup

```bash
velero backup get

velero backup describe fs-backup-optout-annotate --details

velero backup logs fs-backup-optout-annotate
```

---

## Delete Applications

```bash
kubectl delete -f nginx.yaml -n gcp-in
kubectl delete -f busybox.yaml -n gcp-in
kubectl delete -f webserver.yaml -n gcp-in
```

---

## Restore from Backup

```bash
velero restore create fs-restore-optout-annotate --from-backup fs-backup-optout-annotate
```

### Verify Restore

```bash
velero restore get

velero restore describe fs-restore-optout-annotate --details

velero restore logs fs-restore-optout-annotate
```

---

## Verify Restored Data

```bash
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- cat /data/data.txt

kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html

kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html
```

---

## Notes

- The `busybox` pod will not have its volume data restored as it was annotated for FSB exclusion. Only its metadata will be restored.
- All other resources will restore with both metadata and volume data.

---
## Clean Backup & Restore
⚠️ Be careful when deleting backups and restores:
```bash
velero backup delete fs-backup-optout-annotate --confirm

velero restore delete fs-restore-optout-annotate --confirm

kubectl delete ns gcp-in

```
Also, go to **GCP Console → Buckets → Delete namespace `gcp-in`** under **Velero** and **Kopia** buckets if needed.