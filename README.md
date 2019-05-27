# How to Install Cloud Automation Manager (CAM) in an Offline-Installation with CLI

## General
<details><summary>Introduction</summary>
<p>
## Introduction

This describe the process of the Installation of Cloud Automation Manager 3.1.2.1 on ICP 3.1.2.
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

Open the CLI (ssh into) on your ICP-Master-Node. Copy the BASH-Content from this page into the CLI and execute it. Please customize the variables, if you want to make changes.
</p>
</details>

## Preparation
<details><summary>Create Installation/Download-Folder</summary>
<p>
### Create Installation/Download-Folder 
This folder is needed to place the installation-tar-file from IBM Fix Central.

```bash
export INST=/install
mkdir -p ${INST}
 
```
</p>
</details>

<details><summary>Create NFS-folders</summary>
<p>
## Create NFS-folders
#These Folders will be used during the CAM-Installation

```bash
NFSPATH="/nfs/shared/cam"
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

<details><summary>Download Installation File from IBM Fix Central</summary>
<p>

## Download Installation File from IBM Fix Central
https://www-945.ibm.com/support/fixcentral
Download from IBM Fix Central > Search for "icp-cam-x86_64-3.1.2.1.tar.gz"
**!!! Top Right Corner !!!**

**The Output should look like**

```bash
ll ${INST}/icp-cam-x86_64-3.1.2.1.tar.gz
 
-rw-r--r-- 1 root root 10266055420 May 20 10:20 /install/icp-cam-x86_64-3.1.2.1.tar.gz
```
</p>
</details>

<details><summary>(Optional) Extract Chart for customizing values.yaml</summary>
<p>

## Extract TAR-File
```bash
cd $INST
tar -xvf ${INST}/icp-cam-x86_64-3.1.2.1.tar.gz
 
```
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
#VARIABLES END#
cloudctl login -a https://${ICPCLUSTER}:8443 --skip-ssl-validation -u ${CLOUDCTLUSER} -p ${CLOUDCTLPASS} -n services 
docker login ${ICPCLUSTER}:${DOCKERPORT} -u ${CLOUDCTLUSER} -p ${CLOUDCTLPASS}  
cd ${INST}
cloudctl catalog load-archive --archive icp-cam-x86_64-3.1.2.1.tar.gz
 
```
</p>
</details>

<details><summary>Create Service-ID and Service-API-key</summary>
<p>

## Create Service-ID and Service-API-key
Generate a deployment ServiceID API Key
- Important: NOTICE and capture the API-Key from the output of the following commands!!! It is needed later in the "values.yaml"-file
- You need to capture the encrypted string, not just the name of the api-key.

```bash
#VARIABLES BEGIN#
export serviceIDName='service-deploy'
export serviceApiKeyName='service-deploy-api-key'
#VARIABLES END#
cloudctl login -a https://${ICPCLUSTER}:8443 --skip-ssl-validation -u ${CLOUDCTLUSER} -p ${CLOUDCTLPASS} -n services
cloudctl iam service-id-create ${serviceIDName} -d 'Service ID for service-deploy'
cloudctl iam service-policy-create ${serviceIDName} -r Administrator,ClusterAdministrator --service-name 'idmgmt'
cloudctl iam service-policy-create ${serviceIDName} -r Administrator,ClusterAdministrator --service-name 'identity'
cloudctl iam service-api-key-create ${serviceApiKeyName} ${serviceIDName} -d 'Api key for service-deploy'
 
```
</p>
</details>

<details><summary>Create ImagePullSecret</summary>
<p>

## Create ImagePullSecret
Is needed for the Installation process of CAM, so that the installation pods can access the ICP-Docker-Registry, where the Images are stored for the offline-installation.

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
--namespace=services
 
```
</p>
</details>

<details><summary>Edit values.yaml</summary>
<p>

## Edit the "values.yaml"
*Please change the following values/parameters in the values.yaml-file*

- **global.image.secret**=*docker-push-pull-secret*
- **global.iam.deployApiKey**=*CrypticCodeFromTheDeploymentApiKey*
- **offline**=*true*
- **service.namespace**=*services*
- **image.repository**=*"mycluster.icp:8500/services/"*
- **camMongoPV.existingClaimName**=*"cam-mongo-pvc"*
- **camLogsPV.existingClaimName**=*"cam-logs-pvc"*
- **camTerraformPV.existingClaimName**=*"cam-terraform-pvc"*
- **camBPDAppDataPV.existingClaimName**=*"cam-bpd-appdata-pvc"*

#### This is the values.yaml-file of the CAM-3.1.3-Chart
You can copy&paste the following content into a file, where you refer to later, when you install the CAM-Chart with the "helm install -f values.yaml"-command.

Please, make sure, that you change the necessary parts in the values.yaml-file to your environment. 
If you have done everything step-by-step in this tutorial, then you only have to change the **API-KEY**

```YAML
# ##############################################################################
# Licensed Materials - Property of IBM.
# Copyright IBM Corporation 2017. All Rights Reserved.
# U.S. Government Users Restricted Rights - Use, duplication or disclosure
# restricted by GSA ADP Schedule Contract with IBM Corp.
#
# Contributors:
#  IBM Corporation - initial API and implementation
# ##############################################################################
---
global:
  image:
    secretName: "docker-push-pull-secret"
  id:
    productID: "IBMCloudAutomationManager_5737E67_3121_EE_000"
  iam:
    deployApiKey: "1VAmX3YhowgIxf2plwnX4nQYJ8goy1eVZssyBb7BRqLn"
  offline: true
  audit: false
