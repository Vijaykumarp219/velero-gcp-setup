
# Kopia FSB Setup

## Opt-in Pod Annotation:

```yaml

metadata:
	namespace: velero
	annotations:
	backup.velero.io/backup-volumes: <volume name>

```

## Opt-out:

```yaml

metadata:
	namespace: velero
	annotations:
	backup.velero.io/backup-volumes-excludes: <volume name>

```
## Global Opt-out:

```yaml

metadata:
	name: global
	namespace: velero

```