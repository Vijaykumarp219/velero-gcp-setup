# Restore Scenarios

```bash
velero restore create --from-backup full-backup
velero restore create restore-ns --from-backup app-ns-backup --include-namespaces=my-app
```