# arch: ppc64le
# arch: s390x
arch: amd64
service:
  namespace: services
managementConsole:
  port: 30000

secureValues:
  secretName: ""
database:
  bundled: true
image:
  repository: "mycluster.icp:8500/services/"
  tag: 3.1.2.1
  pullPolicy: IfNotPresent
  dockerconfig: ""
proxy:
  useProxy: false
camMongoPV:
  name: "cam-mongo-pv"
  persistence:
    enabled: true
    useDynamicProvisioning: false
    # Specify the name of the Existing Claim to be used by your application
    # empty string means don't use an existClaim
    existingClaimName: "cam-mongo-pvc"
    # Specify the name of the StorageClass
    # empty string means don't use a StorageClass
    storageClassName: ""
    accessMode: ReadWriteMany
    size: 15Gi
camLogsPV:
  name: "cam-logs-pv"
  persistence:
    enabled: true
    useDynamicProvisioning: false
    # Specify the name of the Existing Claim to be used by your application
    # empty string means don't use an existClaim
    existingClaimName: "cam-logs-pvc"
    # Specify the name of the StorageClass
    # empty string means don't use a StorageClass
    storageClassName: ""
    accessMode: ReadWriteMany
    size: 10Gi
camTerraformPV:
  name: "cam-terraform-pv"
  persistence:
    enabled: true
    useDynamicProvisioning: false
    # Specify the name of the Existing Claim to be used by your application
    # empty string means don't use an existClaim
    existingClaimName: "cam-terraform-pvc"
    # Specify the name of the StorageClass
    # empty string means don't use a StorageClass
    storageClassName: ""
    accessMode: ReadWriteMany
    size: 15Gi
camBPDAppDataPV:
  name: "cam-bpd-appdata-pv"
  persistence:
    enabled: true
    useDynamicProvisioning: false
    existingClaimName: "cam-bpd-appdata-pvc"
    storageClassName: ""
    accessMode: ReadWriteMany
    size: 15Gi
camBroker:
  replicaCount: 1
camProxy:
  replicaCount: 1
camAPI:
  replicaCount: 1
  camSecret:
    secretName: cam-api-secret
  certificate:
    certName: cert
camUI:
  replicaCount: 1
  camUISecret:
    secretName: cam-ui-secret
    sessionKey: "opsConsole.sid"
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 1
    memory: 8Gi
camBPDUI:
  bundled: true
camBPDCDS:
  replicaCount: 1
  resources:
    requests:
      memory: 128Mi
      cpu: 100m
    limits:
      memory: 256Mi
      cpu: 200m
  options:
    debug:
      enabled: false
    customSettingsFile: ""
camBPDMDS:
  replicaCount: 1
  resources:
    requests:
      memory: 128Mi
      cpu: 100m
    limits:
      memory: 256Mi
      cpu: 200m
camBPDDatabase:
  bundled: true
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
camBPDExternalDatabase:
  type: ""
  name: ""
  url: ""
  port: ""
  secret: ""
  extlibPV:
    existingClaimName: ""
camBPDResources:
  requests:
    cpu: 1000m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 2Gi
auditService:
  image:
    repository: "mycluster.icp:8500/ibmcom/"
    tag: ""
    pullPolicy: IfNotPresent
    pullSecret: ""
  resources:
    limits:
      cpu: 200m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 256Mi
  config:
    journalPath: '/run/systemd/journal'
camLoggingPolicies:
  logLevel: info
camBpmProvider:
  replicaCount: 0
camIcoProvider:
  replicaCount: 0
```

</p>
</details>

<details><summary>Create PVs and PVCs (One Script) (Login ICP-Master-Node)</summary>
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

<details><summary>Start Installation</summary>
<p>

## Start Installation
Please execute the follwing "helm"-command.
```bash
cd ${INST}
helm install --name cam -f charts/ibm-cam/values.yaml local-charts/ibm-cam --tls
 
```
</p>
</details>

<details><summary>Verify Installation</summary>
<p>

## Verify Installation
You can check, if the installation of your CAM-deployment was successful. Please execute the following commands. All PODs must have a "1" in the column "Available"

```bash
${KUBECTLCLI} get -n services pods
helm test cam --tls
 
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
