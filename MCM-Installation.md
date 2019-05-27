# How to Install Multi Cloud Manager (MCM) with CLI

## General
<details><summary>Introduction</summary>
<p>
## Introduction

This document describes the process of the Multi-Cloud-Manager-Installation 3.1.2 on IBM Cloud Private 3.1.2 within a CLI.
Please go through the complete Installation-Procedure to become familiar with the procedure!
Change the Variables as you need it!
</p>
</details>

<details><summary>Requirements</summary>
<p>
## Requirements

- ICP3.1.2 must be installed
- CLIs must be installed and configured
  - cloudctl
  - kubectl
  - helm
- NFS-Server must be configured and accesible from all ICP-Nodes (Master, Proxy, Worker, ...)
  - NFS-Client Software must be installed
  - In this tutorial the NFS-Server is installed on the ICP-Master-Node
</p>
</details>

<details><summary>How to use this manual</summary>
<p>
## How to use this manual
Go step-by-step from top to bottom through this guide and copy&paste the script-content into the CLI. Sometimes, you have to customized settings as your environment is different.
Open the CLI (ssh into) on your ICP-Master-Node. Copy the BASH-Content from this guide into the CLI and execute it. Please customize the variables, if you want to make changes.
</p>
</details>

## Preparation
<details><summary>Create Installation/Download-Folder</summary>
<p>
### Create Installation/Download-Folder 
This folder is needed to place the installation-tar-file.

```bash
export INST=/install/mcm
mkdir -p ${INST}
 
```
</p>
</details>

<details><summary>//Überarbeiten//Create NFS-folders</summary>
<p>
## Create NFS-folders
#These Folders will be used during the CAM-Installation

```bash
NFSPATH="/nfs/shared/mcm"
mkdir -p \
     ${NFSPATH}/cam_db \
     ${NFSPATH}/cam_terraform/cam-provider-terraform \
     ${NFSPATH}/cam_logs/cam-provider-terraform \
     ${NFSPATH}/cam_bpd_appdata/mysql \
     ${NFSPATH}/cam_bpd_appdata/repositories \
     ${NFSPATH}/cam_bpd_appdata/workspace
chmod -R 2775 \
  ${NFSPATH}/cam_db \
  ${NFSPATH}/cam_logs \
  ${NFSPATH}/cam_terraform \
  ${NFSPATH}/cam_bpd_appdata

chown -R root:1000 \
  ${NFSPATH}/cam_logs \
  ${NFSPATH}/cam_bpd_appdata

chown -R root:1111 \
  ${NFSPATH}/cam_terraform \
  ${NFSPATH}/cam_logs/cam-provider-terraform

chown -R 999:999 \
  ${NFSPATH}/cam_bpd_appdata/mysql \
  ${NFSPATH}/cam_db
   
```
</p>
</details>

<details><summary>Configure NFS-Exports-File and Exports NFS-Folder</summary>
<p>

## Configure NFS-Exports-File and Exports NFS-Folder

```bash
echo "${NFSPATH} *(rw,nohide,insecure,no_subtree_check,async,no_root_squash)" >> /etc/exports
exportfs -a
 
```
</p>
</details>

<details><summary>Download Installation File from IBM Fix Central and extract TAR-File</summary>
<p>

## Download Installation File from IBM Fix Central
https://www-945.ibm.com/support/fixcentral
Download from IBM Fix Central > Search for "icp-cam-x86_64-3.1.2.1.tar.gz"
**!!! Top Right Corner !!!**

**The Output should look like**

```bash

ll $INST/mcm-3.1.2.tgz

-rw-r--r-- 1 root root 10461012484 May 27 10:09 /install/mcm/mcm-3.1.2.tgz

```

</p>
</details>

<details><summary>Extract TAR-File</summary>
<p>

## Extract TAR-File
**It is important to extract both files (mcm-3.1.2.tgz and mcm-ppa-3.1.2.tgz)**

```bash

cd $INST
tar -xvf ${INST}/mcm-3.1.2.tgz
cd mcm-3.1.2/
tar -xvf ${INST}/mcm-3.1.2/mcm-ppa-3.1.2.tgz

```

