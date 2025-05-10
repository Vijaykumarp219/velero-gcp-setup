# Velero Backup - FSB Opt-Out (Global)

This guide demonstrates a **File System Backup (FSB)** using **Kopia** with **Velero**, using the *opt-out global mode* (i.e., FSB is applied to all PVCs in the namespace).

---

## PreChecks

1. Login to the Velero client node.
2. Check if Velero is running:
    ```bash
    kubectl get po -n velero
    ```
3. Check if Velero has an available backup location:
    ```bash
    velero backup-location get
    ```
4. Ensure you have the following YAML files ready:
    - `nginx.yaml`
    - `busybox.yaml`
    - `webserver.yaml`
5. Confirm the storage class is available:
    ```bash
    kubectl get sc
    ```
    Example output:
    ```
    NAME                PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    gcp-fast (default) pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   578d
    ```

---

## Step 1: Deploy Applications

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

## Step 2: Insert Sample Data into PVCs

```bash
# Nginx
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}')   -- sh -c 'echo "This is Velero Backup & restore Test in Nginx" > /usr/share/nginx/html/index.html'

# Busybox
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}')   -- sh -c 'echo "This is Velero Backup & restore Test in busybox" > /data/data.txt'

# Webserver
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}')   -- sh -c 'echo "This is Velero Backup & restore Test in webserver" > /usr/local/apache2/htdocs/index.html'
```

Verify the data:

```bash
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- cat /data/data.txt
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html
```

---

## Step 3: Create Velero Backup

```bash
velero backup create fs-backup-optout-global --include-namespaces gcp-in
```

Verify backup:

```bash
velero backup get
velero backup describe fs-backup-optout-global --details
velero backup logs fs-backup-optout-global
```

---

## Step 4: Delete Applications

```bash
kubectl delete -f nginx.yaml -n gcp-in
kubectl delete -f busybox.yaml -n gcp-in
kubectl delete -f webserver.yaml -n gcp-in
```

---

## Step 5: Restore from Backup

```bash
velero restore create fs-restore-optout-global --from-backup fs-backup-optout-global
```

Verify restore:

```bash
velero restore get
velero restore describe fs-restore-optout-global --details
velero restore logs fs-restore-optout-global
```

---

## Step 6: Verify Restored Data

```bash
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- cat /data/data.txt
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html
```
---
## Clean Backup & Restore
⚠️ Be careful when deleting backups and restores:
```bash

velero  backup  delete  fs-backup-optout-global  --confirm

velero  restore  delete  fs-restore-optout-global  --confirm

kubectl  delete  ns  gcp-in

```
Also, go to **GCP Console → Buckets → Delete namespace `gcp-in`** under **Velero** and **Kopia** buckets if needed.

---

## Summary

- All PVCs in the namespace were backed up using FSB (via Kopia).
- All metadata (Deployments, Services, PVCs) was backed up using Velero.
- This example demonstrates global opt-out where **all pods with PVCs get FSB unless explicitly opted out.**

