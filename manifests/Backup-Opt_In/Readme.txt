Velero Backup - Opt-In Annotation
    - All file system backups (FSB) will be taken in the specified namespace.
    - FSB happens via Kopia.
    - Specific resources that need to be explicitly included in the backup.
    - Velero, by default, uses this approach to discover pod volumes that need to be backed up using FSB. Every pod containing a volume to be backed up using FSB must be annotated with the volume’s name using the backup.velero.io/backup-volumes annotation.
    - All resource metadata, except for pods that are annotated for FSB, will be backed up using Velero and Kopia for file system backups.

    Understanding backup.velero.io/backup-volumes
      -  The annotation backup.velero.io/backup-volumes is used to explicitly include specific volumes for file system backup. It does not act as an exclusion mechanism.
      -  If you annotate a Pod with this, Velero will only backup the volumes listed in the annotation, and it will use the FSB for those volumes.
      -  If the Pod has other volumes that are not listed in the annotation, Velero will back them up using the default Persistent Volume (PV) snapshotting method (if supported by your cloud provider) or skip them.

#Steps to Install Velero Client
    sudo apt install wget
    wget https://github.com/vmware-tanzu/velero/releases/download/v1.15.2/velero-v1.15.2-linux-amd64.tar.gz
    tar -xvf velero-v1.15.2-linux-amd64.tar.gz
    sudo mv velero-v1.15.2-linux-amd64/velero /usr/local/bin

#Velero Server deployment with nodeagent & Kopia
velero install \
    --provider gcp \
    --plugins velero/velero-plugin-for-gcp:v1.11.0 \
    --use-volume-snapshots=false \
    --bucket velero-practice \
    --secret-file /home/gcp032025/velero-tests/fsb/optin/credentials-velero-practice \
    --use-node-agent \
    --uploader-type kopia \
    --wait

gcp032025@kube-master:~/velero-tests/fsb/optin$ kubectl get po -n velero
NAME                     READY   STATUS    RESTARTS   AGE
node-agent-fr4db         1/1     Running   0          29s
velero-576fddf4f-7jjdk   1/1     Running   0          29s

velero backup-location get
NAME      PROVIDER   BUCKET/PREFIX     PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   gcp        velero-practice   Available   2025-04-01 10:52:06 +0000 UTC   ReadWrite     true

#Note After Velero server deployment in k8s must check pods has reachability to google services. My case dns issue
    - Dns config has to change in velero deployment and nodeagent deamon sets.
    dnsConfig:
       nameservers:
        - 8.8.8.8
        - 8.8.4.4
    dnsPolicy: None

#create Namespace & Install apps
kubectl create namespace gcp-in
kubectl apply -f nginx.yaml -n gcp-in
kubectl apply -f busybox.yaml -n gcp-in
kubectl apply -f webserver.yaml -n gcp-in

#Verify apps
kubectl get po -n gcp-in
kubectl get pvc -n gcp-in

#Insert some data in mount points
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "Hello from nginx" > /usr/share/nginx/html/index.html'
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "Hello from busybox" > /data/data.txt'
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "Hello from webserver" > /usr/local/apache2/htdocs/index.html'

#Verify data inside pods
    #kubectl exec -it -n gcp-in nginx-deployment-5c794987d4-r9gr6 -- bash
    #kubectl exec -it -n gcp-in busybox-pod -- sh
    #kubectl exec -n gcp-in busybox-pod -- ls -l /data

kubectl exec -n gcp-in busybox-pod -- cat /data/data.txt
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html

#Create Velero backup
#velero backup create fs-backup-optout-annotate --include-namespaces gcp-in --include-resources deployments,pods,persistentvolumeclaims
velero backup create fs-backup-optin-annotate --include-namespaces gcp-in

#Verify backups
velero backup get
velero backup describe fs-backup-optin-annotate --details
    #velero describe backup fs-backup-test-2 --details
velero backup logs fs-backup-optin-annotate
velero backup delete <backup-name> --confirm

#Delete apps
kubectl delete -f nginx.yaml -n gcp-in
kubectl delete -f busybox.yaml -n gcp-in
kubectl delete -f webserver.yaml -n gcp-in

#Restore apps from backup
velero restore create fs-restore-optin-annotate --from-backup fs-backup-optin-annotate

#Verify restore
velero restore get
velero restore describe fs-backup-optin-annotate --details
velero restore logs fs-backup-optin-annotate


velero restore delete fs-backup-optin-annotate --confirm

#Verify pods inside information data should show
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html
kubectl exec -n gcp-in $(kubectl get pods -n gcp-in -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html

Note:
    - Busy box will not show any information in its mount point because we annotated not to take backup of data. Only metadata will take

#Other
kubectl get pods -n gcp-in | grep busybox
kubectl get pods -n gcp-in --show-labels | grep busybox