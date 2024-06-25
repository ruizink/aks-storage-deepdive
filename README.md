### Azure Kubernetes Services (AKS) - Storage Deep Dive (Lvl 300) 

Repo with all the content needed for the AKS Storage Deep Dive session.

![alt text](./img/banner.jpeg)
<small> Image generated by AI

## Table of Contents
1. [Pre-Requirements](#1-pre-requirements)
2. [Environment Setup](#2-environment-setup)
3. [Lab 1: Provision/enable CSI Storage Drivers and Expand Without Downtime](#3-lab-1-provisionenable-csi-storage-drivers-and-expand-without-downtime)
4. [Lab 2: Provision/enable Azure Elastic SAN in an existing AKS Cluster using iSCSI CSI driver](#4-lab-2-provisionenable-azure-elastic-san-in-an-existing-aks-cluster-using-iscsi-csi-driver)
5. [Lab 3: Provision/enable Azure Container Storage in an AKS Cluster and Stress Testing](#5-lab-3-provisionenable-azure-container-storage-in-an-aks-cluster-and-stress-testing)
6. [Lab 4: Manage and expand Ephemeral storage](#6-lab-4-manage-and-expand-ephemeral-storage)
7. [Lab 5: Provision/enable NetApp Files NFS volumes, SMB volumes and dual-protocol volumes](#7-lab-5-provisionenable-netapp-files-nfs-volumes-smb-volumes-and-dual-protocol-volumes)
8. [Lab 6: Provision/enable NVMe PV in AKS Nodepool](#8-lab-6-provisionenable-nvme-pv-in-aks-nodepool)
9. [Lab 7: Configure Backup on a cluster and how to use Resource Modification to patch backed-up](#9-lab-7-configure-backup-on-a-cluster-and-how-to-use-resource-modification-to-patch-backed-up)
10. [Lab 8: Configure Velero using Azure Blob Storage on a cluster](#10-lab-8-configure-velero-using-azure-blob-storage-on-a-cluster)

## 1. Pre-Requirements
- **Azure Subscription** - [Signup for a free account.](https://azure.microsoft.com/free/)
- **Visual Studio Code** - [Download it for free.](https://code.visualstudio.com/download)
- **GitHub Account** - [Signup for a free account.](https://github.com/signup)
- **AKS Cluster** - [Learn about the Service.](https://azure.microsoft.com/en-us/products/kubernetes-service)
- **Azure CLI** - [Download it for free.](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

## 2. Environment Setup
### 2.1. Create Azure resources

In this step, we'll **provision Azure resources** for our Lab, and put it ready to use.
- AKS (Azure Kubernetes Services) - CNI Overlay Network Plugin & Standard API

### Run this Lab with GitHub Codespaces

This repo also includes [DevContainer configuration](./.devcontainer/devcontainer.json), so you can open the repo using [GitHub Codespaces](https://docs.github.com/en/codespaces/overview). This will allow you to run all these Lab exercises, without having to install or having any extension being used on your local machine. When the Codespace is created, you can run the steps using the same instructions as below.

This is an optional step, and you can skip it if you want to run the Labs on your local machine.

### 2.2. Prepare the local environment (Local Machine or GitHub Codespaces)

Please check the **[pre-requisites](pre-requisites.md)** file for the setup of the environment.

After running all the pre-requisites, please follow the below steps

### 2.2.1. Clone the repository (Local Machine only)

```poweshell
git clone https://github.com/marconsilva/aks-storage-deepdive.git
cd aks-storage-deep-dive
```

### 2.2.2. Login into the cluster
```powershell
#login into the AKS Cluster
az account set --subscription $SUBSCRIPTION_ID
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER --overwrite-existing
```

### 2.2.3. Deploy an Application

The application used in the next steps is the AKS Store Demo application, that can be found on the below repo. 

This sample demo app consists of a group of containerized microservices that can be easily deployed into an Azure Kubernetes Service (AKS) cluster. This is meant to show a realistic scenario using a polyglot architecture, event-driven design, and common open source back-end services (eg - RabbitMQ, MongoDB). The application also leverages OpenAI's GPT-3 models to generate product descriptions. This can be done using either [Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/overview) or [OpenAI](https://openai.com/). We will not be using the OpenAI part of the application in this Lab.

This application is inspired by another demo app called [Red Dog](https://github.com/Azure/reddog-code).

> &#8505; **Note**
> This is not meant to be an example of perfect code to be used in production, but more about showing a realistic application running in AKS. 


**Github Repo:** [AKS Store Demo](https://github.com/Azure-Samples/aks-store-demo)


```powershell
# create the namespace where the demo app will be deployed
kubectl create ns aksappga
```

```powershell
# deploy the app into the namespace
kubectl apply -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml -n aksappga
```

```powershell
# expose the store-front service via unmanaged nginx ingress controller
kubectl apply -f .\nginx-ingress.yaml
```

```powershell
# get the public IP of the ingress controller
kubectl get svc store-front -n aksappga -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

```powershell	
# curl the store-front service
curl http://<ingress-ip>
```

Congratulations! You have successfully deployed the AKS Store Demo application, and have a working cluster for running the AKS Storage Deep Dive Labs.

## 3. Lab 1: Provision/enable CSI Storage Drivers and expand without downtime

The Container Storage Interface (CSI) is a standard for exposing arbitrary block and file storage systems to containerized workloads on Kubernetes. By adopting and using CSI, Azure Kubernetes Service (AKS) can write, deploy, and iterate plug-ins to expose new or improve existing storage systems in Kubernetes without having to touch the core Kubernetes code and wait for its release cycles.

The CSI storage driver support on AKS allows you to natively use:

- **Azure Disks** can be used to create a Kubernetes DataDisk resource. Disks can use Azure Premium Storage, backed by high-performance SSDs, or Azure Standard Storage, backed by regular HDDs or Standard SSDs. For most production and development workloads, use Premium Storage. Azure Disks are mounted as ReadWriteOnce and are only available to one node in AKS. For storage volumes that can be accessed by multiple nodes simultaneously, use Azure Files.
- **Azure Files** can be used to mount an SMB 3.0/3.1 share backed by an Azure storage account to pods. With Azure Files, you can share data across multiple nodes and pods. Azure Files can use Azure Standard storage backed by regular HDDs or Azure Premium storage backed by high-performance SSDs.
- **Azure Blob** storage can be used to mount Blob storage (or object storage) as a file system into a container or pod. Using Blob storage enables your cluster to support applications that work with large unstructured datasets like log file data, images or documents, HPC, and others. Additionally, if you ingest data into Azure Data Lake storage, you can directly mount and use it in AKS without configuring another interim filesystem.

**IMPORTANT**

Starting with Kubernetes version 1.26, in-tree persistent volume types kubernetes.io/azure-disk and kubernetes.io/azure-file are deprecated and will no longer be supported. Removing these drivers following their deprecation is not planned, however you should migrate to the corresponding CSI drivers disk.csi.azure.com and file.csi.azure.com. 

### 3.1. Enable CSI Storage Drivers

Start by checking the current storage classes available in the AKS cluster.

```powershell	
kubectl get storageclass
```

```powershell	
PS C:\farfetch\aks-storage-deepdive> kubectl get storageclass
NAME                PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
default (default)   disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   48m
PS C:\farfetch\aks-storage-deepdive>
```

Now enable the CSI drivers on the AKS cluster, and check the new storage classes available.

```powershell	
az aks update --name $CLUSTER --resource-group $RESOURCE_GROUP --enable-disk-driver --enable-file-driver --enable-blob-driver --enable-snapshot-controller
```

```powershell	
kubectl get storageclass
```

In addition to in-tree driver features, Azure Disk CSI driver supports the following features:

- Performance improvements during concurrent disk attach and detach
- In-tree drivers attach or detach disks in serial, while CSI drivers attach or detach disks in batch. There's significant improvement when there are multiple disks attaching to one node.
- Premium SSD v1 and v2 are supported.
  - PremiumV2_LRS only supports None caching mode
- Zone-redundant storage (ZRS) disk support
  - Premium_ZRS, StandardSSD_ZRS disk types are supported. ZRS disk could be scheduled on the zone or non-zone node, without the restriction that disk volume should be co-located in the same zone as a given node. For more information, including which regions are supported, see Zone-redundant storage for managed disks.
- Snapshot
- Volume clone
- Resize disk PV without downtime

### 3.2. Dynamically create Azure Disks PVs by using the built-in storage classes

When you use the Azure Disk CSI driver on AKS, there are two more built-in StorageClasses that use the Azure Disk CSI storage driver. The other CSI storage classes are created with the cluster alongside the in-tree default storage classes.

- **managed-csi**: Uses Azure Standard SSD locally redundant storage (LRS) to create a managed disk. Effective starting with Kubernetes version 1.29, in Azure Kubernetes Service (AKS) clusters deployed across multiple availability zones, this storage class utilizes Azure Standard SSD zone-redundant storage (ZRS) to create managed disks.
- **managed-csi-premium**: Uses Azure Premium LRS to create a managed disk. Effective starting with Kubernetes version 1.29, in Azure Kubernetes Service (AKS) clusters deployed across multiple availability zones, this storage class utilizes Azure Premium zone-redundant storage (ZRS) to create managed disks.

The reclaim policy in both storage classes ensures that the underlying Azure Disks are deleted when the respective PV is deleted. The storage classes also configure the PVs to be expandable. You just need to edit the persistent volume claim (PVC) with the new size.

Create an example pod and respective PVC by running the kubectl apply command:

```powershell	
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/pvc-azuredisk-csi.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/nginx-pod-azuredisk.yaml
```

After the pod is in the running state, run the following command to create a new file called test.txt.

```powershell	
kubectl exec nginx-azuredisk -- touch /mnt/azuredisk/test.txt
```

To validate the disk is correctly mounted, run the following command and verify you see the test.txt file in the output:

```powershell	
kubectl exec nginx-azuredisk -- ls /mnt/azuredisk
```

### 3.2. Expand Volume without downtime

You can request a larger volume for a PVC. Edit the PVC object, and specify a larger size. This change triggers the expansion of the underlying volume that backs the PV.

In AKS, the built-in managed-csi storage class already supports expansion, so use the PVC created earlier with this storage class. The PVC requested a 10-Gi persistent volume. You can confirm by running the following command:

```powershell	
kubectl exec -it nginx-azuredisk -- df -h /mnt/azuredisk
```

The output of the command resembles the following example:

```output	
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc        9.8G   42M  9.8G   1% /mnt/azuredisk
```

Now we will expand the PVC by increasing the **spec.resources.requests.storage** field running the following command:

```powershell	
kubectl patch pvc pvc-azuredisk --type merge --patch '{"spec": {"resources": {"requests": {"storage": "15Gi"}}}}'
```

Run the following command to confirm the volume size has increased:

```powershell	
kubectl get pv
```

The output of the command resembles the following example:

```output	
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                     STORAGECLASS   REASON   AGE
pvc-391ea1a6-0191-4022-b915-c8dc4216174a   15Gi       RWO            Delete           Bound    default/pvc-azuredisk                     managed-csi             2d2h
(...)
```

And after a few minutes, run the following commands to confirm the size of the PVC:

```powershell	
kubectl get pvc pvc-azuredisk
```

The output of the command resembles the following example:

```output	
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-azuredisk   Bound    pvc-391ea1a6-0191-4022-b915-c8dc4216174a   15Gi       RWO            managed-csi    2d2h
```

Run the following command to confirm the size of the disk inside the pod:

```powershell	
kubectl exec -it nginx-azuredisk -- df -h /mnt/azuredisk
```

The output of the command resembles the following example:

```output	
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc         15G   46M   15G   1% /mnt/azuredisk
```

## 4. Lab 2: Provision/enable Azure Elastic SAN in an existing AKS Cluster using iSCSI CSI driver

This Lab explains how to connect an Azure Elastic storage area network (SAN) volume from an Azure Kubernetes Service (AKS) cluster. To make this connection, enable the Kubernetes iSCSI CSI driver on your cluster. With this driver, you can access volumes on your Elastic SAN by creating persistent volumes on your AKS cluster, and then attaching the Elastic SAN volumes to the persistent volumes.

### 4.1. About the iSCSI CSI driver
The iSCSI CSI driver is an open source project that allows you to connect to a Kubernetes cluster over iSCSI. Since the driver is an open source project, Microsoft won't provide support from any issues stemming from the driver, itself.

The Kubernetes iSCSI CSI driver is available on GitHub:

- [Kubernetes iSCSI CSI driver repository](https://github.com/kubernetes-csi/csi-driver-iscsi)
- [Readme](https://github.com/kubernetes-csi/csi-driver-iscsi/blob/master/README.md)
- [Report iSCSI driver issues](https://github.com/kubernetes-csi/csi-driver-iscsi/issues)

### 4.2. Deploy an Elastic SAN volume

The following command creates an Elastic SAN that uses zone-redundant storage.

```powershell	
az elastic-san create -n $EsanName -g $RESOURCE_GROUP -l $LOCATION --base-size-tib 50 --extended-capacity-size-tib 20 --sku "{name:Premium_ZRS,tier:Premium}"
```

Now that you've configured the basic settings and provisioned your storage, you can create volume groups. Volume groups are a tool for managing volumes at scale. Any settings or configurations applied to a volume group apply to all volumes associated with that volume group.

```powershell
az elastic-san volume-group create --elastic-san-name $EsanName -g $RESOURCE_GROUP -n $EsanVgName
```

Now that you've configured the SAN itself, and created at least one volume group, you can create volumes.

Volumes are usable partitions of the SAN's total capacity, you must allocate a portion of that total capacity as a volume in order to use it. Only the actual volumes themselves can be mounted and used, not volume groups.


```powershell
az elastic-san volume create --elastic-san-name $EsanName -g $RESOURCE_GROUP -v $EsanVgName -n $VolumeName --size-gib 2000
```

### 4.3. Using Kubernetes iSCSI CSI driver with Azure Elastic SAN

Before we start check to see that the driver isn't installed and then run the following script to install the driver.

```powershell	
kubectl -n kube-system get pod -o wide -l app=csi-iscsi-node
```

```powershell	
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\install-driver.ps1
```
After deployment, check the pods status again to verify that the driver installed.

```powershell	
kubectl -n kube-system get pod -o wide -l app=csi-iscsi-node
```

Now lets get all the information needed to connect the Elastic SAN volume that was created previously to the AKS cluster.

```powershell	
az elastic-san volume show -g $RESOURCE_GROUP -e $EsanName -v $EsanVgName -n $VolumeName
```

from the output extract the following information:
- **targetPortalHostname**
- **targetPortalPort**
- **targetIqn**

Now you will need to make a copy of the esan-pv.yaml file and replace the placeholders with the information you got from the previous command.

After creating the pv.yml file, create a persistent volume with the following command:

```powershell	
kubectl apply -f pathtoyourfile/esan-pv.yaml
```

Next, create a persistent volume claim. Use the storage class we defined earlier with the persistent volume we defined. Simply run the following command:

```powershell
kubectl apply -f esan-pvc.yaml
```

To verify your PersistentVolumeClaim is created and bound to the PersistentVolume, run the following command:

```powershell
kubectl get pvc iscsiplugin-pvc
```

Finally lets create a pod manifest that uses this volume.

```powershell
kubectl create ns aksesan
kubectl apply -f esan-pod.yaml
kubectl apply -f esan-pod-service.yaml
```

## 5. Lab 3: Provision/enable Azure Container Storage in an AKS Cluster and Stress Testing



## 6. Lab 4: Manage and expand Ephemeral storage

## 7. Lab 5: Provision/enable NetApp Files NFS volumes, SMB volumes and dual-protocol volumes

## 8. Lab 6: Provision/enable NVMe PV in AKS Nodepool

## 9. Lab 7: Configure Backup on a cluster and how to use Resource Modification to patch backed-up

Azure Kubernetes Service (AKS) backup is a simple, cloud-native process you can use to back up and restore containerized applications and data that run in your AKS cluster. You can configure scheduled backups for cluster state and application data that's stored on persistent volumes in CSI driver-based Azure Disk Storage. The solution gives you granular control to choose a specific namespace or an entire cluster to back up or restore by storing backups locally in a blob container and as disk snapshots. You can use AKS backup for end-to-end scenarios, including operational recovery, cloning developer/test environments, and cluster upgrade scenarios.

Use AKS backup to back up your AKS workloads and persistent volumes that are deployed in AKS clusters. The solution requires the Backup extension to be installed inside the AKS cluster. The Backup vault communicates to the extension to complete operations that are related to backup and restore. Using the Backup extension is mandatory, and the extension must be installed inside the AKS cluster to enable backup and restore for the cluster. When you configure AKS backup, you add values for a storage account and a blob container where backups are stored.

Lets start by creating a new resource group and a new backup vault to store the backups.

```powershell
az group create --name $RESOURCE_GROUP_VAULT --location $LOCATION

az dataprotection backup-vault create `
   --vault-name $BACK_VAULT_NAME `
   -g $RESOURCE_GROUP_VAULT `
   --storage-setting "[{type:'LocallyRedundant',datastore-type:'VaultStore'}]"
```

Lets create the blob storage account and the container to store the backups.

```powershell
az group create --name $SA_RG --location westeurope

az storage account create `
   --name $SA_NAME `
   --resource-group $SA_RG `
   --sku Standard_LRS

$ACCOUNT_KEY=$(az storage account keys list --account-name $SA_NAME -g $SA_RG --query "[0].value" -o tsv)

az storage container create `
   --name $BLOB_CONTAINER_NAME `
   --account-name $SA_NAME `
   --account-key $ACCOUNT_KEY
```

Now lets create first AKS cluster with CSI Disk Driver and Snapshot Controller

```powershell
az aks get-versions -l westeurope -o table

az group create --name $RESOURCE_GROUP_1 --location westeurope

az aks create -g $RESOURCE_GROUP_1 -n $CLUSTER_1 -k "1.27.3" --zones 1 2 3 --node-vm-size "Standard_B2als_v2"
```

We just need now to verify that CSI Disk Driver and Snapshot Controller are installed

```powershell
az aks show -g $RESOURCE_GROUP_1 -n $CLUSTER_1 --query storageProfile
```

If not installed, you can install them using the following command

```powershell
az aks update -g $RESOURCE_GROUP_1 -n $CLUSTER_1 --enable-disk-driver --enable-snapshot-controller
```

Lets now create a second AKS cluster with CSI Disk Driver and Snapshot Controller and verify that they are installed

```powershell
az group create --name $RESOURCE_GROUP_2 --location westeurope

az aks create -g $RESOURCE_GROUP_2 -n $CLUSTER_2 -k "1.27.3" --zones 1 2 3 --node-vm-size "Standard_B2als_v2"

# Verify that CSI Disk Driver and Snapshot Controller are installed

az aks show -g $RESOURCE_GROUP_2 -n $CLUSTER_2 --query storageProfile
```

If not installed, you ca install it with this command:

```powershell
az aks update -g $RESOURCE_GROUP_2 -n $CLUSTER_2 --enable-disk-driver --enable-snapshot-controller
```

We can now prepare the backup extension and install it in the first AKS cluster

```powershell
az k8s-extension create --name azure-aks-backup `
   --extension-type Microsoft.DataProtection.Kubernetes `
   --scope cluster `
   --cluster-type managedClusters `
   --cluster-name $CLUSTER_1 `
   --resource-group $RESOURCE_GROUP_1 `
   --release-train stable `
   --configuration-settings `
   blobContainer=$BLOB_CONTAINER_NAME `
   storageAccount=$SA_NAME `
   storageAccountResourceGroup=$SA_RG `
   storageAccountSubscriptionId=$SUBSCRIPTION_ID

# View Backup Extension installation status

az k8s-extension show --name azure-aks-backup --cluster-type managedClusters --cluster-name $CLUSTER_1 -g $RESOURCE_GROUP
```

We now need to Enable Trusted Access in AKS cluster to allow the backup extension to access the storage account

```powershell
$BACKUP_VAULT_ID=$(az dataprotection backup-vault show --vault-name $BACK_VAULT_NAME -g $RESOURCE_GROUP_VAULT --query id -o tsv)

az aks trustedaccess rolebinding create –n trustedaccess `
   -g $RESOURCE_GROUP_1 `
   --cluster-name $CLUSTER_1 `
   --source-resource-id $BACKUP_VAULT_ID `
   --roles Microsoft.DataProtection/backupVaults/backup-operator

az aks trustedaccess rolebinding list -g $RESOURCE_GROUP_1 --cluster-name $CLUSTER
```

We need to do the same for the second AKS cluster

```powershell

az k8s-extension create --name azure-aks-backup `
   --extension-type Microsoft.DataProtection.Kubernetes `
   --scope cluster `
   --cluster-type managedClusters `
   --cluster-name $CLUSTER_2 `
   --resource-group $RESOURCE_GROUP_2 `
   --release-train stable `
   --configuration-settings `
   blobContainer=$BLOB_CONTAINER_NAME `
   storageAccount=$SA_NAME `
   storageAccountResourceGroup=$SA_RG `
   storageAccountSubscriptionId=$SUBSCRIPTION_ID

# View Backup Extension installation status

az k8s-extension show --name azure-aks-backup --cluster-type managedClusters --cluster-name $CLUSTER_2 -g $RESOURCE_GROUP_2

# Enable Trusted Access in AKS

$BACKUP_VAULT_ID=$(az dataprotection backup-vault show --vault-name $BACK_VAULT_NAME -g $RESOURCE_GROUP_VAULT --query id -o tsv)

az aks trustedaccess rolebinding create `
   -g $RESOURCE_GROUP_2 `
   --cluster-name $CLUSTER_2 `
   –n trustedaccess `
   -s $BACKUP_VAULT_ID `
   --roles Microsoft.DataProtection/backupVaults/backup-operator
```

We can now create the backup policy and backup instance

```powershell
az dataprotection backup-instance create -g MyResourceGroup --vault-name MyVault --backup-instance backupinstance.json

az backup container register --resource-group $RESOURCE_GROUP_1 --vault-name $BACK_VAULT_NAME --subscription $SUBSCRIPTION_ID --backup-management-type AzureKubernetesService --workload-type AzureKubernetesService --query properties.friendlyName -o tsv

$CONTAINER_NAME=$(az backup container list --resource-group $RESOURCE_GROUP_1 --vault-name $BACK_VAULT_NAME --subscription $SUBSCRIPTION_ID --backup-management-type AzureKubernetesService --query "[0].name" -o tsv)

az backup item set-policy --resource-group $RESOURCE_GROUP_1 --vault-name $BACK_VAULT_NAME --subscription $SUBSCRIPTION_ID --container-name $CONTAINER_NAME --item-name $CONTAINER_NAME --policy-name "aks-backup-policy"
```

Finally we can create a backup job for each cluster

```powershell
az backup job start --resource-group $RESOURCE_GROUP_1 --vault-name $BACK_VAULT_NAME --subscription $SUBSCRIPTION_ID --container-name $CONTAINER_NAME --item-name $CONTAINER_NAME --backup-management-type AzureKubernetesService --workload-type AzureKubernetesService --operation TriggerBackup

az backup job start --resource-group $RESOURCE_GROUP_2 --vault-name $BACK_VAULT_NAME --subscription $SUBSCRIPTION_ID --container-name $CONTAINER_NAME --item-name $CONTAINER_NAME --backup-management-type AzureKubernetesService --workload-type AzureKubernetesService --operation TriggerBackup
```

Now lets check the status of the backup jobs

```powershell
az aks get-credentials -n $CLUSTER_1 -g $RESOURCE_GROUP_1 --overwrite-existing
kubectl get pods -n dataprotection-microsoft
```

You should see something like this

```output
# NAME                                                         READY   STATUS    RESTARTS      AGE
# dataprotection-microsoft-controller-7b8977698c-v2rl7         2/2     Running   0             94m
# dataprotection-microsoft-geneva-service-6c8457bbd-jgw49      2/2     Running   0             94m
# dataprotection-microsoft-kubernetes-agent-5558dbbf8f-5tdkc   2/2     Running   2 (94m ago)   94m
```


```powershell
kubectl get nodes
```

```output
# NAME                                 STATUS   ROLES   AGE   VERSION
# aks-systempool-20780455-vmss000000   Ready    agent   28m   v1.25.5
# aks-systempool-20780455-vmss000001   Ready    agent   28m   v1.25.5
# aks-systempool-20780455-vmss000002   Ready    agent   28m   v1.25.5
```

```powershell
kubectl apply -f deploy_disk_lrs.yaml
```

```powershell
kubectl apply -f deploy_disk_zrs_sc.yaml
```

```powershell
kubectl get pods,svc,pv,pvc
```
```output
# NAME                             READY   STATUS    RESTARTS   AGE
# pod/nginx-lrs-7db4886f8c-x4hzz   1/1     Running   0          90s
# pod/nginx-zrs-5567fd9ddc-hbtfs   1/1     Running   0          80s

# NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
# service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   30m

# NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS      REASON   AGE
# persistentvolume/pvc-c3fc20ea-2922-477c-a337-895b8b503a9b   5Gi        RWO            Delete           Bound    default/azure-managed-disk-lrs   managed-csi                86s
# persistentvolume/pvc-f1055e1c-b8e1-4604-8567-1f288daced02   5Gi        RWO            Delete           Bound    default/azure-managed-disk-zrs   managed-csi-zrs            76s

# NAME                                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
# persistentvolumeclaim/azure-managed-disk-lrs   Bound    pvc-c3fc20ea-2922-477c-a337-895b8b503a9b   5Gi        RWO            managed-csi       90s
# persistentvolumeclaim/azure-managed-disk-zrs   Bound    pvc-f1055e1c-b8e1-4604-8567-1f288daced02   5Gi        RWO            managed-csi-zrs   80s
```

```powrshell
kubectl exec nginx-lrs-7db4886f8c-x4hzz -it -- cat /mnt/azuredisk/outfile
# Tue Mar 21 15:00:14 UTC 2023
# Tue Mar 21 15:01:14 UTC 2023
# Tue Mar 21 15:02:14 UTC 2023
# Tue Mar 21 15:03:14 UTC 2023

kubectl exec nginx-zrs-5567fd9ddc-hbtfs -it -- cat /mnt/azuredisk/outfile
# Tue Mar 21 15:00:48 UTC 2023
# Tue Mar 21 15:01:48 UTC 2023
# Tue Mar 21 15:02:48 UTC 2023
# Tue Mar 21 15:03:48 UTC 2023
```

## 10. Lab 8: Configure Velero using Azure Blob Storage on a cluster