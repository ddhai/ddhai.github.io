---
 layout: post
 tags: K8s
 title: Kubernetes Migration - How To Move Data Freely Across Clusters
---
### Kubernetes Migration: How To Move Data Freely Across Clusters

We will be migrating our entire data from Google Kubernetes Engine to Azure Kubernetes Service using Velero.

### Prerequisite
A Kubernetes cluster > 1.10

### Setup Velero with Restic Integration

Velero consists of a client installed on your local computer and a server that runs in your Kubernetes cluster, like Helm.

### Installing Velero Client

You can find the latest release corresponding to your OS and system and download Velero from there:

```shell
$ wget https://github.com/vmware-tanzu/velero/releases/download/v1.3.1/velero-v1.3.1-linux-amd64.tar.gz
```

Extract the tarball (change the version depending on yours) and move the Velero binary to `/usr/local/bin`

```shell
$ tar -xvzf velero-v0.11.0-darwin-amd64.tar.gz
$ sudo mv velero /usr/local/bin/
$ velero help
```

### Create a Bucket for Velero on GCP

Velero needs an object storage bucket where it will store the backup. Create a GCS bucket using:

```shell
gsutil mb gs://<bucket-name>
```

### Create a Service Account for Velero

```shell
# Create a Service Account
gcloud iam service-accounts create velero --display-name "Velero service account"
SERVICE_ACCOUNT_EMAIL=$(gcloud iam service-accounts list --filter="displayName:Velero service account" --format 'value(email)')

#Define Permissions for the Service Account
ROLE_PERMISSIONS=(
compute.disks.get
compute.disks.create
compute.disks.createSnapshot
compute.snapshots.get
compute.snapshots.create
compute.snapshots.useReadOnly
compute.snapshots.delete
compute.zones.get
)

# Create a Role for Velero
PROJECT_ID=$(gcloud config get-value project)

gcloud iam roles create velero.server \
--project $PROJECT_ID \
--title "Velero Server" \
--permissions "$(IFS=","; echo "${ROLE_PERMISSIONS[*]}")"

# Create a Role Binding for Velero
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member serviceAccount:$SERVICE_ACCOUNT_EMAIL \
--role projects/$PROJECT_ID/roles/velero.server

gsutil iam ch serviceAccount:$SERVICE_ACCOUNT_EMAIL:objectAdmin

# Generate Service Key file for Velero and save it for later
gcloud iam service-accounts keys create credentials-velero \
--iam-account $SERVICE_ACCOUNT_EMAIL
```

### Install Velero Server on GKE and AKS

Use the — use-restic flag on the Velero install command to install restic integration.

```shell
$ velero install \
--use-restic \
--bucket  \
--provider gcp \
--secret-file  \
--use-volume-snapshots=false \
--plugins=--plugins restic/restic
$ velero plugin add velero/velero-plugin-for-gcp:v1.0.1
$ velero plugin add velero/velero-plugin-for-microsoft-azure:v1.0.0
```

After that, you can see a DaemonSet of restic and deployment of Velero in your Kubernetes cluster.

```shell
$ kubectl get po -n velero
```

### Restic Components

- In addition, there are three more Custom Resource Definitions and their associated controllers to provide restic support.

#### Restic Repository

- Maintain the complete lifecycle for Velero’s restic repositories.
- Restic lifecycle commands such as restic init check and prune are handled by this CRD controller.

#### PodVolumeBackup

- This CRD backs up the persistent volume based on the annotated pod in selected namespaces.
- This controller executes backup commands on the pod to initialize backups.

#### PodVolumeRestore

- This controller restores the respective pods that were inside restic backups. And this controller is responsible for the restore commands execution.

### Backup an application on GKE

For this blog post, we are considering that Kubernetes already has an application that is using persistent volumes. Or you can install WordPress as an example as explained [here](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/).

We will perform GKE Persistent disk migration to Azure Persistent Disk using Velero.
Follow the below steps:

1. To back up, the deployment or statefulset checks for the volume name that is mounted to backup that particular persistent volume. For example, here pods need to be annotated with Volume Name “data”.

```yaml
volumes:
    - name: data
        persistentVolumeClaim:
            claimName: mongodb 
```

2. Annotate the pods with the volume names, you’d like to take the backup of and only those volumes will be backed up:

```shell
$ kubectl -n NAMESPACE annotate pod/POD_NAME backup.velero.io/backup-volumes=VOLUME_NAME1,VOLUME_NAME2
```

For example,

```shell
$ kubectl -n application annotate pod/wordpress-pod backup.velero.io/backup-volumes=data
```

3. Take a backup of the entire namespace in which the application is running. You can also specify multiple namespaces or skip this flag to backup all namespaces by default. We are going to backup only one namespace in this blog.

```shell
$ velero backup create testbackup --include-namespaces application
```

4. Monitor the progress of backup:

```shell
$ velero backup describe testbackup --details       
```

Once the backup is complete, you can list it using:

```shell
$ velero backup get
```

You can also check the backup on GCP Portal under Storage.
Select the bucket you created and you should see a similar directory structure:

### Restore the application to AKS

Follow the below steps to restore the backup:

1. Make sure to have the same StorageClass available in Azure as used by GKE Persistent Volumes. For example, if the Storage Class of the PVs is “persistent-ssd”, create the same on AKS using the template below:

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: persistent-ssd // same name as GKE storageclass name
provisioner: kubernetes.io/azure-disk
parameters: 
  storageaccounttype: Premium_LRS
  kind: Managed 
```

You can monitor the progress of restore:

```shell
$ velero restore create testrestore --from-backup testbackup 
```
You can also check on GCP Portal, a new folder “restores” is created under the bucket.
In some time, you should be able to see that the application namespace is back and WordPress and MySQL pods are running again.

### Troubleshooting

For any errors/issues related to Velero, you may find below commands helpful for debugging purposes:

```shell
# Describe the backup to see the status
$ velero backup describe testbackup --details

# Check backup logs, and look for errors if any
$ velero backup logs testbackup

# Describe the restore to see the status
$ velero restore describe testrestore --details

# Check restore logs, and look for errors if any
$ velero restore logs testrestore

# Check velero and restic pod logs, and look for errors if any
$ kubectl -n velero logs VELERO_POD_NAME/RESTIC_POD_NAME
NOTE: You can change the default log-level to debug mode by adding --log-level=debug as an argument to the container command in the velero pod template spec.

# Describe the BackupStorageLocation resource and look for any errors in Events
$ kubectl describe BackupStorageLocation default -n velero
```

### Conclusion

The migration of persistent workloads across Kubernetes clusters on different cloud providers is difficult. This became possible by using restic integration with the Velero backup tool. This tool is still said to be in beta quality as mentioned on the official site. I have performed GKE to AKS migration and it went successfully. You can try other combinations of different cloud providers for migrations.

The only drawback of using Velero to migrate data is if your data is too huge, it may take a while to complete the migration. It took me almost a day to migrate a 350 GB disk from GKE to AKS. But, if your data is comparatively less, this should be an efficient and hassle-free way to migrate it.
