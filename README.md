# Backup-of-Kubernetes-Cluster-via-Velero-Minio
This is for on-prem servers

## Prerequisites for Setting Up Velero with Minio S3 Object Storage

Here are the prerequisites for setting up Velero with Minio S3 object storage:

**Infrastructure:**

1. **Kubernetes Cluster:** You need a running Kubernetes cluster (version 1.7 or later) with `kubectl` installed for interacting with the cluster. Ensure the DNS server is functional within the cluster.
2. **Minio Server VM:**  A separate virtual machine dedicated to running the Minio server, which will act as your S3-compatible object storage. 

**Software:**

1* **Minio Client Binary:** You'll need the Minio client binary installed on the Minio server VM. 
2* **Velero Server:**  Velero server needs to be installed on the master node of your Kubernetes cluster. 
3* **Velero CLI:** The Velero command-line interface (CLI) needs to be installed on a machine with access to the Kubernetes cluster (can be the master node itself).
4* **mc** (Minio Client) command-line interface is a tool for interacting with your Minio S3 object storage bucket.

## Steps:

**Step 1: Install Minio Client Binary (on VM dedicated for Minio Server)**
	
	wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20240528171904.0.0_amd64.deb -O minio.deb
	sudo dpkg -i minio.deb
	
**Step 2: Start the Minio S3 Object Storage (on Minio Server VM)**
	
	mkdir ~/minio
	minio server ~/minio --console-address :9001
	
The output looks something like this 

	$minio server ~/minio --console-address :9001
	Formatting 1st pool, 1 set(s), 1 drives per set.
	WARNING: Host local has more than 0 drives of set. A host failure will result in data becoming unavailable.
	MinIO Object Storage Server
	Copyright: 2015-2024 MinIO, Inc.
	License: GNU AGPLv3 - https://www.gnu.org/licenses/agpl-3.0.html
	Version: RELEASE.2024-05-28T17-19-04Z (go1.22.3 linux/amd64)

	API: http://10.1.1.183:9000  http://172.17.0.1:9000  http://127.0.0.1:9000
	   RootUser: minioadmin
	   RootPass: minioadmin

	WebUI: http://10.1.1.183:9001 http://172.17.0.1:9001 http://127.0.0.1:9001
	   RootUser: minioadmin
	   RootPass: minioadmin

	CLI: https://min.io/docs/minio/linux/reference/minio-mc.html#quickstart
	   $ mc alias set 'myminio' 'http://10.1.1.183:9000' 'minioadmin' 'minioadmin'

	Docs: https://min.io/docs/minio/linux/index.html
	Status:         1 Online, 0 Offline.
	STARTUP WARNINGS:
	- Detected default credentials 'minioadmin:minioadmin', we recommend that you change these values with 'MINIO_ROOT_USER' and 'MINIO_ROOT_PASSWORD' environment variables
	- The standard parity is set to 0. This can lead to data loss.

In Order to access the WebUI copy and paste the WebUI in the same server where you have hosted the minio s3 bucket. And the default password are given on the screen itself which  
is minioadmin both as accesskey and secretkey. Create a Bucket in the minio s3 object Storage. 

**Step 3: Install Velero Client (on Kubernetes Master Node)**

	wget https://github.com/vmware-tanzu/velero/releases/download/v1.13.2/velero-v1.13.2-linux-amd64.tar.gz
	tar -zxvf velero-v1.13.2-linux-amd64.tar.gz
	sudo mv velero-v1.13.2-linux-amd64/velero /usr/local/bin/
	
	
	
**Step 4: Install Velero Server (on Kubernetes Master Node)**

Before we install the Velero Server we need to create a file called minioSecret where we will enter the access-key and the secret-key of the minio s3 bucket

	vi minioSecret
	
	[default]
    aws_access_key_id = <YOUR_MINIO_ACCESS_KEY>
    aws_secret_access_key = <YOUR_MINIO_SECRET_KEY> #in our case the both are minioadmin

save and quit the file, then on the same directory run the below command to install velero server on our kubernetes cluster.
	
	velero install --use-node-agent 
	--provider aws \
	--plugins velero/velero-plugin-for-aws:latest \
	--bucket prashant123 \
	--secret-file minioSecret \ 
	--use-volume-snapshots=false \
	--backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<IP address of the minio server>:9000

**Step 5: Configure Velero Backup Storage Location (on Master Node)**

	vi velero-minio-config.yaml

	apiVersion: velero.io/v1
	kind: BackupStorageLocation
	metadata:
	  name: default
	  namespace: velero
	spec:
	  provider: aws
	  default: true
	  objectStorage:
		bucket: <Name of the bucket in minio S3 object Storage>
		prefix: velero
	  config:
		region: minio
		s3ForcePathStyle: "true"
		s3Url: http://<IP address of the minio server>:9000

	---
	apiVersion: velero.io/v1
	kind: VolumeSnapshotLocation
	metadata:
	  name: default
	  namespace: velero
	spec:
	  provider: aws
	  config:
		region: minio


execute this command to see if there are any errors or working fine.
	
	kubectl logs deployment/velero -n velero

To view the backup location execute this command
	
	velero get backup-location
	
	
#  Here's the list of commands that will vbe used to create backup.

#  Backup a namespace and it’s objects.

    velero backup create <backup-name> --include-namespaces <namespace>

#  Backup all deployments in the cluster.

    velero backup create <backup-name> --include-resources deployments

#  Backup the deployments in a namespace.

    velero backup create <backup-name> --include-resources deployments --include-namespaces <namespace>

#  Backup entire cluster including cluster-scoped resources.

    velero backup create <backup-name>

#Backup a namespace and include cluster-scoped resources.

    velero backup create <backup-name> --include-namespaces <namespace> --include-cluster-resources=true

#Include resources matching the label selector.

    velero backup create <backup-name> --selector <key>=<value>

#Include resources that are not matching the selector

    velero backup create <backup-name> --selector "<key> notin (<value>)"

#Exclude kube-system from the cluster backup.

    velero backup create <backup-name> --exclude-namespaces kube-system

#Exclude secrets from the backup.

    velero backup create <backup-name> --exclude-resources secrets

#Exclude secrets and rolebindings.

    velero backup create <backup-name> --exclude-resources secrets,rolebindings



**Important Notes:**

* Replace placeholders like `<access-key>`, `<secret-key>`, `<bucket-name>`, and `<endpoint>` with your actual Minio credentials, bucket name, and server endpoint details.
* Ensure the Velero server and Minio server can communicate. Firewall rules or security groups might need adjustments to allow access.
* This is a basic setup guide. Refer to the Velero documentation for detailed configuration options and best practices: [https://velero.io/docs/v1.7/](https://velero.io/docs/v1.7/)

By following these steps, you should have the prerequisites fulfilled and Velero configured to use your Minio S3 object storage for backing up your Kubernetes cluster resources. 