<details><summary>Output should look like:</summary>
<p>

```bash

ll $INST/mcm-3.1.2/
total 10269008
drwxr-xr-x 2 root root       4096 Feb  4 21:15 ./
drwxr-xr-x 3 root root       4096 May 27 10:13 ../
-rw-r--r-- 1 root root   87716477 Feb  1 03:57 alerttargetcontroller-ppa-0.0.2-20190124T152648Z-TryBuy-TKAI-B6XHUQ.tar.gz
-rw-r--r-- 1 root root 8477099699 Feb  1 04:02 cem-mcm-ppa-2.1.1-TryBuy-TKAI-B6XHUQ.tar.gz
-rw-r--r-- 1 root root 1950621352 Feb  4 21:12 mcm-ppa-3.1.2.tgz

cd $INST/mcm-3.1.2/

tar -xvf $INST/mcm-3.1.2/mcm-ppa-3.1.2.tgz
charts/ibm-mcm-prod-3.1.2.tgz
charts/ibm-mcmk-prod-3.1.2.tgz
images/67ac9d4aac08e94147d9064752f31de453476a51b8f9264cc5acf8d9d7799a5a.tar.gz
images/6eaadbab9eb1ad80d0c9e541405fadb53d349b214ba385f09bc95777a4593e39.tar.gz
images/6bb699f930d685fc4773fa3ff0f62332ef282100ef138cbe728c4e7f56628882.tar.gz
images/5db3f1d91df04ae03580a2d8fa3a479de33d46e1f2861fcf18659436ad774a58.tar.gz
images/c9c5eafe1bfdfc73e53bcb4e175567045bcf8c0eb9dd607c74ecac523fa75cc5.tar.gz
manifest.json
manifest.yaml

```

</p>
</details>

</p>
</details>

## Installation
<details><summary>Load and PUSH Images from TAR-File to ICP-Registry</summary>
<p>

## Load and PUSH Images from TAR-File to ICP-Registry
Let's push the images included in the tar-file to the ICP-Docker-Registry. The installation-process of CAM needs these images.
1. Let's login to your ICP-Cluster in the "services" namespace
2. Let's login to your ICP-Docker-Registry
3. Load and push the images from tar-file into ICP-Docker-Registry

```bash
#VARIABLES BEGIN#
export CLOUDCTLUSER="admin"
export CLOUDCTLPASS="admin"
export ICPCLUSTER="mycluster.icp"
export DOCKERPORT="8500"
export ARCHIVEFILE=${INST}/mcm-3.1.2/mcm-ppa-3.1.2.tgz
#VARIABLES END#
cloudctl login -a https://${ICPCLUSTER}:8443 --skip-ssl-validation -u ${CLOUDCTLUSER} -p ${CLOUDCTLPASS} -n kube-system 
docker login ${ICPCLUSTER}:${DOCKERPORT} -u ${CLOUDCTLUSER} -p ${CLOUDCTLPASS}  
cd ${INST}
cloudctl catalog load-archive --archive ${ARCHIVEFILE} --registry ${ICPCLUSTER}:${DOCKERPORT}/kube-system
 
```

<details><summary>Output should look like:</summary>
<p>

```bash

root@glusterfs01:/install/mcm# cloudctl catalog load-archive --archive $INST/mcm-3.1.2/mcm-ppa-3.1.2.tgz --registry ${ICPCLUSTER}:${DOCKERPORT}/kube-system
Expanding archive
OK

Importing docker image(s)
  Processing image: mcm-compliance-amd64:3.1.2
    Loading Image
    Tagging Image
    Pushing image as: mycluster.icp:8500/kube-system/mcm-compliance-amd64:3.1.2
  Processing image: mcm-compliance-ppc64le:3.1.2
    Loading Image
    Tagging Image
    Pushing image as: mycluster.icp:8500/kube-system/mcm-compliance-ppc64le:3.1.2
    Creating manifest list as: mycluster.icp:8500/kube-system/mcm-compliance:3.1.2
    Annotating manifest list: mycluster.icp:8500/kube-system/mcm-compliance-amd64:3.1.2
    Annotating manifest list: mycluster.icp:8500/kube-system/mcm-compliance-ppc64le:3.1.2

docker images | grep kube-system

```
</p>
</details>



