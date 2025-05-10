Velero Backup - Opt-out: Global
    - All file system backups (FSB) will be taken in the specified namespace.
    - FSB happens via Kopia.
    - All resource metadata backup will happen as a Velero backup.

#Velero Server deployment with nodeagent & Kopia
velero install \
    --provider gcp \
    --plugins velero/velero-plugin-for-gcp:v1.11.0 \
    --use-volume-snapshots=false \
    --default-volumes-to-fs-backup \
    --bucket $BUCKET \
    --secret-file /home/velero/credentials-velero \
    --use-node-agent \
    --uploader-type kopia
    --wait

#Note After Velero server deployment in k8s must check pods has reachability to google services. My case dns issue
    - Dns config has to change in velero deployment and nodeagent deamon sets.
    dnsConfig:
       nameservers:
        - 8.8.8.8
        - 8.8.4.4
    dnsPolicy: None

#create Namespace & Install apps
kubectl create namespace gcp
kubectl apply -f nginx.yaml -n gcp
kubectl apply -f busybox.yaml -n gcp
kubectl apply -f webserver.yaml -n gcp

#Verify apps
kubectl get po -n gcp
kubectl get pvc -n gcp

#Insert some data in mount points
kubectl exec -n gcp $(kubectl get pods -n gcp -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "Hello from nginx-updated" > /usr/share/nginx/html/index.html'
kubectl exec -n gcp $(kubectl get pods -n gcp -l app=busybox -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "Hello from busybox" > /data/data.txt'
kubectl exec -n gcp $(kubectl get pods -n gcp -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- sh -c 'echo "Hello from webserver" > /usr/local/apache2/htdocs/index.html'

#Verify data inside pods
    #kubectl exec -it -n gcp nginx-deployment-5c794987d4-r9gr6 -- bash
    #kubectl exec -it -n gcp busybox-pod -- sh
    #kubectl exec -n gcp busybox-pod -- ls -l /data

kubectl exec -n gcp busybox-pod -- cat /data/data.txt
kubectl exec -n gcp $(kubectl get pods -n gcp -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html
kubectl exec -n gcp $(kubectl get pods -n gcp -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html

#Create Velero backup
#velero backup create fs-backup-optout-global --include-namespaces gcp --include-resources deployments,pods,persistentvolumeclaims
velero backup create fs-backup-optout-global --include-namespaces gcp

#Verify backups
velero backup get
velero backup describe fs-backup-optout-global
velero describe backup fs-backup-test-2 --details
velero backup logs fs-backup-optout-global

#Delete apps
kubectl delete -f nginx.yaml -n gcp
kubectl delete -f busybox.yaml -n gcp
kubectl delete -f webserver.yaml -n gcp

#Restore apps from backup
velero restore create fs-restore-optout-global --from-backup fs-backup-optout

#Verify restore
velero restore get
velero restore describe fs-restore-optout-global
velero restore logs fs-restore-optout-global

#Verify pods inside information data should show
kubectl exec -n gcp $(kubectl get pods -n gcp -l app=nginx -o jsonpath='{.items[0].metadata.name}') -- cat /usr/share/nginx/html/index.html
kubectl exec -n gcp $(kubectl get pods -n gcp -l app=webserver -o jsonpath='{.items[0].metadata.name}') -- cat /usr/local/apache2/htdocs/index.html
kubectl exec -n gcp busybox-pod -- cat /data/data.txt

#Other
kubectl get pods -n gcp | grep busybox
kubectl get pods -n gcp --show-labels | grep busybox