Velero Backup - Opt-In Annotation:
	- All file system backups (FSB) will be taken in the specified namespace.
	- FSB happens via Kopia.
	- Specific resources that need to be explicitly included in the backup.
	- Velero, by default, uses this approach to discover pod volumes that need to be backed up using FSB. 
	- Every pod containing a volume to be backed up using FSB must be annotated with the volume’s name using the backup.velero.io/backup-volumes annotation.
	-  All resource metadata, except for pods that are annotated for FSB, will be backed up using Velero and Kopia for file system backups.
	- Remaining all metadata will be backed up via velero, FSB only via Kopia
	
PreChecks:  
	1. Login to the Velero Client
	2. kubectl get po -n velero
	3. velero backup-location get
	4. Busybox, Nginx, webserver deployment YAML  files available.
	5. kubectl get ns | grep gcp-in 

Modify Namespace, storageClassName in your YAML files.
    kubectl get sc
    NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    gcp-fast (default)   pd.csi.storage.gke.io      Delete          WaitForFirstConsumer   true                   578d
   
Deploy all 3 apps
    kubectl create namespace gcp-in
    kubectl apply -f nginx.yaml -n gcp-in
    kubectl apply -f busybox.yaml -n gcp-in
    kubectl apply -f webserver.yaml -n gcp-in

    Verify apps
    kubectl get po -n gcp-in
    kubectl get pvc -n gcp-in 

Insert some data in mount points & Verify
    Insert data
        kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "This is Velero Backup & restore Test in Nginx" > /usr/share/nginx/html/index.html'
        kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "This is Velero Backup & restore Test in busybox" > /data/data.txt'
        kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "This is Velero Backup & restore Test in  webserver" > /usr/local/apache2/htdocs/index.html'

    Verify data inside pods
        kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- cat /data/data.txt
        kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html
        kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html

Create Velero backup & Verify
    Backup 
        velero backup create fs-backup-optin-annotate --include-namespaces gcp-in

    Verify backups
        velero backup get
        velero backup describe fs-backup-optin-annotate --details
        velero backup logs fs-backup-optin-annotate
    
    Delete apps
        kubectl delete -f nginx.yaml -n gcp-in
        kubectl delete -f busybox.yaml -n gcp-in
        kubectl delete -f webserver.yaml -n gcp-in

Restore apps from backup & Verify
    Restore
        velero restore create fs-restore-optin-annotate --from-backup fs-backup-optin-annotate

    Verify restore
        velero restore get
        velero restore describe fs-restore-optin-annotate  --details
        velero restore logs fs-restore-optin-annotate

Verify pods inside information, data that we inserted above ideally should show except Nginx
        kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- cat /data/data.txt
        kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html
        kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html

    Note:
        - Nginx inserted data will not show in its mount point because we did not annotated to take backup of data by kopia. Only metadata will take backup by velero


Clean backup & Restore, Should be careful
        velero backup delete fs-backup-optin --confirm
        velero restore delete fs-restore-optin-annotate --confirm
Login to GCP console -> Buckets -> Delete Namespace gcp-in under velero & Kopia

#Other
kubectl get pods -n gcp-in | grep busybox
kubectl get pods -n gcp-in --show-labels | grep busybox