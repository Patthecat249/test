# How to Install Cloud Automation Manager (CAM) in an Offline-Installation with CLI

## Introduction
This describe the process of the Installation of Cloud Automation Manager 3.1.2.1 on ICP 3.1.2.
Please go through the complete Installation-Procedure to become familiar with the procedure!
Change the Variables as you need!
Login to the ICP-Master-Node and don't leave the Session (cause of temporary Shell-Variables)!

## Preparation
### Create Installation/Download-Folder 
This folder is needed to place this installation-file.
#VARIABLES BEGIN#
```bash
export INST=/install
```
#VARIABLES END#
```bash
mkdir -p ${INST}
```
## Create NFS-folders
#These are the root-Folders of your PVs of the CAM-Installation
#VARIABLES BEGIN#
```bash
NFSPATH="/nfs/shared/cam"
```
#VARIABLES END#
```bash
mkdir -p \
     ${NFSPATH}/cam_db \
     ${NFSPATH}/cam_terraform/cam-provider-terraform \
     ${NFSPATH}/cam_logs/cam-provider-terraform \
     ${NFSPATH}/cam_bpd_appdata/mysql \
     ${NFSPATH}/cam_bpd_appdata/repositories \
     ${NFSPATH}/cam_bpd_appdata/workspace
chmod -R 2775 \
  ${NFSPATH}/CAM_db \
  ${NFSPATH}/CAM_logs \
  ${NFSPATH}/CAM_terraform \
  ${NFSPATH}/CAM_BPD_appdata

chown -R root:1000 \
  ${NFSPATH}/CAM_logs \
  ${NFSPATH}/CAM_BPD_appdata

chown -R root:1111 \
  ${NFSPATH}/CAM_terraform \
  ${NFSPATH}/CAM_logs/cam-provider-terraform

chown -R 999:999 \
  ${NFSPATH}/CAM_BPD_appdata/mysql \
  ${NFSPATH}/CAM_db
```
## Configure NFS-Exports-File and Exports NFS-Folder
```bash
echo "${NFSPATH} *(rw,nohide,insecure,no_subtree_check,async,no_root_squash)" >> /etc/exports
exportfs -a
```

## Download Installation File from IBM Fix Central
https://www-945.ibm.com/support/fixcentral
Download from IBM Fix Central > Search for "icp-cam-x86_64-3.1.2.1.tar.gz"
**!!! Top Right Corner !!!**

Download "x86 - icp-cam-x86_64-3.1.2.1.tar.gz"
ll ${INST}/icp-cam-x86_64-3.1.2.1.tar.gz 

-rw-r--r-- 1 root root 10266055420 May 20 10:20 /install/icp-cam-x86_64-3.1.2.1.tar.gz

## Extract Chart for customizing values.yaml
### List content
```bash
tar -tf ${INST}/icp-cam-x86_64-3.1.2.1.tar.gz
```

### Extract Content (only chart)
```bash
tar -xvf ${INST}/icp-cam-x86_64-3.1.2.1.tar.gz charts/ibm-cam-3.1.3.tgz
tar -xf ${INST}/charts/ibm-cam-3.1.3.tgz
```
## Load and PUSH Images from TAR-File to ICP-Registry
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
## Start Installation Process
Generate a deployment ServiceID API Key
- Important: NOTICE and capture the API-Key from the output of the following commands!!! It is needed later in the "values.yaml"-file

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
## Create ImagePullSecret
Is needed for the Installation process of CAM, so that the installation pods can access the ICP-Docker-Registry, where the Images are stored for the offline-installation.
```bash
#VARIABLES BEGIN#
export SECRET_NAME="docker-push-pull-secret"
#VARIABLES END#
kubectl create secret docker-registry ${SECRET_NAME} \
--docker-server="${ICPCLUSTER}:${DOCKERPORT}" \
--docker-username="${CLOUDCTLUSER}" \
--docker-password="${CLOUDCTLPASS}" \
--docker-email="admin@admin.local" \
--namespace=services
```







<details><summary>values.yaml</summary>
<p>

#### This is the values.yaml-file of CAM 3.1.3

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
  secretName:
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