## Extract values.yaml-file

```bash

cd $INST/mcm-3.1.2/charts
tar -xvf ibm-mcm-prod-3.1.2.tgz
cd $INST/mcm-3.1.2/charts/ibm-mcm-prod
#vi values.yaml

```

</p>
</details>

<details><summary>Create Image-Push-Pull-secret</summary>
<p>

```bash

#VARIABLES BEGIN#
export SECRET_NAME="docker-push-pull-secret"
export KUBECTLCLI="/usr/local/bin/kubectl"
#VARIABLES END#
${KUBECTLCLI} create secret docker-registry ${SECRET_NAME} \
--docker-server="${ICPCLUSTER}:${DOCKERPORT}" \
--docker-username="${CLOUDCTLUSER}" \
--docker-password="${CLOUDCTLPASS}" \
--docker-email="admin@admin.local" \
--namespace="kube-system"

```

</p>
</details>

<details><summary>Edit values.yaml</summary>
<p>

#### This is the values.yaml-file of the MCM3.1.2-Chart
You can copy&paste the following content into a file, where you refer to later, when you install the MCM-Chart with the "helm install -f values.yaml"-command.

```YAML
# Licensed Materials - Property of IBM
# 5737-E67
# (C) Copyright IBM Corporation 2016, 2019 All Rights Reserved
# US Government Users Restricted Rights - Use, duplication or disclosure restricted by GSA ADP Schedule Contract with IBM Corp.

# Default values for hybrid-cluster-manager-chart.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
controller:
  repository: mcm-controller
  tag: 3.1.2
  pullPolicy: IfNotPresent
  enabled: true
  enableClusterAutoApprove: true

compliance:
  repository: mcm-compliance
  tag: 3.1.2
  pullPolicy: IfNotPresent
  enabled: true
  mcmNamespace: null

etcd:
  repository: etcd
  tag: 3.2.24
  pullPolicy: IfNotPresent
  persistence: false
  useDynamicProvisioning: true
  storageclassName: ""
  storageCapacity: 1Gi
  accessModes: ReadWriteMany

prometheus:
  image:
    name: prometheus
    tag: v2.3.1-f2
  args:
    retention: 24h
  persistentVolume:
    useDynamicProvisioning: true
    enabled: false
    storageClass: ""
    size: 10Gi
  resources:
    limits:
      cpu: 500m
      memory: 2Gi
    requests:
      cpu: 100m
      memory: 128Mi

configmapreload:
  image:
    repository: configmap-reload
    tag: v0.2.2-f2

mongodb:
  image:
    repository: mcm-mongodb
    tag: 3.1.2
    pullPolicy: IfNotPresent
  enabled: true
  rootUsername: root
  rootPassword: root
  username: mongo
  password: mongo
  dbName: mcm
  persistence: false
  useDynamicProvisioning: true
  storageclassName: 
  storageCapacity: 1Gi
  accessModes: ReadWriteMany

apiserver:
  repository: mcm-api
  tag: 3.1.2
  pullPolicy: IfNotPresent
  enabled: true

application:
  repository: mcm-application
  tag: 3.1.2
  pullPolicy: IfNotPresent
  enabled: true

console:
  enabled: true
  mcmui:
    image:
      repository: mcm-ui
      tag: 3.1.2
      pullPolicy: IfNotPresent
  mcmuiapi:
    image:
      repository: mcm-ui-api
      tag: 3.1.2
      pullPolicy: IfNotPresent
  header:
    enabled: true
    image:
      repository: icp-platform-header
      tag: 3.1.2
      pullPolicy: IfNotPresent
    defaultAdminUser: admin
  gremlinserver:
    image:
      repository: mcm-gremlin-server
      tag: 3.3.4
      pullPolicy: IfNotPresent
  mcmsearch:
    image:
      repository: mcm-search
      tag: 3.1.2
      pullPolicy: IfNotPresent

router:
  image:
    repository: icp-management-ingress
    tag: 2.2.3

nodeSelector:
  enabled: false
  arch: ""
  os: ""
  customLabelSelector: ""
  customLabelValue: ""

# this is needed when installing on non-ICP kubernetes clusters
# mcmdev@us.ibm.com
pullSecret:
  token: null
logLevel: 5
```

