# Backup Strategies

```bash
velero backup create full-backup --ttl 24h0m0s
velero backup create app-ns-backup --include-namespaces=my-app
velero backup create fsb-backup --include-namespaces=my-app
velero backup get
```