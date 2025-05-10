# GCP IAM Setup for Velero (Dev Environment)

  

## 1. Create a GCP Storage Bucket

### Log in to gcloud:

```bash

gcloud  auth  login

```
### Set a bucket name:

```bash

BUCKET=velero-k8s-bucket-dev

```

### Create a multiregional bucket:

```bash

gsutil  mb  -c  STANDARD  gs://$BUCKET/

```

### OR for a specific region:

```bash

gsutil  mb  -c  STANDARD  -l  europe-west3  gs://$BUCKET/

```

### Enable Uniform Bucket-Level Access:

```bash

gsutil  uniformbucketlevelaccess  set  on  gs://$BUCKET/

```
---

## 2. Create a Service Account and Bind Roles

### 2.1 View current config:

```bash

gcloud  config  list

```

### 2.2 Set the project ID variable:

```bash

PROJECT_ID=$(gcloud  config  get-value  project)

```

### 2.3 Create the Velero service account:

```bash

GSA_NAME=velero_dev

gcloud  iam  service-accounts  create  $GSA_NAME  \
	--display-name "Velero service account for dev"

```
### 2.4 List service accounts:

```bash

gcloud  iam  service-accounts  list

```
### 2.5 Set the service account email variable:

```bash

SERVICE_ACCOUNT_EMAIL=$(gcloud  iam  service-accounts  list  \
	--filter="displayName:Velero service account for dev" \
	--format  'value(email)')

```
---
## 3. Create a Custom Role for Velero
### 3.1 Log in to gcloud:

```bash

gcloud  auth  login

```
### 3.2 Define required permissions:
```bash

ROLE_PERMISSIONS=(
	compute.disks.get
	compute.disks.create
	compute.disks.createSnapshot
	compute.projects.get
	compute.snapshots.get
	compute.snapshots.create
	compute.snapshots.useReadOnly
	compute.snapshots.delete
	compute.zones.get
	storage.objects.create
	storage.objects.delete
	storage.objects.get
	storage.objects.list
	iam.serviceAccounts.signBlob
)

```
### 3.3 Create the custom IAM role:
```bash

gcloud  iam  roles  create  velero_dev.server  \
	--project $PROJECT_ID \
	--title  "Velero Server for dev"  \
	--permissions "$(IFS=','; echo "${ROLE_PERMISSIONS[*]}")"

```
### 3.4 Bind the IAM role to the service account:
```bash

gcloud  projects  add-iam-policy-binding  $PROJECT_ID  \
	--member serviceAccount:$SERVICE_ACCOUNT_EMAIL \
	--role  projects/$PROJECT_ID/roles/velero_dev.server

```
---
## 4. Grant Storage Access to Velero

  

```bash

gsutil  iam  ch  serviceAccount:$SERVICE_ACCOUNT_EMAIL:objectAdmin  gs://${BUCKET}

```
---
## 5. Generate Velero Credentials

```bash

gcloud  iam  service-accounts  keys  create  credentials-velero-dev  \
	--iam-account $SERVICE_ACCOUNT_EMAIL

```
This will generate the `credentials-velero-dev` JSON key file, which is used during Velero installation.