</p>
</details>

<details><summary>//ÜBERARBEITEN//Create PVs and PVCs (One Script) (Login ICP-Master-Node)</summary>
<p>

## Create PVs and PVCs (One Script) (Login ICP-Master-Node)
- First, you have to customize the variables
- Then copy&paste the output into the CLI of the ICP-Master-Nodes. 

### Tipp:
*Are you in the same session from the beginning of this tutorial? 
- yes=everything should be fine
- no=the Variable **NFSPATH** must be set and will be used!*

**BEGIN COPY&PASTE**
```bash
echo $NFSPATH

#VARIABLES BEGIN#
#VARIABLES for the mongo-database
AAA_PV_NAME="cam-mongo-pv"
AAA_PVC_NAME="cam-mongo-pvc"
AAA_LABEL="cam-mongo"
AAA_SIZE="15Gi"

#VARIABLES for the database-logs
BBB_PV_NAME="cam-logs-pv"
BBB_PVC_NAME="cam-logs-pvc"
BBB_LABEL="cam_logs"
BBB_SIZE="10Gi"

#VARIABLES for terraform
CCC_PV_NAME="cam-terraform-pv"
CCC_PVC_NAME="cam-terraform-pvc"
CCC_LABEL="cam-terraform"
CCC_SIZE="15Gi"

#VARIABLES for appdata
DDD_PV_NAME="cam-bpd-appdata-pv"
DDD_PVC_NAME="cam-bpd-appdata-pvc"
DDD_LABEL="cam-bpd-appdata"
DDD_SIZE="20Gi"

# General VARIABLES
PVCPOLICY="Recycle"
NFSSERVER="10.134.121.201"
NAMESPACE="services"
#VARIABLES END#

# The script starts here
#--- Create PV cam-mongo-pv ---
echo "--- Create PVC ${AAA_PV_NAME} ---"
${KUBECTLCLI} create -f - <<AAA
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "${AAA_PV_NAME}"
  labels:
    type: "${AAA_LABEL}"
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: "${AAA_SIZE}"
  persistentVolumeReclaimPolicy: "${PVCPOLICY}"
  nfs:
    server: "${NFSSERVER}"
    path: "${NFSPATH}/cam_db"
AAA
#--- Create PVC cam-mongo-pvc ---
echo "--- Create PVC ${AAA_PVC_NAME} ---"
${KUBECTLCLI} create -f - <<AAA
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "${AAA_PVC_NAME}"
  namespace: "${NAMESPACE}"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: "${AAA_SIZE}"
  volumeName: "${AAA_PV_NAME}"
  selector:
    matchLabels:
      type: "${AAA_LABEL}"
AAA

#--- Create PV cam-logs-pv ---
echo "--- Create PVC ${BBB_PV_NAME} ---"
${KUBECTLCLI} create -f - <<BBB
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "${BBB_PV_NAME}"
  labels:
    type: "${BBB_LABEL}"
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: "${BBB_SIZE}"
  persistentVolumeReclaimPolicy: "${PVCPOLICY}"
  nfs:
    server: "${NFSSERVER}"
    path: "${NFSPATH}/cam_logs"
BBB

#--- Create PVC cam-logs-pvc ---
echo "--- Create PVC ${BBB_PVC_NAME} ---"
${KUBECTLCLI} create -f - <<BBB
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "${BBB_PVC_NAME}"
  namespace: "${NAMESPACE}"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: "${BBB_SIZE}"
  volumeName: "${BBB_PV_NAME}"
  selector:
    matchLabels:
      type: "${BBB_LABEL}"
BBB

#--- Create PV cam-terraform-pv ---
echo "--- Create PVC ${CCC_PV_NAME} ---"
${KUBECTLCLI} create -f - <<CCC
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "${CCC_PV_NAME}"
  labels:
    type: "${CCC_LABEL}"
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: "${CCC_SIZE}"
  persistentVolumeReclaimPolicy: "${PVCPOLICY}"
  nfs:
    server: "${NFSSERVER}"
    path: "${NFSPATH}/cam_terraform"
CCC

#--- Create PVC cam-terraform-pvc ---
echo "--- Create PVC ${CCC_PVC_NAME} ---"
${KUBECTLCLI} create -f - <<CCC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "${CCC_PVC_NAME}"
  namespace: "${NAMESPACE}"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: "${CCC_SIZE}"
  volumeName: "${CCC_PV_NAME}"
  selector:
    matchLabels:
      type: "${CCC_LABEL}"
CCC

#--- Create PV cam-bpd-appdata-pv ---
echo "--- Create PVC ${DDD_PV_NAME} ---"
${KUBECTLCLI} create -f - <<DDD
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "${DDD_PV_NAME}"
  labels:
    type: "${DDD_LABEL}"
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: "${DDD_SIZE}"
  persistentVolumeReclaimPolicy: "${PVCPOLICY}"
  nfs:
    server: "${NFSSERVER}"
    path: "${NFSPATH}/cam_bpd_appdata"
DDD

#--- Create PVC cam-bpd-appdata-pvc ---
echo "--- Create PVC ${DDD_PVC_NAME} ---"
${KUBECTLCLI} create -f - <<DDD
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "${DDD_PVC_NAME}"
  namespace: "${NAMESPACE}"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: "${DDD_SIZE}"
  volumeName: "${DDD_PV_NAME}"
  selector:
    matchLabels:
      type: "${DDD_LABEL}"
DDD
# Show if PV's and PVC's are created
${KUBECTLCLI} get pvc | grep cam
${KUBECTLCLI} get pv | grep cam
 
```
**END COPY&PASTE**
</p>
</details>


