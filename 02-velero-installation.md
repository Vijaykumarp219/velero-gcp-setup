# Velero Installation Guide

> This document describes the steps to install the Velero client and server in a GCP environment.
---
## 🧰 Prerequisites

- GCP VM with Debian 12

- Kubernetes Cluster (self-hosted or GKE)

- GCP Service Account with required permissions and credentials file (`credentials-velero`)

- GCP Storage Bucket

---

## 1. Install Velero Client (CLI)

### Step 1.1: Login to GCP VM
### Step 1.2: Create a Dedicated User for Velero (Optional, for safety)

```bash

sudo  useradd  -m  velero

sudo  passwd  velero

sudo  usermod  -aG  sudo  velero

sudo  su  -  velero

```
### Step 1.3: Install Velero CLI

```bash

sudo  apt  update

sudo  apt  install  -y  wget

wget  https://github.com/vmware-tanzu/velero/releases/download/v1.15.2/velero-v1.15.2-linux-amd64.tar.gz

tar  -xvf  velero-v1.15.2-linux-amd64.tar.gz

sudo  mv  velero-v1.15.2-linux-amd64/velero  /usr/local/bin

```

---

## 2. Connect Velero CLI to Kubernetes Cluster

### Step 2.1: On Kubernetes Master Node

```bash

cd  /etc/kubernetes/

cat  admin.conf

```
### Step 2.2: On GCP VM (Velero CLI machine)

```bash

sudo  apt  install  -y  kubectl

# Paste the contents of admin.conf from Kubernetes cluster

vim  admin.conf

export  KUBECONFIG=admin.conf

echo  "export KUBECONFIG=admin.conf" >> ~/.bashrc

source  ~/.bashrc

```
---
## 3. Install Velero Server on Kubernetes

Velero can be installed in two primary modes:

-  **Generic** (with snapshotting support)

-  **File System Backup (FSB)** (using Kopia)

---
### 🧭 Option 1: Generic Snapshot Installation
```bash

velero  install  \
	--provider gcp \
	--plugins  velero/velero-plugin-for-gcp:v1.11.0  \
	--bucket $BUCKET \
	--secret-file  ./credentials-velero-dev

```
---

### 🗃️ Option 2: File System Backup (FSB) using Kopia
---
## 🎯 FSB Modes
There are 3 strategies for File System Backup using annotations:

### 1. Opt-in Mode

- Backup specific volumes using annotations.

```yaml

annotations:

backup.velero.io/backup-volumes: <volume-name>

```

- Install command:

```bash

velero  install  \
	--provider gcp \
	--plugins  velero/velero-plugin-for-gcp:v1.11.0  \
	--use-volume-snapshots=false \
	--bucket  $BUCKET  \
	--secret-file /home/velero/credentials-velero-dev \
	--use-node-agent  \
	--uploader-type kopia \
	--wait

```

---

### 2. Opt-out Mode (Annotation-based)

- Exclude specific volumes using annotations.

```yaml

annotations:

backup.velero.io/backup-volumes-excludes: <volume-name>

```

- Install command (same as above):

```bash

velero  install  \
	--provider gcp \
	--plugins  velero/velero-plugin-for-gcp:v1.11.0  \
	--use-volume-snapshots=false \
	--bucket  $BUCKET  \
	--secret-file /home/velero/credentials-velero-dev \
	--use-node-agent  \
	--uploader-type kopia \
	--wait

```
---

### 3. Global Opt-out Mode

- Exclude all volumes from FSB by default.

```bash

velero  install  \
	--provider gcp \
	--plugins  velero/velero-plugin-for-gcp:v1.11.0  \
	--use-volume-snapshots=false \
	--default-volumes-to-fs-backup  \
	--bucket $BUCKET \
	--secret-file  /home/velero/credentials-velero-dev  \
	--use-node-agent \
	--uploader-type  kopia  \
	--wait

```
---

✅ Velero client and server installation completed.

Continue to backup and restore workflows after this step.