<details><summary>Create Namespace</summary>
<p>

## Create Namespace

```bash

MCMNS="mcm-deployments"
${KUBECTLCLI} create namespace ${MCMNS}

```

</p>
</details>

<details><summary>Start Installation</summary>
<p>

## Start Installation
Please execute the follwing "helm"-command.

```bash

helm repo update
cd ${INST}/mcm-3.1.2
helm install --name mcm -f charts/ibm-mcm-prod/values.yaml local-charts/ibm-mcm-prod --tls
 
```
*Note: The Deployment-Process needs round about 7 Minutes!*

</p>
</details>

<details><summary>Verify Installation</summary>
<p>

## Verify Installation
You can check, if the installation of your CAM-deployment was successful. Please execute the following commands. All PODs must have a "1" in the column "Available"

```bash
${KUBECTLCLI} get -n kube-system pods
helm test mcm --tls
 
```
</p>
</details>

## Remove/Delete the Installation

<details><summary>How to clean up your CAM-Deployment</summary>
<p>

## How to clean up your CAM-Deployment
- uninstall the cam-helm-chart
- remove pv's and pvc's **(PLEASE CHECK, that no OTHER PVs and PVCs-names begin with "cam"!!!)**

```bash
#Delete Helm Chart
helm delete --purge cam --tls

#Delete Image-Pull-Secret
${KUBECTLCLI} delete secrets docker-push-pull-secret -n services

#Delete all PVCs in Namespace Service which name begins with "cam..."
${KUBECTLCLI} delete pvc $(k get pvc | grep cam | awk '{print $1}')

#Delete all PVs, which name begins with "cam..."
${KUBECTLCLI} delete pv $(k get pv | grep cam | awk '{print $1}')

#Delete Service-Keys and Service-IDs
cloudctl iam service-api-key-delete service-deploy-api-key service-deploy
cloudctl iam service-api-key-delete service-cloud-automation-manager-api-key service-cloud-automation-manager
cloudctl iam service-id-delete service-deploy
cloudctl iam service-id-delete service-cloud-automation-manager

#Delete NFS-Folder
NFSPATH="/nfs/shared/cam"
rm -rf $NFSPATH
 
```
</p>
</details>
